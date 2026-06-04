# Genomics on Google Cloud: Standardized Operations Guide

**Version:** 1.0  
**Status:** Active  
**Project ID:** `project-62f718c3-2ad1-44d6-acc`  
**Scope:** Genomic data discovery, compute provisioning, environment setup, storage management, and cost-controlled shutdown

---

## Table of Contents

1. [Prerequisites and Scope](#1-prerequisites-and-scope)
2. [Infrastructure Setup: APIs and IAM Roles](#2-infrastructure-setup-apis-and-iam-roles)
3. [BigQuery: Public Genomic Dataset Access](#3-bigquery-public-genomic-dataset-access)
4. [Cost-Optimized Compute: Spot VM Provisioning](#4-cost-optimized-compute-spot-vm-provisioning)
5. [Environment Stability: Software Installation](#5-environment-stability-software-installation)
6. [Scalable Storage: Cloud Storage Best Practices](#6-scalable-storage-cloud-storage-best-practices)
7. [Zero-Waste Shutdown Checklist](#7-zero-waste-shutdown-checklist)
8. [Quick Reference: Common Error Fixes](#8-quick-reference-common-error-fixes)

---

## 1. Prerequisites and Scope

This SOP covers the full workflow for genomic research on Google Cloud Platform (GCP), from initial project configuration through data processing and controlled shutdown. It is intended for researchers transitioning from local HPC environments.

**Assumed starting state:**

- A GCP project exists and you have Owner-level access
- The `gcloud` CLI is installed and authenticated locally
- You have identified your target datasets (BigQuery or Cloud Storage)

**Not covered by this SOP:** Workflow orchestration (Nextflow, Snakemake), Terra/Cromwell, or multi-VM cluster provisioning.

---

## 2. Infrastructure Setup: APIs and IAM Roles

### 2.1 Enable Required APIs

Being a project Owner does **not** automatically activate all APIs. Each API must be explicitly enabled. Failure to do so results in cryptic `403` or `METHOD_NOT_FOUND` errors.

Enable all required APIs in one command:

```bash
gcloud services enable \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  compute.googleapis.com \
  storage.googleapis.com \
  bigquery.googleapis.com \
  --project=project-62f718c3-2ad1-44d6-acc
```

Verify enabled services:

```bash
gcloud services list --enabled --project=project-62f718c3-2ad1-44d6-acc
```

> **Lesson learned:** The Cloud Resource Manager API and IAM API must be enabled first, before any other resource management commands will succeed. Enable these two before running any `gcloud iam` or `gcloud compute` commands.

### 2.2 Required IAM Roles for Genomic Researchers

Assign the following roles to each researcher's Google account. Roles are cumulative; all are needed for standard workflows.

| Role | Purpose | Grant Command |
|------|---------|---------------|
| `roles/compute.instanceAdmin.v1` | Create, start, stop, and SSH into VMs | See below |
| `roles/compute.osAdminLogin` | Use `sudo` inside VMs via OS Login | Required; Owner alone is insufficient |
| `roles/storage.objectViewer` | Read from Cloud Storage buckets | Applied to bucket or project level |
| `roles/storage.objectCreator` | Write results back to Cloud Storage | Applied to output bucket |
| `roles/bigquery.dataViewer` | Query BigQuery public datasets | Project-level grant |
| `roles/bigquery.jobUser` | Execute BigQuery jobs | Project-level grant |
| `roles/iam.serviceAccountUser` | Attach service accounts to VMs | Required for VM creation with service accounts |

Grant roles to a researcher:

```bash
# Replace USER_EMAIL with the researcher's Google account
USER_EMAIL="researcher@institution.edu"
PROJECT="project-62f718c3-2ad1-44d6-acc"

gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER_EMAIL" \
  --role="roles/compute.instanceAdmin.v1"

gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER_EMAIL" \
  --role="roles/compute.osAdminLogin"

gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER_EMAIL" \
  --role="roles/bigquery.dataViewer"

gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER_EMAIL" \
  --role="roles/bigquery.jobUser"
```

> **Critical note on `sudo` access:** The `roles/compute.osAdminLogin` role is mandatory for `sudo` access on VMs using OS Login. The project Owner role alone does not grant this. Without it, software installation will fail silently or with permission errors.

### 2.3 VM Service Account Roles

The VM's default service account (used for Cloud Storage and API access from inside the VM) also needs explicit permissions:

```bash
# Get the default compute service account email
SA_EMAIL=$(gcloud iam service-accounts list \
  --filter="displayName:Compute Engine default service account" \
  --format="value(email)" \
  --project=project-62f718c3-2ad1-44d6-acc)

# Grant Storage access
gcloud projects add-iam-policy-binding project-62f718c3-2ad1-44d6-acc \
  --member="serviceAccount:$SA_EMAIL" \
  --role="roles/storage.objectViewer"
```

---

## 3. BigQuery: Public Genomic Dataset Access

### 3.1 Schema Inspection Before Querying

Column names in public genomic BigQuery datasets vary by dataset version and date snapshot. **Always inspect the schema before writing analysis queries.**

```sql
-- List all columns in a target table before querying
SELECT column_name, data_type
FROM `bigquery-public-data.clinvar_hg38.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'clinvar_variants'
ORDER BY ordinal_position;
```

**Known column name variation (ClinVar example):**

| Dataset version | Significance column name |
|----------------|--------------------------|
| `ncbi_clinvar_hg38_20180701` | `CLNSIG` |
| Newer snapshots | `clinical_significance` |

Always confirm the exact column name with `INFORMATION_SCHEMA` before running any query that filters on clinical significance, consequence, or similar fields.

### 3.2 Standard ClinVar Query Template

```sql
-- Template: adapt column names after schema inspection
SELECT
  name,
  CLNSIG,          -- or clinical_significance depending on version
  CLNDN,
  chromosome,
  start_position,
  reference_bases,
  alternate_bases
FROM `bigquery-public-data.ncbi_clinvar_hg38_20180701.clinvar_variants`
WHERE CLNSIG LIKE '%Pathogenic%'
LIMIT 1000;
```

### 3.3 Cost Controls for BigQuery

- Use `SELECT` with `LIMIT` during exploratory queries to avoid full table scans.
- Check estimated query cost in the BigQuery UI (top right) before running large queries.
- Partition large result tables by chromosome or date to reduce downstream query costs.

---

## 4. Cost-Optimized Compute: Spot VM Provisioning

Spot VMs (formerly Preemptible VMs) cost approximately 60–90% less than standard VMs and are appropriate for interruptible genomic processing jobs.

### 4.1 VM Configuration Specifications

| Parameter | Recommended Value | Notes |
|-----------|-----------------|-------|
| Machine type | `e2-standard-2` | 2 vCPUs, 8GB RAM; scale up for assembly |
| OS Image | Ubuntu 22.04 LTS | Stable; good `apt` package support |
| VM provisioning model | Spot | Check "Spot" under Availability Policy |
| Boot disk size | 50–100 GB | Increase for large reference genomes |
| Cloud API access scopes | Allow full access to all Cloud APIs | **Critical for Storage; see Section 4.3** |

### 4.2 Create Spot VM via CLI

```bash
gcloud compute instances create genomics-spot-vm \
  --project=project-62f718c3-2ad1-44d6-acc \
  --zone=us-central1-a \
  --machine-type=e2-standard-2 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --boot-disk-size=100GB \
  --scopes=cloud-platform
```

> The `--scopes=cloud-platform` flag sets Cloud API access to "Full Access" and is the CLI equivalent of selecting "Allow full access to all Cloud APIs" in the console. Do not omit this flag.

### 4.3 Configuring Cloud API Access Scopes (Console)

When creating a VM via the GCP Console, navigate to **"OS and Storage Access"** (shown under "Advanced options" in newer console layouts):

1. Under **Identity and API access**, select the default service account.
2. Under **Access scopes**, select **"Allow full access to all Cloud APIs"**.
3. Do **not** use "Allow default access" — this restricts Storage API scope and causes `403 Access Denied` errors even on public buckets.

> **Root cause of 403 errors on public buckets:** Access scope restrictions on the VM's service account silently block `gsutil` and `gcloud storage` calls, even when the bucket is public and the service account has the correct IAM role. Both conditions (correct IAM role **and** full API scope) must be satisfied.

### 4.4 SSH into the VM

```bash
gcloud compute ssh genomics-spot-vm \
  --zone=us-central1-a \
  --project=project-62f718c3-2ad1-44d6-acc
```

---

## 5. Environment Stability: Software Installation

### 5.1 Miniconda Setup

Install Miniconda for Python and conda-based bioinformatics tools:

```bash
# Download and install Miniconda
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
eval "$($HOME/miniconda3/bin/conda shell.bash hook)"
conda init bash && source ~/.bashrc
```

### 5.2 Conda Channel Configuration

Channel ordering is critical. Incorrect channel priority causes dependency conflicts. Always configure channels explicitly before installing any packages:

```bash
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

Verify configuration:

```bash
conda config --show channels
# Expected order (highest to lowest priority):
# - conda-forge
# - bioconda
# - defaults
```

### 5.3 Installing Bioinformatics Tools

```bash
# Create an isolated environment for genomics tools
conda create -n genomics python=3.10 -y
conda activate genomics

# Install samtools via conda
conda install -c bioconda samtools -y
```

### 5.4 System-Native Fallback Strategy for Dependency Hell

**Problem:** Conda-installed `bcftools` may fail with shared library errors such as:

```
bcftools: error while loading shared libraries: libgsl.so.25: cannot open shared object file
```

This occurs because the GSL (GNU Scientific Library) version bundled in the Conda environment does not match the version expected by the `bcftools` binary. Re-solving the environment or forcing reinstalls often does not resolve this.

**Solution: Bypass Conda entirely and use the system package manager.**

```bash
# Remove the broken conda package
conda remove bcftools -y

# Install system-native bcftools via apt
sudo apt-get update
sudo apt-get install bcftools -y

# Verify installation and note the system binary path
which bcftools          # returns /usr/bin/bcftools
bcftools --version
```

**Call the system binary directly in scripts** to avoid PATH conflicts with the Conda environment:

```bash
# In analysis scripts, use the full path
/usr/bin/bcftools view input.vcf.gz -o output.vcf.gz -O z

# Or call explicitly when the conda environment is active
$(which bcftools) view input.vcf.gz
```

> **General rule:** If a Conda-installed tool produces shared library (`libXXX.so`) errors that persist after channel reconfiguration, switch to `apt-get install`. System-native packages link against system libraries and are generally more stable on Ubuntu LTS for core bioinformatics tools (`samtools`, `bcftools`, `bwa`, `tabix`).

### 5.5 Verify Tool Installation

```bash
samtools --version    # expect samtools 1.x
/usr/bin/bcftools --version   # expect bcftools 1.x
```

---

## 6. Scalable Storage: Cloud Storage Best Practices

### 6.1 Bucket Creation and Structure

```bash
# Create a regional bucket (lower cost, adequate for single-region compute)
gsutil mb -c STANDARD -l us-central1 \
  gs://project-62f718c3-genomics-raw/

gsutil mb -c STANDARD -l us-central1 \
  gs://project-62f718c3-genomics-results/
```

Recommended bucket structure:

```
gs://project-62f718c3-genomics-raw/
  raw_sequences/           # FASTQ, CRAM: subject to archive lifecycle
  reference_genomes/       # hg38, reference FASTAs
  vcf/                     # VCF and BCF files

gs://project-62f718c3-genomics-results/
  processed/               # Filtered VCFs, annotated outputs
  qc_reports/              # MultiQC, FASTQC outputs
  logs/                    # Run logs, metrics
```

### 6.2 Streaming Large Files Directly from Cloud Storage

Avoid staging large files to the VM's local disk when possible. Process streams directly:

```bash
# Stream a VCF directly into bcftools without local copy
gsutil cat gs://your-bucket/large_file.vcf.gz | \
  /usr/bin/bcftools view -f PASS - -O z -o filtered_output.vcf.gz

# Expected throughput for e2-standard-2: ~240 MiB/s for 5+ GB files
```

Write results back to Cloud Storage immediately:

```bash
gsutil -m cp filtered_output.vcf.gz \
  gs://project-62f718c3-genomics-results/processed/
```

### 6.3 Handling Requester Pays Buckets

Some public genomic datasets (e.g., 1000 Genomes, GTEx, dbGaP-authorized buckets) use Requester Pays, meaning your project is billed for egress. Access is blocked without specifying a billing project.

```bash
# Always specify -u PROJECT_ID for Requester Pays buckets
gsutil -u project-62f718c3-2ad1-44d6-acc \
  cp gs://requester-pays-bucket/file.cram ./

# Check if a bucket uses Requester Pays
gsutil requesterpays get gs://bucket-name
```

> **Symptom of missing `-u` flag:** `AccessDeniedException: 403 USER_PROJECT_MISSING`. This is not a permissions error — it is a billing project requirement. Add `-u PROJECT_ID` to resolve.

### 6.4 Lifecycle Archive Rules for Raw Sequence Data

Raw FASTQ and CRAM files should be automatically transitioned to Nearline or Coldline storage after a defined period to reduce costs.

Create a lifecycle policy file (`lifecycle.json`):

```json
{
  "rule": [
    {
      "action": {
        "type": "SetStorageClass",
        "storageClass": "NEARLINE"
      },
      "condition": {
        "age": 30,
        "matchesPrefix": ["raw_sequences/"]
      }
    },
    {
      "action": {
        "type": "SetStorageClass",
        "storageClass": "COLDLINE"
      },
      "condition": {
        "age": 90,
        "matchesPrefix": ["raw_sequences/"]
      }
    }
  ]
}
```

Apply the policy:

```bash
gsutil lifecycle set lifecycle.json \
  gs://project-62f718c3-genomics-raw/
```

Verify the policy:

```bash
gsutil lifecycle get gs://project-62f718c3-genomics-raw/
```

**Storage class cost reference (us-central1, approximate):**

| Class | Monthly cost/GB | Min storage duration | Best for |
|-------|----------------|---------------------|---------|
| Standard | $0.020 | None | Active processing |
| Nearline | $0.010 | 30 days | Monthly access |
| Coldline | $0.004 | 90 days | Quarterly access |
| Archive | $0.0012 | 365 days | Long-term retention |

---

## 7. Zero-Waste Shutdown Checklist

Run this checklist at the end of every compute session to prevent unexpected billing.

### 7.1 Data Preservation

- [ ] Copy all results and processed files to Cloud Storage:
  ```bash
  gsutil -m cp -r ./results/ gs://project-62f718c3-genomics-results/processed/
  ```
- [ ] Copy logs and QC reports:
  ```bash
  gsutil -m cp ./logs/*.log gs://project-62f718c3-genomics-results/logs/
  ```
- [ ] Verify uploads completed:
  ```bash
  gsutil ls -l gs://project-62f718c3-genomics-results/processed/
  ```
- [ ] Confirm no critical intermediate files remain only on the VM's local disk.

### 7.2 Compress and Archive Outputs

- [ ] Index any VCF or BAM outputs before upload:
  ```bash
  /usr/bin/bcftools index output.vcf.gz
  samtools index output.bam
  ```
- [ ] Compress uncompressed VCFs:
  ```bash
  bgzip -@ 2 output.vcf && tabix output.vcf.gz
  ```

### 7.3 Stop the VM

Stop (do not delete) the VM to preserve the disk while stopping compute billing:

```bash
gcloud compute instances stop genomics-spot-vm \
  --zone=us-central1-a \
  --project=project-62f718c3-2ad1-44d6-acc
```

> Stopping a VM stops CPU/memory billing but **disk storage billing continues**. If the VM is no longer needed, delete it. If the software environment should be reused, stop it.

### 7.4 Delete the VM (if work is complete)

```bash
gcloud compute instances delete genomics-spot-vm \
  --zone=us-central1-a \
  --project=project-62f718c3-2ad1-44d6-acc
```

### 7.5 Billing Verification

- [ ] Go to **GCP Console > Billing > Reports**.
- [ ] Filter by the current project and confirm compute charges have stopped.
- [ ] Set a **Budget Alert** at $50 and $100 to catch accidental VM restarts:
  ```bash
  # Budget alerts are configured via Console: Billing > Budgets & alerts
  ```

---

## 8. Quick Reference: Common Error Fixes

| Error | Root cause | Fix |
|-------|-----------|-----|
| `403 Access Denied` on public bucket | VM API scope too restrictive | Set Cloud API Access Scopes to "Full Access" on VM creation |
| `403 USER_PROJECT_MISSING` | Requester Pays bucket, no billing project set | Add `-u PROJECT_ID` to all `gsutil` commands |
| `sudo: permission denied` via SSH | Missing `compute.osAdminLogin` role | Grant `roles/compute.osAdminLogin` to user |
| `libgsl.so.25: cannot open shared object file` | Conda environment shared library mismatch | Remove conda package; use `sudo apt-get install bcftools` |
| `METHOD_NOT_FOUND` or `API not enabled` | Required GCP API is disabled | Run `gcloud services enable` for the relevant API |
| `INFORMATION_SCHEMA` column not found | Column name differs across dataset versions | Query `INFORMATION_SCHEMA.COLUMNS` to confirm actual column names |
| VM terminated unexpectedly | Spot VM preemption | Expected behavior; resubmit job; consider `--max-run-duration` flag for short jobs |
| `gcloud: command not found` in conda env | Conda PATH shadowing system tools | Call with full path: `/usr/lib/google-cloud-sdk/bin/gcloud` |

---

## Appendix: Useful Commands Reference

```bash
# Check VM status
gcloud compute instances list --project=project-62f718c3-2ad1-44d6-acc

# Check current IAM policy
gcloud projects get-iam-policy project-62f718c3-2ad1-44d6-acc

# Monitor storage bucket usage
gsutil du -sh gs://project-62f718c3-genomics-raw/

# Check enabled APIs
gcloud services list --enabled --project=project-62f718c3-2ad1-44d6-acc

# SSH into VM
gcloud compute ssh genomics-spot-vm --zone=us-central1-a \
  --project=project-62f718c3-2ad1-44d6-acc

# Copy file from VM to bucket
gsutil cp /home/username/output.vcf.gz \
  gs://project-62f718c3-genomics-results/processed/

# Stream large file through bcftools filter
gsutil cat gs://bucket/file.vcf.gz | \
  /usr/bin/bcftools view -f PASS - -O z -o filtered.vcf.gz
```

---

*This document should be reviewed and updated after each new GCP configuration change or when onboarding researchers to new dataset sources. Store the latest version in the project's shared Google Drive or internal wiki.*
