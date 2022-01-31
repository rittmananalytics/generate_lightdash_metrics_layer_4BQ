# generate_lightdash_metrics_layer_4BQ

Auto-generates the Lightdash metrics layer schema.yml file

## Assumptions and Pre-requisites

1. Cloud data warehouse is Google Bigquery
- because of dependency on INFORMATION_SCHEMA, though it may still work with Snowflake etc
- suggested "bq query" method of executing the generated SQL statement, though snowsql will probably work too
- STRING and other BQ-specific datatypes, though these could be changed to VARCHAR etc if running on Snowflake
2. All metric columns have a datatype of FLOAT64 or NUMERIC

## Usage Instructions

Run this macro using the following CLI command, using the following parameter values:

- table_schema : BigQuery dataset name for which you want to generate the schema.yml for
- model_prefix : prefix, if any, you give models in dbt that are not used in the actual table name, for example in dbt your model may be called "wh_products_dim" but deployed in BigQuery as "products_dim"
- pk_suffix : the suffix, including any underscores, you give your primary key columns, for example "_pk"

```
dbt run-operation generate_metrics_schema --args '{table_schema: analytics, model_prefix: wh_, pk_suffix: _pk}'
```

To run the macro end-to-end, feeding the results into BigQuery and then outputting the resulting yaml as a schema.yml file:


```
 dbt run-operation generate_metrics_schema --args '{table_schema: analytics, model_prefix: wh_, pk_suffix: _pk}' | \
  tail -n +3 | \
  bq query -use_legacy_sql=false -format sparse | \
  tail -n +3 > schema.yml
 ```

 ## Example Output

 1. From running macro on its own, outputting the generated SQL to a file: [schema.sql](https://github.com/rittmananalytics/generate_lightdash_metrics_layer_4BQ/blob/main/example_output/schema.sql)

 ```
 dbt run-operation generate_metrics_schema --args '{table_schema: analytics, model_prefix: wh_, pk_suffix: _pk}' > schema.sql
 ```

 2. From executing this SQL and generating the yaml schema file: [schema.yml](https://github.com/rittmananalytics/generate_lightdash_metrics_layer_4BQ/blob/main/example_output/schema.yml)

 ## Next Steps

 After this you will still need to do the following with the schema.yml file:

 1. Add joins between models, for example:

 ```
 - name: wh_ad_campaigns_dim
  meta:
    label: "Wh Ad Campaigns Dim"
    joins:
      - join: wh_ad_campaign_performance_fact
        sql_on: ${wh_ad_campaigns_dim.ad_campaign_pk} = ${wh_ad_campaign_performance_fact.ad_campaign_pk}
      - join: wh_web_sessions_fact
        sql_on: ${wh_ad_campaigns_dim.ad_campaign_pk} = ${wh_web_sessions_fact.ad_campaign_pk} and ${wh_ad_campaign_performance_fact.campaign_date} = date(${wh_web_sessions_fact.session_start_ts})
      - join: wh_web_events_fact
        sql_on: ${wh_web_sessions_fact.web_sessions_pk} = ${wh_web_events_fact.web_sessions_pk}
```

2. Add any derived (calculated) measures
