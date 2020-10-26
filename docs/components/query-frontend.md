---
title: Query Frontend
type: docs
menu: components
---

# Query Frontend

The `thanos query-frontend` command implements a service that can be put in front of Thanos Queriers to improve the read path. It is
based on the [Cortex Query Frontend](https://cortexmetrics.io/docs/architecture/#query-frontend) component so you can find some common features like `Splitting` and `Results Caching`.

Query Frontend is fully stateless and horizontally scalable.

Example command to run Query Frontend:

```bash
thanos query-frontend \
    --http-address     "http://0.0.0.0:9090" \
    --query-frontend.downstream-url="<thanos-querier>:<querier-http-port>"
```

_**NOTE:** Currently only range queries (`/api/v1/query_range` API call) are actually processed through Query Frontend. All other
API calls just directly go to the downstream Querier, which means only range queries are split and cached. But we are planning to support instant queries as well.

For more information please check out [initial design proposal](https://thanos.io/tip/proposals/202004_embedd_cortex_frontend.md/).

## Features

### Splitting

Query Frontend splits a long query into multiple short queries based on the configured `--query-range.split-interval` flag. The default value of `--query-range.split-interval`
is `24h`. When caching is enabled it should be greater than `0`.

There are some benefits from query splitting:

1. Safe guard. It prevents large queries from causing OOM issues to Queries.
2. Better parallelization.
3. Better load balancing for Queries.

### Retry

Query Frontend supports a retry mechanism to retry query when HTTP requests are failing. There is a `--query-range.max-retries-per-request` flag to limit the maximum retry times.

### Caching

Query Frontend supports caching query results and reuses them on subsequent queries. If the cached results are incomplete,
Query Frontend calculates the required subqueries and executes them in parallel on downstream queriers.
Query Frontend can optionally align queries with their step parameter to improve the cacheability of the query results.
Currently, in-memory cache (fifo cache) and memcached are supported.

#### In-memory

[embedmd]:# (../flags/config_response_cache_in_memory.txt yaml)
```yaml
type: IN-MEMORY
config:
  max_size: ""
  max_size_items: 0
  validity: 0s
```
`max_size: ` Maximum memory size of the cache in bytes. A unit suffix (KB, MB, GB) may be applied.

**_NOTE:** If both `max_size` and `max_size_items` are not set, then the *cache* would not be created.

If either of `max_size` or `max_size_items` is set, then there is not limit on other field.
For example - only set `max_size_item` to 1000, then `max_size` is unlimited. Similarly, if only `max_size` is set, then `max_size_items` is unlimited.

Example configuration: [kube-thanos](https://github.com/thanos-io/kube-thanos/blob/master/examples/all/manifests/thanos-query-frontend-deployment.yaml#L50-L54)

#### Memcached

[embedmd]:# (../flags/config_response_cache_memcached.txt yaml)
```yaml
type: MEMCACHED
config:
  addresses: []
  timeout: 0s
  max_idle_connections: 0
  max_async_concurrency: 0
  max_async_buffer_size: 0
  max_get_multi_concurrency: 0
  max_item_size: 0
  max_get_multi_batch_size: 0
  dns_provider_update_interval: 0s
  expiration: 0s
```

`expiration` specifies memcached cache valid time , If set to 0s, so using a default of 24 hours expiration time

Other cache configuration parameters, you can refer to [memcached-index-cache]( https://thanos.io/tip/components/store.md/#memcached-index-cache).


### Slow Query Log

Query Frontend supports `--query-frontend.log-queries-longer-than` flag to log queries running longer than some duration.

## Naming

Naming is hard :) Please check [here](https://github.com/thanos-io/thanos/pull/2434#discussion_r408300683) to see why we chose `query-frontend` as the name.

## Flags

[embedmd]:# (flags/query-frontend.txt $)
```$
usage: thanos query-frontend [<flags>]

query frontend

Flags:
  -h, --help                     Show context-sensitive help (also try
                                 --help-long and --help-man).
      --version                  Show application version.
      --log.level=info           Log filtering level.
      --log.format=logfmt        Log format to use. Possible options: logfmt or
                                 json.
      --tracing.config-file=<file-path>
                                 Path to YAML file with tracing configuration.
                                 See format details:
                                 https://thanos.io/tip/thanos/tracing.md/#configuration
      --tracing.config=<content>
                                 Alternative to 'tracing.config-file' flag
                                 (lower priority). Content of YAML file with
                                 tracing configuration. See format details:
                                 https://thanos.io/tip/thanos/tracing.md/#configuration
      --http-address="0.0.0.0:10902"
                                 Listen host:port for HTTP endpoints.
      --http-grace-period=2m     Time to wait after an interrupt received for
                                 HTTP Server.
      --query-range.split-interval=24h
                                 Split query range requests by an interval and
                                 execute in parallel, it should be greater than
                                 0 when query-range.response-cache-config is
                                 configured.
      --query-range.max-retries-per-request=5
                                 Maximum number of retries for a single query
                                 range request; beyond this, the downstream
                                 error is returned.
      --query-range.max-query-length=0
                                 Limit the query time range (end - start time)
                                 in the query-frontend, 0 disables it.
      --query-range.max-query-parallelism=14
                                 Maximum number of query range requests will be
                                 scheduled in parallel by the Frontend.
      --query-range.response-cache-max-freshness=1m
                                 Most recent allowed cacheable result for query
                                 range requests, to prevent caching very recent
                                 results that might still be in flux.
      --query-range.partial-response
                                 Enable partial response for query range
                                 requests if no partial_response param is
                                 specified. --no-query-range.partial-response
                                 for disabling.
      --query-range.response-cache-config-file=<file-path>
                                 Path to YAML file that contains response cache
                                 configuration.
      --query-range.response-cache-config=<content>
                                 Alternative to
                                 'query-range.response-cache-config-file' flag
                                 (lower priority). Content of YAML file that
                                 contains response cache configuration.
      --labels.split-interval=24h
                                 Split labels requests by an interval and
                                 execute in parallel, it should be greater than
                                 0 when labels.response-cache-config is
                                 configured.
      --labels.max-retries-per-request=5
                                 Maximum number of retries for a single
                                 label/series API request; beyond this, the
                                 downstream error is returned.
      --labels.max-query-parallelism=14
                                 Maximum number of labels requests will be
                                 scheduled in parallel by the Frontend.
      --labels.response-cache-max-freshness=1m
                                 Most recent allowed cacheable result for labels
                                 requests, to prevent caching very recent
                                 results that might still be in flux.
      --labels.partial-response  Enable partial response for labels requests if
                                 no partial_response param is specified.
                                 --no-labels.partial-response for disabling.
      --labels.default-time-range=24h
                                 The default metadata time range duration for
                                 retrieving labels through Labels and Series API
                                 when the range parameters are not specified.
      --labels.response-cache-config-file=<file-path>
                                 Path to YAML file that contains response cache
                                 configuration.
      --labels.response-cache-config=<content>
                                 Alternative to
                                 'labels.response-cache-config-file' flag (lower
                                 priority). Content of YAML file that contains
                                 response cache configuration.
      --cache-compression-type=""
                                 Use compression in results cache. Supported
                                 values are: 'snappy' and ” (disable
                                 compression).
      --query-frontend.downstream-url="http://localhost:9090"
                                 URL of downstream Prometheus Query compatible
                                 API.
      --query-frontend.compress-responses
                                 Compress HTTP responses.
      --query-frontend.log-queries-longer-than=0
                                 Log queries that are slower than the specified
                                 duration. Set to 0 to disable. Set to < 0 to
                                 enable on all queries.
      --log.request.decision=LogFinishCall
                                 Request Logging for logging the start and end
                                 of requests. LogFinishCall is enabled by
                                 default. LogFinishCall : Logs the finish call
                                 of the requests. LogStartAndFinishCall : Logs
                                 the start and finish call of the requests.
                                 NoLogCall : Disable request logging.

```