# Monitoring-of-Spark-Metrics-in-K8s
This is a guide to monitoring spark application metrics in a kubernetes cluster, using kube-prometheus-stack as monitoring architecture.

Note: This documentation assummes that you already have a a running cluster cluster, with spark and kube-prometheus-stack(prometheus, grafana and alertmanager deployed)

Step 1: Update the Spark application (and optionally the Spark Operator) to expose Prometheus metrics via the /metrics/prometheus endpoint.
Step 2: Add a ServiceMonitor to kube-prometheus-stack's Helm values (custom-values.yaml) to scrape metrics from the Spark application.
Step 3: Visualize these metrics in Grafana by importing pre-configured Spark monitoring dashboards.
Step 4: Validate that Prometheus is correctly scraping metrics and that Grafana dashboards display them.

# Step 1: Expose Spark Metrics
To monitor Spark metrics, you need to configure the Spark application and Spark operator to expose metrics from both the driver and executor.

1.1 Update Spark Application to Expose Metrics
You’ll modify the Spark application Helm chart (or Spark operator’s values, depending on your setup) to enable Prometheus metrics. Follow these steps:

1. Create a custom values file for the Spark application (e.g., spark-metrics-values.yaml) and include the following to expose metrics for both driver and executor:

  
  ````
spark:
  metrics:
    prometheus:
      jmxExporterJar: /path/to/jmx_prometheus_javaagent-0.16.1.jar  # Path to the JMX exporter
      port: 8090  # Port for the driver and executor metrics
  sparkConf:
    "spark.metrics.conf.driver.source.jvm.class": "org.apache.spark.metrics.source.JvmSource"
    "spark.metrics.conf.executor.source.jvm.class": "org.apache.spark.metrics.source.JvmSource"
    "spark.metrics.conf.*.sink.prometheusServlet.class": "org.apache.spark.metrics.sink.PrometheusServlet"
    "spark.metrics.conf.*.sink.prometheusServlet.path": /metrics/prometheus

 ````

This configuration:
Enables the JVM metrics for both the driver and executors.
Exposes metrics through the /metrics/prometheus endpoint using Prometheus' format.

2. Deploy the Spark application using ArgoCD:
In your ArgoCD Application manifest for Spark, add this spark-metrics-values.yaml under the helm.valueFiles section:

3. Sync the ArgoCD application, and your Spark application will expose metrics on the specified ports.

1.2 Update Spark Operator to Expose Metrics (if needed)
If you’re also deploying the Spark Operator and want to monitor it, you may need to configure it separately. Add the following values to expose metrics from the Spark Operator. This ensures the Spark Operator is also exposing metrics at /metrics.

 ````
metrics:
  enableMetrics: true
  port: 10254  # Port for the Spark Operator metrics

````

# Step 2: Configure Prometheus to Scrape Spark Metrics
Now that Spark is exposing metrics, you need to configure Prometheus to scrape those metrics by adding a ServiceMonitor to the kube-prometheus-stack configuration.

2.1 Update the Helm Values for kube-prometheus-stack

- Create a custom values file for kube-prometheus-stack (e.g., custom-values.yaml) to include the ServiceMonitor for your Spark application:

````

# custom-values.yaml for kube-prometheus-stack
prometheus:
  additionalServiceMonitors:
    - name: spark-service-monitor
      selector:
        matchLabels:
          app: spark-app  # This label should match your Spark app label
      namespaceSelector:
        matchNames:
          - spark-namespace  # Namespace where Spark is running
      endpoints:
        - port: driverMetrics  # This should match the driver metrics port in Spark config
          path: /metrics/prometheus
          interval: 15s
        - port: executorMetrics  # This should match the executor metrics port in Spark config
          path: /metrics/prometheus
          interval: 15s
    - name: spark-operator-service-monitor  # If you want to scrape the Spark Operator as well
      selector:
        matchLabels:
          app: spark-operator
      namespaceSelector:
        matchNames:
          - spark-namespace  # Namespace where Spark Operator is running
      endpoints:
        - port: metrics  # Port to scrape from the Spark Operator
          path: /metrics
          interval: 15s

 
````

- Update the ArgoCD Application manifest for kube-prometheus-stack to include this custom values file:
- Sync the ArgoCD application for kube-prometheus-stack. Prometheus will now scrape the metrics from the Spark driver, executor, and optionally the Spark Operator.

# Step 3: Visualize Metrics in Grafana
Once Prometheus is scraping the metrics, you can visualize them using Grafana.
3.1 Set up Grafana Dashboards
Import Spark Monitoring Dashboards:

You can import pre-configured Spark monitoring dashboards from Grafana's dashboard repository. For Spark, there are existing dashboards like:
Dashboard ID: 11749 (Apache Spark Metrics Dashboard)
Dashboard ID: 10330 (Spark Application Metrics)
In Grafana, go to the Dashboards section, click on Import, and use the Dashboard IDs mentioned above.

Select Prometheus as your data source when prompted during the dashboard import process.

Customize the dashboard filters to select the correct Spark application (or Spark Operator) using labels that were set in your ServiceMonitor.

# Step 4: Validate the Setup
Check Prometheus Targets:

In the Prometheus UI (http://<prometheus-url>:9090/targets), verify that the Spark application’s ServiceMonitor targets appear under the targets section and that the status is up.
Verify Metrics in Grafana:

Open the Grafana dashboard and ensure that metrics like driver memory usage, executor status, and other JVM metrics are correctly displayed.
