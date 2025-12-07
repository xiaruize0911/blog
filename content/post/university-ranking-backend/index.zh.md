---
title: "使用 Vibe Coding 构建大学排名后端"
date: 2025-12-07
draft: false
categories: ["Building an App using vibe coding"]
tags: ["Python", "Flask", "Backend", "API", "SQLite"]
weight: 3
---

在这篇文章中，我将分享构建 **University Ranking Backend** 的过程。这是一个 RESTful API，用于聚合和提供来自 QS 和 US News 等权威机构的大学排名数据。我们将探讨 Vibe Coding 策略、技术架构和驱动系统的核心算法。

## 📚 完整系列阅读

这个项目内容丰富，我将其分为深入系列文章：

*   **[第一部分：核心逻辑与 API 架构]({{< ref "university-ranking-backend-part-1" >}})**
    *   探索 `app.py` 应用入口
    *   解析 `models/universities.py` 中的"智能表选择"算法
    *   讲解 API 路由的实现
*   **[第二部分：数据工程与脚本]({{< ref "university-ranking-backend-part-2" >}})**
    *   介绍 ETL 流程（提取、转换、加载）
    *   展示如何在不同数据集间标准化大学名称
    *   阐述动态排名发现机制

## 🚀 项目介绍

**University Ranking Backend** 是一个集中式服务，为全球大学提供详细信息。

**核心功能：**
*   **数据聚合**：整合来自多个来源（QS、US News、Niche）的排名数据
*   **多语言支持**：内置英文和中文大学名称支持
*   **智能过滤**：支持按名称、国家、城市和排名标准进行搜索

目标是创建一个轻量级、易查询的接口，让前端应用能轻松使用，无需关心合并不同排名数据的复杂逻辑。

## 💡 Vibe Coding 策略

"Vibe Coding" 是一种 AI 辅助编码方法，开发者与 AI 工具协作加速开发，AI 充当协作伙伴处理重复任务、生成模板代码，并快速探索解决方案。本项目采用 **数据驱动、逻辑其次** 的策略，并得到 AI 的充分协助。

1.  **数据收集与质量检查**：
    *   首先在 `data/` 目录收集原始数据（CSV、JSON 格式）
    *   通过"质量检查"确保数据统一一致、排名准确、格式规范
2.  **脚本化处理流程**：
    *   编写独立脚本（`scripts/` 目录）处理原始数据和填充 SQLite 数据库
    *   这样做能将混乱的数据清洗与干净的应用逻辑分离
3.  **API 作为网关**：
    *   数据库就绪后，Flask API 作为简单网关构建
    *   重点是让端点易用直观（`/filter`、`/search`），而非过度设计

查看 [第一部分]({{< ref "university-ranking-backend-part-1" >}}) 和 [第二部分]({{< ref "university-ranking-backend-part-2" >}}) 了解 "Vibe Coding"（AI 辅助开发）如何影响代码结构的具体例子，比如延迟导入和动态表发现。这些模式都通过 AI 协助优化，避免常见陷阱并提升可扩展性。

## 🛠 技术方案

我们选择了轻量级技术栈，既保证开发速度，又保持部署简洁。

*   **编程语言**：**Python 3.8+** - 数据密集型后端的首选
*   **Web 框架**：**Flask** - 极简且灵活，不同于 Django 的强制性结构，让我们能构建完全所需的系统
*   **数据库**：**SQLite3** - 无服务器、基于文件的数据库，完美适合读多写少的应用
*   **扩展库**：`flask_cors` 处理跨域请求，对公开 API 至关重要

### 项目结构

项目采用清晰、模块化的组织方式：
*   `app.py` - 应用入口
*   `routes/` - API 控制器（如 `universities.py`）
*   `models/` - 数据层（如 SQL 查询构建）
*   `data/` & `scripts/` - 原始数据和处理管道

## 📂 关键函数简介

为保持后端条理清晰，我们将逻辑分散在不同文件中。以下是使系统运转的关键函数：

### 1. 应用入口 (`app.py`)
*   **`create_app`**：这是起点。初始化 Flask 应用，配置安全规则（CORS）使前端能安全通信，加载所有 API 路由。

### 2. API 路由 (`routes/`)
这些函数如同"接待员"，处理用户的 HTTP 请求。
*   **`filter_universities_route`**（映射到 `/filter`）：主搜索引擎。监听用户输入（搜索词、国家、城市）并将其传递给数据库逻辑。
*   **`get_university`**（映射到 `/<id>`）：获取单个大学的完整资料。用户点击结果查看详情时调用。
*   **`get_countries`**（在 `dropdown.py` 中）：辅助函数扫描数据库列出所有国家，填充前端下拉菜单，让用户知道可以按哪些条件过滤。

### 3. 数据逻辑 (`models/`)
这是繁重工作发生的地方。
*   **`filter_universities`**："大脑"所在。动态构建 SQL 查询，根据用户选择将大学数据与对应排名表联接。

## 🧮 算法：动态 SQL 构建

这个后端的核心不是复杂的机器学习模型，而是聪明的 **动态 SQL 查询构建器**。

**挑战**：用户既想看全球排名，也想看特定国家的排名（如"中国的全球顶级大学"）。

**解决方案**：**智能表选择**。

代码动态检查请求国家是否有对应排名表。若有，就无缝切换 `JOIN` 操作到该表，获得更精准的本地排名。

### 关键代码片段

这是来自 `models/universities.py` 的 `filter_universities` 函数，处理此逻辑。详细行级讲解见 [第一部分]({{< ref "university-ranking-backend-part-1" >}})。

```python
def filter_universities(query=None, sort_credit=None, country=None, city=None):
    conn = get_db_connection()
    # 默认排名数据源
    if sort_credit is None:
        sort_credit = "US_News_best global universities_Rankings"
    
    join_table = sort_credit
    
    # 智能逻辑：检查是否存在国家特定的排名表
    if country:
        candidate_country = country.lower().strip()
        candidate_table = f"US_News_best global universities in {candidate_country}_Rankings"
        # ... (检查该表是否在数据库中存在) ...
        # 若存在，切换到国家特定表
        join_table = candidate_table

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

## 🔗 相关链接
*   **GitHub 仓库**: [xiaruize0911/University_Ranking_Backend](https://github.com/xiaruize0911/University_Ranking_Backend)
