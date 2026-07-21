# Lab 9: GitOps Pipeline & Environment Isolation in Snowflake

## Overview & Objectives

In modern data engineering, manual SQL execution in web UI consoles leads to untracked schema drift, unversioned code, and deployment risk. **GitOps** brings software engineering best practices—version control, peer review, and automated environment isolation—directly into data platform workflows.

In this lab, you will transition from executing ad-hoc queries in Snowflake to managing and deploying a modular data pipeline directly from GitHub using Snowflake’s native **Git Integration** capabilities.

By the end of this lab, you will be able to:
1. Store GitHub personal access tokens (PAT) securely inside Snowflake using `SECRET` objects.
2. Link external Git repositories to Snowflake as native `GIT REPOSITORY` stages.
3. Parse semi-structured JSON payloads stored in Snowflake `VARIANT` columns using `LATERAL FLATTEN`.
4. Execute version-controlled SQL transformation scripts from Git using `EXECUTE IMMEDIATE FROM`.
5. Implement Git-based environment isolation (`dev` vs. `prod`) across Snowflake schemas.

---

## Prerequisites

* Active access to the course Snowflake account (`DS5111_DB` database).
* A GitHub account.
* Local Git installation and your text editor of choice (e.g., VS Code, Vim).

---

## Architecture & Data Context

In Lab 8, you containerized an ingestion pipeline streaming raw YouTube transcript payloads into Snowflake. These records currently reside in `RAW_TRANSCRIPTS` inside `DS5111_DB`.

Each row in `RAW_TRANSCRIPTS` contains two primary columns:
* `JSON_PAYLOAD` (`VARIANT`): A semi-structured JSON record formatted as:
  ```json
  {
    "video_id": "dbUIjFXIpis",
    "cleaned_text": "In this lecture we explore advanced deep learning ablation models using PyTorch.",
    "tech_terms": ["PyTorch", "Deep Learning"],
    "book_names": ["Clean Code"]
  }
  ```
* `INSERTED_AT` (`TIMESTAMP_NTZ`): Ingestion timestamp.

You will build a 3-tier modular transformation pipeline (`Staging` $\rightarrow$ `Dimension` $\rightarrow$ `Fact`) versioned in Git and executed natively inside Snowflake.

---

## Repository Structure

Your GitHub repository (`ds5111-lab9-gitops`) must adhere to the following layout:

```text
ds5111-pipeline/
├── bin/                 <-- Ingestion scripts (Previous labs)
└── transform/           <-- Transformation pipeline (Lab 9)
    ├── 01_stg_youtube_transcripts.sql
    ├── 02_dim_videos.sql
    ├── 03_fct_entities.sql
    └── orchestrate_pipeline.sql
```

---

## Part 1: GitHub Setup & Secret Configuration

### Step 1.1: Work in your current github repository
1. You will do your work on your existing repository, access it in your preferred method jupyter/vscode/cli/aws-web.

### Step 1.2: Generate a GitHub Personal Access Token (PAT)
1. In GitHub, go to **Settings** $\rightarrow$ **Developer Settings** $\rightarrow$ **Personal Access Tokens** $\rightarrow$ **Tokens (classic)**.
2. Click **Generate new token (classic)**.
3. Name it `Snowflake Git Integration Token`.
4. Select scope: `repo` (Full control of repositories).
5. Generate and **copy the token** immediately.

### Step 1.3: Create Snowflake Secret & Git Repository Stage
Open a new SQL Worksheet in Snowflake and execute the following commands. Replace `<your-computing-id>` and `<your-github-username>` with your actual values:

```sql
-- Set active context
USE DATABASE DS5111_DB;
USE SCHEMA PUBLIC;

-- 1. Create a Snowflake Secret to store your PAT securely
CREATE OR REPLACE SECRET GITHUB_PAT_SECRET
  TYPE = PLAINTEXT_PASSWORD
  USERNAME = '<your-github-username>'
  PASSWORD = '<your-github-personal-access-token>';

-- 2. Create the native Git Repository stage
CREATE OR REPLACE GIT REPOSITORY DS5111_GIT_STAGE
  ORIGIN = 'https://github.com/<your-github-username>/ds5111-lab9-gitops.git'
  API_INTEGRATION = GITHUB_API_INTEGRATION
  GIT_CREDENTIALS = GITHUB_PAT_SECRET;

-- 3. Verify connection and list repo files
ALTER GIT REPOSITORY DS5111_GIT_STAGE FETCH;
SHOW GIT BRANCHES IN DS5111_GIT_STAGE;
LIST @DS5111_GIT_STAGE/branches/main;
```

---

## Part 2: Building the Transformation Pipeline

Locally on your machine, create the four SQL scripts inside the `scripts/` folder.

