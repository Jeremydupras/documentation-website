---
layout: default
title: Deploy on Google Kubernetes Engine
nav_order: 3
grand_parent: Migration workflows
parent: Choose your deployment
permalink: /migration-assistant/migration-phases/deploy/deploying-to-gke/
---

# Deploy on Google Kubernetes Engine

On Google Cloud, Migration Assistant provides a Terraform module that provisions a Google Kubernetes Engine (GKE) cluster and the supporting Google Cloud resources, then installs the same Migration Assistant Helm chart and workflow engine used on every other platform. This is the recommended path on Google Cloud because it prepares the surrounding environment---networking, Workload Identity, a Cloud Storage snapshot bucket, and Cloud Logging---rather than leaving it to you.

The migration itself runs exactly as it does elsewhere. Only the platform setup differs.

## Prerequisites

Before you begin, make sure you have the following:

- [Terraform](https://developer.hashicorp.com/terraform/install) or [OpenTofu](https://opentofu.org/docs/intro/install/) 1.6 or later.
- The [gcloud CLI](https://cloud.google.com/sdk/docs/install) authenticated with `gcloud auth application-default login`.
- A Google Cloud project with billing enabled.
- The required APIs enabled:

  ```bash
  gcloud services enable container.googleapis.com storage.googleapis.com
  ```
  {% include copy.html %}

## Step 1: Make the container images available

The Migration Assistant images (migration console, reindex-from-snapshot, and---for live capture and replay---the capture proxy and traffic replayer) must be available in a registry your GKE cluster can pull from, such as Artifact Registry in the same project. Build and push them from the repository root:

```bash
./gradlew buildImagesToRegistry -PregistryEndpoint=REGION-docker.pkg.dev/PROJECT
```
{% include copy.html %}

The Terraform module sets a fully qualified image `repository` for each image at deploy time, so pods pull from your registry rather than defaulting to Docker Hub. If you install the chart directly instead of through the module, pass the equivalent `--set images.<name>.repository=...` overrides for each image.
{: .note }

## Step 2: Provision the infrastructure

From `deployment/terraform/gcp`, plan and apply the module:

```bash
terraform init
terraform plan -var="project=my-project" -var="region=us-central1"
terraform apply -var="project=my-project" -var="region=us-central1"
```
{% include copy.html %}

The module creates a GKE Standard cluster with private nodes and Cloud NAT for outbound access, a VPC (unless you supply an existing one), Workload Identity bindings for the Migration Assistant service accounts, and a generated Cloud Storage bucket for snapshots. The cluster and bucket names are generated; retrieve them with `terraform output`.

The following table describes the most commonly used variables. Run `terraform plan` to see the full set.

| Variable | Default | Description |
|:---------|:--------|:------------|
| `project` | *(required)* | Google Cloud project ID |
| `region` | `us-central1` | Google Cloud region |
| `migration_release` | Chart default | Migration Assistant release tag applied to all images; must exist in your registry |
| `node_machine_type` | `e2-standard-4` | Node machine type |
| `node_disk_size` | `50` | Node boot disk size, in GB |
| `create_vpc` | `true` | Create a new VPC, or set to `false` to use `existing_vpc_name` and `existing_subnet_name` |
| `max_zones` | `2` | Maximum zones for regional node placement. Set to `1` for single-zone (higher throughput, lower cost) |
| `source_connectivity` / `target_connectivity` | `{mode = "none"}` | Private connectivity for source/target traffic: `none` (public), `psc_consumer` (Private Service Connect), or `vpc_peering` |
| `gcs_connectivity` | `{mode = "private_google_access"}` | Private Google Access for Cloud Storage snapshot traffic; set to `none` for the public path |
| `enable_private_endpoint` | `false` | Restrict the GKE control plane to a private IP only. Requires VPN or a bastion for `kubectl` access |

## Step 3: Get cluster credentials and install the chart

```bash
gcloud container clusters get-credentials $(terraform output -raw cluster_name) \
  --region $(terraform output -raw cluster_location) --project my-project

helm install migration-assistant \
  ../../k8s/charts/aggregates/migrationAssistantWithArgo \
  --values ../../k8s/charts/aggregates/migrationAssistantWithArgo/valuesGke.yaml \
  --set gcp.project=my-project
```
{% include copy.html %}

The `valuesGke.yaml` overlay sets `cloudProvider: gcp` and wires up the GCP-specific integrations (Cloud Logging and the Cloud Storage artifact repository). Supply your project with `--set gcp.project`.

## Step 4: Verify the deployment

```bash
kubectl get pods -n ma
```
{% include copy.html %}

You should see the Migration Console, the Argo workflow controller, and the Argo server in the `Running` state. Access the console the same way as on any other platform:

```bash
kubectl exec -it migration-console-0 -n ma -- /bin/bash
```
{% include copy.html %}

From here, the migration flow is identical to every other deployment: verify the version, load the sample configuration, run a pilot, validate it, then run the full migration. See [Backfill]({{site.url}}{{site.baseurl}}/migration-assistant/migration-phases/backfill/).

## Snapshots on Cloud Storage

The module provisions a Cloud Storage bucket for snapshots and grants the node service account access to it. Migration Assistant reads and writes snapshots there using a `gs://` repository URI; the URI scheme selects the Cloud Storage backend automatically. For the snapshot repository configuration and the source-cluster `repository-gcs` plugin requirement, see [Snapshot repository backend]({{site.url}}{{site.baseurl}}/migration-assistant/migration-phases/backfill/#snapshot-repository-backend-amazon-s3-or-google-cloud-storage).

## Observability

### Logs

Workload logs (migration console, capture proxy, traffic replayer, RFS workers, and Argo workflow steps) are shipped to **Google Cloud Logging** by the bundled fluent-bit collector, which authenticates through Workload Identity. No extra setup is required. View them in the Cloud Logging console or with `gcloud`:

```bash
gcloud logging read \
  'resource.type="k8s_container" resource.labels.namespace_name="ma"' \
  --project my-project --limit 50
```
{% include copy.html %}

Filter by `resource.labels.container_name` to narrow to a single workload, or by `severity>=WARNING` to surface problems.

### Metrics and dashboards

The chart deploys Prometheus by default and the workloads export metrics to it. A pre-built migration dashboard ships with the chart, but the in-cluster Grafana that renders it is **off by default**, because many operators already run their own dashboarding stack. To enable the bundled Grafana with the dashboard preloaded:

```yaml
conditionalPackageInstalls:
  grafana: true
```

## Private networking

To run a migration with no public-internet data path---private source and target connectivity, private Cloud Storage access, and a private control plane---set the `source_connectivity`, `target_connectivity`, and `gcs_connectivity` variables and `enable_private_endpoint = true`. When the control plane is private, run `kubectl` and `terraform apply` (for `deploy_helm`) from inside the VPC or a network routed to it.

## Sizing: shard size and node disk

RFS workers request ephemeral storage proportional to `maxShardSizeBytes` (the upper bound on a single shard's materialized size), using `ceil(2.5 * maxShardSizeBytes)`. The default `maxShardSizeBytes` of 80 GiB produces a 200 GiB ephemeral-storage request per worker, which does not fit on the default `e2-standard-4` node with a 50 GB boot disk---the RFS pod stays `Pending` with `Insufficient ephemeral-storage`.

For production, keep the 80 GiB default and size nodes accordingly (larger `node_disk_size` and, if needed, `node_machine_type`). For small, development, or test clusters, lower `maxShardSizeBytes` in the workflow configuration to match the largest shard you actually expect.

{: .warning }
> Do not set `maxShardSizeBytes` below your largest real shard. If a shard exceeds the configured ceiling, the migration fails mid-shard.

## Remove the deployment

```bash
terraform destroy -var="project=my-project"
```
{% include copy.html %}

{% include migration-phase-navigation.html %}
