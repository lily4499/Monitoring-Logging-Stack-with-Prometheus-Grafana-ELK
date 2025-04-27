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

## 🛠️ Steps, CLI Commands, and Purpose

### 1. Create Kubernetes Namespaces
```bash
kubectl apply -f namespaces/monitoring-namespace.yaml
kubectl apply -f namespaces/logging-namespace.yaml
```
**Purpose:** Isolate monitoring and logging components into separate Kubernetes namespaces for better organization and security.

### 2. Deploy Prometheus
```bash
kubectl apply -f prometheus/prometheus-configmap.yaml
kubectl apply -f prometheus/prometheus-deployment.yaml
kubectl apply -f prometheus/prometheus-service.yaml
```
**Purpose:** Deploy Prometheus to collect CPU, memory, and other Kubernetes metrics.

### 3. Deploy Grafana with Auto-Provisioned Dashboards and Alerts
```bash
kubectl apply -f grafana/grafana-dashboard-provider.yaml
kubectl apply -f grafana/grafana-dashboards.yaml
kubectl apply -f grafana/grafana-deployment.yaml
kubectl apply -f grafana/grafana-service.yaml
kubectl apply -f grafana/grafana-datasource.yaml
```
**Purpose:** Deploy Grafana and automatically configure dashboards and Prometheus datasource for visualization.

### 4. Deploy ELK Stack
```bash
kubectl apply -f elk/elasticsearch-deployment.yaml
kubectl apply -f elk/kibana-deployment.yaml
kubectl apply -f elk/filebeat-daemonset.yaml
```
**Purpose:** Deploy Elasticsearch for storing logs, Kibana for log visualization, and Filebeat to collect and forward container logs.

### 5. Deploy OAuth2 Proxy for Grafana Authentication
```bash
kubectl apply -f grafana/oauth2-proxy-deployment.yaml
```
**Purpose:** Secure Grafana by requiring GitHub (or other OAuth provider) login for access.

### 6. Create RBAC for Securing Grafana Access
```bash
kubectl apply -f grafana/grafana-rbac.yaml
```
**Purpose:** Limit dashboard access to specific users/groups using Kubernetes RBAC.

### 7. Configure Ingress for OAuth2 Authentication
```bash
kubectl apply -f grafana/oauth2-ingress.yaml
```
**Purpose:** Expose Grafana securely over the internet through Ingress with OAuth2 authentication.

### 8. Apply Grafana Auto-Provisioned Alerts
```bash
kubectl apply -f grafana/alerts.yaml
```
**Purpose:** Automatically set up critical CPU and memory usage alerts in Grafana.

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

## 🎯 Bonus Script
- `setup.sh` — Automates all the above CLI steps in order.

## 🚨 Sample Alerts

**Grafana Alerts**
- CPU usage > 80% for 5 minutes.
- Memory usage > 70%.

**Kibana Alerts**
- Detect Error logs with `log.level: "error"`.
- Alert on high log volume.

## 🔒 Secured Access
- **Grafana**:
  - OAuth2 Proxy + GitHub OAuth login.
  - Slack notification integration.
- **Kibana**:
  - (Optional) OAuth2 Proxy or Kibana RBAC.
  - Slack notification via Connectors.

## 🧠 Skills Demonstrated
- Centralized monitoring (Prometheus)
- Centralized logging (ELK Stack)
- Visualization and alerting (Grafana/Kibana)
- Secure access management (OAuth, RBAC)
- Automation with YAML and scripts
- Compliance readiness (PCI/SOC2)

## 📌 Notes
- Use TLS/HTTPS for Ingress (recommended via cert-manager).
- Create GitHub OAuth App to integrate OAuth2 login.
- Ensure CPU/Memory resource limits are configured.

## 🚀 Future Expansions
- Add Loki instead of Elasticsearch.
- Use Prometheus AlertManager.
- Push custom application metrics.

---

✅ **Ready for GitHub repo and production deployment!** 🚀

