This file contains my answers to the questions of homework #3

## Question 1: Context Engineering

Try the following experiment:

1. Open ChatGPT in a private browser window: https://chatgpt.com
2. Enter this prompt: "Create a Kestra flow that loads NYC taxi data from CSV to BigQuery"
3. Then, use Kestra's AI Copilot with the same prompt

After trying the same prompt in ChatGPT vs Kestra's AI Copilot, what is the primary reason AI Copilot generates better Kestra flows?

- AI Copilot uses a more powerful model
- AI Copilot has access to current Kestra plugin documentation
- AI Copilot uses more tokens
- AI Copilot has internet access

### Answer from ChatGPT
```
id: nyc_taxi_csv_to_bigquery
namespace: zoomcamp

variables:
  project_id: "{{ envs.GCP_PROJECT_ID }}"
  dataset: ny_taxi
  table: yellow_tripdata_2024_01
  bucket: "{{ envs.GCP_BUCKET }}"
  file: yellow_tripdata_2024-01.csv.gz

tasks:
  - id: download_csv
    type: io.kestra.plugin.core.http.Download
    uri: "https://github.com/DataTalksClub/nyc-tlc-data/releases/download/yellow/{{ vars.file }}"

  - id: create_dataset
    type: io.kestra.plugin.gcp.bigquery.Query
    projectId: "{{ vars.project_id }}"
    serviceAccount: "{{ secret('GCP_CREDS') }}"
    sql: |
      CREATE SCHEMA IF NOT EXISTS `{{ vars.project_id }}.{{ vars.dataset }}`;

  - id: load_to_bigquery
    type: io.kestra.plugin.gcp.bigquery.Load
    from: "{{ outputs.download_csv.uri }}"
    destinationTable: "{{ vars.project_id }}.{{ vars.dataset }}.{{ vars.table }}"
    format: CSV
    csvOptions:
      fieldDelimiter: ","
      skipLeadingRows: 1
      allowQuotedNewLines: true
    autodetect: true
    writeDisposition: WRITE_TRUNCATE
```

### Answer with minimaxai/minimax-m3
```
id: nyc-taxi-csv-to-bigquery
namespace: data-engineering

description: >
  Loads NYC Yellow Taxi trip data from a public CSV source into BigQuery.
  Designed for monthly batch backfills.

inputs:
  - id: year
    type: STRING
    defaults: "2024"

  - id: month
    type: STRING
    defaults: "01"

  - id: taxi_type
    type: STRING
    defaults: "yellow"
    description: "yellow | green | fhvhv"

  - id: gcs_bucket
    type: STRING
    defaults: "nyc-taxi-raw-data"

  - id: bq_project
    type: STRING
    required: true

  - id: bq_dataset
    type: STRING
    defaults: "nyc_taxi"

  - id: bq_table
    type: STRING
    defaults: "trips"

tasks:
  # 1. Resolve the source URL for the requested month
  - id: build_url
    type: io.kestra.core.tasks.debugs.Return
    format: "{{ render(vars.taxi_type + '/tripdata/' + vars.taxi_type + '_tripdata_' + vars.year + '-' + vars.month + '.parquet') }}"

  # 2. Download the file from the NYC TLC public bucket
  - id: download_raw
    type: io.kestra.plugin.gcp.gcs.Download
    from: "{{ 'https://d37ci6vzurychx.cloudfront.net/trip-data/' + render(vars.taxi_type + '_tripdata_' + vars.year + '-' + vars.month + '.parquet') }}"
    bucket: "{{ vars.gcs_bucket }}"
    key: "raw/{{ vars.taxi_type }}/{{ vars.year }}/{{ vars.month }}/data.parquet"

  # 3. Convert Parquet to newline-delimited JSON (NDJSON)
  - id: parquet_to_json
    type: io.kestra.plugin.scripts.python.Script
    inputFiles:
      main.py: |
        import pandas as pd
        df = pd.read_parquet("/data/data.parquet")
        df.columns = [c.lower() for c in df.columns]
        df.to_json("/data/data.ndjson", orient="records", lines=True, date_format="iso")
    containerImage: "ghcr.io/kestra-io/pyarrow:latest"
    inputDirs:
      - /data
    outputDirs:
      - /data

  # 4. Load NDJSON into a BigQuery staging table
  - id: load_to_bigquery
    type: io.kestra.plugin.gcp.bigquery.Load
    from: "{{ outputs.parquet_to_json.outputFiles['/data/data.ndjson'] }}"
    destinationTable: "{{ vars.bq_project }}.{{ vars.bq_dataset }}.{{ vars.bq_table }}_staging"
    writeDisposition: WRITE_TRUNCATE
    sourceFormat: "NEWLINE_DELIMITED_JSON"
    autodetect: true
    projectId: "{{ vars.bq_project }}"

  # 5. MERGE staging into the production table (deduplicates by primary key)
  - id: merge_into_production
    type: io.kestra.plugin.gcp.bigquery.Query
    sql: |
      MERGE `{{ vars.bq_project }}.{{ vars.bq_dataset }}.{{ vars.bq_table }}` T
      USING `{{ vars.bq_project }}.{{ vars.bq_dataset }}.{{ vars.bq_table }}_staging` S
      ON T.vendorid = S.vendorid
         AND T.tpep_pickup_datetime = S.tpep_pickup_datetime
         AND T.tpep_dropoff_datetime = S.tpep_dropoff_datetime
         AND T.pulocationid = S.pulocationid
      WHEN NOT MATCHED THEN
        INSERT ROW
    projectId: "{{ vars.bq_project }}"

  # 6. Clean up the staging table
  - id: drop_staging
    type: io.kestra.plugin.gcp.bigquery.Query
    sql: "DROP TABLE `{{ vars.bq_project }}.{{ vars.bq_dataset }}.{{ vars.bq_table }}_staging`"
    projectId: "{{ vars.bq_project }}"

# Schedule for monthly backfills (runs at 06:00 on the 8th day of each month
# — the public parquet files are typically refreshed around day 5–7)
triggers:
  - id: monthly_schedule
    type: io.kestra.core.models.triggers.types.Schedule
    cron: "0 6 8 * *"
    inputs:
      year: "{{ trigger.date | date('yyyy') }}"
      month: "{{ (trigger.date | date('MM') | int - 1) | string | zfill(2) }}"
```

