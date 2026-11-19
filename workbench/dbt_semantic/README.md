# DemoEd dbt Semantic Layer


> Technical documentation for the **`dbt_semantic`** project built using dbt 1.12.5 over the DemoEd enterprise data warehouse. This README documents the SQL models, semantic definitions, metrics, dimensions, custom calendar, and local MetricFlow CLI, but not the data warehouse models.
>


## Model inventory

All  models  use `{{ config(materialized='view', bind=False) }}` based on the tables in the enterprise data warehouse declared as sources. 

| dbt SQL model | Semantic name | Grain / purpose | Redshift source relations |
| --- | --- | --- | --- |
| `vws_account.sql` | `account` | Conformed account lookup and account hierarchy; one row per account | `common.dim_account`, `reporting.dim_account_metrics` |
| `vws_arr_account.sql` | `arr` | One row per `account_id` / `mon_year` / `mon_lastday`; ARR variants pre-aggregated separately | `revenue.vw_fact_arr` |
| `vws_licenses_account.sql` | `licenses` | Monthly, district-account license snapshot | `licensing.vw_fact_license_order`, `common.dim_month`, `common.dim_account` |
| `vws_launches_account.sql` | `launches` | Monthly, school-account usage snapshot | `content_delivery_usage.fact_students_usage_monthly_snapshots` |
| `vws_completions_account.sql` | `completions` | Monthly, school-account completion snapshot | `content_delivery_usage.fact_students_completions_monthly_snapshots` |
| `vws_month.sql` | `custom_month` | One row per month; fiscal/school-year custom time mappings | `common.dim_month` |
| `vws_calendar.sql` | `calendar` | One row per day; standard daily time spine. There is no need in this table but dbt 1.12.5 requires a day grain model. | `common.dim_calendar` |

The corresponding YAML snapshots are `account.yml`, `arr_account.yml`, `licenses_account.yml`, `launches_account.yml`, `completions_account.yml`, `custom_month.yml`, and `calendar.yml`. 


## Metrics complete catalog from the September YAML

All metrics are semi-additiveand use **`type: simple`**, **`agg: sum`**, and **`non_additive_dimension: {name: mon_lastday, window_agg: max}`**. This selects the last reporting-month snapshot for a multi-month result instead of summing month-end snapshots across time. 

| Metric name (use with `mf`) | Semantic model | Source expression | Interpretation |
| --- | --- | --- | --- |
| `arr` | `arr` | `preliminary_arr` | Default Preliminary ARR balance |
| `preliminary_arr` | `arr` | `preliminary_arr` | Preliminary ARR balance |
| `backdated_arr` | `arr` | `backdated_arr` | Backdated ARR balance |
| `true_arr` | `arr` | `true_arr` | True ARR balance |
| `licenses` | `licenses` | `studentcount` | License student count from monthly district snapshot |
| `active_users` | `launches` | `active_users` | Precalculated active-user count in school snapshot |
| `launches` | `launches` | `launches` | Precalculated launch count in school snapshot |
| `active_users_month` | `launches` | `active_users_month` | Month-specific active-user count column |
| `launches_month` | `launches` | `launches_month` | Month-specific launch count column |
| `users_with_completions` | `completions` | `users_with_completions` | Precalculated count of students with completions |
| `completions` | `completions` | `completions` | Precalculated completion count |
| `users_with_completions_month` | `completions` | `users_with_completions_month` | Month-specific users-with-completions column |
| `completions_month` | `completions` | `completions_month` | Month-specific completions column |



## Semantic entities and dimensions

### Join entities

| Semantic model | Entity | Type | Physical expression |
| --- | --- | --- | --- |
| `account` | `account` | Primary | `account_id` |
| `arr` | `account` | Foreign | `account_id` |
| `licenses` | `account` | Foreign | `account_id` |
| `launches` | `account` | Foreign | `account_id` |
| `completions` | `account` | Foreign | `account_id` |
| `custom_month` | `calendar` | Primary | `mon_year`  |
| `calendar` | `calendar` | Primary | `cal_date` |

The metric models define their aggregation time as `mon_lastday`.

### Categorical dimensions exposed in `account.yml`

