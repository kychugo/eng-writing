# Firebase + GitHub Actions Setup Guide

This guide explains, step by step, how to connect a Firebase project to a GitHub Actions workflow so that your Firebase Realtime Database security rules are automatically deployed every time you push to `main`. It is written specifically for the configuration used in this repository but is general enough to apply to any static-site project that uses Firebase RTDB alongside GitHub Pages.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Firebase Project Setup](#2-firebase-project-setup)
3. [Repository Configuration Files](#3-repository-configuration-files)
   - [firebase.json](#firebasejson)
   - [.firebaserc](#firebaserc)
   - [database.rules.json](#databaserulesjson)
4. [Getting a Firebase CI Token](#4-getting-a-firebase-ci-token)
5. [Adding the Token as a GitHub Actions Secret](#5-adding-the-token-as-a-github-actions-secret)
6. [The GitHub Actions Workflow File](#6-the-github-actions-workflow-file)
   - [Full Annotated YAML](#full-annotated-yaml)
   - [Step-by-Step Explanation](#step-by-step-explanation)
7. [Enabling GitHub Pages](#7-enabling-github-pages)
8. [Verifying the Deployment](#8-verifying-the-deployment)
9. [Troubleshooting](#9-troubleshooting)
10. [Security Best Practices](#10-security-best-practices)

---

## 1. Prerequisites

Before you begin, make sure you have the following:

| Requirement | Why it is needed |
|-------------|-----------------|
| A GitHub account | To host the repository and run Actions |
| A Google account | To create and own the Firebase project |
| [Node.js](https://nodejs.org/) ≥ 18 (LTS) installed locally | Required to run `npx firebase-tools` |
| A Firebase project | The RTDB and its rules live inside a Firebase project |
| Your repository pushed to GitHub | The workflow file must exist on the `main` branch |

> **Tip:** You do not need to install `firebase-tools` globally. Every command in this guide uses `npx --yes firebase-tools …`, which downloads the latest version on demand.

---

## 2. Firebase Project Setup

### 2a. Create a Firebase project

1. Open [https://console.firebase.google.com](https://console.firebase.google.com) and sign in.
2. Click **Add project**.
3. Enter a project name (e.g. `my-writing-app`). Firebase generates a unique **Project ID** (e.g. `my-writing-app-1a2b3c4d`). Note this ID — you will need it later.
4. Choose whether to enable Google Analytics (optional for RTDB-only projects) and click **Create project**.

### 2b. Enable the Realtime Database

1. In the left sidebar, click **Build → Realtime Database**.
2. Click **Create Database**.
3. Choose a database location (the closest region to your users).
4. Select **Start in locked mode** — the workflow will push your own rules immediately after setup.
5. Click **Enable**.

### 2c. Grant the service account the correct role

The CI token you will generate later is tied to your Google account, so the account that owns the project already has full admin access. If you are using a **service account** instead of `login:ci`, grant it the **Firebase Realtime Database Admin** role:

1. Open [https://console.cloud.google.com/iam-admin/iam](https://console.cloud.google.com/iam-admin/iam) for your project.
2. Find the service account (usually `firebase-adminsdk-xxxxx@PROJECT_ID.iam.gserviceaccount.com`).
3. Click the pencil icon and add the role **Firebase Realtime Database Admin**.
4. Save.

---

## 3. Repository Configuration Files

Three files in the repository root tell `firebase-tools` what to deploy and to which project.

### `firebase.json`

This file defines which Firebase features to deploy and where to find their configuration.

```json
{
  "database": {
    "rules": "database.rules.json"
  }
}
```

| Key | Value | Meaning |
|-----|-------|---------|
| `database` | object | Activates the Realtime Database deployer |
| `rules` | `"database.rules.json"` | Path to the security rules file, relative to `firebase.json` |

If you want to deploy Firestore rules, Cloud Functions, or Hosting at the same time, add those sections here. For this project only `database` rules are deployed.

### `.firebaserc`

This file maps the alias `default` to a Firebase Project ID so you can run `firebase deploy` without specifying `--project` every time.

```json
{
  "projects": {
    "default": "YOUR_PROJECT_ID"
  }
}
```

Replace `YOUR_PROJECT_ID` with the Project ID you noted in [Step 2a](#2a-create-a-firebase-project).

> **Note:** The workflow also passes `--project YOUR_PROJECT_ID` explicitly as a safety net so that the correct project is targeted even if `.firebaserc` is missing or incorrect.

### `database.rules.json`

This file contains the Firebase Realtime Database security rules in JSON format. Rules control which users can read or write which paths.

```json
{
  "rules": {
    "public_path": {
      ".read": true,
      ".write": false
    },
    "private_path": {
      ".read": "auth != null",
      ".write": "auth != null && auth.token.email === 'admin@example.com'"
    }
  }
}
```

Common rule primitives:

| Expression | Meaning |
|-----------|---------|
| `true` | Allow everyone (including unauthenticated users) |
| `false` | Deny everyone |
| `auth != null` | Allow any signed-in user |
| `auth.uid === $uid` | Allow only the user whose UID matches the path variable |
| `auth.token.email === 'x@y.com'` | Allow only the user with that exact email |

Every time you edit `database.rules.json` and push to `main`, the GitHub Actions workflow automatically pushes the updated rules to Firebase — no manual `firebase deploy` is needed.

---

## 4. Getting a Firebase CI Token

GitHub Actions cannot open a browser to complete the interactive OAuth login. Instead, `firebase-tools` offers a special command that generates a long-lived token you can store as a secret.

### Step-by-step

1. Open a terminal on your local machine.
2. Run:

   ```bash
   npx --yes firebase-tools login:ci
   ```

3. A browser window opens asking you to sign in with Google. Sign in with the account that owns the Firebase project.
4. After you authorise the CLI, the terminal prints:

   ```
   ✔  Success! Use this token to login on a CI server:

   1//XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
   ```

5. **Copy the entire token** (starting with `1//`). You will paste it into GitHub in the next step.

> **Important:** Treat this token like a password. It grants programmatic access to all Firebase projects owned by the account. Do not commit it to source code, print it in logs, or share it publicly.

---

## 5. Adding the Token as a GitHub Actions Secret

GitHub Actions secrets are encrypted environment variables that are injected into workflow runs without appearing in logs.

### Step-by-step

1. Navigate to your repository on GitHub.
2. Click **Settings** (the gear icon in the top navigation bar).
3. In the left sidebar, click **Secrets and variables → Actions**.
4. Click **New repository secret**.
5. Fill in the form:
   - **Name:** `FIREBASE_TOKEN`
   - **Secret:** paste the token you copied in Step 4
6. Click **Add secret**.

The secret is now stored encrypted. The workflow can reference it as `${{ secrets.FIREBASE_TOKEN }}`.

> **Rotating the token:** If you ever suspect the token is compromised, run `npx firebase-tools logout` and then `npx firebase-tools login:ci` again to generate a new one, then update the secret in GitHub.

---

## 6. The GitHub Actions Workflow File

The workflow file lives at `.github/workflows/deploy.yml`. It runs automatically on every push to `main` and on manual trigger.

### Full Annotated YAML

```yaml
# ─── Workflow name (shown in the Actions tab) ────────────────────────────────
name: Deploy to GitHub Pages

# ─── Triggers ─────────────────────────────────────────────────────────────────
on:
  push:
    branches:
      - main          # Run on every push to main
  workflow_dispatch:  # Also allow manual runs from the Actions tab

# ─── Permissions ──────────────────────────────────────────────────────────────
# These grant the GITHUB_TOKEN the minimum access needed for Pages deployment.
permissions:
  contents: read    # Read repo files
  pages: write      # Write to GitHub Pages
  id-token: write   # Required for OIDC-based Pages deployment

# ─── Concurrency ──────────────────────────────────────────────────────────────
# Only one Pages deployment runs at a time; new runs queue rather than cancel.
concurrency:
  group: pages
  cancel-in-progress: false

# ─── Jobs ─────────────────────────────────────────────────────────────────────
jobs:
  deploy:
    # Tie this job to the GitHub Pages environment so the URL is shown in the UI
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}

    runs-on: ubuntu-latest   # Use the latest Ubuntu runner

    steps:
      # 1. Check out the repository at the commit that triggered the workflow
      - name: Checkout
        uses: actions/checkout@v4

      # 2. Configure the Pages environment (sets up required metadata)
      - name: Setup Pages
        uses: actions/configure-pages@v5

      # 3. Upload the entire repository root as the Pages artifact
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.'   # Deploy everything from the root directory

      # 4. Publish the uploaded artifact to GitHub Pages
      - name: Deploy to GitHub Pages
        id: deployment            # ID lets later steps reference outputs
        uses: actions/deploy-pages@v4

      # 5. Push the Firebase Realtime Database security rules
      #    Replace YOUR_PROJECT_ID with your actual Firebase project ID (e.g. my-app-1a2b3c4d)
      - name: Deploy Firebase Database Rules
        run: npx --yes firebase-tools deploy --only database --project YOUR_PROJECT_ID
        env:
          # Inject the CI token from the repository secret
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

### Step-by-Step Explanation

| Step | Action | Key detail |
|------|--------|-----------|
| **Checkout** | `actions/checkout@v4` | Clones the repo at the triggering commit into the runner workspace |
| **Setup Pages** | `actions/configure-pages@v5` | Reads repository settings and prepares the Pages upload environment |
| **Upload artifact** | `actions/upload-pages-artifact@v3` | Packages the directory at `path: '.'` into a compressed artifact and registers it for deployment |
| **Deploy to GitHub Pages** | `actions/deploy-pages@v4` | Takes the uploaded artifact and publishes it; outputs `page_url` |
| **Deploy Firebase Database Rules** | `npx --yes firebase-tools deploy` | Downloads `firebase-tools` on the fly, reads `firebase.json` and `database.rules.json`, authenticates with `FIREBASE_TOKEN`, and pushes the rules to the specified Firebase project |

#### Why `npx --yes`?

`npx --yes` downloads and runs `firebase-tools` from npm without prompting for confirmation. This avoids the need to install the package globally in the workflow or add it to a `package.json`. The `--yes` flag answers "yes" to all prompts automatically.

#### Why `--only database`?

The `--only database` flag restricts the deploy to the Realtime Database rules only. Without it, `firebase deploy` would attempt to deploy every feature listed in `firebase.json` (Hosting, Functions, Firestore, etc.), which could fail or cause unintended changes if those features are not configured.

#### Why `--project YOUR_PROJECT_ID`?

Explicitly specifying the project prevents the tool from accidentally deploying to the wrong Firebase project if `.firebaserc` is missing or if the runner has cached credentials from a previous run.

---

## 7. Enabling GitHub Pages

The Pages deployment step only works if GitHub Pages is enabled for your repository.

1. Go to your repository on GitHub.
2. Click **Settings → Pages**.
3. Under **Source**, select **GitHub Actions** (not "Deploy from a branch").
4. Click **Save**.

That is all. The `GITHUB_TOKEN` permissions declared in the workflow (`pages: write`, `id-token: write`) handle everything else automatically.

---

## 8. Verifying the Deployment

### Check the Actions run

1. Go to your repository on GitHub.
2. Click the **Actions** tab.
3. Click the most recent **Deploy to GitHub Pages** workflow run.
4. Expand each step and confirm it shows a green ✓.

A successful run looks like:

```
✔  Checkout                         ~1s
✔  Setup Pages                      ~1s
✔  Upload artifact                  ~5s
✔  Deploy to GitHub Pages           ~15s
✔  Deploy Firebase Database Rules   ~20s
```

### Check the Firebase console

1. Open [https://console.firebase.google.com](https://console.firebase.google.com) and select your project.
2. Click **Build → Realtime Database → Rules**.
3. Confirm the rules shown match the contents of your `database.rules.json`.

### Check the live site

The URL is printed in the **Deploy to GitHub Pages** step output and also shown in **Settings → Pages**. Open it in a browser and confirm the site loads.

---

## 9. Troubleshooting

### ❌ `Error: FIREBASE_TOKEN not set`

The workflow could not find the secret.

- Confirm the secret is named exactly `FIREBASE_TOKEN` (case-sensitive) in **Settings → Secrets and variables → Actions**.
- If you are working in a fork, secrets from the parent repository are not inherited. Add the secret to your fork separately.

### ❌ `Error: Invalid project ID`

The `--project` flag value does not match any Firebase project your account can access.

- Open `.github/workflows/deploy.yml` and verify the `--project` value matches the **Project ID** shown in the Firebase console (e.g. `my-app-a1b2c`), not the display name.
- Confirm you signed in with `login:ci` using the Google account that owns that Firebase project.

### ❌ `Error: Permission denied`

The account associated with `FIREBASE_TOKEN` does not have permission to deploy database rules.

- Go to **Firebase console → Project settings → Service accounts** or use the Google Cloud IAM page to verify the account has at minimum the **Firebase Realtime Database Admin** role.
- If you rotated your Google account password, the token may have been revoked. Run `npx firebase-tools login:ci` again and update the secret.

### ❌ GitHub Pages shows a 404

- Confirm Pages is set to use **GitHub Actions** as the source (not "Deploy from a branch").
- Check that the **Upload artifact** step succeeded — if it failed, there is nothing to deploy.
- Ensure the `permissions` block in the workflow includes `pages: write` and `id-token: write`.

### ❌ `Your token has expired`

CI tokens can be revoked if you sign out of all Google sessions or if the token has been unused for an extended period.

Run the following to generate a fresh token:

```bash
npx firebase-tools login:ci
```

Then update the `FIREBASE_TOKEN` secret in **Settings → Secrets and variables → Actions**.

### ❌ `Cannot find module 'firebase-tools'`

This should not occur with `npx --yes`, but if it does, try adding a dedicated install step:

```yaml
- name: Install firebase-tools
  run: npm install -g firebase-tools

- name: Deploy Firebase Database Rules
  run: firebase deploy --only database --project YOUR_PROJECT_ID
  env:
    FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

---

## 10. Security Best Practices

### Keep secrets out of source code

Never hardcode the `FIREBASE_TOKEN` or any API key in a file that gets committed to the repository. Use GitHub Actions secrets exclusively.

### Use `--only database`

Always scope `firebase deploy` with `--only <feature>` to prevent unintended side effects. Deploying all features at once from CI can overwrite production Hosting content or Functions unexpectedly.

### Apply the principle of least privilege in database rules

Default to denying access and only open paths that need to be readable or writable. The pattern below is a safe starting point:

```json
{
  "rules": {
    ".read": false,
    ".write": false,
    "public_data": {
      ".read": true
    },
    "user_data": {
      "$uid": {
        ".read": "auth != null && auth.uid === $uid",
        ".write": "auth != null && auth.uid === $uid"
      }
    }
  }
}
```

### Review rules changes in pull requests

Because the workflow only runs on pushes to `main`, open a pull request for any change to `database.rules.json`. Review the diff before merging to avoid accidentally opening or closing data access.

### Rotate the CI token periodically

Generate a new token every few months with `npx firebase-tools login:ci` and update the GitHub secret. This limits the window of exposure if the token is ever compromised.

### Use branch protection on `main`

Require at least one pull request review and passing status checks before merging to `main`. This prevents a single committer from accidentally pushing rules changes that lock everyone out.

---

## Quick Reference

```bash
# Generate a new CI token (run locally)
npx --yes firebase-tools login:ci

# Test a deploy locally before pushing (uses your personal login, not the token)
# Replace YOUR_PROJECT_ID with your actual Firebase project ID (e.g. my-app-1a2b3c4d)
npx --yes firebase-tools deploy --only database --project YOUR_PROJECT_ID

# Validate rules syntax without deploying
# Replace YOUR_PROJECT_ID with your actual Firebase project ID (e.g. my-app-1a2b3c4d)
npx --yes firebase-tools database:rules:get --project YOUR_PROJECT_ID
```

| File | Purpose |
|------|---------|
| `.github/workflows/deploy.yml` | Defines the CI/CD pipeline |
| `firebase.json` | Tells `firebase-tools` what to deploy |
| `.firebaserc` | Maps the `default` alias to your Project ID |
| `database.rules.json` | The Realtime Database security rules |

---

*For the official Firebase documentation see [firebase.google.com/docs/database/security](https://firebase.google.com/docs/database/security). For GitHub Actions documentation see [docs.github.com/en/actions](https://docs.github.com/en/actions).*
