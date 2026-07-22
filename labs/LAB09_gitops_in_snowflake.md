# Lab 9: GitOps Pipeline & Environment Isolation in Snowflake

## Overview & Objectives

In modern data engineering, manual SQL execution in web UI consoles leads to untracked schema drift, unversioned code, and deployment risk. **GitOps** brings software engineering best practices—version control, peer review, and automated environment isolation—directly into data platform workflows.

In this lab, you will transition from executing ad-hoc queries in Snowflake to managing and deploying a modular data pipeline directly from GitHub using Snowflake’s native **Git Integration** capabilities.

By the end of this lab, you will be able to:
1. Store GitHub personal access tokens (PAT) securely inside Snowflake using `SECRET` objects.
2. Link external Git repositories to Snowflake as native `GIT REPOSITORY` stages.
3. Parse semi-structured JSON payloads stored in Snowflake `VARIANT` columns using `LATERAL FLATTEN`.
4. Execute version-controlled SQL transformation scripts from Git using `EXECUTE IMMEDIATE FROM`.
5. Implement Git-based environment isolation (`dev` vs. base schema) across Snowflake schemas.

---

## Prerequisites

* Active access to the course Snowflake account (`DS5111_DB` database).
* Access to your personal schema (`<UVAID>`).
* A GitHub account.
* Local Git installation and your text editor of choice (e.g., VS Code, Vim).

---

## Architecture & Data Context

In Lab 8, you containerized an ingestion pipeline streaming raw YouTube transcript payloads into Snowflake. These records reside in `RAW_TRANSCRIPTS` inside your primary schema (`DS5111_DB.<UVAID>`).

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

You will build a 3-tier modular transformation pipeline (Staging → Dimension → Fact) versioned in Git and executed natively inside Snowflake.

---

## Repository Structure

Your GitHub repository (`ds5111-pipeline`) must adhere to the following layout:

```text
ds5111-pipeline/
├── bin/            <-- Ingestion scripts (Previous labs)
└── transform/      <-- Transformation pipeline (Lab 9)
    ├── 01_stg_youtube_transcripts.sql
    ├── 02_dim_videos.sql
    ├── 03_fct_entities.sql
    └── orchestrate_pipeline.sql
```

---

## Part 1: GitHub Setup & Secret Configuration

### Step 1.1: Work in Your Current GitHub Repository
You will do your work inside your existing lab repository using your preferred development environment (VS Code, Vim, CLI, etc.).

### Step 1.2: Prepare Your Repository & Authentication Strategy

* **Path A (Default / Recommended): Public Repository**
  If your GitHub repository is **Public**, no Personal Access Token (PAT) or Snowflake secret configuration is required. Snowflake can pull public repositories directly using the account's built-in `github_public_integration`.

