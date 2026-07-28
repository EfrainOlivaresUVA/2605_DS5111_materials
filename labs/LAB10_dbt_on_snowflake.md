# Lab 11: Jinja Modeling, Data Quality, and `dbt-expectations`

## Overview

In this lab, you will extend your existing GitOps pipeline by migrating your Lab 9 SQL scripts into standard dbt models, writing dynamic data marts using Jinja templating, and implementing automated data quality tests using the `dbt-expectations` package.

Crucially, **you will execute this entirely within Snowflake**. Instead of configuring an external server, you will author your configuration files locally (or on GitHub), push them to your repository, and use Snowflake's native `EXECUTE DBT PROJECT` engine to compile and run your pipeline directly against your Git branch.

---

**DO NOT EXECUTE THIS COMMAND. This is for your understanding of Snowflake RBAC and External Network Access.**

By default, standard roles cannot create native dbt projects, nor can they reach out to the public internet to download `packages.yml` dependencies like `dbt-expectations`. To make this lab possible, I executed the following commands as `ACCOUNTADMIN` behind the scenes:

```sql
-- 1. Allow students to create dbt project objects in current and future DEV schemas
GRANT CREATE DBT PROJECT ON ALL SCHEMAS IN DATABASE DS5111_DB TO ROLE DS5111_STUDENT_ROLE;
GRANT CREATE DBT PROJECT ON FUTURE SCHEMAS IN DATABASE DS5111_DB TO ROLE DS5111_STUDENT_ROLE;

-- 2. Create a Network Rule to allow outbound traffic to the dbt package hub and GitHub releases
USE ROLE ACCOUNTADMIN;
CREATE OR REPLACE NETWORK RULE dbt_hub_network_rule
  MODE = EGRESS
  TYPE = HOST_PORT
  VALUE_LIST = ('hub.getdbt.com', 'github.com', 'raw.githubusercontent.com', 'codeload.github.com');

-- 3. Create the External Access Integration using that rule
CREATE OR REPLACE EXTERNAL ACCESS INTEGRATION dbt_external_access
  ALLOWED_NETWORK_RULES = (dbt_hub_network_rule)
  ENABLED = true;

-- 4. Grant students permission to use this network integration
GRANT USAGE ON INTEGRATION dbt_external_access TO ROLE DS5111_STUDENT_ROLE;

```

---

## Prerequisites

1. Ensure your **Lab 9 Pull Request** is merged into the `main` branch.
2. In your local terminal (or GitHub), make sure you are on `main`, pull the latest changes, and create a new feature branch for Lab 11:

```bash
git checkout main
git pull origin main
git checkout -b LAB11_dbt_expectations

```

---

## Task 1: Initialize the dbt Project (GitHub Side)

For Snowflake to recognize your repository as a valid dbt project, you need to add configuration files to the root of your repository.

1. Create a file named `dbt_project.yml` in the root of your repository and add the following:

```yaml
name: 'ds5111_transforms'
version: '1.0.0'
config-version: 2
profile: 'default'

model-paths: ["models"]
test-paths: ["tests"]
macro-paths: ["macros"]

```

2. Create a file named `packages.yml` (also in the root) to declare the third-party testing macros:

```yaml
packages:
  - package: calogica/dbt_expectations
    version: [">=0.8.0", "<0.10.0"]

```

3. Create a file named `profiles.yml` (also in the root) to map your specific target schema. Even though Snowflake native execution handles authentication automatically, the dbt parser still strictly requires this file to map the environment:

```yaml
default:
  target: dev
  outputs:
    dev:
      type: snowflake
      # Account and User are ignored natively, but required by dbt core
      account: "native_execution" 
      user: "native_execution"
      role: DS5111_STUDENT_ROLE
      database: DS5111_DB
      schema: <UVAID>  # <-- Replace with your actual computing ID
      warehouse: DS5111_WH # <-- Replace with your assigned warehouse

```

---

## Task 2: Migrating Legacy SQL to dbt (GitHub Side)

In Lab 9, you wrote raw Data Definition Language (DDL) like `CREATE OR REPLACE TABLE...`. A core dbt paradigm is that **models must be pure `SELECT` statements**; dbt handles the environment isolation and table creation behind the scenes.

