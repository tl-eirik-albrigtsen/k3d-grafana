# Grafana Cloud

<style type="text/css">
  .reveal h3, .reveal p, .reveal h4 {
    text-transform: none;
    text-align: left;
  }
  .reveal ul {
    display: block;
  }
  .reveal ol {
    display: block;
  }
  .reveal {
    background: #353535 !important;
  }
</style>


Cloud managed grafana
- managed HA prometheus/alertmanager
- agent as metric scraper (prometheus)
- agent as log ingestor (loki)
- agent as tracing ingestor (tempo)
- https://github.com/tl-eirik-albrigtsen/k3d-grafana

NOTES:
- with a focus on debuggability + discoverability

---
## Prometheus
Prometheus without prometheus.

- grafana cloud agent
- HA prometheus on their end
- embeds part of prometheus libraries

NOTES:
- HA prometheus, how? Breaking down various parts. Split prometheuses, sidecars to them so they can remote write to a bucket, add smart queriers that can defer to correct sub-prometheus OR S3 bucket.
- Hard thing to run yourself, but it can be done, but this is effectively one of the key things you pay for.

- HOW: The agent scrapes your services and the config implements part of prometheus.

---
## Prometheus Configuration

`ConfigMap` named `grafana-agent`

```json
# agent.yml

prometheus:
    configs:
      - host_filter: true
        name: agent
        remote_write:
          - basic_auth:
                password: base64token
                username: 53715
            url: ..grafana.net/api/prom/push
```

NOTES:
- Doing remote_write (acts like a prometheus), but pushes its metrics up stream.
- one scraper per node as the agent is a DS.
- CFG: scrape configs and relabelling happens in this file
- might have to tweak normal service monitors
- decide on conventions / accounts for multi environments?

---
## Loki
Log ingestor

