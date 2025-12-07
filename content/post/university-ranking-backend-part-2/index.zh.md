---
title: "University Ranking Backend：代码深度解析（第二部分）"
date: 2025-12-07
draft: false
categories: ["Building an App using vibe coding"]
tags: ["Python", "Flask", "Backend", "代码分析", "数据工程"]
weight: 3
---

在 [第一部分]({{< ref "university-ranking-backend-part-1" >}})，我们探讨了核心搜索逻辑和 API 结构。本篇深入 **数据层**——如何规范化、存储和检索数据。我们还看看驱动后端功能的实用脚本。

## 1. 数据"胶水"：`utils/normalize_name.py`

从多个来源（QS、US News、Niche）聚合数据的最大挑战是它们命名大学的方式不一。"MIT" 在一个数据集中是"Massachusetts Institute of Technology"，在另一个可能是"Mass Inst of Tech"。

这个文件包含标准化名称的逻辑，以便我们能将数据链接在一起。

### 函数：`normalize_name`

*   **输入**：`name`（字符串），如"University of California, Berkeley (UCB)"
*   **逻辑**：
    1.  **转小写**：全部转小写避免大小写不匹配
    2.  **清理**：删除括号内文本（如缩写）和标点
    3.  **别名检查**：检查已知别名字典（`ALIASES`）。若输入"mit"，自动转换为"massachusetts institute of technology"
*   **输出**：清洁、标准化的字符串（如"university of california berkeley"）

```python
def normalize_name(name: str) -> str:
    # 转小写
    name = name.lower()
    # 删除括号内文本，如"(MIT)"
    name = re.sub(r'\(.*?\)', '', name)
    # 删除标点
    name = re.sub(r'[^\w\s]', '', name)
    # 规范化空格
    name = re.sub(r'\s+', ' ', name)
    if name in ALIASES:
        # 若名称是别名，替换为全名
        name = ALIASES[name]
    return name.strip()
```

---

## 2. 详细资料逻辑：`models/university.py`

虽然 `models/universities.py`（复数）处理列表和搜索，但这个文件处理 *单一* 大学资料的深度挖掘。

### 函数：`get_university_by_id`

这个函数很有趣，因为它必须从 *许多* 不同表收集数据构建完整资料。

*   **输入**：`univ_id`（整数）
*   **逻辑**：
    1.  **基础信息**：从 `Universities` 表获取主要信息（名称、位置、照片）
    2.  **翻译**：与 `University_names_en_to_zh` 联接获中文名称
    3.  **动态排名检索**：
        *   查询 `sqlite_master` 查找数据库中所有以 `_Rankings` 结尾的表
        *   遍历每张排名表检查该大学是否在其中
        *   这使系统易于扩展——若添加新排名表（如"Mars_University_Rankings"），这函数自动使用它，无需代码改动
    4.  **统计数据**：从 `UniversityStats` 获取统计信息（学生数等）
*   **输出**：包含该大学已知所有信息的大字典

```python
def get_university_by_id(univ_id):
    conn = get_db_connection()
    
    # 第一步：获取大学基础信息
    cur = conn.execute("""
        SELECT Universities.*, T.chinese_name 
        FROM Universities 
        LEFT JOIN University_names_en_to_zh AS T ON Universities.id = T.id 
        WHERE Universities.id = ?
    """, (univ_id,))
    row = cur.fetchone()
    if not row:
        return None

    university = dict(row)
    normalized_name = university['normalized_name']

    # 第二步：从所有 *_Rankings 表获取排名
    cur = conn.execute("SELECT name FROM sqlite_master WHERE type='table' AND name LIKE '%_Rankings'")
    ranking_tables = [r["name"] for r in cur.fetchall()]
    rankings = []
    for table in ranking_tables:
        try:
            cur = conn.execute(
                f"""SELECT subject, source, rank_value 
                    FROM "{table}" WHERE normalized_name = ?""",
                (normalized_name,)
            )
            rankings += [dict(r) for r in cur.fetchall()]
        except Exception as e:
            # 跳过格式错误或不匹配的表
            continue
    # 第三步：从 UniversityStats 获取统计数据
    cur = conn.execute(
        "SELECT type, count, year FROM UniversityStats WHERE normalized_name = ?",
        (normalized_name,)
    )
    stats = [dict(r) for r in cur.fetchall()]
    
    conn.close()

    rankings = sorted(rankings, key= lambda x: -1 if "global" in x["subject"] or "World" in x["subject"] else x["rank_value"])

    # 合并所有信息
    university["rankings"] = rankings
    university["stats"] = stats
    return university
```

---

## 3. 学科排名：`routes/ranking_detail.py`