* **Path B (Optional / Advanced): Private Repository**
  If your repository is **Private**, generate a GitHub Personal Access Token (PAT) so Snowflake can authenticate:
  1. Go directly to [github.com/settings/tokens](https://github.com/settings/tokens) **OR**:
     * Click your **Profile Picture** (top right corner) → **Settings**.
     * Scroll to the **very bottom** of the left-hand navigation sidebar.
     * Click **Developer settings** → **Personal access tokens** → **Tokens (classic)**.
  2. Click **Generate new token (classic)**.
  3. Name it `Snowflake Git Token`, select the `repo` scope check box, and generate the token.
  4. **Copy the generated token string immediately** (GitHub only shows it once).

---

### Step 1.3: Create the Snowflake Git Repository Stage

Open a SQL Worksheet in Snowflake and execute the setup queries corresponding to your repository visibility choice below. Replace `<UVAID>` with your computing ID (e.g., `TXT1SR`) and `<your-github-username>` with your GitHub username.

#### Path A: Default Setup (Public Repositories)

```sql
-- Set active context to your primary schema
USE DATABASE DS5111_DB;
USE SCHEMA <UVAID>;

-- Create native Git Stage referencing public integration
CREATE OR REPLACE GIT REPOSITORY DS5111_GIT_STAGE
  ORIGIN = 'https://github.com/<your-github-username>/ds5111-pipeline.git'
  API_INTEGRATION = GITHUB_PUBLIC_INTEGRATION;
```

#### Path B: Advanced Setup (Private Repositories with Secret)

```sql
-- Set active context to your primary schema
USE DATABASE DS5111_DB;
USE SCHEMA <UVAID>;

-- 1. Create a Snowflake Secret object to store your PAT securely
CREATE OR REPLACE SECRET GITHUB_PAT_SECRET
  TYPE = PLAINTEXT_PASSWORD
  USERNAME = '<your-github-username>'
  PASSWORD = '<your-github-personal-access-token>';

-- 2. Create native Git Stage using credentials
CREATE OR REPLACE GIT REPOSITORY DS5111_GIT_STAGE
  ORIGIN = 'https://github.com/<your-github-username>/ds5111-pipeline.git'
  API_INTEGRATION = GITHUB_API_INTEGRATION
  GIT_CREDENTIALS = GITHUB_PAT_SECRET;
```

#### Verify the Stage Connection (All Students)

After creating your stage, run the following commands to fetch repository metadata and verify your file tree:

```sql
-- Fetch the latest commits down from GitHub
ALTER GIT REPOSITORY DS5111_GIT_STAGE FETCH;

-- Show tracked branches inside the stage
SHOW GIT BRANCHES IN DS5111_GIT_STAGE;

-- List files inside the main branch
LIST @DS5111_GIT_STAGE/branches/main;
```

---

## Part 2: Building the Transformation Pipeline

### Step 2.1: Create Your Lab Feature Branch (Local Git)

Before authoring any SQL transformation files, create and switch to a dedicated working branch in your local terminal. Do **not** commit directly to `main`.

```bash
# Ensure you are on main and up to date
git checkout main
git pull origin main

# Create and switch to your feature branch for Lab 9
git checkout -b LAB09_gitops_snowflake
```

> 💡 **GitOps Rule:** All initial development and testing happen inside `LAB09_gitops_snowflake` running against your primary schema (`<UVAID>`). In Part 4, you will create a sub-feature branch to test isolated changes in a `DEV_<UVAID>` sandbox.

Locally on your machine, create the four SQL scripts inside the `transform/` folder.

### File 1: `transform/01_stg_youtube_transcripts.sql`
Extracts raw attributes from `JSON_PAYLOAD` into explicit relational types. Notice that we explicitly reference `<UVAID>.RAW_TRANSCRIPTS` so this staging script can safely run from any schema context:

```sql
-- Step 1: Staging View (JSON Variant Parsing)
CREATE OR REPLACE VIEW STG_YOUTUBE_TRANSCRIPTS AS
SELECT
    JSON_PAYLOAD:video_id::STRING AS VIDEO_ID,
    JSON_PAYLOAD:cleaned_text::STRING AS CLEANED_TEXT,
    JSON_PAYLOAD:tech_terms AS TECH_TERMS_ARRAY,
    JSON_PAYLOAD:book_names AS BOOK_NAMES_ARRAY,
    INSERTED_AT
FROM <UVAID>.RAW_TRANSCRIPTS;
```

### File 2: `transform/02_dim_videos.sql`
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

### File 3: `transform/03_fct_entities.sql`
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

### File 4: `transform/orchestrate_pipeline.sql`
Master script that chains execution directly from the Git stage:

```sql
-- Master Pipeline Orchestrator
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/LAB09_gitops_snowflake/transform/01_stg_youtube_transcripts.sql;
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/LAB09_gitops_snowflake/transform/02_dim_videos.sql;
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/LAB09_gitops_snowflake/transform/03_fct_entities.sql;
```

---

## Part 3: Commit, Fetch, and First Execution

1. Commit and push your local files to GitHub:
   ```bash
   git add transform/
   git commit -m "feat: add lab9 transformation pipeline"
   git push origin LAB09_gitops_snowflake
   ```

2. In your Snowflake Worksheet, ensure your context is set to `<UVAID>`, refresh the Git stage, and run the orchestrator script:
   ```sql
   USE DATABASE DS5111_DB;
   USE SCHEMA <UVAID>;

   -- Fetch latest commits from GitHub
   ALTER GIT REPOSITORY DS5111_GIT_STAGE FETCH;

   -- Execute pipeline directly from Git
   EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/LAB09_gitops_snowflake/transform/orchestrate_pipeline.sql;

   -- Verify outputs
   SELECT * FROM DIM_VIDEOS LIMIT 10;
   SELECT * FROM FCT_TECH_TERMS LIMIT 10;
   SELECT * FROM FCT_BOOK_MENTIONS LIMIT 10;
   ```

---

## Part 4: Environment Isolation & Branch Testing

To prevent testing changes directly in your base schema, you will use Git branching and target schema isolation.

### Step 4.1: Create a Feature Branch
In your local terminal, branch off of `LAB09_gitops_snowflake`:
```bash
git checkout -b feature-add-char-count
```

### Step 4.2: Modify `transform/02_dim_videos.sql`
Update `transform/02_dim_videos.sql` to calculate a new column `CHAR_COUNT`:

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
git add transform/02_dim_videos.sql
git commit -m "feat: add char_count metric to dim_videos"
git push origin feature-add-char-count
```

### Step 4.3: Test Feature Branch in Personal Dev Schema
In Snowflake, switch your context to an isolated development schema (`DEV_<UVAID>`), fetch the latest repo state, and execute from the feature branch:

```sql
-- Set dev context
CREATE SCHEMA IF NOT EXISTS DEV_<UVAID>;
USE SCHEMA DEV_<UVAID>;

-- Fetch new branch from Git stage (located in primary schema)
ALTER GIT REPOSITORY <UVAID>.DS5111_GIT_STAGE FETCH;

-- Execute from feature branch into DEV schema
EXECUTE IMMEDIATE FROM @<UVAID>.DS5111_GIT_STAGE/branches/feature-add-char-count/transform/01_stg_youtube_transcripts.sql;
EXECUTE IMMEDIATE FROM @<UVAID>.DS5111_GIT_STAGE/branches/feature-add-char-count/transform/02_dim_videos.sql;

-- Verify CHAR_COUNT exists in DEV schema
SELECT VIDEO_ID, WORD_COUNT, CHAR_COUNT FROM DIM_VIDEOS;
```

### Step 4.4: Merge Your Feature Branch

Now that you've verified your transformation script works in your isolated `DEV_<UVAID>` schema:

1. Open a **Pull Request** on GitHub.
2. Set the **Base Branch** to `LAB09_gitops_snowflake` and the **Compare Branch** to `feature-add-char-count`.
3. Review your changes and merge the Pull Request.

> ⚠️ **Important Check:** Make sure you are merging into your working lab branch (`LAB09_gitops_snowflake`), NOT `main`!

### Step 4.5: Verify Deployment in Base Schema
Switch back to your primary schema, pull the merged updates from GitHub, and re-run the pipeline orchestrator:

```sql
-- 1. Switch back to primary context
USE SCHEMA <UVAID>;

-- 2. Fetch the newly merged changes from GitHub
ALTER GIT REPOSITORY DS5111_GIT_STAGE FETCH;

-- 3. Execute the full pipeline against primary schema
EXECUTE IMMEDIATE FROM @DS5111_GIT_STAGE/branches/LAB09_gitops_snowflake/transform/orchestrate_pipeline.sql;

-- 4. Confirm CHAR_COUNT is now live in your base schema
SELECT VIDEO_ID, WORD_COUNT, CHAR_COUNT FROM DIM_VIDEOS LIMIT 10;
```

---

## Grading Rubric (10 Points Total)

Submit the Pull Request URL for your `LAB09_gitops_snowflake` branch via Canvas.

| Category | Criteria | Points |
| :--- | :--- | :---: |
| **1. Repository & Git Stage Setup** | • Public/accessible GitHub repository created with correct `transform/` directory structure.<br>• Snowflake `SECRET` and `GIT REPOSITORY` stage successfully configured inside `<UVAID>` schema. | **2 pts** |
| **2. Transformation Scripts & Logic** | • `01_stg_youtube_transcripts.sql`: Correct extraction from `JSON_PAYLOAD` referencing `<UVAID>.RAW_TRANSCRIPTS`.<br>• `02_dim_videos.sql`: Accurate text metric logic (`WORD_COUNT`, array sizes).<br>• `03_fct_entities.sql`: Proper implementation of `LATERAL FLATTEN` to unnest arrays. | **3 pts** |
| **3. GitOps Pipeline Orchestration** | • `orchestrate_pipeline.sql` created and successfully executed using `EXECUTE IMMEDIATE FROM` against the `@DS5111_GIT_STAGE` stage. | **2 pts** |
| **4. Branching & Environment Isolation** | • Demonstrated workflow on `feature-add-char-count` branch targeting `DEV_<UVAID>` schema.<br>• Successful PR/merge into `LAB09_gitops_snowflake` and primary schema execution showing `CHAR_COUNT`. | **2 pts** |
| **5. Deliverables & Verification** | • Submission includes valid GitHub PR URL, execution screenshot/query log, and complete text output from the verification query. | **1 pt** |
| **Total** | | **10 pts** |

---

### Late Policy & Submission Format
* **Submission Deadline:** [Insert Date/Time]
* Submit your GitHub Pull Request URL, execution screenshots, and verification output via Canvas.
* Code that fails to execute due to uncommitted files or incorrect Git stage paths will receive a deduction under Category 3.
