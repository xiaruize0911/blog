---
title: "University Ranking Backend: Code Deep Dive (Part 1)"
date: 2025-12-07
draft: false
categories: ["Building an App using vibe coding"]
tags: ["Python", "Flask", "Backend", "Code Analysis"]
weight: 2
---

In this post, we will dive deep into the code structure of the **University Ranking Backend**. We'll explore the entry point of the application and the core logic that powers the university search and filtering system.

## 🏗️ High-Level Overview

The backend follows a standard **Model-View-Controller (MVC)** pattern (where the "View" is JSON output).

*   **`app.py`**: The entry point that initializes the app and registers routes.
*   **`routes/`**: Handles HTTP requests, parses parameters, and delegates logic to models.
*   **`models/`**: Contains the business logic and database interactions.
*   **`db/`**: Manages the SQLite database connection.

---

## 1. The Entry Point: `app.py`

This file is the heart of the application. It sets up the Flask server, configures security, and connects all the moving parts.

### Function: Global Scope / `if __name__ == "__main__":`

*   **Input**: None (executed on script start).
*   **Logic**:
    1.  **Initialization**: Creates the `Flask` application instance.
    2.  **CORS Setup**: Applies `CORS(app)` to allow cross-origin requests. This is crucial because our frontend is hosted separately and needs permission to talk to this API.
    3.  **Blueprint Registration**: It registers three main blueprints. Think of blueprints as "plugins" that add specific routes to the app:
        *   `universities_bp` (at `/universities`)
        *   `dropdown_bp` (at `/dropdown`)
        *   `ranking_detail_bp` (at `/subject_rankings`)
    4.  **Execution**: Starts the development server on port 10000.
*   **Output**: A running HTTP server.

```python
from flask import Flask
from routes.universities import universities_bp
from routes.dropdown import dropdown_bp
from routes.ranking_detail import ranking_detail_bp
from flask_cors import CORS

app = Flask(__name__)
CORS(app)

# Register blueprints
app.register_blueprint(universities_bp, url_prefix="/universities")
app.register_blueprint(dropdown_bp, url_prefix="/dropdown")
app.register_blueprint(ranking_detail_bp, url_prefix="/subject_rankings")

if __name__ == "__main__":
    app.run(host='0.0.0.0', port=10000)
```

---

## 2. The Core Logic: `models/universities.py`

This file contains the "brain" of the search feature. It handles the complex logic of searching, filtering, and sorting universities.

### Function: `filter_universities`

This is the most important function in the entire project. It dynamically builds SQL queries based on user input.

*   **Input**: 
    *   `query`: Search keyword (e.g., "Harvard").
    *   `sort_credit`: The ranking table to sort by (default: "US News Global").
    *   `country`: Filter by country.
    *   `city`: Filter by city.

*   **Logic**:
    1.  **Database Connection**: Connects to the SQLite database.
    2.  **Dynamic Table Selection (The "Smart" Part)**: 
        *   It defaults to the global ranking table.
        *   **Smart Feature**: If a user selects a country (e.g., "China"), the code checks if a specific table exists for that country (e.g., `US_News_best global universities in china_Rankings`). If it does, it automatically switches to that table for more accurate local rankings.
    3.  **Query Construction**:
        *   It starts with a base SQL query selecting university details.
        *   It performs a `LEFT JOIN` with the selected ranking table to get the `rank_value`.
    4.  **Filtering**: Appends `WHERE` clauses for the search query, country, and city.
    5.  **Sorting**: Sorts results by `rank_value` (ASC) so the #1 university appears first.

*   **Output**: A list of dictionaries, where each dictionary represents a university with its rank and details.

```python
def filter_universities(query=None, sort_credit=None, country=None, city=None):
    conn = get_db_connection()
    if sort_credit is None:
        sort_credit = "US_News_best global universities_Rankings"
    
    # Smart Logic: Check for country-specific ranking table
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

---

## 3. The API Layer: `routes/universities.py`

This file acts as the "receptionist." It receives requests from the internet and passes them to the models.

### Function: `filter_universities_route` (`GET /universities/filter`)

*   **Input**: URL Query parameters (`query`, `sort_credit`, `country`, `city`).
*   **Logic**:
    1.  Extracts these parameters from the request.
    2.  Calls the `models.universities.filter_universities` function we described above.
    3.  Wraps the result in a JSON response.
*   **Output**: A JSON list of filtered universities.

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

### Function: `get_university` (`GET /universities/<int:univ_id>`)

*   **Input**: `univ_id` (integer) from the URL.
*   **Logic**:
    1.  Fetches the full profile for a single university using its ID.
    2.  Checks if the university exists.
*   **Output**: A JSON object of the university profile, or a 404 error if not found.

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

## 🌊 The Vibe Coding Perspective

You might notice some interesting patterns here that were developed using **"Vibe Coding"**—an AI-assisted development approach. The AI helped identify these patterns and optimize them for flow, adaptability, and extensibility.

1.  **Smart Fallbacks & Auto-Discovery**: 
    In `filter_universities`, we don't hardcode a list of supported countries or ranking tables. We simply ask the database, *"Do you have a table for this?"* (`SELECT name FROM sqlite_master...`). 
    *   **AI-Assisted Design**: With AI assistance, we were able to quickly explore and implement this dynamic discovery pattern. The AI helped identify that querying `sqlite_master` was more maintainable than managing a configuration file. If I scrape a new country's data tomorrow (e.g., "Best Universities in Mars"), I just drop the table into SQLite. The API *automatically* starts using it without a single line of code change.

2.  **Lazy Imports**: 
    In `routes/universities.py`, you'll see imports *inside* the functions (e.g., `from models.university import get_university_by_id`). 
    *   **AI-Assisted Pattern**: The AI suggested this pattern to avoid circular dependency headaches that often plague Flask applications. This approach, refined with AI assistance, lets developers move fast, refactor files, and copy-paste logic without breaking the app initialization. It's about removing friction during development and making the codebase more maintainable.


---

## 4. Utility Routes: `routes/dropdown.py`

These functions help the frontend build its UI by providing lists of options.

### Function: `get_countries` (`GET /dropdown/countries`)

*   **Input**: None.
*   **Logic**: Scans the database to find all unique countries listed.
*   **Output**: A JSON list of country names (e.g., `["USA", "China", "UK", ...]`).

```python
@dropdown_bp.route('/countries', methods=['GET'])
def get_countries():
    from models.countries import get_countries_db
    countries = get_countries_db()
    return countries
```

### Function: `get_ranking_options` (`GET /dropdown/ranking_options`)

*   **Input**: Optional filters like `source` or `subject`.
*   **Logic**: Returns metadata about the available ranking tables. This tells the frontend what options to show in the "Sort By" dropdown.
*   **Output**: A JSON list of ranking categories.

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