1. Create a `models/` directory in the root of your repo if you haven't already.
2. Copy your Lab 9 scripts (`stg_youtube_transcripts.sql`, `dim_videos.sql`, `fct_book_mentions.sql`, `fct_tech_terms.sql`) into this `models/` directory. Be sure to remove the numbering prefixes (e.g., `01_`) **because in dbt, the name of the file precisely corresponds to the table/view name in the database**.
3. Open each file and **remove** the `CREATE OR REPLACE TABLE...` header.
4. **Remove trailing semicolons:** dbt complies your `SELECT` statements into larger DDL strings behind the scenes. A semicolon at the end of your file will break the dbt compiler.
5. Add the dbt configuration block at the top of each file: `{{ config(materialized='table') }}`.
* *NOTE: `stg_youtube_transcripts` is actually a view, so adapt that line accordingly.*
* *NOTE: If one of your transform SQL files generates two tables, you must split it. In dbt, one model file = one table.*


6. Replace hardcoded table references in your `FROM` and `JOIN` clauses with the dbt `ref()` function.
* **Crucial Source Mapping Exception:** Only use the `{{ ref() }}` function for models you are actively building in this dbt project. For your absolute lowest-level raw data tables (e.g., `raw_transcripts`), continue to use a standard SQL `FROM` clause (e.g., `FROM raw_transcripts`).



For example, your refactored `fct_book_mentions.sql` should now look similar to this:

```sql
-- CREATE OR REPLACE TABLE DIM_VIDEOS AS was removed and converted to dbt syntax
-- the name of the file itself now represents the name of the table/view
-- the 'table' in the next line carries the table/view directive
{{ config(materialized='table') }}

SELECT 
    video_id,
    book_title,
    mention_timestamp
-- the line FROM STG_YOUTUBE_TRANSCRIPT becomes ...
FROM {{ ref('stg_youtube_transcripts') }}
WHERE book_title IS NOT NULL
-- NOTICE: There is NO semicolon at the end of this query!

```

---

### 2.2 Execute the Migrated Models (Iterative Check)

Before adding new features, let's verify that your core migration was successful. In software engineering, this is known as the iterative development loop: migrate, build, verify, and *then* enhance.

1. **Commit & Push:** Commit your `dbt_project.yml`, `packages.yml`, `profiles.yml`, and your newly refactored `models/` directory, then push them to your branch on GitHub.
2. **Sync Snowflake:** Open a SQL Worksheet in your Snowflake Workspace IDE and fetch your latest code:

```sql
USE ROLE DS5111_STUDENT_ROLE;
USE DATABASE DS5111_DB;
USE SCHEMA <UVAID>;

-- Pull the latest branch containing your migrated models
ALTER GIT REPOSITORY student_pipeline_repo FETCH;

```

3. **Create the Project Object:** Initialize the native dbt project in your schema (this uses the external integration we set up to automatically run `dbt deps` behind the scenes):

```sql
-- Initialize the native dbt project in your primary schema
CREATE OR REPLACE DBT PROJECT ds5111_pipeline
  FROM '@student_pipeline_repo/branches/LAB11_dbt_expectations'
  EXTERNAL_ACCESS_INTEGRATIONS = (dbt_external_access);

```

4. **Execute the Build:** Run the dbt project to dynamically compile your `SELECT` statements into physical tables.

```sql
-- Execute the Build
EXECUTE DBT PROJECT ds5111_pipeline ARGS = 'run';

```

5. **Verify:** Check the query results pane to ensure your foundational tables (`stg_youtube_transcripts`, `dim_videos`, `fct_book_mentions`, `fct_tech_terms`) built successfully without any legacy DDL errors.

---

## Task 3: Authoring a Dynamic Mart with Jinja (GitHub Side)

Now that your foundation is migrated, let's utilize dbt's superpower: Jinja templating. We want to create a wide table that counts the occurrences of specific tech terms per video.

Create a new file at `models/mart_tech_term_pivot.sql` and add the following:

```sql
{{ config(materialized='table') }}

-- 1. Define a Python-style list of core terms we want to track
{% set core_terms = ['python', 'sql', 'dbt', 'snowflake', 'aws', 'docker'] %}

SELECT
    video_id,
    
    -- 2. Loop through the list to dynamically generate our columns
    {% for term in core_terms %}
    
    SUM(CASE WHEN LOWER(term_name) = '{{ term }}' THEN 1 ELSE 0 END) AS count_{{ term }}_mentions
    
    -- 3. Add a comma if it's not the last item in the loop
    {% if not loop.last %},{% endif %}
    
    {% endfor %}

FROM {{ ref('fct_tech_terms') }}
GROUP BY video_id

```

### 🛑 FOR COMPARISON ONLY - DO NOT EXECUTE 🛑

