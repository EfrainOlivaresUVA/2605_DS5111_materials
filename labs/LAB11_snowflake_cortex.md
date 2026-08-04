# Lab 11: In-Database Generative AI & Unstructured Analytics with Snowflake Cortex

## Executive Summary & Learning Objectives

In modern data engineering, over 80% of enterprise data exists in unstructured or semi-structured forms—such as customer tickets, audio transcripts, PDFs, and log files. Historically, analyzing this data required exporting raw text out of the data warehouse into external Python environments, managing third-party API credentials (e.g., OpenAI, Anthropic), and re-ingesting model outputs.

**Snowflake Cortex AI** fundamentally changes this paradigm by running Large Language Models (LLMs) and vector embedding models directly inside the data warehouse kernel.

By completing this lab, you will learn how to:

1. **Manage Dynamic User Schemas:** Safely execute isolated lab environments inside a shared course database.
2. **Execute In-Database Text Classification (`CORTEX.COMPLETE`):** Constrain LLM outputs into deterministic enumerations for structured reporting.
3. **Perform Semantic Entity Extraction (`CORTEX.EXTRACT_ANSWER`):** Extract fine-grained metadata from unstructured text using question-answering pipelines.
4. **Compute Vector Embeddings & Similarity Scores (`EMBED_TEXT_768` + `VECTOR_COSINE_SIMILARITY`):** Quantify semantic similarity between natural language search concepts and document repositories without generative text.
5. **Build Native Snowsight Visualizations:** Transform Cortex-generated data marts directly into executive charts (Pie, Bar, Ranked Metrics).

---

## Technical Prerequisites & Setup

### Environment Architecture
* **Database:** `DS5111_DB`
* **Schema:** `<YOUR_COMPUTING_ID>` (User-isolated sandbox)
* **Compute Warehouse:** `COMPUTE_WH`

---

### Step 0: Initializing Your Isolated Session Context

Open a clean **SQL Worksheet** in Snowsight and run the following configuration block. This script dynamically resolves your active user ID (`CURRENT_USER()`) and switches your session context to your personal schema, preventing accidental cross-talk with classmates.

```sql
-- Step 0.1: Ensure active database context
USE DATABASE DS5111_DB;

-- Step 0.2: Dynamically resolve personal user schema
SET MY_SCHEMA = (SELECT CURRENT_USER());

-- Step 0.3: Context switch to personal schema
USE SCHEMA IDENTIFIER($MY_SCHEMA);

-- Step 0.4: Verify active environment
SELECT CURRENT_DATABASE(), CURRENT_SCHEMA(), CURRENT_USER();
```

> ⚠️ **CRITICAL GOTCHA — SESSION VARIABLES:**
> If you close your worksheet tab or re-login to Snowflake, your session variable (`$MY_SCHEMA`) resets. Always execute **Step 0** at the beginning of any query session before running downstream commands!

---

## Step 1: Populating the Baseline Dataset

We will build a uniform source table named `cortex_youtube_demo` containing metadata and full-length text transcripts across **8 video lectures** spanning four distinct technical domains:

* **Data Engineering**
* **Cloud AI**
* **Quantitative Finance**
* **Cloud Architecture.**

Run the DDL and DML statements below:

