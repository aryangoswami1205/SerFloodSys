# 📚 LESSON — The Three Big Ideas Behind SerHydroSys

> **Target audience:** You! A beginner who is learning AWS and Python for the first time.  
> Everything below is explained as simply as possible — like you're five years old. 🧒

---

## 1️⃣  What is AWS Lambda?

### The simple analogy

Imagine you have a **magic robot helper** that lives in the cloud (someone else's computer far away). You can give this robot a set of instructions (your Python code). The robot sleeps all day long and costs you nothing while it sleeps. But the moment something happens — say, a timer goes off every hour — the robot **wakes up, does the job, and goes back to sleep**.

That's AWS Lambda.

### The technical bits (still simple)

| Question | Answer |
|---|---|
| **What is it?** | A service from Amazon that runs your code without you having to manage a server. |
| **Why "serverless"?** | Not because there are _no_ servers — Amazon still uses servers. You just never see them, patch them, or pay for them when they're idle. |
| **When does it run?** | Only when it is _triggered_. A trigger can be a timer (every 15 minutes), an HTTP request, a new file appearing in S3, etc. |
| **How much does it cost?** | You pay only for the milliseconds your code actually runs. If it runs for 200 ms once a day, you pay for 200 ms a day — often falling within the free tier. |

### How Lambda fits into SerHydroSys

We write our flood-check code in a file called `lambda_function.py`. Inside that file there is a special function called `lambda_handler`. When Lambda wakes up, it calls `lambda_handler`, our code fetches the river data, checks the water level, and (maybe) saves an alert. Then Lambda goes back to sleep. 💤

---

## 2️⃣  What is an S3 Bucket?

### The simple analogy

Think of S3 as a **giant, magical filing cabinet in the sky**. ☁️🗄️

- The **filing cabinet** itself is called a **bucket**.
- Each **file** you put inside is called an **object**.
- You can give each file a name (called a **key**), like `alerts/20260503T120000Z.json`.
- The filing cabinet never runs out of drawers — it grows automatically.
- You can lock individual drawers so only certain people can open them.

### The technical bits (still simple)

| Question | Answer |
|---|---|
| **Full name** | Amazon Simple Storage Service (S3). |
| **What can you store?** | Anything — text files, images, videos, JSON, CSVs, zip archives … anything up to 5 TB per object. |
| **How is it organised?** | Flat — there are no real "folders." The key `alerts/2026/flood.json` just _looks_ like folders, but S3 treats the whole string as a single key. |
| **Is it durable?** | Extremely. Amazon stores your data across multiple data centres so the chance of losing a file is essentially zero (99.999999999% durability — that's eleven 9s!). |
| **How much does it cost?** | Fractions of a cent per GB per month. For small projects it's practically free. |

### How S3 fits into SerHydroSys

When our Lambda function detects a flood-level reading, it creates a small JSON file containing the alert details and drops it into our S3 bucket (`serhydrosys-flood-alerts`). Later we can:

- Read those files to build a dashboard 📊
- Feed them into another Lambda that sends email or SMS notifications 📧
- Analyse historical alerts to study flood patterns 🔬

---

## 3️⃣  What is `boto3`?

### The simple analogy

Imagine AWS is a **huge toy store** with hundreds of aisles (services). You can't just shout from outside and expect things to happen. You need a **walkie-talkie** that speaks the store's language.

**`boto3` is that walkie-talkie.** 📻

It's a Python library (a bundle of pre-written code) that knows how to talk to every single AWS service — S3, Lambda, DynamoDB, SQS, you name it.

### The technical bits (still simple)

| Question | Answer |
|---|---|
| **What is it?** | The official AWS SDK (Software Development Kit) for Python. |
| **Who made it?** | Amazon themselves. |
| **How do you install it?** | `pip install boto3` (or it's already available inside Lambda). |
| **How does it authenticate?** | It reads your AWS credentials from environment variables or a config file (`~/.aws/credentials`). Inside Lambda the credentials are provided automatically. |

### Key `boto3` concepts

```text
boto3.client("s3")          ← Creates a "client" that speaks S3's language.
client.put_object(...)      ← Tells S3: "Here, store this file for me."
client.get_object(...)      ← Tells S3: "Give me back the file I stored."
client.list_objects_v2(...)  ← Tells S3: "Show me everything in this bucket."
```

### How `boto3` fits into SerHydroSys

In our `lambda_function.py` we do three things with boto3:

1. **Import it** — `import boto3`
2. **Create an S3 client** — `s3_client = boto3.client("s3")`
3. **Upload the alert** — `s3_client.put_object(Bucket=..., Key=..., Body=...)`

That's it! Three lines and our Python code can store files in the cloud. ✨

---

## 🗺️  Putting It All Together

Here's how the full system works, step by step:

```
┌─────────────────┐         ┌──────────────────────┐         ┌───────────────┐
│  CloudWatch     │  timer  │   AWS Lambda          │  HTTP   │  PEGELONLINE  │
│  (scheduler)    │────────▶│   lambda_handler()    │────────▶│  REST API     │
│  "Run every     │         │                       │◀────────│  (river data) │
│   15 minutes"   │         │  FOR each station in  │         └───────────────┘
└─────────────────┘         │   MONITORED_STATIONS: │
                            │   1. Fetch water level│
                            │   2. Compare threshold│
                            │   3. If exceeded →    │
                            │      append to alerts │
                            │                       │         ┌───────────────┐
                            │  If any alerts:       │  boto3  │  Amazon S3    │
                            │   → save combined JSON│────────▶│  (alert file) │
                            └──────────────────────┘         └───────────────┘
```

1. **CloudWatch** wakes up Lambda every 15 minutes (we'll set this up later during deployment).
2. **Lambda** runs `lambda_handler()`, which **loops through every station** in `MONITORED_STATIONS` and calls PEGELONLINE for each one.
3. If **any** station exceeds its individual threshold, its alert is appended to a master `alerts_list`.
4. After the loop, if `alerts_list` is not empty, Lambda uses **`boto3`** to save a **single combined JSON** into the **S3 bucket**.
5. If all stations are safe, Lambda simply logs "all clear" and goes back to sleep.

---

## 4️⃣  Scaling to Arrays — Why Config Dictionaries Beat Hardcoding

### The simple analogy

Imagine you're a teacher with one student. You write that student's name, grade, and seat number on a sticky note and tape it to your desk. Easy! But next semester you get **30 students**. Are you going to tape 30 separate sticky notes all over your desk? Of course not — you'd use a **class roster** (a list). 📋

That's exactly what we did in our code.

### What changed?

In the first version of `lambda_function.py` we had **two hardcoded variables**:

```python
# ❌ Old approach — works for one station only
STATION_ID = "KÖLN"
FLOOD_THRESHOLD_METRES = 5.0
```

If we wanted to add a second station we'd have to copy-paste a huge chunk of code, rename variables, and pray we didn't introduce a typo. That's **fragile**.

In the new version we replaced those two variables with a single **list of dictionaries**:

```python
# ✅ New approach — scales to any number of stations
MONITORED_STATIONS = [
    {"station_id": "KÖLN",    "label": "Cologne (Rhine)",  "threshold_m": 6.20},
    {"station_id": "PASSAU",  "label": "Passau (Danube)",   "threshold_m": 7.00},
    {"station_id": "DRESDEN", "label": "Dresden (Elbe)",    "threshold_m": 4.00},
]
```

Now our `lambda_handler` uses a **`for` loop** to process every station automatically:

```python
for station_config in MONITORED_STATIONS:
    # fetch → check → maybe append alert
```

### Why is this better?

| Problem with hardcoding | How the config array solves it |
|---|---|
| Adding a station means editing the logic code. | Adding a station means adding **one dictionary** to the list — zero logic changes. |
| Each station needs its own threshold variable. | Each dictionary carries its **own** threshold right next to its station ID. |
| Forgetting to update one variable causes silent bugs. | All config lives in **one place** — easy to review and hard to miss. |
| Duplicated code is hard to maintain. | The `for` loop handles 3 stations or 300 stations with the **same code**. |

### The master `alerts_list` pattern

Instead of saving one S3 file per station, we collect all triggered alerts into a single Python list called `alerts_list`. After the loop finishes, we check:

```python
if alerts_list:          # Is the list non-empty?
    save_alerts_to_s3()  # Yes → save one combined JSON to S3
else:
    print("All clear!")  # No  → do nothing, save money 💰
```

This is a very common pattern in data engineering called **batch processing** — gather results first, then act on them in bulk.

### Real-world takeaway

In professional data engineering, configuration is almost **never** hardcoded inside business logic. Instead, it lives in:

- Python lists/dicts (what we're doing now) ✅
- YAML or JSON config files
- Environment variables
- Databases or parameter stores (like AWS SSM Parameter Store)

The key principle: **Separate _what_ to process from _how_ to process it.** That way your processing code stays clean and your configuration can grow independently.

---

## 5️⃣  API Versioning and URL Encoding

### Why do APIs have versions?

Think of an API like a **restaurant menu**. 🍽️ The restaurant might update its menu every year — adding new dishes, removing old ones, changing the format. But they can't just rip the old menu out of everyone's hands! Loyal customers (your code) are still ordering from the old menu.

So the restaurant keeps the **old menu available at one counter** (`rest2009`) and introduces the **new menu at a different counter** (`rest-api/v2`). Eventually they retire the old counter.

That's exactly what happened with PEGELONLINE:

| Version | URL path | Status |
|---|---|---|
| Legacy | `/webservices/rest2009/...` | ⚠️ Deprecated — returns 404 for many stations |
| Modern (v2) | `/webservices/rest-api/v2/...` | ✅ Current and maintained |

**Lesson:** Always check an API's documentation for the **latest supported version**. Using a deprecated endpoint is like ordering from a menu that no longer exists — you'll get errors.

### What is URL encoding (percent-encoding)?

URLs can only contain a limited set of "safe" characters: letters `A-Z`, digits `0-9`, and a few symbols like `-`, `_`, `.`, `~`. Everything else — spaces, umlauts, accented letters, etc. — must be **encoded** before being placed in a URL.

The encoding works by replacing each unsafe character with a `%` sign followed by two hexadecimal digits representing the character's byte value:

| Original character | URL-encoded form | Why? |
|---|---|---|
| `Ö` (umlaut) | `%C3%96` | Non-ASCII character — not safe in URLs |
| ` ` (space) | `%20` | Spaces break URL parsing |
| `ü` | `%C3%BC` | Another non-ASCII character |

### How we handle it in Python

Python's standard library includes `urllib.parse.quote()` — a function that does this encoding for us:

```python
from urllib.parse import quote

quote("KÖLN", safe="")         # → "K%C3%96LN"
quote("PASSAU DONAU", safe="") # → "PASSAU%20DONAU"
quote("DRESDEN", safe="")      # → "DRESDEN"  (no special chars, no change)
```

The `safe=""` parameter tells Python: "Don't leave _any_ characters unencoded." Without it, some characters like `/` would be left as-is, which could break our URL.

### Why this matters for SerHydroSys

Our station `"KÖLN"` contains the German umlaut `Ö`. Without URL encoding, the API would receive a malformed URL and return a `404 Not Found` error — exactly the bug we hit in our first Docker test! By encoding dynamically, our code handles **any** station name safely, even if we add stations with spaces, accents, or other special characters in the future.

---

## 6️⃣  Step 4: Container Isolation — Why We Use Docker

### The simple analogy

Imagine you're baking a cake. 🎂 You _could_ bake it in your home kitchen — but your kitchen has your roommate's spices, your cat walking on the counter, and a wonky oven that runs 10° too hot. If the cake turns out wrong, was it the recipe or the kitchen?

Now imagine a **portable, perfectly clean mini-kitchen** that you can set up _anywhere_ — your house, your friend's house, an AWS data centre — and it always has the exact same oven, exact same tools, exact same temperature. That's **Docker**.

A **Docker container** is that portable mini-kitchen for your code.

### Why not just run `python lambda_function.py` directly on my Mac?

It _works_, but here's what can go wrong:

| Risk when running directly on macOS | How Docker eliminates it |
|---|---|
| Your Mac has Python 3.12, but AWS Lambda uses Python 3.9. Subtle language differences can cause bugs that only appear after deployment. | The Docker image pins `python:3.9-slim` — **exactly** the version Lambda uses. |
| You might have extra packages installed globally (`pip install ...`) that mask a missing dependency in `requirements.txt`. | The container starts **empty** — if `requirements.txt` is missing a package, it fails immediately, not after you deploy. |
| Environment variables, file paths, and OS libraries differ between macOS (Darwin) and Lambda (Amazon Linux). | The container runs Linux, matching Lambda's real operating system. |
| "It works on my machine" — the classic developer excuse when code breaks in production. | Containers are **reproducible** — if it works in Docker, it will work on any machine running Docker (your Mac, your colleague's PC, a CI server, AWS). |

### Key Docker concepts

| Concept | What it means |
|---|---|
| **Image** | A read-only blueprint/recipe. Our `Dockerfile` _describes_ the image. Think of it as a frozen snapshot of an operating system + your code + your dependencies. |
| **Container** | A running instance of an image. When you `docker run`, Docker thaws the snapshot and starts it up. You can run many containers from the same image. |
| **Dockerfile** | The recipe file that tells Docker how to build the image — what base OS to start from, what files to copy, what commands to run. |
| **Layer caching** | Docker saves each step (`FROM`, `COPY`, `RUN`) as a layer. If a layer hasn't changed since the last build, Docker reuses the cached version instead of rebuilding it. That's why we copy `requirements.txt` _before_ `lambda_function.py` — dependency installs are slow but rarely change, so they stay cached. |

### Our Dockerfile — line by line

```dockerfile
# Start from a small, official Python image
FROM python:3.9-slim

# Set our working directory inside the container
WORKDIR /app

# Copy requirements FIRST (for layer caching)
COPY requirements.txt .

# Install Python packages
RUN pip install --no-cache-dir -r requirements.txt

# Copy our source code
COPY lambda_function.py .

# Run the script when the container starts
CMD ["python", "lambda_function.py"]
```

### How to use it

Open your terminal inside the `SerHydroSys/` folder and run:

```bash
# 1️⃣  Build the image (give it a name tag)
docker build -t serhydrosys .

# 2️⃣  Run a container from that image
docker run --rm serhydrosys
```

- `docker build -t serhydrosys .` — Reads the `Dockerfile` in the current directory (`.`), executes each step, and saves the result as an image named `serhydrosys`.
- `docker run --rm serhydrosys` — Starts a container from the image, runs `python lambda_function.py`, prints the output, and automatically removes the container (`--rm`) when it's done.

> **💡 Tip:** The S3 upload will still fail inside Docker (no AWS credentials) — that's expected! We're validating that the **API calls and logic** work correctly. We'll wire up real AWS credentials during deployment.

---

## 7️⃣  Step 5: AWS Serverless Architecture — Deploying for Real

### Zero-dependency code: a serverless superpower

In our first version, we used the `requests` library to call the PEGELONLINE API. That's a great library — but it's a **third-party package**. To use it in Lambda, we'd need to:

1. Create a folder with our code + the `requests` package files.
2. ZIP the folder.
3. Upload the ZIP to Lambda (or to S3, then link it).

That's annoying for a small function! So we replaced `requests` with Python's **built-in** `urllib.request` module — which is included with every Python installation and is already available inside Lambda.

| Approach | What you upload to Lambda | Complexity |
|---|---|---|
| With `requests` | A ZIP file containing your code + `requests` + all its sub-dependencies | Medium — need to package correctly |
| With `urllib` (zero-dep) | Just **copy-paste** `lambda_function.py` into the Lambda editor | Trivially easy ✨ |

**Rule of thumb for serverless engineering:** If a built-in module can do the job, prefer it over a third-party library. Fewer dependencies = faster cold starts, simpler deployments, and fewer things that can break.

### What is IAM? (Identity and Access Management)

IAM is like the **bouncer at every door** inside AWS. 🚪

- When your Lambda function tries to write to S3, IAM checks: "Does this function have permission to do that?"
- If you haven't explicitly said "yes" via an IAM policy, the answer is **always no**. This is called the **Principle of Least Privilege**.
- We created an IAM policy that says: "This Lambda function may `s3:PutObject`, but ONLY into the `alerts/` folder of our specific bucket."

```json
{
    "Effect": "Allow",
    "Action": "s3:PutObject",
    "Resource": "arn:aws:s3:::serhydrosys-flood-alerts/alerts/*"
}
```

**Why is this important?** Imagine if every Lambda function could read/write to every S3 bucket in your account. One buggy function could accidentally delete your entire data lake! IAM prevents that by ensuring each function can only touch what it's explicitly allowed to touch.

### What is EventBridge?

EventBridge is the **alarm clock** of AWS. ⏰

You tell it: "Run my Lambda function every 1 hour" (using a **rate expression** like `rate(1 hour)`), and it does exactly that — 24 hours a day, 7 days a week, without you needing a server running somewhere.

Other things that can trigger Lambda:
- A new file appearing in S3
- An HTTP request (via API Gateway)
- A message arriving in a queue (SQS)
- A database change (DynamoDB Streams)

### The complete serverless mental model

```
You write code (lambda_function.py)
          ↓
    AWS runs it for you (Lambda)
          ↓
    On a schedule (EventBridge)
          ↓
    With permission controls (IAM)
          ↓
    Saving results (S3)
          ↓
    With logs (CloudWatch)
```

**You manage:** Your Python code. That's it.
**AWS manages:** Servers, scaling, networking, patching, availability — everything else.

That's what "serverless" really means. Not "no servers" — but "**not your problem.**" 🎉

### The full deployment walkthrough

We created a dedicated **[AWS_DEPLOYMENT_GUIDE.md](AWS_DEPLOYMENT_GUIDE.md)** file with step-by-step screenshots-style instructions for:

1. Creating the S3 bucket
2. Creating the Lambda function and pasting your code
3. Configuring IAM permissions (least privilege)
4. Setting up an EventBridge schedule (every hour)

Refer to that file for the hands-on walkthrough.

---

## 8️⃣  Step 6: Live Dashboard & S3 CORS

### The `latest_status.json` pattern

Our Lambda now ALWAYS writes a file called `latest_status.json` to S3 — even when all stations are safe. This file is a **snapshot** of every station's current state:

```json
{
  "generated_at": "2026-05-05T12:00:00+00:00",
  "stations_checked": 3,
  "stations": [
    { "label": "Cologne (Rhine)", "water_level_m": 1.75, "threshold_m": 6.20, "status": "SAFE" },
    { "label": "Passau (Danube)",  "water_level_m": 4.10, "threshold_m": 7.00, "status": "SAFE" },
    { "label": "Dresden (Elbe)",   "water_level_m": 0.78, "threshold_m": 4.00, "status": "SAFE" }
  ]
}
```

The dashboard reads this one file via `fetch()` and renders a card for each station. Simple, fast, and cheap — no API Gateway or database needed.

### What is S3 CORS?

**CORS** stands for **Cross-Origin Resource Sharing**. Browsers enforce a security rule: a web page on `mydashboard.com` is **not allowed** to fetch data from `mybucket.s3.amazonaws.com` unless the S3 bucket explicitly says "I allow requests from `mydashboard.com`."

Without CORS configuration, the browser silently blocks the `fetch()` call and you see an error in the console.

**How to enable CORS on your S3 bucket:**

1. Go to your S3 bucket → **Permissions** tab → **CORS configuration** → **Edit**.
2. Paste this JSON:

```json
[
  {
    "AllowedHeaders": ["*"],
    "AllowedMethods": ["GET"],
    "AllowedOrigins": ["*"],
    "ExposeHeaders": [],
    "MaxAgeSeconds": 3600
  }
]
```

3. Click **Save**.

> `"AllowedOrigins": ["*"]` means "allow requests from any website." For a portfolio project that's fine. In production you'd lock this down to your specific domain.

### The frontend architecture

```
frontend-dashboard/
├── index.html    ← Semantic HTML5 shell (header, summary bar, card grid)
├── styles.css    ← Dark-mode climate-tech design with glassmorphism
└── app.js        ← Fetches latest_status.json, renders cards, auto-refreshes
```

The dashboard is **100% static** — no build tools, no Node.js, no framework. You can host it on S3 Static Website Hosting, GitHub Pages, or Netlify for free.

---

## 🧪  What's Next?

Here's our progress:

| Step | Status | What we did / will do |
|---|---|---|
| **Script** | ✅ Done | Built `lambda_function.py` with multi-station monitoring. |
| **Docker test** | ✅ Done | Created a `Dockerfile` to test in an isolated Linux container. |
| **API fix** | ✅ Done | Upgraded to PEGELONLINE v2 API + URL encoding for umlauts. |
| **Zero-dep refactor** | ✅ Done | Replaced `requests` with built-in `urllib` for simpler deployment. |
| **AWS deploy** | ✅ Done | S3 bucket, Lambda function, IAM policy, EventBridge schedule. |
| **Dashboard** | ✅ Done | Dark-mode WebGIS frontend reading `latest_status.json` from S3. |
| **Notify** | ⬜ Next | Add SNS (Simple Notification Service) to send email/SMS alerts. |
| **Polish** | ⬜ | Add a map view, historical charts, and deploy the frontend. |

One step at a time — you've got this! 🚀
