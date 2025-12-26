# kube-prometheus-stack

* The kube-prometheus-stack is the standard, pre-configured collection of tools for end-to-end Kubernetes cluster monitoring. It combines Prometheus, Grafana, Kube-State-Metrics, and other essential components into a single deployable Helm chart, simplifying the setup process

## Key Components of the Stack: 
The kube-prometheus-stack includes several integrated tools that work together to provide comprehensive observability of your Kubernetes environment: 

* **Prometheus:** The core monitoring and alerting tool. It collects metrics from various endpoints at specified intervals and stores them in a time-series database. It uses PromQL for querying and analysis.

* **Grafana:** A visualization layer that uses Prometheus as a data source to display metrics through powerful, customizable dashboards. The stack provides many ready-to-use, pre-built dashboards for critical Kubernetes metrics. 

* **Kube-State-Metrics (KSM)**: An exporter that listens to the Kubernetes API server and generates metrics about the state of the Kubernetes objects themselves (e.g., number of running pods, deployment status, PVC (Persistent Volume Claims) status). This is distinct from resource usage metrics. 

* **Node Exporter:** An exporter that collects hardware and OS metrics (CPU usage, memory utilization, disk I/O) from the underlying nodes in the cluster.  

* **Alertmanager:** Handles alerts fired by Prometheus rules, grouping and routing notifications to various platforms like Slack, PagerDuty, or email.  

**How They Work Together**
* Exporters (Node Exporter, KSM, cAdvisor) expose metrics endpoints from the cluster nodes, Kubernetes API, and containers.
* Prometheus automatically discovers and scrapes these metrics endpoints, thanks to the configuration managed by the Prometheus Operator and its CRDs.
* Grafana connects to Prometheus to query the stored time-series data and render the pre-configured dashboards, offering a visual overview of cluster health.
* Alertmanager receives alerts from Prometheus when predefined conditions are met and sends out notifications to your teams.  

reference link: https://faun.pub/kube-prometheus-with-terraform-and-helm-24cac706fbdb
