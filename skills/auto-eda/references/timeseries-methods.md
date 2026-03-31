# 时间序列分析方法

> 业务数据中最常见的数据类型之一。趋势、季节性、突变点——每一个都可能是业务洞察。

---

## 一、时序分析路径决策树

```
时间序列数据 →
├── 目标：理解历史模式？
│   └── 分解分析（趋势 + 季节性 + 残差）
│
├── 目标：检测异常/突变？
│   ├── 单指标 → 控制图 / Z-score / STL 残差异常
│   └── 多指标 → 相关突变点检测
│
├── 目标：预测未来？
│   ├── 数据量小 / 快速基线 → 指数平滑 / ARIMA
│   ├── 有明显季节性 + 节假日 → Prophet
│   ├── 多变量 / 复杂特征 → LightGBM + 时序特征工程
│   └── 超长序列 / 多步预测 → 考虑深度学习（提醒用户超出 EDA 范围）
│
└── 目标：比较不同时期/分组？
    └── DiD / 中断时间序列分析 (ITS)
```

---

## 二、时序数据预处理

### 2.1 时间列处理

```python
# 解析时间列
df['date'] = pd.to_datetime(df['date'])
df = df.sort_values('date')

# 设置时间索引
df = df.set_index('date')

# 检查时间间隔是否均匀
time_diffs = df.index.to_series().diff().dropna()
print(f"时间间隔统计:\n{time_diffs.describe()}")
# 如果不均匀 → 需要重采样
```

### 2.2 缺失时间点处理

```python
# 生成完整时间范围
full_range = pd.date_range(start=df.index.min(), end=df.index.max(), freq='D')
df = df.reindex(full_range)

# 缺失值填充策略
df['value'] = df['value'].interpolate(method='time')  # 时间插值（推荐）
# 或 df['value'].fillna(method='ffill')  # 前向填充
```

### 2.3 频率与重采样

| 原始频率 | 分析需求 | 方法 |
|---------|---------|------|
| 秒/分钟级 | 看日趋势 | `df.resample('D').agg({'value': 'sum', 'count': 'count'})` |
| 日级 | 看周/月趋势 | `df.resample('W').mean()` |
| 不规则间隔 | 统一频率 | 先 reindex 再插值 |

---

## 三、时序分解

### 3.1 STL 分解（推荐）

```python
from statsmodels.tsa.seasonal import STL

stl = STL(df['value'], period=7)  # period: 季节性周期（日数据=7，月数据=12）
result = stl.fit()

fig = result.plot()
# result.trend    — 趋势成分
# result.seasonal — 季节性成分
# result.resid    — 残差（异常信号）
```

### 3.2 分解结果解读

| 成分 | 业务含义 | 关注点 |
|------|---------|--------|
| 趋势 | 长期增长/衰退方向 | 趋势是否在转折？什么时候开始的？ |
| 季节性 | 周期性波动（周、月、年） | 周几/哪个月是高峰？振幅是否稳定？ |
| 残差 | 去除趋势和季节性后的异常 | 大残差 = 异常事件候选 |

---

## 四、平稳性检验

### 4.1 为什么要检验平稳性

ARIMA 等经典模型要求序列平稳（均值和方差不随时间变化）。非平稳序列需要差分处理。

### 4.2 ADF 检验

```python
from statsmodels.tsa.stattools import adfuller

result = adfuller(df['value'].dropna())
print(f"ADF 统计量: {result[0]:.4f}")
print(f"p-value: {result[1]:.4f}")
# p < 0.05 → 平稳
# p >= 0.05 → 非平稳，需要差分
```

### 4.3 差分策略

```python
# 一阶差分（去趋势）
df['diff_1'] = df['value'].diff(1)

# 季节性差分（去季节性，如周期=7）
df['diff_seasonal'] = df['value'].diff(7)

# 再次检验平稳性
```

**业务语言**："一阶差分"= 看每天的变化量而非绝对值；"季节性差分"= 和上周同一天比。

---

## 五、预测方法

### 5.1 方法选择指南

| 场景 | 推荐方法 | 理由 |
|------|---------|------|
| 快速基线，数据简单 | 移动平均 / 指数平滑 | 实现简单，可解释 |
| 经典时序，无外部变量 | ARIMA / SARIMA | 统计严谨，适合短期预测 |
| 有季节性 + 节假日 + 趋势变化 | Prophet | 自动处理季节性，易于调参 |
| 有丰富外部特征 | LightGBM + 时序特征 | 灵活，处理非线性关系 |

### 5.2 Prophet（推荐首选）