### Answer from AI Copilot
```
id: nightingale_701885
namespace: zoomcamp

tasks:
  - id: download_nyc_taxi_data
    type: io.kestra.plugin.core.http.Download
    uri: https://storage.googleapis.com/kestra-example-files/nyc_taxi_data.csv
    saveAs: nyc_taxi_data.csv
  - id: load_to_bigquery
    type: io.kestra.plugin.gcp.bigquery.Load
    from: "{{outputs.download_nyc_taxi_data.uri}}"
    destinationTable: your_project.your_dataset.your_table # IMPORTANT: Replace with your actual BigQuery project, dataset, and table
    format: CSV
    autodetect: true
    csvOptions:
      fieldDelimiter: ","
```

The answers from ChatGPT and AI Copilot are incredibly similar. The ChatGPT version attmeps to create the table if it doesn't already exist, whereas the Copilot AI assumes it is already there. I think both versions would work if I had a Google Cloud project setup to accept the data. The Minimax-m3 version on the other hand is quite bizarre. It clearly uses code from GitHub for the data engineering class. 

This leads me to the answer: B. The AI Copilot has access to the current Kestra plugin documentation, and that documentation contains up to date information on how to use Kestra's plugins. It also doesn't seem to hallucinate as much and add confusing code that wasn't requested.


## Question 2: RAG vs No RAG

Run both `1_chat_without_rag.yaml` and `2_chat_with_rag.yaml` in the Kestra UI. Read the execution logs for each.

The non-RAG response about Kestra 1.1 features is best described as:

- Accurate and specific, matching the actual release notes
- Vague, generic, or fabricated — the model guesses from training data
- Empty — the model refuses to answer without context
- Identical to the RAG version

### Answer to chat_without_rag:
```
Kestra 1.1 introduced several exciting features that significantly enhanced its capabilities. Here are at least 5 major features with brief descriptions:
1. **Task Type Selection Helper:** This was a significant UX improvement. Instead of manually knowing and typing the `type:` for each task, users can now leverage an **autocomplete and suggestion feature** directly within the Kestra UI and YAML editor. As you start typing `type:`, Kestra provides a dropdown list of available task types, making it much easier and faster to write workflows, especially for new users or when exploring less familiar tasks.
2. **Plugin Management UI:** Kestra 1.1 introduced a dedicated **user interface for managing plugins**. This means administrators and users with appropriate permissions no longer need to rely solely on command-line tools or file system interactions to install, update, or remove Kestra plugins. The UI provides a centralized and more accessible way to view installed plugins, their versions, and potentially trigger installation/uninstallation processes.
3. **Authentication UI:** This release added a **user interface for managing authentication methods and users within Kestra**. Prior to 1.1, configuring authentication typically involved editing configuration files directly. The Authentication UI streamlines this process, allowing administrators to manage users, roles, and potentially different authentication providers (like LDAP, OIDC, etc.) through a graphical interface, improving security management and ease of use.
4. **Tenant Management UI:** Kestra 1.1 introduced **UI support for managing tenants**, which are isolated workspaces within a single Kestra instance. This feature is crucial for multi-tenancy scenarios where different teams or departments need their own Kestra environments without interfering with each other. The Tenant Management UI allows administrators to create, manage, and assign resources to different tenants, simplifying the setup and oversight of isolated environments.
5. **Blueprint Generation & Management (Preview):** While likely in a preview state, Kestra 1.1 started laying the groundwork for **Blueprint generation and management**. Blueprints are essentially pre-defined, reusable workflow templates that can be easily shared and instantiated. This feature aims to accelerate workflow development by providing a library of common patterns and solutions, allowing users to quickly create new workflows from existing, validated templates rather than starting from scratch. These features collectively made Kestra 1.1 much more user-friendly, administratively robust, and scalable, especially for larger organizations and teams.
```

