# Runtime ETL Pipeline (Snowflake + Groq Agent) — Steps

This repo uses GitHub Actions to generate a runtime ETL `SELECT` statement using **Groq LLM** and then run the ETL through **Snowflake**.

Workflow file: `.github/workflows/blank.yml`

---

## 1) What was implemented

The workflow now:

- Triggers on **every push** and also supports **manual run** (`workflow_dispatch`)
- Sets up Python (3.11)
- Installs dependencies:
  - `snowflake-connector-python`
  - `langchain-groq`
- Builds a Groq LLM client using the secret `GROQ_API_KEY`
- Prompts the model to return **ONLY** a single Snowflake `SELECT` statement
- Cleans/sanitizes model output to avoid invalid SQL getting sent to Snowflake
- Prints:
  - the raw model output
  - the final cleaned `SELECT`
  - a **preview of the query output** (first 10 rows) into the Actions logs
- Connects to Snowflake and:
  - ensures the target table exists (creates empty schema using `SELECT ... WHERE 1=0`)
  - creates or replaces a Snowflake `TASK` dynamically
  - executes the task immediately

---

## 2) Configure GitHub Actions Secrets (required)

In GitHub:
`Repo → Settings → Secrets and variables → Actions → Secrets`

Create these **Secrets**:

- `GROQ_API_KEY`
- `SNOWFLAKE_ACCOUNT`
- `SNOWFLAKE_USER`
- `SNOWFLAKE_PASSWORD`
- `SNOWFLAKE_ROLE`
- `SNOWFLAKE_DATABASE`
- `SNOWFLAKE_SCHEMA`

Security note: never hardcode keys in code/YAML. If a key was ever pasted into chat/logs, rotate/regenerate it.

---

## 3) Configure GitHub Actions Variables (required for push runs)

Because `push` runs do not have `workflow_dispatch` inputs, set these **Variables** in:
`Repo → Settings → Secrets and variables → Actions → Variables`

Required:

- `ETL_SOURCE_TABLE`
  - Example: `RAW_DB.RAW_SCHEMA.ST_ORDERS`
- `ETL_TARGET_TABLE`
  - Example: `DWH_DB.DWH_SCHEMA.TT_ORDERS`

Optional:

- `SNOWFLAKE_WAREHOUSE`
  - Default used by workflow: `COMPUTE_WH`

---

## 4) How to run

### A) Run automatically (on every push)

- Commit and push any change to GitHub.
- The workflow triggers automatically.
- It uses:
  - `vars.ETL_SOURCE_TABLE`
  - `vars.ETL_TARGET_TABLE`
  - `vars.SNOWFLAKE_WAREHOUSE` (or default `COMPUTE_WH`)

### B) Run manually (workflow_dispatch)

In GitHub:
`Actions → Runtime ETL Pipeline (Snowflake + Groq Agent) → Run workflow`

Fill in inputs:

- `pipeline_name`
- `source_table`
- `target_table`
- `warehouse`

Manual inputs override Variables.

---

## 5) Snowflake SQL helpers

### Create schema

```sql
CREATE SCHEMA IF NOT EXISTS <DB_NAME>.SAMPLE;
```

### Check schema/table columns (needed for correct INSERT samples)

```sql
DESC TABLE st_Orders;
DESC TABLE tt_orders;
```

### View raw data in staging table

```sql
SELECT *
FROM st_Orders
LIMIT 50;
```

---

## 6) Troubleshooting

### Pipeline did not trigger after push

Cause: workflow only triggers on what’s under `on:` in the YAML.

Fix: ensure `.github/workflows/blank.yml` includes `push:` (it does in the current version).

### Snowflake error: `unexpected 'To'` / SQL compilation errors

Example error:

`SQL compilation error: syntax error ... unexpected 'To'.`

Cause: model output included prose/markdown (e.g. “To do this…”) instead of a clean `SELECT`.

Fix: the workflow now:

- prompts for ONLY a `SELECT`
- strips markdown fences and obvious prose lines
- validates the final SQL begins with `SELECT`

If it still fails, look at the Actions logs:

- **Raw LLM output**
- **Final SELECT used for task**

Then fix the prompt / sanitization based on what the model produced.

### “Show me the query result in pipeline output”

Snowflake `TASK` execution does not return a result set to the caller, so the workflow runs the generated `SELECT` separately with:

- `LIMIT 10`
- prints column names and up to 10 rows to the Actions logs

---

## 7) Notes / assumptions

- The ETL logic uses **LLM-generated SQL**. For production:
  - add stronger SQL validation / allow-listing
  - consider removing dynamic SQL generation or constrain it heavily
- Ensure your Snowflake role has privileges to:
  - read from source table
  - create/replace task
  - create/insert into target table

