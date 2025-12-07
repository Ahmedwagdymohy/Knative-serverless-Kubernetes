#  Knative: From Zero to Production

Welcome! This repository documents my journey with **Knative**, serverless solution for Kubernetes. 


##  What is Knative?

Think of it as "Superpowers for Kubernetes." It abstracts away the boring infrastructure work (managing deployments, services, ingress) and gives you:
*   **Serverless Containers:** Run any container, anywhere.
*   **Scale-to-Zero:** Save money! If no one uses your app, it scales down to 0 pods. 📉
*   **Traffic Splitting:** Easily run canary deployments (e.g., send 10% of traffic to v2). 🚦

---

##  Prerequisites

Before you start, make sure your toolbox is ready.

### 1. Cluster Requirements
You need a Kubernetes cluster. I have already made a kubernetes cluster with the last project of [KEDA](https://github.com/Ahmedwagdymohy/Event-driven-autoscaling-KEDA-GKE) check it out for IAC files :
*   **Nodes:** 3-4
*   **Specs:** 2 vCPUs, 4GB RAM per node

### 2. CLI Tools
You will need the Knative CLI (`kn`) to interact with your services easily.

**MacOS:**
```
brew install knative/client/kn
```

**Linux:**
```
curl -L https://github.com/knative/client/releases/download/knative-v1.11.0/kn-linux-amd64 -o kn
chmod +x kn
sudo mv kn /usr/local/bin/
```

---

## Installation Guide

We are using **Knative Serving** with **Kourier** as the networking layer.

### Why Kourier?
**Kourier** is a lightweight ingress controller that routes external HTTP traffic to your Knative services. Think of it as the "traffic cop" that makes your serverless containers accessible from the outside world.

- **Knative Serving** = The engine that manages your serverless apps (autoscaling, revisions, etc.)
- **Kourier** = The gateway that routes external requests to those apps

Without Kourier (or an alternative like Istio), your services would run but be unreachable! We chose Kourier because it's lightweight, fast to install, and production-ready without the overhead of a full service mesh.

### Step 1: Install Knative Serving
Apply the Custom Resource Definitions (CRDs) and core components:
```
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.0/serving-crds.yaml
kubectl apply -f https://github.com/knative/serving/releases/download/knative-v1.11.0/serving-core.yaml
```

### Step 2: Install Networking (Kourier)
```
kubectl apply -f https://github.com/knative/net-kourier/releases/download/knative-v1.11.0/kourier.yaml
```

### Step 3: Configure DNS
Tell Knative to use Kourier and configure the magic DNS (sslip.io):
```
kubectl patch configmap/config-network \
  --namespace knative-serving \
  --type merge \
  --patch '{"data":{"ingress-class":"kourier.ingress.networking.knative.dev"}}'
```

---

## Your First Service

Let's deploy a simple "Hello World" app using the `kn` CLI.

```
# Create a namespace
kubectl create namespace hello-world

# Deploy the service
kn service create hello \
  --image gcr.io/knative-samples/helloworld-go \
  --port 8080 \
  --namespace hello-world
```

**Test it out:**
```bash
# Get the Kourier LoadBalancer IP
IP=$(kubectl get svc kourier -n kourier-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Test the service (replace 'hello-world' with your namespace)
curl -H "Host: hello.hello-world.example.com" http://$IP

# Output: Hello Knative!
```

> **Magic Trick:** Wait 90 seconds without sending requests, and watch the pods terminate. Send a request again, and watch it spin up instantly! 

---

##  Advanced: Traffic Splitting

Knative makes "Canary Deployments" incredibly simple. Here is how to split traffic 90/10 between an old version and a new one.

1.  **Update your service** (this creates a new *Revision*).
2.  **Split the traffic:**

```
kn service update hello \
  --traffic @latest=10 \
  --traffic hello-00001=90 \
  --namespace hello-world
```
Now, 10% of users see the new feature, and 90% stay safe on the stable version.

---

## 🚑 Troubleshooting Cheatsheet

Stuck? Here are the commands I use to debug:

| Action | Command |
| :--- | :--- |
| **Check Service Status** | `kn service describe <name> -n <ns>` |
| **List Revisions** | `kn revision list -n <ns>` |
| **View Pod Logs** | `kubectl logs -l serving.knative.dev/service=<name>` |
| **System Logs** | `kubectl logs -l app=controller -n knative-serving` |


## 📚 Resources & Credits

*   **Original Guide:** [The Complete Guide to Knative by Salwan Mohamed](https://www.linkedin.com/in/salwan-mohamed/)
*   **Official Docs:** [Knative Documentation](https://knative.dev/docs/)

