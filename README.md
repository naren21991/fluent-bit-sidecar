# Fluent Bit as a Sidecar in Kubernetes

This repository demonstrates how to use **Fluent Bit as a sidecar container** in a Kubernetes pod for log collection and forwarding to a logging backend (e.g., OpenSearch, Elasticsearch, or cloud services).

---

## 📌 Overview

Fluent Bit is a **lightweight log processor**. When deployed as a **sidecar**, it:
- Collects logs from the application container via a shared volume.
- Parses structured/unstructured logs.
- Forwards logs to a backend (e.g., OpenSearch).

This approach keeps logging concerns separate from your application, improving **observability**, **scalability**, and **maintainability**.

---

## ⚙️ Configuration

### **Containers**
- **Application container** → generates logs (writes to `/var/log/<app>`).
- **Fluent Bit container** → tails, parses, and forwards logs.

### **Resources**
- Lightweight (≈ `100m` CPU, `100Mi` memory).  
- Tune based on log volume.

### **Environment Variables**
Fluent Bit uses env vars for backend configuration:
- `FLUENT_OPENSEARCH_HOST` → Logging backend hostname  
- `FLUENT_OPENSEARCH_PORT` → Backend port (e.g., `9200`)  
- `FLUENT_OPENSEARCH_USER` → Username  
- `FLUENT_OPENSEARCH_PASSWORD` → Password (⚠️ use **Kubernetes Secrets**)  
- `FLUENT_LOG_LEVEL` → Optional (e.g., `info`, `debug`)  

### **Volumes**
- **Log Volume** (`/var/log/<app>`) → Shared between app & Fluent Bit.  
- **Config Volume** (`/fluent-bit/etc/`) → Mounts configuration from a ConfigMap.  

---

## 📂 Fluent Bit ConfigMap

A sample configuration (mounted at `/fluent-bit/etc/`):

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluent-bit
  namespace: logging
data:
  fluent-bit.conf: |
    [SERVICE]
        Flush        1
        Daemon       Off
        Log_Level    info
        Parsers_File parsers.conf

    @INCLUDE input.conf
    @INCLUDE filter.conf
    @INCLUDE output.conf

  input.conf: |
    [INPUT]
        Name              tail
        Path              /var/log/hello/hello.log
        Tag               hello.log
        Refresh_Interval  5
        Read_from_Head    true
        Key               log

  filter.conf: |
    [FILTER]
        Name        parser
        Match       hello.log
        Key_Name    log
        Parser      json
        Reserve_Data true

  output.conf: |
    [OUTPUT]
        Name  opensearch
        Match *
        Host ${FLUENT_OPENSEARCH_HOST}
        Port ${FLUENT_OPENSEARCH_PORT}
        HTTP_User ${FLUENT_OPENSEARCH_USER}
        HTTP_Passwd ${FLUENT_OPENSEARCH_PASSWORD}
        Index hello-logs
        tls On
        tls.verify Off
        Logstash_Format Off
        Logstash_Prefix fluent-bit
        Suppress_Type_Name On

  parsers.conf: |
    [PARSER]
        Name        json
        Format      json
        Time_Key    timestamp
        Time_Format %Y-%m-%dT%H:%M:%S%z