Why use Jinja? Because dbt takes your dynamic `for` loop and compiles it into the following raw SQL before sending it to Snowflake. If you were tracking 50 terms instead of 6, the Jinja template remains exactly the same size, while the raw SQL becomes a massive, error-prone wall of text:
```sql
SELECT
    video_id,
    SUM(CASE WHEN LOWER(term_name) = 'python' THEN 1 ELSE 0 END) AS count_python_mentions,
    SUM(CASE WHEN LOWER(term_name) = 'sql' THEN 1 ELSE 0 END) AS count_sql_mentions,
    SUM(CASE WHEN LOWER(term_name) = 'dbt' THEN 1 ELSE 0 END) AS count_dbt_mentions,
    SUM(CASE WHEN LOWER(term_name) = 'snowflake' THEN 1 ELSE 0 END) AS count_snowflake_mentions,
    SUM(CASE WHEN LOWER(term_name) = 'aws' THEN 1 ELSE 0 END) AS count_aws_mentions,
    SUM(CASE WHEN LOWER(term_name) = 'docker' THEN 1 ELSE 0 END) AS count_docker_mentions
FROM DS5111_DB.<UVAID>.fct_tech_terms
GROUP BY video_id

```

---

## Task 4: Configure Data Quality Expectations (GitHub Side)

Inside your `models/` folder, create a file named `schema.yml` and declare the data contracts for your migrated tables:

```yaml
models:
  - name: stg_youtube_transcripts
    columns:
      - name: video_id
        tests:
          - not_null
          - dbt_expectations.expect_column_values_to_match_regex:
              regex: "^[A-Za-z0-9_-]{11}$"

  - name: dim_videos
    tests:
      - dbt_expectations.expect_table_columns_to_match_set:
          column_list: ["video_id", "channel_id", "upload_date"]
    columns:
      - name: video_id
        tests:
          - unique
          - not_null

  - name: fct_book_mentions
    columns:
      - name: video_id
        tests:
          - relationships:
              to: ref('dim_videos')
              field: video_id
      - name: book_title
        tests:
          - dbt_expectations.expect_column_value_lengths_to_be_between:
              min_value: 2
              max_value: 250

```

**Action:** Commit your new Jinja model and `models/schema.yml`. Push them to your branch on GitHub.

---

## Task 5: Build & Execute Natively (Snowflake Side)

Switch over to your Snowflake Workspace IDE to compile and execute the updated pipeline.

1. Open a new SQL Worksheet.
2. Execute the following commands sequentially to pull your latest branch, build the new model, and run the test suite:

```sql
USE ROLE DS5111_STUDENT_ROLE;
USE DATABASE DS5111_DB;
USE SCHEMA <UVAID>;

-- 1. Sync Snowflake's Git Stage with your new branch
ALTER GIT REPOSITORY student_pipeline_repo FETCH;

-- 2. Build the new Jinja model
EXECUTE DBT PROJECT ds5111_pipeline ARGS = 'run';

-- 3. Execute the dbt test suite against your pipeline
EXECUTE DBT PROJECT ds5111_pipeline ARGS = 'test';

```

3. Confirm that the models successfully build and all tests return a passing status in the query results pane.

---

## Task 6: Failure Auditing with `--store-failures`

1. On your local machine (or GitHub), intentionally edit `models/schema.yml` to force a failure. Change the `book_title` length test on `fct_book_mentions` to an unrealistic threshold (e.g., set `min_value: 150`).
2. Commit and push the broken configuration to your feature branch.
3. In Snowflake, fetch the repository again and run the test suite with the failure-storing flag:

```sql
ALTER GIT REPOSITORY student_pipeline_repo FETCH;
   
EXECUTE DBT PROJECT ds5111_pipeline 
  ARGS = 'test --store-failures --select fct_book_mentions';

```

4. Snowflake will automatically create an audit table containing the bad records. Query it to inspect the failing data:

```sql
USE DATABASE DS5111_DB;
USE SCHEMA <UVAID>;
   
SELECT * 
FROM <UVAID>.dbt_test__audit.<test_name_from_output>;

```

5. **Take a screenshot** of your Snowflake worksheet showing these failing records.
6. Revert your `min_value` back to `2`, push the fix, fetch in Snowflake, and confirm the test suite passes green once more.

---

## Submission & Deliverables

1. Open a Pull Request on GitHub merging your lab branch into `main`.
2. Submit the following to Canvas:
* The **URL of your Pull Request**.
* A **screenshot** of the Snowflake query results pane showing a passing execution of the `test` command.
* A **screenshot** of your Snowflake audit table query from Task 6 showing the isolated failing records.
