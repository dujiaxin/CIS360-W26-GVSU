
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

* **Instructor Inputs:** I will provide each group with **5 "Seed Papers"** related to data fusion (e.g., urban planning, environmental science).
* **Group Inputs:** Your group may find additional relevant papers** using GVSU library resources or Google Scholar to expand the network.
* **The Task:** You must read these 10 papers and manually extract structured information. You are doing the work of the "AI Extractor" described in modern research.



### Phase 2: Ontology & Schema Design (The Structure)

You must design a database schema (Ontology) to store the extracted information. Your schema must explicitly handle the relationships shown in **Figure 3 of the project description**.

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

### A. The Dataset (Spreadsheet/CSV) (will be 5 points in your midterm)

Before importing into your system, submit the raw structured data you extracted from the 10 papers.

* *Columns must include:* Paper Title, Dataset Name, Method Name, Uncertainty Type (U1/U2/U3), and Description.

Sample excel will be provided

### B. The System Prototype (Code/Database Export)

Submit your database file (SQL dump, GraphML export, or JSON file) and the queries used to search it.

* If you build a user interface (e.g., a simple Python Streamlit app or a web form), submit the code.
* If you use a pure database approach, submit a script with **5 complex SQL or Cypher queries** that demonstrate the system's power.

### C. Final Report (5–6 Pages)

1. **Ontology Design:** A diagram showing your nodes (Data, Method, Paper) and relationships. Explain how you modeled "Uncertainty."
2. **Extraction Process:** Describe how you standardized the data. (e.g., "How did we decide what counts as U1 vs U2?").
3. **Search Demonstration:** Screenshots of your system answering the specific queries listed in Phase 3.
4. **Challenges:** Discuss the difficulty of converting unstructured text (PDFs) into structured rows/nodes.

---

## 4. Grading Rubric (Total: 100 Points)



| **Category**                          | **Criteria**                                                                                                                                                                                                                     | **Points** |
|--------------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|-----------|
| **Data Engineering**<br>(Evidenced in Report) | **Extraction & Expansion**<br>• (5 pts) Accurate extraction of Data/Method/Uncertainty entities from the 5 instructor papers.<br>• (4 pts) Relevance and quality of the 5 additional papers found by the group.               | 9         |
| **System Design**<br>(Evidenced in Report)    | **Ontology & Schema**<br>• (5 pts) Database schema accurately links Data to Fusion Methods using a logical structure (Graph nodes or SQL tables).<br>**Uncertainty Modeling**<br>• (4 pts) Correct application of the "Three-Filter" framework (U1: Conception, U2: Measurement, U3: Analysis) within the schema. | 9         |
| **Functionality**<br>(Evidenced in Demo)      | **Searchability**<br>• (5 pts) System successfully executes queries to discover relationships (e.g., "Show all methods fused with Census Data" or "List datasets with U2 uncertainty").<br>**Implementation**<br>• (3 pts) The database is populated with the 10 papers and runs without critical errors during the demo. | 8         |
| **Report Quality**                            | **Clarity & Documentation**<br>• (4 pts) Clear visualization of the schema/ontology, professional writing, and a concise explanation of the design process.                                                                  | 4         |
| **Total**                                    |                                                                                                                                                                                                                                 | **30**    |

---

## 5. Helpful Hints for Students

**For the Database:** You may use **Neo4j** (Graph DB) as it fits the "Knowledge Graph" concept best, or **PostgreSQL** if you are more comfortable with relational tables.



**For Uncertainty:** If a paper doesn't explicitly say "This is U2 uncertainty," use your judgment based on the definitions provided in class (e.g., "sensor noise" is U2; "model bias" is U3).


* **Tools:**
* *Data Entry:* Excel/Google Sheets (to organize before coding).
* *Visualization:* Neo4j Bloom or simple Python network charts.
* Please use AI as mush as possible.