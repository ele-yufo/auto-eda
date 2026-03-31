# 数据清洗 & 特征工程

> 从"数据已加载"到"数据可分析"之间的关键步骤。这通常占据分析时间的 60-80%，也是最容易引入偏差的环节。

---

## 一、数据清洗决策树

```
数据已加载 →
├── 1. 缺失值处理
│   ├── 缺失 < 5% → 删除行 或 简单填充
│   ├── 缺失 5-30% → 根据类型选择填充策略
│   ├── 缺失 > 50% → 考虑删除该列（先调查缺失原因）
│   └── ⚠️ 缺失不随机？→ 缺失本身可能是特征
│
├── 2. 异常值处理
│   ├── 明显错误（年龄 -5, 金额 9999999）→ 修正或标记
│   ├── 极端但合理 → 保留，考虑 winsorize
│   └── ⚠️ 不要自动删除异常值——先理解它们
│
├── 3. 数据类型修正
│   ├── ID 被识别为数字 → 转 str
│   ├── 日期是字符串 → 转 datetime
│   └── 分类变量是数字编码 → 确认含义
│
├── 4. 重复值处理
│   ├── 完全重复 → 去重
│   └── 主键重复但值不同 → 调查数据源问题
│
└── 5. 一致性检查
    ├── 同一实体不同拼写/编码 → 标准化
    ├── 逻辑矛盾（结束日期 < 开始日期）→ 标记
    └── 跨列约束违反 → 报告
```

---

## 二、缺失值处理

### 2.1 先诊断再处理

```python
# 缺失值全景
missing = df.isnull().sum()
missing_pct = (missing / len(df) * 100).round(1)
missing_report = pd.DataFrame({
    '缺失数': missing,
    '缺失率%': missing_pct,
    '数据类型': df.dtypes
}).query('缺失数 > 0').sort_values('缺失率%', ascending=False)

print(missing_report)
```

### 2.2 缺失机制判断

| 机制 | 含义 | 处理方式 |
|------|------|---------|
| MCAR (完全随机) | 缺失与任何变量无关 | 安全删除或简单填充 |
| MAR (随机) | 缺失与其他已观测变量有关 | 条件填充 / 多重插补 |
| MNAR (非随机) | 缺失与缺失值本身有关 | ⚠️ 最危险，需要领域知识，缺失本身可能是特征 |

**快速判断**：按缺失/非缺失分组，检查其他变量的分布是否有显著差异。有差异 → 不是 MCAR。

### 2.3 填充策略速查

| 数据类型 | 缺失少 | 缺失多 |
|---------|--------|--------|
| 连续-对称 | 均值 | 中位数 + 标记缺失 |
| 连续-偏态 | 中位数 | 中位数 + 标记缺失 |
| 分类 | 众数 | 单独类别"Unknown" |
| 时序 | 插值 | 前向/后向填充 |
| 有分组结构 | 组内均值/中位数 | 组内统计 + 标记 |

```python
# 标记缺失（缺失本身可能有信息）
df['col_was_missing'] = df['col'].isnull().astype(int)

# 分组填充
df['col'] = df.groupby('group')['col'].transform(lambda x: x.fillna(x.median()))
```

⚠️ **永远在报告中注明**清洗前后的行数变化和填充策略。

---

## 三、异常值处理

### 3.1 检测方法

```python
# IQR 方法
Q1 = df['col'].quantile(0.25)
Q3 = df['col'].quantile(0.75)
IQR = Q3 - Q1
lower = Q1 - 1.5 * IQR
upper = Q3 + 1.5 * IQR
outliers = df[(df['col'] < lower) | (df['col'] > upper)]

# Z-score 方法
from scipy import stats
z_scores = abs(stats.zscore(df['col'].dropna()))
outliers = df[z_scores > 3]
```

### 3.2 处理决策

| 情况 | 处理 |
|------|------|
| 数据录入错误（年龄 = 999） | 修正为缺失，按缺失值处理 |
| 极端但真实（CEO 薪资） | 保留，分析时标注 |
| 会影响均值/回归 | 考虑 winsorize（截尾到 1%/99% 分位数） |
| 建模场景 | 保留并用稳健方法（中位数、树模型） |

```python
# Winsorize（不删除，而是限制极端值）
from scipy.stats.mstats import winsorize
df['col_winsorized'] = winsorize(df['col'], limits=[0.01, 0.01])
```

### 3.3 何时可以删除异常值

默认立场是**不删除**，但以下情况可以考虑删除（需在报告中说明）：

| 场景 | 可以删除？ | 条件 |
|------|-----------|------|
| 明确的数据录入错误 | ✅ | 值在物理/逻辑上不可能（年龄 -5，负数金额） |
| 重复录入导致的异常 | ✅ | 确认为重复数据 |
| 影响统计检验的极端值 | ⚠️ 谨慎 | 先做 winsorize，对比删除前后结论是否变化 |
| 影响回归/均值的极端值 | ⚠️ 谨慎 | 用稳健方法（中位数、树模型）替代删除 |
| 用户要求排除的特定记录 | ✅ | 记录排除原因 |
| "看起来不正常"但无法解释 | ❌ | 保留，单独标注，分析时报告其影响 |

