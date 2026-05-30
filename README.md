# GCP-Public-Bucket
Create a public bucket in GCP to demonstrate mistakes in cloud configuration

## Introduction

In this lab we simulate one of the most common and dangerous cloud security mistakes — a misconfigured Cloud Storage bucket that exposes sensitive files to anyone on the internet.

This lab documents real permission errors encountered during setup, including GCP VM access scopes and IAM misconfigurations. These are not just obstacles — they are important security concepts worth understanding.

---

## What is Cloud Storage Misconfiguration?

A Cloud Storage bucket is a container for storing files on Google Cloud. By default, buckets are private — only authorized users can access them. A misconfiguration happens when access controls are set incorrectly, making the bucket publicly accessible to anyone on the internet without any credentials.

Despite being one of the most well-known security issues in cloud computing, misconfigured storage buckets remain one of the top causes of data breaches. Real-world examples include exposed customer databases, leaked employee records, and publicly accessible backup files — all sitting open on the internet waiting to be found.

There are two dangerous permission settings that make a bucket public:
- **`allUsers`** — anyone on the internet, no account needed
- **`allAuthenticatedUsers`** — anyone with a Google account, no need to be part of your organization

---

## What Are We Building?

In this lab we will:
- Deploy a free GCP VM and fix its storage permissions correctly
- Create a Cloud Storage bucket with fake sensitive files
- Intentionally misconfigure it to be publicly accessible

## Prerequisites

- GCP account with free trial ($300 credit)
- Basic Linux command line knowledge
- A browser for testing public access

## Phase 1 — Create the GCP VM

### Step 1 — Create the instance

Go to **GCP Console → Compute Engine → VM Instances → Create Instance**

| Setting | Value |
|---|---|
| Name | `lab2-vm` |
| Region | `us-east1`, `us-west1`, or `us-central1` |
| Machine type | `e2-micro` |
| Boot disk OS | Debian |
| Boot disk size | 30GB Standard HDD |
| Ops Agent | Uncheck |
| Firewall | Allow HTTP and HTTPS |

Click **Create** and wait ~1 minute.

> ⚠️ You will see a ~$7/month estimate. Google automatically waives this for e2-micro in the correct region with a 30GB standard disk.

### Step 2 — SSH into your VM

Click the **SSH button** next to your instance in the GCP console.

---

## Phase 2 — Fix VM Permissions to Access Storage

This phase documents a real error you will hit when trying to upload files from your VM to Cloud Storage. Understanding it is as important as the lab itself.

GCP VMs have **two separate permission layers** that both need to be configured:

- **IAM Roles** — what the service account is allowed to do at the project level
- **Access Scopes** — what APIs the VM itself is allowed to call

Most people only configure IAM roles and forget access scopes — the VM then throws errors even though the IAM role looks correct.

### Step 3 — Grant the IAM role first

Go to **GCP Console → IAM & Admin → IAM**

Find the compute service account — it looks like:
```
[PROJECT_NUMBER]-compute@developer.gserviceaccount.com
```

Click the **pencil icon** → **Add another role** → search **Storage Admin** → **Save**

<img width="889" height="475" alt="image" src="https://github.com/user-attachments/assets/45a05854-424e-4bc8-b709-00be4b7b0560" />


### Step 4 — Try uploading a file (this will fail)

Go back to your VM terminal and try:

```bash
gcloud storage cp ~/test.txt gs://YOUR_BUCKET_NAME/
```

You will get this error:

```
googlecloudsdk.api_lib.storage.errors.GcsApiError:
[PROJECT_NUMBER-compute@developer.gserviceaccount.com] does not
have permission to access b instance [bucket-name].
Provided scope(s) are not authorized.
```

<img width="875" height="125" alt="6" src="https://github.com/user-attachments/assets/17495b86-6260-4edb-8e91-d24d3496f315" />


The IAM role is set correctly but the VM's **access scope** is still blocking the API call. Both layers must be configured.

### Step 5 — Fix the access scope

You cannot change access scopes on a running VM. You must stop it first:

1. Go to **Compute Engine → VM Instances**
2. Click the three dots next to your VM → **Stop** — wait until it shows Stopped
3. Click your VM name → **Edit**
4. Scroll down to **Service Accounts**
5. Under **Access scopes** select **Allow full access to all Cloud APIs**
6. Click **Save**
7. Click **Start** and wait for the VM to come back up

