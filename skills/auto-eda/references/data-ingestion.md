# 数据接入指南

> 各种格式的数据读取陷阱和最佳实践。Agent 已经知道基本的 pandas 读取语法——这里聚焦于**容易踩的坑**。

---

## 一、CSV 文件

### 关键陷阱

- **编码问题**（中文数据最常见的坑）：先用 `chardet` 检测编码再读取，不要盲猜 utf-8
  ```python
  import chardet
  with open('data.csv', 'rb') as f:
      encoding = chardet.detect(f.read(10000))['encoding']
  df = pd.read_csv('data.csv', encoding=encoding)
  ```
- **分隔符**：欧洲数据常用分号 `;`，TSV 用 `\t`。不确定时用 `sep=None, engine='python'` 自动推断
- **大文件**：超过内存时用 `chunksize=100000` 分块读取；用 `usecols` 只读必要列；用 `dtype` 指定类型减少内存

---

## 二、Excel 文件 (.xlsx)

### 关键陷阱

- **ID 列被当数字**：`dtype={'客户ID': str}` 避免 ID 丢失前导零
- **多 Sheet**：`pd.read_excel('data.xlsx', sheet_name=None)` 返回 dict，按需选择
- **日期解析**：`parse_dates=['日期列']` 自动转换
- **跳过标题行**：`skiprows=2` 跳过装饰性标题
- ⚠️ `openpyxl` 的 `data_only=True` 会**永久丢失公式**——只在确认不需要公式时使用
- ⚠️ 大文件用 `read_only=True` 模式减少内存

---

## 三、PDF 表格提取

### 方法选择

| 方法 | 依赖 | 适用场景 |
|------|------|---------|
| `tabula-py` | Java 运行时 | 大多数表格型 PDF |
| `camelot` | Ghostscript | 更精确的线框表格 |

### 关键陷阱

- ⚠️ **扫描版 PDF（图片）无法直接提取** → 需要 OCR，提醒用户
- ⚠️ 复杂合并单元格可能提取不准确 → 提取后必须人工验证
- ⚠️ `tabula-py` 需要 Java → macOS: `brew install java`

---

## 四、数据库连接

### 安全实践（比语法更重要）

- ❌ **绝不在代码中硬编码密码** → 用 `os.environ.get('DB_PASSWORD')`
- ❌ **绝不执行** `UPDATE/DELETE/DROP` → 只做 `SELECT`
- ⚠️ 大表先 `LIMIT 1000` 预览，确认后再全量拉取
- ✅ 使用 `sqlalchemy.create_engine()` 统一连接接口

---

## 五、多文件合并

### 关键陷阱

- **合并前必须检查 schema 一致性**：列名、列数、数据类型是否一致
- 列不一致时用 `pd.concat(dfs, join='outer')` 并调查差异原因
- 横向 JOIN（不同数据集关联）参考 `references/data-preparation.md` 的多表关联章节

---

## 六、数据预览清单

拿到数据后，按此清单快速了解数据：

```
1. 基本信息: df.shape, df.dtypes, 内存占用
2. 前几行: df.head()
3. 缺失值: df.isnull().sum() + 缺失率
4. 重复行: df.duplicated().sum()
5. 数值统计: df.describe()
6. 分类统计: df.describe(include='object')
7. 唯一值: df.nunique()
```

数据清洗和特征工程的后续步骤参考 `references/data-preparation.md`。
