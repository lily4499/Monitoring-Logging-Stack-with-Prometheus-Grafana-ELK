# 📦 Monitoring & Logging Stack with Prometheus, Grafana & ELK

## 📋 Description
Implement centralized observability with Prometheus, Grafana, and ELK Stack for a production-grade Kubernetes app.

## 🎯 Objectives
- Monitor CPU/memory usage of containers and nodes.
- Create Grafana dashboards and alerts automatically.
- Ship logs from containers to Elasticsearch using Filebeat.
- Visualize logs and create alert rules in Kibana.
- Secure dashboards access with OAuth2 proxy and RBAC.
- Integrate Slack for alert notifications.

## 🗂️ Final Folder Structure
```bash
monitoring-logging-stack/
├── grafana/
│   ├── grafana-deployment.yaml
│   ├── grafana-service.yaml
│   ├── grafana-datasource.yaml
│   ├── grafana-dashboard-provider.yaml
│   ├── grafana-dashboards.yaml
│   ├── alerts.yaml
│   ├── oauth2-proxy-deployment.yaml
│   ├── oauth2-ingress.yaml
│   ├── grafana-rbac.yaml
├── prometheus/
│   ├── prometheus-deployment.yaml
│   ├── prometheus-service.yaml
│   ├── prometheus-configmap.yaml
├── elk/
│   ├── elasticsearch-deployment.yaml
│   ├── kibana-deployment.yaml
│   ├── filebeat-daemonset.yaml
├── namespaces/
│   ├── monitoring-namespace.yaml
│   ├── logging-namespace.yaml
├── README.md
├── setup.sh
```

---
## file-setup.py

