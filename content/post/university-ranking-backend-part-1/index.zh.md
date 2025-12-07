---
title: "University Ranking Backend：代码深度解析（第一部分）"
date: 2025-12-07
draft: false
categories: ["Building an App using vibe coding"]
tags: ["Python", "Flask", "Backend", "代码分析"]
weight: 2
---

本篇将深入 **University Ranking Backend** 的代码结构。我们探索应用入口和驱动大学搜索过滤系统的核心逻辑。

## 🏗️ 整体架构

后端遵循标准的 **模型-视图-控制器 (MVC)** 模式（其中"视图"是 JSON 响应）。

*   **`app.py`**：初始化应用、注册路由的入口点
*   **`routes/`**：处理 HTTP 请求、解析参数、委托给模型
*   **`models/`**：包含业务逻辑和数据库交互
*   **`db/`**：管理 SQLite 数据库连接

---

## 1. 入口点：`app.py`

这个文件是应用的心脏。它设置 Flask 服务器、配置安全，连接所有组件。

### 函数：全局作用域 / `if __name__ == "__main__":`

*   **输入**：无（脚本启动时执行）
*   **逻辑**：
    1.  **初始化**：创建 `Flask` 应用实例
    2.  **CORS 配置**：应用 `CORS(app)` 允许跨域请求。这很关键，因为前端单独托管，需要权限与此 API 通信
    3.  **蓝图注册**：注册三个主蓝图。蓝图如同"插件"为应用添加特定路由：
        *   `universities_bp`（在 `/universities`）
        *   `dropdown_bp`（在 `/dropdown`）
        *   `ranking_detail_bp`（在 `/subject_rankings`）
    4.  **启动**：在端口 10000 启动开发服务器
*   **输出**：运行中的 HTTP 服务器

```python
from flask import Flask
from routes.universities import universities_bp
from routes.dropdown import dropdown_bp
from routes.ranking_detail import ranking_detail_bp
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

# 注册蓝图
app.register_blueprint(universities_bp, url_prefix="/universities")
app.register_blueprint(dropdown_bp, url_prefix="/dropdown")
app.register_blueprint(ranking_detail_bp, url_prefix="/subject_rankings")

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=10000)
```

---

## 2. 核心逻辑：`models/universities.py`

这个文件包含搜索功能的"大脑"。处理复杂的搜索、过滤、排序逻辑。

### 函数：`filter_universities`

这是整个项目最关键的函数。根据用户输入动态构建 SQL 查询。

*   **输入**：
    *   `query`：搜索关键词（如"Harvard"）
    *   `sort_credit`：用于排序的排名表（默认："US News Global"）
    *   `country`：按国家过滤
    *   `city`：按城市过滤

*   **逻辑**：
    1.  **数据库连接**：连接到 SQLite 数据库
    2.  **动态表选择（"智能"部分）**：
        *   默认为全球排名表
        *   **智能特性**：用户选择国家时（如"China"），代码检查该国是否有专用表（如 `US_News_best global universities in china_Rankings`）。若有，自动切换到该表获更精准的本地排名
    3.  **查询构建**：
        *   从基础 SQL 开始，选择大学详情
        *   与选定排名表 `LEFT JOIN` 获取 `rank_value`
    4.  **过滤**：为搜索词、国家、城市添加 `WHERE` 子句
    5.  **排序**：按 `rank_value`（升序）排列，#1 大学最先出现

*   **输出**：字典列表，每个字典代表一所大学及其排名信息

```python
def filter_universities(query=None, sort_credit=None, country=None, city=None):
    conn = get_db_connection()
    if sort_credit is None:
        sort_credit = "US_News_best global universities_Rankings"
    
    # 智能逻辑：检查是否存在国家特定排名表
    join_table = sort_credit
    if country:
        candidate_country = country.lower().strip()
        candidate_table = f"US_News_best global universities in {candidate_country}_Rankings"
        try:
            cur = conn.execute("SELECT name FROM sqlite_master WHERE type='table' AND name=?", (candidate_table,))
            if cur.fetchone():
                join_table = candidate_table
        except Exception:
            join_table = sort_credit

    # 基础查询
    sql = """
        SELECT Universities.id, Universities.normalized_name, Universities.name,
               Universities.country, Universities.city, Universities.photo,
               R.rank_value, T.chinese_name
        FROM Universities
        LEFT JOIN University_names_en_to_zh AS T ON Universities.id = T.id
    """

    # 动态联接
    if join_table:
        sql += f"""
            LEFT JOIN "{join_table}" AS R
            ON Universities.normalized_name = R.normalized_name
        """
    
    # ... (搜索词、国家、城市的过滤逻辑) ...

    sql += " ORDER BY R.rank_value ASC NULLS LAST LIMIT 200"
    
    cursor = conn.execute(sql, params)
    return [dict(row) for row in cursor.fetchall()]
```