```sql
-- Step 1.1: Create source table structure
CREATE OR REPLACE TABLE cortex_youtube_demo (
    video_id VARCHAR(20),
    title VARCHAR(255),
    channel_title VARCHAR(100),
    upload_date DATE,
    view_count INT,
    description VARCHAR(1000),
    transcript_text VARCHAR(4000)
);

-- Step 1.2: Populate rich baseline data
INSERT INTO cortex_youtube_demo VALUES
(
  'vid_101', 
  'Building Modern Data Pipelines with dbt and Snowflake', 
  'Data Engineering Academy', 
  '2026-03-15', 
  14500,
  'Learn how to transform raw database tables into clean data marts using Jinja models, testing, and Snowflake warehouses.',
  'Welcome back everyone! Today we are diving into dbt and Snowflake. We start by configuring project profiles in YAML. Then we write staging models using SQL views. Notice how Jinja templating allows us to dynamically pivot technology terms across millions of rows without writing redundant SQL. Finally, we run automated data quality tests using dbt-expectations to catch duplicate primary keys before pushing to production.'
),
(
  'vid_102', 
  'Intro to Vector Search and Snowflake Cortex', 
  'Cloud AI Weekly', 
  '2026-04-02', 
  8900,
  'A deep dive into in-database LLMs, vector embeddings, and Retrieval-Augmented Generation (RAG) using Snowflake Cortex.',
  'In this tutorial, we explore Snowflake Cortex AI functions. You do not need third-party API keys or external Python services anymore. We generate 768-dimensional vector embeddings directly inside standard SELECT queries using EMBED_TEXT_768. Then we use complete LLM prompts with llama3-8b to automatically generate video summaries and perform sentiment extraction directly inside our data warehouse boundary.'
),
(
  'vid_103', 
  'Automated Algorithmic Trading with Python and AWS Lambda', 
  'Quant & Cloud Dev', 
  '2026-05-20', 
  22100,
  'How to build serverless trading pipelines for QQQ ETF momentum strategies using Alpaca API and cloud functions.',
  'Hey traders! Today we are looking at cloud automation for quantitative finance. We write a custom Python package deployed to AWS Lambda functions that periodically fetches QQQ ETF price quotes using the Alpaca API. The script evaluates exponential moving average cross signals and places automated market orders while maintaining strict trailing stop-loss orders in real time.'
),
(
  'vid_104', 
  'Advanced Data Lake Architecture on AWS S3 & Iceberg', 
  'Architecting the Cloud', 
  '2026-06-11', 
  5400,
  'Understanding Apache Iceberg table formats, parquet storage layers, and query optimization for high-volume analytics.',
  'Modern data lakes are transitioning from simple S3 storage layouts to open table formats like Apache Iceberg. In this session, we dissect snapshot isolation, schema evolution, and time-travel capabilities. We also show how Snowflake seamlessly reads external Iceberg catalogs, dramatically reducing storage costs while preserving enterprise performance.'
),
(
  'vid_105', 
  'Streamlining Data Orchestration with Airflow and Snowflake', 
  'Data Engineering Academy', 
  '2026-06-25', 
  11200,
  'Automating dbt runs, staging SQL queries, and Python ETL workers using Apache Airflow DAGs.',
  'Welcome back! Today we are linking our dbt models directly into Apache Airflow pipelines. We configure Airflow operators to trigger dbt run tasks inside Docker containers. We also show how to catch pipeline failures in Python and push alerts directly to Slack when upstream data quality tests fail inside Snowflake.'
),
(
  'vid_106', 
  'Deploying LLMs in Production with Meta Llama 3 & Python', 
  'Cloud AI Weekly', 
  '2026-07-01', 
  18300,
  'Fine-tuning foundation models, managing token context windows, and building low-latency inference pipelines.',
  'In this episode, we build a production microservice for LLM inference using Llama3 models. We evaluate prompt engineering strategies, custom system messages, and quantization techniques to optimize throughput. We also benchmark inference latencies using GPUs vs cloud data warehouses.'
),
(
  'vid_107', 
  'Real-Time Options Pricing & Risk Management with Python', 
  'Quant & Cloud Dev', 
  '2026-07-10', 
  19800,
  'Calculating option Greeks, implied volatility surfaces, and Monte Carlo risk simulations using Pandas and NumPy.',
  'Today we dive deep into financial risk modeling. We write vectorized Python code using NumPy and Pandas to simulate thousands of price paths for equity options. We compute Black-Scholes pricing models and calculate Delta and Gamma hedging ratios dynamically as market data streams in.'
),
(
  'vid_108', 
  'Zero-Copy Cloning and Data Sharing in Snowflake', 
  'Architecting the Cloud', 
  '2026-07-18', 
  7200,
  'Enterprise data warehousing tricks: instant database cloning, role-based access control, and Secure Data Sharing.',
  'Snowflake revolutionized data sharing by eliminating traditional ETL export files. In this lesson, we demonstrate Zero-Copy Cloning to instant-spin dev schemas without duplicating physical storage costs. Then we configure Secure Data Shares to grant external partner accounts live, read-only access to our analytical marts.'
);

-- Step 1.3: Verify insertion
SELECT COUNT(*) FROM cortex_youtube_demo;

SELECT * FROM CORTEX_YOUTUBE_DEMO LIMIT 100;
```

