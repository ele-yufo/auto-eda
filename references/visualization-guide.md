# 可视化选择指南

> 选对图表比做美图更重要。面向管理者的图表原则：少即是多，突出 insight。

---

## 一、数据类型 → 图表类型决策矩阵

| 要展示的关系 | 数据类型 | 推荐图表 | 代码关键参数 |
|---|---|---|---|
| 单变量分布 | 连续 | 直方图 + KDE | `sns.histplot(kde=True)` |
| 单变量分布 | 分类 | 条形图 | `sns.countplot()` |
| 两变量关系 | 连续 × 连续 | 散点图 | `sns.scatterplot()` |
| 两变量关系 | 连续 × 分类 | 箱线图/小提琴图 | `sns.boxplot()` / `sns.violinplot()` |
| 两变量关系 | 分类 × 分类 | 热力图/堆叠条形图 | `sns.heatmap()` |
| 多变量关系 | 连续 × 多个 | 配对图 | `sns.pairplot()` |
| 相关性矩阵 | 多连续变量 | 热力图 | `sns.heatmap(annot=True)` |
| 趋势 | 时间序列 | 折线图 | `plt.plot()` / `sns.lineplot()` |
| 组成 | 比例 | 条形图（水平） | `plt.barh()` |
| 比较排名 | 分类+数值 | 水平条形图 | `plt.barh(sorted)` |
| 分布对比 | 多组连续 | 小提琴图 | `sns.violinplot()` |
| 异常值 | 连续 | 箱线图 | `sns.boxplot()` |

**注意**：
- ❌ 避免饼图（人眼不擅长比较角度），用水平条形图替代
- ❌ 避免 3D 图表（增加复杂度但没有信息增益）
- ✅ 优先使用条形图和折线图（最直觉）

---

## 二、面向管理者的设计原则

### 2.1 少即是多
- 每张图只传达**一个** insight
- 删除不必要的网格线、边框、图例（如果只有一个系列）
- 标题写 insight 而不是描述："VIP 流失率是普通客户的 3 倍" 而非 "各等级客户流失率对比"

### 2.2 注意力引导
- 用颜色高亮关键数据点，其余用灰色
- 添加注释箭头指向关键变化点
- 参考线标注均值或目标值

### 2.3 一致性
- 同一报告中所有图表使用相同的色板
- 日期格式统一
- 数值格式统一（保留 1-2 位小数）

---

## 三、标准样式配置

Agent 生成图表时使用以下统一样式：

```python
import matplotlib.pyplot as plt
import matplotlib
import seaborn as sns

# 中文字体配置（优先尝试多种中文字体）
def setup_chinese_font():
    """配置中文字体支持"""
    import platform
    system = platform.system()
    
    font_candidates = {
        'Darwin': ['PingFang SC', 'Hiragino Sans GB', 'STHeiti', 'Arial Unicode MS'],
        'Windows': ['Microsoft YaHei', 'SimHei', 'SimSun'],
        'Linux': ['WenQuanYi Micro Hei', 'Noto Sans CJK SC', 'Droid Sans Fallback']
    }
    
    candidates = font_candidates.get(system, font_candidates['Linux'])
    available_fonts = [f.name for f in matplotlib.font_manager.fontManager.ttflist]
    
    for font in candidates:
        if font in available_fonts:
            plt.rcParams['font.sans-serif'] = [font] + plt.rcParams['font.sans-serif']
            plt.rcParams['axes.unicode_minus'] = False
            return font
    
    # 回退：尝试系统默认
    plt.rcParams['axes.unicode_minus'] = False
    return None

# 报告级别样式
REPORT_STYLE = {
    'figure.figsize': (10, 6),
    'figure.dpi': 150,
    'figure.facecolor': 'white',
    'axes.facecolor': 'white',
    'axes.grid': True,
    'axes.grid.axis': 'y',
    'grid.alpha': 0.3,
    'grid.linestyle': '--',
    'axes.spines.top': False,
    'axes.spines.right': False,
    'axes.labelsize': 12,
    'axes.titlesize': 14,
    'axes.titleweight': 'bold',
    'xtick.labelsize': 10,
    'ytick.labelsize': 10,
    'legend.fontsize': 10,
    'legend.frameon': False,
}

# 应用样式
plt.rcParams.update(REPORT_STYLE)
setup_chinese_font()
sns.set_palette("husl")  # 色盲友好色板
```

---

## 四、配色方案

### 4.1 默认色板（色盲友好）

```python
# 主色板 — 分类数据（最多 6 色）
CATEGORICAL_COLORS = [
    '#4C72B0',  # 蓝
    '#DD8452',  # 橙
    '#55A868',  # 绿
    '#C44E52',  # 红
    '#8172B3',  # 紫
    '#937860',  # 棕
]

# 渐变色板 — 连续数据
SEQUENTIAL_CMAP = 'YlOrRd'      # 从黄到红
DIVERGING_CMAP = 'RdBu_r'       # 红-白-蓝（相关性矩阵）

# 二元对比
POSITIVE_COLOR = '#55A868'       # 绿色 = 好/高/增长
NEGATIVE_COLOR = '#C44E52'       # 红色 = 坏/低/下降
NEUTRAL_COLOR = '#CCCCCC'        # 灰色 = 背景/非重点
HIGHLIGHT_COLOR = '#4C72B0'      # 蓝色 = 高亮重点
```

### 4.2 暗色模式色板（Dashboard 用）

```python
DARK_THEME = {
    'bg': '#0f172a',
    'card_bg': 'rgba(255,255,255,0.05)',
    'text': '#e2e8f0',
    'text_secondary': '#94a3b8',
    'border': 'rgba(255,255,255,0.1)',
    'accent': '#3b82f6',
    'success': '#22c55e',
    'warning': '#f59e0b',
    'danger': '#ef4444',
}
```

---

## 五、常见场景的图表选择

| 分析场景 | 推荐图表组合 |
|---------|------------|
| 数据画像（Phase 1） | 缺失值热力图 + 数值分布直方图 + 相关矩阵热力图 |
| 分组对比 | 箱线图/小提琴图 + 分组条形图 |
| 时序趋势 | 折线图 + 移动平均 + 突变点标注 |
| 客户分群 | 散点图(PCA 降维) + 分群雷达图 + 分群特征条形图 |
| 模型结果 | 特征重要性条形图 + SHAP 蜂群图 + ROC 曲线 |
| 漏斗分析 | 漏斗图（水平条形图模拟） |
| 相关性发现 | 散点图 + 回归线 + 置信带 |
