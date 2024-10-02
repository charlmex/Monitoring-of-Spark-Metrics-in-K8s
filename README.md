# Monitoring-of-Spark-Metrics-in-K8s
This is a guide to monitoring spark application metrics in a kubernetes cluster, using kube-prometheus-stack as monitoring architecture. This approach is best if you have a custom spark deployment or a legacy spark application where built-in metrics export is not configured or available, you have special monitoring needs or you are using a non prometheus monitoring solutions. For a spark deployment with built in prometheus metrics exporter that exposes metrics without requiring any special exporter, like Stackable Helm chart pls refer to the other documentation in this repository

**Note:** This documentation assummes that you already have a a running cluster cluster, with spark and kube-prometheus-stack(prometheus, grafana and alertmanager deployed) and  ArgoCD installed and configured for managing your Kubernetes applications.

- step 1: Deploy JMX Exporter as a Separate Application
- Step 2: Update the Spark application (and optionally the Spark Operator) to expose Prometheus metrics via the /metrics/prometheus endpoint.
- Step 3: Add a ServiceMonitor to kube-prometheus-stack's Helm values (custom-values.yaml) to scrape metrics from the Spark application.
- Step 4: Visualize these metrics in Grafana by importing pre-configured Spark monitoring dashboards.
- Step 5: Validate that Prometheus is correctly scraping metrics and that Grafana dashboards display them.

# Step 1: Deploy JMX Exporter as a Separate Application
1.1 Create a GitLab Repository for JMX Exporter
Create a new repository in GitLab, named jmx-exporter.
Inside this repository, create the following files:
jmx-exporter-deployment.yaml:

  ````
apiVersion: apps/v1
kind: Deployment
metadata:
  name: jmx-exporter
  namespace: spark-namespace  # Replace with your Spark namespace
  labels:
    app: jmx-exporter
spec:
  replicas: 1
  selector:
    matchLabels:
      app: jmx-exporter
  template:
    metadata:
      labels:
        app: jmx-exporter
    spec:
      containers:
        - name: jmx-exporter
          image: prom/jmx-exporter:0.16.1  # Use the JMX Exporter image
          ports:
            - containerPort: 5556  # Expose the JMX Exporter port
          args:
            - "--config.file=/etc/jmx-exporter/config.yml"
          volumeMounts:
            - name: config-volume
              mountPath: /etc/jmx-exporter/
      volumes:
        - name: config-volume
          configMap:
            name: jmx-exporter-config  # This ConfigMap will hold the JMX Exporter config
---
apiVersion: v1
kind: Service
metadata:
  name: jmx-exporter
  namespace: spark-namespace  # Replace with your Spark namespace
spec:
  ports:
    - port: 5556
      targetPort: 5556
  selector:
    app: jmx-exporter

  ````
create a jmx-exporter-configmap.yaml:

  ````
apiVersion: v1
kind: ConfigMap
metadata:
  name: jmx-exporter-config
  namespace: spark-namespace  # Replace with your Spark namespace
data:
  config.yml: |
    rules:
      - pattern: ".*"  # Match all JMX MBeans, you can adjust this pattern to filter your desired metrics

  ````
1.2 Create an ArgoCD Application for JMX Exporter
Create an ArgoCD application manifest (e.g., jmx-exporter-app.yaml) to deploy the JMX Exporter:

  ````
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: jmx-exporter
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://gitlab.com/your-username/jmx-exporter'  # Update with your GitLab repo URL
    targetRevision: HEAD
    path: .  # Assuming your YAML files are in the root of the repo
  destination:
    server: https://kubernetes.default.svc
    namespace: spark-namespace  # Replace with your Spark namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

  ````

# Step 2: Expose Spark Metrics
To monitor Spark metrics, you need to configure the Spark application and Spark operator to expose metrics from both the driver and executor.

2.1. Update Spark Application to Expose Metrics
You’ll modify the Spark application Helm chart (or Spark operator’s values, depending on your setup) to enable Prometheus metrics. Follow these steps:

1. Create a custom values file (spark-metrics-values.yaml) for your Spark application to expose metrics:

  
  ````
