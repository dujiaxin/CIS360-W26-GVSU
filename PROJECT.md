
# CIS 360: Group Project — Building a Scientific Knowledge Graph

**Theme:** *Constructing a Searchable Database for Data Fusion Research*
**Weight:** 30% of Final Grade
**Group Size:** 3–4 Students
**Due Date:** Week 15 (April 20–24)

## 1. Project Overview

In the scientific community, thousands of datasets and methods are published every year, but they are often "siloed" in PDF documents. Researchers struggle to find which datasets can be combined (fused) effectively and what uncertainties exist in that process.

**Your Goal:**
Your group will build a **functional "Scientific Knowledge System"** (a prototype). You will take unstructured information (academic papers) and convert it into a structured, searchable database (Knowledge Graph or Relational DB) that allows a user to query connections between **Data**, **Methods**, and **Uncertainty**.

You are simulating the role of an Information Architect: designing the storage schema, performing the data extraction (ETL), and building the search interface.

---

## 2. Project Workflow & Requirements

### Phase 1: Information Retrieval & Extraction (The Data Source)

You cannot build a system without data.

* **Instructor Inputs:** I will provide each group with **5 "Seed Papers"** related to data fusion (e.g., urban planning, environmental science). [Link to the files](https://drive.google.com/drive/folders/1ida193bDjJFB1ZQ8Akg6O_Hcx_PrrJh2?usp=sharing)
* **Group Inputs:** Your group may find additional relevant papers** using GVSU library resources or Google Scholar to expand the network.
* **The Task:** You must read these papers and extract structured information. You are doing the work of the "AI Extractor" described in modern research.



### Phase 2: Ontology & Schema Design (The Structure)

You must design a database schema (Ontology) to store the extracted information. 

**Required Entities to Model:**

1. **Datasets:** Metadata including Sensor Type, Spatial Coverage, Date.


2. **Fusion Methods:** The algorithm or technique used to combine the data.


3. **Uncertainty (Crucial):** You must tag the uncertainty level described in the paper using the **Three-Filter Framework**:


*U1 (Conception):* Ambiguity in how the real world is defined/abstracted.


*U2 (Measurement):* Sensor errors, resolution limits, missing data.


*U3 (Analysis):* Algorithmic assumptions or processing errors.



### Phase 3: System Implementation (The Build)

You must implement this in a software tool (e.g., Neo4j, PostgreSQL, or a Python-backed SQLite/JSON system). The system must be **functional**, not just a drawing.

**Minimum Functional Requirements:**

**Storage:** All given papers' data must be stored in your database.

**Linkage:** The system must link Datasets to the Methods they use.


**Searchability:** The system must answer queries such as:
* *"Show me all fusion methods used for Traffic Data."*
* *"List all papers that report U2 (Measurement) uncertainty for Satellite Imagery."*
* *"Which datasets are commonly fused with Census Data?"*



---

## 3. Deliverables

### A. The Dataset (Spreadsheet/CSV) (will be 10 points in your midterm)

Before importing into your system, submit the raw structured data you extracted from the 10 papers.

* *Columns must include:* Paper Title, Dataset Name, Method Name, Uncertainty Type (U1/U2/U3), and Description.

[Sample excel](https://docs.google.com/spreadsheets/d/1Nkp6V2jpxQ-SMtyuCzLz_ia_z2fjhe_k/edit?usp=sharing&ouid=100433744693995346627&rtpof=true&sd=true)

### B. The System Prototype (Code/Database Export)
**Submission Method:** GitHub Repository Link

Submit your database file (SQL dump, GraphML export, or JSON file) and the queries used to search it.

* **Option 1 (UI):** If you build a user interface (e.g., a simple Python Streamlit app or a web form), submit the source code.
* **Option 2 (Database Script):** If you use a pure database approach, submit a script with **5 complex SQL or Cypher queries** that demonstrate the system's power. *Note: These must include the 3 specific "Stakeholder Queries" listed in the Final Report section below.*

### C. Final Report (5–6 Pages)

1.  **Ontology Design**
    * Provide a diagram showing your nodes (`Data`, `Method`, `Paper`) and relationships.
    * Explicitly explain how you modeled "Uncertainty" within this schema.

2.  **Lifecycle Justification**
    * *Requirement:* Describe your Data Lifecycle decisions in detail:
        * **Extraction (Collection):** You converted unstructured text (PDFs) into structured rows. What was the hardest part of standardizing the 'Method Name' across different papers?
        * **Modeling (Storage):** You chose a specific model (Graph or Relational). Explain *why* this model fits the 'Data Fusion' problem better than a simple Excel spreadsheet. *(Hint: Discuss how you handled the many-to-many relationships between Datasets and Methods).*

3.  **Search Demonstration**
    * Provide screenshots of your system answering the following three specific queries:
        * **The 'Linkage' Query:** Find all Fusion Methods that have been applied to *both* 'Dataset A' and 'Dataset B'. *(Demonstrates graph traversal).*
        * **The 'Uncertainty' Query:** Find all papers that report **U2 (Measurement)** uncertainty for a specific sensor type. *(Demonstrates filtering).*
        * **The 'Discovery' Query:** Find the most 'popular' dataset in your graph (the one with the most connections to different methods). *(Demonstrates aggregation).*

4.  **Challenges**
    * Discuss the difficulty of converting unstructured text (PDFs) into structured rows/nodes.

---

## 4. Grading Rubric (Total: 50 Points)

| Category | Criteria | Points |
| :--- | :--- | :--- |
| **Data Engineering**<br>*(Midterm)* | **Extraction & Expansion**<br>• (8 pts) Accurate extraction of Data/Method/Uncertainty entities from the 5 instructor papers.<br>• (2 pts) Relevance and quality of additional papers found by the group. | **10**<br>*(Midterm)* |
| **System Design**<br>*(Report)* | **Ontology & Schema**<br>• (5 pts) Database schema accurately links Data to Fusion Methods using a logical structure (Graph nodes or SQL tables).<br><br>**Uncertainty Modeling & Lifecycle**<br>• (5 pts) Correct application of the "Three-Filter" framework (U1/U2/U3).<br>*(Assessment of PI-3 Lifecycle)*<br>• **High:** Justifies the Graph model by explaining that "Data Fusion" is inherently relational; clearly articulates extraction challenges.<br>• **Low:** Describes the database but does not explain *why* it was chosen over a flat file; ignores extraction challenges. **Clarity & Documentation**<br>• (5 pts) Clear visualization of schema/ontology, professional writing, and concise explanation of design process.| **15** |
| **Functionality**<br>*(Demo)* | **Searchability & Analytics**<br>• (7 pts) System successfully executes queries to discover relationships.<br>• **High:** Queries are syntactically correct and accurately answer the business questions. The 'Linkage' query correctly handles the join/traversal.<br>• **Low:** Queries return errors or irrelevant data (e.g., listing all methods instead of filtering).<br><br>**Implementation**<br>• (3 pts) Database populated with papers and runs without critical errors during demo. | **10** |
| **Impress Me** | **Innovation**<br>• (5 pts) If the instructor has an *aha* moment (e.g., unique visualization, novel insight, or advanced UI). | **5** |
| **Participation** | **Self and Peer Evaluation**<br>• (10 pts) Completed individually during the Final Exam. | **10**<br>*(Final Exam)* |
| **TOTAL** | | **50** |
---


## 5. Helpful Hints for Students

**For the Database:** You may use **Neo4j** (Graph DB) as it fits the "Knowledge Graph" concept best, or **PostgreSQL** if you are more comfortable with relational tables.



**For Uncertainty:** If a paper doesn't explicitly say "This is an uncertainty/limitation," use your judgment based on the definitions provided in class (e.g., "sensor noise" is U2; "model bias" is U3).


* **Tools:**
* *Data Entry:* Excel/Google Sheets (to organize before coding).
* *Visualization:* Neo4j Bloom or simple Python network charts.
* Please use AI as mush as possible.