这个路由处理特定学科排名，如"计算机科学"或"医学"。

### 函数：`ranking_detail` (`GET /subject_rankings/ranking_detail`)

*   **输入**：查询参数 `table`、`source` 和 `subject`
*   **逻辑**：
    1.  验证三个参数都存在
    2.  调用 `get_ranking_detail`（来自 `models.ranking_options`）获取特定列表
    3.  用户想看特定类别完整排名列表时使用，而非仅看单所大学排名
*   **输出**：该学科排名大学的 JSON 列表

```python
@ranking_detail_bp.route('/ranking_detail', methods=['GET'])
def ranking_detail():
    table = request.args.get('table')
    source = request.args.get('source')
    subject = request.args.get('subject')
    print(f'ranking_detail {{"table": "{table}", "source": "{source}", "subject": "{subject}"}}')
    if not table or not source or not subject:
        return jsonify({'error': 'Missing parameters'}), 400
    detail = get_ranking_detail(table, source, subject)
    return jsonify(detail)
```

---

## 4. 数据库连接：`db/database.py`

简单但关键的工具函数。

### 函数：`get_db_connection`

*   **输入**：无
*   **逻辑**：
    1.  连接到 `config.DATABASE` 定义的 SQLite 文件
    2.  设置 `conn.row_factory = sqlite3.Row`。这是 Flask/SQLite 开发者的小贴士：让你像字典一样访问数据库行（如 `row['name']`）而非元组（如 `row[1]`）
*   **输出**：活跃的数据库连接对象

```python
def get_db_connection():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn
```

---

## 5. 数据导入：`scripts/data_to_db_universities.py`

数据如何进入系统？这个脚本展示 ETL（提取、转换、加载）过程。

### 脚本逻辑

1.  **提取**：使用 Pandas 读取原始 CSV 文件（`data/merged_data.csv`）
2.  **转换**：
    *   仅选择需要的列（Name、Country、City 等）
    *   重命名列以匹配 SQL 架构（如 `Name` -> `name`）
3.  **加载**：
    *   连接到 SQLite 数据库
    *   遍历 Pandas DataFrame
    *   为每行执行 `INSERT` SQL 命令
4.  **结果**：填充 `Universities` 表，这是整个应用的骨架

```python
# 加载 CSV
df = pd.read_csv("data/merged_data.csv")

# 选择并重命名必需列
df = df[[
    "Name", "normalizedName", "Country", "Country Code", "City", "Photo", "Blurb"
]].copy()

# 重命名匹配 SQL 表
df.columns = [
    "name", "normalized_name", "country", "country_code", "city", "photo", "blurb"
]

# 连接到 SQLite 数据库
conn = sqlite3.connect("University_rankings.db")
cursor = conn.cursor()

# 插入行
for _, row in df.iterrows():
    cursor.execute("""
        INSERT INTO Universities (
            normalized_name, name, country, country_code, city, photo, blurb
        ) VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        row["normalized_name"], row["name"], row["country"],
        row["country_code"], row["city"], row["photo"], row["blurb"]
    ))

# 提交并关闭
conn.commit()
conn.close()
```

---

## 🌊 Vibe Coding 视角

Vibe Coding（AI 辅助编码）如何应用于数据工程和脚本？关键是使用 AI 快速原型解决方案、验证假设，同时保持高质量和可扩展性。

1.  **动态发现**：
    在 `get_university_by_id` 中，我们遍历 *所有* 以 `_Rankings` 结尾的表。我们没创建复杂的 `RankingRegistry` 类或配置文件，只是让数据库告诉我们它有什么。
    *   **AI 辅助架构**：在 AI 指导下，我们在更复杂的替代方案中确定了这个更简单的方法。它既松散又灵活，易于扩展。无需"注册"新数据集，直接加载即可。AI 帮我们避免过度工程。

2.  **实用规范化**：
    `normalize_name` 函数不使用复杂 NLP 模型或模糊匹配库，只用简单字典（`ALIASES`）和正则表达式。
    *   **AI 辅助权衡**：AI 帮我们决定使用简单正则和别名而非复杂 NLP，认识到这以 1% 的工作量解决了 99% 的问题。用 AI 辅助编码时，你专注特定数据问题（如"MIT"vs"Massachusetts Institute of Technology"），让 AI 处理样板和常见模式，然后转向更有趣的问题。

---

## 总结

**University Ranking Backend** 展示了如何构建灵活、数据驱动的 API。通过将"规范化"逻辑分离到实用工具，并用动态 SQL 查询发现表，系统能用最少代码改动处理新排名数据。

后续可能会探讨如何部署这个系统或前端如何使用这个 API！