---

## 3. API 层：`routes/universities.py`

这个文件充当"接待员"。接收网络请求并传递给模型。

### 函数：`filter_universities_route` (`GET /universities/filter`)

*   **输入**：URL 查询参数（`query`、`sort_credit`、`country`、`city`）
*   **逻辑**：
    1.  从请求中提取这些参数
    2.  调用 `models.universities.filter_universities` 函数
    3.  将结果包装为 JSON 响应
*   **输出**：过滤后的大学 JSON 列表

```python
@universities_bp.route("/filter", methods=["GET"])
def filter_universities_route():
    query = request.args.get("query")
    sort_credit = request.args.get("sort_credit")
    country = request.args.get("country")
    city = request.args.get("city")

    results = filter_universities(query, sort_credit, country, city)
    return jsonify(results)
```

### 函数：`get_university` (`GET /universities/<int:univ_id>`)

*   **输入**：来自 URL 的 `univ_id`（整数）
*   **逻辑**：
    1.  使用 ID 获取单个大学的完整资料
    2.  检查大学是否存在
*   **输出**：大学资料 JSON 对象，若未找到则返回 404 错误

```python
@universities_bp.route("/<int:univ_id>", methods=["GET"])
def get_university(univ_id):
    from models.university import get_university_by_id
    result = get_university_by_id(univ_id)
    if result:
        return jsonify(result)
    return jsonify({"error": "University not found"}), 404
```

---

## 🌊 Vibe Coding 视角

你可能注意到这里有些有趣的模式，它们通过 **"Vibe Coding"**（AI 辅助开发）实现。AI 帮助识别这些模式并优化，实现流畅、适应性强的设计。

1.  **智能回退与自动发现**：
    在 `filter_universities` 中，我们不硬编码支持的国家或排名表列表。只是询问数据库，*"你有这张表吗？"* (`SELECT name FROM sqlite_master...`)
    *   **AI 辅助设计**：借助 AI 协助，我们快速探索并实现了动态发现模式。AI 帮助识别查询 `sqlite_master` 比管理配置文件更易维护。明天若抓取新国家数据（如"火星最佳大学"），只需将表放入 SQLite，API *自动*启用它，无需改动一行代码。

2.  **延迟导入**：
    在 `routes/universities.py` 中，你会看到导入 *在* 函数内部（如 `from models.university import get_university_by_id`）
    *   **AI 辅助模式**：AI 建议这个模式避免常困扰 Flask 应用的循环依赖。这种方法经 AI 优化后，让开发者快速迭代、重构文件、复制粘贴逻辑，不会破坏应用初始化。这是关于减少开发摩擦、提升代码库可维护性。

---

## 4. 实用路由：`routes/dropdown.py`

这些函数通过提供选项列表帮助前端构建 UI。

### 函数：`get_countries` (`GET /dropdown/countries`)

*   **输入**：无
*   **逻辑**：扫描数据库查找所有国家
*   **输出**：国家名称 JSON 列表（如 `["USA", "China", "UK", ...]`）

```python
@dropdown_bp.route('/countries', methods=['GET'])
def get_countries():
    from models.countries import get_countries_db
    countries = get_countries_db()
    return countries
```

### 函数：`get_ranking_options` (`GET /dropdown/ranking_options`)

*   **输入**：可选过滤器如 `source` 或 `subject`
*   **逻辑**：返回可用排名表的元数据。告诉前端在"排序方式"下拉菜单显示什么选项
*   **输出**：排名类别 JSON 列表

```python
@dropdown_bp.route('/ranking_options', methods=['GET'])
def get_ranking_options():
    start_time = time()
    source = request.args.get('source')
    subject = request.args.get('subject')
    from models.ranking_options import ranking_options
    tables = ranking_options(source, subject)
    end_time = time()
    duration = end_time - start_time
    print(f"Duration: {duration} seconds")
    return jsonify(tables)
```
