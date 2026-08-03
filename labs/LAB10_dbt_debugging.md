# Debugging tests for dbt in Snowflake


```sql
USE ROLE DS5111_STUDENT_ROLE;
USE DATABASE DS5111_DB;
USE SCHEMA <UVAID>;

-- ONCE YOU HAVE THINGS RUNNING AND YOU WANT TO ITERATE
-- by changing schema.yml and seeing a test break
-- or you are debugging why one broke.  This assumes you're not
-- running the model, i.e. rebuilding tables, you're just updating tests.


-- 1.  Start by editing your schema.yml on your github repo.
-- For this purpose of the lab, commit via the UI directly into
-- your feature branch, THE SAME BRANCH AS IN STEP 3 BELOW!!
-- Make sure your change is committed.

-- 2. Pull the latest branch containing your migrated models
-- you have to do this every time you update github
-- this is what allows snowflake to 'git pull'
ALTER GIT REPOSITORY student_pipeline_repo FETCH;

-- 3. Now that snowflake is caught up to date with github
-- you still need to rebuild the dbt object.  This command takes
-- the changes that are now synched from github and allows your dbt
-- functionality to also get in sync with your repo.
CREATE OR REPLACE DBT PROJECT ds5111_pipeline
  FROM '@student_pipeline_repo/branches/LAB11_dbt_expectations'
  EXTERNAL_ACCESS_INTEGRATIONS = (dbt_external_access);

-- 4. Finally, don't run build, just run the 'test'
EXECUTE DBT PROJECT ds5111_pipeline ARGS = 'test';
```

After you run test, you will either see something like a table that
has a column "SUCCESS" and a value of 'TRUE' in one row.

**OR**

You will see an error message.  That message DOES contain information
about your errors.  However, it also gives you a sql statement to dig deeper.

Look closely at the error, you will see it ends with `run select system$get_dbt_log('01c62208-070c-f7a8-0074-5603015426f2')`.
The actual hash will be different of course.  Copy that from select on forward like this...

```sql
-- don't forget to add the ; at the end
select system$get_dbt_log('01c62208-070c-f7a8-0074-5603015426f2');
```

Then execute that.  A table with the full error log should show up.  Double click on the value field and expand the text.
You should be able to scroll through that and as you read it from top to bottom you'll find information about 
what test(s) failed.

At this point, take that feedback, and go back to the top of this loop and go through again.



# If nothing is working, check your RAW_TRANSCRIPTS table, it should have book and tech term entries.
If it doesn't then everything downstream will break because it depends on this info to build lower tables.

Run this to create a fake RAW_TRANSCRIPTS table so you can proceed.  **NB:** make sure you replace all the <UVAID>, there are 5

```sql
USE DATABASE DS5111_DB;
USE ROLE DS5111_STUDENT_ROLE;
USE SCHEMA <UVAID>;

-- 1. Drop existing table if present
DROP TABLE IF EXISTS <UVAID>.RAW_TRANSCRIPTS;

-- 2. Recreate table with VARIANT and TIMESTAMP_TZ data types
CREATE TABLE <UVAID>.RAW_TRANSCRIPTS (
    JSON_PAYLOAD VARIANT,
    INSERTED_AT TIMESTAMP_TZ
);

-- 3. Populate baseline data using SELECT over VALUES
INSERT INTO <UVAID>.RAW_TRANSCRIPTS (JSON_PAYLOAD, INSERTED_AT)
SELECT 
    PARSE_JSON(raw_json),
    ts_string::TIMESTAMP_TZ
FROM (
    VALUES
    (
        $${
  "book_names": [
    "Clean Code"
  ],
  "cleaned_text": "In this lecture we explore advanced deep learning ablation models using PyTorch.",
  "tech_terms": [
    "PyTorch",
    "Deep Learning"
  ],
  "video_id": "dbUIjFXIpis"
}$$,
        '2026-05-25 14:52:52.480 -0700'
    ),
    (
        $${
  "book_names": [
    "The AWK Programming Language"
  ],
  "cleaned_text": "An introductory guide to setting up automated cron jobs and shell paths.",
  "tech_terms": [
    "Linux",
    "Bash",
    "Cron"
  ],
  "video_id": "dQw4w9WgXcQ"
}$$,
        '2026-05-25 14:52:53.507 -0700'
    )
) AS t(raw_json, ts_string);

-- 4. Verify table output
SELECT 
    JSON_PAYLOAD:video_id::VARCHAR AS video_id,
    JSON_PAYLOAD:cleaned_text::VARCHAR AS text,
    INSERTED_AT
FROM <UVAID>.RAW_TRANSCRIPTS;

SELECT * FROM RAW_TRANSCRIPTS;
```
