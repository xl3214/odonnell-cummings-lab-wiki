# Genomics Cloud Bootcamp
## Document 1: The New Researcher's Step-by-Step Lab Guide

**Audience:** New team members with no prior Google Cloud experience  
**Duration:** ~3 hours (end-to-end)  
**Goal:** Complete one full genomics workflow — from cloud discovery, to compute, to safe storage — on real data

---

## Table of Contents

1. [Before You Start: Quick-Start Checklist](#1-before-you-start-quick-start-checklist)
2. [Phase 1: BigQuery — Explore the Data](#2-phase-1-bigquery--explore-the-data)
3. [Phase 2: Virtual Workbench — Build Your VM](#3-phase-2-virtual-workbench--build-your-vm)
4. [Phase 2 Continued: Install Tools and Process Data](#4-phase-2-continued-install-tools-and-process-data)
5. [Phase 3: Storage — Save Your Results and Clean Up](#5-phase-3-storage--save-your-results-and-clean-up)

---

## 1. Before You Start: Quick-Start Checklist

Complete every item on this checklist before moving on. Skipping any step will cause failures in later phases.

### 1.1 Enable Required APIs

New GCP projects ship with most APIs disabled. You must enable them manually.

Go to **GCP Console → APIs & Services → Library** and enable each of the following, or run the block below in Cloud Shell:

```bash
gcloud services enable \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  compute.googleapis.com \
  storage.googleapis.com \
  bigquery.googleapis.com \
  --project=YOUR_PROJECT_ID
```

Replace `YOUR_PROJECT_ID` with your actual project ID (e.g., `project-62f718c3-2ad1-44d6-acc`).

Verify all five are active:

```bash
gcloud services list --enabled --project=YOUR_PROJECT_ID
```

> **Common Error — "The caller does not have permission" or blank UI panels:** This is usually a sign that `cloudresourcemanager.googleapis.com` is not yet enabled. Enable it first, then reload the console.

### 1.2 Grant Your Account the Required IAM Roles

Being a Project Owner gives you billing and policy control, but **not** admin access inside a running VM. You must explicitly grant yourself the following roles.

Run each command in Cloud Shell, replacing `YOUR_EMAIL`:

```bash
PROJECT="YOUR_PROJECT_ID"
USER="YOUR_EMAIL@institution.edu"

# Create and manage Compute Engine VMs
gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER" \
  --role="roles/compute.instanceAdmin.v1"

# Use sudo inside VMs (required; Owner alone is NOT sufficient)
gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER" \
  --role="roles/compute.osAdminLogin"

# Query BigQuery datasets
gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER" \
  --role="roles/bigquery.dataViewer"

# Run BigQuery jobs (without this, queries will be rejected)
gcloud projects add-iam-policy-binding $PROJECT \
  --member="user:$USER" \
  --role="roles/bigquery.jobUser"
```

**Quick-Start Checklist Summary:**

- [ ] `cloudresourcemanager.googleapis.com` enabled
- [ ] `iam.googleapis.com` enabled
- [ ] `compute.googleapis.com` enabled
- [ ] `storage.googleapis.com` enabled
- [ ] `bigquery.googleapis.com` enabled
- [ ] `roles/compute.instanceAdmin.v1` granted to your account
- [ ] `roles/compute.osAdminLogin` granted to your account
- [ ] `roles/bigquery.dataViewer` granted to your account
- [ ] `roles/bigquery.jobUser` granted to your account

---

## 2. Phase 1: BigQuery — Explore the Data

In this phase, you will query a public genomic database to count pathogenic ClinVar variants on chromosome 17 — without downloading a single file.

### 2.1 Open BigQuery

Navigate to **GCP Console → BigQuery**. Your project should appear in the left panel under "Explorer."

### 2.2 Inspect the Schema First

> **Critical step:** Column names in public datasets vary by version. Never assume a column name — always check the schema before writing a query.

Open a new query tab and run:

```sql
SELECT column_name, data_type
FROM `bigquery-public-data.ncbi_clinvar_hg38_20180701.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'clinvar_variants'
ORDER BY ordinal_position;
```

Note the exact name of the clinical significance column. In this dataset it is **`CLNSIG`** (not `clinical_significance`, which appears in newer dataset versions).

### 2.3 Count Pathogenic Variants on Chromosome 17

```sql
SELECT
  chromosome,
  CLNSIG AS clinical_significance,
  COUNT(*) AS variant_count
FROM
  `bigquery-public-data.ncbi_clinvar_hg38_20180701.clinvar_variants`
WHERE
  chromosome = '17'
  AND CLNSIG LIKE '%Pathogenic%'
GROUP BY
  chromosome,
  CLNSIG
ORDER BY
  variant_count DESC;
```

Expected result: a table showing counts of Pathogenic and Likely Pathogenic variants on chromosome 17.

> **Common Error — "Unrecognized name: CLNSIG":** The column does not exist under that name in this version of the table. Run the `INFORMATION_SCHEMA` query above and confirm the exact column name.

> **Common Error — "Access Denied: BigQuery bigquery-public-data":** Your account is missing `roles/bigquery.dataViewer`. Return to Section 1.2 and grant it.

### 2.4 Cost Awareness

Before running any large query, check the estimated byte scanned count shown in the top-right of the BigQuery editor. The first 1 TB/month is free. For exploratory work, always use `LIMIT` to avoid scanning full tables unnecessarily.

---

## 3. Phase 2: Virtual Workbench — Build Your VM

You will create a Spot VM: a deeply discounted (60–90% cheaper) compute instance suitable for interruptible processing jobs.

### 3.1 Create the Spot VM

Run this command in Cloud Shell, or adapt the parameters in the GCP Console under **Compute Engine → Create Instance**:

```bash
gcloud compute instances create genomics-workbench \
  --project=YOUR_PROJECT_ID \
  --zone=us-central1-a \
  --machine-type=e2-standard-2 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --provisioning-model=SPOT \
  --instance-termination-action=STOP \
  --boot-disk-size=100GB \
  --scopes=cloud-platform
```

**Key parameters explained:**

| Flag | Value | Why |
|------|-------|-----|
| `--machine-type` | `e2-standard-2` | 2 vCPUs, 8GB RAM — sufficient for bcftools on a 5GB VCF |
| `--provisioning-model` | `SPOT` | Reduces cost by ~80%; may be preempted with 30s notice |
| `--instance-termination-action` | `STOP` | On preemption, disk is preserved (not deleted) |
| `--scopes` | `cloud-platform` | **Critical:** Grants the VM full Cloud API access, required for Storage |

> **Common Error — "Access Denied" when using gsutil inside the VM:** This means `--scopes=cloud-platform` was not set at VM creation time. API scopes **cannot be changed** on a running VM — you must delete and recreate it with the correct flag.

### 3.2 Grant the VM's Service Account Storage Access

The VM runs as a service account. That service account also needs the Storage role:

```bash
# Get the default compute service account email
SA=$(gcloud iam service-accounts list \
  --filter="displayName:Compute Engine default service account" \
  --format="value(email)" \
  --project=YOUR_PROJECT_ID)

# Grant it read access to Cloud Storage
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:$SA" \
  --role="roles/storage.objectViewer"
```

> **Important:** Both the `--scopes=cloud-platform` flag **and** the `roles/storage.objectViewer` IAM grant are required. Either one alone is not sufficient.

### 3.3 SSH Into the VM

```bash
gcloud compute ssh genomics-workbench \
  --zone=us-central1-a \
  --project=YOUR_PROJECT_ID
```

You are now inside your VM. All subsequent commands in Phase 2 and 4 run here.

---

## 4. Phase 2 Continued: Install Tools and Process Data

### 4.1 Install Miniconda

```bash
# Download Miniconda installer
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh

# Install silently to ~/miniconda3
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3

# Activate for this session
eval "$($HOME/miniconda3/bin/conda shell.bash hook)"

# Persist activation across future SSH sessions
conda init bash && source ~/.bashrc
```

Verify:

```bash
conda --version   # expect conda 23.x or higher
```

### 4.2 Configure Conda Channels

Channel ordering determines which version of a package gets installed. Set this before installing anything:

```bash
conda config --add channels defaults
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

### 4.3 Install samtools via Conda

```bash
conda create -n genomics python=3.10 -y
conda activate genomics
conda install -c bioconda samtools -y
samtools --version
```

### 4.4 Install bcftools — System-Native Method (Required)

> **Do not install bcftools via Conda for this workflow.** Conda's version links against `libgsl.so.25`, which may not be present in the VM's base system libraries. The result is a runtime crash that cannot be fixed by reinstalling through Conda.

Install from the system package manager instead:

```bash
sudo apt-get update
sudo apt-get install bcftools -y
```

Verify the installation and note its path:

```bash
which bcftools        # should return /usr/bin/bcftools
/usr/bin/bcftools --version
```

Always call bcftools using the full path `/usr/bin/bcftools` in scripts to avoid conflicts with any Conda environment that may be active.

### 4.5 Locate and Verify the Target File

> **Common Error — File not found / 404 on download:** Public datasets are reorganized over time. Always verify the file path exists before attempting to download or stream it.

```bash
# List the target directory to confirm the file is present
gcloud storage ls gs://genomics-public-data/references/

# Or use gsutil
gsutil ls gs://genomics-public-data/references/
```

Do not proceed to the next step until you have confirmed the exact path to your VCF file.

### 4.6 Run bcftools stats on the 5GB VCF

Stream the file directly from Cloud Storage without staging it locally:

```bash
gsutil cat gs://YOUR_BUCKET/your_file.vcf.gz | \
  /usr/bin/bcftools stats - > vcf_stats.txt
```

Or if working with a local copy:

```bash
/usr/bin/bcftools stats your_file.vcf.gz > vcf_stats.txt
```

> **Common Confusion — "The command ran but I only see the header":** This is expected behavior. `bcftools stats` writes a header block immediately, then computes summary statistics across the entire file. On a 5GB file, the data rows take several minutes to calculate. **Do not interrupt the process.** Wait until the command prompt returns before checking the output.

Check that the output file has content:

```bash
wc -l vcf_stats.txt       # should be several hundred lines
grep "^SN" vcf_stats.txt  # summary numbers — confirm these are present
```

---

## 5. Phase 3: Storage — Save Your Results and Clean Up

### 5.1 Create a Private Results Bucket

```bash
# Create a regional bucket (lower latency and cost than multi-region for compute jobs)
gsutil mb -c STANDARD -l us-central1 \
  gs://YOUR_PROJECT_ID-genomics-results/
```

### 5.2 Copy Results to Cloud Storage

```bash
# Upload the stats file
gsutil cp vcf_stats.txt \
  gs://YOUR_PROJECT_ID-genomics-results/phase2/

# Verify the upload
gsutil ls -l gs://YOUR_PROJECT_ID-genomics-results/phase2/
```

### 5.3 Stop the VM to End Billing

Stopping the VM immediately ends CPU and memory charges. The disk is preserved so you can restart and resume later.

Run this from Cloud Shell (not from inside the VM):

```bash
gcloud compute instances stop genomics-workbench \
  --zone=us-central1-a \
  --project=YOUR_PROJECT_ID
```

Confirm the instance status is `TERMINATED`:

```bash
gcloud compute instances list --project=YOUR_PROJECT_ID
```

### 5.4 Delete the VM (When Work is Fully Complete)

If you do not need the VM again, delete it to eliminate disk storage charges:

```bash
gcloud compute instances delete genomics-workbench \
  --zone=us-central1-a \
  --project=YOUR_PROJECT_ID
```

### 5.5 End-of-Session Checklist

- [ ] `vcf_stats.txt` uploaded to Cloud Storage and verified
- [ ] No critical files remain only on the VM's local disk
- [ ] VM is stopped or deleted
- [ ] GCP Billing dashboard checked — compute charges no longer accruing
- [ ] Budget alert set at $50 under **Billing → Budgets & alerts**

**Congratulations — you have completed the Genomics Cloud Bootcamp lab.**  
Proceed to Document 2: *Lessons from the Trenches* for deeper explanations of every error you may have encountered.
