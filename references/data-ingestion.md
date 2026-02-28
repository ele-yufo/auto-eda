# 数据接入指南

> 各种格式的数据读取最佳实践。用户可能给你 CSV、Excel、PDF、数据库连接——你都要能处理。

---

## 一、CSV 文件

### 基本读取
```python
import pandas as pd

# 自动推断
df = pd.read_csv('data.csv')

# 指定编码（中文数据常见问题）
df = pd.read_csv('data.csv', encoding='utf-8')  # 或 'gbk', 'gb2312', 'gb18030'
```

### 编码检测
```python
import chardet

with open('data.csv', 'rb') as f:
    raw = f.read(10000)  # 读取前 10000 字节足以检测
    result = chardet.detect(raw)
    encoding = result['encoding']

df = pd.read_csv('data.csv', encoding=encoding)
```

### 分隔符推断
```python
# pandas 通常能自动推断，但有时需要指定
df = pd.read_csv('data.tsv', sep='\t')           # Tab 分隔
df = pd.read_csv('data.csv', sep=';')             # 分号分隔（欧洲常见）
df = pd.read_csv('data.csv', sep=None, engine='python')  # 自动推断分隔符
```

### 大文件处理
```python
# 分块读取（内存不够时）
chunks = pd.read_csv('large.csv', chunksize=100000)
df = pd.concat(chunks, ignore_index=True)

# 只读取需要的列
df = pd.read_csv('large.csv', usecols=['id', 'amount', 'date'])

# 指定数据类型减少内存
df = pd.read_csv('large.csv', dtype={'id': str, 'amount': float})
```

---

## 二、Excel 文件 (.xlsx)

### 基本读取
```python
# 默认读第一个 sheet
df = pd.read_excel('data.xlsx')

# 读取所有 sheet（返回 dict）
all_sheets = pd.read_excel('data.xlsx', sheet_name=None)
for sheet_name, sheet_df in all_sheets.items():
    print(f"Sheet: {sheet_name}, Shape: {sheet_df.shape}")
```

### 关键参数
```python
df = pd.read_excel(
    'data.xlsx',
    sheet_name='Sheet1',        # 指定 sheet
    header=0,                    # 标题行位置（None 则无标题）
    skiprows=2,                  # 跳过前 N 行
    usecols='A:F',              # 只读 A-F 列
    dtype={'客户ID': str},       # 指定数据类型（避免 ID 被当数字）
    parse_dates=['日期列'],      # 自动解析日期
)
```

### openpyxl 陷阱
- ⚠️ `data_only=True` 打开后再保存，公式会被替换为值，**永久丢失公式**
- ⚠️ openpyxl 使用 **1-based 索引**（row=1, column=1 = A1）
- 大文件：使用 `read_only=True` 模式减少内存

```python
from openpyxl import load_workbook

# 读取计算后的值（不保留公式）
wb = load_workbook('data.xlsx', data_only=True)

# 读取公式（保留公式但值可能为 None）
wb = load_workbook('data.xlsx', data_only=False)

# 大文件只读模式
wb = load_workbook('large.xlsx', read_only=True)
```

---

## 三、PDF 表格提取

### 使用 tabula-py（推荐）
```python
# pip install tabula-py（需要 Java 运行时）
import tabula

# 提取所有页面的表格
tables = tabula.read_pdf('report.pdf', pages='all')

# 提取特定页面
table = tabula.read_pdf('report.pdf', pages='3-5')

# 如果表格跨页
table = tabula.read_pdf('report.pdf', pages='all', lattice=True)
```

### 使用 camelot（替代方案）
```python
# pip install camelot-py[cv]（需要 Ghostscript）
import camelot

tables = camelot.read_pdf('report.pdf', pages='1-3')
for table in tables:
    print(table.df)  # 转为 pandas DataFrame
```

### PDF 提取注意事项
- ⚠️ 扫描版 PDF（图片）无法直接提取 → 需要 OCR
- ⚠️ 复杂合并单元格可能提取不准确 → 需要手动验证
- ⚠️ tabula-py 需要 Java 运行时 → 安装: `brew install java` (macOS)

---

## 四、数据库连接

### SQLAlchemy 连接
```python
from sqlalchemy import create_engine
import pandas as pd

# PostgreSQL
engine = create_engine('postgresql://user:password@host:5432/dbname')

# MySQL
engine = create_engine('mysql+pymysql://user:password@host:3306/dbname')

# SQLite
engine = create_engine('sqlite:///path/to/database.db')

# 读取
df = pd.read_sql('SELECT * FROM table_name LIMIT 1000', engine)
df = pd.read_sql_table('table_name', engine)
```

### 安全实践
- ❌ 不要在代码中硬编码密码
- ✅ 使用环境变量：`os.environ.get('DB_PASSWORD')`
- ⚠️ 大表查询先加 `LIMIT` 预览，确认后再全量拉取
- ⚠️ 只做 `SELECT` 操作，**绝不执行** `UPDATE/DELETE/DROP`

---

## 五、多文件合并

### 同结构多文件
```python
import glob

files = glob.glob('data/*.csv')
dfs = [pd.read_csv(f) for f in files]
df = pd.concat(dfs, ignore_index=True)
```

### 合并前检查
```python
# 检查所有文件的列是否一致
for f in files:
    temp = pd.read_csv(f, nrows=0)
    print(f"{f}: {list(temp.columns)}")

# 如果列不一致，用 outer join
df = pd.concat(dfs, ignore_index=True, join='outer')
```

---

## 六、数据预览清单

拿到数据后，按此清单快速了解数据：

```python
# 1. 基本信息
print(f"Shape: {df.shape}")
print(f"Columns: {list(df.columns)}")
print(f"Dtypes:\n{df.dtypes}")
print(f"Memory: {df.memory_usage(deep=True).sum() / 1024**2:.1f} MB")

# 2. 前几行
df.head()

# 3. 缺失值
print(f"Missing:\n{df.isnull().sum()}")
print(f"Missing %:\n{(df.isnull().sum() / len(df) * 100).round(1)}")

# 4. 重复行
print(f"Duplicates: {df.duplicated().sum()}")

# 5. 数值列统计
df.describe()

# 6. 分类列统计
df.describe(include='object')

# 7. 唯一值
print(f"Unique Values:\n{df.nunique()}")
```