| video_id | title | channel_title | upload_date | view_count | description | transcript_text |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **vid_101** | Building Modern Data Pipelines with dbt and Snowflake | Data Engineering Academy | 2026-03-15 | 14,500 | Learn how to transform raw database tables into clean data marts using Jinja models, testing, and Snowflake warehouses. | Welcome back everyone! Today we are diving into dbt and Snowflake. We start by configuring project profiles in YAML. Then we write staging models using SQL views. Notice how Jinja templating allows us to dynamically pivot technology terms across millions of rows without writing redundant SQL. Finally, we run automated data quality tests using dbt-expectations to catch duplicate primary keys before pushing to production. |
| **vid_102** | Intro to Vector Search and Snowflake Cortex | Cloud AI Weekly | 2026-04-02 | 8,900 | A deep dive into in-database LLMs, vector embeddings, and Retrieval-Augmented Generation (RAG) using Snowflake Cortex. | In this tutorial, we explore Snowflake Cortex AI functions. You do not need third-party API keys or external Python services anymore. We generate 768-dimensional vector embeddings directly inside standard SELECT queries using EMBED_TEXT_768. Then we use complete LLM prompts with llama3-8b to automatically generate video summaries and perform sentiment extraction directly inside our data warehouse boundary. |
| **vid_103** | Automated Algorithmic Trading with Python and AWS Lambda | Quant & Cloud Dev | 2026-05-20 | 22,100 | How to build serverless trading pipelines for QQQ ETF momentum strategies using Alpaca API and cloud functions. | Hey traders! Today we are looking at cloud automation for quantitative finance. We write a custom Python package deployed to AWS Lambda functions that periodically fetches QQQ ETF price quotes using the Alpaca API. The script evaluates exponential moving average cross signals and places automated market orders while maintaining strict trailing stop-loss orders in real time. |
| **vid_104** | Advanced Data Lake Architecture on AWS S3 & Iceberg | Architecting the Cloud | 2026-06-11 | 5,400 | Understanding Apache Iceberg table formats, parquet storage layers, and query optimization for high-volume analytics. | Modern data lakes are transitioning from simple S3 storage layouts to open table formats like Apache Iceberg. In this session, we dissect snapshot isolation, schema evolution, and time-travel capabilities. We also show how Snowflake seamlessly reads external Iceberg catalogs, dramatically reducing storage costs while preserving enterprise performance. |
| **vid_105** | Streamlining Data Orchestration with Airflow and Snowflake | Data Engineering Academy | 2026-06-25 | 11,200 | Automating dbt runs, staging SQL queries, and Python ETL workers using Apache Airflow DAGs. | Welcome back! Today we are linking our dbt models directly into Apache Airflow pipelines. We configure Airflow operators to trigger dbt run tasks inside Docker containers. We also show how to catch pipeline failures in Python and push alerts directly to Slack when upstream data quality tests fail inside Snowflake. |
| **vid_106** | Deploying LLMs in Production with Meta Llama 3 & Python | Cloud AI Weekly | 2026-07-01 | 18,300 | Fine-tuning foundation models, managing token context windows, and building low-latency inference pipelines. | In this episode, we build a production microservice for LLM inference using Llama3 models. We evaluate prompt engineering strategies, custom system messages, and quantization techniques to optimize throughput. We also benchmark inference latencies using GPUs vs cloud data warehouses. |
| **vid_107** | Real-Time Options Pricing & Risk Management with Python | Quant & Cloud Dev | 2026-07-10 | 19,800 | Calculating option Greeks, implied volatility surfaces, and Monte Carlo risk simulations using Pandas and NumPy. | Today we dive deep into financial risk modeling. We write vectorized Python code using NumPy and Pandas to simulate thousands of price paths for equity options. We compute Black-Scholes pricing models and calculate Delta and Gamma hedging ratios dynamically as market data streams in. |
| **vid_108** | Zero-Copy Cloning and Data Sharing in Snowflake | Architecting the Cloud | 2026-07-18 | 7,200 | Enterprise data warehousing tricks: instant database cloning, role-based access control, and Secure Data Sharing. | Snowflake revolutionized data sharing by eliminating traditional ETL export files. In this lesson, we demonstrate Zero-Copy Cloning to instant-spin dev schemas without duplicating physical storage costs. Then we configure Secure Data Shares to grant external partner accounts live, read-only access to our analytical marts. |

---

## Step 2: Case 1 — LLM Topic Categorization (`CORTEX.COMPLETE`)

### Pedagogical Goal
Standard SQL `CASE WHEN` or string keyword matching is fragile. If a transcript doesn't explicitly contain the word "Data Engineering", traditional logic fails. 

By utilizing `SNOWFLAKE.CORTEX.COMPLETE()`, we instruct an open-source foundation model (`llama3-8b`) to read the full context of each transcript and classify it into **one of four strict enum categories**.

### SQL Implementation

