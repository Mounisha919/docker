
# 🔥 Dockerized Microservices Logging — Beginner-Friendly but Senior-Sounding

---

## 1. The Big Picture (One Line)

👉 Apps in containers write logs → Docker/K8s stores them on the node → an agent like Fluent Bit/Fluentd/Filebeat reads those logs → sends them to Elasticsearch (where logs are stored and indexed) → Kibana shows the logs in dashboards and alerts.

---

## 2. Who Does What (Roles)

* **App (inside container)** → Just prints logs (`stdout`/`stderr`).
* **Docker/K8s runtime** → Saves those logs as files on the node (like `/var/log/containers/...`).
* **Log collector (Fluent Bit/Fluentd/Filebeat)** → Reads (tails) the log files → adds extra info (like pod name, namespace) → forwards logs.
* **Log aggregator (Fluentd/Logstash)** → Optional middle layer → can transform data, buffer, and balance load.
* **Elasticsearch (ES)** → Database that stores logs and makes them searchable.
* **Kibana** → UI to view/search logs, make dashboards, and set alerts.

---

## 3. Why ELK/EFK?

* **EFK (Elasticsearch + Fluentd + Kibana)** → Open source, full control.
* **Splunk** → Paid, enterprise, very good but costly.
* **Cloud logging (AWS CloudWatch, GCP Logging)** → Easy setup but less flexible if you run hybrid/multi-cloud.

---

## 4. How Logs Flow (Step by Step)

1. App writes log → goes to `stdout/stderr`.
2. Docker/containerd saves logs into files (`.log` files).
3. Fluent Bit/Fluentd tails those files.
4. The agent adds Kubernetes info (namespace, pod name, labels).
5. Filtering: remove DEBUG logs, mask passwords, parse stack traces.
6. Forward logs securely (TLS + auth) to Elasticsearch.
7. Elasticsearch indexes the logs.
8. Kibana shows dashboards & alerts.
9. Old logs → moved to cheaper storage (S3, cold nodes) or deleted after retention.

---

## 5. MUST DO in Production

* Use **JSON structured logs** (not plain text). Example:

  ```json
  {
    "@timestamp": "2025-09-30T12:00:00Z",
    "level": "ERROR",
    "service": "payments",
    "trace_id": "abc123",
    "user_id": "u456",
    "message": "Payment failed",
    "http.status": 500
  }
  ```
* Add **correlation IDs** (trace id / request id) so you can follow one request across many services.

---

## 6. Docker & K8s Specifics

### Log rotation in Docker

If you don’t rotate logs, containers can fill disk.
Fix → `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Restart Docker → problem solved.

### Sending logs directly to Fluentd

```bash
docker run --log-driver=fluentd \
  --log-opt fluentd-address=10.0.0.5:24224 \
  myapp
```

In K8s → mostly use DaemonSet log collectors.

---

## 7. Fluentd / Fluent Bit Setup (Easy Words)

* Runs on each node (DaemonSet in K8s).
* Mounts `/var/log/containers` to read logs.
* Config:

  * **source** = where logs come from
  * **filter** = enrich/modify logs
  * **match** = send logs to Elasticsearch

Example (simplified):

```xml
<source>
  @type tail
  path /var/log/containers/*.log
  @type json
</source>

<filter kubernetes.**>
  @type kubernetes_metadata
</filter>

<match kubernetes.**>
  @type elasticsearch
  host es-logging
  port 9200
</match>
```

---

## 8. Elasticsearch in Simple Terms

* Logs are stored in **time-based indexes** (like per day).
* **ILM (Index Lifecycle Management)**:

  * Hot = recent logs, fast search.
  * Warm = older logs, less replicas.
  * Cold = rarely used, cheap storage.
  * Delete = after X days (say 30 days).

Example → keep logs 30 days in ES → after that → snapshot to S3 or delete.

---

## 9. Kibana (UI)

* Create dashboards → error rate per service, slow endpoints.
* Search logs → `"service:payments AND level:ERROR"`.
* Alerts → e.g., "if error rate >100/min in payments service, send Slack alert".

---

## 10. Real-Time Problems & Fixes (War Stories)

### Problem 1: Disk full

* Symptom → node unreachable, pods crash.
* Check → `du -sh /var/lib/docker/containers/*`.
* Fix → set Docker log rotation (`daemon.json`).

### Problem 2: Missing logs in Kibana

* Symptom → logs from some pods not showing.
* Check → `kubectl logs ds/fluentd -n kube-logging`.
* Check log path exists `/var/log/containers/`.
* Fix → correct volume mounts, restart DaemonSet.

### Problem 3: Elasticsearch red cluster

* Symptom → queries fail, Kibana shows red.
* Check → `curl -X GET :9200/_cluster/health?pretty`.
* Fix → add disk space, move old data to warm/cold, or delete indices.

### Problem 4: High log volume → ES slow

* Symptom → delay in seeing logs.
* Fix → add ES ingest nodes, reduce DEBUG logs, use Fluent Bit for better performance.

---

## 11. Security & Compliance

* Encrypt traffic (TLS everywhere).
* Use API keys or certs for agents → ES.
* Mask sensitive data (credit card, passwords) before sending logs.
* Control access in Kibana (RBAC).

---

## 12. How to Sound Experienced in Interview (Cheat Script)

👉 Example Answer:

> “We don’t rely on `docker logs` in production. We use a centralized logging pipeline. Fluent Bit runs as DaemonSet, tails container logs, enriches with Kubernetes metadata, and forwards to Elasticsearch. We use ILM in Elasticsearch so hot data stays on SSD, older data moves to warm/cold storage, and snapshots go to S3. Kibana is used for dashboards, alerts, and troubleshooting. We also enable Docker log rotation to prevent disk full issues, use structured JSON logs with correlation IDs, and add TLS + RBAC for security. When we faced missing logs earlier, we debugged the Fluentd DaemonSet, fixed volume mounts, and set buffer configs to handle backpressure. This setup makes logs reliable, searchable, and cost-efficient.”

---

✅ This way → you **explain like a senior** but **clear enough for yourself**.

---

⚡ Now I can:

1. Create a **one-page printable cheat sheet** (commands + flow + common issues).
2. Or write **10 war stories** (symptom → debug → fix → prevention) that you can memorize and narrate in interviews.

👉 Which one do you want me to make first?
