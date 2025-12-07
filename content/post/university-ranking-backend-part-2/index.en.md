---
title: "University Ranking Backend: Code Deep Dive (Part 2)"
date: 2025-12-07
draft: false
categories: ["Building an App using vibe coding"]
tags: ["Python", "Flask", "Backend", "Code Analysis", "Data Engineering"]
weight: 3
---

In [Part 1]({{< ref "university-ranking-backend-part-1" >}}), we explored the core search logic and API structure. In this second part, we'll dig into the **Data Layer**—how data is normalized, stored, and retrieved in detail. We'll also look at the utility scripts that power the backend's intelligence.

## 1. The Data "Glue": `utils/normalize_name.py`

One of the biggest challenges in aggregating data from multiple sources (QS, US News, Niche) is that they all name universities differently. "MIT" might be "Massachusetts Institute of Technology" in one dataset and "Mass Inst of Tech" in another.

This file contains the logic to standardize names so we can link data together.

### Function: `normalize_name`

*   **Input**: `name` (string), e.g., "University of California, Berkeley (UCB)".
*   **Logic**:
    1.  **Lowercase**: Converts everything to lowercase to avoid case mismatches.
    2.  **Clean Up**: Removes text in parentheses (like abbreviations) and punctuation.
    3.  **Alias Check**: It checks a dictionary of known aliases (`ALIASES`). If the input is "mit", it automatically converts it to "massachusetts institute of technology".
*   **Output**: A clean, standardized string (e.g., "university of california berkeley").

```python
def normalize_name(name: str) -> str:
    # Lowercase
    name = name.lower()
    # Remove text in parentheses, like "(MIT)"
    name = re.sub(r'\(.*?\)', '', name)
    # Remove punctuation
    name = re.sub(r'[^\w\s]', '', name)
    # Normalize spaces
    name = re.sub(r'\s+', ' ', name)
    if name in ALIASES:
        # If the name is an alias, replace it with the full name
        name = ALIASES[name]
    return name.strip()
```

---

## 2. Detailed Profile Logic: `models/university.py`

While `models/universities.py` (plural) handles lists and searching, this file handles the deep dive into a *single* university's profile.

### Function: `get_university_by_id`

This function is interesting because it has to gather data from *many* different tables to build a complete profile.

*   **Input**: `univ_id` (integer).
*   **Logic**:
    1.  **Basic Info**: Fetches the main details (name, location, photo) from the `Universities` table.
    2.  **Translation**: Joins with `University_names_en_to_zh` to get the Chinese name.
    3.  **Dynamic Ranking Retrieval**:
        *   It queries `sqlite_master` to find *all* tables in the database that end with `_Rankings`.
        *   It loops through every single ranking table and checks if this university is listed there.
        *   This allows the system to be easily extensible—if you add a new ranking table (e.g., "Mars_University_Rankings"), this function automatically picks it up without code changes.
    4.  **Stats**: Fetches statistical data (student count, etc.) from `UniversityStats`.
*   **Output**: A massive dictionary containing everything known about that university.

```python
def get_university_by_id(univ_id):
    conn = get_db_connection()
    
    # Step 1: Get university basic info
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

    # Step 2: Get rankings from all *_Rankings tables
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
            # Skip malformed or mismatched tables
            continue
    # Step 3: Get stats from UniversityStats
    cur = conn.execute(
        "SELECT type, count, year FROM UniversityStats WHERE normalized_name = ?",
        (normalized_name,)
    )
    stats = [dict(r) for r in cur.fetchall()]
    
    conn.close()

    rankings = sorted(rankings, key= lambda x: -1 if "global" in x["subject"] or "World" in x["subject"] else x["rank_value"])

    # Combine everything
    university["rankings"] = rankings
    university["stats"] = stats
    return university
```

---

## 3. Subject Rankings: `routes/ranking_detail.py`

This route handles specific subject rankings, like "Computer Science" or "Medicine".

### Function: `ranking_detail` (`GET /subject_rankings/ranking_detail`)

