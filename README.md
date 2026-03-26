
# dbt Cloud Demo
[![hackmd-github-sync-badge](https://hackmd.io/MttD1NxSTY-6U_nAWtCK6w/badge)](https://hackmd.io/MttD1NxSTY-6U_nAWtCK6w)

<p align="center">
  <img src="https://hackmd.io/_uploads/HJlu1BVYbl.png" width="600"/>
</p>

This demo explores the key enhancements introduced by the dbt Fusion Engine in the dbt interface (formerly dbt Cloud), with a particular focus on State-Aware Orchestration. Through hands-on examples, it demonstrates how these improvements impact project performance and execution behavior.

You can find the project used in this demo [here](https://github.com/danielryvero/dbt-datavault-demo) to follow along and evaluate the benefits of the Fusion Engine firsthand.  

## Table of Contents

1. [dbt parse & compile](#parse-compile)
2. [Broad SQL comprehension](#sql-comprehension)
3. [Preview CTE button](#preview-cte)
4. [Column lineage](#column-lineage)
5. [Suggestions](#suggestions)
6. [Go to Definition button](#go-to-definition)
7. [Static analysis](#static-analysis)
8. [Job runs](#job-runs)
   - [First job run](#first-job-run)
   - [Second job run](#second-job-run)
9. [State Awareness Orchestration – Model insert](#sao-model-insert)
   - [First SAO job run](#sao-model-first-run)
   - [Record insertion in a model](#model-record-insert)
   - [Second SAO job run](#sao-model-second-run) 
   - [Partial conclusions](#model-partial-conclusions)
10. [State Awareness Orchestration – Source insert](#sao-source-insert)
    - [Record insertion in a source](#source-record-insert)
    - [Source Configuration](#source-configuration)
    - [Partial conclusions](#source-partial-conclusions)
11. [SAO without loaded_at_field](#sao-no-loaded-at-field)
    - [Job run after source insert](#job-after-source-insert)
12. [Managing build frequency](#advanced-frequency)
13. [SAO triggered by code change](#sao-code-change)
    - [Job after code change](#job-after-code-change)
14. [Using loaded_at_query](#loaded-at-query)
    - [Insert within interval](#insert-within-interval)
    - [Job after insert within interval](#job-after-within-interval)
    - [Insert outside interval](#insert-outside-interval)
    - [Job after insert outside interval](#job-after-outside-interval)
15. [Insert in different source](#insert-different-source)
16. [How dbt detects inserts](#how-dbt-detects-inserts)
17. [Snowflake code snippets](#snowflake-code-snippets) 

## 1. dbt parse & dbt compile <a id="parse-compile"></a>

![compile_parse_comparison](https://hackmd.io/_uploads/SJRDN0Gobe.png)


At first glance, parsing and compile times in this 41-model Data Vault project show roughly a 1-second difference between the Fusion and Core dbt engines.

## 2. Broad SQL comprehension and faster parsing <a id="sql-comprehension"></a>
![chrome_CEOjclPBtp](https://hackmd.io/_uploads/rkbYUbXFbg.gif)
Because Fusion engine is a SQL compiler rather than a string processor, it marks SQL errors as soon as you save the model. In the Core version, these errors would only appear during the model run, causing a failed run and wasting warehouse resources.

Notice that warnings and errors appear in the "<span style="color:#2563eb;"><strong>Results</strong></span>" tab almost instantly after saving. The whole project parses much faster than before.

## 3. <span style="color:#2563eb;"><strong>Preview CTE</strong></span> button <a id="preview-cte"></a>

![chrome_Og3SNX0nAH](https://hackmd.io/_uploads/SJ_eoAR_be.gif)
This button appears when you save the model. dbt Fusion detects a new CTE and enables the button to quickly retrieve data from the table. It resolves the ***ref*** macro and shows the data without needing to navigate to the referenced model or comment out the rest of the code.

> Note that each time you click the button, a query runs against the warehouse, consuming credits. We recommend using this feature in development with a dedicated, cost-effective warehouse to facilitate optimization.

These are the queries that run in the warehouse when you click the "<span style="color:#2563eb;"><strong>Preview CTE</strong></span>" button:
![image 1](https://hackmd.io/_uploads/SkZsi0Cu-g.png)
Clicking the button again to review results—even after a short period—restarts the warehouse, so no Persisted Query result from the Snowflake metadata layer is used even if the data hasn't changed. This happens because dbt's observability framework tracks every operation at the metadata level and inserts an auditable record in ```"YOUR_DB_NAME"."OBSERVABILITY"."DBT_INVOCATIONS"```. 

## 4. Column lineage and references to it <a id="column-lineage"></a>

![chrome_DywN9SouVq](https://hackmd.io/_uploads/ryOVjARdWe.gif)
When you hover over a column, dbt shows its origin and data type. Right-click and select "<span style="color:#2563eb;"><strong>Go to References</strong></span>" to see which models use this column. In this case, it appears in the model initially selected in the lineage and its downstream model.

> Fusion retrieves this information from the warehouse's information schema, caches it, and reuses it without querying the warehouse again (on Snowflake).

## 5. Suggestions <a id="suggestions"></a>

![chrome_vUaK6MXgMl](https://hackmd.io/_uploads/HkoMnARdbl.gif)
When writing column names in a CTE, dbt Fusion knows which columns you can include. If you write a column name that's not in that model's columns, it underlines it in red and shows the available columns to choose from. 

> When you hover over the invalid column name, dbt displays possible replacements.

## 6. “<span style="color:#2563eb;"><strong>Go to Definition</strong></span>” button <a id="go-to-definition"></a>

![chrome_ezsE63vOoS](https://hackmd.io/_uploads/HJZS3RCdWx.gif)
Right-click on a column name and select "<span style="color:#2563eb;"><strong>Go to Definition</strong></span>" to jump directly to the CTE where that column originates. Hovering over the "*" in a ```SELECT *``` statement reveals which columns come from that CTE. 

## 7. Static analysis in work <a id="static-analysis"></a>

Due to the[`static_analysis`](https://docs.getdbt.com/reference/resource-configs/static-analysis) that the dbt Fusion engine performs over the models of the project, we get the primary benefits of the SQL comprehension of it. It builds a logical plan and knows what types are used in a query, and how they change, regardless of their origin. 

<div style="background-color:#1e1e1e; padding:12px; border-radius:6px; font-family:monospace;">
<span style="color:#ff4d4f;"><strong>dbt1000:</strong></span>
<span style="color:#fadb14;">
Detected unsafe introspection which may lead to non-deterministic static analysis. 
To suppress this warning, set static_analysis to 'unsafe' in the nodes' configuration. Learn more: [https://docs.getdbt.com/docs/fusion/new-concepts](https://docs.getdbt.com/docs/fusion/new-concepts). Nodes: 'model.demo_dv.HUB_CUSTOMER' (this), 'model.demo_dv.HUB_NATION' (this), 'model.demo_dv.HUB_ORDERS' (this), 'model.demo_dv.HUB_PART' (this), 'model.demo_dv.HUB_REGION' (this), 'model.demo_dv.HUB_SUPPLIER' (this), 'model.demo_dv.LNK_CUSTOMER_NATION' (this), 'model.demo_dv.LNK_LINEITEM_ORDERS_PART_SUPPLIER' (this), 'model.demo_dv.LNK_NATION_REGION' (this), 'model.demo_dv.LNK_ORDERS_CUSTOMER' (this) and 15 others
</span>
</div>

The dbt macro “**{{ this }}**” performs an introspective query, given that it consults the state of a model in the database. Since this operation depends on the result of something and not pure SQL code, the ”unsafe” warning appears. This only means that the models with that macro will be analyzed Just In Time (JIT) instead of Ahead Of Time (AOT). This is the default behavior of static analysis: non-introspective models  depend on AOT rendering and introspective models always use JIT rendering.

## 8. Job runs <a id="job-runs"></a>

### First Job Runs shows the following: <a id="first-job-run"></a>

                Fusion Engine Job                            Core Engine Job

![image 2](https://hackmd.io/_uploads/ryRWT0CdWl.png)

              Time elapsed: 2m, 1s                            Time elapsed: 3m, 22s

As shown in the screenshot, a standard job was run in each environment. With identical configuration, the dbt Fusion Engine completed in 2 minutes 1 second versus 3 minutes 22 seconds for the Core Engine run. Note that <span style="color:#9d2d1d; font-weight:600;">state-aware orchestration </span> is not enabled—this will be analyzed later. 

After a first run, a record is inserted in a source table:
![image 3](https://hackmd.io/_uploads/rJ3OpR0_-e.png)

<a href="#record-insertion-model10"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

### Second Job Run shows similar results: <a id="second-job-run"></a>

                Fusion Engine Job                            Core Engine Job

![image 4](https://hackmd.io/_uploads/S1PipACdZx.png)

            Time elapsed: 2m, 32s                              Time elapsed: 3m, 33s

Note that there is an extra <span style="color:#9d2d1d; font-weight:600;">dbt compile</span> command, that adds 11 extra seconds to the Job run. This command is updating the applied state or logical state of the model after the doc generation, granting an updated <span style="color:#f59e0b; font-weight:600;">manifest.json</span> and <span style="color:#f59e0b; font-weight:600;">run_results.json</span> file. dbt generates the executable SQL (from the source model, test, and analysis files) but does not run it. This ensures the changes are reflected in the manifest for the next time a CI job is run and keeps track of only the relevant changes.

## 9. State Awareness Orchestration (SAO) enabled (Record inserted in Model) <a id="sao-model-insert"></a>

Following the [Prerequisites](https://www.notion.so/dbt-Cloud-Fusion-2f6a318a6173806b8c08fc06f3023d7e?pvs=21) to use the <span style="color:#9d2d1d; font-weight:600;">state-aware orchestration </span>, the following Job was set up and run in a Staging environment with dbt Fusion Engine (Job Config is shown in the rightmost part).

### First Fusion <span style="color:#9d2d1d; font-weight:900;">State-Awared </span> Job run: <a id="sao-model-first-run"></a>

![image 5](https://hackmd.io/_uploads/rymeCRAOZe.png)

### Record insertion in a Model: <a id="model-record-insert"></a>
After that run, a record was inserted in one of the models, ```EDW_DV_DEMO.RAW_DATAVAULT.HUB_CUSTOMER``` (this is a model, not a source):
![image 6](https://hackmd.io/_uploads/Syk7CAR_bl.png)

<a href="#record-insertion-model9"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

### Second Fusion <span style="color:#9d2d1d; font-weight:900;">State-Awared </span> Job run: <a id="sao-model-second-run"></a>

A second Run of the same Job was executed to attempt load the new data into the models.

          First Fusion Engine Job                       After-insertion Fusion Engine Job

![image 7](https://hackmd.io/_uploads/HJnHA0A_Wx.png)


Job logs show that no models were built:
![chrome_wzMAIgRTrQ](https://hackmd.io/_uploads/rkm_C0ROWe.gif)

> When no models are built, the model run time is just 38 seconds for this project. 

Model timing for both Jobs:

        First Fusion Engine Job                         After-insertion Fusion Engine Job

![image 8](https://hackmd.io/_uploads/ryYoC0AObg.png)


### Partial Conclusions <a id="model-partial-conclusions"></a>

These results were unexpected—the job didn't rebuild all models downstream of <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span>. This happens because <span style="color:#9d2d1d; font-weight:600;">state-awareness </span> checks the source tables to identify new changes, so data must be inserted there for this feature to work. 

Selecting the inserted CUSTOMER_KEY from <span style="color:#4ade80; font-weight:600;">CUSTOMER_BAL</span> reveals that the record doesn't appear in the table. This confirms that models downstream of <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span> weren't rebuilt—the record didn't make it to <span style="color:#4ade80; font-weight:600;">CUSTOMER_BAL</span>.
![image 9](https://hackmd.io/_uploads/H1SS1JJtbl.png)

## 10. State Awareness Orchestration (SAO) enabled (Record inserted in Source) <a id="sao-source-insert"></a>

### Record insertion in a Source <a id="source-record-insert"></a>

A record was inserted into the Source table (```EDW_DV_DEMO.LANDING.CUSTOMER```) where the <span style="color:#b08968; font-weight:600;">loaded_at_field</span> was configured in the <span style="color:#f59e0b; font-weight:600;">source.yml</span>: 
![image 10](https://hackmd.io/_uploads/rJGOJyJtbl.png)

<a href="#record-insertion-model10"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

### Source Configuration <a id="source-configuration"></a>

<div style="display:flex; gap:20px; align-items:flex-start;">

<div style="flex:1;">

The <span style="color:#b08968; font-weight:600;">loaded_at_field</span> was set—as it should be—at the source level, referencing a table field to measure freshness and track when records are inserted into the table. 
</br>
</br>
</br>
We could have a <span style="color:#b08968; font-weight:600;">loaded_at_query</span> field with custom SQL logic to define a custom freshness condition instead of the <span style="color:#b08968; font-weight:600;">loaded_at_field</span> used this time.  
</div>

<div style="flex:1;">

![image 11](https://hackmd.io/_uploads/B1XJlk1tWl.png)

</div>

</div>

After this, the Job ran differently:
![chrome_fHckZmAR8C](https://hackmd.io/_uploads/HJl031kyFZg.gif)

Models that were run match the list of downstream models from source ```SF_SAMPLE.CUSTOMER``` in the lineage image below:
![image 12](https://hackmd.io/_uploads/B1YexJyYWx.png)

In the following image, we can see the first record we inserted (into <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span>) and the second record we inserted (into ```SF_SAMPLE.CUSTOMER```), note that CUSTOMER_KEY has the values 30007, 150002 and 150003:
![image 13](https://hackmd.io/_uploads/rJmWlJktbe.png)

### Partial Conclusions <a id="source-partial-conclusions"></a>

When a new record is inserted into a source while <span style="color:#9d2d1d; font-weight:600;">state-aware orchestration </span> is enabled for the Job, all downstream models of that source are rebuilt, while others are skipped. Inserting such record in a downstream model won’t trigger this feature since the metadata detection works at the source level.
In this example, no freshness criteria (this will be addressed later on) were defined for the downstream models, so dbt rebuilds them instantly. However, if freshness conditions were set, the dbt Fusion Engine would evaluate them and only run the models necessary to satisfy the defined Service Level Agreements (SLAs).

## 11. <span style="color:#9d2d1d; font-weight:600;">State-Awareness </span> testing in source without <span style="color:#b08968; font-weight:900;">loaded_at_field</span> configured in <span style="color:#f59e0b; font-weight:600;">source.yml</span> <a id="sao-no-loaded-at-field"></a>

This is the <span style="color:#f59e0b; font-weight:600;">source.yml</span> file for the ```EDW_DV_DEMO.LANDING.NATION``` table. This table doesn’t have a timestamp field and therefore it doesn’t have the <span style="color:#b08968; font-weight:600;">loaded_at_field</span> configured. 
![image 14](https://hackmd.io/_uploads/SkUGe1kK-g.png)

Record insertion in source table ```EDW_DV_DEMO.LANDING.NATION```: 
![image 15](https://hackmd.io/_uploads/HyOQe1JtWg.png)

<a href="#record-insertion-model11"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

### Job Run after record insertion in source: <a id="job-after-source-insert"></a>
![image 16](https://hackmd.io/_uploads/HyWExkkYZl.png)

The record insertion was detected and only the downstream models of that source were run. This happens because <span style="color:#9d2d1d; font-weight:600;">state-aware orchestration </span> uses shared state tracking to determine which models need to be built by detecting changes in code or data every time a job runs. It works out-of-the-box (natively), with an optional configuration setting for more advanced controls, that were not set for this source table.

## 12. Managing building frequency with advanced parameters: <a id="advanced-frequency"></a>

The configuration for the <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span> model is now this:

```yaml
models:
  - name: HUB_CUSTOMER
    description: "HUB customer table"

    config:
      access: public
      contract:
        enforced: true
      freshness: 
        build_after: 
          count: 1
          period: hour
          updates_on: any
```

Notice the freshness configuration for the model. The build_after parameter was set to <span style="color:#b784c7; font-weight:600;">count: 1</span>, <span style="color:#b784c7; font-weight:600;">period: hour</span> and <span style="color:#b784c7; font-weight:600;">updates_on: any</span>. This means that this model will only be built if it has new data in any of its upstream models (in this case, the only upstream model is the source) or has a code change and it was not built in the last hour.

> For this configuration to take effect, a commit into the branch must be done, because the <span style="color:#9d2d1d; font-weight:600;">state-aware orchestration </span> works in a STG environment that is running off a custom branch, so changes must be reflected in that branch for the Job to recognize the config, otherwise it will assume the default config and rebuild the model.

A new record is inserted into the source table ```EDW_DV_DEMO.LANDING.CUSTOMER``` at <span style="color:#fb923c; font-weight:500;">2026-02-23 13:06:35</span>: 
![image 17](https://hackmd.io/_uploads/rJFjlkXtZe.png)

<a href="#record-insertion-model9"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

Job triggering before <span style="color:#b784c7; font-weight:600;">build_after</span> count hits:
![chrome_SWGhH8QwHf](https://hackmd.io/_uploads/SydRwZXYWx.gif)

The <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span> model was not rebuilt. Others were built because they use the very same ```EDW_DV_DEMO.LANDING.CUSTOMER``` table as a source, but the one we configured stated this in the resulting log: 

```bash
12:07:07
Started model RAW_DATAVAULT.HUB_CUSTOMER
12:07:07
Reused [-------] model RAW_DATAVAULT.HUB_CUSTOMER (incremental - New changes detected. 
Did not meet build_after of 1h. Last updated 42m 20s ago)
```

This proves the serverity of the <span style="color:#b784c7; font-weight:600;">build_after</span> config for models, allowing to decide further conditions than the freshness to build models. The <span style="color:#b784c7; font-weight:600;">updates_on</span> parameter allows to make this configuration dependent on all the sources of the selected model as well, assuring SLAs delivery.

## 13. Checking that <span style="color:#9d2d1d; font-weight:900;">State-Awareness </span> is also triggered by a code change in a model. <a id="sao-code-change"></a>

The following demonstration includes adding code to the existing model <span style="color:#4ade80; font-weight:600;">CUSTOMER_BAL</span> and committing the changes into the STG branch, where a <span style="color:#9d2d1d; font-weight:600;">State-Awared </span> job has been set.
![chrome_D10YHyZh0l](https://hackmd.io/_uploads/BJV19-7tWx.gif)


### <span style="color:#9d2d1d; font-weight:900;">State-Awared </span> Job triggering after code change: <a id="job-after-code-change"></a>

![chrome_FhBJbrj0yE](https://hackmd.io/_uploads/H1JJ-kXFbl.gif)

The only model that was built was in fact <span style="color:#4ade80; font-weight:600;">CUSTOMER_BAL</span> and the dependencies downstream of it (tests and on-run hooks). This occurs because the <span style="color:#9d2d1d; font-weight:600;">State-Awareness </span> runs if there is new data in the source and time constraints for model building comply or if there is a code change in any of the models, indicating a change in the compiled code of the model.

## 14. Using <span style="color:#b08968; font-weight:900;">loaded_at_query</span> instead of <span style="color:#b08968; font-weight:900;">loaded_at_field</span> <a id="loaded-at-query"></a>

The following configuration has been set for the HUB_CUSTOMER model:
![image 18](https://hackmd.io/_uploads/S1llbJmKbx.png)

### Record insertion within interval defined by <span style="color:#b08968; font-weight:600;">loaded_at_query</span> parameter <a id="insert-within-interval"></a>

A record is inserted into the source table ```EDW_DV_DEMO.LANDING.CUSTOMER``` (CUSTOMER_KEY = 150009) at <span style="color:#fb923c; font-weight:500;">2026-02-25 09:34:36</span>: 
![image 19](https://hackmd.io/_uploads/S11WZkmKZx.png)

<a href="#record-insertion-model9"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

### Job running after record insertion within interval defined <a id="job-after-within-interval"></a>

The <span style="color:#9d2d1d; font-weight:600;">State-Awared </span> Job runs at <span style="color:#fb923c; font-weight:500;">2026-02-25 09:35:55</span>, almost instantly after the record was inserted. All models downstream <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span> (including it) were rebuilt:

![chrome_kQuDGCGnz9](https://hackmd.io/_uploads/r14f-yXt-e.gif)

### Record insertion outside interval defined by <span style="color:#b08968; font-weight:600;">loaded_at_query</span> parameter <a id="insert-outside-interval"></a>

A second record is inserted into ```EDW_DV_DEMO.LANDING.CUSTOMER```  (CUSTOMER_KEY = 150011) at <span style="color:#fb923c; font-weight:500;">2026-02-25 09:59:07</span>:
![image 20](https://hackmd.io/_uploads/ryD7b1QtZe.png)

<a href="#record-insertion-model9"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

### Job running after record insertion outside interval defined <a id="job-after-outside-interval"></a>

The <span style="color:#9d2d1d; font-weight:600;">State-Awared </span> Job runs at <span style="color:#fb923c; font-weight:500;">2026-02-25 10:22:04</span>, about 23 minutes after the record was inserted. No models were built because no new changes were detected. This happened because of the <span style="color:#b08968; font-weight:600;">loaded_at_query</span> field limitting the checked data to be only of the last 5 minutes:
![chrome_agXCrUuWUN](https://hackmd.io/_uploads/BkuVbkQKbl.gif)

## 15. Inserting a record in a different source than the one with the <span style="color:#b08968; font-weight:600;">loaded_at_query</span> field configured <a id="insert-different-source"></a>

In this example, a record was inserted into the ```EDW_DV_DEMO.LANDING.ORDERS``` (ORDER_KEY = 6000001):
![image 21](https://hackmd.io/_uploads/SJES-J7tWx.png)

<a href="#record-insertion-model15"
   style="background:#2563eb; color:white; padding:1px 12px; 
   border-radius:6px; text-decoration:none; font-weight:600;">
   Go to Code Snippet
</a>

The <span style="color:#9d2d1d; font-weight:600;">State-Awared </span> Job was kicked off immediately afterwards. Models downstream of the ```EDW_DV_DEMO.LANDING.ORDERS``` source table were run, this is not the case of the <span style="color:#4ade80; font-weight:600;">HUB_CUSTOMER</span> model, because the previous inserted record had been inserted more than 5 minutes ago:
![chrome_CznTovQ0tl](https://hackmd.io/_uploads/BJZlj-7YZg.gif)

## 16. How does dbt detect insertion of records into tables? <a id="how-dbt-detects-inserts"></a>

The following analysis comes right after the last Job we kicked, where we had a condition set by a <span style="color:#b08968; font-weight:600;">loaded_at_query</span>  field and we inserted a record into another source that didn’t have a <span style="color:#b08968; font-weight:600;">loaded_at_</span> field.

When kicking off a Job, there are a series of queries we can analyze in the Warehouse used for it. The first one is this, where dbt checks the INFORMATION_SCHEMA and extracts the <span style="color:#b08968; font-weight:600;">last_altered</span> tables from TABLES:

```sql
SELECT
table_schema,
table_name,
last_altered,
(table_type = 'VIEW' OR table_type = 'MATERIALIZED VIEW') AS is_view
FROM "EDW_DV_DEMO".INFORMATION_SCHEMA.TABLES
WHERE table_schema = 'LANDING' and table_name = 'LINEITEM' 
	OR table_schema = 'LANDING' and table_name = 'NATION' 
	OR table_schema = 'LANDING' and table_name = 'ORDERS' 
	OR table_schema = 'LANDING' and table_name = 'PART' 
	OR table_schema = 'LANDING' and table_name = 'PARTSUPP' 
	OR table_schema = 'LANDING' and table_name = 'REGION' 
	OR table_schema = 'LANDING' and table_name = 'SUPPLIER'
```

Then, the second query is this one, where dbt enforces the condition set for the CUSTOMER source table:

```sql
with source_query as (
select max(DT_LOAD)
from EDW_DV_DEMO.LANDING.CUSTOMER
where DT_LOAD >= current_timestamp - interval '5 minutes'
)
select
(select * from source_query) as max_loaded_at,
convert_timezone('UTC', current_timestamp()) as snapshotted_at
```

Then it executes a ```DESCRIBE TABLE``` for each of the tables in the project, to update the metadata it has about them: 
![image 22](https://hackmd.io/_uploads/BJPjfZQFbe.png)

After using ```DESCRIBE TABLE``` and ```SHOW OBJECTS IN SCHEMA``` commands, the first command used to indeed select records that will be used downstream is this:

```sql
create or replace  
	temporary view EDW_DV_DEMO.RAW_DATAVAULT.HUB_ORDERS__dbt_tmp as (
WITH ORDERS AS (
	SELECT
		MD5(O_ORDERKEY) HUB_ID_ORDERS
		, O_ORDERKEY AS ORDER_KEY
		, 'SDG_ES' AS TENANT
		, CURRENT_TIMESTAMP::TIMESTAMP_NTZ AS DT_LOAD
		, 'EDW_DV_DEMO.LANDING.ORDERS' AS RECORD_SOURCE
	FROM EDW_DV_DEMO.LANDING.ORDERS
	)
SELECT
	ORDERS.HUB_ID_ORDERS
	, ORDERS.ORDER_KEY
	, ORDERS.TENANT
	, ORDERS.DT_LOAD
	, ORDERS.RECORD_SOURCE
FROM ORDERS ORDERS
LEFT JOIN EDW_DV_DEMO.RAW_DATAVAULT.HUB_ORDERS HUB
	ON HUB.HUB_ID_ORDERS =  ORDERS.HUB_ID_ORDERS
WHERE HUB.HUB_ID_ORDERS IS NULL);
```

This command creates a temporary view that selects the newly inserted record into the source. There is internal Engine processing based on the <span style="color:#b08968; font-weight:600;">last_altered</span> field queried at the beginning of the Job. This proves the efficiency of dbt when consuming metadata from the queries run against the Warehouse to detect the changes in source tables.
</br>
</br>

# Code snippets used in Snowflake <a id="snowflake-code-snippets"></a>
### 1. Record insertion in a Model:<a id="record-insertion-model9"></a>
```sql
INSERT INTO EDW_DV_DEMO.RAW_DATAVAULT.HUB_CUSTOMER (
HUB_ID_CUSTOMER, CUSTOMER_KEY, TENANT, DT_LOAD, RECORD_SOURCE)
SELECT
    MD5(150001), --Choose ID to insert (>150000)
    150001,      --Chosen ID
    'SDG_ES_CORE',
    CAST(CURRENT_TIMESTAMP AS TIMESTAMP_NTZ),
    'EDW_DV_DEMO.LANDING.CUSTOMER';
```
Checking record insertion:
```sql
SELECT *
FROM EDW_DV_DEMO.RAW_DATAVAULT.HUB_CUSTOMER
WHERE CUSTOMER_KEY = 150001 --Enter ID inserted above
;
```

### 2. Record inserted in a Source:<a id="record-insertion-model10"></a>
```sql
INSERT INTO EDW_DV_DEMO.LANDING.CUSTOMER (
C_CUSTKEY, C_NAME, C_ADDRESS, C_NATIONKEY,
C_PHONE, C_ACCTBAL, C_MKTSEGMENT, C_COMMENT,
DT_LOAD
)
SELECT
    150002,
    'Customer#000150002',
    'DD7m6OBUJqlbTpiYmzK9SYuJBiuasfhius', --Random description
    21,
    '20-222-421-2026',                    --Random phone number
    2222.08,
    'BUILDING',
    'manually inserted record for state awareness 2',
    CAST(CURRENT_TIMESTAMP AS TIMESTAMP_NTZ);
```

Checking record insertion:
```sql
SELECT 
    DT_LOAD, 
    C_CUSTKEY, 
    C_NAME, 
    C_ADDRESS, 
    C_PHONE
FROM EDW_DV_DEMO.LANDING.CUSTOMER 
WHERE C_CUSTKEY = 150002;
```

### 3. Inserting record in a source without <span style="color:#b08968; font-weight:600;">loaded_at_field</span>:<a id="record-insertion-model11"></a>
```sql
INSERT INTO EDW_DV_DEMO.LANDING.NATION (
    N_NATIONKEY, N_NAME, N_REGIONKEY, N_COMMENT)
SELECT 
    25, 
    'CUBA', 
    1, 
    'test country. bad experiment';
```
Check record insertion:
```sql
SELECT * 
FROM EDW_DV_DEMO.LANDING.NATION
WHERE N_NAME = 'CUBA';
```
### 4. Source configuration for <span style="color:#b08968; font-weight:600;">loaded_at_query</span> field:<a id="record-insertion-model14"></a>
```yml
        config:
          loaded_at_query: |
            select max(DT_LOAD)
            from {{ this }}
            where DT_LOAD >= current_timestamp - interval '5 minutes'
```            
### 5. Inserting record in a different source than the one with the <span style="color:#b08968; font-weight:600;">loaded_at_query</span> field configured:<a id="record-insertion-model15"></a>
``` sql
INSERT INTO EDW_DV_DEMO.LANDING.ORDERS (
	O_ORDERKEY,	O_CUSTKEY, O_ORDERSTATUS,
	O_TOTALPRICE, O_ORDERDATE, O_ORDERPRIORITY,
	O_CLERK, O_SHIPPRIORITY, O_COMMENT
    )
SELECT 
    6000001,
    104548,
    'O',
    2245.29,
    TO_DATE('31/08/1996', 'DD/MM/YYYY'),
    '2-HIGH',
    'Clerk#000000411',
    0,
    'comment for extra record inserted in orders 1';
```
Checking record insertion:
```sql
SELECT *
FROM EDW_DV_DEMO.LANDING.ORDERS
WHERE O_ORDERKEY =(
    SELECT MAX(O_ORDERKEY) FROM EDW_DV_DEMO.LANDING.ORDERS);
```