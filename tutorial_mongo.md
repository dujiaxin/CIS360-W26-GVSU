# Tutorial: Building a Data Fusion Knowledge Management System with FastAPI and MongoDB

This step-by-step tutorial walks you through building a full-stack web application that manages academic research data using **FastAPI** (Python web framework) and **MongoDB Atlas** (cloud database). By the end you will have a working app with a dashboard, CRUD operations, search, pagination, and a data import pipeline.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Project Setup](#2-project-setup)
3. [MongoDB Atlas Setup](#3-mongodb-atlas-setup)
4. [Step 1 -- Database Connection Module](#step-1----database-connection-module)
5. [Step 2 -- Pydantic Models](#step-2----pydantic-models)
6. [Step 3 -- Data Import Script (ETL)](#step-3----data-import-script-etl)
7. [Step 4 -- FastAPI Application (main.py)](#step-4----fastapi-application-mainpy)
8. [Step 5 -- Jinja2 HTML Templates](#step-5----jinja2-html-templates)
9. [Step 6 -- Running the Application](#step-6----running-the-application)
10. [Step 7 -- Testing Everything](#step-7----testing-everything)
11. [Key Concepts Explained](#key-concepts-explained)
12. [Troubleshooting](#troubleshooting)

---

## 1. Prerequisites

Before you begin, make sure you have:

- **Python 3.10+** installed (check with `python3 --version`)
- **pip** package manager
- A **MongoDB Atlas** account (free tier works -- see Section 3)
- Your data files (`.xlsx` and `.csv`) in a parent directory
- A code editor (VS Code recommended)

---

## 2. Project Setup

Create the project directory and install dependencies:

```bash
mkdir mongodb_app
cd mongodb_app
```

Create a file called `requirements.txt` with these contents:

```
fastapi
uvicorn[standard]
motor
pymongo[srv]
pandas
jinja2
python-multipart
openpyxl
itsdangerous
```

Install everything:

```bash
pip install -r requirements.txt
```

**What each package does:**

| Package | Purpose |
|---------|---------|
| `fastapi` | Web framework (handles routes, requests, responses) |
| `uvicorn` | ASGI server that runs your FastAPI app |
| `motor` | Async MongoDB driver (used by FastAPI for non-blocking DB calls) |
| `pymongo[srv]` | Sync MongoDB driver (used by the import script) |
| `pandas` | Reads Excel/CSV files during data import |
| `jinja2` | Template engine for rendering HTML pages |
| `python-multipart` | Parses HTML form submissions |
| `openpyxl` | Reads `.xlsx` Excel files (used by pandas) |
| `itsdangerous` | Session cookie signing (used by session middleware) |

Create the templates directory:

```bash
mkdir templates
```

Your final project structure will look like this:

```
mongodb_app/
├── main.py              # FastAPI application with all routes
├── database.py          # MongoDB connection setup
├── models.py            # Pydantic data models
├── import_data.py       # Script to import Excel/CSV data into MongoDB
├── requirements.txt     # Python dependencies
└── templates/           # HTML templates (14 files)
    ├── base.html
    ├── dashboard.html
    ├── papers.html
    ├── paper_detail.html
    ├── edit_paper.html
    ├── methods.html
    ├── method_detail.html
    ├── edit_method.html
    ├── datasets.html
    ├── dataset_detail.html
    ├── edit_dataset.html
    ├── contributors.html
    ├── contributor_detail.html
    └── search.html
```

---

## 3. MongoDB Atlas Setup

MongoDB Atlas is a free cloud-hosted MongoDB service. Follow these steps to create your database:

### 3.1 Create an Account

1. Go to [https://www.mongodb.com/cloud/atlas](https://www.mongodb.com/cloud/atlas)
2. Click **"Try Free"** and create an account
3. Choose the **M0 Free Tier** cluster

### 3.2 Create a Cluster

1. Select a cloud provider (AWS, GCP, or Azure) and a region close to you
2. Name your cluster (e.g., `CIS360GVSU`)
3. Click **"Create Cluster"** (takes 1-3 minutes)

### 3.3 Set Up Database Access

1. Go to **Database Access** in the left sidebar
2. Click **"Add New Database User"**
3. Choose **Password** authentication
4. Set a username (e.g., `mongoadmin`) and password (e.g., `mongo985865`)
5. Under **Database User Privileges**, select **"Read and write to any database"**
6. Click **"Add User"**

### 3.4 Set Up Network Access

1. Go to **Network Access** in the left sidebar
2. Click **"Add IP Address"**
3. Click **"Allow Access from Anywhere"** (adds `0.0.0.0/0`) -- this is fine for development
4. Click **"Confirm"**

### 3.5 Get Your Connection String

1. Go to **Database** in the left sidebar
2. Click **"Connect"** on your cluster
3. Choose **"Drivers"**
4. Copy the connection string. It will look like:

```
mongodb+srv://mongoadmin:<password>@yourcluster.xxxxx.mongodb.net/?appName=YourApp
```

5. Replace `<password>` with the password you created in step 3.3

**Save this connection string** -- you will use it in `database.py` and `import_data.py`.

---

## Step 1 -- Database Connection Module

Create `database.py`. This module manages the async MongoDB connection used by FastAPI.

```python
"""MongoDB connection via Motor (async driver) for FastAPI."""

from motor.motor_asyncio import AsyncIOMotorClient

MONGO_URI = (
    "mongodb+srv://mongoadmin:mongo985865"
    "@cis360gvsu.yfiz4js.mongodb.net/?appName=CIS360GVSU"
)
DB_NAME = "data_fusion_kms"

client: AsyncIOMotorClient = None


async def connect_db():
    global client
    client = AsyncIOMotorClient(MONGO_URI)


async def close_db():
    global client
    if client:
        client.close()


def get_db():
    return client[DB_NAME]
```

**How this works:**

- `MONGO_URI` -- your Atlas connection string (replace with your own)
- `DB_NAME` -- the database name. MongoDB creates it automatically on first write.
- `connect_db()` / `close_db()` -- called when the FastAPI app starts/stops
- `get_db()` -- returns the database object for querying collections
- **Motor** is the async version of PyMongo. FastAPI is async, so we need an async driver to avoid blocking the event loop.

> **Important:** Replace the `MONGO_URI` value with your own connection string from Section 3.5.

---

## Step 2 -- Pydantic Models

Create `models.py`. These models define the shape of data for form validation.

```python
"""Pydantic models for form validation."""

from typing import Optional
from pydantic import BaseModel


class PaperForm(BaseModel):
    doi: Optional[str] = None
    title: Optional[str] = None
    author: Optional[str] = None
    publication_title: Optional[str] = None
    publication_date: Optional[str] = None
    url: Optional[str] = None
    keywords: Optional[str] = None
    abstract: Optional[str] = None
    publisher: Optional[str] = None
    field_of_study: Optional[str] = None
    is_data_fusion: Optional[str] = None
    classification_reason: Optional[str] = None


class MethodForm(BaseModel):
    name: str
    method_key: Optional[str] = None
    doi: Optional[str] = None
    description: Optional[str] = None
    u1: Optional[str] = None
    u3: Optional[str] = None


class DatasetForm(BaseModel):
    data_name: str
    doi: Optional[str] = None
    dataset_url: Optional[str] = None
    method_key: Optional[str] = None
    data_type: Optional[str] = None
    collection_method: Optional[str] = None
    u2: Optional[str] = None
    spatial_coverage: Optional[str] = None
    temporal_coverage: Optional[str] = None
    format: Optional[str] = None
    license: Optional[str] = None
    provenance: Optional[str] = None
```

**Key points:**

- `Optional[str] = None` means the field is not required and defaults to `None`
- `name: str` on `MethodForm` means the method name is **required**
- These models mirror the document structure stored in MongoDB

---

## Step 3 -- Data Import Script (ETL)

Create `import_data.py`. This script reads Excel/CSV files from the parent directory and loads them into MongoDB. Place your `.xlsx` and `.csv` data files in the **parent directory** (one level up from `mongodb_app/`).

```python
#!/usr/bin/env python3
"""Import all Excel/CSV files into MongoDB Atlas."""

import os
import re
import pandas as pd
from pathlib import Path
from pymongo import MongoClient

MONGO_URI = (
    "mongodb+srv://mongoadmin:mongo985865"
    "@cis360gvsu.yfiz4js.mongodb.net/?appName=CIS360GVSU"
)
DB_NAME = "data_fusion_kms"
DATA_DIR = os.path.join(os.path.dirname(__file__), "..")
```

> **Replace `MONGO_URI`** with your own connection string.

### 3.1 Helper Functions

Add these helper functions after the constants:

```python
def clean(val):
    """Clean a cell value to string or None."""
    if val is None or (isinstance(val, float) and pd.isna(val)):
        return None
    s = str(val).strip()
    s = s.replace('\xa0', ' ').replace('\n', ' ').strip()
    return s if s else None


def contributor_name(filename):
    """Derive contributor name from filename."""
    name = Path(filename).stem
    for suffix in [' Data Fusion', ' Data', ' Dataset Midterm Part 2', 'DataFusionKg']:
        name = name.replace(suffix, '')
    name = name.replace('_', ' ').strip()
    if name and name[0].islower():
        name = name.title()
    return name


def normalize_col(col):
    """Normalize column name to a standard form."""
    if col is None:
        return ''
    col = str(col).strip().lower().replace('\xa0', '').replace(' ', '_')
    col = re.sub(r'[^a-z0-9_]', '', col)
    return col
```

**What these do:**

- `clean()` -- converts cell values to clean strings or `None`. Handles NaN, whitespace, non-breaking spaces.
- `contributor_name()` -- extracts a human-readable name from the filename (e.g., `"Brandon Minor.xlsx"` becomes `"Brandon Minor"`)
- `normalize_col()` -- normalizes column headers so `"Publication Date"`, `"publication_date"`, and `"PublicationDate"` all become `"publication_date"`

### 3.2 Column Mapping Dictionaries

Different contributors use different column names. These maps translate them to a standard schema:

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
    result = {}
    for col in df.columns:
        norm = normalize_col(col)
        if norm in col_map:
            result[col] = col_map[norm]
    return result
```

### 3.3 Sheet Detection Functions

Excel workbooks may have multiple sheets. These functions figure out which sheet contains papers, methods, or datasets:

```python
def find_sheet(wb_sheets, candidates):
    for sheet in wb_sheets:
        norm = sheet.strip().lower().replace(' ', '_').replace('_', '')
        for c in candidates:
            if c in norm:
                return sheet
    return None


def detect_sheet_type(xls, sheet_name):
    df = pd.read_excel(xls, sheet_name=sheet_name, nrows=0)
    cols_norm = set(normalize_col(c) for c in df.columns)
    if 'method_name' in cols_norm or 'methodname' in cols_norm:
        return 'method'
    if 'method_key' in cols_norm and 'u1' in cols_norm:
        return 'method'
    if 'data_name' in cols_norm or 'dataname' in cols_norm:
        return 'dataset'
    if 'dataseturl' in cols_norm or 'dataset_url' in cols_norm:
        return 'dataset'
    if 'data_type' in cols_norm or 'collection_method' in cols_norm:
        return 'dataset'
    if 'doi' in cols_norm and ('title' in cols_norm or 'abstract' in cols_norm):
        return 'paper'
    if 'title' in cols_norm and 'author' in cols_norm:
        return 'paper'
    return None
```

### 3.4 Import Functions

Two import strategies -- one for multi-sheet workbooks, one for single-sheet/CSV files:

```python
def import_three_sheet(filepath, db, contributor_id):
    """Import a multi-sheet Excel workbook."""
    xls = pd.ExcelFile(filepath)
    sheets = xls.sheet_names
    paper_count = method_count = dataset_count = 0

    # Try to match sheets by name first
    paper_sheet = find_sheet(sheets, ['doi', 'paper', 'summary', 'exam'])
    method_sheet = find_sheet(sheets, ['fusion', 'method'])
    data_sheet = find_sheet(sheets, ['data', 'dataset'])

    # Fall back to header detection for unmatched sheets
    assigned = {paper_sheet, method_sheet, data_sheet}
    for s in sheets:
        if s in assigned:
            continue
        stype = detect_sheet_type(xls, s)
        if stype == 'paper' and not paper_sheet:
            paper_sheet = s
        elif stype == 'method' and not method_sheet:
            method_sheet = s
        elif stype == 'dataset' and not data_sheet:
            data_sheet = s

    # Import papers
    if paper_sheet:
        df = pd.read_excel(xls, sheet_name=paper_sheet)
        col_renames = map_columns(df, PAPER_COL_MAP)
        df = df.rename(columns=col_renames)
        docs = []
        for _, row in df.iterrows():
            doi = clean(row.get('doi') or row.get('title'))
            title = clean(row.get('title'))
            if not doi and not title:
                continue
            docs.append({
                "doi": clean(row.get('doi')), "title": title,
                "author": clean(row.get('author')),
                "publication_title": clean(row.get('publication_title')),
                "publication_date": clean(row.get('publication_date')),
                "url": clean(row.get('url')), "keywords": clean(row.get('keywords')),
                "abstract": clean(row.get('abstract')),
                "publisher": clean(row.get('publisher')),
                "field_of_study": clean(row.get('field_of_study')),
                "is_data_fusion": clean(row.get('is_data_fusion')),
                "classification_reason": clean(row.get('classification_reason')),
                "contributor_id": contributor_id,
            })
        if docs:
            db.papers.insert_many(docs)
            paper_count = len(docs)

    # Import methods
    if method_sheet:
        df = pd.read_excel(xls, sheet_name=method_sheet)
        col_renames = map_columns(df, METHOD_COL_MAP)
        df = df.rename(columns=col_renames)
        docs = []
        for _, row in df.iterrows():
            name = clean(row.get('name'))
            if not name:
                continue
            docs.append({
                "name": name, "method_key": clean(row.get('method_key')),
                "doi": clean(row.get('doi')),
                "description": clean(row.get('description')),
                "u1": clean(row.get('u1')), "u3": clean(row.get('u3')),
                "contributor_id": contributor_id,
            })
        if docs:
            db.methods.insert_many(docs)
            method_count = len(docs)

    # Import datasets
    if data_sheet and data_sheet != paper_sheet and data_sheet != method_sheet:
        df = pd.read_excel(xls, sheet_name=data_sheet)
        col_renames = map_columns(df, DATASET_COL_MAP)
        df = df.rename(columns=col_renames)
        docs = []
        for _, row in df.iterrows():
            data_name = clean(row.get('data_name'))
            doi = clean(row.get('doi'))
            if not data_name and not doi:
                continue
            docs.append({
                "doi": doi, "data_name": data_name,
                "dataset_url": clean(row.get('dataset_url')),
                "method_key": clean(row.get('method_key')),
                "data_type": clean(row.get('data_type')),
                "collection_method": clean(row.get('collection_method')),
                "u2": clean(row.get('u2')),
                "spatial_coverage": clean(row.get('spatial_coverage')),
                "temporal_coverage": clean(row.get('temporal_coverage')),
                "format": clean(row.get('format')),
                "license": clean(row.get('license')),
                "provenance": clean(row.get('provenance')),
                "contributor_id": contributor_id,
            })
        if docs:
            db.datasets.insert_many(docs)
            dataset_count = len(docs)

    return paper_count, method_count, dataset_count


def import_single_sheet(filepath, db, contributor_id, is_csv=False):
    """Import a single-sheet Excel or CSV file."""
    if is_csv:
        df = pd.read_csv(filepath, encoding='latin-1')
    else:
        df = pd.read_excel(filepath)

    # Map all three column types at once
    paper_renames = map_columns(df, PAPER_COL_MAP)
    method_renames = map_columns(df, METHOD_COL_MAP)
    dataset_renames = map_columns(df, DATASET_COL_MAP)

    all_renames = {}
    all_renames.update(paper_renames)
    all_renames.update(method_renames)
    all_renames.update(dataset_renames)
    df = df.rename(columns=all_renames)

    paper_docs = []
    method_docs = []
    dataset_docs = []

    for _, row in df.iterrows():
        doi = clean(row.get('doi'))
        title = clean(row.get('title'))
        if not doi and not title:
            continue

        paper_docs.append({
            "doi": doi, "title": title,
            "author": clean(row.get('author')),
            "publication_title": clean(row.get('publication_title')),
            "publication_date": clean(row.get('publication_date')),
            "url": clean(row.get('url')), "keywords": clean(row.get('keywords')),
            "abstract": clean(row.get('abstract')),
            "publisher": clean(row.get('publisher')),
            "field_of_study": clean(row.get('field_of_study')),
            "is_data_fusion": clean(row.get('is_data_fusion')),
            "classification_reason": clean(row.get('classification_reason')),
            "contributor_id": contributor_id,
        })

        # Only insert method if method name exists
        method_name = clean(row.get('name'))
        if method_name:
            u1 = clean(row.get('u1'))
            u3 = clean(row.get('u3'))
            utype = clean(row.get('uncertainty_type'))
            if utype and not u1:
                u1 = utype
            method_docs.append({
                "name": method_name, "method_key": clean(row.get('method_key')),
                "doi": doi, "description": clean(row.get('description')),
                "u1": u1, "u3": u3,
                "contributor_id": contributor_id,
            })

        # Only insert dataset if data_name exists
        data_name = clean(row.get('data_name'))
        if data_name:
            dataset_docs.append({
                "doi": doi, "data_name": data_name,
                "dataset_url": clean(row.get('dataset_url')),
                "method_key": clean(row.get('method_key')),
                "data_type": clean(row.get('data_type')),
                "collection_method": clean(row.get('collection_method')),
                "u2": clean(row.get('u2')),
                "spatial_coverage": clean(row.get('spatial_coverage')),
                "temporal_coverage": clean(row.get('temporal_coverage')),
                "format": clean(row.get('format')),
                "license": clean(row.get('license')),
                "provenance": clean(row.get('provenance')),
                "contributor_id": contributor_id,
            })

    if paper_docs:
        db.papers.insert_many(paper_docs)
    if method_docs:
        db.methods.insert_many(method_docs)
    if dataset_docs:
        db.datasets.insert_many(dataset_docs)

    return len(paper_docs), len(method_docs), len(dataset_docs)
```

### 3.5 Main Entry Point

```python
def main():
    client = MongoClient(MONGO_URI)
    db = client[DB_NAME]

    # Drop existing collections (fresh start)
    db.papers.drop()
    db.methods.drop()
    db.datasets.drop()
    db.contributors.drop()
    print("Cleared existing collections.")

    total_papers = total_methods = total_datasets = 0

    files = sorted(os.listdir(DATA_DIR))
    excel_files = [f for f in files if f.endswith('.xlsx')]
    csv_files = [f for f in files if f.endswith('.csv')]

    for filename in excel_files + csv_files:
        filepath = os.path.join(DATA_DIR, filename)
        name = contributor_name(filename)

        # Upsert contributor
        result = db.contributors.update_one(
            {"source_file": filename},
            {"$setOnInsert": {"name": name, "source_file": filename}},
            upsert=True,
        )
        cid = result.upserted_id or db.contributors.find_one({"source_file": filename})["_id"]

        is_csv = filename.endswith('.csv')

        try:
            if is_csv:
                p, m, d = import_single_sheet(filepath, db, cid, is_csv=True)
            else:
                xls = pd.ExcelFile(filepath)
                if len(xls.sheet_names) >= 2:
                    p, m, d = import_three_sheet(filepath, db, cid)
                else:
                    p, m, d = import_single_sheet(filepath, db, cid)

            total_papers += p
            total_methods += m
            total_datasets += d
            print(f"  {filename}: {p} papers, {m} methods, {d} datasets")
        except Exception as e:
            print(f"  ERROR {filename}: {e}")

    # Create indexes for faster queries
    db.papers.create_index("doi")
    db.papers.create_index("contributor_id")
    db.methods.create_index("doi")
    db.methods.create_index("contributor_id")
    db.datasets.create_index("doi")
    db.datasets.create_index("contributor_id")
    print("\nIndexes created.")

    client.close()
    print(f"\nTotal: {total_papers} papers, {total_methods} methods, {total_datasets} datasets")
    print(f"Database: {DB_NAME} on MongoDB Atlas")


if __name__ == '__main__':
    main()
```

**Run the import:**

```bash
python3 import_data.py
```

You should see output like:

```
Cleared existing collections.
  Aiden Mack.xlsx: 5 papers, 5 methods, 5 datasets
  Alexander Thompson.xlsx: 5 papers, 5 methods, 5 datasets
  ...
Total: 299 papers, 313 methods, 526 datasets
Database: data_fusion_kms on MongoDB Atlas
```

**Verify in Atlas:** Go to your MongoDB Atlas dashboard > Browse Collections. You should see four collections: `contributors`, `papers`, `methods`, `datasets`.

---

## Step 4 -- FastAPI Application (main.py)

This is the core of the application. Create `main.py` with these sections:

### 4.1 Imports and App Setup

```python
#!/usr/bin/env python3
"""Data Fusion Knowledge Management System - FastAPI + MongoDB."""

import re
from contextlib import asynccontextmanager

from bson import ObjectId
from fastapi import FastAPI, Request, Form
from fastapi.responses import RedirectResponse
from fastapi.templating import Jinja2Templates
from starlette.middleware.sessions import SessionMiddleware

from database import connect_db, close_db, get_db


@asynccontextmanager
async def lifespan(app):
    await connect_db()
    yield
    await close_db()


app = FastAPI(lifespan=lifespan)
app.add_middleware(SessionMiddleware, secret_key="data-fusion-kms-2026")
templates = Jinja2Templates(directory="templates")
```

**What this does:**

- `lifespan` -- connects to MongoDB when the app starts, disconnects when it stops
- `SessionMiddleware` -- enables session cookies (used for flash messages)
- `Jinja2Templates` -- loads HTML templates from the `templates/` directory

### 4.2 Helper Functions

```python
def flash(request: Request, message: str, category: str = "info"):
    """Store a flash message in the session (shown once on the next page)."""
    if "_messages" not in request.session:
        request.session["_messages"] = []
    request.session["_messages"].append({"message": message, "category": category})


def get_flashed_messages(request: Request):
    """Retrieve and clear flash messages from the session."""
    messages = request.session.pop("_messages", [])
    return messages


def oid(id_str: str) -> ObjectId:
    """Convert a string ID to a MongoDB ObjectId."""
    return ObjectId(id_str)


def doc_with_id(doc):
    """Convert MongoDB doc: add 'id' field as a string version of '_id'."""
    if doc is None:
        return None
    doc["id"] = str(doc["_id"])
    return doc
```

**Why `doc_with_id`?** MongoDB uses `ObjectId` for `_id`, but HTML templates need string IDs for URLs. This helper adds an `id` string field to every document.

### 4.3 Smart Search Helper

This function handles both simple keyword searches and natural-language queries:

```python
STOP_WORDS = {
    "a", "an", "the", "is", "are", "was", "were", "be", "been", "being",
    "in", "on", "at", "to", "for", "of", "with", "by", "from", "as",
    "and", "or", "not", "no", "but", "if", "then", "so", "do", "did",
    "has", "have", "had", "will", "would", "shall", "should", "may",
    "can", "could", "might", "must", "it", "its", "i", "me", "my",
    "we", "us", "our", "you", "your", "he", "she", "they", "them",
    "this", "that", "these", "those", "all", "each", "every", "any",
    "show", "find", "get", "list", "give", "used", "using", "about",
}


def build_regex(text: str) -> str:
    """Build a regex that matches any meaningful keyword in the query."""
    words = text.split()
    keywords = [w for w in words if w.lower().strip(".,!?;:\"'") not in STOP_WORDS]
    if not keywords:
        keywords = words
    # Short, simple queries -> literal match
    if len(keywords) <= 2 and len(words) <= 2:
        return re.escape(text)
    # Longer / natural-language queries -> match any keyword
    return "|".join(re.escape(k.strip(".,!?;:\"'")) for k in keywords)
```

**How it works:**

- `"fusion"` (short query) -> regex `fusion` (exact substring match)
- `"Show me all fusion methods used for Traffic Data."` -> regex `fusion|methods|Traffic|Data` (matches any keyword)
- Stop words like "show", "me", "all", "for" are filtered out

### 4.4 Dashboard Route

```python
@app.get("/", name="dashboard")
async def dashboard(request: Request):
    db = get_db()
    stats = {
        "papers": await db.papers.count_documents({}),
        "methods": await db.methods.count_documents({}),
        "datasets": await db.datasets.count_documents({}),
        "contributors": await db.contributors.count_documents({}),
    }

    # Top 10 fields of study (aggregation pipeline)
    top_fields_cursor = db.papers.aggregate([
        {"$match": {"field_of_study": {"$nin": [None, ""]}}},
        {"$group": {"_id": "$field_of_study", "cnt": {"$sum": 1}}},
        {"$sort": {"cnt": -1}},
        {"$limit": 10},
    ])
    top_fields = [{"field_of_study": r["_id"], "cnt": r["cnt"]}
                  async for r in top_fields_cursor]

    # Top 10 publishers
    top_pub_cursor = db.papers.aggregate([
        {"$match": {"publisher": {"$nin": [None, ""]}}},
        {"$group": {"_id": "$publisher", "cnt": {"$sum": 1}}},
        {"$sort": {"cnt": -1}},
        {"$limit": 10},
    ])
    top_publishers = [{"publisher": r["_id"], "cnt": r["cnt"]}
                      async for r in top_pub_cursor]

    # Recent 10 papers
    recent_cursor = db.papers.find().sort("_id", -1).limit(10)
    recent_papers = []
    async for p in recent_cursor:
        doc_with_id(p)
        c = await db.contributors.find_one({"_id": p.get("contributor_id")})
        p["contributor_name"] = c["name"] if c else None
        recent_papers.append(p)

    return templates.TemplateResponse(request, "dashboard.html", {
        "stats": stats,
        "top_fields": top_fields,
        "top_publishers": top_publishers,
        "recent_papers": recent_papers,
        "messages": get_flashed_messages(request),
    })
```

**MongoDB concepts used here:**

- `count_documents({})` -- count all documents in a collection
- `aggregate([...])` -- run an aggregation pipeline (like SQL `GROUP BY`)
- `$nin` -- "not in" operator, filters out `None` and empty strings
- `$group` -- groups documents by a field and counts them
- `$sort` / `$limit` -- sort descending by count, take top 10
- `.sort("_id", -1)` -- sort by `_id` descending (newest first)

### 4.5 Papers CRUD Routes

```python
@app.get("/papers", name="papers")
async def papers_list(request: Request, q: str = "", field: str = "", page: int = 1):
    db = get_db()
    per_page = 25
    query = {}
    conditions = []

    if q:
        rgx = {"$regex": build_regex(q), "$options": "i"}
        conditions.append({"$or": [
            {"title": rgx}, {"author": rgx}, {"doi": rgx},
            {"abstract": rgx}, {"keywords": rgx},
        ]})
    if field:
        conditions.append({"field_of_study": {"$regex": re.escape(field), "$options": "i"}})
    if conditions:
        query = {"$and": conditions} if len(conditions) > 1 else conditions[0]

    total = await db.papers.count_documents(query)
    total_pages = max(1, (total + per_page - 1) // per_page)

    cursor = db.papers.find(query).sort("_id", -1).skip((page - 1) * per_page).limit(per_page)
    rows = []
    async for p in cursor:
        doc_with_id(p)
        c = await db.contributors.find_one({"_id": p.get("contributor_id")})
        p["contributor_name"] = c["name"] if c else None
        rows.append(p)

    fields_list = await db.papers.distinct("field_of_study")
    fields_list = [{"field_of_study": f} for f in fields_list if f]

    return templates.TemplateResponse(request, "papers.html", {
        "papers": rows, "search": q, "field": field,
        "fields": fields_list, "page": page,
        "total_pages": total_pages, "total": total,
        "messages": get_flashed_messages(request),
    })
```

**Pagination explained:**

- `skip((page - 1) * per_page)` -- skip documents for previous pages
- `limit(per_page)` -- return only 25 documents
- `total_pages = (total + per_page - 1) // per_page` -- ceiling division

**Search explained:**

- `$regex` with `$options: "i"` -- case-insensitive regex search (replaces SQL `LIKE`)
- `$or` -- match any of the listed fields
- `distinct("field_of_study")` -- get unique values for the filter dropdown

The remaining CRUD routes follow the same pattern. Here is the **Add**, **Detail**, **Edit**, and **Delete** for papers:

```python
@app.get("/papers/add", name="add_paper")
async def add_paper_form(request: Request):
    return templates.TemplateResponse(request, "edit_paper.html", {
        "paper": None,
        "messages": get_flashed_messages(request),
    })


@app.post("/papers/add")
async def add_paper_submit(
    request: Request,
    doi: str = Form(""), title: str = Form(""), author: str = Form(""),
    publication_title: str = Form(""), publication_date: str = Form(""),
    url: str = Form(""), keywords: str = Form(""), abstract: str = Form(""),
    publisher: str = Form(""), field_of_study: str = Form(""),
    is_data_fusion: str = Form(""), classification_reason: str = Form(""),
):
    db = get_db()
    doc = {
        "doi": doi or None, "title": title or None, "author": author or None,
        "publication_title": publication_title or None,
        "publication_date": publication_date or None,
        "url": url or None, "keywords": keywords or None,
        "abstract": abstract or None, "publisher": publisher or None,
        "field_of_study": field_of_study or None,
        "is_data_fusion": is_data_fusion or None,
        "classification_reason": classification_reason or None,
    }
    await db.papers.insert_one(doc)
    flash(request, "Paper added.", "success")
    return RedirectResponse(request.url_for("papers"), status_code=303)


@app.get("/papers/{paper_id}", name="paper_detail")
async def paper_detail(request: Request, paper_id: str):
    db = get_db()
    paper = await db.papers.find_one({"_id": oid(paper_id)})
    if not paper:
        flash(request, "Paper not found.", "danger")
        return RedirectResponse(request.url_for("papers"), status_code=303)
    doc_with_id(paper)
    c = await db.contributors.find_one({"_id": paper.get("contributor_id")})
    paper["contributor_name"] = c["name"] if c else None

    methods = []
    if paper.get("doi"):
        async for m in db.methods.find({"doi": paper["doi"]}):
            doc_with_id(m)
            methods.append(m)
    datasets = []
    if paper.get("doi"):
        async for d in db.datasets.find({"doi": paper["doi"]}):
            doc_with_id(d)
            datasets.append(d)

    return templates.TemplateResponse(request, "paper_detail.html", {
        "paper": paper,
        "methods": methods, "datasets": datasets,
        "messages": get_flashed_messages(request),
    })


@app.get("/papers/{paper_id}/edit", name="edit_paper")
async def edit_paper_form(request: Request, paper_id: str):
    db = get_db()
    paper = await db.papers.find_one({"_id": oid(paper_id)})
    doc_with_id(paper)
    return templates.TemplateResponse(request, "edit_paper.html", {
        "paper": paper,
        "messages": get_flashed_messages(request),
    })


@app.post("/papers/{paper_id}/edit")
async def edit_paper_submit(
    request: Request, paper_id: str,
    doi: str = Form(""), title: str = Form(""), author: str = Form(""),
    publication_title: str = Form(""), publication_date: str = Form(""),
    url: str = Form(""), keywords: str = Form(""), abstract: str = Form(""),
    publisher: str = Form(""), field_of_study: str = Form(""),
    is_data_fusion: str = Form(""), classification_reason: str = Form(""),
):
    db = get_db()
    await db.papers.update_one({"_id": oid(paper_id)}, {"$set": {
        "doi": doi or None, "title": title or None, "author": author or None,
        "publication_title": publication_title or None,
        "publication_date": publication_date or None,
        "url": url or None, "keywords": keywords or None,
        "abstract": abstract or None, "publisher": publisher or None,
        "field_of_study": field_of_study or None,
        "is_data_fusion": is_data_fusion or None,
        "classification_reason": classification_reason or None,
    }})
    flash(request, "Paper updated.", "success")
    return RedirectResponse(request.url_for("paper_detail", paper_id=paper_id), status_code=303)


@app.post("/papers/{paper_id}/delete", name="delete_paper")
async def delete_paper(request: Request, paper_id: str):
    db = get_db()
    await db.papers.delete_one({"_id": oid(paper_id)})
    flash(request, "Paper deleted.", "success")
    return RedirectResponse(request.url_for("papers"), status_code=303)
```

**MongoDB CRUD operations:**

| Operation | MongoDB Method |
|-----------|---------------|
| Create | `db.papers.insert_one(doc)` |
| Read one | `db.papers.find_one({"_id": oid(id)})` |
| Read many | `db.papers.find(query)` |
| Update | `db.papers.update_one({"_id": oid(id)}, {"$set": {...}})` |
| Delete | `db.papers.delete_one({"_id": oid(id)})` |

### 4.6 Methods and Datasets Routes

The methods and datasets routes follow the exact same CRUD pattern as papers. The only differences are the field names and which fields are searched. See the complete `main.py` in the reference zip file.

### 4.7 Contributors Routes

Contributors are read-only (no add/edit/delete):

```python
@app.get("/contributors", name="contributors")
async def contributors_list(request: Request):
    db = get_db()
    rows = []
    async for c in db.contributors.find().sort("name", 1):
        doc_with_id(c)
        c["paper_count"] = await db.papers.count_documents({"contributor_id": c["_id"]})
        c["method_count"] = await db.methods.count_documents({"contributor_id": c["_id"]})
        c["dataset_count"] = await db.datasets.count_documents({"contributor_id": c["_id"]})
        rows.append(c)

    return templates.TemplateResponse(request, "contributors.html", {
        "contributors": rows,
        "messages": get_flashed_messages(request),
    })
```

### 4.8 Global Search Route

Searches across all three collections at once:

```python
@app.get("/search", name="search")
async def search_all(request: Request, q: str = ""):
    if not q:
        return templates.TemplateResponse(request, "search.html", {
            "q": "", "papers": [], "methods": [], "datasets": [],
            "messages": get_flashed_messages(request),
        })

    db = get_db()
    rgx = {"$regex": build_regex(q), "$options": "i"}

    found_papers = []
    async for p in db.papers.find({"$or": [
        {"title": rgx}, {"author": rgx}, {"doi": rgx},
        {"abstract": rgx}, {"keywords": rgx},
    ]}).limit(50):
        doc_with_id(p)
        c = await db.contributors.find_one({"_id": p.get("contributor_id")})
        p["contributor_name"] = c["name"] if c else None
        found_papers.append(p)

    # ... same pattern for methods and datasets (see full source) ...

    return templates.TemplateResponse(request, "search.html", {
        "q": q,
        "papers": found_papers, "methods": found_methods,
        "datasets": found_datasets,
        "messages": get_flashed_messages(request),
    })
```

### 4.9 App Entry Point

```python
if __name__ == "__main__":
    import uvicorn
    uvicorn.run("main:app", host="0.0.0.0", port=8000, reload=True)
```

---

## Step 5 -- Jinja2 HTML Templates

Create 14 HTML files in the `templates/` directory. All templates extend `base.html`.

### 5.1 Base Template (`templates/base.html`)

This is the layout that every page inherits. It includes:

- A dark sidebar with navigation links
- A mobile-responsive navbar
- Flash message display
- Bootstrap 5 CSS/JS via CDN

**Key differences from Flask templates:**

| Flask | FastAPI |
|-------|---------|
| `{{ url_for('papers') }}` | `{{ request.url_for('papers') }}` |
| `{% if request.endpoint == 'papers' %}` | `{% if '/papers' in request.url.path %}` |
| `get_flashed_messages(with_categories=true)` | `messages` variable passed from route |

The flash message block in `base.html` looks like:

```html
{% if messages %}
    {% for msg in messages %}
    <div class="alert alert-{{ msg.category }} alert-dismissible fade show" role="alert">
        {{ msg.message }}
        <button type="button" class="btn-close" data-bs-dismiss="alert"></button>
    </div>
    {% endfor %}
{% endif %}
```

### 5.2 Template Files Summary

All 14 templates are provided in the reference zip file. Here is what each one does:

| Template | Purpose |
|----------|---------|
| `base.html` | Shared layout (sidebar, navbar, flash messages) |
| `dashboard.html` | Stats cards, top 10 tables, recent papers |
| `papers.html` | Paper list with search, field filter, pagination |
| `paper_detail.html` | Single paper view with related methods/datasets |
| `edit_paper.html` | Add/edit paper form (same template for both) |
| `methods.html` | Method list with search and pagination |
| `method_detail.html` | Single method view with uncertainty types and related paper |
| `edit_method.html` | Add/edit method form |
| `datasets.html` | Dataset list with search and pagination |
| `dataset_detail.html` | Single dataset view with related paper |
| `edit_dataset.html` | Add/edit dataset form |
| `contributors.html` | Contributor list with contribution counts |
| `contributor_detail.html` | All contributions by one contributor |
| `search.html` | Global search across papers, methods, datasets |

---

## Step 6 -- Running the Application

### 6.1 Import Data First

Make sure your `.xlsx` and `.csv` data files are in the parent directory, then:

```bash
cd mongodb_app
python3 import_data.py
```

### 6.2 Start the Server

```bash
python3 main.py
```

Or equivalently:

```bash
uvicorn main:app --reload --port 8000
```

### 6.3 Open in Browser

Go to [http://localhost:8000](http://localhost:8000)

You should see the dashboard with:

- Paper, method, dataset, and contributor counts
- Top 10 fields of study and publishers
- 10 most recent papers

---

## Step 7 -- Testing Everything

Manually verify each feature:

1. **Dashboard** -- check that counts and tables load at `/`
2. **Papers list** -- go to `/papers`, verify pagination works
3. **Search** -- search for "fusion" on the papers page
4. **Paper detail** -- click a paper title, check related methods/datasets appear
5. **Add paper** -- click "Add Paper", fill the form, submit
6. **Edit paper** -- click "Edit" on a paper, change a field, submit
7. **Delete paper** -- click "Delete", confirm, verify it is gone
8. **Repeat steps 2-7** for Methods and Datasets
9. **Contributors** -- go to `/contributors`, click a name, verify their contributions
10. **Global search** -- go to `/search`, try:
    - Simple: `Kalman`
    - Long query: `Show me all fusion methods used for Traffic Data.`

---

## Key Concepts Explained

### MongoDB vs SQL

| Concept | SQL (SQLite) | MongoDB |
|---------|-------------|---------|
| Table | `CREATE TABLE papers (...)` | Collection `papers` (created automatically) |
| Row | Row with fixed columns | Document (flexible JSON-like object) |
| Primary Key | `id INTEGER PRIMARY KEY` | `_id` (auto-generated ObjectId) |
| Foreign Key | `FOREIGN KEY (contributor_id)` | Store `contributor_id` as ObjectId reference |
| Search | `WHERE title LIKE '%fusion%'` | `{"title": {"$regex": "fusion", "$options": "i"}}` |
| Join | `LEFT JOIN contributors c ON ...` | Separate query: `db.contributors.find_one({"_id": ...})` |
| Count | `SELECT COUNT(*) FROM papers` | `db.papers.count_documents({})` |
| Group By | `GROUP BY field ORDER BY cnt DESC` | Aggregation pipeline with `$group`, `$sort` |
| Index | `CREATE INDEX idx ON papers(doi)` | `db.papers.create_index("doi")` |

### Async/Await

FastAPI is async. Every database call uses `await`:

```python
# This does NOT block the server while waiting for MongoDB
paper = await db.papers.find_one({"_id": oid(paper_id)})
```

This means your server can handle many requests at the same time, even if MongoDB is slow.

### ObjectId

MongoDB generates a unique 24-character hex string for every document:

```python
# In MongoDB: {"_id": ObjectId("69dee9cfec886a8f7f532577")}
# In URL:     /papers/69dee9cfec886a8f7f532577
# In Python:  oid("69dee9cfec886a8f7f532577") -> ObjectId object
```

### POST/Redirect/GET Pattern

After form submissions, we redirect instead of rendering directly:

```python
return RedirectResponse(request.url_for("papers"), status_code=303)
```

`status_code=303` tells the browser to use GET for the redirect, preventing duplicate form submissions on refresh.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| `ServerSelectionTimeoutError` | Check your MongoDB Atlas Network Access -- add your IP or `0.0.0.0/0` |
| `Authentication failed` | Verify username/password in your connection string matches Atlas Database Access |
| `Address already in use` (port 8000) | Kill the existing process: `lsof -ti:8000 \| xargs kill -9` |
| `ModuleNotFoundError: No module named 'motor'` | Run `pip install -r requirements.txt` |
| Search returns no results for long queries | Make sure you are using the `build_regex()` function, not `re.escape()` directly |
| Template error `unhashable type: 'dict'` | Use the Starlette 1.0 API: `TemplateResponse(request, "name.html", context)` not `TemplateResponse("name.html", {"request": request, ...})` |
| Import shows `ERROR filename: ...` | Check that the Excel file is not corrupted; open it manually to verify |
