📢 Note on the Use of AI in This Guide
This technical setup guide was created with the assistance of AI tools to ensure clarity, thoroughness, and ease of use. While every effort was made to write detailed instructions and anticipate potential issues, Kubernetes environments can vary, and slight differences may cause unexpected errors.

It is highly recommended that as you follow this guide, you leverage AI tools (such as ChatGPT or similar) alongside your work — especially when copying YAML files, running commands in a highly precise terminal environment, or troubleshooting errors that may arise. AI can help identify small syntax issues, environment mismatches, or configuration adjustments quickly, greatly improving your experience and efficiency during setup.

Using AI as a real-time assistant will help ensure a smoother, faster, and more confident deployment process.
________________________________________________________________________________________________________

📈 Kubernetes Monitoring Pipeline Setup
Welcome!
This guide explains how we built a lightweight, scalable pipeline to monitor Kubernetes resource usage using InfluxDB and Telegraf.

It’s broken into two parts:
Overview: What the monitoring pipeline does and why it matters.
Technical Setup: How you can recreate it yourself (with YAML files and kubectl commands).
________________________________________________________________________________________________________

OVERVIEW:

📖 About This Monitoring Setup
This project installs a lightweight system inside your Kubernetes cluster that automatically collects resource usage data and stores it for you.

It uses two main tools: Telegraf and InfluxDB.
________________________________________________________________________________________________________

🛠️ Techstack:

📦 Telegraf
Telegraf is a lightweight agent that collects data.
It runs on every Kubernetes node (one per node).
It scrapes metrics like CPU usage, memory usage, pod counts, and more.
It sends all the collected data to a database (InfluxDB) for storage.
Think of Telegraf like a "sensor" that watches your cluster and takes regular snapshots of its health.


💾 InfluxDB
InfluxDB is a time-series database.
It stores the resource metrics collected by Telegraf.
It lets you query and analyze your historical data later.
It can connect to visualization tools like Grafana if you want dashboards.
Think of InfluxDB like a "time machine" — it remembers everything about your cluster’s resource usage over time.


🔗 How the Pipeline Works
Telegraf runs automatically on every Kubernetes node.
Every few seconds, Telegraf scrapes live data (how busy each node and pod is).
Telegraf forwards that data to InfluxDB.
InfluxDB stores the data safely inside your cluster.

You can then:
Query InfluxDB directly to see trends over time
(Optional) Hook up Grafana to create live dashboards


📈 How You Will Use This Setup
You will install InfluxDB and Telegraf once, using provided YAML files.
Metrics will automatically start flowing from your Kubernetes nodes into InfluxDB.
You can view, query, or export your data from InfluxDB anytime.
No manual scraping, no external tools — everything runs inside your cluster, securely.

________________________________________________________________________________________________________

🛠️ Full Technical Setup Guide:
Deploying Kubernetes Monitoring Pipeline (Telegraf + InfluxDB) on GKE

📖 Overview
This guide walks you through setting up a lightweight monitoring system inside your Google Kubernetes Engine (GKE) cluster.

You’ll deploy:
InfluxDB — stores your metrics
Telegraf — collects Kubernetes metrics and sends them to InfluxDB

Both components run securely inside your private cluster — no external exposure unless you choose to add it later.
________________________________________________________________________________________________________

DISCLAIMER: COPY & PASTE THE CODE IN THE QUOTES: '' INTO YOUR TERMINAL

📦 Part 1: Deploy InfluxDB (Database for Metrics Storage)
Step 1. Create the Monitoring Namespace
Why? To keep monitoring components separate from your application workloads for cleaner organization.

COMMAND: 'kubectl create namespace monitoring'

Step 2. Create a Kubernetes Secret for InfluxDB Credentials
Why? Secrets securely store sensitive information like passwords without hardcoding them into deployment files.

COMMAND: 
'kubectl -n monitoring create secret generic influxdb-info \
  --from-literal=user=influxadmin \
  --from-literal=password="<YOUR_PASSWORD>" \
  --from-literal=org="kube-monitoring" \
  --from-literal=bucket="telegraf"'
  
🔵 Replace <YOUR_PASSWORD> with your secure password.


Step 3. Create the InfluxDB Deployment YAML
Why?This YAML file defines all required InfluxDB resources: namespace, secret, storage, deployment, and service.

Create the file
COMMAND: 'nano influxdb-only.yml'

Paste the following YAML:

yaml
Copy
Edit
---
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
---
apiVersion: v1
kind: Secret
metadata:
  name: influxdb-info
  namespace: monitoring
stringData:
  user: influxadmin
  password: "<YOUR_PASSWORD>"
  org: kube-monitoring
  bucket: telegraf
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: influxdb-pvc
  namespace: monitoring
spec:
  accessModes: ["ReadWriteMany"] # Adjust to ["ReadWriteOnce"] if your GKE storage class requires
  resources:
    requests:
      storage: 10Gi
---
apiVersion: v1
kind: Service
metadata:
  name: influxdb
  namespace: monitoring
spec:
  ports:
    - port: 8086
      targetPort: 8086
  selector:
    app: influxdb
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: influxdb
  namespace: monitoring
spec:
  replicas: 1
  selector:
    matchLabels:
      app: influxdb
  template:
    metadata:
      labels:
        app: influxdb
    spec:
      containers:
        - name: influxdb
          image: influxdb:2.6
          env:
            - name: INFLUXD_ADMIN_USER
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: user
            - name: INFLUXD_ADMIN_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: password
            - name: INFLUXD_ORG
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: org
            - name: INFLUXD_BUCKET
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: bucket
          ports:
            - containerPort: 8086
          volumeMounts:
            - name: storage
              mountPath: /var/lib/influxdb2
      volumes:
        - name: storage
          persistentVolumeClaim:
            claimName: influxdb-pvc

            