### Answer to chat_with_rag:
```
Kestra 1.1 introduced several major features, including:
1. **New Filters**: The UI filters were completely redesigned for improved usability, offering explicit filter options, single-click resets, the ability to save frequently used filter combinations, and customizable table columns.
2. **No-Code Dashboard Editor**: This feature allows users to create and edit dashboards using a no-code, multi-panel editor directly from the UI, similar to the existing no-code flow editor. Users can design monitoring dashboards, business intelligence views, or executive reports without writing YAML.
3. **Human Task**: For Enterprise Edition users, this feature enables human-in-the-loop workflows. Executions can be paused and require manual approval from specific users or groups before proceeding, useful for critical operations like data quality checks or infrastructure changes.
4. **Multi-Agent AI Systems**: AI agents can now use other AI agents as tools, facilitating sophisticated multi-agent orchestration workflows where a primary agent can delegate subtasks to specialized expert agents.
5. **Fix with AI**: When tasks fail, Kestra 1.1 provides AI-powered suggestions to help diagnose and resolve issues, speeding up troubleshooting and getting executions back on track.
6. **Dozens of New Plugins**: The release brought numerous new community-contributed plugins, expanding integrations across various categories like Data & Database (Liquibase, dlt), SaaS & API (Airtable, Stripe, Shopify), Cloud & Infrastructure (Dataform, AWS CloudWatch), and AI Model Providers (OCI GenAI, Cloudflare Workers AI, LocalAI, DashScope, ZhipuAI).
```

Based on those two output, the answer is B. The version without RAG is indeed vague and generic...

## Question 3: Token usage — short summary

Run `4_simple_agent.yaml` with `summary_length = short` (leave the other inputs as defaults).

Open the execution logs and find the token usage logged by the `log_token_usage` task.

What is the approximate **output** token count for `multilingual_agent`?

- 5-15 tokens
- 60-100 tokens
- 200-400 tokens
- 500+ tokens

Ok I ran the code and I got the following results:
```
INFO 2026-07-01 16:20:34.124📊 Token Usage Summary:

Multilingual Agent:
- Input tokens: 282
- Output tokens: 70
- Total tokens: 352

English Brevity Agent:
- Input tokens: 85
- Output tokens: 34
- Total tokens: 119

💡 Tip: Monitor token usage to understand costs and optimize prompts!
```

The number of output tokens was 70. This means the answer is B (60-100 tokens)

## Question 4: Token usage — long summary

Run `4_simple_agent.yaml` again with `summary_length = long`.

Compare the `multilingual_agent` output token count to your result from Question 3. Roughly how many times more output tokens does the long summary use?

- About the same (within 20%)
- 2-5x more
- 10-20x more
- 50x more

I get the following results:
```
INFO 2026-07-01T23:23:47.122186Z 📊 Token Usage Summary:

Multilingual Agent:
- Input tokens: 282
- Output tokens: 199
- Total tokens: 481

English Brevity Agent:
- Input tokens: 214
- Output tokens: 54
- Total tokens: 268

💡 Tip: Monitor token usage to understand costs and optimize prompts!
```

The number of output tokens for the long version is 199. This is 2.84 times more than for the short version. This means the answr is B (2-5x more).

## Question 5: Modifying a flow

Open `4_simple_agent.yaml` in the Kestra flow editor. Find the `english_brevity` task and change its prompt from asking for exactly **1 sentence** to asking for exactly **3 sentences**.

Save the flow, then run it with `summary_length = long`.

Compare the `english_brevity` output token count to the original 1-sentence version (also with `summary_length = long`). How do they compare?

- About the same (within 20%)
- 2-4x more
- 5-10x more
- 10x+ more

Ok I made the modification and ran the code. Here is the output:
```
INFO 2026-07-01T23:38:29.060391Z 📊 Token Usage Summary:

Multilingual Agent:
- Input tokens: 282
- Output tokens: 180
- Total tokens: 462

English Brevity Agent:
- Input tokens: 195
- Output tokens: 87
- Total tokens: 282

💡 Tip: Monitor token usage to understand costs and optimize prompts!
```

The number of output tokens changed from 54 to 87. That's not quite a factor fo 2, but not the same.

## Question 6: Best Practices

Based on what you learned in this module, for production workflows requiring deterministic, repeatable results with strict compliance requirements (e.g., financial reporting, workflows in highly regulated industries), which approach is most appropriate?

- Always use AI agents for maximum flexibility and adaptation
- Use traditional task-based workflows for predictability and auditability
- Use only RAG without agents for better performance
- Use web search tools exclusively to ensure current data
