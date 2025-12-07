---
title: "Building a University Ranking Backend with Vibe Coding"
date: 2025-12-07
draft: false
categories: ["Building an App using vibe coding"]
tags: ["Python", "Flask", "Backend", "API", "SQLite"]
weight: 3
---

In this post, I'll share the journey of building a **University Ranking Backend**, a RESTful API designed to aggregate and serve university ranking data from prestigious sources like QS and US News. We'll explore the "vibe coding" strategy used to build it, the technical methodology, and the core algorithms that make it tick.

## 📚 Read the Full Series

This project is too interesting to cover in just one post! I've broken down the technical details into a deep-dive series:

*   **[Part 1: The Core Logic & API Structure]({{< ref "university-ranking-backend-part-1" >}})**
    *   Explores the `app.py` entry point.
    *   Explains the "Smart Table Selection" algorithm in `models/universities.py`.
    *   Walks through the API routes.
*   **[Part 2: Data Engineering & Scripts]({{< ref "university-ranking-backend-part-2" >}})**
    *   Details the ETL process (Extract, Transform, Load).
    *   Shows how we normalize university names across different datasets.
    *   Explains the dynamic ranking discovery logic.

## 🚀 Project Introduction

The **University Ranking Backend** is a centralized service that provides detailed information about universities worldwide. 

**Key Features:**
*   **Aggregated Rankings**: Combines data from multiple sources (QS, US News, Niche).
*   **Multilingual Support**: Built-in support for English and Chinese university names.
*   **Smart Filtering**: Allows users to search by name, country, city, and specific ranking criteria.

The goal was to create a lightweight, queryable interface that frontend applications could easily consume without worrying about the complex logic of merging different ranking datasets.

## 💡 Vibe Coding Strategy

"Vibe coding" is an AI-assisted coding approach where developers work with AI tools to accelerate development, using the AI as a collaborative partner to handle repetitive tasks, generate boilerplate code, and explore solutions faster. For this project, the strategy was **Data-First, Logic-Second** with heavy AI assistance.

1.  **Data Collection & "Vibe Check"**: 
    *   We started by gathering raw data (CSVs, JSONs) in the `data/` folder. 
    *   The "vibe check" here was ensuring the data *felt* right—consistent names, correct rankings, and clean formatting.
2.  **Scripting the Flow**: 
    *   Instead of jumping straight into the API, we wrote standalone scripts (`scripts/`) to process this raw data and populate a SQLite database. This separated the messy data cleaning from the clean application logic.
3.  **API as a Gateway**: 
    *   Once the database was ready, the Flask API was built as a simple gateway. The focus was on making the endpoints intuitive (`/filter`, `/search`) rather than over-engineering the architecture.

Check out the [Part 1]({{< ref "university-ranking-backend-part-1" >}}) and [Part 2]({{< ref "university-ranking-backend-part-2" >}}) posts for specific examples of how "Vibe Coding" (AI-assisted development) influenced the code structure, like using lazy imports and dynamic table discovery. These patterns were refined with AI assistance to avoid common pitfalls and optimize for extensibility.

## 🛠 Methodology

We chose a lightweight tech stack to keep the development fast and the deployment simple.

*   **Language**: **Python 3.8+** - The go-to for data-heavy backend work.
*   **Framework**: **Flask** - Minimalist and flexible. Unlike Django, it doesn't force a structure, allowing us to build exactly what we needed.
*   **Database**: **SQLite3** - Serverless and file-based. Perfect for read-heavy, write-rare applications like this.
*   **Extensions**: `flask_cors` to handle Cross-Origin Resource Sharing, essential for a public API.

### Project Structure
The project follows a clean, modular structure:
*   `app.py`: Entry point.
*   `routes/`: API Controllers (e.g., `universities.py`).
*   `models/`: Data Layer (e.g., SQL query construction).
*   `data/` & `scripts/`: Raw data and processing pipelines.

## 📂 Key Functions Explained

To keep the backend organized, we split the logic into different files. Here’s a simple introduction to the key functions that make the system work:

### 1. The Entry Point (`app.py`)
*   **`create_app`**: This is the starting line. It initializes the Flask application, sets up security rules (CORS) so the frontend can talk to it safely, and plugs in all the different API routes.

### 2. The API Routes (`routes/`)
These functions act as the "receptionists," handling incoming HTTP requests from users.
*   **`filter_universities_route`** (mapped to `/filter`): The main search engine. It listens for user inputs—like a search term, a selected country, or a city—and passes them to the database logic.
*   **`get_university`** (mapped to `/<id>`): Fetches the full profile for a single university. This is used when a user clicks on a specific result to see more details.
*   **`get_countries`** (in `dropdown.py`): A helper function that scans the database to list all available countries, populating the dropdown menu on the frontend so users know what they can filter by.

### 3. The Data Logic (`models/`)
This is where the heavy lifting happens.
*   **`filter_universities`**: The "brain" of the operation. It dynamically builds a SQL query to join university data with the correct ranking table based on the user's choice.

## 🧮 The Algorithm: Dynamic SQL Construction

The core intelligence of this backend isn't a complex machine learning model, but a smart **Dynamic SQL Query Builder**.

The challenge: Users might want to rank universities globally, or they might want to see how they rank specifically within a country (e.g., "Best Global Universities in China").

The solution: **Smart Table Selection**.

The code dynamically checks if a specific ranking table exists for the requested country. If it does, it seamlessly switches the `JOIN` operation to that specific table for more accurate local rankings.

### Key Code Snippet

Here is the `filter_universities` function from `models/universities.py` that handles this logic. For a line-by-line breakdown, see [Part 1]({{< ref "university-ranking-backend-part-1" >}}).

```python
def filter_universities(query=None, sort_credit=None, country=None, city=None):
    conn = get_db_connection()
    # Default ranking source
    if sort_credit is None:
        sort_credit = "US_News_best global universities_Rankings"
    
    join_table = sort_credit
    
    # Smart Logic: Check for country-specific ranking table
    if country:
        candidate_country = country.lower().strip()
        candidate_table = f"US_News_best global universities in {candidate_country}_Rankings"
        # ... (code checks if candidate_table exists in DB) ...
        # If exists, switch to country-specific table
        join_table = candidate_table

    # Base Query
    sql = """
        SELECT Universities.id, Universities.normalized_name, Universities.name,
               Universities.country, Universities.city, Universities.photo,
               R.rank_value, T.chinese_name
        FROM Universities
        LEFT JOIN University_names_en_to_zh AS T ON Universities.id = T.id
    """

    # Dynamic Join
    if join_table:
        sql += f"""
            LEFT JOIN "{join_table}" AS R
            ON Universities.normalized_name = R.normalized_name
        """
    
    # ... (Filtering logic for query, country, city) ...

    sql += " ORDER BY R.rank_value ASC NULLS LAST LIMIT 200"
    
    cursor = conn.execute(sql, params)
    return [dict(row) for row in cursor.fetchall()]
```

## 🔗 Links
*   **GitHub Repository**: [xiaruize0911/University_Ranking_Backend](https://github.com/xiaruize0911/University_Ranking_Backend)
