# Direct DB Connection

Since version 4.3 Grafana can use MySQL as a native data source. The Grafana-Zabbix plugin can use this data source for querying data directly from a Zabbix database.

One of the most resource intensive queries for Zabbix API is the history query. For long time intervals `history.get`
returns a huge amount of data. In order to display it, the plugin should adjust time series resolution
by using [consolidateBy](/reference/functions/#consolidateby). Ultimately, Grafana displays this reduced
time series, but that data should be loaded and processed on the client side first. Direct DB Connection solves these two problems by moving consolidation to the server side. Thus, the client gets a 'ready-to-use' dataset which is much smaller. This allows the data to load faster and the client doesn't spend time processing the data.

Also, many users see better performance from direct database queries versus API calls. This could be the result of several reasons,
such as the additional PHP layer and additional SQL queries (user permissions checks).

## Data Flow

This chart illustrates how the plugin uses both Zabbix API and the MySQL data source for querying different types
of data from Zabbix. MySQL data source is used only for pulling history and trend data instead of `history.get`
and `trend.get` API calls.

[![Direct DB Connection](../img/reference-direct-db-connection.svg)](../img/reference-direct-db-connection.svg)

## Query structure

Below is an example query for getting history in the Grafana-Zabbix Plugin:

**MySQL**:
```sql
SELECT itemid AS metric, clock AS time_sec, {aggFunc}(value) as value
FROM {historyTable}
WHERE itemid IN ({itemids})
  AND clock > {timeFrom} AND clock < {timeTill}
GROUP BY time_sec DIV {intervalSec}, metric
ORDER BY time_sec ASC
```

**PostgreSQL**:
```sql
SELECT to_char(itemid, 'FM99999999999999999999') AS metric, 
  clock / {intervalSec} * {intervalSec} AS time, 
  {aggFunc}(value) AS value
FROM {historyTable}
WHERE itemid IN ({itemids})
  AND clock > {timeFrom} AND clock < {timeTill}
GROUP BY 1, 2
ORDER BY time ASC
```

where `{aggFunc}` is one of `[AVG, MIN, MAX, SUM, COUNT]` aggregation functions, `{historyTable}` is a history table,
`{intervalSec}` - consolidation interval in seconds.

When getting trends, the plugin additionally queries a particular value column (`value_avg`, `value_min` or `value_max`) which
depends on `consolidateBy` function value:

**MySQL**:
```sql
SELECT itemid AS metric, clock AS time_sec, {aggFunc}({valueColumn}) as value
FROM {trendsTable}
WHERE itemid IN ({itemids})
  AND clock > {timeFrom} AND clock < {timeTill}
GROUP BY time_sec DIV {intervalSec}, metric
ORDER BY time_sec ASC
```

**PostgreSQL**:
```sql
SELECT to_char(itemid, 'FM99999999999999999999') AS metric, 
  clock / {intervalSec} * {intervalSec} AS time, 
  {aggFunc}({valueColumn}) AS value
FROM {trendsTable}
WHERE itemid IN ({itemids})
  AND clock > {timeFrom} AND clock < {timeTill}
GROUP BY 1, 2
ORDER BY time ASC
```

**Note**: these queries may be changed in future, so look into sources for actual query structure.

As you can see, the Grafana-Zabbix plugin uses aggregation by a given time interval. This interval is provided by Grafana and depends on the panel width in pixels. Thus, Grafana displays the data in the proper resolution.

## Functions usage with Direct DB Connection

There's only one function that directly affects the backend data. This function is `consolidateBy`. Other functions work on the client side and transform data that comes from the backend. So you should clearly understand that this is pre-aggregated data (by AVG, MAX, MIN, etc). 

For example, say you want to group values by 1 hour interval and `max` function. If you just apply `groupBy(10m, max)` function, your result will be wrong, because you would transform data aggregated by default `AVG` function. You should use `consolidateBy(max)` coupled with `groupBy(10m, max)` in order to get a precise result.