- `grafana-cloud-logs`
- [derived-fields](https://grafana.com/docs/grafana/latest/datasources/loki/#derived-fields)

NOTES:
- AFAIKT it's a renamed `promtail` which uses parts of `loki` libraries
- simple log ingestor that trawls docker logs and pushes to grafana cloud.
- No indexing, no schemas, but we can keep pushing json to it.
- For debugging. Livetail, grep style.
- Derive fields that can hotlink to other things (even within grafana, trace discoverability)
- derive fields is just a capture regex
- because of that, we can also plug elastic into it instead, only req is that we log traceIDs consistently (loose LogQL)

---
## Tempo
A trace collector that pushes to grafana cloud.

- `grafana-agent-traces`
- replaces jaeger
- trace discoverability from logs and metrics

NOTES:
- can speak pretty much all the formats zipkin,jaeger,otel,thrift and has grpc support
- i tested it with otel and grpc against rust and go

---
## Demo

NOTES:
- show dashboards with logs
- explore {app="otel-example"} | logfmt | latency < 500ms
- explore json for {app="}
- show metrics: demo_ find bucket: histogram

---
## Alertmanager
Managed.

- LogQL in alert definitions possible
- No `PrometheusRule` - [agent/discussions#456](https://github.com/grafana/agent/discussions/456)


NOTES:
- alert on logs with field error, rate of that over 5m
- Authorize it metric access from your grafana cloud account
- Configure alerts by API.
- NO PrometheusRule crds YET.
- Their format is equivalent, but is managed via cortex-tool, conversion.
- Asked about their plans: MAYBE CONTROLLER

---
## Compute
2 node `k3d` cluster under `wrk`

```
grafana-agent-deployment-cffcf64bc-fqvc5   36m          73Mi
grafana-agent-logs-h22wf                   11m          32Mi
grafana-agent-logs-zjqwr                   10m          31Mi
grafana-agent-m4dts                        32m          66Mi
grafana-agent-traces-vmnh8                 8m           27Mi
grafana-agent-traces-z6pzc                 2m           27Mi
grafana-agent-wm56d                        44m          157Mi
```

NOTES:
- under load, 22k reqs in 30s against service gen big traces
- apparently each trace agent can deal with 5k spans/s
- computation at lookup itme (loki, prometheus, tempo)
- pricing by usage

---
## Quotes
Rough quote
- 400k time series
- 400-500GB logs daily
- 16B events/mo

-> 150k anually

NOTES:
- currently at 300M events/mo
- 30d retention loki/tempo
- price model on commitment: he gave a rough calculation that worked out to be ~ sign 3 years pay 2 years
- metric biggest contributor to that cost
- we should probably reduce labelling in our biggest metrics

---
## Rust

PoC: https://github.com/clux/controller-rs/pull/10
https://twitter.com/sszynrae/status/1369405372222603264

NOTES:
- small twitter thread about it, generally positive
- got retweeted by 3 community mgrs working for grafana
- incl @Grafana :D

---
### Rust Tracing story
Need the 5 standard libraries when using otel:

- `opentelemetry` - primitives
- `opentelemetry-otlp` - transport
- `tracing-opentelemetry` - layer impl
- `tracing-subscriber` - layers + fmt
- `tracing` - user: spans/events


NOTES:
- libraries generally need to be instrumented: https://github.com/clux/kube-rs/pull/455

Couple of really nice things with rust ecosystem; tracing hotswaps out log implementations.
- warn/error/info become events on spans and are visible (image of logs in span)
- can easily attach traceID in a format that we have configured loki for (gif of explore split)

covers traces <-> logs

PITFALLS:
- otel grpc writer can infinte recurse trace itself: https://github.com/open-telemetry/opentelemetry-rust/issues/473
- otel service naming missing: https://github.com/open-telemetry/opentelemetry-rust/issues/475

---
### Rust Tracing setup

```rust
    let (t, _abort) = opentelemetry_otlp::new_pipeline()
        .with_endpoint(&env::var("OPENTELEMETRY_URL")?)
        .install()?;

    let otel = tracing_opentelemetry::layer().with_tracer(t);
    let logger = tracing_subscriber::fmt::layer().json();
    let filter_layer = EnvFilter::try_from_default_env()
        .or_else(|_| EnvFilter::try_new("info"))?;

    let collector = Registry::default()
        .with(otel)
        .with(logger)
        .with(filter_layer);
    tracing::subscriber::set_global_default(collector)?;
```

NOTES:
- 2 layers + filter layer (control sampling with evar)


---
### Tracing Instrumentation

```rust
use tracing::info;

#[instrument(skip(ctx), fields(traceID))]
async fn reconcile(f: Foo) -> Result<()> {
    Span::current().record("traceID", &display(&trace_id()));
    let r = do_work(f).await?;
    info!("worked! {}", r);
    Ok(())
}
```

NOTES:
- generates async aware spans, embeds traceID into logs
- more ways of doing this
- swapping logs::info to tracing::info creates events in spans
- events logged with fmt layer
- traces sent with events with otel layer

---
### Rust Metric story
As usual, but awaiting exemplar support.

- [tikv/rust-prometheus#393](https://github.com/tikv/rust-prometheus/issues/393)
- [tikv/rust-prometheus#395](https://github.com/tikv/rust-prometheus/pull/395)
- [metrics-rs/metrics#175](https://github.com/metrics-rs/metrics/issues/175)

NOTES:
- funnily enough i think this is actually technically the easiest thing todo, the promethus library is just heavily overengineered, and feedback is slow, and I didn't want to waste all my week doing it.
- metrics -> traces require exemplars

---
## Notes
- https://github.com/TrueLayer/rust-observability/
- https://github.com/joe-elliott/tempo-otel-example

NOTES:
- can hook in a lot of this in rust-observability repo
- golang example by tempo creator, most official exemplar example does not even work
- https://github.com/joe-elliott/tempo-otel-example/issues/2
