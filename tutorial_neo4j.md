# Tutorial: Building a Data Fusion Knowledge Management System with Neo4j and Streamlit

This tutorial walks you through building a complete knowledge management web app using **Neo4j Aura** (a free cloud graph database) and **Streamlit** (a pure-Python UI framework). No HTML, no CSS, no JavaScript — everything is written in plain Python.

By the end you will have a running app with a dashboard, full CRUD operations, natural-language search, and a data import pipeline that loads Excel/CSV files into a graph database.

---

## Table of Contents

1. [What You Will Build](#1-what-you-will-build)
2. [Prerequisites](#2-prerequisites)
3. [Project Setup](#3-project-setup)
4. [Neo4j Aura Setup](#4-neo4j-aura-setup)
5. [Step 1 — Database Connection Module (database.py)](#step-1--database-connection-module)
6. [Step 2 — Data Import Script (import_data.py)](#step-2--data-import-script)
7. [Step 3 — The Streamlit App (app.py)](#step-3--the-streamlit-app)
8. [Running the Application](#8-running-the-application)
9. [Testing Everything](#9-testing-everything)
10. [Key Concepts Explained](#10-key-concepts-explained)
11. [Troubleshooting](#11-troubleshooting)

---

## 1. What You Will Build

A **Data Fusion Knowledge Management System** with six pages:

| Page | What It Does |
|------|-------------|
| Dashboard | Overview stats, top fields of study, top publishers, recent papers |
| Papers | Browse, search, filter by field, view details, add/edit/delete |
| Methods | Browse, search, view details with uncertainty types, add/edit/delete |
| Datasets | Browse, search, view details, add/edit/delete |
| Contributors | See all contributors with contribution counts and detail view |
| Search | Natural-language search across all three entity types at once |

**Why Neo4j?** The data has natural graph relationships — Papers link to Methods and Datasets via shared DOI, and all entities link back to Contributors. Neo4j stores these as first-class graph edges, making relationship queries fast and intuitive.

**Why Streamlit?** You write `st.title("Hello")` and it renders a web page. No templates, no routes, no HTML. Perfect for students new to web development.

---

## 2. Prerequisites

- **Python 3.10+** (check with `python3 --version`)
- A free **Neo4j Aura** account (Section 4 walks you through it)
- Your `.xlsx` and `.csv` data files in a folder
- Any code editor (VS Code recommended)

---

## 3. Project Setup

Create the project folder:

```bash
mkdir neo4j_app
cd neo4j_app
```

Create `requirements.txt`:

```
streamlit
neo4j
pandas
openpyxl
```

Install everything:

```bash
pip install -r requirements.txt
```

**Package summary:**

| Package | Purpose |
|---------|---------|
| `streamlit` | Turns Python scripts into interactive web apps |
| `neo4j` | Official Python driver for Neo4j — runs Cypher queries |
| `pandas` | Reads Excel and CSV files |
| `openpyxl` | Engine that pandas uses to open `.xlsx` files |

Your final folder structure:

```
neo4j_app/
├── app.py           # The entire Streamlit UI (one file)
├── database.py      # Neo4j connection and query helpers
├── import_data.py   # Reads Excel/CSV and loads them into Neo4j
└── requirements.txt
```

Place your `.xlsx` / `.csv` data files **one level up** (in the parent directory). The import script scans `../` automatically.

---

## 4. Neo4j Aura Setup

Neo4j Aura is a free cloud-hosted Neo4j service — no local installation needed.

### 4.1 Create an Account

1. Go to [https://neo4j.com/cloud/aura/](https://neo4j.com/cloud/aura/)
2. Click **"Start Free"** and sign up
3. After logging in, click **"New Instance"**
4. Choose **AuraDB Free**
5. Name it (e.g. `CIS360GVSU`) and click **"Create"**

### 4.2 Save Your Credentials

Immediately after creation, Neo4j shows you a credentials file. **Download it or copy it now** — the password is only shown once.

It looks like this:

```
NEO4J_URI=neo4j+s://xxxxxxxx.databases.neo4j.io
NEO4J_USERNAME=xxxxxxxx
NEO4J_PASSWORD=xxxxxxxxxxxxxxxxxxxxxxxxxxxx
NEO4J_DATABASE=xxxxxxxx
```

> **Important:** Wait about 60 seconds for the instance to finish starting before connecting.

### 4.3 Check It Is Running

Go to [https://console.neo4j.io](https://console.neo4j.io). Your instance should show a green **"Running"** status.

---

## Step 1 — Database Connection Module

Create `database.py`. This file handles everything related to connecting to Neo4j and running queries.

```python
"""
Neo4j connection helper.
The driver is cached so Streamlit reuses one connection across reruns.
"""

import streamlit as st
from neo4j import GraphDatabase

NEO4J_URI      = "neo4j+s://406508c5.databases.neo4j.io"   # replace with yours
NEO4J_USER     = "406508c5"                                  # replace with yours
NEO4J_PASSWORD = "ltBLw-nl5IyRewPm6sjmvcKCfRx2POWsA4b_AfuglKc"  # replace with yours
NEO4J_DATABASE = "406508c5"                                  # replace with yours


@st.cache_resource
def get_driver():
    """Return a cached Neo4j driver (created once per Streamlit session)."""
    return GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))


def run_query(query, params=None):
    """Run a Cypher query and return a list of record dicts."""
    driver = get_driver()
    with driver.session(database=NEO4J_DATABASE) as session:
        result = session.run(query, params or {})
        return [dict(record) for record in result]


def run_write(query, params=None):
    """Run a write Cypher query (CREATE / MERGE / SET / DELETE)."""
    driver = get_driver()
    with driver.session(database=NEO4J_DATABASE) as session:
        session.execute_write(lambda tx: tx.run(query, params or {}))
```

> **Replace the four credential values** with the ones from your Neo4j Aura credentials file.

### How It Works

**`@st.cache_resource`** — This decorator tells Streamlit to create the driver only once and reuse it for every page interaction. Without this, a new database connection would be made on every button click, which is very slow.

**`run_query(query, params)`** — Opens a session, runs a Cypher `MATCH` query, and returns results as a Python list of dictionaries. Example:

```python
rows = run_query("MATCH (p:Paper) RETURN p.title AS title LIMIT 5")
# rows = [{"title": "..."}, {"title": "..."}, ...]
```

**`run_write(query, params)`** — Wraps a write query in an explicit transaction using `execute_write`. This ensures the write either fully succeeds or is rolled back if something goes wrong.

**Parameters (`params`)** — Always pass user input as parameters, never as string concatenation. This prevents Cypher injection:

```python
# Safe:
run_query("MATCH (p:Paper) WHERE p.doi = $doi RETURN p", {"doi": user_input})

# Never do this:
run_query(f"MATCH (p:Paper) WHERE p.doi = '{user_input}' RETURN p")
```

---

## Step 2 — Data Import Script

Create `import_data.py`. This script reads all `.xlsx` and `.csv` files from the parent directory and loads them into Neo4j as a graph.

### 2.1 The Graph Model

Before writing any code, understand what the graph will look like:

```
(:Contributor {name, source_file})
       |
       | [:CONTRIBUTED]
       ↓
   (:Paper {doi, title, author, publication_title, publication_date,
            url, keywords, abstract, publisher, field_of_study,
            is_data_fusion, classification_reason})
       |
       |-- [:HAS_METHOD]  → (:Method {name, method_key, doi, description, u1, u3})
       |
       └-- [:HAS_DATASET] → (:Dataset {doi, data_name, dataset_url, method_key,
                                        data_type, collection_method, u2,
                                        spatial_coverage, temporal_coverage,
                                        format, license, provenance})
```

Every node type is called a **label** in Neo4j. Edges between nodes are called **relationships**. This is different from SQL where you need join tables.

### 2.2 Imports and Constants

```python
#!/usr/bin/env python3
import os
import re
import pandas as pd
from pathlib import Path
from neo4j import GraphDatabase

NEO4J_URI      = "neo4j+s://406508c5.databases.neo4j.io"
NEO4J_USER     = "406508c5"
NEO4J_PASSWORD = "ltBLw-nl5IyRewPm6sjmvcKCfRx2POWsA4b_AfuglKc"
NEO4J_DATABASE = "406508c5"

DATA_DIR = os.path.join(os.path.dirname(__file__), "..")
```

`DATA_DIR` points one level up from `neo4j_app/` — that is where the Excel files live.

### 2.3 Helper Functions

```python
def clean(val):
    """Convert a spreadsheet cell to a clean string, or None if empty."""
    if val is None or (isinstance(val, float) and pd.isna(val)):
        return None
    s = str(val).strip().replace('\xa0', ' ').replace('\n', ' ').strip()
    return s if s else None


def contributor_name(filename):
    """Extract a human-readable contributor name from the filename."""
    name = Path(filename).stem
    for suffix in [' Data Fusion', ' Data', ' Dataset Midterm Part 2', 'DataFusionKg']:
        name = name.replace(suffix, '')
    name = name.replace('_', ' ').strip()
    if name and name[0].islower():
        name = name.title()
    return name


def normalize_col(col):
    """Normalize a column header to lowercase with no special characters."""
    if col is None:
        return ''
    col = str(col).strip().lower().replace('\xa0', '').replace(' ', '_')
    return re.sub(r'[^a-z0-9_]', '', col)
```

These are identical to the MongoDB version — the spreadsheet cleaning logic does not depend on the database.

### 2.4 Column Mapping

Each contributor named their columns differently. These dictionaries map every known variant to a standard field name:

```python
PAPER_COL_MAP = {
    'doi': 'doi', 'title': 'title', 'paper_title': 'title',
    'author': 'author', 'authors': 'author', 'name_author': 'author',
    'publication_title': 'publication_title', 'publication': 'publication_title',
    'publicationdate': 'publication_date', 'publication_date': 'publication_date',
    'url': 'url', 'keywords': 'keywords',
    'abstract': 'abstract', 'publisher': 'publisher',
    'field_of_study': 'field_of_study', 'fieldofstudy': 'field_of_study',
    'isdatafusionpaper': 'is_data_fusion', 'is_datafusion_paper': 'is_data_fusion',
    'datafusionclassificationreason': 'classification_reason',
    'datafusionclassification': 'classification_reason',
    'reason': 'classification_reason',
}

METHOD_COL_MAP = {
    'method_name': 'name', 'methodname': 'name', 'name': 'name',
    'method_key': 'method_key', 'methodkey': 'method_key',
    'doi': 'doi', 'doi_method_block': 'doi',
    'description': 'description',
    'u1': 'u1', 'u3': 'u3',
    'uncertainty_type_u1u3': 'u1',
    'uncertainty_type_u1': 'u1', 'uncertainty_type_u3': 'u3',
}

DATASET_COL_MAP = {
    'doi': 'doi', 'doi_dataset_block': 'doi',
    'data_name': 'data_name', 'dataname': 'data_name', 'dataset_name': 'data_name',
    'dataseturl': 'dataset_url', 'dataset_url': 'dataset_url',
    'method_key': 'method_key', 'methodkey': 'method_key',
    'method_key_dataset': 'method_key',
    'data_type': 'data_type', 'datatype': 'data_type',
    'collection_method': 'collection_method', 'collectionmethod': 'collection_method',
    'u2': 'u2', 'u2_dataset': 'u2', 'uncertainty_type_u2': 'u2',
    'spatialcoverage': 'spatial_coverage', 'spatial_coverage': 'spatial_coverage',
    'temporalcoverage': 'temporal_coverage', 'temporal_coverage': 'temporal_coverage',
    'format': 'format', 'license': 'license', 'provenance': 'provenance',
}


def map_columns(df, col_map):
    """Return a rename dict for a DataFrame's columns using the given map."""
    result = {}
    for col in df.columns:
        norm = normalize_col(col)
        if norm in col_map:
            result[col] = col_map[norm]
    return result
```

### 2.5 Neo4j Write Functions

Each function is designed to be passed to `session.execute_write()`, which wraps it in a transaction:

```python
def upsert_contributor(tx, name, source_file):
    tx.run(
        "MERGE (c:Contributor {source_file: $source_file}) "
        "ON CREATE SET c.name = $name",
        source_file=source_file, name=name,
    )
```

`MERGE` is like "insert if not exists". If a `Contributor` with this `source_file` already exists, nothing happens. If it does not exist, it is created.

```python
def insert_paper(tx, row, source_file):
    tx.run("""
        CREATE (p:Paper {
            doi: $doi, title: $title, author: $author,
            publication_title: $publication_title,
            publication_date: $publication_date,
            url: $url, keywords: $keywords, abstract: $abstract,
            publisher: $publisher, field_of_study: $field_of_study,
            is_data_fusion: $is_data_fusion,
            classification_reason: $classification_reason
        })
        WITH p
        MATCH (c:Contributor {source_file: $source_file})
        CREATE (c)-[:CONTRIBUTED]->(p)
    """, source_file=source_file, **row)
```

This creates the `Paper` node and immediately creates the `CONTRIBUTED` relationship to its contributor — all in one Cypher statement. `WITH p` passes the newly created paper node to the next clause.

```python
def insert_method(tx, row, source_file):
    tx.run("""
        CREATE (m:Method {
            name: $name, method_key: $method_key, doi: $doi,
            description: $description, u1: $u1, u3: $u3
        })
        WITH m
        MATCH (c:Contributor {source_file: $source_file})
        CREATE (c)-[:CONTRIBUTED]->(m)
    """, source_file=source_file, **row)


def insert_dataset(tx, row, source_file):
    tx.run("""
        CREATE (d:Dataset {
            doi: $doi, data_name: $data_name, dataset_url: $dataset_url,
            method_key: $method_key, data_type: $data_type,
            collection_method: $collection_method, u2: $u2,
            spatial_coverage: $spatial_coverage,
            temporal_coverage: $temporal_coverage,
            format: $format, license: $license, provenance: $provenance
        })
        WITH d
        MATCH (c:Contributor {source_file: $source_file})
        CREATE (c)-[:CONTRIBUTED]->(d)
    """, source_file=source_file, **row)
```

### 2.6 Linking Papers to Methods and Datasets

After all nodes are created, a second pass creates `HAS_METHOD` and `HAS_DATASET` edges wherever papers and methods/datasets share the same DOI:

```python
def link_by_doi(session):
    session.run("""
        MATCH (p:Paper), (m:Method)
        WHERE p.doi IS NOT NULL AND m.doi IS NOT NULL AND p.doi = m.doi
        MERGE (p)-[:HAS_METHOD]->(m)
    """)
    session.run("""
        MATCH (p:Paper), (d:Dataset)
        WHERE p.doi IS NOT NULL AND d.doi IS NOT NULL AND p.doi = d.doi
        MERGE (p)-[:HAS_DATASET]->(d)
    """)
```

`MERGE` on a relationship means "create this edge only if it does not already exist", preventing duplicates.

### 2.7 The Main Function

```python
def main():
    driver = GraphDatabase.driver(NEO4J_URI, auth=(NEO4J_USER, NEO4J_PASSWORD))

    with driver.session(database=NEO4J_DATABASE) as session:
        # Start fresh
        print("Clearing existing graph...")
        session.run("MATCH (n) DETACH DELETE n")

        files = sorted(os.listdir(DATA_DIR))
        excel_files = [f for f in files if f.endswith('.xlsx')]
        csv_files   = [f for f in files if f.endswith('.csv')]

        for filename in excel_files + csv_files:
            filepath = os.path.join(DATA_DIR, filename)
            name = contributor_name(filename)
            session.execute_write(upsert_contributor, name, filename)

            try:
                if filename.endswith('.csv'):
                    p, m, d = import_single_sheet(filepath, session, filename, is_csv=True)
                else:
                    xls = pd.ExcelFile(filepath)
                    if len(xls.sheet_names) >= 2:
                        p, m, d = import_three_sheet(filepath, session, filename)
                    else:
                        p, m, d = import_single_sheet(filepath, session, filename)

                print(f"  {filename}: {p} papers, {m} methods, {d} datasets")
            except Exception as e:
                print(f"  ERROR {filename}: {e}")

        link_by_doi(session)

        # Create indexes for faster lookups
        session.run("CREATE INDEX paper_doi   IF NOT EXISTS FOR (p:Paper)       ON (p.doi)")
        session.run("CREATE INDEX method_doi  IF NOT EXISTS FOR (m:Method)      ON (m.doi)")
        session.run("CREATE INDEX dataset_doi IF NOT EXISTS FOR (d:Dataset)     ON (d.doi)")
        session.run("CREATE INDEX contrib_file IF NOT EXISTS FOR (c:Contributor) ON (c.source_file)")

    driver.close()


if __name__ == '__main__':
    main()
```

`MATCH (n) DETACH DELETE n` deletes every node and every relationship in the database — a fresh start. `DETACH DELETE` is needed because Neo4j will not let you delete a node that still has relationships attached to it.

**Run the import:**

```bash
python3 import_data.py
```

Expected output:

```
Clearing existing graph...
  Aiden Mack.xlsx: 5 papers, 5 methods, 5 datasets
  Alexander Thompson.xlsx: 5 papers, 5 methods, 5 datasets
  ...
Creating DOI relationships...
Creating indexes...
Done: 299 papers, 313 methods, 526 datasets
```

---

## Step 3 — The Streamlit App

Create `app.py`. This is the complete UI — all six pages in a single Python file.

### 3.1 Page Config and Navigation

```python
import re
import streamlit as st
from database import run_query, run_write

st.set_page_config(
    page_title="Data Fusion KMS",
    page_icon="🔬",
    layout="wide",
    initial_sidebar_state="expanded",
)

PAGES = ["Dashboard", "Papers", "Methods", "Datasets", "Contributors", "Search"]
page = st.sidebar.selectbox("Navigate", PAGES)
```

`st.set_page_config()` sets the browser tab title and icon.  
`st.sidebar.selectbox()` creates a dropdown in the left sidebar — selecting an option reruns the script with `page` set to the chosen value.

### 3.2 Search Helper

```python
STOP_WORDS = {
    "a","an","the","is","are","was","were","in","on","at","to","for","of",
    "and","or","not","do","has","have","will","can","it","we","you",
    "show","find","get","list","give","used","using","about",
    # ... full list in source file
}

def build_regex(text):
    """Convert a query string to a case-insensitive Neo4j regex pattern."""
    words = text.split()
    keywords = [w.strip(".,!?;:\"'") for w in words
                if w.lower().strip(".,!?;:\"'") not in STOP_WORDS]
    if not keywords:
        keywords = words
    # Short queries → literal match
    if len(keywords) <= 2 and len(words) <= 2:
        return re.escape(text)
    # Long / natural-language → match any keyword
    return "|".join(re.escape(k) for k in keywords if k)
```

This lets users type `"Show me all fusion methods used for Traffic Data."` and get meaningful results. The function strips stop words and joins the remaining keywords with `|` (regex OR).

The regex is then used in Cypher like this:

```python
rgx = f"(?i).*({build_regex(q)}).*"
# (?i) = case-insensitive flag
# .* = match anything before/after the keyword
```

### 3.3 Reusable UI Helpers

```python
def metric_row(papers, methods, datasets, contributors):
    c1, c2, c3, c4 = st.columns(4)
    c1.metric("📄 Papers",       papers)
    c2.metric("⚙️ Methods",      methods)
    c3.metric("🗄️ Datasets",    datasets)
    c4.metric("👥 Contributors", contributors)


def show_table(rows, columns=None):
    """Display a list of dicts as a styled dataframe."""
    if not rows:
        st.info("No records found.")
        return
    import pandas as pd
    df = pd.DataFrame(rows)
    if columns:
        df = df[[c for c in columns if c in df.columns]]
    st.dataframe(df, use_container_width=True, hide_index=True)


def paginate(items, per_page=25, key="page"):
    """Show a page number input and return the current page slice."""
    total = len(items)
    if total == 0:
        return []
    total_pages = max(1, (total + per_page - 1) // per_page)
    if total_pages > 1:
        page_num = st.number_input(
            f"Page (1–{total_pages})", min_value=1,
            max_value=total_pages, value=1, step=1, key=key,
        )
    else:
        page_num = 1
    start = (page_num - 1) * per_page
    st.caption(f"Showing {start + 1}–{min(start + per_page, total)} of {total}")
    return items[start:start + per_page]
```

`st.columns(4)` creates four equal-width columns side by side.  
`st.metric()` renders a large number with a label — perfect for dashboard stats.  
`st.dataframe()` renders a pandas DataFrame as an interactive sortable table.

### 3.4 Dashboard Page

```python
if page == "Dashboard":
    st.title("🔬 Data Fusion KMS — Dashboard")

    counts = run_query("""
        RETURN
          count { MATCH (p:Paper)       RETURN p } AS papers,
          count { MATCH (m:Method)      RETURN m } AS methods,
          count { MATCH (d:Dataset)     RETURN d } AS datasets,
          count { MATCH (c:Contributor) RETURN c } AS contributors
    """)
    c = counts[0]
    metric_row(c["papers"], c["methods"], c["datasets"], c["contributors"])
```

The Cypher `count { ... }` subquery syntax counts matching nodes inline — equivalent to four separate `COUNT(*)` queries in SQL but in one round trip.

```python
    top_fields = run_query("""
        MATCH (p:Paper)
        WHERE p.field_of_study IS NOT NULL AND p.field_of_study <> ''
        RETURN p.field_of_study AS field, count(p) AS cnt
        ORDER BY cnt DESC LIMIT 10
    """)
```

This is the equivalent of SQL `GROUP BY field_of_study ORDER BY COUNT(*) DESC LIMIT 10`.

### 3.5 Papers Page — List and Search

```python
elif page == "Papers":
    st.title("📄 Papers")

    col1, col2 = st.columns([3, 1])
    with col1:
        q = st.text_input("Search (title, author, DOI, abstract, keywords)")
    with col2:
        fields_raw = run_query("""
            MATCH (p:Paper)
            WHERE p.field_of_study IS NOT NULL AND p.field_of_study <> ''
            RETURN DISTINCT p.field_of_study AS f ORDER BY f
        """)
        field_opts = ["All Fields"] + [r["f"] for r in fields_raw]
        field_sel = st.selectbox("Field of Study", field_opts)
```

`st.columns([3, 1])` creates two columns where the first is 3× wider than the second — gives the search bar more space.

Building the query dynamically:

```python
    filters = []
    params  = {}
    if q:
        rgx = build_regex(q)
        filters.append(
            "(p.title =~ $rgx OR p.author =~ $rgx OR p.doi =~ $rgx "
            " OR p.abstract =~ $rgx OR p.keywords =~ $rgx)"
        )
        params["rgx"] = f"(?i).*({rgx}).*"
    if field_sel != "All Fields":
        filters.append("p.field_of_study = $field")
        params["field"] = field_sel

    where = ("WHERE " + " AND ".join(filters)) if filters else ""

    rows = run_query(f"""
        MATCH (c:Contributor)-[:CONTRIBUTED]->(p:Paper)
        {where}
        RETURN elementId(p) AS eid, p.title AS title, p.author AS author,
               p.doi AS doi, p.field_of_study AS field_of_study,
               c.name AS contributor
        ORDER BY id(p) DESC
    """, params)
```

`=~` is Neo4j's regex match operator. `elementId(p)` returns a unique string identifier for the node — we use this to select individual papers for the detail/edit view.

### 3.6 Papers Page — Detail, Edit, Delete

```python
    if rows:
        options = {f"{r['title'] or r['doi'] or 'Untitled'} [{r['eid'][-6:]}]": r["eid"]
                   for r in rows[:200]}
        selected_label = st.selectbox("Select a paper", list(options.keys()))
        eid = options[selected_label]

        detail = run_query(
            "MATCH (c:Contributor)-[:CONTRIBUTED]->(p:Paper) WHERE elementId(p)=$eid "
            "RETURN p, c.name AS contributor_name",
            {"eid": eid}
        )
```

The selectbox shows paper titles with the last 6 characters of their element ID in brackets to distinguish papers with identical titles.

The edit form uses `st.form()`:

```python
        with st.expander("✏️ Edit this Paper"):
            with st.form("edit_paper_form"):
                title = st.text_input("Title", value=p.get('title') or "")
                # ... more fields ...
                submitted = st.form_submit_button("💾 Update Paper")
                if submitted:
                    run_write("""
                        MATCH (p:Paper) WHERE elementId(p) = $eid
                        SET p.title = $title, p.author = $author, ...
                    """, {"eid": eid, "title": title or None, ...})
                    st.success("Paper updated!")
                    st.rerun()
```

`st.expander()` creates a collapsible section — keeps the page clean.  
`st.form()` groups widgets and only submits when the button is clicked.  
`st.rerun()` refreshes the page after a successful write, so the updated data is immediately visible.

Delete is a simple button outside the form:

```python
        if st.button("🗑️ Delete this Paper"):
            run_write(
                "MATCH (p:Paper) WHERE elementId(p)=$eid DETACH DELETE p",
                {"eid": eid}
            )
            st.success("Paper deleted.")
            st.rerun()
```

`DETACH DELETE` removes the node and all its relationships at once.

### 3.7 Papers Page — Add New

```python
    st.subheader("➕ Add a New Paper")
    with st.form("add_paper_form"):
        doi   = st.text_input("DOI")
        title = st.text_input("Title")
        # ... more fields ...
        if st.form_submit_button("➕ Add Paper"):
            run_write("""
                CREATE (p:Paper {doi:$doi, title:$title, ...})
            """, {"doi": doi or None, "title": title or None, ...})
            st.success(f"Paper '{title}' added!")
            st.rerun()
```

Empty string inputs are converted to `None` (`doi or None`) so Neo4j stores them as null rather than as empty strings.

### 3.8 Methods and Datasets Pages

These pages follow the exact same pattern as Papers:

1. Search bar → build Cypher WHERE clause → run query
2. Show paginated table
3. Selectbox for detail view with expanders for edit/delete
4. Add form at the bottom

The only differences are the field names and which fields are searched. See `app.py` for the full code.

### 3.9 Contributors Page

```python
elif page == "Contributors":
    rows = run_query("""
        MATCH (c:Contributor)
        RETURN c.name AS name, c.source_file AS source_file,
          count { MATCH (c)-[:CONTRIBUTED]->(p:Paper) }   AS papers,
          count { MATCH (c)-[:CONTRIBUTED]->(m:Method) }  AS methods,
          count { MATCH (c)-[:CONTRIBUTED]->(d:Dataset) } AS datasets
        ORDER BY c.name
    """)
```

This uses Neo4j's inline `count {}` subqueries to compute contribution counts for each contributor in a single query — no joins, no subqueries in separate round trips.

The detail section uses graph traversal:

```python
    c_papers = run_query(
        "MATCH (c:Contributor {source_file:$sf})-[:CONTRIBUTED]->(p:Paper) "
        "RETURN p.title AS title, p.doi AS doi, p.field_of_study AS field",
        {"sf": src_file}
    )
```

This is where Neo4j really shines — following the `CONTRIBUTED` relationship to find all papers by a specific contributor is a single graph traversal, not a JOIN.

### 3.10 Search Page

```python
elif page == "Search":
    q = st.text_input(
        "Search across papers, methods, and datasets",
        placeholder="e.g. Show me all fusion methods used for Traffic Data."
    )

    if q:
        rgx = f"(?i).*({build_regex(q)}).*"
        params = {"rgx": rgx}

        papers = run_query("""
            MATCH (c:Contributor)-[:CONTRIBUTED]->(p:Paper)
            WHERE p.title =~ $rgx OR p.author =~ $rgx OR p.doi =~ $rgx
               OR p.abstract =~ $rgx OR p.keywords =~ $rgx
            RETURN p.title AS title, p.author AS author,
                   p.doi AS doi, c.name AS contributor
            LIMIT 50
        """, params)

        methods = run_query("""
            MATCH (c:Contributor)-[:CONTRIBUTED]->(m:Method)
            WHERE m.name =~ $rgx OR m.description =~ $rgx OR m.doi =~ $rgx
            RETURN m.name AS name, m.doi AS doi,
                   m.description AS description, c.name AS contributor
            LIMIT 50
        """, params)

        datasets = run_query("""
            MATCH (c:Contributor)-[:CONTRIBUTED]->(d:Dataset)
            WHERE d.data_name =~ $rgx OR d.data_type =~ $rgx
               OR d.doi =~ $rgx OR d.collection_method =~ $rgx
            RETURN d.data_name AS data_name, d.data_type AS data_type,
                   d.doi AS doi, c.name AS contributor
            LIMIT 50
        """, params)
```

Three parallel queries, each searching relevant fields. Results are displayed in separate sections with `st.subheader()`.

---

## 8. Running the Application

### Step 1 — Suppress the Streamlit Email Prompt

Do this once so Streamlit does not ask for your email every time:

```bash
mkdir -p ~/.streamlit
cat > ~/.streamlit/credentials.toml << 'EOF'
[general]
email = ""
EOF
```

### Step 2 — Import Data

```bash
cd neo4j_app
python3 import_data.py
```

Wait for it to finish. You should see all contributor files listed with counts, followed by "Done: 299 papers, 313 methods, 526 datasets".

### Step 3 — Start the App

```bash
streamlit run app.py
```

Streamlit will print:

```
  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
```

Open [http://localhost:8501](http://localhost:8501) in your browser.

### Step 4 — Stop the App

Press `Ctrl+C` in the terminal.

---

## 9. Testing Everything

Work through this checklist:

- [ ] **Dashboard** — counts match (299 papers, 313 methods, 526 datasets, 33 contributors)
- [ ] **Papers** — list loads, pagination works (next/previous pages)
- [ ] **Papers search** — type "fusion" and see filtered results
- [ ] **Papers field filter** — select a field from the dropdown
- [ ] **Paper detail** — select a paper, see full details expand
- [ ] **Paper edit** — change a field, click Update, see it reflected
- [ ] **Paper delete** — delete a paper, verify it disappears
- [ ] **Add paper** — fill the form, submit, see it in the list
- [ ] **Methods** — repeat search/detail/edit/delete/add steps
- [ ] **Datasets** — repeat search/detail/edit/delete/add steps
- [ ] **Contributors** — list loads with counts; select one and see their contributions
- [ ] **Search — short query** — type "Kalman", see relevant methods
- [ ] **Search — long query** — type "Show me all fusion methods used for Traffic Data."
- [ ] **Search — no results** — type a nonsense string, see "No results found"

---

## 10. Key Concepts Explained

### Graph Database vs Relational Database

| Concept | SQL (SQLite) | Neo4j |
|---------|-------------|-------|
| Table | `CREATE TABLE papers (...)` | Node label `:Paper` |
| Row | `(1, 'doi', 'title', ...)` | `(:Paper {doi: ..., title: ...})` |
| Foreign key | `contributor_id INTEGER` | Relationship `[:CONTRIBUTED]` |
| Join | `LEFT JOIN contributors ON contributor_id = id` | `MATCH (c)-[:CONTRIBUTED]->(p)` |
| Index | `CREATE INDEX ON papers(doi)` | `CREATE INDEX FOR (p:Paper) ON (p.doi)` |
| Count | `SELECT COUNT(*) FROM papers` | `count { MATCH (p:Paper) RETURN p }` |
| Group by | `GROUP BY field ORDER BY cnt DESC` | `RETURN field, count(p) AS cnt ORDER BY cnt DESC` |
| Filter | `WHERE title LIKE '%fusion%'` | `WHERE p.title =~ '(?i).*fusion.*'` |
| Update | `UPDATE papers SET title=? WHERE id=?` | `MATCH (p) WHERE elementId(p)=$eid SET p.title=$title` |
| Delete | `DELETE FROM papers WHERE id=?` | `MATCH (p) WHERE elementId(p)=$eid DETACH DELETE p` |

### Cypher Query Language Basics

Cypher uses ASCII art to describe graph patterns:

```cypher
-- Match all papers and their contributors
MATCH (c:Contributor)-[:CONTRIBUTED]->(p:Paper)
RETURN c.name, p.title

-- Find all methods linked to a specific paper via DOI
MATCH (p:Paper {doi: "10.1234/example"})-[:HAS_METHOD]->(m:Method)
RETURN m.name, m.description

-- Create a new node
CREATE (p:Paper {title: "New Paper", doi: "10.5678/new"})

-- Update properties
MATCH (p:Paper) WHERE elementId(p) = $eid
SET p.title = $new_title

-- Delete a node and all its relationships
MATCH (p:Paper) WHERE elementId(p) = $eid
DETACH DELETE p
```

### Streamlit Widget Reference

| Widget | Code | What It Shows |
|--------|------|---------------|
| Title | `st.title("Hello")` | Large heading |
| Text input | `st.text_input("Label")` | Single-line text box |
| Text area | `st.text_area("Label")` | Multi-line text box |
| Selectbox | `st.selectbox("Label", options)` | Dropdown menu |
| Button | `st.button("Click me")` | Clickable button |
| Form | `st.form("key")` | Groups widgets, submits together |
| Columns | `st.columns(3)` | Side-by-side layout |
| Expander | `st.expander("Title")` | Collapsible section |
| Metric | `st.metric("Label", value)` | Big number card |
| Dataframe | `st.dataframe(df)` | Interactive table |
| Success | `st.success("Done!")` | Green alert box |
| Error | `st.error("Failed.")` | Red alert box |
| Info | `st.info("Note.")` | Blue alert box |
| Rerun | `st.rerun()` | Refresh the page |

### How Streamlit Handles State

Every time a user interacts with a widget (clicks a button, types in a text box, changes a dropdown), Streamlit **reruns the entire script from top to bottom**. This means:

- You do NOT need to write event handlers or callbacks
- Every variable is recalculated on every interaction
- `@st.cache_resource` prevents the database driver from being recreated on every rerun
- `st.form()` prevents reruns until the submit button is clicked (important for forms with many fields)

---

## 11. Troubleshooting

| Problem | Solution |
|---------|----------|
| `ServiceUnavailable: Failed to establish connection` | Your Neo4j Aura instance may still be starting up — wait 60 seconds and try again. Also check Network Access in the Neo4j console. |
| `AuthError: The client is unauthorized` | Double-check your `NEO4J_USER` and `NEO4J_PASSWORD` in `database.py` and `import_data.py`. The password is only shown once when you create the instance. |
| Import is very slow | Each row is inserted in its own transaction. This is safe but slow for large datasets. Normal for 500–1000 rows. |
| `streamlit: command not found` | Run `pip install streamlit` or make sure your Python environment is active. |
| Streamlit asks for an email every time | Run the one-time setup: `mkdir -p ~/.streamlit && echo '[general]\nemail = ""' > ~/.streamlit/credentials.toml` |
| Page is blank or keeps loading | Open the terminal where you ran `streamlit run` and check for error messages. |
| `KeyError` in the app | A query returned a field name that the template did not expect. Check `show_table()` column lists match the `RETURN` aliases in the query. |
| Changes made in the app do not persist | You are connected to the wrong database. Confirm `NEO4J_DATABASE` in `database.py` matches your Aura instance ID. |
| `DETACH DELETE` error | You tried to delete a node that has relationships without using `DETACH`. Always use `DETACH DELETE` in this app. |