*   **Input**: Query parameters `table`, `source`, and `subject`.
*   **Logic**:
    1.  Validates that all three parameters are present.
    2.  Calls `get_ranking_detail` (from `models.ranking_options`) to fetch the specific list.
    3.  This is used when a user wants to see the full list of rankings for a specific category, rather than just one university's position.
*   **Output**: A JSON list of universities ranked in that specific subject.

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

## 4. The Database Connection: `db/database.py`

A simple but essential utility.

### Function: `get_db_connection`

*   **Input**: None.
*   **Logic**:
    1.  Connects to the SQLite file defined in `config.DATABASE`.
    2.  Sets `conn.row_factory = sqlite3.Row`. This is a pro tip for Flask/SQLite developers: it allows you to access database rows like dictionaries (e.g., `row['name']`) instead of just tuples (e.g., `row[1]`).
*   **Output**: An active database connection object.

```python
def get_db_connection():
    conn = sqlite3.connect(DATABASE)
    conn.row_factory = sqlite3.Row
    return conn
```

---

## 5. Data Ingestion: `scripts/data_to_db_universities.py`

How does data get into the system? This script shows the ETL (Extract, Transform, Load) process.

### Script Logic

1.  **Extract**: Reads a raw CSV file (`data/merged_data.csv`) using Pandas.
2.  **Transform**:
    *   Selects only the columns we need (Name, Country, City, etc.).
    *   Renames columns to match our SQL schema (e.g., `Name` -> `name`).
3.  **Load**:
    *   Connects to the SQLite database.
    *   Iterates through the Pandas DataFrame.
    *   Executes an `INSERT` SQL command for every row.
4.  **Result**: Populates the `Universities` table, which serves as the backbone for the entire application.

```python
# Load the new CSV
df = pd.read_csv("data/merged_data.csv")

# Select and rename required columns
df = df[[
    "Name", "normalizedName", "Country", "Country Code", "City", "Photo", "Blurb"
]].copy()

# Rename to match SQL table
df.columns = [
    "name", "normalized_name", "country", "country_code", "city", "photo", "blurb"
]

# Connect to SQLite database
conn = sqlite3.connect("University_rankings.db")
cursor = conn.cursor()

# Insert rows
for _, row in df.iterrows():
    cursor.execute("""
        INSERT INTO Universities (
            normalized_name, name, country, country_code, city, photo, blurb
        ) VALUES (?, ?, ?, ?, ?, ?, ?)
    """, (
        row["normalized_name"], row["name"], row["country"],
        row["country_code"], row["city"], row["photo"], row["blurb"]
    ))

# Commit and close
conn.commit()
conn.close()
```

---

## 🌊 The Vibe Coding Perspective

How does Vibe Coding (AI-assisted coding) apply to data engineering and scripts? It's about using AI to quickly prototype solutions, validate assumptions, and refactor code while maintaining high quality and extensibility.

1.  **Dynamic Discovery**: 
    In `get_university_by_id`, we loop through *all* tables ending in `_Rankings`. We didn't create a complex `RankingRegistry` class or a configuration file. We just let the database tell us what it has. 
    *   **AI-Assisted Architecture**: With AI guidance, we identified this simpler approach over more complex alternatives. It's loose, flexible, and incredibly easy to extend. You don't have to "register" a new dataset; you just load it. The AI helped us avoid over-engineering.

2.  **Pragmatic Normalization**: 
    The `normalize_name` function isn't using a complex NLP model or fuzzy matching library. It uses a simple dictionary (`ALIASES`) and some regex. 
    *   **AI-Assisted Trade-offs**: The AI helped us decide to use simple regex and aliasing rather than complex NLP, recognizing that it solves 99% of the problem with 1% of the effort. When using AI-assisted coding, you focus on the specific data issues you see (like "MIT" vs "Massachusetts Institute of Technology"), let the AI handle boilerplate and common patterns, and move on to more interesting problems.

---

## Conclusion

The **University Ranking Backend** demonstrates how to build a flexible, data-driven API. By separating the "normalization" logic into utilities and using dynamic SQL queries to discover tables, the system can handle new ranking data with minimal code changes.

In the next part, we might explore how to deploy this system or how the frontend consumes this API!
