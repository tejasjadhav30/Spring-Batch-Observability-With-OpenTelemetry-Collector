# Spring-Batch-Observability-With-OpenTelemetry-Collector

##  Introduction

This documentation outlines the implementation of **OpenTelemetry (Otel)** within a **Spring Batch** application. The goal is to:

* Export metrics using the **OTLP** protocol
* Visualise them via **Google Managed Prometheus (GMP)** and **Dynatrace**
* Plot custom graphs on **Grafana** for deeper insights

This setup includes application-level configurations and Helm chart updates for deploying the **OpenTelemetry Collector**.

---

##  Problem Statement

The Spring Batch application currently lacks visibility into job performance and system health due to the absence of standardized observability. Without key metrics like read/write counts or job status:

* Monitoring becomes unreliable
* Debugging is time-consuming

###  Objective

To integrate OpenTelemetry into the application to export metrics via the OTLP protocol, and visualize them using:

* Google Managed Prometheus
* Dynatrace
* Grafana

---

##  Goals

* Integrate OpenTelemetry with Spring Batch
* Use OTLP protocol for metrics
* Export to GMP and Dynatrace
* Real-time visualization with Grafana

---

##  Tech Stack

* Spring Boot (**Spring Batch**)
* Micrometer (`1.14.5`)
* OpenTelemetry Collector (`otel/opentelemetry-collector-contrib:0.126.0`)
* Google Managed Prometheus
* Dynatrace
* Grafana

---

##  Architecture Flow

1. Spring Batch exposes Micrometer metrics via OTLP on **port 4318**
2. **OpenTelemetry Collector** receives metrics through its OTLP receiver
3. Processors handle:

   * Memory management
   * Batching
   * GCP resource detection
4. Exporters forward metrics to **GMP** and optionally to logs
5. GMP stores time-series metrics
6. Grafana connects to GMP for visualization

![Screenshot 2025-07-02 172637](https://github.com/user-attachments/assets/031f38e3-ffc3-4d54-8b51-3c13bf3bab59)

---

##  Helm Values Update

> Download the helm chart of opentelemetry contrib from opentelemetry source. and make below changes.



---

## OTLP Receiver Configuration

```yaml
receivers:
  otlp:
    protocols:
      http:
        endpoint: 0.0.0.0:4318
```

---

##  Processors Configuration

```yaml
processors:
  batch:
    send_batch_max_size: 200
    send_batch_size: 200
    timeout: 5s
  memory_limiter:
    check_interval: 5s
    limit_percentage: 80
    spike_limit_percentage: 25
  resourcedetection:
    detectors: [gcp]
    timeout: 10s
  cumulativetodelta: {}
```

### Processor Purpose:

* `batch`: Bundles metrics for efficient transmission
* `memory_limiter`: Prevents OOM crashes
* `resourcedetection`: Auto-detects GCP metadata
* `cumulativetodelta`: Converts histogram to delta metrics

---

##  Exporter Configuration

```yaml
exporters:
  googlemanagedprometheus:
    project: issuer-switch
  debug: {}
```

---

##  Service Pipeline Configuration

```yaml
service:
  extensions: [health_check]
  pipelines:
    metrics:
      receivers: [otlp]
      processors: [resourcedetection, memory_limiter, batch]
      exporters: [googlemanagedprometheus, debug]
```

---

##  Kubernetes IAM Setup

### Annotate Kubernetes Service Account:

```yaml
annotations:
  iam.gke.io/gcp-service-account: gmp-test-sa@PROJECT_ID.iam.gserviceaccount.com
```

### IAM Binding:

```bash
gcloud config set project PROJECT_ID \
&&
gcloud iam service-accounts create gmp-test-sa \
&&
gcloud iam service-accounts add-iam-policy-binding \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:PROJECT_ID.svc.id.goog[NAMESPACE_NAME/default]" \
  gmp-test-sa@PROJECT_ID.iam.gserviceaccount.com \
&&
kubectl annotate serviceaccount \
  --namespace NAMESPACE_NAME \
  default \
  iam.gke.io/gcp-service-account=gmp-test-sa@PROJECT_ID.iam.gserviceaccount.com
```

### Required Roles:

* `roles/monitoring.editor`
* `roles/monitoring.metricWriter`
* `roles/monitoring.viewer`

---

##  Spring Boot Setup

### Gradle Dependencies:

```gradle
runtimeOnly 'io.micrometer:micrometer-registry-otlp:1.14.5'
implementation 'org.springframework.boot:spring-boot-starter-actuator'
```

### OTLP Export Config:

```properties
management.otlp.metrics.export.enabled=true
management.otlp.metrics.export.url=${OTLP_METRICS_EXPORT_URL:http://localhost:4318/v1/metrics}
management.otlp.metrics.export.step=1s
management.otlp.metrics.export.baseTimeUnit=SECONDS
```

---

##  Convert Histogram to Summary Metrics

### In `application.properties`:

```properties
management.metrics.distribution.percentiles.hikaricp.connections.creation=1
management.metrics.distribution.percentiles.spring.batch.chunk.write=1
management.metrics.distribution.percentiles.spring.batch.item.process=1
management.metrics.distribution.percentiles.spring.batch.item.read=1
management.metrics.distribution.percentiles.spring.batch.job.active=1
management.metrics.distribution.percentiles.spring.batch.job=1
management.metrics.distribution.percentiles.spring.batch.step.active=1
```

>  Reason: GMP has limited support for OTLP histograms, so converting to summary ensures compatibility.

---

##  Kubernetes Metadata Tags

```properties
management.opentelemetry.resource-attributes.namespace=${NAMESPACE_NAME:default}
management.opentelemetry.resource-attributes.instance=${HOSTNAME:default}
management.metrics.tags.pod=${POD_NAME}
```

---

##  ConfigMap Environment Variables

```json
{
  "name": "OTLP_METRICS_EXPORT_URL",
  "value": "http://otel-collector.management.svc.cluster.local:4318/v1/metrics"
},
{
  "name": "NAMESPACE_NAME",
  "valueFrom": {
    "fieldRef": {
      "fieldPath": "metadata.namespace"
    }
  }
},
{
  "name": "POD_NAME",
  "valueFrom": {
    "fieldRef": {
      "fieldPath": "metadata.name"
    }
  }
}
```


##  Final Result

* OTLP receives metrics on port **4318**
* Metrics are exported to **Google Managed Prometheus**
* **Grafana** visualizes and plots metrics in real-time

## Author
* Tejas Jadhav 