```sql
-- Step 2.1: Materialize AI-Classified Topic Mart
CREATE OR REPLACE TABLE mart_cortex_topics AS
SELECT 
    video_id,
    title,
    channel_title,
    TRIM(
        SNOWFLAKE.CORTEX.COMPLETE(
            'llama3-8b', 
            'Classify this video transcript into EXACTLY ONE of these categories: [Data Engineering, Cloud AI, Quantitative Finance, Cloud Architecture]. Return ONLY the exact category name. Transcript: ' || transcript_text
        )
    ) AS primary_topic
FROM cortex_youtube_demo;

-- Step 2.2: Aggregation Query with Traceable Video IDs
SELECT 
    primary_topic, 
    COUNT(*) AS video_count,
    LISTAGG(video_id, ', ') WITHIN GROUP (ORDER BY video_id) AS matching_video_ids
FROM mart_cortex_topics 
GROUP BY 1 
ORDER BY video_count DESC;
```

### Deep Dive: How the SQL Works
* `SNOWFLAKE.CORTEX.COMPLETE('llama3-8b', prompt)` invokes the Llama 3 8-billion parameter model directly on Snowflake's GPU compute infrastructure.
* **Prompt Engineering Strategy:** We explicitly constrain the model's output space using `EXACTLY ONE` and `Return ONLY the exact category name`. This prevents conversational fluff that breaks relational `GROUP BY` logic.
* `LISTAGG(video_id, ', ')` concatenates all `video_id` records that fell into a given topic bucket into a readable list, enabling auditability against our seed data.

> 💡 **PRO TIP — PROMPT STRENGTH:**
> If an LLM occasionally appends punctuation (e.g., `'Data Engineering.'` vs `'Data Engineering'`), you can sanitize results using `REPLACE(TRIM(...), '.', '')`.

### Snowsight Visualization Guide
1. Run **Step 2.2** in your worksheet.
2. In the results panel below the query, switch from the **Table** tab to the **Chart** tab.
3. Configure the left sidebar properties:

* **Chart type:** `Pie chart` (or Donut chart)
* **Categories:** `PRIMARY_TOPIC`
* **Values:** `VIDEO_COUNT`
* **Aggregate:** `Sum`

4. **Expected Outcome:** You should see an evenly distributed pie chart rendering the four topic domains (e.g., 2 videos per topic).

## Step 2B: The Non-Deterministic Nature of LLMs & Enterprise Guardrails

### The Experiment: Force-Testing Output Drift

In traditional relational database engineering, functions are **deterministic**: `1 + 1` always equals `2`, and `UPPER('snowflake')` always returns `'SNOWFLAKE'`. 

Large Language Models, however, are **probabilistic statistical engines**. They construct responses by calculating token probability distributions. To observe this in practice, let's drop our topic mart, re-execute the exact same query, and inspect the raw output strings directly.

```sql
-- Step 2B.1: Re-run classification, retaining the raw un-trimmed LLM output for analysis
CREATE OR REPLACE TABLE mart_cortex_topics_experiment AS
SELECT 
    video_id,
    title,
    channel_title,
    SNOWFLAKE.CORTEX.COMPLETE(
        'llama3-8b', 
        'Classify this video transcript into EXACTLY ONE of these categories: [Data Engineering, Cloud AI, Quantitative Finance, Cloud Architecture]. Return ONLY the exact category name and nothing else. Transcript: ' || transcript_text
    ) AS raw_llm_response
FROM cortex_youtube_demo;

-- Step 2B.2: Inspect raw response distribution
SELECT 
    raw_llm_response, 
    COUNT(*) AS video_count,
    LISTAGG(video_id, ', ') WITHIN GROUP (ORDER BY video_id) AS matching_video_ids
FROM mart_cortex_topics_experiment
GROUP BY 1 
ORDER BY 2 DESC;
```

---

### The Cautionary Tale: Why the Distribution Shifted

When running this experiment, you may notice your distribution change from a clean `2-2-2-2` spread to `3-2-2-1`, `5-2-1-0`, or see new unexpected rows in your aggregation table.

Over-relying directly on raw generative AI outputs inside enterprise analytics pipelines introduces serious vulnerabilities. Here are the **three primary technical reasons** why this output drift occurs:

#### 1. Semantic Overlap in Unstructured Text
Unstructured text rarely fits neatly into rigid conceptual silos. Consider `vid_105` (*Streamlining Data Orchestration with Airflow and Snowflake*) or `vid_102` (*Intro to Vector Search and Snowflake Cortex*). Does `vid_102` belong under **Cloud AI** or **Data Engineering**? Because the text contains strong signals for both domains, minor probability shifts during token generation can tip the model toward different categories across separate runs.

