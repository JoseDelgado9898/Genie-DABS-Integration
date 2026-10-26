# nyc_taxi_trip_analytics

Databricks Asset Bundle (DABs) project for the **NYC Taxi Trip Analytics** Genie space
(source space `01f1cbd25f651306b99fbf75526250c1` on `adb-7405617603277213.13.azuredatabricks.net`).

> Only the `dev` target is defined. staging/prod targets and CI/CD are intentionally omitted while
> dabs-migrator is paused for staging/prod use — see the
> [open dabs-migrator issues](https://github.com/databricks-solutions/technical-services-solutions/issues?q=is%3Aissue+is%3Aopen+dabs-migrator).

## Layout

- `databricks.yml` — bundle entrypoint, variables, and the `dev` target.
- `resources/genie_spaces/nyc_taxi_trip_analytics.yml` — the Genie space, including its full
  `serialized_space` (tables, column configs, instructions, SQL snippets, example SQL, sample
  questions, benchmarks).

## Variables

| Variable | Default | Used for |
|---|---|---|
| `catalog` | `samples` | Catalog of the `trips` table referenced throughout `serialized_space` |
| `schema` | `nyctaxi` | Schema of the `trips` table |
| `warehouse_id` | lookup of `Serverless Starter Warehouse` | SQL warehouse that runs Genie queries |

## Validating

Only the Databricks CLI is required — no Python environment.

```bash
databricks bundle validate -p azure_classic
```

## Deploying

```bash
databricks bundle deploy -t dev -p azure_classic
```

`mode: development` deploys a separate copy titled `[dev <user>] NYC Taxi Trip Analytics`; it does
not modify the original space.

## Syncing UI edits back

`serialized_space` is inline (not `file_path`) because bundle variables are not interpolated inside
a referenced `.geniespace.json`. As a result, `databricks bundle generate genie-space --watch`
can't round-trip into this file. To pick up UI edits, re-export with
`databricks api get "/api/2.0/genie/spaces/<space_id>?include_serialized_space=true"` and replace
the body, re-applying the `${var.catalog}.${var.schema}.trips` substitution.