Step 4. Apply the InfluxDB Manifest

COMMAND: 'kubectl apply -f influxdb-only.yml'


Step 5. Verify InfluxDB is Running
COMMAND: 'kubectl -n monitoring get pods,svc'

✅ Confirm:
A Pod like influxdb-xxxxx is Running
A Service influxdb is exposing port 8086/TCP


Step 6. Port-Forward to Access InfluxDB Locally
COMMAND: 'kubectl -n monitoring port-forward svc/influxdb 8086:8086'

Paste into web searchbar:
http://localhost:8086

✅ You should see the InfluxDB login page.


📈 Part 2: Deploy Telegraf (Metric Collector)
Step 1. Create the Telegraf Deployment YAML

Create the file:
COMMAND: 'nano telegraf-only.yml'

Paste the following YAML:

yaml
Copy
Edit
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: telegraf-config
  namespace: monitoring
data:
  telegraf.conf: |
    [[inputs.kubernetes]]
      bearer_token = "/var/run/secrets/kubernetes.io/serviceaccount/token"
    [[inputs.kube_inventory]]
    [[outputs.influxdb_v2]]
      urls = ["http://influxdb.monitoring.svc.cluster.local:8086"]
      token = "${INFLUX_TOKEN}"
      organization = "${INFLUX_ORG}"
      bucket = "${INFLUX_BUCKET}"
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: telegraf
  namespace: monitoring
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: telegraf-monitoring
subjects:
  - kind: ServiceAccount
    name: telegraf
    namespace: monitoring
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: telegraf
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: telegraf
  template:
    metadata:
      labels:
        app: telegraf
    spec:
      serviceAccountName: telegraf
      containers:
        - name: telegraf
          image: influxdata/telegraf:1.34
          env:
            - name: INFLUX_TOKEN
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: token
            - name: INFLUX_ORG
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: org
            - name: INFLUX_BUCKET
              valueFrom:
                secretKeyRef:
                  name: influxdb-info
                  key: bucket
          volumeMounts:
            - name: config
              mountPath: /etc/telegraf/telegraf.conf
              subPath: telegraf.conf
      volumes:
        - name: config
          configMap:
            name: telegraf-config

            
Step 2. Apply the Telegraf Manifest
COMMAND: 'kubectl apply -f telegraf-only.yml'

Step 3. Verify Telegraf Pods
COMMAND: 'kubectl -n monitoring get pods -l app=telegraf'
✅ Each node should have one telegraf-xxxxx pod Running.


Step 4. Inspect Telegraf Logs
COMMAND: 'kubectl -n monitoring logs -l app=telegraf --tail=20'

✅ Look for:
I! Loaded inputs: kube_inventory kubernetes
I! Loaded outputs: influxdb_v2
________________________________________________________________________________________________________

📊 How to Log into InfluxDB, Visualize Data, and Create Dashboards
🔑 Log Into InfluxDB

Keep your port-forward running:
COMMAND: 'kubectl -n monitoring port-forward svc/influxdb 8086:8086'

Open your browser:
http://localhost:8086

Log in using:
Username: influxadmin
Password: <YOUR_PASSWORD>
Organization: kube-monitoring


🔎 Explore Your Data
Inside the InfluxDB UI:

Click Data Explorer.
Select Bucket → telegraf.
Pick a measurement like kubernetes_pod_container.
Start querying metrics like CPU, memory, pod statuses.

✅ Data points should appear every few seconds.



📈 Create a Dashboard
Go to Dashboards → + Create Dashboard.
Add a Cell (graph, gauge, table, etc.).

Set:
Source Bucket: telegraf
Fields: e.g., usage_cpu_nanocores, memory_usage_bytes
Choose a visualization (e.g., line chart).
Save your dashboard!
✅ Now you have live dashboards tracking your Kubernetes cluster resource usage!



🧠 How to Build Your Own Queries for Custom Visualizations
Using the Visual Query Builder
Inside InfluxDB:

Go to Data Explorer.
Choose your bucket and measurement.
Pick the field you want to graph.
Set time ranges and filters easily.
✅ No coding needed!


Writing Your Own Queries (Flux)
If you want more control, you can write your own Flux queries.

📈 Example 1: CPU Usage per Pod
flux
from(bucket: "telegraf")
  |> range(start: -15m)
  |> filter(fn: (r) => r._measurement == "kubernetes_pod_container")
  |> filter(fn: (r) => r._field == "usage_cpu_nanocores")
  |> group(columns: ["pod_name"])
  |> aggregateWindow(every: 1m, fn: mean)
  |> yield(name: "mean")
✅ Shows average CPU usage per pod, smoothed over 1-minute windows.



📊 Example 2: Memory Usage per Node
flux
from(bucket: "telegraf")
  |> range(start: -30m)
  |> filter(fn: (r) => r._measurement == "kubernetes_node")
  |> filter(fn: (r) => r._field == "memory_usage_bytes")
  |> group(columns: ["node_name"])
  |> aggregateWindow(every: 5m, fn: max)
  |> yield(name: "max")
✅ Shows maximum memory usage per node over the last 30 minutes.



🎯 Final Takeaways
Explore → Build fast queries
Customize → Write your own queries
Visualize → Create powerful live dashboards
✅ You now have full control over monitoring your Kubernetes cluster with real-time visualizations!



References:
link: https://www.influxdata.com/blog/deploying-influxdb-telegraf-to-monitor-kubernetes/