#### 2. Formatting Drift & Tokenizer Variations
Small foundation models (such as `llama3-8b`) occasionally introduce minor formatting quirks—such as appending a trailing period (`"Cloud AI."`), adding a leading newline, or echoing introductory text (`"Category: Cloud AI"`). 
Because standard SQL `GROUP BY` logic relies on exact byte-for-byte string matching, SQL treats `"Cloud AI."` and `"Cloud AI"` as **two completely distinct dimensions**, splitting your aggregated metrics and breaking dashboard reporting.

#### 3. Probabilistic GPU Sampling
Under the hood, Snowflake Cortex distributes inference tasks across parallel GPU clusters. Unless sampling temperature is strictly forced to `0.0` at the API layer, foundation models use top-$p$ / top-$k$ probabilistic sampling to select the next token. This inherent randomness means identical inputs can yield subtle variations in output text.

---

### Mitigation & Testing Strategies for Production Pipelines

As data engineers, our primary goal when integrating AI into data warehouses is to build **deterministic guardrails around non-deterministic models**. 

We employ three core strategies to mitigate drift:

1. **Prompt Hardening & Output Formatting:** Explicitly instruct the model to omit punctuation, conversational preamble, or markdown formatting (or require structured output like JSON).
2. **SQL Normalization Layer (Dimension Snapping):** Implement a staging layer using CTEs combined with `CASE WHEN ... ILIKE` regular expressions to map fuzzy string responses back into clean, controlled business dimensions.
3. **Automated Assertion Testing:** Write SQL unit tests or `dbt-expectations` rules that check for `NULL` values or unmapped categories (`'Unclassified / Other'`) before pushing LLM-processed data into downstream reporting tables.

---

### Production Implementation: The Guardrailed Pipeline

Below is the production-grade design pattern that guarantees clean, resilient aggregations regardless of LLM text drift:

```sql
-- Step 2B.4: Materialize AI Topics with Production Guardrails
CREATE OR REPLACE TABLE mart_cortex_topics AS
WITH raw_ai_output AS (
    SELECT 
        video_id,
        title,
        channel_title,
        SNOWFLAKE.CORTEX.COMPLETE(
            'llama3-8b', 
            'Classify this video transcript into EXACTLY ONE of these categories: [Data Engineering, Cloud AI, Quantitative Finance, Cloud Architecture]. Return ONLY the exact category name. Transcript: ' || transcript_text
        ) AS raw_llm_response
    FROM cortex_youtube_demo
)
SELECT 
    video_id,
    title,
    channel_title,
    raw_llm_response,
    -- Guardrail: Map fuzzy/probabilistic LLM text into clean relational dimensions
    CASE 
        WHEN raw_llm_response ILIKE '%Data Engineering%' THEN 'Data Engineering'
        WHEN raw_llm_response ILIKE '%Cloud AI%' THEN 'Cloud AI'
        WHEN raw_llm_response ILIKE '%Quantitative Finance%' THEN 'Quantitative Finance'
        WHEN raw_llm_response ILIKE '%Cloud Architecture%' THEN 'Cloud Architecture'
        ELSE 'Unclassified / Other'
    END AS primary_topic
FROM raw_ai_output;

-- Step 2B.5: Data Quality Test — Verify zero unclassified rows
SELECT 
    COUNT(*) AS unclassified_count 
FROM mart_cortex_topics 
WHERE primary_topic = 'Unclassified / Other';
-- Expected Output: 0 rows

-- Step 2B.6: Final Deterministic Aggregation for Visualization
SELECT 
    primary_topic, 
    COUNT(*) AS video_count,
    LISTAGG(video_id, ', ') WITHIN GROUP (ORDER BY video_id) AS matching_video_ids
FROM mart_cortex_topics 
GROUP BY 1 
ORDER BY video_count DESC;
```

## Deep Dive: Ground Truth, Model Scale, and AI Evals

### 1. Ground Truth vs. Empirical Model Behavior

In standard software development, code execution is binary: an algorithm either passes tests or throws a bug. In Data Engineering with Generative AI, we must distinguish between **Ground Truth** and **Empirical Model Behavior**:

* **Ground Truth (The Target Baseline):** The dataset created for this lab was intentionally designed with an even split—**2 videos per technical domain** across 4 domains (Data Engineering, Cloud AI, Quantitative Finance, and Cloud Architecture). A perfect system yields a `2-2-2-2` distribution.
* **Empirical Model Behavior (The Raw Output):** Running a zero-shot prompt with `llama3-8b` often produces a skewed distribution such as `5-2-1-0` or `4-3-1-0`. 

---

### 2. Why Small Foundation Models Skew (The Root Causes)

