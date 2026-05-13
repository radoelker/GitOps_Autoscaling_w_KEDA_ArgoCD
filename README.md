# KEDA + ArgoCD: GitOps Autoscaling with Real Alerts

### The situation

Every night, warehouse partners batch-upload thousands of shipment records. The records land in an SQS queue. A Kubernetes worker service reads from that queue and writes to the database.

The problem: there's only ever one worker pod. Last Tuesday at 2 AM, a warehouse uploaded 50,000 records. The queue grew for hours. The single worker couldn't keep up. By 6 AM when the support team arrived, customers were calling asking where their packages were. The backlog took until noon to clear.

### What we planed

We plan to use an EDA (event driven archticture) with KEDA for the right scaling and GitOPS (ArgoCD)  to avoid human intervention (and likely error) at night hours. Additionally, scaling events shall be reported.

1. **Auto-scaling based on queue depth** — not CPU (the worker barely uses any CPU; it just waits for I/O). KEDA makes this possible.
2. **All changes go through Git** — no more `kubectl apply` from laptops, no more "who changed this config at 3 AM?" ArgoCD enforces this.
3. **Real alerts when scaling happens** — the on-call engineer should get an email or SMS, not find out from customers.

## Architecture

```
┌─────────────┐     uploads messages     ┌──────────────────┐
│  Warehouse  │ ──────────────────────►  │  AWS SQS Queue   │
│  Simulator  │                          │  (shipments)     │
└─────────────┘                          └────────┬─────────┘
                                                  │ queue depth metric
                                                  ▼
                                          ┌───────────────┐
                                          │     KEDA      │
                                          │  ScaledObject │
                                          └──────┬────────┘
                                                 │ drives replica count
                                                 ▼
                                      ┌─────────────────────┐
                                      │  Worker Deployment  │
                                      │  (0 → N pods)       │
                                      └──────────┬──────────┘
                                                 │ pods consume queue
                                     ┌───────────┼───────────┐
                                     ▼           ▼           ▼
                                  Worker 1   Worker 2   Worker N

                                     when scaling happens:
                                                 │
                                                 ▼
                                     ┌───────────────────────┐
                                     │  Event Exporter       │
                                     │  (watches k8s events) │
                                     └──────────┬────────────┘
                                                │
                                                ▼
                                     ┌───────────────────────┐
                                     │  SNS Notifier pod     │
                                     │  (HTTP → SNS publish) │
                                     └──────────┬────────────┘
                                                │
                               ┌────────────────┴────────────────┐
                               ▼                                 ▼
                          Email                              SMS
                       (your inbox)                      (your phone)
```

**GitOps flow — how changes reach the cluster:**

```
  Your laptop          GitHub               ArgoCD              EKS cluster
       │                  │                   │                     │
       ├── git push ───►  │                   │                     │
       │                  ├── webhook ──────► │                     │
       │                  │                   ├── sync ───────────► │
       │                  │                   │    (kubectl apply)   │
```

You never `kubectl apply` directly. Git is the only path to production.

#### What the worker really does

But the worker is different. Neither consuming lots of CPU nor memory. Look at what it actually does:

```
Pick up a message from SQS
  ↓
Read some data from the database
  ↓
Transform the data (milliseconds of work)
  ↓
Write to another database table
  ↓
Delete the message from SQS
  ↓
Repeat
```

Almost all of that is waiting — waiting for the network, waiting for the database. The pod sits at 3–5% CPU regardless of how many messages are in the queue.

HPA sees 3% CPU and thinks: "all good, no scaling needed." The queue has 50,000 messages. Customers are waiting.

**CPU is the wrong signal.** Queue depth is the right signal.

### Why KEDA?

**KEDA** (Kubernetes Event Driven Autoscaling) lets you scale on the thing that actually matters: the queue depth itself. KEDA actually creates and manages an HPA under the hood — it just feeds it a custom metric (queue depth) instead of CPU. While HPA has a hard minimum of 1 pod,  KEDA can scale down to 0.

### Why GitOps (ArgoCD)?

GitOps means one thing: Git is the single source of truth for what should be running in the cluster. **ArgoCD** enforces this. If your YAML is in Git, it's deployed. If it's not in Git, it doesn't exist in the cluster.