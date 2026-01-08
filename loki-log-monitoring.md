# Grafana LOKI -- Log Monitoring
* Loki is a horizontally scalable, highly available, multi-tenant log aggregation system inspired by Prometheus. It is designed to be very cost effective and easy to operate. It does not index the contents of the logs, but rather a set of labels for each log stream.
*  Unlike other logging systems, Loki does not index the contents of the logs, but only indexes metadata about your logs as a set of labels for each log stream.

## Loki logging stack
A typical Loki-based logging stack consists of 3 components:
* **Agent:** An agent or client, for example Grafana Alloy, or Promtail, which is distributed with Loki. The agent scrapes logs, turns the logs into streams by adding labels, and pushes the streams to Loki through an HTTP API. 
* **Loki:** The main server, responsible for ingesting and storing logs and processing queries.
* **Grafana:** for querying and displaying log data.

![alt text](image-1.png) 

## Loki Features
1. **Efficient storage** - Loki stores log data in highly compressed chunks because it indexes only the set of labels. significantly smaller than other log aggregation tools
2. **LogQL, the Loki query language** - LogQL is the query language for Loki
3. **Alerting** - Loki includes a component called the ruler, which can continually evaluate queries against your logs, and perform an action based on the result. This allows you to monitor your logs for anomalies or events. Loki integrates with Prometheus Alertmanager, or the alert manager within Grafana.
4. **Grafana integration** -  Loki integrates with Grafana, Mimir, and Tempo, providing a complete observability stack, and seamless correlation between logs, metrics and traces.
5. **Third-party integrations** - Several third-party agents (clients) have support for Loki, via plugins. This lets you keep your existing observability setup while also shipping logs to Loki. 