### File 1: `scripts/01_stg_youtube_transcripts.sql`
Extracts raw attributes from `JSON_PAYLOAD` into explicit relational types:

```sql
-- Step 1: Staging View (JSON Variant Parsing)
CREATE OR REPLACE VIEW STG_YOUTUBE_TRANSCRIPTS AS
SELECT
    JSON_PAYLOAD:video_id::STRING AS VIDEO_ID,
    JSON_PAYLOAD:cleaned_text::STRING AS CLEANED_TEXT,
    JSON_PAYLOAD:tech_terms AS TECH_TERMS_ARRAY,
    JSON_PAYLOAD:book_names AS BOOK_NAMES_ARRAY,
    INSERTED_AT
FROM RAW_TRANSCRIPTS;
```

### File 2: `scripts/02_dim_videos.sql`
Creates a video metadata dimension table with calculated text metrics:

```sql
-- Step 2: Dimension Table (Metrics & Aggregations)
CREATE OR REPLACE TABLE DIM_VIDEOS AS
SELECT
    VIDEO_ID,
    CLEANED_TEXT,
    ARRAY_SIZE(TECH_TERMS_ARRAY) AS TECH_TERM_COUNT,
    ARRAY_SIZE(BOOK_NAMES_ARRAY) AS BOOK_NAME_COUNT,
    ARRAY_SIZE(SPLIT(CLEANED_TEXT, ' ')) AS WORD_COUNT,
    INSERTED_AT AS PROCESSED_AT
FROM STG_YOUTUBE_TRANSCRIPTS;
```

### File 3: `scripts/03_fct_entities.sql`
Unnests array structures using `LATERAL FLATTEN` into relational fact tables:

```sql
-- Step 3a: Flatten Tech Terms into Fact Table
CREATE OR REPLACE TABLE FCT_TECH_TERMS AS
SELECT
    VIDEO_ID,
    f.value::STRING AS TECH_TERM,
    INSERTED_AT AS PROCESSED_AT
FROM STG_YOUTUBE_TRANSCRIPTS,
LATERAL FLATTEN(input => TECH_TERMS_ARRAY) f;

-- Step 3b: Flatten Book Mentions into Fact Table
CREATE OR REPLACE TABLE FCT_BOOK_MENTIONS AS
SELECT
    VIDEO_ID,
    f.value::STRING AS BOOK_NAME,
    INSERTED_AT AS PROCESSED_AT
FROM STG_YOUTUBE_TRANSCRIPTS,
LATERAL FLATTEN(input => BOOK_NAMES_ARRAY) f;
```

### File 4: `scripts/orchestrate_pipeline.sql`
Master script that chains execution directly from the Git stage:

```sql
-- Master Pipeline Orchestrator
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/main/scripts/01_stg_youtube_transcripts.sql;
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/main/scripts/02_dim_videos.sql;
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/main/scripts/03_fct_entities.sql;
```

---

## Part 3: Commit, Fetch, and First Execution

1. Commit and push your local files to GitHub:
   ```bash
   git add .
   git commit -m "feat: add lab9 transformation pipeline"
   git push origin main
   ```

2. In your Snowflake Worksheet, refresh the Git stage and run the orchestrator script:
   ```sql
   -- Fetch latest commits from GitHub
   ALTER GIT REPOSITORY DS5111_GIT_STAGE FETCH;

   -- Execute pipeline directly from Git
   EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/main/scripts/orchestrate_pipeline.sql;

   -- Verify outputs
   SELECT * FROM DIM_VIDEOS LIMIT 10;
   SELECT * FROM FCT_TECH_TERMS LIMIT 10;
   SELECT * FROM FCT_BOOK_MENTIONS LIMIT 10;
   ```

---

## Part 4: Environment Isolation & Branch Testing

To prevent testing changes directly in production, you will use Git branching and target schema isolation.

### Step 4.1: Create a Feature Branch
In your local terminal:
```bash
git checkout -b feature/add-char-count
```

### Step 4.2: Modify `scripts/02_dim_videos.sql`
Update `scripts/02_dim_videos.sql` to calculate a new column `CHAR_COUNT`:

```sql
-- Step 2: Dimension Table (Updated with CHAR_COUNT)
CREATE OR REPLACE TABLE DIM_VIDEOS AS
SELECT
    VIDEO_ID,
    CLEANED_TEXT,
    ARRAY_SIZE(TECH_TERMS_ARRAY) AS TECH_TERM_COUNT,
    ARRAY_SIZE(BOOK_NAMES_ARRAY) AS BOOK_NAME_COUNT,
    ARRAY_SIZE(SPLIT(CLEANED_TEXT, ' ')) AS WORD_COUNT,
    LENGTH(CLEANED_TEXT) AS CHAR_COUNT,
    INSERTED_AT AS PROCESSED_AT
FROM STG_YOUTUBE_TRANSCRIPTS;
```

