# 🚀 AWS Deployment Guide — SerHydroSys

> **Audience:** A beginner deploying their first AWS Lambda function.
> **Time required:** ~20 minutes.
> **Cost:** Entirely within the AWS Free Tier for low-usage projects.

---

## Prerequisites

- An [AWS Account](https://aws.amazon.com/free/) (free tier is fine).
- Your `lambda_function.py` file (the zero-dependency version using `urllib`).
- A web browser — everything is done via the AWS Management Console.

---

## Step 1: Create the S3 Bucket for Alerts

The S3 bucket is where your Lambda will store flood-alert JSON files.

1. **Open the S3 Console:**
   - Go to [https://console.aws.amazon.com/s3/](https://console.aws.amazon.com/s3/)
   - Click **"Create bucket"** (orange button, top right).

2. **Configure the bucket:**
   | Setting | Value |
   |---|---|
   | **Bucket name** | `serhydrosys-flood-alerts` (must be globally unique — add your initials if taken, e.g. `serhydrosys-flood-alerts-ag`) |
   | **AWS Region** | `eu-central-1` (Frankfurt) — closest to the German data source |
   | **Object Ownership** | ACLs disabled (recommended) |
   | **Block all public access** | ✅ Leave checked (we do NOT want the public reading our alerts) |
   | **Bucket Versioning** | Disabled (not needed for this project) |

3. **Click "Create bucket"** at the bottom of the page.

4. **Done!** You should see your new bucket in the S3 dashboard. It's empty for now — Lambda will populate it.

> **⚠️ Important:** If you changed the bucket name (e.g. added your initials), you'll need to update the `ALERT_BUCKET_NAME` environment variable in your Lambda function (Step 2, section 5).

---

## Step 2: Create the Lambda Function

1. **Open the Lambda Console:**
   - Go to [https://console.aws.amazon.com/lambda/](https://console.aws.amazon.com/lambda/)
   - Click **"Create function"** (orange button).

2. **Choose creation method:**
   - Select **"Author from scratch"**.

3. **Configure the function:**
   | Setting | Value |
   |---|---|
   | **Function name** | `SerHydroSys-FloodAlert` |
   | **Runtime** | `Python 3.9` |
   | **Architecture** | `x86_64` |
   | **Execution role** | "Create a new role with basic Lambda permissions" |

4. **Click "Create function".**

5. **Paste your code:**
   - You'll see the Lambda code editor with a default `lambda_function.py`.
   - **Delete ALL the default code.**
   - Open your local `lambda_function.py`, **copy the entire contents**, and **paste** them into the editor.
   - Click **"Deploy"** (the blue button above the editor).

6. **Add the environment variable:**
   - Click the **"Configuration"** tab → **"Environment variables"** → **"Edit"**.
   - Click **"Add environment variable"**.
   - Key: `ALERT_BUCKET_NAME`
   - Value: `serhydrosys-flood-alerts` (or whatever you named your bucket)
   - Click **"Save"**.

7. **Increase the timeout:**
   - Still in the **"Configuration"** tab → **"General configuration"** → **"Edit"**.
   - Change **Timeout** from 3 seconds to **30 seconds** (we're making 3 API calls to Germany, which can take a few seconds each).
   - Click **"Save"**.

8. **Test it manually:**
   - Click the **"Test"** tab.
   - Event name: `ManualTest`
   - Event JSON: `{}` (empty object — our function doesn't use the event).
   - Click **"Test"**.
   - You should see the function output in the "Execution result" panel!
   - ❌ It will likely fail with an **AccessDenied** error on S3 — that's because we haven't given it permission yet. That's Step 3!

---

## Step 3: Give Lambda Permission to Write to S3 (IAM)

### What is IAM?

**IAM** stands for **Identity and Access Management**. Think of it as the **security guard** at the entrance to every AWS service. 🔐

- Every person, application, or service in AWS has an **identity** (who are you?).
- Every identity has a set of **permissions** (what are you allowed to do?).
- By default, a new Lambda function can do **nothing** except write logs. We need to explicitly grant it permission to write to our S3 bucket.

This is called the **Principle of Least Privilege** — give each service only the minimum permissions it needs, nothing more.

### Add S3 write permission

1. **Open the Lambda Console** → click on your `SerHydroSys-FloodAlert` function.

2. **Go to Configuration → Permissions.**
   - You'll see a **"Role name"** link (something like `SerHydroSys-FloodAlert-role-abc123`).
   - **Click that link** — it opens the IAM console for that role.

3. **Add an inline policy:**
   - Click **"Add permissions"** → **"Create inline policy"**.
   - Click the **"JSON"** tab (top right of the policy editor).
   - **Delete** the default content and **paste** this:

   ```json
   {
       "Version": "2012-10-17",
       "Statement": [
           {
               "Sid": "AllowS3PutAlerts",
               "Effect": "Allow",
               "Action": "s3:PutObject",
               "Resource": "arn:aws:s3:::serhydrosys-flood-alerts/alerts/*"
           }
       ]
   }
   ```

   > **⚠️ Note:** If your bucket has a different name, replace `serhydrosys-flood-alerts` in the Resource ARN.

   **What this policy says in plain English:**
   - `Effect: Allow` → "Yes, you may…"
   - `Action: s3:PutObject` → "…upload files…"
   - `Resource: arn:aws:s3:::serhydrosys-flood-alerts/alerts/*` → "…but ONLY into the `alerts/` folder of this specific bucket."

4. **Click "Next".**
   - Policy name: `SerHydroSys-S3-Write`
   - Click **"Create policy"**.

5. **Test again:**
   - Go back to your Lambda function → **"Test"** tab → click **"Test"**.
   - This time, the function should run successfully! If water levels are normal, you'll see `NO_ALERTS`. If any station exceeds its threshold, an alert file will appear in your S3 bucket.

---

## Step 4: Schedule Automatic Execution with EventBridge

Right now, the function only runs when you manually click "Test". We want it to run **automatically every hour**.

### What is EventBridge?

**EventBridge** (formerly CloudWatch Events) is like an **alarm clock** for your AWS services. ⏰ You set a schedule, and EventBridge triggers your Lambda function at the specified interval.

### Create the schedule

1. **Open the EventBridge Console:**
   - Go to [https://console.aws.amazon.com/events/](https://console.aws.amazon.com/events/)
   - In the left sidebar, click **"Rules"** → **"Create rule"**.

2. **Configure the rule:**
   | Setting | Value |
   |---|---|
   | **Name** | `SerHydroSys-HourlyCheck` |
   | **Description** | Trigger flood-level checks every hour |
   | **Event bus** | `default` |
   | **Rule type** | Schedule |

3. **Click "Next"** → Set the schedule:
   - Select **"A schedule that runs at a regular rate"**.
   - Rate expression: `rate(1 hour)`
   - Click **"Next"**.

4. **Select target:**
   | Setting | Value |
   |---|---|
   | **Target type** | AWS service |
   | **Select a target** | Lambda function |
   | **Function** | `SerHydroSys-FloodAlert` |

5. **Click "Next"** → **"Next"** → **"Create rule"**.

6. **Done!** Your Lambda function will now execute **every hour, 24/7**. 🎉

### Verify it's working

- After one hour, go to your **S3 bucket** and check the `alerts/` folder.
- Go to your **Lambda function** → **"Monitor"** tab → **"View CloudWatch logs"** to see the execution logs.

---

## Architecture Diagram

```
┌──────────────────┐        ┌────────────────────────┐        ┌───────────────┐
│  EventBridge     │ timer  │   AWS Lambda            │  HTTP  │  PEGELONLINE  │
│  (scheduler)     │───────▶│   SerHydroSys-FloodAlert│───────▶│  REST API v2  │
│  rate(1 hour)    │        │                         │◀───────│  (river data) │
└──────────────────┘        │  FOR each station:      │        └───────────────┘
                            │   1. urllib → fetch data│
                            │   2. Check threshold    │
                            │   3. Collect alerts     │
                            │                         │        ┌───────────────┐
                            │  If alerts:             │ boto3  │  Amazon S3    │
                            │   → save combined JSON  │───────▶│  alerts/*.json│
                            └────────────────────────┘        └───────────────┘
                                       │
                                       │ logs
                                       ▼
                            ┌────────────────────────┐
                            │  CloudWatch Logs       │
                            │  (execution history)   │
                            └────────────────────────┘

IAM Role: SerHydroSys-FloodAlert-role
  └─ Policy: s3:PutObject on arn:aws:s3:::serhydrosys-flood-alerts/alerts/*
```

---

## Cost Estimate (Free Tier)

| Service | Free Tier Allowance | Our Usage | Monthly Cost |
|---|---|---|---|
| **Lambda** | 1M requests + 400,000 GB-seconds/month | ~720 requests (1/hour × 24 × 30) | **$0.00** |
| **S3** | 5 GB storage, 20,000 GET, 2,000 PUT | A few KB of JSON files | **$0.00** |
| **EventBridge** | 14M events/month | 720 events | **$0.00** |
| **CloudWatch Logs** | 5 GB ingestion/month | Tiny log volume | **$0.00** |
| | | **Total** | **$0.00** 🎉 |

---

## Troubleshooting

| Problem | Solution |
|---|---|
| `AccessDenied` when writing to S3 | Check Step 3 — the IAM policy might have the wrong bucket name in the Resource ARN. |
| `Task timed out after 3 seconds` | Increase the Lambda timeout to 30 seconds (Step 2, section 7). |
| `404 Not Found` from PEGELONLINE | The station name might be wrong. Check the API directly: `https://www.pegelonline.wsv.de/webservices/rest-api/v2/stations.json` |
| Function runs but no S3 file appears | Water levels are probably below all thresholds (normal!). Check CloudWatch Logs to confirm. |
| `No module named 'boto3'` | This should never happen in Lambda (boto3 is built-in). If testing locally, run `pip install boto3`. |
