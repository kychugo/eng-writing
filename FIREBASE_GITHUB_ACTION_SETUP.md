# Setting Up Firebase with GitHub Actions

This guide walks through every step required to deploy Firebase configuration (such as Realtime Database security rules) automatically via GitHub Actions. It covers creating the Firebase project, authenticating the CI environment, configuring the workflow, and verifying that deployments succeed.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Firebase Project Setup](#2-firebase-project-setup)
3. [Initialize Firebase in Your Repository](#3-initialize-firebase-in-your-repository)
4. [Write Your Database Security Rules](#4-write-your-database-security-rules)
5. [Generate a Firebase CI Token](#5-generate-a-firebase-ci-token)
6. [Add the Token as a GitHub Actions Secret](#6-add-the-token-as-a-github-actions-secret)
7. [Create the GitHub Actions Workflow](#7-create-the-github-actions-workflow)
8. [Understanding the Workflow File](#8-understanding-the-workflow-file)
9. [Push and Verify the Deployment](#9-push-and-verify-the-deployment)
10. [Updating the Project ID for a Fork](#10-updating-the-project-id-for-a-fork)
11. [Troubleshooting](#11-troubleshooting)
12. [Security Notes](#12-security-notes)

---

## 1. Prerequisites

Before starting, make sure you have:

| Requirement | Notes |
|---|---|
| **Node.js ≥ 18** | Required to run `firebase-tools` locally. Download from [nodejs.org](https://nodejs.org). |
| **A Google account** | Used to sign in to Firebase and to own the Firebase project. |
| **A GitHub repository** | The repository that will run the GitHub Actions workflow. |
| **Firebase project** | Created in the [Firebase Console](https://console.firebase.google.com). See §2. |

You do **not** need to install `firebase-tools` globally; the workflow uses `npx --yes firebase-tools` to download it on-the-fly inside the runner.

---

## 2. Firebase Project Setup

### 2.1 Create a Firebase project

1. Go to the [Firebase Console](https://console.firebase.google.com).
2. Click **Add project**.
3. Enter a project name (e.g. `my-app-prod`). Firebase generates a unique project ID such as `my-app-prod-a1b2c`.
4. Choose whether to enable Google Analytics (optional for database-only projects).
5. Click **Create project** and wait for provisioning to finish.

### 2.2 Enable the Realtime Database

1. In the left sidebar click **Build → Realtime Database**.
2. Click **Create Database**.
3. Choose the server location closest to your users.
4. Select **Start in locked mode** (recommended — you will define rules in §4).
5. Click **Enable**.

Your database URL will look like:
```
https://my-app-prod-a1b2c-default-rtdb.firebaseio.com
```

### 2.3 Note your project ID

The project ID (e.g. `my-app-prod-a1b2c`) appears in the Firebase Console URL and in **Project settings → General**. You will need it in later steps.

---

## 3. Initialize Firebase in Your Repository

Run the following commands from the root of your local repository clone.

```bash
# Install firebase-tools locally (or use npx if you prefer not to install globally)
npm install -g firebase-tools   # or: npx firebase-tools

# Log in with your Google account
firebase login

# Initialise the project — choose "Database" when prompted
firebase init database
```

The interactive prompt will ask:

| Prompt | What to enter |
|---|---|
| *Which Firebase project?* | Select the project you created in §2 |
| *What file should be used for Realtime Database rules?* | Accept the default `database.rules.json` |

After initialisation, two files are created or updated in your repository root:

### `firebase.json`

Tells the Firebase CLI which features to manage and where their config files live.

```json
{
  "database": {
    "rules": "database.rules.json"
  }
}
```

### `.firebaserc`

Maps the alias `default` to your Firebase project ID.

```json
{
  "projects": {
    "default": "my-app-prod-a1b2c"
  }
}
```

Commit both files:

```bash
git add firebase.json .firebaserc
git commit -m "chore: add Firebase configuration"
```

---

## 4. Write Your Database Security Rules

`database.rules.json` controls who can read and write each path in your Realtime Database. Firebase uses a JSON rule language where `.read` and `.write` define access, and `.validate` enforces data shape.

### Minimal example (authenticated users only)

```json
{
  "rules": {
    ".read": "auth != null",
    ".write": "auth != null"
  }
}
```

### Admin-restricted write example

```json
{
  "rules": {
    "public_data": {
      ".read": true,
      ".write": false
    },
    "private_data": {
      ".read": "auth != null",
      ".write": "auth != null && auth.token.email === 'admin@example.com'"
    }
  }
}
```

### Rule variables reference

| Variable | Meaning |
|---|---|
| `auth` | The authenticated user object; `null` if unauthenticated |
| `auth.uid` | The user's Firebase UID |
| `auth.token.email` | The user's verified email address |
| `data` | The existing data at the current path |
| `newData` | The data that will exist after a write |
| `$variable` | A wildcard that matches any child key at that level |
| `now` | Current server time in milliseconds |

After editing `database.rules.json`, commit the change:

```bash
git add database.rules.json
git commit -m "feat: update Realtime Database security rules"
```

> **Tip:** Test your rules in the Firebase Console under **Realtime Database → Rules → Rules Playground** before deploying.

---

## 5. Generate a Firebase CI Token

GitHub Actions runners cannot complete an interactive browser login. Instead, you generate a long-lived CI token once on your local machine and store it as a secret.

```bash
npx firebase-tools login:ci
```

1. A browser window opens asking you to sign in with Google.
2. Choose the Google account that **owns** the Firebase project.
3. Grant the requested permissions.
4. Return to the terminal. You will see output similar to:

```
✔  Success! Use this token to login on a CI server:

1//0gXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

5. **Copy the entire token string** (starting with `1//`). Treat it like a password — anyone with this token can deploy to your Firebase project.

---

## 6. Add the Token as a GitHub Actions Secret

Secrets are encrypted environment variables stored in GitHub and injected into workflow runs at runtime. They are never visible in logs.

1. Go to your repository on GitHub.
2. Click **Settings** (top navigation bar).
3. In the left sidebar click **Secrets and variables → Actions**.
4. Click **New repository secret**.
5. Fill in the fields:
   - **Name:** `FIREBASE_TOKEN`
   - **Secret:** paste the full token string from §5
6. Click **Add secret**.

The secret is now available to all workflows in the repository as `${{ secrets.FIREBASE_TOKEN }}`.

---

## 7. Create the GitHub Actions Workflow

Create the workflow file at `.github/workflows/deploy.yml`. If the `.github/workflows/` directory does not exist, create it.

```bash
mkdir -p .github/workflows
```

Paste the following into `.github/workflows/deploy.yml`, replacing `YOUR_PROJECT_ID` with your actual Firebase project ID:

```yaml
name: Deploy

on:
  push:
    branches:
      - main
  workflow_dispatch:         # allows manual runs from the Actions tab

permissions:
  contents: read
  pages: write      # required by actions/deploy-pages
  id-token: write   # required by actions/deploy-pages

concurrency:
  group: deploy
  cancel-in-progress: false  # never cancel an in-progress deployment

jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest

    steps:
      # ── 1. Check out the repository ─────────────────────────────────────────
      - name: Checkout
        uses: actions/checkout@v4

      # ── 2. Deploy static site to GitHub Pages ───────────────────────────────
      - name: Setup Pages
        uses: actions/configure-pages@v5

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: '.'

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4

      # ── 3. Deploy Firebase Realtime Database rules ───────────────────────────
      - name: Deploy Firebase Database Rules
        run: npx --yes firebase-tools deploy --only database --project YOUR_PROJECT_ID
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

If you do not use GitHub Pages, remove steps 2–4 (Setup Pages, Upload artifact, Deploy to GitHub Pages) along with the `permissions`, `environment`, and `concurrency` blocks, and simplify the job to:

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Deploy Firebase Database Rules
        run: npx --yes firebase-tools deploy --only database --project YOUR_PROJECT_ID
        env:
          FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

Commit and push:

```bash
git add .github/workflows/deploy.yml
git commit -m "ci: add deploy workflow with Firebase database rules deployment"
git push origin main
```

---

## 8. Understanding the Workflow File

### Trigger (`on`)

```yaml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

- The workflow runs automatically on every push to `main`.
- `workflow_dispatch` adds a **Run workflow** button in the Actions tab for manual runs.

### Permissions

```yaml
permissions:
  contents: read
  pages: write      # required by actions/deploy-pages
  id-token: write   # required by actions/deploy-pages
```

These permissions are needed only for the GitHub Pages deployment steps. If you only need Firebase deployment (no GitHub Pages), reduce this to `contents: read` or remove the block entirely (GitHub Actions defaults to `contents: read`).

### Concurrency

```yaml
concurrency:
  group: deploy
  cancel-in-progress: false
```

Ensures that only one deployment runs at a time. The `group` name (`deploy`) is arbitrary — it just needs to be consistent across workflow runs. Setting `cancel-in-progress: false` prevents a new push from cancelling a deployment that is already in progress (important to avoid partial deployments).

### The Firebase deployment step

```yaml
- name: Deploy Firebase Database Rules
  run: npx --yes firebase-tools deploy --only database --project YOUR_PROJECT_ID
  env:
    FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

| Part | Explanation |
|---|---|
| `npx --yes firebase-tools` | Downloads and runs the latest `firebase-tools` without prompting for confirmation |
| `deploy --only database` | Deploys only the Realtime Database rules, not Hosting or other Firebase features |
| `--project YOUR_PROJECT_ID` | Explicitly targets the project (avoids relying on `.firebaserc` alone) |
| `FIREBASE_TOKEN` | The CI token from §5, injected from GitHub Secrets |

### Deploying multiple Firebase features

To deploy Hosting rules, Firestore rules, or Storage rules in the same step, extend the `--only` flag:

```yaml
run: npx --yes firebase-tools deploy --only database,firestore,storage --project YOUR_PROJECT_ID
```

---

## 9. Push and Verify the Deployment

### 9.1 Trigger the workflow

After pushing `.github/workflows/deploy.yml` to `main`, navigate to your repository on GitHub and click the **Actions** tab. You should see a workflow run named **Deploy** (or the name you gave it) in progress.

### 9.2 Read the run logs

Click into the run to expand each step. A successful Firebase deployment step looks like:

```
✔  Database: rules from database.rules.json
✔  Deploy complete!

Project Console: https://console.firebase.google.com/project/my-app-prod-a1b2c/overview
```

### 9.3 Confirm in the Firebase Console

1. Open the [Firebase Console](https://console.firebase.google.com).
2. Select your project.
3. Go to **Build → Realtime Database → Rules**.
4. The rules displayed should match the content of your `database.rules.json`.

### 9.4 Workflow status badges (optional)

Add a status badge to your README to show the current build status:

```markdown
![Deploy](https://github.com/YOUR_USERNAME/YOUR_REPO/actions/workflows/deploy.yml/badge.svg)
```

---

## 10. Updating the Project ID for a Fork

If you fork this repository, you must update the project ID in two places:

**`.firebaserc`**

```json
{
  "projects": {
    "default": "your-forked-project-id"
  }
}
```

**`.github/workflows/deploy.yml`**

```yaml
- name: Deploy Firebase Database Rules
  run: npx --yes firebase-tools deploy --only database --project your-forked-project-id
  env:
    FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

Then repeat §5 and §6 to generate a new CI token for your Google account and add it as the `FIREBASE_TOKEN` secret in your forked repository.

---

## 11. Troubleshooting

### `Error: Failed to get Firebase project` / `HTTP Error: 403`

**Cause:** The `FIREBASE_TOKEN` is invalid, expired, or belongs to an account that does not have access to the project.

**Fix:**
1. Re-run `npx firebase-tools login:ci` locally.
2. Delete the old `FIREBASE_TOKEN` secret in GitHub (Settings → Secrets → Actions).
3. Add the new token as `FIREBASE_TOKEN`.

---

### `Error: Must supply a non-empty --project`

**Cause:** The `--project` flag is missing or the project ID is the literal placeholder `YOUR_PROJECT_ID`.

**Fix:** Open `.github/workflows/deploy.yml` and replace `YOUR_PROJECT_ID` with your real Firebase project ID.

---

### `Permission denied` when deploying database rules

**Cause:** The Google account associated with the token does not have the **Firebase Realtime Database Admin** role.

**Fix:**
1. Go to the [Google Cloud Console IAM page](https://console.cloud.google.com/iam-admin/iam) for your project.
2. Find the account used to generate the token.
3. Add the role **Firebase Realtime Database Admin** (or **Firebase Admin**).
4. Re-run the workflow.

---

### Workflow does not trigger on push

**Cause:** The branch name in the workflow does not match your default branch.

**Fix:** Check whether your default branch is `main` or `master`:

```bash
git branch -a
```

Update the `branches` list in the workflow if needed:

```yaml
on:
  push:
    branches:
      - master   # change to match your default branch
```

---

### `firebase-tools` version conflicts

**Cause:** `npx --yes firebase-tools` always fetches the latest version, which may introduce breaking changes.

**Fix:** Pin to a specific version in the workflow:

```yaml
run: npx --yes firebase-tools@13 deploy --only database --project YOUR_PROJECT_ID
```

Check the latest stable version at [npmjs.com/package/firebase-tools](https://www.npmjs.com/package/firebase-tools).

---

### Firebase step runs but rules are not updated in the Console

**Cause:** Browser cache. Firebase Console may show cached rules.

**Fix:** Hard-refresh the Firebase Console page (`Ctrl+Shift+R` / `Cmd+Shift+R`) or open it in an incognito window.

---

## 12. Security Notes

- **Never commit the CI token** (`FIREBASE_TOKEN`) to your repository. Always store it as a GitHub Actions secret.
- **Rotate the token** periodically or if you suspect it has been exposed. Run `npx firebase-tools login:ci` again and update the secret.
- **Use least-privilege rules** in `database.rules.json`. Avoid `".read": true` or `".write": true` at the root level unless your data is intentionally public.
- **Review open access paths** before each deploy. Consider adding a `firebase deploy --dry-run` step (if supported by your CLI version) to preview rule changes before they take effect.
- **Use environment-specific projects** (e.g. separate Firebase projects for development and production) to avoid accidental overwrites.

---

*For the full Firebase Realtime Database rules reference, see the [official documentation](https://firebase.google.com/docs/database/security).*
