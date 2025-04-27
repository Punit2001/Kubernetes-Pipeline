# Kubernetes-Pipeline
This project implements a Kubernetes-based monitoring pipeline to track CPU and memory resource consumption across pods and nodes in a cluster.
It collects, aggregates, and visualizes real-time resource usage to optimize cluster performance and prevent over-provisioning or resource exhaustion.

Tech Stack
Minikube — Local Kubernetes cluster setup

Telegraf — Metrics scraper (collects CPU and memory usage from Kubernetes pods)

InfluxDB — Time-series database to store resource metrics