When raw model outputs deviate from Ground Truth, it is usually driven by two core factors:

1. **Vocabulary Overlap (Semantic Collisions):** Keywords like *Snowflake*, *SQL*, and *Data Warehousing* appear across transcripts spanning multiple domains (e.g., Iceberg storage, vector search, and dbt pipelines). Without explicit boundary definitions, the model over-indexes on shared terminology and over-categorizes records under broad buckets like *Data Engineering*.
2. **Model Scale & Attention Fine-Grainedness:** Smaller models like `llama3-8b` prioritize low latency and low computational overhead. However, smaller parameter counts mean coarser attention mechanisms, making subtle conceptual boundary distinctions harder to draw zero-shot.

---

### 3. Improving Model Accuracy: Scaling & Prompt Engineering

To shift model predictions closer to Ground Truth (`2-2-2-2`), data engineers use two main optimization levers in Snowflake Cortex:

#### Method A: Model Scaling (Upgrading Parameter Size)
Upgrading from a smaller foundation model to a higher-capacity model improves reasoning and contextual separation out of the box:

```sql
-- Upgrading from llama3-8b to llama3-70b
SNOWFLAKE.CORTEX.COMPLETE(
    'llama3-70b', 
    'Classify this video transcript into EXACTLY ONE of these categories: [Data Engineering, Cloud AI, Quantitative Finance, Cloud Architecture]. Return ONLY the exact category name. Transcript: ' || transcript_text
)
```

#### Method B: Prompt Engineering (Adding Category Boundaries)
Instead of relying on implicit model knowledge, explicitly define the scope of each category inside your prompt string:

```sql
-- Providing explicit semantic anchors to llama3-8b
SNOWFLAKE.CORTEX.COMPLETE(
    'llama3-8b', 
    'Classify this video transcript into EXACTLY ONE category using these rules:
     - Quantitative Finance: Options, algorithmic trading, stocks, ETFs, financial risk modeling.
     - Cloud AI: Vector embeddings, LLM inference, RAG, prompt engineering, AI microservices.
     - Cloud Architecture: Data sharing, zero-copy cloning, S3 storage, Apache Iceberg, open table formats.
     - Data Engineering: dbt transformations, Airflow orchestration, data pipelines, SQL data quality testing.
     
     Return ONLY the category name. Transcript: ' || transcript_text
)
```

---

### 4. Key Takeaway for Enterprise Data Pipelines

> **Data Engineering Principle:** When an LLM output fails to match expected benchmarks, it does not mean your SQL query failed. It means your pipeline requires an **AI Evaluation ("Eval") cycle**. As a data engineer, your role extends beyond executing queries—you must systematically evaluate model performance, engineer prompt boundaries, adjust model parameter scale, and enforce SQL guardrails to guarantee production reliability.

### 5. Repeat the table build with bigger model
* **Use llama3-70b** and you should see a more consistent `4 2 2 0` spread

## Step 2C: Semantic Overlap & Prompt Engineering Rules

### The 4–2–2 Result: Model Capability vs. Business Taxonomy

When re-running classification with higher-capacity models like `llama3-70b`, you may observe an empirical distribution of **4–2–2–0**:

* **Data Engineering (4):** `vid_101, vid_104, vid_105, vid_108`
* **Cloud AI (2):** `vid_102, vid_106`
* **Quantitative Finance (2):** `vid_103, vid_107`
* **Cloud Architecture (0):** *(Zero videos assigned)*

---

### Why Does `llama3-70b` Classify "Cloud Architecture" as "Data Engineering"?

This output is not a model failure—it reflects high-level contextual reasoning. 

Consider the two videos intended for **Cloud Architecture**:
* `vid_104`: *Apache Iceberg Table Formats & S3 Data Lakes*
* `vid_108`: *Zero-Copy Cloning & Data Sharing in Snowflake*

Across real-world technical literature, managing data lake table formats and warehouse cloning are core **Data Engineering** responsibilities. Because `llama3-70b` understands industry context deeply, it reasons that these transcripts belong under *Data Engineering* rather than generic *Cloud Architecture*.

---

### The Solution: Prompt Engineering with Explicit Category Rules

A foundation model cannot guess an organization's internal business taxonomy. To enforce strict category boundaries without changing your taxonomy names, you must provide **explicit rules** inside the prompt string.

#### Guided Prompt Implementation