<img width="1070" height="474" alt="7" src="https://github.com/user-attachments/assets/02d3d4c1-45d2-4414-b23f-aa7899b7800c" />

### Step 6 — Verify the fix

SSH back into your VM and retry:

```bash
gcloud storage cp ~/test.txt gs://YOUR_BUCKET_NAME/
```

It should succeed now. Both permission layers are correctly configured.

<img width="881" height="107" alt="1" src="https://github.com/user-attachments/assets/066f7f16-09fa-4dfb-9def-93b64d1d3c19" />

### Why this matters for security

This two-layer permission model is actually a security feature — even if an attacker compromises the service account's IAM credentials, the VM's access scope adds a second barrier. However it is also a common source of misconfiguration that leaves developers confused and sometimes causes them to over-grant permissions trying to fix it.

---

## Phase 3 — Create the Storage Bucket

### Step 4 — Create the bucket

Go to **GCP Console → Cloud Storage → Buckets → Create**

| Setting | Value |
|---|---|
| Name | `thewiredmind-lab2-[random number]` |
| Region | `us-east1` |
| Storage class | Standard |
| Access control | Uniform |
| Public access prevention | Enforced (ON) |

> ⚠️ Bucket names must be globally unique across all of GCP. Use a number or your name to make it unique.

Click **Create**.

<img width="826" height="357" alt="image" src="https://github.com/user-attachments/assets/8047e2f7-a663-49ea-9477-dfa47b120cd0" />

### Step 5 — Verify the bucket exists

SSH into your VM and run:

```bash
gcloud storage buckets list --format="value(name)"
```

Your bucket name should appear in the list.

---

## Phase 4 — Upload Sensitive Files

We create realistic-looking fake files to make the demo impactful. These files look like real sensitive data a company would store on cloud storage.

### Step 6 — Create the fake files on the VM

```bash
mkdir ~/lab-bucket-files
cd ~/lab-bucket-files

# Fake employee data
cat > employees.csv << 'EOF'
employee_id,first_name,last_name,email,salary,department,ssn
1001,John,Smith,jsmith@company.com,85000,Engineering,XXX-XX-1234
1002,Sarah,Johnson,sjohnson@company.com,92000,Finance,XXX-XX-5678
1003,Mike,Williams,mwilliams@company.com,78000,HR,XXX-XX-9012
1004,Emily,Brown,ebrown@company.com,105000,Engineering,XXX-XX-3456
EOF

# Fake database backup
cat > db_backup_2024.sql << 'EOF'
-- Production Database Backup
-- Date: 2024-01-15
-- CONFIDENTIAL - DO NOT SHARE

CREATE TABLE users (
  id INT PRIMARY KEY,
  username VARCHAR(50),
  password_hash VARCHAR(255),
  email VARCHAR(100),
  credit_card_last4 VARCHAR(4)
);

INSERT INTO users VALUES (1, 'admin', '$2b$12$hashedpassword...', 'admin@company.com', '4242');
INSERT INTO users VALUES (2, 'jsmith', '$2b$12$hashedpassword...', 'jsmith@company.com', '1234');
EOF

# Fake config file
cat > config.env << 'EOF'
# Production Config
DB_HOST=10.128.0.5
DB_PASSWORD=Pr0ductionP@ss2024
API_KEY=prod-api-key-abc123xyz
STRIPE_KEY=sk_live_fakekeyforlab
EOF
```

<img width="777" height="245" alt="3" src="https://github.com/user-attachments/assets/fa3672fe-7973-4e53-ad41-26d1e6495964" />

<img width="602" height="110" alt="4" src="https://github.com/user-attachments/assets/203ead75-69f7-491a-9519-c51065556708" />

<img width="696" height="317" alt="2" src="https://github.com/user-attachments/assets/7c347648-2cae-4d45-8619-9c8faf754a33" />


### Step 7 — Upload the files to the bucket

```bash
gcloud storage cp ~/lab-bucket-files/* gs://YOUR_BUCKET_NAME/
```

Expected output:
```
Copying file:///home/user/lab2-bucket-files/config.env to gs://YOUR_BUCKET_NAME/config.env
Copying file:///home/user/lab2-bucket-files/db_backup_2024.sql to gs://YOUR_BUCKET_NAME/db_backup_2024.sql
Copying file:///home/user/lab2-bucket-files/employees.csv to gs://YOUR_BUCKET_NAME/employees.csv
Completed files 3/3 | 885B/885B
```