```python
# pip install prophet
from prophet import Prophet

# 准备数据格式
prophet_df = df.reset_index().rename(columns={'date': 'ds', 'value': 'y'})

model = Prophet(
    yearly_seasonality=True,
    weekly_seasonality=True,
    daily_seasonality=False,
    changepoint_prior_scale=0.05,  # 控制趋势灵活度
)

# 添加节假日（中国）
# model.add_country_holidays(country_name='CN')

model.fit(prophet_df)

# 预测未来 30 天
future = model.make_future_dataframe(periods=30)
forecast = model.predict(future)

# 可视化
model.plot(forecast)
model.plot_components(forecast)  # 分解趋势 + 季节性
```

### 5.3 ARIMA

```python
from statsmodels.tsa.arima.model import ARIMA
import pmdarima as pm  # pip install pmdarima

# 自动选择 (p, d, q) 参数
auto_model = pm.auto_arima(
    df['value'],
    seasonal=True, m=7,  # m=季节性周期
    stepwise=True,
    suppress_warnings=True,
)
print(auto_model.summary())

# 预测
forecast = auto_model.predict(n_periods=30, return_conf_int=True)
```

### 5.4 LightGBM 时序特征工程

```python
def create_time_features(df):
    """从时间索引创建特征"""
    df['dayofweek'] = df.index.dayofweek
    df['month'] = df.index.month
    df['quarter'] = df.index.quarter
    df['dayofyear'] = df.index.dayofyear
    df['is_weekend'] = df.index.dayofweek.isin([5, 6]).astype(int)

    # 滞后特征
    for lag in [1, 7, 14, 28]:
        df[f'lag_{lag}'] = df['value'].shift(lag)

    # 滚动统计
    for window in [7, 14, 28]:
        df[f'rolling_mean_{window}'] = df['value'].shift(1).rolling(window).mean()
        df[f'rolling_std_{window}'] = df['value'].shift(1).rolling(window).std()

    return df
```

⚠️ **数据泄漏警告**：所有滞后和滚动特征必须 `shift(1)` 以上，绝不能用当天的数据。训练/测试集必须按时间顺序划分，不能随机。

---

## 六、异常检测

### 6.1 基于 STL 残差

```python
residuals = result.resid
threshold = residuals.std() * 3  # 3-sigma 规则
anomalies = df[abs(residuals) > threshold]
```

### 6.2 基于移动窗口

```python
rolling_mean = df['value'].rolling(window=7).mean()
rolling_std = df['value'].rolling(window=7).std()
upper = rolling_mean + 3 * rolling_std
lower = rolling_mean - 3 * rolling_std
anomalies = df[(df['value'] > upper) | (df['value'] < lower)]
```

### 6.3 突变点检测

```python
# 使用 ruptures 库
# pip install ruptures
import ruptures as rpt

signal = df['value'].values
algo = rpt.Pelt(model="rbf").fit(signal)
change_points = algo.predict(pen=10)  # pen 控制灵敏度
```

**业务语言**：突变点 = "指标在某个时间点发生了显著的结构性变化，之前的规律不再适用"。

---

## 七、时序分析的偏见检查

| 检查项 | 说明 |
|--------|------|
| 季节性混淆 | "销售额下降"是真正下降还是季节性淡季？同比 vs 环比 |
| 异常值驱动 | 趋势变化是否被少数极端值拉动？尝试 winsorize 后再看 |
| 预测区间 | 永远展示置信区间，不给单点预测 |
| 外推风险 | 历史规律不一定延续——提醒用户预测越远越不可靠 |
| 数据频率陷阱 | 日级数据聚合为月级后，波动被平滑，可能掩盖问题 |
| 趋势 vs 因果 | "两条线同时上升"不代表有因果关系 |
| 预测评估 | 用时间划分的验证集评估，不能用随机划分 |

---

## 八、结果呈现

### 图表推荐

| 分析内容 | 推荐图表 |
|---------|---------|
| 原始趋势 | 折线图 + 移动平均线 |
| 分解结果 | 四面板图（原始 + 趋势 + 季节性 + 残差） |
| 异常点 | 折线图 + 红色标注异常点 |
| 预测 | 历史实线 + 预测虚线 + 置信区间阴影 |
| 突变点 | 折线图 + 垂直虚线标注突变时间 |
| 季节性模式 | 热力图（x=周/月, y=小时/天） |

### 业务语言翻译

| 统计语言 | 业务语言 |
|---------|---------|
| ADF p < 0.05 | 这个指标有稳定的波动规律 |
| 一阶差分 d=1 | 我们分析的是每天的增量变化 |
| ARIMA(1,1,1) | 基于历史趋势和近期波动的预测模型 |
| 置信区间 [100, 150] | 预计在 100-150 之间，但超出此范围也不罕见 |
| 突变点 | XX 日前后，指标的运行规律发生了结构性变化 |