```sql
-- Step 2C.1: Materialize Topic Mart with Explicit Category Rules
CREATE OR REPLACE TABLE mart_cortex_topics AS
SELECT 
    video_id,
    title,
    channel_title,
    TRIM(
        SNOWFLAKE.CORTEX.COMPLETE(
            'llama3-70b', 
            'Classify this video transcript into EXACTLY ONE category using these explicit rules:
             - Quantitative Finance: Options pricing, algorithmic trading, stocks, ETFs, financial risk modeling.
             - Cloud AI: Vector embeddings, LLM inference, RAG, prompt engineering, AI microservices.
             - Cloud Architecture: Open table formats, S3 data lake storage layouts, zero-copy cloning, data sharing.
             - Data Engineering: dbt transformations, Airflow DAG orchestration, data pipelines, SQL data quality testing.
             
             Return ONLY the exact category name. Transcript: ' || transcript_text
        )
    ) AS primary_topic
FROM cortex_youtube_demo;

-- Step 2C.2: Verify Ground Truth Distribution (2-2-2-2)
SELECT 
    primary_topic, 
    COUNT(*) AS video_count,
    LISTAGG(video_id, ', ') WITHIN GROUP (ORDER BY video_id) AS matching_video_ids
FROM mart_cortex_topics 
GROUP BY 1 
ORDER BY video_count DESC;
```

---

### Expected Output & Key Takeaway

By providing explicit domain boundaries, the query output successfully converges on our target **Ground Truth distribution**:

| PRIMARY_TOPIC | VIDEO_COUNT | MATCHING_VIDEO_IDS |
| :--- | :--- | :--- |
| **Cloud AI** | 2 | `vid_102, vid_106` |
| **Cloud Architecture** | 2 | `vid_104, vid_108` |
| **Data Engineering** | 2 | `vid_101, vid_105` |
| **Quantitative Finance** | 2 | `vid_103, vid_107` |

> 📌 **Core Engineering Principle:**
> Larger models (`70B`) provide superior reasoning, but higher reasoning alone does not eliminate ambiguity. When domain terms overlap in real-world contexts, use **Prompt Engineering (explicit rule boundaries)** to align probabilistic LLMs with your target schema definitions.

---

## Step 3: Case 2 — Multi-Entity Extraction (`CORTEX.EXTRACT_ANSWER`)

### Pedagogical Goal
Suppose business leadership wants to know which software frameworks, tools, or libraries are most frequently mentioned across all video transcripts. Writing regular expressions (`REGEXP_SUBSTR`) for hundreds of potential software packages is impractical.

Using `SNOWFLAKE.CORTEX.EXTRACT_ANSWER()`, we perform question-answering over unstructured text to extract target technologies into structured data fields.

### SQL Implementation

```sql
-- Step 3.1: Materialize Extracted Entity Mart
CREATE OR REPLACE TABLE mart_cortex_tech AS
SELECT 
    video_id,
    title,
    channel_title,
    TRIM(
        SNOWFLAKE.CORTEX.EXTRACT_ANSWER(
            transcript_text, 
            'What main software library, database, framework, or cloud service is discussed?'
        )[0]:answer::STRING
    ) AS primary_tool
FROM cortex_youtube_demo;

-- Step 3.2: Aggregation Query for Frequency Distribution
SELECT 
    primary_tool, 
    COUNT(*) AS mentions
FROM mart_cortex_tech
WHERE primary_tool IS NOT NULL
GROUP BY 1
ORDER BY mentions DESC;
```

### Deep Dive: How the SQL Works
* `SNOWFLAKE.CORTEX.EXTRACT_ANSWER(source_text, question)` returns a semi-structured **VARIANT** array containing key-value pairs (`answer`, `score`).
* `[0]:answer::STRING` uses Snowflake semi-structured dot-notation to select the top prediction (`[0]`), extract the value associated with the `"answer"` key, and cast it to a native SQL `VARCHAR`.

### Snowsight Visualization Guide
1. Run **Step 3.2** in your worksheet.
2. Click the **Chart** tab above the query output grid.
3. Configure the left sidebar properties:

* **Chart type:** `Bar chart`
* **X-Axis:** `MENTIONS`
* **Y-Axis / Category:** `PRIMARY_TOOL`

4. **Expected Outcome:** A horizontal bar chart ranking mentioned tools (e.g., `dbt`, `Snowflake`, `Python`, `AWS Lambda`, `Apache Iceberg`).

---

## Step 4: Case 3 — Vector Embeddings & Semantic Search

### Pedagogical Goal
Generative text generation is only one side of modern AI. Semantic Search relies on **Vector Embeddings**—mathematical arrays of floating-point numbers that capture the conceptual meaning of text in high-dimensional space.

