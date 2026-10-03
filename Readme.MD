# eks-data-platform

A batch data platform on Amazon EKS that ingests public transport data (Translink SEQ GTFS), transforms it with Spark, and lands it as Iceberg tables on S3 that can be queried with Athena.

The project applies production platform practices to data workloads: infrastructure as code, GitOps delivery, spot capacity for compute, disruption budgets, observability, and a one-command build and teardown.

> **Status:** Stage 0, scaffolding. Everything below the roadmap is planned, not built yet.

## Why this project

Most data platform examples stop at "it runs". This one treats the platform the way a production Kubernetes fleet is treated:

- Rebuilt from code at any time, with no manual steps
- Delivered through GitOps (ArgoCD app-of-apps)
- Cost-aware, using Karpenter spot nodes for Spark executors and full teardown between sessions
- Safe under disruption, using PodDisruptionBudgets, topology spread and priority classes
- Observable, with metrics for DAG runs and Spark jobs

## Architecture (planned)

```
Translink GTFS feed
        │
        ▼
  Airflow DAG (KubernetesExecutor)
        │  download
        ▼
  S3 raw bucket (zip / CSV)
        │
        ▼
  Spark job (Spark Operator, spot executors)
        │  clean, type, partition
        ▼
  S3 curated: Iceberg tables (Glue catalog)
        │
        ▼
  Athena queries · data-quality checks · Grafana dashboards
```

A full diagram will go in `docs/architecture.md`.

## Stack

| Layer | Tool |
|---|---|
| Infrastructure | Terraform (VPC, EKS, S3, IAM) |
| Delivery | ArgoCD app-of-apps |
| Compute | Karpenter: on-demand for system pods, spot for Spark executors |
| Orchestration | Apache Airflow (official Helm chart, KubernetesExecutor, git-sync) |
| Processing | Apache Spark via the Spark Operator |
| Table format | Apache Iceberg with the AWS Glue catalog |
| Query | Amazon Athena |
| Identity | EKS Pod Identity or IRSA |
| Observability | Prometheus and Grafana |
| Local exploration | DuckDB, PySpark |

## Roadmap

| Stage | Goal | Done when |
|---|---|---|
| 0. Scaffolding | Repo, README, scope | This README is published |
| 1. Local exploration | Understand the data before building | GTFS loaded into DuckDB, 10 queries written, PySpark job runs locally |
| 2. Cluster from code | EKS from Terraform, ArgoCD bootstrapped | `up.sh` builds the cluster and `down.sh` destroys it cleanly; budget alarm set |
| 3. Nodes, storage, identity | Karpenter NodePools, S3 buckets, pod IAM | Rebuild from zero with one command |
| 4. Ingestion | Airflow on EKS | A DAG downloads GTFS into S3 raw |
| 5. Transform | Spark on EKS | Airflow triggers a Spark job that writes Parquet to curated |
| 6. Lakehouse and quality | Iceberg tables, Athena, checks | Athena queries Iceberg tables; row-count and null checks run as a DAG task |
| 7. Production-shaped | Resilience and observability | PDBs, topology spread, priority classes, Grafana dashboards for DAGs and Spark |
| 8. Ship | Cost and write-up | Cost per run documented, architecture diagram, decisions log complete |

## Repository layout (planned)

```
eks-data-platform/
├── README.md
├── docs/
│   ├── architecture.md      # diagram and data flow
│   ├── design.md            # scope, decisions and trade-offs
│   └── cost.md              # cost per run, spot savings
├── terraform/
│   ├── modules/             # vpc, eks, karpenter, s3, iam
│   └── envs/sandbox/
├── gitops/
│   ├── bootstrap/           # app-of-apps root
│   └── apps/                # airflow, spark-operator, nodepools, monitoring
├── dags/                    # Airflow DAGs (synced via git-sync)
├── spark/jobs/              # PySpark jobs
├── local/                   # DuckDB queries and notebooks
└── scripts/
    ├── up.sh
    └── down.sh
```

## Data source

[Translink SEQ GTFS static feed](https://translink.com.au/about-translink/open-data): routes, stops, trips and timetables for South East Queensland public transport. Used under Translink's open data licence.

## Cost control

- Sandbox cluster in a personal AWS account, with an AWS Budgets alarm
- Spot instances for Spark executors
- The cluster is destroyed at the end of every working session; S3 data persists

## Decisions and trade-offs

Recorded in `docs/design.md` as the project progresses.