```python
import os

# Define the base path
base_path = "/home/lilia/VIDEOS/monitoring-logging-stack"

# Define the file structure and their real-world contents
files = {
    "grafana/grafana-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: grafana
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: grafana
  template:
    metadata:
      labels:
        app: grafana
    spec:
      containers:
      - name: grafana
        image: grafana/grafana:latest
        ports:
        - containerPort: 3000
""",

    "grafana/grafana-service.yaml": """apiVersion: v1
kind: Service
metadata:
  name: grafana-service
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300
  selector:
    app: grafana
""",

    "grafana/grafana-datasource.yaml": """apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-datasource
  namespace: monitoring
data:
  datasource.yaml: |
    apiVersion: 1
    datasources:
      - name: Prometheus
        type: prometheus
        access: proxy
        url: http://prometheus-service:9090
""",

    "grafana/grafana-dashboard-provider.yaml": """apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboard-provider
  namespace: monitoring
data:
  provider.yaml: |
    apiVersion: 1
    providers:
      - name: 'default'
        orgId: 1
        folder: ''
        type: file
        disableDeletion: false
        updateIntervalSeconds: 10
        options:
          path: /var/lib/grafana/dashboards
""",

    "grafana/grafana-dashboards.yaml": """apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-dashboards
  namespace: monitoring
data:
  cpu-memory-dashboard.json: |
    {
      "title": "CPU & Memory Usage",
      "panels": [
        {
          "type": "graph",
          "title": "CPU Usage",
          "targets": [{
            "expr": "sum(rate(container_cpu_usage_seconds_total{container!=''}[5m])) by (pod)",
            "legendFormat": "{{pod}}"
          }]
        },
        {
          "type": "graph",
          "title": "Memory Usage",
          "targets": [{
            "expr": "sum(container_memory_usage_bytes{container!=''}) by (pod)",
            "legendFormat": "{{pod}}"
          }]
        }
      ]
    }
""",

    "grafana/alerts.yaml": """apiVersion: 1
groups:
- name: container-alerts
  interval: 1m
  rules:
  - uid: cpu-high-usage
    title: CPU High Usage
    condition: B
    data:
      - refId: A
        datasourceUid: prometheus
        model:
          expr: sum(rate(container_cpu_usage_seconds_total{container!=""}[5m])) by (pod)
      - refId: B
        datasourceUid: prometheus
        model:
          reduce: mean()
          conditions:
          - evaluator:
              type: gt
              params: [0.8]
            operator: and
            reducer: last
            type: query
    for: 5m
    annotations:
      summary: High CPU usage detected
    labels:
      severity: warning
""",

    "grafana/oauth2-proxy-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: oauth2-proxy
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: oauth2-proxy
  template:
    metadata:
      labels:
        app: oauth2-proxy
    spec:
      containers:
      - name: oauth2-proxy
        image: quay.io/oauth2-proxy/oauth2-proxy:latest
        args:
        - --provider=github
        - --email-domain=*
        - --upstream=http://grafana-service:3000
        - --cookie-secret=<your-cookie-secret>
        - --client-id=<your-client-id>
        - --client-secret=<your-client-secret>
        ports:
        - containerPort: 4180
""",

    "grafana/oauth2-ingress.yaml": """apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: monitoring
  annotations:
    kubernetes.io/ingress.class: nginx
    nginx.ingress.kubernetes.io/auth-url: "https://grafana.example.com/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://grafana.example.com/oauth2/start?rd=$request_uri"
spec:
  rules:
  - host: grafana.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: oauth2-proxy
            port:
              number: 4180
""",

    "grafana/grafana-rbac.yaml": """apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: monitoring
  name: grafana-access-role
rules:
- apiGroups: [""]
  resources: ["services", "pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: grafana-access-binding
  namespace: monitoring
subjects:
- kind: User
  name: <github-username>
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: grafana-access-role
  apiGroup: rbac.authorization.k8s.io
""",

    "prometheus/prometheus-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: prometheus-server
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: prometheus
  template:
    metadata:
      labels:
        app: prometheus
    spec:
      containers:
      - name: prometheus
        image: prom/prometheus:latest
        ports:
        - containerPort: 9090
""",

    "prometheus/prometheus-service.yaml": """apiVersion: v1
kind: Service
metadata:
  name: prometheus-service
  namespace: monitoring
spec:
  type: NodePort
  ports:
  - port: 9090
    targetPort: 9090
    nodePort: 30090
  selector:
    app: prometheus
""",

    "prometheus/prometheus-configmap.yaml": """apiVersion: v1
kind: ConfigMap
metadata:
  name: prometheus-config
  namespace: monitoring
data:
  prometheus.yml: |
    global:
      scrape_interval: 15s
    scrape_configs:
      - job_name: 'kubernetes'
        kubernetes_sd_configs:
          - role: node
""",

    "elk/elasticsearch-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: elasticsearch
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
      - name: elasticsearch
        image: docker.elastic.co/elasticsearch/elasticsearch:8.3.0
        env:
        - name: discovery.type
          value: single-node
        ports:
        - containerPort: 9200
""",

    "elk/kibana-deployment.yaml": """apiVersion: apps/v1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
spec:
  replicas: 1
  selector:
    matchLabels:
      app: kibana
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
      - name: kibana
        image: docker.elastic.co/kibana/kibana:8.3.0
        ports:
        - containerPort: 5601
        env:
        - name: ELASTICSEARCH_HOSTS
          value: "http://elasticsearch:9200"
""",

    "elk/filebeat-daemonset.yaml": """apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: filebeat
  namespace: logging
spec:
  selector:
    matchLabels:
      app: filebeat
  template:
    metadata:
      labels:
        app: filebeat
    spec:
      containers:
      - name: filebeat
        image: docker.elastic.co/beats/filebeat:8.3.0
        args: ["-e", "-E", "output.elasticsearch.hosts=['http://elasticsearch:9200']"]
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
""",

    "namespaces/monitoring-namespace.yaml": """apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
""",

    "namespaces/logging-namespace.yaml": """apiVersion: v1
kind: Namespace
metadata:
  name: logging
""",

    "README.md": """# Monitoring & Logging Stack with Prometheus, Grafana & ELK

Follow the README instructions to deploy Monitoring and Logging system with Prometheus, Grafana, and ELK Stack!
""",

    "setup.sh": """#!/bin/bash

echo "Applying Namespaces..."
kubectl apply -f namespaces/

echo "Deploying Prometheus..."
kubectl apply -f prometheus/

echo "Deploying Grafana..."
kubectl apply -f grafana/

echo "Deploying ELK Stack..."
kubectl apply -f elk/

echo "✅ Monitoring and Logging Stack Setup Complete!"
"""
}

# Create directories and files
for filepath, content in files.items():
    full_path = os.path.join(base_path, filepath)
    os.makedirs(os.path.dirname(full_path), exist_ok=True)
    with open(full_path, 'w') as file:
        file.write(content)

print(f"All files created successfully under {base_path}")


```

---