**原则**：删除异常值必须有明确理由，且在 `_eda_state.md` 中记录。同时报告"包含/排除异常值"两种情况下的结论差异。

---

## 四、特征工程

### 何时做特征工程？

| 分析阶段 | 需要特征工程？ | 说明 |
|---------|-------------|------|
| 探索性 EDA | 基础即可 | 日期拆分、简单比率。不要过度工程化 |
| 分组对比/统计检验 | 一般不需要 | 直接用原始特征 |
| 建模/预测 | 需要 | 编码、缩放、交互特征都可能有帮助 |
| 聚类/分群 | 需要标准化 | 量纲差异会严重影响聚类结果 |

**原则**：探索阶段保持简单，建模阶段按需工程化。不要在 EDA 阶段花大量时间做复杂特征工程——那是正式建模项目的事。

### 4.1 日期时间特征

```python
df['year'] = df['date'].dt.year
df['month'] = df['date'].dt.month
df['dayofweek'] = df['date'].dt.dayofweek
df['is_weekend'] = df['date'].dt.dayofweek.isin([5, 6]).astype(int)
df['quarter'] = df['date'].dt.quarter
df['days_since_event'] = (df['date'] - pd.Timestamp('2024-01-01')).dt.days
```

### 4.2 分类变量编码

| 方法 | 适用 | 注意 |
|------|------|------|
| One-Hot | 类别 < 10 且无序 | 高基数会爆维 |
| Label Encoding | 有序分类（低/中/高） | 模型可能误解为数值距离 |
| Target Encoding | 高基数分类 | ⚠️ 必须在训练集上计算，防止泄漏 |
| Frequency Encoding | 高基数 + 快速基线 | 简单有效 |

```python
# One-Hot
df = pd.get_dummies(df, columns=['category'], drop_first=True)

# Target Encoding（注意防泄漏）
from sklearn.model_selection import KFold
# 需用交叉验证方式在训练集上计算
```

### 4.3 数值变换

| 场景 | 方法 | 代码 |
|------|------|------|
| 严重右偏 (偏度 > 2) | 对数变换 | `np.log1p(df['col'])` |
| 量纲差异大 | 标准化 | `StandardScaler()` |
| 需要 [0,1] 范围 | 归一化 | `MinMaxScaler()` |
| 异常值多 | 稳健缩放 | `RobustScaler()` |

### 4.4 交互特征

```python
# 比率特征
df['revenue_per_user'] = df['revenue'] / df['users'].clip(lower=1)

# 差值特征
df['price_diff'] = df['current_price'] - df['original_price']

# 分箱
df['age_group'] = pd.cut(df['age'], bins=[0, 18, 30, 50, 100],
                          labels=['<18', '18-30', '30-50', '50+'])
```

---

## 五、多表关联

### 5.1 JOIN 前检查

```python
# 检查 JOIN 键的唯一性和覆盖率
print(f"左表唯一键: {df_left['key'].nunique()}, 总行: {len(df_left)}")
print(f"右表唯一键: {df_right['key'].nunique()}, 总行: {len(df_right)}")

# 检查交集
overlap = set(df_left['key']).intersection(set(df_right['key']))
print(f"交集: {len(overlap)} / 左表: {len(set(df_left['key']))} / 右表: {len(set(df_right['key']))}")
```

### 5.2 JOIN 策略

| 场景 | 方法 | 注意 |
|------|------|------|
| 一对一 | `pd.merge(left, right, on='key')` | 检查合并后行数是否变化 |
| 一对多 | `pd.merge(orders, customers, on='customer_id')` | 结果行数 = 多的一方 |
| 多对多 | ⚠️ 先聚合到一对一或一对多 | 直接 merge 会行数爆炸 |
| 左表为主 | `pd.merge(..., how='left')` | 保留左表所有行 |

```python
# 合并后验证
merged = pd.merge(df_left, df_right, on='key', how='left')
print(f"合并前: {len(df_left)} 行 → 合并后: {len(merged)} 行")
# 如果行数增加 → 右表存在重复键，需要调查
```

---

## 六、清洗质量检查清单

每次清洗后，运行以下检查：

```
□ 行数变化是否合理？（清洗前 → 清洗后，删了多少？为什么？）
□ 填充后的分布是否合理？（填充值没有创造出不存在的峰值？）
□ 数据类型是否正确？（ID 是 str？日期是 datetime？）
□ 没有引入数据泄漏？（未来信息没有混入特征？）
□ 编码后的分类变量含义是否正确？
□ 缺失值处理策略已记录在 _eda_state.md 中？
□ 清洗决策能向用户解释清楚？
```

---

## 七、数据隐私速查

在开始分析之前，快速检查数据中是否包含敏感信息：

```
□ 个人姓名、身份证号、手机号、邮箱地址
□ 银行卡号、信用卡信息
□ 医疗记录、健康状态
□ 精确地理位置（经纬度、详细地址）
□ 任何可用于直接识别个人身份的字段
```

如果发现敏感数据：
1. **立即告知用户**："我注意到数据中包含 [姓名/手机号/...]，建议在分析前脱敏处理"
2. 在分析过程中不展示原始敏感值（用 ID 或脱敏值替代）
3. Dashboard 中不包含可识别个人的数据
4. 在 `_eda_state.md` 中标注数据隐私风险
