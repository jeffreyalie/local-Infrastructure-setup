
## Kube promethius stack diagram

---
```
                                [ USER / ADMIN ]
                                        |
                         1. Apply YAML (ServiceMonitors, Rules) <----------------- Via gitea repo
                                        |
                                        v
[--------------------------- KUBERNETES API SERVER ---------------------------]
                                        |
                                        | (Watches for changes)
                                        v
                         [ PROMETHEUS OPERATOR POD ]
                         "The Manager / Foreman"
                                        |
           -------------------------------------------------------------
           | (Configures)               | (Configures)                 | (Configures)
           v                            v                              v
   [ PROMETHEUS POD ] <---------- [ ALERTMANAGER ]                [ GRAFANA POD ]
   "The Database"      (Alerts)    "The Notifier"                 "The Visualizer"
           |                                                           |
           | 2. PULL / SCRAPE                                          | 3. QUERY
           | (Every 15-30s)                                            | (On Demand)
           |                                                           |
           |                                                           v
           |                                                     (User Web Browser)
           |                                                     "lxd-dashboard.local"
|          | 
           |
           |
           |                           
           |---> [ NODE EXPORTER ] ----> (Hardware/OS Metrics)
           |
           |---> [ YOUR GO APP ] 
                    |
                    |-- (:8080/metrics) <--- (Exposed via Go Client Library) (API for Prometheus to pull)
                    |
                    |-- [ INTERNAL APP LOGIC ]
                            |
                            |-- (LXD SDK) ----> [ LXD SERVER ]
                            |-- (Vault SDK) ---> [ OPENBAO ]
```

---
## Kube-promethius-grafana stack components

```
Component	        Pod Name (Typical)	                Purpose
Prometheus	        prometheus-kube-prometheus-0	        The database that "scrapes" and stores metrics.
Grafana	                grafana-xxxxxxx-xxxx	                The web UI where you build dashboards.
Alertmanager	        alertmanager-kube-prometheus-0	        Handles the logic for sending emails/Slack pings when things break.
Operator	        prometheus-operator-xxxx-xxxx	        The manager that coordinates the pods above.
Node Exporter	        prometheus-node-exporter-xxxx	        One pod per node in your cluster to monitor hardware.
                                                                label them via ServiceMonitors yaml and operator auto picks.
Go App                  Go app pod                              (:8080/metrics) <--- (Exposed via Go Client Library) (API for Prometheus to pull)
                                                                label them via ServiceMonitors yaml and operator auto picks.
```
---

## Node exporter connects via

```
Method	                What the Exporter uses to "Login"	Where you put the Creds
Node Exporter	        Local System Permissions	        Linux User/Group (root/prometheus)
SNMP	                Community String / SNMPv3 Auth	        Exporter snmp.yml config
HTTP API	        API Key / Basic Auth	                Exporter config file or Env Vars
Metrics Endpoints	Bearer Tokens / TLS Certs	        Exporter config file
Logs	                OS Read Permissions	                chmod / chown on the log files

```
---

## Note
Go app / exporter dont need authentication to prometheus: We label them via ServiceMonitors yaml and operator auto picks.
Go app/exporter only need authentication for the hardware/VM/system they are collecting data from