<img width="881" height="107" alt="1" src="https://github.com/user-attachments/assets/a9686ef9-b92c-45b9-944f-ac5d0a507d66" />


### Step 8 — Verify files are in the bucket

```bash
gcloud storage ls gs://YOUR_BUCKET_NAME/
```

<img width="1366" height="532" alt="image" src="https://github.com/user-attachments/assets/dc99f827-1ef4-474f-8df6-2fd62bf07088" />


---

## Phase 5 — Misconfigure the Bucket

### Step 9 — Disable public access prevention

GCP enables public access prevention by default as a security guard. We remove it to simulate a misconfiguration:

```bash
gcloud storage buckets update gs://YOUR_BUCKET_NAME \
  --no-public-access-prevention
```

Or in the GCP Console:
- Click your bucket → **Permissions** tab
- Click **Edit** next to "Public access prevention"
- Set to **Not enforced**
- Save

<img width="422" height="84" alt="image" src="https://github.com/user-attachments/assets/cf832b2a-93aa-4bb1-ac70-0eeaa9892f53" />


### Step 10 — Make the bucket publicly readable

```bash
gcloud storage buckets add-iam-policy-binding gs://YOUR_BUCKET_NAME \
  --member=allUsers \
  --role=roles/storage.objectViewer
```

GCP will warn you:

```
Enabling allUsers for storage.objectViewer on gs://YOUR_BUCKET_NAME
This bucket is now publicly accessible.
```

This is the misconfiguration. The bucket is now fully open to the internet — no credentials required to read its contents.

### Step 11 — Verify the misconfiguration

```bash
gsutil iam get gs://YOUR_BUCKET_NAME
```

You will see `allUsers` with `roles/storage.objectViewer` in the output — confirming anyone on the internet can read every file.

---

## Phase 6 — Discover It Like an Attacker

This phase demonstrates multiple methods attackers use to find and access exposed buckets. You do not need any special tools — just a browser and gsutil.

### Step 12 — Access via browser (zero authentication)

Open your browser and go to:
```
https://storage.googleapis.com/YOUR_BUCKET_NAME/employees.csv
```

The file downloads immediately. No login. No credentials. No account needed.

<img width="535" height="375" alt="image" src="https://github.com/user-attachments/assets/2b16e222-8d5e-40c6-9887-384c72ce7c56" />

### Step 13 — List all files via browser

Any attacker can see every file in the bucket using this URL:
```
https://storage.googleapis.com/storage/v1/b/YOUR_BUCKET_NAME/o
```

This returns a JSON list of every object in the bucket — file names, sizes, creation dates — completely publicly accessible.


### Step 14 — Download files anonymously via gsutil

An attacker does not need a GCP account to access public buckets:

```bash
# List bucket contents
gsutil ls gs://YOUR_BUCKET_NAME

# Download the database backup
gsutil cp gs://YOUR_BUCKET_NAME/db_backup_2024.sql ./stolen-backup.sql

# Read a file directly without downloading
gsutil cat gs://YOUR_BUCKET_NAME/config.env
```

### Step 15 — Google dorking to find the bucket

Attackers use Google to find exposed buckets by searching:
```
site:storage.googleapis.com YOUR_COMPANY_NAME
```

Buckets get indexed by Google over time — once indexed they appear in search results and anyone searching your company name on Google Cloud storage could find them.

<img width="821" height="593" alt="Screenshot (40)" src="https://github.com/user-attachments/assets/01a56151-bb90-41a0-b985-5882978eee9c" />

### How attackers find buckets they don't know about

Attackers find exposed buckets by:
- **Name guessing** — companies use predictable names like `companyname-backup`, `companyname-prod`, `companyname-data`
- **Google dorking** — `site:storage.googleapis.com companyname`
- **Automated scanners** — tools that brute force thousands of common bucket name patterns per minute
- **Source code leaks** — bucket names often appear in leaked config files or public GitHub repos

---

## Cleanup

### Delete the bucket when done

```bash
# Delete all files
gcloud storage rm gs://YOUR_BUCKET_NAME/**

# Delete the bucket
gcloud storage buckets delete gs://YOUR_BUCKET_NAME
```

> ⚠️ Always clean up after labs to avoid unexpected charges. Storage costs are minimal but it's good practice.

---

*Part of the TheWiredMind GCP Security Labs series.*
*Follow on [TikTok](https://www.tiktok.com/@the.wired.mind) and [YouTube](https://www.youtube.com/@thewiredmind-u3n) for video walkthroughs.*

---

 
