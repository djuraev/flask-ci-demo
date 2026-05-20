# GitHub Actions CI Pipeline — Complete Beginner Guide

> **Who is this for?**  
> This guide assumes you have never used GitHub Actions or Docker Hub before.  
> Every single click is explained. If you follow the steps in order, it will work.

---

## Table of Contents

1. [What Are We Building?](#1-what-are-we-building)
2. [Prerequisites — What You Need Before Starting](#2-prerequisites)
3. [Project Structure — What Files We Will Create](#3-project-structure)
4. [Step 1 — Create the Project Folder](#step-1--create-the-project-folder)
5. [Step 2 — Create the Flask App (`app.py`)](#step-2--create-the-flask-app-apppy)
6. [Step 3 — Create the Tests (`test_app.py`)](#step-3--create-the-tests-test_apppy)
7. [Step 4 — Create `requirements.txt`](#step-4--create-requirementstxt)
8. [Step 5 — Create the `Dockerfile`](#step-5--create-the-dockerfile)
9. [Step 6 — Test Everything Locally](#step-6--test-everything-locally)
10. [Step 7 — Create a GitHub Repository](#step-7--create-a-github-repository)
11. [Step 8 — Push Your Code to GitHub](#step-8--push-your-code-to-github)
12. [Step 9 — Create a Docker Hub Account and Access Token](#step-9--create-a-docker-hub-account-and-access-token)
13. [Step 10 — Add Secrets to Your GitHub Repository](#step-10--add-secrets-to-your-github-repository)
14. [Step 11 — Create the GitHub Actions Workflow](#step-11--create-the-github-actions-workflow)
15. [Step 12 — Push and Watch the Pipeline Run](#step-12--push-and-watch-the-pipeline-run)
16. [Step 13 — Verify the Image on Docker Hub](#step-13--verify-the-image-on-docker-hub)
17. [What Happens on Every Push](#what-happens-on-every-push)
18. [Troubleshooting](#troubleshooting)
19. [Glossary](#glossary)

---

## 1. What Are We Building?

We are building a **tiny Python web app** and a **CI/CD pipeline** that automatically:

```
You push code to GitHub
        │
        ▼
GitHub Actions wakes up automatically
        │
        ├─► Job 1: Install dependencies → Run all tests
        │         (if any test fails, the pipeline stops here)
        │
        └─► Job 2: Build a Docker image → Push it to Docker Hub
                  (only runs after Job 1 passes)
```

The web app has one endpoint `/` that returns random values as JSON:

```json
{
  "number": 4271,
  "word": "xkptmq",
  "name": "Charlie"
}
```

And a `/health` endpoint used by the pipeline to verify the app is running:

```json
{ "status": "ok" }
```

**Why does this matter?**  
Every real software team runs something like this. Instead of someone manually building and deploying after every code change, the pipeline does it automatically — and refuses to proceed if the tests fail.

---

## 2. Prerequisites

Make sure you have all of these installed on your computer before starting.

### 2.1 Python 3.10 or newer

Check if it is installed:
```bash
python3 --version
```
Expected output: `Python 3.10.x` or higher.  
If not installed: https://www.python.org/downloads/

### 2.2 Git

Check if it is installed:
```bash
git --version
```
Expected output: `git version 2.x.x`  
If not installed: https://git-scm.com/downloads

### 2.3 Docker Desktop (optional — only needed to test locally)

Download from: https://www.docker.com/products/docker-desktop/  
After installing, open Docker Desktop and wait until it says **"Engine running"**.

### 2.4 A GitHub account

Sign up free at: https://github.com

### 2.5 A Docker Hub account

Sign up free at: https://hub.docker.com

### 2.6 A code editor

Use any editor you like. [VS Code](https://code.visualstudio.com/) is a good free choice.

---

## 3. Project Structure

By the end of this guide your project folder will look like this:

```
flask-ci-demo/
├── app.py                          ← the Flask web app
├── test_app.py                     ← automated tests
├── requirements.txt                ← Python dependency list
├── Dockerfile                      ← instructions to build the Docker image
└── .github/
    └── workflows/
        └── ci.yml                  ← the GitHub Actions pipeline
```

> **What is `.github/workflows/`?**  
> GitHub looks for CI pipeline files in exactly this folder. The name `ci.yml` can be anything — what matters is that the file is in `.github/workflows/`.

---

## Step 1 — Create the Project Folder

Open your terminal (macOS: `Terminal.app`, Windows: `PowerShell` or `Command Prompt`) and run:

```bash
mkdir flask-ci-demo
cd flask-ci-demo
```

You are now inside your new project folder. All files in the next steps go here.

---

## Step 2 — Create the Flask App (`app.py`)

Create a file called `app.py` and paste this content:

```python
from flask import Flask, jsonify
import random
import string

app = Flask(__name__)


def random_number():
    """Returns a random integer between 1 and 9999."""
    return random.randint(1, 9999)


def random_word():
    """Returns a random lowercase word of 4–8 characters."""
    length = random.randint(4, 8)
    return "".join(random.choices(string.ascii_lowercase, k=length))


def random_name():
    """Returns a random English first name."""
    names = [
        "Alice", "Bob", "Charlie", "Diana", "Edward",
        "Fatima", "George", "Hannah", "Ivan", "Julia",
    ]
    return random.choice(names)


@app.route("/health")
def health():
    return jsonify({"status": "ok"})


@app.route("/")
def index():
    return jsonify({
        "number": random_number(),
        "word":   random_word(),
        "name":   random_name(),
    })


if __name__ == "__main__":
    app.run(host="0.0.0.0", port=8080, debug=False)
```

### What each part does

| Part | What it does |
|------|-------------|
| `Flask(__name__)` | Creates the web application object |
| `@app.route("/health")` | When someone visits `/health`, run the `health()` function |
| `@app.route("/")` | When someone visits `/`, run the `index()` function |
| `jsonify({...})` | Converts a Python dictionary to a JSON HTTP response |
| `app.run(host="0.0.0.0", port=8080)` | Starts the web server on port 8080, accessible from all network interfaces |

---

## Step 3 — Create the Tests (`test_app.py`)

Create a file called `test_app.py` and paste this content:

```python
import pytest
from app import app, random_number, random_word, random_name


@pytest.fixture
def client():
    app.config["TESTING"] = True
    with app.test_client() as client:
        yield client


def test_health_returns_200(client):
    response = client.get("/health")
    assert response.status_code == 200


def test_health_returns_ok(client):
    data = client.get("/health").get_json()
    assert data["status"] == "ok"


def test_index_returns_200(client):
    assert client.get("/").status_code == 200


def test_index_has_all_fields(client):
    data = client.get("/").get_json()
    assert "number" in data
    assert "word" in data
    assert "name" in data


def test_random_number_in_range():
    for _ in range(50):
        n = random_number()
        assert 1 <= n <= 9999


def test_random_word_length():
    for _ in range(50):
        w = random_word()
        assert 4 <= len(w) <= 8


def test_random_word_is_lowercase_alpha():
    for _ in range(50):
        assert random_word().isalpha()
        assert random_word().islower()


def test_random_name_is_valid():
    valid = {
        "Alice", "Bob", "Charlie", "Diana", "Edward",
        "Fatima", "George", "Hannah", "Ivan", "Julia",
    }
    for _ in range(30):
        assert random_name() in valid
```

### What is a test?

A test is code that checks if your app behaves correctly.

- `assert response.status_code == 200` means: "assert (confirm) that the status code is 200. If it is not, the test fails."
- `for _ in range(50)` means: run the check 50 times. We do this for random functions because we want to check many different outputs.
- If **any** `assert` fails, the entire pipeline stops and refuses to build the Docker image. This is exactly what you want — broken code never reaches Docker Hub.

---

## Step 4 — Create `requirements.txt`

Create a file called `requirements.txt` and paste this:

```
flask==3.0.3
pytest==8.2.2
```

### What is this file?

This file tells Python which libraries your app needs and which exact versions to install. The CI pipeline reads this file and installs these libraries automatically on GitHub's servers.

> **Why pin exact versions?** (the `==3.0.3` part)  
> If you don't pin versions, a library might update and break your app. Pinning ensures the same version is always installed on every machine — your laptop, your CI server, and your Docker container.

---

## Step 5 — Create the `Dockerfile`

Create a file called `Dockerfile` (no extension, capital D) and paste this:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir flask==3.0.3

COPY app.py .

EXPOSE 8080

CMD ["python", "app.py"]
```

### What each line does

| Line | Meaning |
|------|---------|
| `FROM python:3.12-slim` | Start from an official Python 3.12 image. `-slim` means smaller file size |
| `WORKDIR /app` | All following commands run inside the `/app` folder inside the container |
| `COPY requirements.txt .` | Copy your `requirements.txt` from your computer into the container |
| `RUN pip install ...` | Install Flask inside the container |
| `COPY app.py .` | Copy your app code into the container |
| `EXPOSE 8080` | Document that port 8080 is used (does not actually open the port) |
| `CMD ["python", "app.py"]` | The command that runs when someone starts a container from this image |

> **Why copy `requirements.txt` before `app.py`?**  
> Docker builds images in layers. If you copy `app.py` first and it changes, Docker rebuilds the `pip install` layer too — even though dependencies didn't change. Copying `requirements.txt` first means the expensive `pip install` layer is cached and only reruns when dependencies actually change.

---

## Step 6 — Test Everything Locally

Before pushing to GitHub, verify that everything works on your machine.

### 6.1 Install dependencies

```bash
pip install -r requirements.txt
```

### 6.2 Run the tests

```bash
pytest test_app.py -v
```

Expected output:
```
test_app.py::test_health_returns_200 PASSED
test_app.py::test_health_returns_ok PASSED
test_app.py::test_index_returns_200 PASSED
test_app.py::test_index_has_all_fields PASSED
test_app.py::test_random_number_in_range PASSED
test_app.py::test_random_word_length PASSED
test_app.py::test_random_word_is_lowercase_alpha PASSED
test_app.py::test_random_name_is_valid PASSED

8 passed in 0.xx seconds
```

If you see `FAILED` next to any test, fix the error before continuing.

### 6.3 Run the app locally

```bash
python app.py
```

Open your browser and go to:
- `http://localhost:8080/` — should return JSON with number, word, name
- `http://localhost:8080/health` — should return `{"status": "ok"}`

Press `Ctrl+C` to stop the app.

### 6.4 Test the Docker build locally (optional but recommended)

```bash
# Build the image
docker build -t flask-demo .

# Run a container from the image
docker run -p 8080:8080 flask-demo
```

Open `http://localhost:8080/` — same output as before. Press `Ctrl+C` to stop.

---

## Step 7 — Create a GitHub Repository

1. Go to [https://github.com](https://github.com) and sign in.
2. Click the **green "New"** button in the top-left, or go to [https://github.com/new](https://github.com/new).
3. Fill in the form:
   - **Repository name:** `flask-ci-demo`
   - **Visibility:** Public (or Private — both work)
   - **Do NOT** check "Add a README file"
   - **Do NOT** check "Add .gitignore"
   - **Do NOT** check "Choose a license"
4. Click **"Create repository"** (green button at the bottom).
5. GitHub shows you a page with setup instructions. **Leave this page open** — you will need the repository URL in the next step.

---

## Step 8 — Push Your Code to GitHub

Go back to your terminal (make sure you are inside the `flask-ci-demo` folder) and run these commands one by one:

```bash
# Step 1: Initialize a git repository in this folder
git init

# Step 2: Tell git who you are (use your real email/name)
git config user.email "you@example.com"
git config user.name "Your Name"

# Step 3: Stage all files for the first commit
git add .

# Step 4: Create the first commit
git commit -m "initial commit: flask app + tests + dockerfile"

# Step 5: Rename the default branch to "main"
git branch -M main

# Step 6: Connect your local folder to GitHub
#         Replace YOUR_USERNAME with your GitHub username
git remote add origin https://github.com/YOUR_USERNAME/flask-ci-demo.git

# Step 7: Push the code to GitHub
git push -u origin main
```

After `git push`, go back to your browser and refresh the GitHub repository page. You should see your files there: `app.py`, `test_app.py`, `requirements.txt`, `Dockerfile`.

> **Did git ask for a username and password?**  
> GitHub no longer accepts passwords for `git push`. You need a Personal Access Token. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token. Give it `repo` scope. Use the token as the password.

---

## Step 9 — Create a Docker Hub Account and Access Token

The CI pipeline needs permission to push images to Docker Hub. We give it that permission via an **access token** (not your real password).

### 9.1 Create a Docker Hub account

If you do not have one:
1. Go to [https://hub.docker.com](https://hub.docker.com)
2. Click **Sign Up**
3. Choose a username (this will be part of your image name, e.g. `johndoe/flask-demo`)
4. Verify your email

### 9.2 Create an Access Token

An access token is like a password, but:
- You can revoke it at any time without changing your real password
- You can give it limited permissions (read-only vs. read/write)

Steps:
1. Sign in to [https://hub.docker.com](https://hub.docker.com)
2. Click your profile picture in the top-right → **"Account Settings"**
3. In the left sidebar, click **"Security"**
4. Click the blue **"New Access Token"** button
5. Fill in the form:
   - **Access Token Description:** `github-actions-flask-demo`
   - **Access permissions:** Select **"Read, Write, Delete"**
6. Click **"Generate"**
7. **IMPORTANT:** A token is shown on screen. Copy it NOW — it will never be shown again.
   It looks like: `dckr_pat_xxxxxxxxxxxxxxxxxxxxxxxxxxxx`
8. Paste it somewhere safe temporarily (Notepad, TextEdit) — you will need it in the next step.

---

## Step 10 — Add Secrets to Your GitHub Repository

Your CI workflow needs two pieces of sensitive information:
- Your Docker Hub username
- Your Docker Hub access token

You must **never put these directly in your code**. Instead, GitHub stores them as encrypted secrets.

### 10.1 Open Repository Settings

1. Go to your repository on GitHub: `https://github.com/YOUR_USERNAME/flask-ci-demo`
2. Click the **"Settings"** tab (last tab in the top navigation bar, with a gear icon)
3. In the left sidebar, find the section called **"Security"**
4. Click **"Secrets and variables"** to expand it
5. Click **"Actions"**

You are now on the **"Actions secrets and variables"** page.

### 10.2 Add the first secret: `DOCKERHUB_USERNAME`

1. Click the green **"New repository secret"** button
2. Fill in:
   - **Name:** `DOCKERHUB_USERNAME`
   - **Secret:** your Docker Hub username (e.g. `johndoe`)
3. Click **"Add secret"**

### 10.3 Add the second secret: `DOCKERHUB_TOKEN`

1. Click **"New repository secret"** again
2. Fill in:
   - **Name:** `DOCKERHUB_TOKEN`
   - **Secret:** paste the access token you copied in Step 9.2 (starts with `dckr_pat_`)
3. Click **"Add secret"**

### 10.4 Verify

You should now see two secrets listed:
```
DOCKERHUB_TOKEN    Updated just now
DOCKERHUB_USERNAME Updated just now
```

> **Can I read the secrets back?** No — GitHub masks the values permanently. If you lose the token, generate a new one on Docker Hub (Step 9.2) and update the secret here.

---

## Step 11 — Create the GitHub Actions Workflow

GitHub Actions workflows must be placed in a specific folder: `.github/workflows/`.

### 11.1 Create the folder

In your terminal (inside `flask-ci-demo`):

```bash
# macOS / Linux
mkdir -p .github/workflows

# Windows (PowerShell)
New-Item -ItemType Directory -Force -Path .github\workflows
```

### 11.2 Create the workflow file

Create a file at `.github/workflows/ci.yml` and paste this exact content:

```yaml
name: CI Pipeline

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  IMAGE_NAME: ${{ secrets.DOCKERHUB_USERNAME }}/flask-demo

jobs:

  # ── Job 1: Install dependencies and run tests ─────────────────────────────
  test:
    name: Build & Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up Python 3.12
        uses: actions/setup-python@v5
        with:
          python-version: "3.12"

      - name: Cache pip packages
        uses: actions/cache@v4
        with:
          path: ~/.cache/pip
          key: pip-${{ hashFiles('requirements.txt') }}

      - name: Install dependencies
        run: pip install -r requirements.txt

      - name: Run tests
        run: pytest test_app.py -v


  # ── Job 2: Build and push Docker image ────────────────────────────────────
  docker:
    name: Docker Build & Push
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'

    steps:
      - name: Checkout source code
        uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push image
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            ${{ env.IMAGE_NAME }}:latest
            ${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Print image info
        run: |
          echo "Image pushed:"
          echo "  ${{ env.IMAGE_NAME }}:latest"
          echo "  ${{ env.IMAGE_NAME }}:${{ github.sha }}"
```

### 11.3 Understanding the key parts

**`on: push: branches: [main]`**  
The pipeline runs automatically whenever you push a commit to the `main` branch.

**`on: pull_request: branches: [main]`**  
The pipeline also runs on pull requests. Tests run, but the Docker push is skipped (because the `if:` condition requires a push event directly to main).

**`needs: test`**  
Job 2 (docker) will not start until Job 1 (test) finishes successfully. If tests fail, Docker Hub never receives a broken image.

**`if: github.ref == 'refs/heads/main' && github.event_name == 'push'`**  
The Docker push only happens on direct pushes to main — not on pull requests. This prevents pushing images for every draft PR.

**`${{ secrets.DOCKERHUB_USERNAME }}`**  
This is how the workflow reads the secrets you set in Step 10. GitHub replaces these with the real values at runtime and masks them in all logs.

**`tags: | ${{ env.IMAGE_NAME }}:latest ${{ env.IMAGE_NAME }}:${{ github.sha }}`**  
Two tags are pushed:
- `:latest` — always points to the newest build
- `:<commit-sha>` — an immutable tag tied to the exact commit that produced it (e.g. `:a3f5c8d...`)

---

## Step 12 — Push and Watch the Pipeline Run

### 12.1 Push the workflow file

```bash
git add .github/workflows/ci.yml
git commit -m "ci: add github actions pipeline"
git push
```

### 12.2 Open the Actions tab

1. Go to your repository on GitHub
2. Click the **"Actions"** tab (between "Pull requests" and "Projects")
3. You should see a workflow run titled **"CI Pipeline"** with an orange spinning circle next to it — this means it is running right now

### 12.3 Watch the run

1. Click on the run title (e.g. `ci: add github actions pipeline`)
2. You see two boxes: **"Build & Test"** and **"Docker Build & Push"**
3. Click **"Build & Test"** to expand it
4. Click any step (e.g. **"Run tests"**) to see the live log output

Expected log output for the test step:
```
test_app.py::test_health_returns_200 PASSED
test_app.py::test_health_returns_ok PASSED
test_app.py::test_index_returns_200 PASSED
test_app.py::test_index_has_all_fields PASSED
test_app.py::test_random_number_in_range PASSED
test_app.py::test_random_word_length PASSED
test_app.py::test_random_word_is_lowercase_alpha PASSED
test_app.py::test_random_name_is_valid PASSED

8 passed in 0.16s
```

5. After Job 1 finishes (green checkmark), Job 2 starts automatically
6. Watch the **"Build and push image"** step — this takes 1–2 minutes on the first run

### 12.4 Both jobs pass

When both jobs show a green checkmark, the full pipeline succeeded. The image is now on Docker Hub.

---

## Step 13 — Verify the Image on Docker Hub

1. Go to [https://hub.docker.com](https://hub.docker.com) and sign in
2. Click **"Repositories"** in the top navigation
3. Click on **`flask-demo`**
4. Click the **"Tags"** tab

You should see two tags:
```
latest        pushed X minutes ago    ~50 MB
a3f5c8d9...   pushed X minutes ago    ~50 MB   ← the commit SHA tag
```

### 13.1 Pull and run the image (optional verification)

```bash
# Pull from Docker Hub
docker pull YOUR_DOCKERHUB_USERNAME/flask-demo:latest

# Run it
docker run -p 8080:8080 YOUR_DOCKERHUB_USERNAME/flask-demo:latest

# Test it (new terminal tab)
curl http://localhost:8080/
curl http://localhost:8080/health
```

---

## What Happens on Every Push

After this setup, every time you push a commit to `main`:

```
git push
    │
    └─► GitHub Actions starts automatically (within ~5 seconds)
          │
          ├─► Job 1: Build & Test
          │     1. Checks out your code
          │     2. Installs Python 3.12
          │     3. Restores cached pip packages (fast if deps unchanged)
          │     4. pip install -r requirements.txt
          │     5. pytest test_app.py -v
          │        ✓ all 8 tests pass → Job 1 succeeds
          │        ✗ any test fails  → pipeline stops, Docker not touched
          │
          └─► Job 2: Docker Build & Push  (only starts after Job 1 succeeds)
                1. Checks out your code
                2. Sets up Docker Buildx
                3. Logs in to Docker Hub (using your secrets)
                4. docker build + docker push
                   Tags: :latest and :<commit-sha>
                5. Prints confirmation

Total time: ~2 minutes for a warm cache, ~4 minutes first time
```

---

## Troubleshooting

### "DOCKERHUB_USERNAME secret not found" or image name is empty

**Cause:** You mistyped the secret name when adding it.  
**Fix:** Go to repository Settings → Secrets and variables → Actions. Delete the secret and recreate it with the exact name `DOCKERHUB_USERNAME` (all caps, underscore).

---

### "unauthorized: incorrect username or password"

**Cause:** The `DOCKERHUB_TOKEN` secret contains the wrong value, or you used your real password instead of an access token.  
**Fix:** Generate a new access token on Docker Hub (Step 9.2). Update the `DOCKERHUB_TOKEN` secret on GitHub (Step 10.3).

---

### "denied: requested access to the resource is denied"

**Cause:** The access token was created with read-only permissions.  
**Fix:** On Docker Hub, delete the old token and create a new one with **"Read, Write, Delete"** permissions.

---

### Tests pass locally but fail in the pipeline

**Cause:** Usually a dependency version mismatch or a missing library.  
**Fix:** Make sure `requirements.txt` lists all libraries your tests use. Run `pip freeze > requirements.txt` locally to capture exact installed versions, then commit the updated file.

---

### "docker: command not found" in the pipeline

**Cause:** This should not happen on `ubuntu-latest` runners — Docker is pre-installed.  
**Fix:** Make sure `runs-on: ubuntu-latest` is set in the `docker` job.

---

### The Docker job is skipped on pull requests

**This is intentional behavior.** The condition `if: github.event_name == 'push'` prevents the Docker push from running on PRs. Only merged commits (direct pushes) to `main` trigger the Docker push.

---

### Pipeline did not start after pushing

**Check these in order:**
1. The workflow file is in `.github/workflows/ci.yml` (double-check the path)
2. The `on: push: branches: [main]` matches your actual branch name (`main` vs `master`)
3. Check the **Actions** tab — there may be a YAML parse error shown there
4. YAML is very sensitive to indentation. Every level of indentation must be exactly 2 spaces, never tabs.

---

### How to check if your YAML is valid

Paste the contents of `ci.yml` into [https://www.yamllint.com](https://www.yamllint.com). It will show you exactly which line has a syntax error.

---

## Glossary

| Term | Plain English |
|------|---------------|
| **CI (Continuous Integration)** | Every code push automatically triggers tests. "Continuous" = happens every time, not just before a release. |
| **CD (Continuous Delivery/Deployment)** | After tests pass, the app is automatically built and shipped. |
| **Pipeline** | A sequence of automated steps that run in order (or in parallel). |
| **Job** | A group of steps that runs on one machine. Multiple jobs can run in parallel. |
| **Step** | A single task inside a job — either a shell command or a reusable action. |
| **Action** | A pre-built, reusable step from the GitHub marketplace. e.g. `actions/checkout@v4` checks out your code. |
| **Runner** | The virtual machine that executes a job. `ubuntu-latest` = a fresh Ubuntu Linux VM. |
| **Secret** | An encrypted variable stored in GitHub. Values are hidden in all logs. |
| **Docker image** | A snapshot of your app and everything it needs to run. Like a ZIP file of a pre-configured computer. |
| **Docker container** | A running instance of a Docker image. Like opening that ZIP file and running the app inside. |
| **Docker Hub** | A public registry where Docker images are stored and shared. Like GitHub, but for Docker images. |
| **Tag** | A label on a Docker image version. `:latest` = newest. `:<sha>` = tied to a specific commit. |
| **SHA** | A unique identifier for a git commit (e.g. `a3f5c8d9`). Every commit gets a different SHA. |
| **`needs:`** | A YAML key that makes one job wait for another to succeed before starting. |
| **`if:`** | A condition that controls whether a job or step runs at all. |
| **`pytest`** | A Python testing framework. It finds functions that start with `test_` and runs them. |
| **`assert`** | A Python statement that checks if something is true. If it is false, the test fails. |
| **`pip`** | Python's package manager. `pip install flask` downloads and installs Flask. |
| **`requirements.txt`** | A list of Python packages your project needs, one per line. |
| **`Dockerfile`** | A recipe for building a Docker image. Each line is one instruction. |
| **Access token** | A password substitute that can be scoped and revoked independently. Safer than your real password. |

---

*End of guide. If something does not work, re-read the Troubleshooting section, then check the Actions tab on GitHub — the error message there usually tells you exactly what went wrong.*
