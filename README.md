# Cold-Chain AI Dispatcher

![Cold-Chain AI Dispatcher](docs/images/banner.png)

An AI agent that answers questions about refrigerated trucks in plain English.
It reads fleet data from a SQL database, live weather from a public API, and company rules from a vector database.
Then it writes a short report with the action that the dispatcher must take.

## Contents

1. [The problem](#1-the-problem)
2. [The solution](#2-the-solution)
3. [The stack and why we use it](#3-the-stack-and-why-we-use-it)
4. [How it works](#4-how-it-works)
5. [Project layout](#5-project-layout)
6. [Build it from scratch](#6-build-it-from-scratch)
7. [Deploy to AWS EC2](#7-deploy-to-aws-ec2)
8. [Troubleshooting](#8-troubleshooting)
9. [Known limits](#9-known-limits)

## 1. The problem

Some trucks carry food or medicine that must stay cold.
If the cargo gets too warm, it spoils.
A dispatcher watches the trucks and must act fast when something goes wrong.

To make one decision, the dispatcher must check three systems:

| Question | Where the answer is |
| :--- | :--- |
| Where is the truck, and how cold is the cargo? | An old company database |
| Is the weather a risk at that location? | A weather service |
| What do the company rules tell me to do? | A long policy document (SOP) |

This takes time.
The old database also has cryptic column names such as `IOT_TEMP_VAL_C`, so it is hard to query.

## 2. The solution

The dispatcher types one question into a chat page. For example:

> Find any active shipments near Los Angeles (Latitude ~33.8, Longitude ~-118.1). Check the local weather there, and tell me if the current cargo temperature violates the SOP for fresh perishables.

The AI agent then does these steps:

1. It reads the question and decides which tools it needs.
2. It writes a SQL query and gets the truck data.
3. It gets the live weather at the truck location.
4. It searches the SOP for the temperature rule.
5. It writes a report with a summary, a data table, and the action plan.

The agent can also decide to use fewer tools.
For a general question, such as "What is a Tier 2 escalation?", it searches only the SOP.

### Safety

You must not give an AI full access to a company database.
This project uses three controls:

- **Read-only login.** The agent uses the `USR_FDE_RO` login. This login can read data, but it cannot change data.
- **Clean view only.** The agent can see only the view `FDE_VIEWS.VW_ACTIVE_FLEET`. The database denies access to the raw table.
- **Audit log.** The app writes each tool call and each answer to the table `FDE_VIEWS.AgentAuditLog`.

## 3. The stack and why we use it

| Part | Tool | Why |
| :--- | :--- | :--- |
| Language | Python 3.12 | All the AI libraries support Python first. |
| Agent | LangGraph | It lets the agent loop between "think" and "use a tool" until it has an answer. |
| LLM | OpenAI, DeepSeek, or local Ollama | You select one in `.env`. Ollama runs free on your own machine. |
| Legacy database | Microsoft SQL Server 2022 in Docker | Many large companies still use SQL Server. Docker makes it easy to start. |
| Rules search | Pinecone | It finds SOP text by meaning, not only by exact words. |
| Embeddings | OpenAI or `BAAI/bge-m3` | These models change text into vectors for Pinecone. `bge-m3` is free and local. |
| Weather | Open-Meteo API | It is free and needs no API key. |
| User interface | Streamlit | It makes a chat page in pure Python. |
| Hosting | AWS EC2 and GitHub Actions | One button in GitHub deploys the new code to the server. |

## 4. How it works

```
                 Dispatcher
                     |
                     v
          +---------------------+
          |  Streamlit chat UI  |----> AgentAuditLog (SQL)
          +---------------------+
                     |
                     v
          +---------------------+
          |  LangGraph agent    |
          |  reasoner <-> tools |
          +---------------------+
             |       |        |
             v       v        v
     SQL Server   Open-Meteo   Pinecone
     (read-only   (weather)    (SOP rules)
      view)
```

The agent has two nodes:

- **reasoner**: The LLM reads the conversation. It writes a final answer, or it asks for one or more tools.
- **tools**: This node runs the tools and gives the results back to the reasoner.

The loop stops when the reasoner writes an answer and asks for no more tools.

## 5. Project layout

```
.github/workflows/deploy.yml         Manual deploy to EC2
.streamlit/config.toml               Streamlit settings
data/raw/                            The logistics dataset (CSV)
data/policy/                         SOP documents for Pinecone
data/cache/                          File hashes, so ingestion skips files that did not change
docs/images/                         Images for this README
docs/instrutions.md                  The original build notes, phase by phase
scripts/ingest_legacy_data.py        Loads the CSV into SQL Server
scripts/setup_security_and_view.sql  Makes the view and the read-only login
scripts/ingest_sop_pinecone.py       Loads the SOP files into Pinecone
src/agent_tools.py                   The 3 tools
src/orchestrator.py                  The LangGraph agent and a terminal chat loop
src/prompts/system_prompt.txt        The report format for the LLM
src/ui.py                            The Streamlit app
```

## 6. Build it from scratch

This section builds the full project in 12 steps.
Each step tells you **why** you do it, gives the **code**, and shows the **expected output**.

You can use this section in two ways:

- **From scratch:** Make an empty folder and write each file yourself.
- **From the repo:** Clone the repo. The files already exist. Read the code in each step, then run it.

Your output will not be exactly the same as the examples.
Numbers, weather values, and LLM text change from run to run.

| Step | What you build | Time |
| :--- | :--- | :--- |
| 1 to 3 | Tools, folders, packages, and `.env` | 30 minutes |
| 4 to 7 | The legacy database and its security layer | 30 minutes |
| 8 | The SOP in Pinecone | 15 minutes |
| 9 to 11 | The tools, the agent, and the system prompt | 45 minutes |
| 12 | The web app | 20 minutes |

### Step 1: Install the tools

**Why:** Each tool connects one part of the system.

Install these items:

| Tool | Used for |
| :--- | :--- |
| Python 3.12 | All the code |
| Docker Desktop | The SQL Server database |
| [ODBC Driver 18 for SQL Server](https://learn.microsoft.com/en-us/sql/connect/odbc/download-odbc-driver-for-sql-server) | The link between Python and SQL Server |
| VS Code with the "SQL Server (mssql)" extension | Manual SQL queries |
| A free [Pinecone](https://app.pinecone.io/) account | The SOP vector database |
| [Ollama](https://ollama.com/) (optional) | A free local LLM |

Check the install:

```bash
python --version
docker --version
ollama --version
```

Expected output:

```
Python 3.12.10
Docker version 28.x.x, build xxxxxxx
ollama version is 0.x.x
```

### Step 2: Make the project folders and install the packages

**Why:** A virtual environment (`venv`) keeps the packages for this project apart from other projects.

```bash
mkdir cold-chain-logistics-FDE-Project
cd cold-chain-logistics-FDE-Project
mkdir data data\raw data\policy data\cache scripts src src\prompts
python -m venv venv
venv\Scripts\activate
```

On Linux or macOS, use `mkdir -p data/raw data/policy data/cache scripts src/prompts` and `source venv/bin/activate`.

Make the file `requirements.txt`.
Fixed versions make sure that the code runs the same on each machine.

```text
faiss-cpu==1.15.0
langchain-community==0.4.2
langchain-huggingface==1.2.2
langchain-openai==1.6.0
langchain-pinecone==0.2.13
langgraph==1.2.11
langgraph-checkpoint==4.2.0
openai==3.3.1
openpyxl==3.1.5
pandas==3.0.5
pinecone==7.3.0
pyodbc==5.3.0
pypdf==6.16.2
python-dotenv==1.2.3
sentence-transformers==6.0.0
SQLAlchemy==2.0.52
streamlit==1.62.0
torch==2.13.0
```

The full list is in [requirements.txt](requirements.txt).

```bash
pip install -r requirements.txt
```

Expected output (last line):

```
Successfully installed SQLAlchemy-2.0.52 langgraph-1.2.11 pinecone-7.3.0 streamlit-1.62.0 ...
```

The install takes 5 to 10 minutes, because `torch` is large.

### Step 3: Make the `.env` file

**Why:** Keys and passwords must not go into the code.
The code reads them from `.env`, and Git ignores this file.

Make `.gitignore`:

```text
.env
__pycache__/
venv/
.venv/
```

Make `.env` in the project root:

```ini
# LLM: OPENAI, DEEPSEEK, or any other value for local Ollama
Agent_llm=OLLAMA
OPENAI_API_KEY=
DEEPSEEK_API_KEY=

# Embeddings: OPENAI, or any other value for the local model
Embeddings_model=LOCAL
Local_Embedding_Model=BAAI/bge-m3
PINECONE_API_KEY=your-pinecone-key

# SQL Server
SQL_SERVER_HOST=localhost
SQL_SERVER_PORT=1433
SQL_ADMIN_USER=sa
SQL_ADMIN_PASSWORD=your-strong-sa-password
SQL_AGENT_USER=USR_FDE_RO
SQL_AGENT_PASSWORD=your-agent-password
```

SQL Server needs a strong password: 8 or more characters, with uppercase, lowercase, digits, and symbols.

If you use Ollama, download the model:

```bash
ollama pull qwen2.5:7b
```

Expected output (last line):

```
success
```

### Step 4: Start the legacy database

**Why:** This step makes a copy of an old company database on your machine.

```bash
docker run -e "ACCEPT_EULA=Y" -e "MSSQL_SA_PASSWORD=your-strong-sa-password" -p 1433:1433 --name legacy-mssql -d mcr.microsoft.com/mssql/server:2022-latest
```

Use the same password as `SQL_ADMIN_PASSWORD` in `.env`.
Wait about 20 seconds for SQL Server to start. Then check it:

```bash
docker ps
```

Expected output:

```
CONTAINER ID   IMAGE                                        STATUS         PORTS                    NAMES
3f9c1a2b7d4e   mcr.microsoft.com/mssql/server:2022-latest   Up 25 seconds  0.0.0.0:1433->1433/tcp   legacy-mssql
```

If the container is not in the list, see [Troubleshooting](#8-troubleshooting).

### Step 5: Load the dataset into a "legacy" table

**Why:** Real legacy tables often have cryptic column names.
This script copies that problem on purpose, so that Step 6 can fix it.

Download the dataset from [Kaggle](https://www.kaggle.com/datasets/datasetengineer/logistics-and-supply-chain-dataset).
Save it as `data/raw/dynamic_supply_chain_logistics_dataset.csv`.
The file has 32,065 rows. Each row is one truck reading.

Make `scripts/ingest_legacy_data.py`:

```python
from pathlib import Path
import os
import urllib
import pandas as pd
from sqlalchemy import create_engine
from dotenv import load_dotenv

# Find the project root, so the script works from any folder.
project_root = Path(__file__).resolve().parent.parent
load_dotenv(project_root / ".env")

data_path = project_root / "data" / "raw" / "dynamic_supply_chain_logistics_dataset.csv"

print(f"Loading CSV from {data_path}...")
df = pd.read_csv(data_path)

# Rename clean columns to old-style names, like a system from the year 2000.
legacy_mapping = {
    'timestamp': 'TS_UTC',
    'vehicle_gps_latitude': 'V_LAT',
    'vehicle_gps_longitude': 'V_LON',
    'iot_temperature': 'IOT_TEMP_VAL_C',
    'cargo_condition_status': 'CGO_COND_CD',
    'risk_classification': 'RISK_CLS_TXT',
    'delay_probability': 'DELAY_PROB_DEC',
    'port_congestion_level': 'PRT_CNG_LVL',
    'route_risk_level': 'RT_RSK_IDX'
}
df_legacy = df[list(legacy_mapping.keys())].rename(columns=legacy_mapping)
df_legacy['SYS_INGEST_FLAG'] = 'Y'

print("Connecting to legacy MSSQL Database...")
connection_string = (
    "DRIVER={ODBC Driver 18 for SQL Server};"
    f"SERVER={os.getenv('SQL_SERVER_HOST', 'localhost')},{os.getenv('SQL_SERVER_PORT', '1433')};"
    "DATABASE=master;"
    f"UID={os.getenv('SQL_ADMIN_USER')};"
    f"PWD={os.getenv('SQL_ADMIN_PASSWORD')};"
    "Encrypt=no;"
    "TrustServerCertificate=yes;"
)
params = urllib.parse.quote_plus(connection_string)
engine = create_engine(f"mssql+pyodbc:///?odbc_connect={params}")

table_name = 'TBL_SC_FLEET_HIST_RAW'
print(f"Ingesting into {table_name}. This may take a minute...")
df_legacy.to_sql(table_name, engine, if_exists='replace', index=False, schema='dbo')

print("✅ Legacy data ingestion complete!")
```

Key points:

- `if_exists='replace'` deletes and makes the table again. You can run the script again with no duplicate rows.
- `Encrypt=no` and `TrustServerCertificate=yes` are necessary, because the Docker database has no real certificate. Use real certificates in production.

Run it:

```bash
python scripts/ingest_legacy_data.py
```

Expected output:

```
Loading CSV from C:\...\data\raw\dynamic_supply_chain_logistics_dataset.csv...
Connecting to legacy MSSQL Database...
Ingesting into TBL_SC_FLEET_HIST_RAW. This may take a minute...
✅ Legacy data ingestion complete!
```

Check the result in VS Code:

1. Open the SQL Server extension and add a connection.
2. Use server `localhost`, port `1433`, user `sa`, and your `sa` password.
3. Set "Trust server certificate" to on and "Encrypt" to Optional.
4. Run this query:

```sql
SELECT COUNT(*) AS total_rows FROM dbo.TBL_SC_FLEET_HIST_RAW;
```

Expected output:

```
total_rows
----------
32065
```

### Step 6: Add the security layer

**Why:** An LLM can write a bad query.
The database must block that query, so you do not depend on the LLM to behave.

Make `scripts/setup_security_and_view.sql`.
Change `AgentPassword2026!` to the value of `SQL_AGENT_PASSWORD` in your `.env`.

```sql
-- 1. A schema for the AI views
CREATE SCHEMA FDE_VIEWS;
GO

-- 2. A view that changes the legacy names into clear names
CREATE VIEW FDE_VIEWS.VW_ACTIVE_FLEET AS
SELECT
    TS_UTC AS [Timestamp],
    V_LAT AS [Latitude],
    V_LON AS [Longitude],
    CAST(IOT_TEMP_VAL_C AS FLOAT) AS [Current_Temperature_C],
    CGO_COND_CD AS [Cargo_Condition_Code],
    RISK_CLS_TXT AS [Risk_Classification],
    DELAY_PROB_DEC AS [Delay_Probability],
    PRT_CNG_LVL AS [Port_Congestion_Level],
    RT_RSK_IDX AS [Route_Risk_Index]
FROM dbo.TBL_SC_FLEET_HIST_RAW;
GO

-- 3. A login for the AI agent
CREATE LOGIN USR_FDE_RO WITH PASSWORD = 'AgentPassword2026!';
CREATE USER USR_FDE_RO FOR LOGIN USR_FDE_RO;
GO

-- 4. Read access to the view only. Block all other access.
GRANT SELECT ON FDE_VIEWS.VW_ACTIVE_FLEET TO USR_FDE_RO;
DENY SELECT ON dbo.TBL_SC_FLEET_HIST_RAW TO USR_FDE_RO;
DENY INSERT, UPDATE, DELETE, ALTER ON SCHEMA::dbo TO USR_FDE_RO;
GO
```

Key points:

- The view is a **semantic layer**. The LLM sees `Current_Temperature_C`, not `IOT_TEMP_VAL_C`. Clear names help the LLM write correct SQL.
- `GO` works only in SQL tools such as VS Code. It does not work in Python.

Run the script in VS Code with the `sa` connection.

Expected output:

```
Commands completed successfully.
```

Now make a second connection in VS Code with user `USR_FDE_RO` and the agent password.
Run these two tests:

```sql
-- Test 1: This query must work.
SELECT TOP 2 Latitude, Longitude, Current_Temperature_C FROM FDE_VIEWS.VW_ACTIVE_FLEET;
```

Expected output:

```
Latitude            Longitude           Current_Temperature_C
------------------  ------------------  ---------------------
40.375568475194925  -77.0143177425258   0.5743997381982293
33.50781833905297   -117.03690240506452 -9.753493205204748
```

```sql
-- Test 2: This query must fail.
SELECT TOP 5 * FROM dbo.TBL_SC_FLEET_HIST_RAW;
```

Expected output:

```
Msg 229, Level 14, State 5
The SELECT permission was denied on the object 'TBL_SC_FLEET_HIST_RAW', database 'master', schema 'dbo'.
```

If Test 2 fails with this error, the security layer works.

### Step 7: Make the audit log table

**Why:** A manager must be able to see what the AI did and why.

Run this SQL with the `sa` connection:

```sql
CREATE TABLE FDE_VIEWS.AgentAuditLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    Timestamp DATETIME DEFAULT GETDATE(),
    SessionID VARCHAR(50),
    NodeExecuted VARCHAR(50),
    ToolName VARCHAR(100),
    Content NVARCHAR(MAX)  -- NVARCHAR(MAX) holds long JSON and long LLM answers
);

GRANT INSERT ON FDE_VIEWS.AgentAuditLog TO USR_FDE_RO;
```

The agent can add rows to this table.
It cannot read, change, or delete the rows, so it cannot hide what it did.

Expected output:

```
Commands completed successfully.
```

### Step 8: Load the SOP into Pinecone

**Why:** The LLM does not know your company rules.
This step stores the rules so that the agent can search them by meaning.
This method is called RAG (Retrieval-Augmented Generation).

#### 8a. Write the SOP

Make `data/policy/Cold_Chain_Incident_SOP_v2.md`.
This is a short sample. In a real company, you put the real SOP files here.

```markdown
# Southern California Logistics Operations
## Standard Operating Procedure (SOP): Cold-Chain & Transit Anomalies
**Version:** 2.4 | **Effective Date:** Jan 2021 | **Confidentiality:** Internal Operations Only

### 1. Temperature Control & Spoilage Prevention (Cold-Chain)
All refrigerated fleets must maintain strict IoT temperature compliance to prevent cargo spoilage.
* **Fresh Perishables:** IoT temperature must remain between 0.0°C and 4.0°C.
* **Critical Breach:** If `IOT_TEMP_VAL_C` exceeds **4.0°C**, an immediate cold-chain breach is declared.
* **Mitigation Protocol:** The dispatcher must immediately contact the driver to restart the auxiliary cooling unit. If ETA delay is greater than 1 hour, divert the vehicle to the nearest emergency cold-storage facility.

### 2. Route Congestion & Diversion Tactics
* **Port of Long Beach / LA:** If port congestion level (`PRT_CNG_LVL`) exceeds a severity index of **7.0**, standard routing is suspended.
* **Mitigation Protocol:** Do not hold freight at the port. Divert all active shipments to the **Inland Empire Overflow Depot (San Bernardino)** for cross-docking.

### 3. Risk Classification Triggers
Any shipment classified as **"High Risk"** (`RISK_CLS_TXT` = High Risk) combined with a delay probability (`DELAY_PROB_DEC`) greater than **0.65** must be escalated to the Tier 2 Logistics Manager.
```

#### 8b. Write the ingestion script

This is a minimal version of `scripts/ingest_sop_pinecone.py`.
It loads one Markdown file with the local embedding model.

```python
import os
from pathlib import Path
from dotenv import load_dotenv
from pinecone import Pinecone, ServerlessSpec
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_pinecone import PineconeVectorStore
from langchain_text_splitters import MarkdownHeaderTextSplitter, RecursiveCharacterTextSplitter

project_root = Path(__file__).resolve().parent.parent
load_dotenv(project_root / ".env")

INDEX_NAME = "fde-sop-index-local"
DIMENSION = 1024  # BAAI/bge-m3 makes vectors with 1024 numbers

embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-m3", model_kwargs={"device": "cpu"})

# Make the index one time only.
pc = Pinecone(api_key=os.getenv("PINECONE_API_KEY"))
if INDEX_NAME not in pc.list_indexes().names():
    print(f"Creating index {INDEX_NAME}...")
    pc.create_index(
        name=INDEX_NAME,
        dimension=DIMENSION,
        metric="cosine",
        spec=ServerlessSpec(cloud="aws", region="us-east-1"),
    )

# Cut the SOP at each heading, then cut long sections into 600-character chunks.
sop_path = project_root / "data" / "policy" / "Cold_Chain_Incident_SOP_v2.md"
md_splitter = MarkdownHeaderTextSplitter([("#", "Header_1"), ("##", "Header_2"), ("###", "Header_3")])
text_splitter = RecursiveCharacterTextSplitter(chunk_size=600, chunk_overlap=60)
chunks = text_splitter.split_documents(md_splitter.split_text(sop_path.read_text(encoding="utf-8")))

for chunk in chunks:
    chunk.metadata["source_file"] = sop_path.name
    chunk.metadata["file_format"] = "MD"

# Fixed IDs: a second run replaces the chunks. It does not add copies.
ids = [f"{sop_path.name}-chunk-{i}" for i in range(len(chunks))]
PineconeVectorStore(index_name=INDEX_NAME, embedding=embeddings).add_documents(chunks, ids=ids)
print(f"Uploaded {len(chunks)} chunks to {INDEX_NAME}.")
```

Key points:

- **Why cut at headings?** Each SOP rule stays in one chunk. A search for "fresh perishables" then returns the full temperature rule.
- **Why `chunk_overlap=60`?** If a long section is cut, the next chunk repeats the last 60 characters. Thus a sentence is not lost at the cut.
- **Why the metadata?** The agent shows `source_file` in its answer, so the user knows which document gave the rule.

The full script in the repo adds these features:

- It reads `.md`, `.txt`, `.pdf`, `.csv`, and `.xlsx` files.
- It keeps an MD5 hash of each file in `data/cache/`. It skips files that did not change.
- It removes the chunks of deleted files from Pinecone.
- It uses OpenAI embeddings and the index `fde-sop-index-openai` (1536 numbers) if `Embeddings_model=OPENAI`.

Run the full script:

```bash
python scripts/ingest_sop_pinecone.py
```

Expected output on the first run:

```
🤗 Mode: Local Fallback Settings Activated. Launching [BAAI/bge-m3] (1024 Dim)...
Connecting to Pinecone Index Target: [fde-sop-index-local]...
Creating isolated target index: fde-sop-index-local (1024 Dim)...
Found 1 policy file(s) in C:\...\data\policy...
🔄 Processing updates: Cold_Chain_Incident_SOP_v2.md...
  📤 Upserting 4 chunk(s) in batches of 100...
✅ Ingestion & cache update complete.
```

The first run also downloads the `bge-m3` model (about 2 GB).

Expected output on the second run:

```
✨ Skipped (Unchanged): Cold_Chain_Incident_SOP_v2.md
🌴 Index is already up-to-date.
```

### Step 9: Write the tools

**Why:** A tool is a Python function that the LLM can call.
The LLM cannot connect to a database or the internet by itself.
The tools do this work for it.

Make `src/agent_tools.py`.
This version is the same as the repo file, except for one item.
The repo file also caches the embedding model in Streamlit, so the model loads one time only.

```python
import os
import urllib
import requests
from pathlib import Path
from dotenv import load_dotenv
from sqlalchemy import create_engine, text
from langchain_core.tools import tool
from langchain_huggingface import HuggingFaceEmbeddings
from langchain_pinecone import PineconeVectorStore

project_root = Path(__file__).resolve().parent.parent
load_dotenv(project_root / ".env")

if not os.getenv("PINECONE_API_KEY"):
    raise ValueError("CRITICAL: Ensure PINECONE_API_KEY is present in your active .env profile.")

# Use the same model and index as Step 8, or the search finds nothing.
embeddings = HuggingFaceEmbeddings(model_name="BAAI/bge-m3", model_kwargs={"device": "cpu"})
vector_store = PineconeVectorStore(index_name="fde-sop-index-local", embedding=embeddings)
retriever = vector_store.as_retriever(search_kwargs={"k": 2})  # return the 2 best chunks

# The agent connects with the read-only login, never with "sa".
connection_string = (
    "DRIVER={ODBC Driver 18 for SQL Server};"
    f"SERVER={os.getenv('SQL_SERVER_HOST', 'localhost')},{os.getenv('SQL_SERVER_PORT', '1433')};"
    "DATABASE=master;"
    f"UID={os.getenv('SQL_AGENT_USER', 'USR_FDE_RO')};"
    f"PWD={os.getenv('SQL_AGENT_PASSWORD')};"
    "Encrypt=no;"
    "TrustServerCertificate=yes;"
)
engine = create_engine(f"mssql+pyodbc:///?odbc_connect={urllib.parse.quote_plus(connection_string)}")


@tool
def query_telemetry_db(sql_query: str) -> str:
    """
    Executes a SQL SELECT query against the FDE_VIEWS.VW_ACTIVE_FLEET view.
    Columns available:
    Timestamp, Latitude, Longitude, Current_Temperature_C, Cargo_Condition_Code,
    Risk_Classification, Delay_Probability, Port_Congestion_Level, Route_Risk_Index.
    Always write standard T-SQL queries.
    """
    if not sql_query.strip().upper().startswith("SELECT"):
        return "SECURITY BLOCK: Only SELECT operations are authorized on this view."
    try:
        with engine.connect() as conn:
            cursor = conn.execute(text(sql_query))
            columns = list(cursor.keys())
            rows = cursor.fetchmany(10)  # never send thousands of rows to the LLM
        if not rows:
            return "No records matched the query criteria."
        output = f"COLUMNS: {', '.join(columns)}\n"
        for row in rows:
            output += str(tuple(row)) + "\n"
        return output
    except Exception as e:
        return f"Database Error: {str(e)}"


@tool
def fetch_corridor_conditions(latitude: float, longitude: float) -> str:
    """
    Fetches real-time weather and corridor conditions from a live REST API for given GPS coordinates.
    Provides temperature, wind speed, and computed corridor congestion index.
    """
    try:
        url = f"https://api.open-meteo.com/v1/forecast?latitude={latitude}&longitude={longitude}&current_weather=true"
        response = requests.get(url, timeout=6)
        response.raise_for_status()
        payload = response.json().get("current_weather", {})
        temp = payload.get("temperature", "N/A")
        wind = payload.get("windspeed", 0.0)

        # Demo rule: strong wind means a high transit risk.
        congestion_index = 8.5 if wind > 10.0 else 2.5
        status_note = "High Transit Disruption" if wind > 10.0 else "Corridor Normal"

        return (
            f"--- LIVE CORRIDOR TELEMETRY ---\n"
            f"Target GPS: {latitude}, {longitude}\n"
            f"External Temp: {temp}°C | Wind Speed: {wind} km/h\n"
            f"Corridor Risk: {status_note} (Congestion Index: {congestion_index}/10)\n"
            f"-------------------------------"
        )
    except Exception as e:
        return f"Corridor API Communication Failure: {str(e)}"


@tool
def search_compliance_sop(query: str) -> str:
    """
    Searches enterprise Standard Operating Procedures (SOPs) indexed in the Pinecone Vector DB.
    Use this to retrieve regulatory thresholds, cold-chain breach mitigations, and rerouting rules.
    """
    try:
        matched_docs = retriever.invoke(query)
        if not matched_docs:
            return "No matching compliance clauses found."
        context = "\n\n".join(
            f"[Source: {doc.metadata.get('source_file', 'SOP')} | Format: {doc.metadata.get('file_format', 'RAW')}]\n{doc.page_content}"
            for doc in matched_docs
        )
        return f"--- COMPLIANCE SOP CONTEXT ---\n{context}\n------------------------------"
    except Exception as e:
        return f"Vector Store Retrieval Error: {str(e)}"


if __name__ == "__main__":
    print("\n--- Testing Tool 1: SQL Telemetry View ---")
    print(query_telemetry_db.invoke("SELECT TOP 2 Latitude, Longitude, Current_Temperature_C FROM FDE_VIEWS.VW_ACTIVE_FLEET"))

    print("\n--- Testing Tool 2: Live Corridor API ---")
    print(fetch_corridor_conditions.invoke({"latitude": 33.77, "longitude": -118.19}))

    print("\n--- Testing Tool 3: Pinecone Vector Retrieval ---")
    print(search_compliance_sop.invoke("What are the temperature rules for fresh perishables?"))
```

Key points:

- **The docstring is the instruction for the LLM.** The LLM reads it to decide when to call the tool. The column list in `query_telemetry_db` tells the LLM how to write the SQL. If you change the view, change this docstring too.
- **The tools return error text. They do not stop with an exception.** The LLM reads the error and can try again with a better query.
- **Two safety checks for SQL.** The Python code blocks queries that do not start with `SELECT`. The database login (Step 6) blocks all other access.

Test each tool alone before you connect the LLM:

```bash
python src/agent_tools.py
```

Expected output:

```
--- Testing Tool 1: SQL Telemetry View ---
COLUMNS: Latitude, Longitude, Current_Temperature_C
(40.375568475194925, -77.0143177425258, 0.5743997381982293)
(33.50781833905297, -117.03690240506452, -9.753493205204748)

--- Testing Tool 2: Live Corridor API ---
--- LIVE CORRIDOR TELEMETRY ---
Target GPS: 33.77, -118.19
External Temp: 21.4°C | Wind Speed: 7.6 km/h
Corridor Risk: Corridor Normal (Congestion Index: 2.5/10)
-------------------------------

--- Testing Tool 3: Pinecone Vector Retrieval ---
--- COMPLIANCE SOP CONTEXT ---
[Source: Cold_Chain_Incident_SOP_v2.md | Format: MD]
All refrigerated fleets must maintain strict IoT temperature compliance to prevent cargo spoilage.
* **Fresh Perishables:** IoT temperature must remain between 0.0°C and 4.0°C.
...
------------------------------
```

If one tool fails, fix it now.
It is much harder to find the fault after the LLM is connected.

### Step 10: Write the system prompt

**Why:** The system prompt tells the LLM its role, the tool order, and the report format.
A fixed format makes each answer easy to read.

Make `src/prompts/system_prompt.txt`:

```text
You are a Senior Data Scientist for a logistics company.
Your core objective is to allow business stakeholders to "chat with their data" regarding cold-chain anomalies and route disruptions.

CRITICAL OPERATIONAL FLOW:
1. Always use query_telemetry_db to check the database telemetry first.
2. Use fetch_corridor_conditions to check external hazards.
3. Use search_compliance_sop to query the Pinecone SOP database.

BUSINESS OUTPUT FORMAT:
You must analyze the data and present your final response to the business user using this exact structure.
Be concise, analytical, and action-oriented. Do not output raw JSON or tool data.

### 1. Executive Summary
* [1-2 sentences summarizing the core anomaly or situation.]
* [State the immediate operational risk.]

### 2. Telemetry & Environment Analysis
| Location (Lat/Lon) | Current Temp | Cargo Risk | Weather / Congestion |
| :--- | :--- | :--- | :--- |
| [Data] | [Data] | [Data] | [Data] |

*Analysis:* [1 brief sentence explaining if the environment or the equipment is causing the anomaly.]

### 3. Required Action Plan
1. **[Immediate Step 1]**: [Action derived from SOP]
2. **[Immediate Step 2]**: [Action derived from SOP]

*SOP Compliance Citation:* [Cite the exact document and rule driving this action plan]
```

The prompt is in a separate file, so a non-developer can change it with no code change.

### Step 11: Build the agent

**Why:** The agent decides which tools to call, and in which order.
LangGraph runs a loop: the LLM thinks, a tool runs, and the LLM thinks again.

Make `src/orchestrator.py`:

```python
import os
import sys
from pathlib import Path
from typing import Annotated, TypedDict
from dotenv import load_dotenv
from langchain_core.messages import BaseMessage, SystemMessage
from langgraph.graph import StateGraph, START
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.checkpoint.memory import MemorySaver

project_root = Path(__file__).resolve().parent.parent
if str(project_root) not in sys.path:
    sys.path.insert(0, str(project_root))  # lets "from src..." work from any folder

from src.agent_tools import query_telemetry_db, fetch_corridor_conditions, search_compliance_sop

load_dotenv(project_root / ".env")


# 1. The state: the list of messages. add_messages appends new messages to the list.
class AgentState(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


# 2. Select the LLM from .env.
AGENT_LLM_SETTING = os.getenv("Agent_llm", "OLLAMA").strip().upper()

if AGENT_LLM_SETTING == "OPENAI":
    print("🤖 Brain Mode: Utilizing Cloud OpenAI Reasoner (gpt-4o)...")
    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI(model="gpt-4o", temperature=0)
elif AGENT_LLM_SETTING == "DEEPSEEK":
    print("🐳 Brain Mode: Utilizing Flagship DeepSeek Cloud Reasoner...")
    from langchain_openai import ChatOpenAI
    llm = ChatOpenAI(
        model="deepseek-v4-flash",
        temperature=0,
        openai_api_key=os.getenv("DEEPSEEK_API_KEY"),
        base_url="https://api.deepseek.com",  # DeepSeek uses the OpenAI API format
        max_tokens=2048,
    )
else:
    print("🤗 Brain Mode: Local Fallback Activated. Binding Local Ollama (qwen2.5:7b)...")
    from langchain_community.chat_models import ChatOllama
    llm = ChatOllama(model="qwen2.5:7b", temperature=0, num_predict=1024)

# 3. Tell the LLM which tools exist. It reads their names, arguments, and docstrings.
fde_tools = [query_telemetry_db, fetch_corridor_conditions, search_compliance_sop]
llm_with_tools = llm.bind_tools(fde_tools)


# 4. The reasoner node: one LLM call.
def reasoning_node(state: AgentState):
    response = llm_with_tools.invoke(state["messages"])
    return {"messages": [response]}


# 5. The graph: START -> reasoner -> (tools -> reasoner)* -> END
print("⚙️ Compiling LangGraph FDE Orchestrator...")
graph_builder = StateGraph(AgentState)
graph_builder.add_node("reasoner", reasoning_node)
graph_builder.add_node("tools", ToolNode(fde_tools))
graph_builder.add_edge(START, "reasoner")
graph_builder.add_conditional_edges("reasoner", tools_condition)  # tool call -> "tools", else END
graph_builder.add_edge("tools", "reasoner")

fde_agent = graph_builder.compile(checkpointer=MemorySaver())  # keeps the chat per thread_id


# 6. A terminal chat for tests.
if __name__ == "__main__":
    system_instructions = (project_root / "src" / "prompts" / "system_prompt.txt").read_text(encoding="utf-8")
    thread_config = {"configurable": {"thread_id": "production_test_1"}}
    fde_agent.invoke({"messages": [SystemMessage(content=system_instructions)]}, config=thread_config)

    while True:
        user_input = input("\nDispatcher > ")
        if user_input.lower() in ["exit", "quit"]:
            break
        events = fde_agent.stream({"messages": [("user", user_input)]}, config=thread_config, stream_mode="updates")
        for event in events:
            for node_name, node_state in event.items():
                if node_name == "tools":
                    print("   [System] 🔄 Retrieving external data elements via ToolNode...")
                elif node_name == "reasoner":
                    latest_msg = node_state["messages"][-1]
                    if latest_msg.content:
                        print(f"\n🤖 FDE Agent:\n{latest_msg.content}")
```

Key points:

- **`tools_condition`** reads the last LLM message. If the message asks for a tool, the graph goes to `tools`. If not, the graph stops.
- **`MemorySaver`** stores each conversation by `thread_id`. The agent remembers earlier questions in the same thread.
- **`temperature=0`** makes the LLM give the most likely answer. This is better for facts and SQL.
- **`stream_mode="updates"`** returns each node result when it is ready. The user sees progress, and does not wait in silence.

Start the terminal chat:

```bash
python src/orchestrator.py
```

Expected output at start:

```
🤗 Mode: Connecting to Local Fallback [BAAI/bge-m3] Index (1024 Dim Space)...
🤗 Brain Mode: Local Fallback Activated. Binding Local Ollama (qwen2.5:7b)...
⚙️ Compiling LangGraph FDE Orchestrator...

=======================================================
🚀 FDE Supply Chain Orchestrator State Machine Online
   Configured Execution: [LLM: OLLAMA] -> [Embeddings: LOCAL]
=======================================================

Dispatcher >
```

#### Test 1: The "domino effect" test

The agent must use all 3 tools.

```
Dispatcher > Find any active shipments near Los Angeles (Latitude ~33.8, Longitude ~-118.1). Check the local weather there, and tell me if the current cargo temperature violates the SOP for fresh perishables.
```

The LLM writes a SQL query similar to this one:

```sql
SELECT TOP 5 Timestamp, Latitude, Longitude, Current_Temperature_C, Risk_Classification
FROM FDE_VIEWS.VW_ACTIVE_FLEET
WHERE Latitude BETWEEN 33.5 AND 34.1 AND Longitude BETWEEN -118.4 AND -117.8
```

Expected output (example):

```
   [System] 🔄 Retrieving external data elements via ToolNode...
   [System] 🔄 Retrieving external data elements via ToolNode...
   [System] 🔄 Retrieving external data elements via ToolNode...

🤖 FDE Agent:
### 1. Executive Summary
* A shipment near Los Angeles (33.87, -118.06) reports a cargo temperature of -8.29°C, classified as High Risk.
* This is outside the 0.0°C to 4.0°C range for fresh perishables, so the cargo is at risk of freeze damage.

### 2. Telemetry & Environment Analysis
| Location (Lat/Lon) | Current Temp | Cargo Risk | Weather / Congestion |
| :--- | :--- | :--- | :--- |
| 33.87, -118.06 | -8.29°C | High Risk | 21.4°C, wind 7.6 km/h, Corridor Normal |

*Analysis:* The outside weather is mild, so the cooling equipment is the likely cause of the anomaly.

### 3. Required Action Plan
1. **Contact the driver**: Restart the auxiliary cooling unit and verify the setpoint.
2. **Divert if delayed**: If the ETA delay is greater than 1 hour, divert to the nearest emergency cold-storage facility.

*SOP Compliance Citation:* Cold_Chain_Incident_SOP_v2.md, Section 1 "Temperature Control & Spoilage Prevention".
```

The dataset has 13 readings near Los Angeles. The SQL tool returns a maximum of 10 rows.

#### Test 2: The "restraint" test

The agent must not query the database, because the question is about rules only.

```
Dispatcher > I'm a new dispatcher on the night shift. Can you quickly explain the difference between a Tier 1 and Tier 2 escalation?
```

Expected behavior:

- The agent calls `search_compliance_sop` only.
- The answer says that a "High Risk" shipment with a delay probability greater than 0.65 goes to the Tier 2 Logistics Manager.
- The sample SOP does not define Tier 1. A good agent says this, and does not invent a rule.

Type `exit` to stop.

### Step 12: Build the web app

**Why:** A dispatcher needs a chat page, not a terminal.
Streamlit makes a web page from a Python script.

Make `.streamlit/config.toml`:

```toml
[server]
fileWatcherType = "none"
```

This setting stops the Streamlit file watcher. The watcher has problems with the `torch` package.

This is a minimal version of `src/ui.py`:

```python
import sys
import uuid
from pathlib import Path
import streamlit as st
from langchain_core.messages import HumanMessage, SystemMessage

project_root = Path(__file__).resolve().parent.parent
sys.path.insert(0, str(project_root))
from src.orchestrator import fde_agent

st.set_page_config(page_title="FDE Dispatch Console", page_icon="🧊", layout="wide")
st.title("Cold-Chain Incident Control Dashboard")

# Streamlit runs the full script again after each click.
# st.session_state keeps values between these runs.
if "thread_id" not in st.session_state:
    st.session_state.thread_id = str(uuid.uuid4())  # one thread for each browser tab
    st.session_state.history = []

config = {"configurable": {"thread_id": st.session_state.thread_id}}

for role, content in st.session_state.history:
    st.chat_message(role).markdown(content)

if question := st.chat_input("Ask about trucks, temperatures, routes, or SOPs"):
    st.chat_message("user").markdown(question)

    messages = [HumanMessage(content=question)]
    if not st.session_state.history:
        # Send the system prompt one time, at the start of the thread.
        prompt = (project_root / "src" / "prompts" / "system_prompt.txt").read_text(encoding="utf-8")
        messages.insert(0, SystemMessage(content=prompt))
    st.session_state.history.append(("user", question))

    answer = ""
    with st.status("Agent is working...") as status:
        for event in fde_agent.stream({"messages": messages}, config=config, stream_mode="updates"):
            for node_name, node_state in event.items():
                msg = node_state["messages"][-1]
                if node_name == "reasoner" and msg.tool_calls:
                    for call in msg.tool_calls:
                        st.write(f"Tool: `{call['name']}`")
                        st.json(call["args"])  # show the input that the LLM made
                elif node_name == "reasoner" and msg.content:
                    answer = msg.content
        status.update(label="Done", state="complete")

    st.chat_message("assistant").markdown(answer)
    st.session_state.history.append(("assistant", answer))
```

The full `src/ui.py` in the repo adds these features:

- It shows the raw output of each tool in a box that you can expand.
- It writes each tool call and each answer to `FDE_VIEWS.AgentAuditLog`.
- It has a second page, **Security & Audit Logs**, that shows the audit table after an admin login.
- It has a button that clears the chat and starts a new thread.

Start the app:

```bash
streamlit run src/ui.py
```

Expected output:

```
  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://192.168.x.x:8501
```

Open http://localhost:8501 and ask the Test 1 question from Step 11.
With the full repo version of `src/ui.py`, you see this sequence on the page:

1. A status box with the text "Initializing Core Reasoner Node...".
2. One line for each tool, for example "⚡ Intent Recognized: `query_telemetry_db`", with the SQL inside.
3. The final report in the same format as Step 11.

To check the audit log, run this query with the `sa` connection:

```sql
SELECT TOP 5 Timestamp, NodeExecuted, ToolName FROM FDE_VIEWS.AgentAuditLog ORDER BY LogID DESC;
```

Expected output (example):

```
Timestamp                NodeExecuted     ToolName
-----------------------  ---------------  -------------------------
2026-10-07 09:15:42.120  reasoner_final   LLM Text Synthesis
2026-10-07 09:15:31.877  tools            search_compliance_sop
2026-10-07 09:15:30.410  reasoner         search_compliance_sop
2026-10-07 09:15:29.902  tools            fetch_corridor_conditions
2026-10-07 09:15:28.655  reasoner         fetch_corridor_conditions
```

## 7. Deploy to AWS EC2

### Set up the server one time

1. Start an EC2 instance: Ubuntu, `c7i-flex.large`, 30 GB storage.
2. In the security group, open port 8501 for the app. Open port 1433 only if the database is on a different server.
3. Install Docker and start SQL Server as in Step 4. Add `-v mssql_data:/var/opt/mssql` to keep the data after a restart.
4. Install the app:
   ```bash
   sudo apt update && sudo apt install -y python3-pip python3-venv git
   cd /home/ubuntu
   git clone https://github.com/SyakeerRahman/cold-chain-logistics-FDE-Project.git
   cd cold-chain-logistics-FDE-Project
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```
5. Install the ODBC driver:
   ```bash
   curl -fsSL https://packages.microsoft.com/keys/microsoft.asc | sudo gpg --dearmor -o /usr/share/keyrings/microsoft-prod.gpg
   curl -fsSL https://packages.microsoft.com/config/ubuntu/$(lsb_release -rs)/prod.list | sudo tee /etc/apt/sources.list.d/mssql-release.list
   sudo apt-get update
   sudo ACCEPT_EULA=Y apt-get install -y msodbcsql18 unixodbc-dev
   ```
6. Copy your `.env` file to the server.
7. Do Steps 5 to 8 on the server, or point `SQL_SERVER_HOST` to your database.
8. Make the file `/etc/systemd/system/streamlit.service`:
   ```ini
   [Unit]
   Description=Streamlit Cold-Chain Dispatch Console
   After=network.target

   [Service]
   User=ubuntu
   WorkingDirectory=/home/ubuntu/cold-chain-logistics-FDE-Project
   ExecStart=/home/ubuntu/cold-chain-logistics-FDE-Project/venv/bin/streamlit run src/ui.py --server.port=8501 --server.address=0.0.0.0
   Restart=always

   [Install]
   WantedBy=multi-user.target
   ```
   **Why systemd:** It starts the app when the server starts. It also starts the app again if it stops.
9. Start the service:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable streamlit
   sudo systemctl start streamlit
   sudo systemctl status streamlit
   ```
   Expected output (part):
   ```
   ● streamlit.service - Streamlit Cold-Chain Dispatch Console
        Active: active (running)
   ```

Open `http://<EC2 public IP>:8501`.

### Deploy new code

1. In GitHub, make the environment `FDE-YT-Cold`.
2. Add the secrets `EC2_HOST`, `EC2_USER`, and `EC2_SSH_KEY` to that environment.
3. Go to **Actions > Enterprise Manual Deploy > Run workflow**.

The workflow in [.github/workflows/deploy.yml](.github/workflows/deploy.yml) does these steps:

```yaml
script: |
  cd /home/ubuntu/cold-chain-logistics-FDE-Project
  git pull origin main
  source venv/bin/activate
  sudo systemctl restart streamlit
```

It does not install new packages.
After you change `requirements.txt`, run `pip install -r requirements.txt` on the server.

## 8. Troubleshooting

| Symptom | Cause | Fix |
| :--- | :--- | :--- |
| `docker ps` does not show `legacy-mssql` | The `sa` password is too weak, so SQL Server stopped. | Run `docker logs legacy-mssql`. Remove the container with `docker rm legacy-mssql`, then start it again with a stronger password. |
| `Data source name not found and no default driver specified` | ODBC Driver 18 is not installed. | Install ODBC Driver 18 for SQL Server (Step 1). |
| `Login failed for user 'USR_FDE_RO'` | The password in the SQL script is not the same as `SQL_AGENT_PASSWORD`. | Run `ALTER LOGIN USR_FDE_RO WITH PASSWORD = '...';` as `sa`. |
| `CRITICAL: Ensure PINECONE_API_KEY is present in your active .env profile.` | `.env` is missing, or it has no Pinecone key. | Add the key to `.env` in the project root. |
| A Pinecone error about vector dimension | `Embeddings_model` changed after ingestion. | Use the same value for Step 8 and the app. Delete `data/cache/ingestion_hash_cache.json` and run Step 8 again. |
| `search_compliance_sop` returns `No matching compliance clauses found.` | The index is empty. | Run Step 8. |
| A connection error to Ollama | The Ollama server is not running. | Start the Ollama app, or run `ollama serve`. |
| The agent answers with no tool calls | Small local models sometimes skip tools. | Set `Agent_llm=OPENAI` or `Agent_llm=DEEPSEEK`. |

## 9. Known limits

- The full `src/ui.py` in the repo does not send `src/prompts/system_prompt.txt` to the LLM. Thus, its answers can have a different format. The minimal version in Step 12 sends it.
- The agent memory (`MemorySaver`) is in RAM. A restart deletes all chat history.
- The SQL tool returns a maximum of 10 rows for each query.
- The weather tool calculates the congestion index from the wind speed only. It is a demo value.
- `src/agent_tools.py` connects to Pinecone when Python imports it. If `PINECONE_API_KEY` is missing, the app stops at start.
