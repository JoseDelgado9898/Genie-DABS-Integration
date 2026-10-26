# Genie Agents as Code with DABs

A demo of defining a **Genie agent (Genie space) as code** with Databricks Asset Bundles, so the
same agent can be promoted between environments (dev → staging → prod) like any other bundle
resource instead of being rebuilt by hand in each workspace's UI.

The example agent is **NYC Taxi Trip Analytics**, exported from space
`01f1cbd25f651306b99fbf75526250c1` on `adb-7405617603277213.13.azuredatabricks.net`.

## Why `serialized_space`

A Genie space is mostly not its title and warehouse. Its value is in the curated body: the tables it
can query, column descriptions and entity matching, text instructions, SQL snippets (filters,
measures, expressions), example question→SQL pairs, sample questions, and benchmarks. The
`genie_spaces` bundle resource carries all of that in a single field, `serialized_space`.

Putting that body in the bundle means:

- **The whole agent is versioned in Git.** Instruction or example-SQL changes are reviewed in a PR
and diffed like code.
- **Promotion is a deploy.** `databricks bundle deploy -t <target>` creates or updates the space in
the target workspace with the exact same curation, with no copy-pasting between UIs.
- **Environment differences are variables, not edits.** Everything that changes between
environments is a bundle variable resolved at deploy time (see below), so one definition serves
every environment.

Without `serialized_space`, a bundle can only create an **empty** space (title + warehouse) that
passes `bundle validate` but has no tables, instructions, or sample questions.

## How environment differences are handled

The exported body hardcoded `samples.nyctaxi.trips` in 40 places: the table identifier plus every
SQL snippet, example query, and benchmark. Each of those is rewritten as
`${var.catalog}.${var.schema}.trips`, and the warehouse ID is resolved by name:


| Variable       | Default                                  | Used for                                                              |
| -------------- | ---------------------------------------- | --------------------------------------------------------------------- |
| `catalog`      | `samples`                                | Catalog of the `trips` table referenced throughout `serialized_space` |
| `schema`       | `nyctaxi`                                | Schema of the `trips` table                                           |
| `warehouse_id` | lookup of `Serverless Starter Warehouse` | SQL warehouse that runs Genie queries                                 |


To point the same agent at another environment's data, override the variables. Set them per target
in `databricks.yml`, or on the command line:

```bash
databricks bundle deploy -t dev --var catalog=my_catalog --var schema=my_schema
```



### Why inline instead of `file_path`

`genie_spaces` also accepts `file_path` pointing at an exported `.geniespace.json`. However, bundle
variables are **not** interpolated inside that file: a `${var.catalog}` there would deploy as the
literal text and break every query. Keeping `serialized_space` inline in the resource YAML is what
makes the body environment-aware. After resolution it matches the live space exactly.

## Layout

- `databricks.yml` — bundle entrypoint, variables, and the `dev` target.
- `resources/genie_spaces/nyc_taxi_trip_analytics.yml` — the Genie space and its full
`serialized_space`.



## Validating and deploying

Only the Databricks CLI is required, no Python environment.

```bash
databricks bundle validate -t dev
databricks bundle deploy -t dev
```

The CLI selects the `~/.databrickscfg` profile whose host matches the target's `workspace.host`.
Pass `-p <profile>` to be explicit. `mode: development` deploys a separate copy titled
`[dev <user>] NYC Taxi Trip Analytics` and leaves the original space untouched.

> Only the `dev` target is defined so far. This project was scaffolded with dabs-migrator, which is
> currently paused for staging/prod use, so staging/prod targets and CI/CD are intentionally omitted
> — see the
> [open dabs-migrator issues](https://github.com/databricks-solutions/technical-services-solutions/issues?q=is%3Aissue+is%3Aopen+dabs-migrator).
> Promoting further means adding those targets (each with its own `workspace.host` and variable
> values) and deploying them through CI.



## Syncing UI edits back

Because the body is inline, `databricks bundle generate genie-space --watch` can't round-trip into
this file. To pick up edits made in the UI, re-export with
`databricks api get "/api/2.0/genie/spaces/<space_id>?include_serialized_space=true"`, replace the
body, and re-apply the `${var.catalog}.${var.schema}.trips` substitution.