Commit and push your feature branch:
```bash
git add scripts/02_dim_videos.sql
git commit -m "feat: add char_count metric to dim_videos"
git push origin feature/add-char-count
```

### Step 4.3: Test Feature Branch in Personal Dev Schema
In Snowflake, switch your context to your personal development schema (`DEV_<your_computing_id>`), fetch the latest repo state, and execute from the feature branch:

```sql
-- Set dev context
CREATE SCHEMA IF NOT EXISTS DEV_<your_computing_id>;
USE SCHEMA DEV_<your_computing_id>;

-- Fetch new branches
ALTER GIT REPOSITORY PUBLIC.DS5111_GIT_STAGE FETCH;

-- Execute from feature branch into DEV schema
EXECUTE IMMEDIATE FROM @PUBLIC.DS5111_GIT_STAGE/branches/feature/add-char-count/scripts/01_stg_youtube_transcripts.sql;
EXECUTE IMMEDIATE FROM @PUBLIC.DS5111_GIT_STAGE/branches/feature/add-char-count/scripts/02_dim_videos.sql;

-- Verify CHAR_COUNT exists in DEV schema
SELECT VIDEO_ID, WORD_COUNT, CHAR_COUNT FROM DIM_VIDEOS;
```

### Step 4.4: Promote to Production
Once verified in `DEV`:
1. Merge `feature/add-char-count` into `main` via GitHub Pull Request or CLI:
   ```bash
   git checkout main
   git merge feature/add-char-count
   git push origin main
   ```
2. In Snowflake, switch context back to `PUBLIC` (or your `PROD` schema), fetch, and re-run `orchestrate_pipeline.sql`:
   ```sql
   USE SCHEMA PUBLIC;
   ALTER GIT REPOSITORY DS5111_GIT_STAGE FETCH;
   EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/main/scripts/orchestrate_pipeline.sql;
   
   -- Confirm updated schema in PROD
   SELECT VIDEO_ID, WORD_COUNT, CHAR_COUNT FROM DIM_VIDEOS;
   ```

---

## Submission Deliverables

1. **GitHub Repository URL:** Link to your accessible `ds5111-lab9-gitops` repository.
2. **Snowflake Execution Screenshot / Query Log:** A screenshot showing successful execution of `EXECUTE IMMEDIATE FROM` and query results from `DIM_VIDEOS` showing `CHAR_COUNT`.
3. **Verification Query Output:** Submit the text output of running:
   ```sql
   SELECT 
       (SELECT COUNT(*) FROM DIM_VIDEOS) AS TOTAL_VIDEOS,
       (SELECT COUNT(*) FROM FCT_TECH_TERMS) AS TOTAL_TECH_TERMS,
       (SELECT COUNT(*) FROM FCT_BOOK_MENTIONS) AS TOTAL_BOOK_MENTIONS;
   ```

---

## Grading Rubric (10 Points Total)

Submissions will be evaluated based on the following criteria:

| Category | Criteria | Points |
| :--- | :--- | :---: |
| **1. Repository & Git Stage Setup** | • Public/accessible GitHub repository created with correct `scripts/` directory structure.<br>• Snowflake `SECRET` and `GIT REPOSITORY` stage successfully configured and fetched without authorization errors. | **2 pts** |
| **2. Transformation Scripts & Logic** | • `01_stg_youtube_transcripts.sql`: Correct extraction and casting from `JSON_PAYLOAD` `VARIANT`.<br>• `02_dim_videos.sql`: Accurate text metric logic (`WORD_COUNT`, array sizes).<br>• `03_fct_entities.sql`: Proper implementation of `LATERAL FLATTEN` to unnest arrays into `FCT_TECH_TERMS` and `FCT_BOOK_MENTIONS`. | **3 pts** |
| **3. GitOps Pipeline Orchestration** | • `orchestrate_pipeline.sql` created and successfully executed using `EXECUTE IMMEDIATE FROM` against the `@DS5111_GIT_STAGE` stage. | **2 pts** |
| **4. Branching & Environment Isolation** | • Demonstrated workflow on `feature/add-char-count` branch targeting `DEV_<computing_id>` schema.<br>• Successful PR/merge into `main` and production execution in `PUBLIC` schema showing `CHAR_COUNT`. | **2 pts** |
| **5. Deliverables & Verification** | • Submission includes valid GitHub URL, execution screenshot/query log, and complete text output from the verification query. | **1 pt** |
| **Total** | | **10 pts** |

---

### Late Policy & Submission Format
* **Submission Deadline:** [Insert Date/Time]
* Submit your GitHub URL, execution screenshots, and verification output via Canvas.
* Code that fails to execute due to uncommitted files or incorrect Git stage paths will receive a deduction under Category 3.
