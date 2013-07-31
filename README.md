Tenacity
========

What is it?
-----------

Tenacity is a Dropwizard module that incorporates Hystrix. Hystrix is a resiliency library from Netflix. Hystrix goals are to:

1. Stop cascading failures.
2. Fail-fast and rapidly recover.
3. Reduce mean-time-to-discovery (with dashboards)
4. Reduce mean-time-to-recovery (with dynamic configuration)

Tenacity aids at making Hystrix dropwizard-friendly and constructing an ecosystem for developers to quickly leverage the benefits of Hystrix.

1. Bundles setup necessary property strategies, metrics, dynamic configuration, and some resource endpoints (e.g. for dashboards).
2. Dropwizard-configuration style (YAML) for dependencies.
3. Building blocks to inject different configuration dependencies into your dependency wrappers (`TenacityCommand<ReturnType>`).
4. Ability to unit test Hystrix. We have tried to mitigate a lot of the problems because Hystrix relies heavily on static state.

Modules
-------

-   `tenacity-core`:        The building blocks to quickly use Hystrix within the context of Dropwizard.
-   `tenacity-dashboard`:       Adds the Hystrix dashboard accessiable at `/tenacity`. *Warning: this disables gzip encoding to support text/event-streams*
-   `tenacity-service`:       Utilizes the tenacity-dashboard to view different configurable clusters. Currently this is hosted at http://tenacity.int.yammer.com
-   `tenacity-*-legacy`:        Support for legacy versions of Dropwizard. Currently there is a conflict with <0.6.0 versions (around Bundles).
    Use these instead of their non-legacy counterparts.

How To Use
==========

Here is a sample `TenacityCommand` that always succeeds:

        public class AlwaysSucceed extends TenacityCommand<String> {
            public AlwaysSucceed(TenacityPropertyStore tenacityPropertyStore) {
                super("Example", "AlwaysSucceed", tenacityPropertyStore, DependencyKey.ALWAYS_SUCCEED);
            }

            @Override
            protected String run() throws Exception {
                return "value";
            }

            @Override
            protected String getFallback() {
                return "fallback";
            }
        }

There are two methods that must be overriden, `run()` and `getFallback()`. `getFallback()` is *always* invoked. It can be invoked one of four different ways:

1. `run()` throws an exception.
2. `run()` is never executed due to an open circuit.
3. `run()` is never executed due a queue-rejection. This can be because there aren't enough threads or queue space available.
4. `run()` takes too long to return a result. The thread waiting for the result will not receive the result from the invocation of `getFallback()` instead.

![Alt text](https://raw.github.com/wiki/Netflix/Hystrix/images/hystrix-flow-chart-original.png)



Configuration
=============

When any percentile data is needed this should be calculated using historical data. Look somewhere between a week to a month
and take the `max` of a particular metric.

1. Tenacity
  -   executionIsolationThreadTimeoutInMillis = `ceil(1.1 * (max(p99) + max(median)))`
  -   threadpool
      *   size = `(p99 in seconds) * (m1 rate req/sec)` [minimum of 2, anything over 20 should be discussed]

2. HTTP client
  -   connectTimeout = `33% of executionIsolationThreadTimeoutInMillis`
  -   timeout (readTimeout) = `66% of executionIsolationThreadTimeoutInMillis`
  -   -   there is a good chance that a single client may use multiple commandKeys that might warrant a range of different connectionTimeouts and read timeouts. 
  -   In this case the current approach is to take the max value across all specific timeouts (alternatively the clients could be split into multiple clients on the service level)

