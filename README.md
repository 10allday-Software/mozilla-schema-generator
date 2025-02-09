[![CircleCI](https://circleci.com/gh/mozilla/mozilla-schema-generator/tree/main.svg?style=svg)](https://circleci.com/gh/mozilla/mozilla-schema-generator/tree/main)
[![Latest Version](https://img.shields.io/pypi/v/mozilla-schema-generator.svg)](https://pypi.python.org/pypi/mozilla-schema-generator/)

# Mozilla Schema Generator

A library for generating full representations of Mozilla telemetry pings.

See [Mozilla Pipeline Schemas](https://www.github.com/mozilla-services/mozilla-pipeline-schemas)
for the more generic structure of pings. This library takes those generic structures and fills in
all of the probes we expect to see in the appropriate places.

## Telemetry Integration

There are two generic ping types we're targeting for this library:

1. [The Common Ping Format](http://gecko-docs.mozilla.org.s3.amazonaws.com/toolkit/components/telemetry/telemetry/data/main-ping.html)
   is used for many legacy pings from Firefox Desktop ping, including the "main" ping
2. [The Glean Ping Format](https://github.com/mozilla/glean_parser) is the common structure being used for
   all newly instrumented products at Mozilla, including mobile browsers.

This library takes the information for what should be in those pings from the [Probe Info Service](https://www.github.com/mozilla/probe-scraper).

## Data Store Integration

The primary use of the schemas is for integration with the
[Schema Transpiler](https://www.github.com/mozilla/jsonschema-transpiler). 
The schemas that this repository generates can be transpiled into Avro and Bigquery. They define
the schema of the Avro and BigQuery tables that the [BQ Sink](https://www.github.com/mozilla/gcp-ingestion)
writes to.

### BigQuery Limitations and Splitting

[BigQuery has a hard limit of ten thousand columns on any single table](https://cloud.google.com/bigquery/quotas). 
This library can take that limitation into account by splitting schemas into multiple tables,
although so far we have been able to avoid this complication. We retain schema
splitting support here as an option to use in the future. The option is currently disabled.

When a schema is split, each
table has some common information duplicated in every table, and then a set
of fields that are unique to that table. The join of these tables gives the full
set of fields available from the ping.

To decide on a table split, we include the `table_group` configuration in the configuration
file. For example, `payload/histograms` has `table_group: histograms`; this indicates that
there will be a table outputted with just histograms.

If a single table expands beyond the configured column limit, we move the new fields to the next table.
For example, we could have main_histograms_1 and main_histograms_2.

_Note_: Tables are only split if the `--split` parameter is provided, and this
is option is not currently used for our production configuration.

## Validation

When we validate pings against a schema in the data pipeline, we use the generic versions
rather than the versions generated by this repository's machinery. While the schemas produced
here are guaranteed to be more correct since they include explicit definitions of every metric and probe,
we find in practice there are too many edge cases where a probe is sent with the incorrect type
and we need to coerce it to the correct type when loading to BigQuery.
We also purposely represent some complex types as JSON strings in schemas, relying on the BQ loader
to coerce objects to string.
We could still consider using the generated schemas for validation in the future, but
additional work would be required to ensure it does not lead to mass rejection of pings.

## Usage

### Main Ping

Generate the Full Main Ping schema:

```
mozilla-schema-generator generate-main-ping
```

Generate the Main Ping schema divided among tables (for BigQuery):
```
mozilla-schema-generator generate-main-ping --split --out-dir main-ping
```

The `out-dir` parameter will be the namespace for the pings.

To see a full list of options, run `mozilla-schema-generator generate-main-ping --help`.


### Glean

Generate all Glean ping schemas - one for each application, for each ping
that application sends:

```
mozilla-schema-generator generate-glean-pings
```

Write schemas to a directory:
```
mozilla-schema-generator generate-glean-pings --out-dir glean-ping
```

To see a full list of options, run `mozilla-schema-generator generate-glean-pings --help`.


## Configuration Files

Configuration files are by default found in `/config`. You can also specify your own when running the generator.

Configuration files match certain parts of a ping to certain types of probes or metrics. The nesting
of the config file matches the ping it is filling in. For example, Glean stores probe types under
the `metrics` key, so the nesting looks like this:
```
{
    "metrics": {
        "string": {
            <METRIC_ID>: {...}
        }
    }
}
```

While the generic schema doesn't include information about the specific `<METRIC_ID>`s being included,
the schema-generator does. To include the correct metrics that we would find in that section of the ping,
we would organize the `config.yaml` file like this:

```
metrics:
    string:
        match:
            type: string
```

The `match` key indicates that we should fill-in this section of the ping schema with metrics,
and the `type: string` makes sure we only put string metrics in there. You can do an exact
match on any field available in the ping info from the [probe-info-service](https://probeinfo.telemetry.mozilla.org/glean/glean/metrics),
which also contains the [Desktop probes](https://probeinfo.telemetry.mozilla.org/firefox/all/main/all_probes).

There are a few additional keywords allowable under any field:
* `contains` - e.g. `process: contains: main`, indicates that the `process` field is an array
  and it should only match those that include the entry `main`.
* `not` - e.g. `send_in_pings: not: glean_ping_info`, indicates that we should match
  any field for `send_in_pings` _except_ `glean_ping_info`.

### `table_group` Key

This specific field is for indicating which table group that section of the ping should be included in when
splitting the schema. Currently we do not split any pings. See the section on [BigQuery
Limitations and Splitting](#bigquery-limitations-and-splitting) for more info.

## Allowing schema incompatible changes

On every run of the schema generator, there is a check for incompatible changes
between the previous revision and current generated revision. A schema
incompatible change includes a removal of a schema or a column, or a change in
the type definition of a column.

There are two methods to get around these restrictions. If you are actively
developing the schema generator and need to introduce a schema incompatible
change, set `MPS_VALIDATE_BQ=false`.

If a schema incompatible change needs to be introduced in production (i.e.
`generated-schemas`), then modify the `incompatibility-allowlist` at the root of
the repository. Add documents in the form of
`{namespace}.{doctype}.{docversion}`. Globs are allowed. For example, add the
following line to allow remove schemas under the `my_glean_app` namespace:

```bash
my_glean_app.*
```

Once the commit has gone through successfully, this line should be removed from
the document.

## Development and Testing

Install requirements:

```bash
make install-requirements
```

Ensure that the mozilla-pipeline-schemas submodule has been checked out:

```bash
git submodule init
git submodule update --remote
```

Run tests:

```bash
make test
```

Publish generated schemas to [mozilla-generated-schemas/test-generated-schemas](https://github.com/mozilla-services/mozilla-pipeline-schemas/tree/test-generated-schemas)
run:

```bash
git fetch origin

git checkout <branch-to-test>

export MPS_SSH_KEY_BASE64=$(cat ~/.ssh/id_rsa | base64)

# generate all schemas for current main
git checkout main && git pull make build && make run

# generate all schemas with changes and compare with main
git checkout <branch-to-test> make build && make run
```
