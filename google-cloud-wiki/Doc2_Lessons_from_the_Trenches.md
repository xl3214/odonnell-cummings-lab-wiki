# Genomics Cloud Bootcamp
## Document 2: Lessons from the Trenches

**Audience:** New researchers who have completed (or attempted) the Bootcamp lab  
**Purpose:** Explain the *why* behind every failure mode — so you can diagnose problems yourself, not just follow steps

---

## Table of Contents

1. [Why Your Query or Download Might Fail (and How to Fix It)](#1-why-your-query-or-download-might-fail-and-how-to-fix-it)
2. [The Identity vs. Scope Distinction: VM Permissions Explained](#2-the-identity-vs-scope-distinction-vm-permissions-explained)
3. [The Bioinformatics Dependency Guide: Conda vs. Sudo](#3-the-bioinformatics-dependency-guide-conda-vs-sudo)
4. [The Path Trap: Public Data Moves](#4-the-path-trap-public-data-moves)
5. [The Silent Error: When a Command "Works" but Isn't Done](#5-the-silent-error-when-a-command-works-but-isnt-done)
6. [Reference: Error Code Decoder](#6-reference-error-code-decoder)

---

## 1. Why Your Query or Download Might Fail (and How to Fix It)

This section covers the four most common access failures in a cloud genomics workflow, what each one actually means, and the minimum fix required to resolve it.

---

### Failure 1: The API Gap

**Symptom:** A GCP Console UI feature does nothing, returns a blank panel, or shows a cryptic permission error. Cloud Shell commands return `METHOD_NOT_FOUND` or `The caller does not have permission`.

**What's happening:** Google Cloud Platform is a collection of independent services, each governed by its own API. A new project starts with almost all of these disabled. Trying to use a service before enabling its API produces opaque failures — the error often does not say "API not enabled." It says something that looks like a permissions error, which sends you down the wrong troubleshooting path.

The two APIs that must be enabled *first*, before anything else can be configured, are:

- `cloudresourcemanager.googleapis.com` — Required for any project-level resource management, including starring projects in the UI and running `gcloud projects` commands.
- `iam.googleapis.com` — Required for any IAM role assignment. Without this, `gcloud iam` and `gcloud projects add-iam-policy-binding` will fail silently or with misleading errors.

**The fix:**

```bash
# Enable the two prerequisite APIs first
gcloud services enable \
  cloudresourcemanager.googleapis.com \
  iam.googleapis.com \
  --project=YOUR_PROJECT_ID

# Then enable the rest
gcloud services enable \
  compute.googleapis.com \
  storage.googleapis.com \
  bigquery.googleapis.com \
  --project=YOUR_PROJECT_ID
```

**Mental model:** Think of GCP APIs like electrical circuits. The project is the breaker box. Each API is a breaker. The service won't turn on until you flip the switch — and the breaker panel itself (`cloudresourcemanager`) needs to be powered before you can flip anything else.

---

### Failure 2: Owner ≠ Admin (The Missing Role)

**Symptom:** You successfully SSH into a VM, but any command that requires elevated privileges fails:

```
sudo: effective uid is not 0, is /usr/bin/sudo on a file system with the 'nosuid' option set
```

or simply:

```
user is not in the sudoers file. This incident will be reported.
```

**What's happening:** GCP uses OS Login to manage SSH access to VMs. OS Login ties SSH user identity to your Google account's IAM roles. The `Project Owner` role grants you control over resources (creating, deleting, billing) but does not grant you administrator access *inside* a VM's operating system.

For `sudo` to work, your account needs `roles/compute.osAdminLogin`. Without it, you are logged in as a standard unprivileged user, even if you created the VM yourself.

**The fix:**

```bash
gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="user:YOUR_EMAIL" \
  --role="roles/compute.osAdminLogin"
```

After granting the role, log out of the VM and SSH back in for it to take effect. The role update is not applied to active sessions.

**Mental model:** Being the `Owner` of a building means you own the deed. `osAdminLogin` is the master key to the rooms inside. You can own a building without having a key made for yourself — you must create it explicitly.

---

### Failure 3: The 403 Mystery (Access Denied on Public Buckets)

**Symptom:** `gsutil cp` or `gcloud storage cp` returns a `403 Access Denied` error when reading from a publicly accessible Cloud Storage bucket — even though the bucket does not require any credentials.

```
AccessDeniedException: 403 service-account@project.iam.gserviceaccount.com
does not have storage.objects.get access to the Google Cloud Storage object.
```

**What's happening:** This failure requires understanding that VM access to Cloud Storage is controlled by *two independent gates*, both of which must be open:

| Gate | What it controls | Where it is set |
|------|-----------------|-----------------|
| IAM Role | What the VM's service account is *allowed* to do | IAM policy on the project or bucket |
| API Scope | What Cloud APIs the VM is *permitted to call at all* | Set at VM creation time, under "OS and storage access" |

The IAM role (`roles/storage.objectViewer`) says "this identity is authorized to read Storage objects." The API scope says "this VM is allowed to make Storage API calls." If the API scope is set to "Default" instead of "Full Access," the VM's `gsutil` calls are blocked at the network layer before IAM is even checked — and the error you receive looks like an IAM error.

This is why granting the IAM role alone is not sufficient. Both gates must be open.

**The fix (must be done at VM creation — cannot be changed on a running VM):**

```bash
# Correct VM creation command — note --scopes=cloud-platform
gcloud compute instances create my-vm \
  --machine-type=e2-standard-2 \
  --image-family=ubuntu-2204-lts \
  --image-project=ubuntu-os-cloud \
  --provisioning-model=SPOT \
  --scopes=cloud-platform          # <-- This is the critical flag
```

If you already have a running VM without this scope:

```bash
# 1. Stop the VM
gcloud compute instances stop my-vm --zone=us-central1-a

# 2. Export your results to Cloud Storage first (or accept losing local work)
# 3. Delete the VM
gcloud compute instances delete my-vm --zone=us-central1-a

# 4. Recreate it with --scopes=cloud-platform
```

Also grant the service account the IAM role:

```bash
SA=$(gcloud iam service-accounts list \
  --filter="displayName:Compute Engine default service account" \
  --format="value(email)" \
  --project=YOUR_PROJECT_ID)

gcloud projects add-iam-policy-binding YOUR_PROJECT_ID \
  --member="serviceAccount:$SA" \
  --role="roles/storage.objectViewer"
```

---

### Failure 4: Column Name Mismatch in BigQuery

**Symptom:** A BigQuery query that looks correct returns `Unrecognized name: CLNSIG` or `Unrecognized name: clinical_significance`.

**What's happening:** Public genomic datasets in BigQuery are versioned by date snapshot. The ClinVar dataset at `ncbi_clinvar_hg38_20180701` uses `CLNSIG` for clinical significance. Newer snapshots rename this column to `clinical_significance`. Neither version is wrong — they are different tables.

**The fix:** Always run an `INFORMATION_SCHEMA` check before writing analysis queries:

```sql
SELECT column_name, data_type
FROM `bigquery-public-data.ncbi_clinvar_hg38_20180701.INFORMATION_SCHEMA.COLUMNS`
WHERE table_name = 'clinvar_variants'
ORDER BY ordinal_position;
```

Make this the first query you run against any new BigQuery table. It costs nothing (metadata queries are not billed) and prevents wasted debugging time.

---

## 2. The Identity vs. Scope Distinction: VM Permissions Explained

This is the most conceptually confusing part of Google Cloud for researchers coming from HPC, where SSH access is governed by a single `~/.ssh/authorized_keys` file.

### Two Separate Systems

GCP VM access is controlled by two independent systems that must both grant permission for an action to succeed:

**System 1: IAM (Identity and Access Management)**  
Answers: *Who is this, and what are they allowed to do?*  
IAM roles are attached to a Google identity (your account or a service account). They define which GCP resources and actions are permitted.

**System 2: API Access Scopes**  
Answers: *Is this VM permitted to call this Cloud API at all?*  
Scopes are set at VM creation time and define which Google Cloud APIs the VM can reach out to. They act as an allowlist on outbound API calls from the VM.

### Why Both Are Required

Think of it as a two-door security system:

```
VM makes a gsutil call
        │
        ▼
  ┌─────────────┐
  │  API Scope  │  ← "Is Cloud Storage on the allowed API list for this VM?"
  │  (Gate 1)   │      Set at VM creation. --scopes=cloud-platform = all APIs allowed.
  └──────┬──────┘
         │ PASS
         ▼
  ┌─────────────┐
  │  IAM Role   │  ← "Does this service account have permission to read this bucket?"
  │  (Gate 2)   │      Set via gcloud iam. roles/storage.objectViewer = read allowed.
  └──────┬──────┘
         │ PASS
         ▼
   Request succeeds
```

If either gate fails, the request fails. The error message for a Gate 1 failure (scope) often looks identical to a Gate 2 failure (IAM), which is why this is so confusing to debug.

### Practical Rule

When creating any VM that will access Cloud Storage, BigQuery, or other GCP services:

- **Always** use `--scopes=cloud-platform` (full access) unless you have a specific security reason to restrict scopes.
- **Always** grant the VM's service account the appropriate IAM roles for each resource it needs to access.
- Treat these as a checklist, not alternatives.

### The sudo Problem is a Separate Layer

SSH access and `sudo` access are a third independent system. Your Google account identity (from IAM) controls whether you can SSH in. The `osAdminLogin` role controls whether you get elevated privileges inside the OS once connected. A user can have full SSH access but no `sudo` — which is exactly what happens if `osAdminLogin` is not granted.

---

## 3. The Bioinformatics Dependency Guide: Conda vs. Sudo

### The Problem with Conda on Cloud VMs

Conda is an excellent package manager for local workstations and HPC environments where you have stable, long-lived home directories and reproducible system configurations. On cloud VMs, two characteristics make it less reliable:

- **VMs are ephemeral.** The base OS (Ubuntu 22.04) on a new VM is clean and minimal. Conda packages sometimes depend on shared system libraries (like `libgsl`, `libhdf5`, or `libopenblas`) that are present on typical HPC nodes but absent from a minimal cloud VM image.
- **Conda environments isolate from the system — but not completely.** When a Conda package's bundled library version doesn't match the system's installed version, you get a shared library runtime error at execution time, not at installation time. This makes it particularly hard to diagnose.

### The libgsl Failure Mode

The specific error encountered during the Bootcamp:

```
bcftools: error while loading shared libraries: libgsl.so.25:
cannot open shared object file: No such file or directory
```

This means:

1. Conda installed a version of `bcftools` that was compiled against `libgsl` version 2.5 (`libgsl.so.25`).
2. The VM does not have this exact version of the GNU Scientific Library installed at the system level.
3. Conda's bundled copy of `libgsl` is in the wrong location or has a version mismatch.

Reinstalling `bcftools` through Conda does not fix this. Re-solving the environment does not fix this. The Conda environment is self-consistent, but the runtime link cannot be satisfied.

### The Decision Framework

Use this framework to decide how to install any bioinformatics tool:

```
Does the tool need to interact with other Conda-managed packages
(e.g., a Python library calling samtools via subprocess)?
        │
       YES ──► Install via Conda. Accept that you may need to debug
                library conflicts.
        │
        NO
        │
        ▼
Is the tool a compiled binary (C/C++) that only reads/writes files?
(e.g., bcftools, bwa, tabix, bgzip)
        │
       YES ──► Prefer system-native install via apt-get.
                This is faster, more stable, and avoids library conflicts.
        │
        NO
        │
        ▼
Is the tool a Python package with no compiled extensions?
        │
       YES ──► Install via Conda or pip inside a Conda env.
```

### The System-Native Install Pattern

For core bioinformatics CLI tools, use the system package manager:

```bash
sudo apt-get update
sudo apt-get install -y \
  bcftools \
  samtools \
  tabix \
  bwa
```

These packages are compiled and linked against the exact library versions present in Ubuntu 22.04's package repository. They will not produce shared library errors.

**Always call system-installed tools by their full path** when a Conda environment is active, to prevent the Conda environment's PATH from shadowing the system binary:

```bash
# Unreliable when conda env is active — may call wrong binary
bcftools stats input.vcf.gz

# Reliable — always calls the system-installed version
/usr/bin/bcftools stats input.vcf.gz
```

You can check which binary will be called in your current environment:

```bash
which bcftools          # shows the path that will be used
type -a bcftools        # shows all locations, in priority order
```

### When Conda is the Right Choice

Conda remains the right choice for:

- Python-based tools and their dependencies (`pysam`, `cyvcf2`, `snakemake`, `pandas`)
- Tools that require a specific Python version or isolated environment
- Reproducibility across different machines (with `conda env export > environment.yml`)
- R packages via the `r-base` and `r-bioc-*` channels

For these cases, Conda's isolation is a feature, not a liability.

### Stability Rule of Thumb

> If a tool ships as a compiled binary and your primary interaction with it is reading and writing genomic files, install it with `apt-get`. If a tool is a Python or R package, install it with Conda. When in doubt, try `apt-get` first — it is faster to diagnose a missing package than a library conflict.

---

## 4. The Path Trap: Public Data Moves

### What Happens

Public genomic datasets hosted on Cloud Storage (1000 Genomes, ENCODE, dbGaP open-access, GEO) are managed by independent organizations. File paths, bucket names, and directory structures change over time due to:

- Dataset versioning (new genome builds, new analysis releases)
- Bucket reorganization or renaming
- Data retirement or restriction changes

A file path that worked in a paper published two years ago may return a 404 today.

### The Standard Verification Step

Before any download or streaming command, verify the path:

```bash
# Check if a specific file exists
gcloud storage ls gs://genomics-public-data/references/GRCh38/

# Or with gsutil
gsutil ls gs://genomics-public-data/references/GRCh38/

# Check a specific file
gsutil stat gs://genomics-public-data/references/GRCh38/Homo_sapiens_assembly38.fasta
```

If the path returns `BucketNotFoundException` or `CommandException: No URLs matched`, the data has moved. Search the dataset's documentation or GitHub repository for the current canonical path.

### Requester Pays Buckets

Some public buckets (particularly 1000 Genomes and controlled-access datasets) require you to specify a billing project for egress costs, even if the data itself is free to access. Without this, you receive:

```
AccessDeniedException: 403 USER_PROJECT_MISSING
```

This is not a permissions error. Add `-u YOUR_PROJECT_ID` to your `gsutil` command:

```bash
gsutil -u YOUR_PROJECT_ID cp \
  gs://requester-pays-bucket/file.cram ./
```

---

## 5. The Silent Error: When a Command "Works" but Isn't Done

### The bcftools stats Timing Issue

`bcftools stats` produces output in two distinct phases:

**Phase 1 — Header (immediate):** The command prints a block of `#` comment lines containing metadata, column definitions, and parameter information. This appears within the first second of execution.

**Phase 2 — Data rows (delayed):** Lines beginning with `SN`, `TSTV`, `SiS`, `AF`, `QUAL`, `IDD`, `ST`, `DP` contain the actual computed statistics. These require a full pass through the file. For a 5.3GB VCF, this takes 3–8 minutes on an `e2-standard-2`.

**The trap:** The terminal appears to be "working" during Phase 2, but no output is visible. Researchers unfamiliar with this behavior assume the command has stalled and press Ctrl+C — interrupting a job that was running correctly.

**How to confirm the job is still running:**

```bash
# In a second SSH session to the same VM, check active processes
ps aux | grep bcftools

# Or check CPU usage
top
```

If `bcftools` appears in `ps` output with non-zero CPU, it is working. Wait for it to finish.

**Confirming the output is complete:**

After the command returns to the prompt, verify the statistics rows are present:

```bash
# Should return several lines starting with "SN"
grep "^SN" vcf_stats.txt

# Total line count should be in the hundreds for a standard VCF
wc -l vcf_stats.txt
```

If `grep "^SN" vcf_stats.txt` returns nothing, the job was interrupted before completing. Rerun the command.

### The General Principle

Cloud compute is billed by time. The instinct on HPC is to kill a job that appears stalled and resubmit. On cloud VMs, killing a correct-but-slow job wastes the time already billed and restarts the clock. Before interrupting any long-running bioinformatics command, verify it is actually stalled (0% CPU, no output files growing) rather than simply slow.

---

## 6. Reference: Error Code Decoder

| Error message | Actual cause | Fix |
|---------------|-------------|-----|
| `AccessDeniedException: 403 ... does not have storage.objects.get access` | VM API scope too restrictive OR service account missing IAM role | Set `--scopes=cloud-platform` at VM creation AND grant `roles/storage.objectViewer` to service account |
| `AccessDeniedException: 403 USER_PROJECT_MISSING` | Requester Pays bucket, billing project not specified | Add `-u YOUR_PROJECT_ID` to all `gsutil` commands for this bucket |
| `METHOD_NOT_FOUND` or `API has not been used in project` | GCP API not enabled | Run `gcloud services enable [API_NAME]` |
| `sudo: user is not in the sudoers file` | Missing `roles/compute.osAdminLogin` | Grant role, then log out and SSH back in |
| `libXXX.so.XX: cannot open shared object file` | Conda shared library mismatch | Uninstall from Conda; install via `sudo apt-get install` |
| `Unrecognized name: CLNSIG` (BigQuery) | Column name differs between dataset versions | Run `INFORMATION_SCHEMA.COLUMNS` query first |
| `No URLs matched` or `BucketNotFoundException` | File path or bucket has changed | Verify path with `gsutil ls` before downloading |
| `bcftools stats` produces only header lines | Job still running; Phase 2 data calculation in progress | Wait for completion; verify with `ps aux | grep bcftools` |
| VM terminated unexpectedly | Spot VM preempted by GCP | Expected behavior for Spot instances; resubmit job |