- `account__state_code` state or province code.
- `account__country` country.
- `account__customer_name` commercial customer display grouping.
- `account__district_name` district name.
- `account__name` account name.

These are the five categorical dimensions explicitly marked in that YAML. Other physical attributes present in `vws_account.sql` are **not automatically MetricFlow dimensions**.

### Time dimensions

- `metric_time` MetricFlow's global aggregation time dimension.
- `arr__mon_lastday`, `licenses__mon_lastday`, `launches__mon_lastday`, `completions__mon_lastday` model time dimensions at **month** granularity.
- `custom_month` exposes `mon_firstday` and `mon_lastday` at month granularity; `calendar` exposes `cal_date` at **day** granularity.

For a given metric, confirm which entity-qualified dimensions MetricFlow actually permits with `mf list dimensions --metrics <metric>`. They depend on the semantic graph and installed engine version.

## Time spines and custom calendars



**`vws_month` / `custom_month.yml`** uses:

```yaml
time_spine:
  standard_granularity_column: mon_firstday
  custom_granularities:
    - name: fiscalyear
      column_name: fiscalyear
    - name: fiscalmonth
      column_name: fiscalyear_mon
    - name: schoolyear
      column_name: schoolyear
    - name: schoolmonth
      column_name: schoolyear_mon
    - name: mon_year
      column_name: mon_year
```

This provides the following time-grain query names:

| MetricFlow group-by | Warehouse mapping |
| --- | --- |
| `metric_time__fiscalyear` | `fiscalyear` |
| `metric_time__fiscalmonth` | `fiscalyear_mon` |
| `metric_time__schoolyear` | `schoolyear` |
| `metric_time__schoolmonth` | `schoolyear_mon` |
| `metric_time__mon_year` | `mon_year` (`YYYYMM`) |

The fiscal year runs **July through June**; July is fiscal month **1**. In the time-spine YAML, the custom-granularity fields are **plain columns**, not ordinary `dimension: {type: categorical}` definitions. That matters for time-aware queries.

**Why `mon_firstday`?** On Redshift, MetricFlow normalizes a month date via `DATE_TRUNC('month', ...)`, producing the **first day** of the month. The underlying ARR snapshots still use `mon_lastday` as their as-of date, but joining to a month-end-keyed time spine could mismatch September 30 with September 1. Thus the month time spine uses `mon_firstday`, while metric models retain `agg_time_dimension: mon_lastday`.

**`vws_calendar` / `calendar.yml`** is the **daily** time spine with `standard_granularity_column: cal_date` and `cal_date` declared as a day-grain time dimension. dbt Core 1.12.5 requires a day time spine even for monthly preaggregated metrics; the daily model was added to satisfy that validation requirement  but is not used directly anywhere

**Known CLI discoverability limitation (observed on 1.12.3/1.12.5):** `mf list dimensions --metrics arr` returned `metric_time`, `arr__mon_lastday`, and account dimensions but did **not** enumerate the custom `metric_time__fiscalyear` / `metric_time__fiscalmonth` variants, even though queries against those names succeeded. Don't assume absence from `mf list dimensions` always means a custom time grain cannot be used.



### Parse/build, validate, list, and query

Run in the directory containing `dbt_project.yml`:

```bash
# Connection and dbt models
dbt debug
dbt parse
dbt ls --resource-type model
# Builds only tagged semantic models, provided the source relations exist:
dbt run --select tag:semantic

# Semantic validation and discovery
mf validate-configs
mf list metrics
mf list dimensions --metrics arr
mf list entities --metrics arr

# Basic time and geography queries
mf query --metrics arr --group-by metric_time__month
mf query --metrics arr --group-by account__state_code
mf query --metrics arr --group-by account__country

# Custom fiscal calendar queries (tested as queryable in this project)
mf query --metrics arr --group-by metric_time__fiscalyear
mf query --metrics arr --group-by metric_time__fiscalmonth

# Multiple metrics from one semantic model
mf query --metrics arr,preliminary_arr,true_arr,backdated_arr --group-by metric_time__month

# Inspect SQL; confirm the local CLI supports the flag
mf query --metrics arr --group-by metric_time__month --explain
```


See the [main README](../../README.md) for architecture and the [demo README](../../demo/README.md) for end-to-end examples.
