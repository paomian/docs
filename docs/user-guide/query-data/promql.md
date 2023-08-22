# Prometheus Query Language


GreptimeDB can be used as a drop-in replacement for Prometheus in Grafana, because GreptimeDB supports PromQL (Prometheus Query Language). GreptimeDB has reimplemented PromQL natively in Rust and exposes the ability to several interfaces, including the HTTP API of Prometheus, the HTTP API of GreptimeDB, and the SQL interface.

## Prometheus' HTTP API

<!-- Maybe add a section to introduce the simulated interfaces, when there is more than one supported -->

Prometheus server has a bunch of HTTP APIs (see their [official document](https://prometheus.io/docs/prometheus/latest/querying/api)), and GreptimeDB has implemented the [`range_query`](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries) interface, which allows you to query the data in a given time range with PromQL.

We keep the setting of the path and parameter the same as that in Prometheus, so you can use the same client to query GreptimeDB.

## GreptimeDB's HTTP API

GreptimeDB also exposes an HTTP API for querying with PromQL. You can find it on `/promql` path under the current stable API version `/v1`, in **GreptimeDB HTTP API Port**. For example:

```shell
curl -X GET \
  -H 'Authorization: Basic {{authorization if exists}}' \
  -G \
  --data-urlencode 'db=public' \
  --data-urlencode 'query=avg(system_metrics{idc="idc_a"})' \
  --data-urlencode 'start=1667446797' \
  --data-urlencode 'end=1667446799' \
  --data-urlencode 'step=1s' \
  http://localhost:4000/v1/promql
```

The input parameters are similar to the [`range_query`](https://prometheus.io/docs/prometheus/latest/querying/api/#range-queries) in Prometheus' HTTP API:

- `db=<database name>`: Required when using GreptimeDB with authorization, otherwise can be omitted if you are using the default `public` database.
- `query=<string>`: Required. Prometheus expression query string.
- `start=<rfc3339 | unix_timestamp>`: Required. The start timestamp, which is inclusive. It is used to set the range of time in `TIME INDEX` column.
- `end=<rfc3339 | unix_timestamp>`: Required. The end timestamp, which is inclusive. It is used to set the range of time in `TIME INDEX` column.
- `step=<duration | float>`: Required. Query resolution step width in duration format or float number of seconds.

Here are some examples for each type of parameter:
- rfc3339
  - `2015-07-01T20:11:00Z` (default to seconds resolution)
  - `2015-07-01T20:11:00.781Z` (with milliseconds resolution)
  - `2015-07-02T04:11:00+08:00` (with timezone offset)
- unix timestamp
  - `1435781460` (default to seconds resolution)
  - `1435781460.781` (with milliseconds resolution)
- duration
  - `1h` (1 hour)
  - `5d1m` (5 days and 1 minute)
  - `2` (2 seconds)
  - `2s` (also 2 seconds)

The result format is the same as `/sql` interface described in [query data](./query-data/sql.md#http-api).

```json
{
  "code": 0,
  "output": [
    {
      "records": {
        "schema": {
          "column_schemas": [
            {
              "name": "ts",
              "data_type": "TimestampMillisecond"
            },
            {
              "name": "AVG(system_metrics.cpu_util)",
              "data_type": "Float64"
            },
            {
              "name": "AVG(system_metrics.memory_util)",
              "data_type": "Float64"
            },
            {
              "name": "AVG(system_metrics.disk_util)",
              "data_type": "Float64"
            }
          ]
        },
        "rows": [
          [
            1667446798000,
            80.1,
            70.3,
            90
          ],
          [
            1667446799000,
            80.1,
            70.3,
            90
          ]
        ]
      }
    }
  ],
  "execution_time_ms": 5
}
```
<!-- TODO New paths that compatible with Promethues APIs -->

## SQL

GreptimeDB also extends SQL grammar to support PromQL. You can start with the `TQL` (Time-series Query Language) keyword to write parameters and queries. The grammar looks like this:

```sql
TQL [EVAL|EVALUATE] (<START>, <END>, <STEP>) <QUERY>
```

`<START>` specifies the query start range and `<END>` specifies the end time. `<STEP>` identifies the query resolution step width. All of them can either be an unquoted number (represent UNIX timestamp for `<START>` and `<END>`, and duration in seconds for `<STEP>`), or a quoted string (represent an RFC3339 timestamp for `<START>` and `<END>`, and duration in string format for `<STEP>`).

For example:

```sql
TQL EVAL (1676738180, 1676738780, '10s') sum(some_metric)
```

You can write the above command in all places that support SQL, including the GreptimeDB HTTP API, SDK, PostgreSQL and MySQL client etc.

## Limitations

Though GreptimeDB supports a rich set of data types, the PromQL implementation is still limited to the following types:
- timestamp: `Timestamp`
- tag: `String`
- value: `Double`

Currently only a subset of PromQL is supported. Here attaches the compatibility list. You can also check our latest compliance report in this [tracking issue](https://github.com/GreptimeTeam/greptimedb/issues/1042).

### Literal

Both string and float literals are supported, with the same [rule](https://prometheus.io/docs/prometheus/latest/querying/basics/#literals) as PromQL.

### Selector

Both instant and range selector are supported. The only exception is the label matching on metric name, e.g.: `{__name__!="request_count}"` (but the equal-matching case is supported: `{__name__="request_count}"`).

Time duration and offset are supported, but `@` modifier is not supported yet.

### Binary 

*Pure literal binary-expr like `1+1` is not supported yet.*

- Supported:
    | Operator | Example  |
    | :------- | :------- |
    | add      | `a + b`  |
    | sub      | `a - b`  |
    | mul      | `a * b`  |
    | div      | `a / b`  |
    | mod      | `a % b`  |
    | eqlc     | `a == b` |
    | neq      | `a != b` |
    | gtr      | `a > b`  |
    | lss      | `a < b`  |
    | gte      | `a >= b` |
    | lte      | `a <= b` |

- Unsupported:
    | Operator | Progress |
    | :------- | :------- |
    | power    | TBD      |
    | atan2    | TBD      |
    | and      | TBD      |
    | or       | TBD      |
    | unless   | TBD      |

### Aggregators

- Supported:
    | Aggregator | Example                   |
    | :--------- | :------------------------ |
    | sum        | `sum by (foo)(metric)`    |
    | avg        | `avg by (foo)(metric)`    |
    | min        | `min by (foo)(metric)`    |
    | max        | `max by (foo)(metric)`    |
    | stddev     | `stddev by (foo)(metric)` |
    | stdvar     | `stdvar by (foo)(metric)` |


- Unsupported:
    | Aggregator   | Progress |
    | :----------- | :------- |
    | count        | TBD      |
    | grouping     | TBD      |
    | topk         | TBD      |
    | bottomk      | TBD      |
    | count_values | TBD      |
    | quantile     | TBD      |

### Instant Functions

- Supported:
    | Function | Example         |
    | :------- | :-------------- |
    | abs      | `abs(metric)`   |
    | ceil     | `ceil(metric)`  |
    | exp      | `exp(metric)`   |
    | ln       | `ln(metric)`    |
    | log2     | `log2(metric)`  |
    | log10    | `log10(metric)` |
    | sqrt     | `sqrt(metric)`  |
    | acos     | `acos(metric)`  |
    | asin     | `asin(metric)`  |
    | atan     | `atan(metric)`  |
    | sin      | `sin(metric)`   |
    | cos      | `cos(metric)`   |
    | tan      | `tan(metric)`   |

- Unsupported:
    | Function                   | Progress |
    | :------------------------- | :------- |
    | absent                     | TBD      |
    | scalar                     | TBD      |
    | sgn                        | TBD      |
    | sort                       | TBD      |
    | sort_desc                  | TBD      |
    | timestamp                  | TBD      |
    | acosh                      | TBD      |
    | asinh                      | TBD      |
    | atanh                      | TBD      |
    | sinh                       | TBD      |
    | cosh                       | TBD      |
    | tanh                       | TBD      |
    | deg                        | TBD      |
    | rad                        | TBD      |
    | *other multiple input fns* | TBD      |

### Range Functions

- Supported:
    | Function           | Example                       |
    | :----------------- | :---------------------------- |
    | idelta             | `idelta(metric[5m])`          |
    | \<aggr\>_over_time | `count_over_time(metric[5m])` |

- Unsupported:
    | Function         | Example |
    | :--------------- | :------ |
    | stddev_over_time | TBD     |
    | stdvar_over_time | TBD     |
    | changes          | TBD     |
    | delta            | TBD     |
    | rate             | TBD     |
    | deriv            | TBD     |
    | increase         | TBD     |
    | idelta           | TBD     |
    | irate            | TBD     |
    | reset            | TBD     |