# spark-metrics-values.yaml

spark:
  metrics:
    prometheus:
      jmxExporterJar: /opt/jmx_prometheus_javaagent-0.16.1.jar  # Path to the JMX exporter
      port: 8090  # Port for the driver and executor metrics
  sparkConf:
    "spark.metrics.conf.driver.source.jvm.class": "org.apache.spark.metrics.source.JvmSource"
    "spark.metrics.conf.executor.source.jvm.class": "org.apache.spark.metrics.source.JvmSource"
    "spark.metrics.conf.*.sink.prometheusServlet.class": "org.apache.spark.metrics.sink.PrometheusServlet"
    "spark.metrics.conf.*.sink.prometheusServlet.path": /metrics/prometheus

  driver:
    metrics:
      port: 8090  # Port where driver metrics will be exposed
  executor:
    metrics:
      port: 8090  # Port where executor metrics will be exposed

operator:
  metrics:
    enableMetrics: true
    port: 10254  # Port for Spark Operator metrics

metadata:
  labels:
    app: spark-app  # The label that your ServiceMonitor will select

 ````

This configuration:
Enables the JVM metrics for both the driver and executors.
Exposes metrics through the /metrics/prometheus endpoint using Prometheus' format.

2. Deploy the Spark application using ArgoCD:
In your ArgoCD application manifest for the Spark application, include the custom values file:

  ````
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: spark-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://your-git-repo/spark-app'  # URL to your Spark app Git repo
    targetRevision: HEAD
    path: .  # Assuming your Helm chart is in the root of this repo
    helm:
      valueFiles:
        - spark-metrics-values.yaml  # Include your custom values file here
  destination:
    server: https://kubernetes.default.svc
    namespace: spark-namespace  # Namespace where the Spark app is deployed
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

  ````


4. Sync the ArgoCD application, and your Spark application will expose metrics on the specified ports.



# Step 3: Configure Prometheus to Scrape Spark Metrics
Now that Spark is exposing metrics, you need to configure Prometheus to scrape those metrics by adding a ServiceMonitor to the kube-prometheus-stack configuration.

3.1 Update the Helm Values for kube-prometheus-stack

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
        - port: 8090  # This should match the driver and executor metrics port in Spark config
          path: /metrics/prometheus
          interval: 15s
    - name: spark-operator-service-monitor
      selector:
        matchLabels:
          app: spark-operator  # Ensure the Spark Operator has this label
      namespaceSelector:
        matchNames:
          - spark-namespace  # Namespace where Spark Operator is running
      endpoints:
        - port: 10254  # Port to scrape from the Spark Operator
          path: /metrics
          interval: 15s

 
````

- Update the ArgoCD Application manifest for kube-prometheus-stack to include this custom values file:
- Sync the ArgoCD application for kube-prometheus-stack. Prometheus will now scrape the metrics from the Spark driver, executor, and optionally the Spark Operator.

# Step 4: Visualize Metrics in Grafana
Once Prometheus is scraping the metrics, you can visualize them using Grafana.
4.1 Set up Grafana Dashboards
Import Spark Monitoring Dashboards:

You can import pre-configured Spark monitoring dashboards from Grafana's dashboard repository. For Spark, there are existing dashboards like:
Dashboard ID: 11749 (Apache Spark Metrics Dashboard)
Dashboard ID: 10330 (Spark Application Metrics)
In Grafana, go to the Dashboards section, click on Import, and use the Dashboard IDs mentioned above.

Select Prometheus as your data source when prompted during the dashboard import process.

Customize the dashboard filters to select the correct Spark application (or Spark Operator) using labels that were set in your ServiceMonitor.

# Step 5: Validate the Setup
Check Prometheus Targets:

In the Prometheus UI (http://<prometheus-url>:9090/targets), verify that the Spark application’s ServiceMonitor targets appear under the targets section and that the status is up.
Verify Metrics in Grafana:

Open the Grafana dashboard and ensure that metrics like driver memory usage, executor status, and other JVM metrics are correctly displayed.