## 🛠️ Steps, CLI Commands, and Purpose
---
### 1. Create Kubernetes Namespaces
```bash
kubectl apply -f namespaces/monitoring-namespace.yaml
kubectl apply -f namespaces/logging-namespace.yaml
```
**Purpose:** Isolate monitoring and logging components into separate Kubernetes namespaces for better organization and security.
---
### 2. Deploy Prometheus
```bash
kubectl apply -f prometheus/prometheus-configmap.yaml
kubectl apply -f prometheus/prometheus-deployment.yaml
kubectl apply -f prometheus/prometheus-service.yaml
```
**Purpose:** Deploy Prometheus to collect CPU, memory, and other Kubernetes metrics.
---
### 3. Deploy Grafana with Auto-Provisioned Dashboards and Alerts
```bash
kubectl apply -f grafana/grafana-dashboard-provider.yaml
kubectl apply -f grafana/grafana-dashboards.yaml
kubectl apply -f grafana/grafana-deployment.yaml
kubectl apply -f grafana/grafana-service.yaml
kubectl apply -f grafana/grafana-datasource.yaml
```
**Purpose:** Deploy Grafana and automatically configure dashboards and Prometheus datasource for visualization.
---
### 4. Deploy ELK Stack
```bash
kubectl apply -f elk/elasticsearch-deployment.yaml
kubectl apply -f elk/kibana-deployment.yaml
kubectl apply -f elk/filebeat-daemonset.yaml
```
**Purpose:** Deploy Elasticsearch for storing logs, Kibana for log visualization, and Filebeat to collect and forward container logs.
---
### 5. Deploy OAuth2 Proxy for Grafana Authentication
```bash
kubectl apply -f grafana/oauth2-proxy-deployment.yaml
```
**Purpose:** Secure Grafana by requiring GitHub (or other OAuth provider) login for access.
---
### 6. Create RBAC for Securing Grafana Access
```bash
kubectl apply -f grafana/grafana-rbac.yaml
```
**Purpose:** Limit dashboard access to specific users/groups using Kubernetes RBAC.
---
### 7. Configure Ingress for OAuth2 Authentication
```bash
kubectl apply -f grafana/oauth2-ingress.yaml
```
**Purpose:** Expose Grafana securely over the internet through Ingress with OAuth2 authentication.
---
### 8. Apply Grafana Auto-Provisioned Alerts
```bash
kubectl apply -f grafana/alerts.yaml
```
**Purpose:** Automatically set up critical CPU and memory usage alerts in Grafana.
---
### 9. Set Up Slack Webhooks

- Grafana: Create Slack Contact Point manually or via API.
```bash
For Grafana (Slack Contact Point)
Option 1: Manually in Grafana UI
Go to Alerting → Contact Points → New Contact Point.
Choose Slack as the type.
Paste your Slack Webhook URL (you get it by creating a Slack Incoming Webhook App).
Set a name like slack-notifications.
Save.

Option 2: Using Grafana API (Automated)
curl -X POST \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <your-grafana-api-key>" \
  -d '{
    "name": "slack-notifications",
    "type": "slack",
    "settings": {
      "url": "https://hooks.slack.com/services/<your-webhook-id>"
    }
  }' \
  http://<grafana-ip>:3000/api/contact-points
✅ After this, when setting an alert, you choose the slack-notifications contact point.

```
  
- Kibana: Create Slack Connector via Stack Management > Connectors.
```plain text
For Kibana (Slack Connector)
Manual setup through Kibana UI:
Go to Stack Management → Rules and Connectors → Connectors → Create Connector.
Choose Slack as the connector type.
Paste your Slack Webhook URL.
Save the connector (example name: slack-alerts).

✅ After this, when you create a Rule in Kibana (for error logs or log volume spikes), select this Slack Connector under Actions.
```
  
**Purpose:** Configure Slack to receive alert notifications from Grafana and Kibana.
---
### 10. Create Example Alerts
- Grafana: CPU/Memory alerts are auto-provisioned via `alerts.yaml`.
  
- Kibana: Manually create Rule:
  - Rule Type: Kibana Query Rule
  - Query: `log.level: "error"`
 ```plain text
📜 For Kibana (Manually Create Rule)
Manually create a Rule in Kibana for Error Log Detection:
  Go to Stack Management → Rules and Connectors → Create Rule.
  Select Rule Type: Kibana Query Rule.
  Configure the Rule:
  - Index pattern: filebeat-*
  - KQL Query:
        log.level: "error"
  - Condition: More than 0 matches within 1 minute.
Set Action: Use the Slack Connector you created earlier.
Save.

✅ This will trigger Slack notifications if any error logs are detected in real-time.
``` 
**Purpose:** Enable real-time alerts for system anomalies and error logs.
---
## 🎯 Bonus Script
- `setup.sh` — Automates all the above CLI steps in order.
---
## 🚨 Sample Alerts

**Grafana Alerts**
- CPU usage > 80% for 5 minutes.
- Memory usage > 70%.

**Kibana Alerts**
- Detect Error logs with `log.level: "error"`.
- Alert on high log volume.
---
## 🔒 Secured Access
- **Grafana**:
  - OAuth2 Proxy + GitHub OAuth login.
  - Slack notification integration.
- **Kibana**:
  - (Optional) OAuth2 Proxy or Kibana RBAC.
  - Slack notification via Connectors.
---
## 🧠 Skills Demonstrated
- Centralized monitoring (Prometheus)
- Centralized logging (ELK Stack)
- Visualization and alerting (Grafana/Kibana)
- Secure access management (OAuth, RBAC)
- Automation with YAML and scripts
- Compliance readiness (PCI/SOC2)
---
## 📌 Notes
- Use TLS/HTTPS for Ingress (recommended via cert-manager).
- Create GitHub OAuth App to integrate OAuth2 login.
- Ensure CPU/Memory resource limits are configured.
---
## 🚀 Future Expansions
- Add Loki instead of Elasticsearch.
- Use Prometheus AlertManager.
- Push custom application metrics.

---