In this step, we do **not** generate new text. Instead, we:
1. Convert a natural language query (`$SEARCH_CONCEPT`) into a 768-dimensional vector embedding using `EMBED_TEXT_768`.
2. Convert each video transcript into a 768-dimensional vector embedding.
3. Compute the **Cosine Similarity** (angle between vectors) to yield a mathematical match percentage ($0.0\%$ to $100.0\%$).

### SQL Implementation

```sql
-- Step 4.1: Define target search concept variable
SET SEARCH_CONCEPT = 'automated Python trading algorithms and cloud execution';

-- Step 4.2: Materialize Vector Similarity Mart
CREATE OR REPLACE TABLE mart_cortex_relevance AS
SELECT 
    title,
    channel_title,
    ROUND(
        VECTOR_COSINE_SIMILARITY(
            SNOWFLAKE.CORTEX.EMBED_TEXT_768('snowflake-arctic-embed-m', $SEARCH_CONCEPT),
            SNOWFLAKE.CORTEX.EMBED_TEXT_768('snowflake-arctic-embed-m', transcript_text)
        ) * 100, 1
    ) AS relevance_match_pct
FROM cortex_youtube_demo;

-- Step 4.3: Query Ranked Output for Charting
SELECT 
    title, 
    relevance_match_pct 
FROM mart_cortex_relevance 
ORDER BY relevance_match_pct ASC; -- ASC so highest match renders at top of horizontal chart
```

### Deep Dive & Critical Function Namespace Gotcha

> 🚨 **CRITICAL GOTCHA — FUNCTION NAMESPACES:**
> * `SNOWFLAKE.CORTEX.EMBED_TEXT_768()` resides within the `SNOWFLAKE.CORTEX` schema namespace.
> * `VECTOR_COSINE_SIMILARITY()` is a built-in Snowflake scalar function and lives in the **root System namespace**.
> 
> ❌ **Incorrect:** `SNOWFLAKE.CORTEX.VECTOR_COSINE_SIMILARITY(...)` $\rightarrow$ Throws `Unknown user-defined function` error.  
> ✅ **Correct:** `VECTOR_COSINE_SIMILARITY(...)`

### Snowsight Visualization Guide
1. Run **Step 4.3** in your worksheet.
2. Click the **Chart** tab above the result grid.
3. Configure the left sidebar properties:
   * **Chart type:** `Bar chart`
   * **Orientation:** `Horizontal`
   * **X-Axis:** `RELEVANCE_MATCH_PCT`
   * **Y-Axis:** `TITLE`
4. **Expected Outcome:** A ranked bar chart where *"Automated Algorithmic Trading with Python and AWS Lambda"* distinctly tops the list with an $\sim 85\%+$ match score, while unrelated topics (e.g., Data Lakes) drop toward the bottom ($\sim 20\% - 30\%$).

---

## Lab Verification Checklist & Deliverables

To complete this assignment, submit a single PDF document or Markdown report containing:

- [ ] **Verification Query Output:** Run `SELECT * FROM DS5111_DB.<YOUR_COMPUTING_ID>.mart_cortex_topics;` and screenshot the resultant table.
- [ ] **Chart Deliverable 1:** Screenshot of your **Topic Categorization Pie Chart** (Step 2).
- [ ] **Chart Deliverable 2:** Screenshot of your **Tech Stack Mentions Horizontal Bar Chart** (Step 3).
- [ ] **Chart Deliverable 3:** Screenshot of your **Semantic Search Vector Relevance Ranked Bar Chart** (Step 4).
- [ ] **Short Answer Question:** In 2–3 sentences, explain why `VECTOR_COSINE_SIMILARITY` is computationally faster and less prone to hallucinations than invoking `CORTEX.COMPLETE` for search and ranking tasks.

---

## Summary Reference Table of Cortex Functions Used

| Function Name | Return Type | Primary Enterprise Use Case |
| :--- | :--- | :--- |
| `SNOWFLAKE.CORTEX.COMPLETE()` | `VARCHAR` | Conversational text generation, strict enum classification, translation, summarization. |
| `SNOWFLAKE.CORTEX.EXTRACT_ANSWER()` | `VARIANT` (JSON) | Structured entity extraction, question answering over documents. |
| `SNOWFLAKE.CORTEX.EMBED_TEXT_768()` | `VECTOR(FLOAT, 768)` | Generating dense mathematical vector representations for semantic search and RAG architectures. |
| `VECTOR_COSINE_SIMILARITY()` | `FLOAT` | Measuring mathematical angular distance between two vector embeddings ($0.0$ to $1.0$). |
