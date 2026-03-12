# The Writer's Desk

An AI-powered English writing and vocabulary platform for HKDSE students. Practice essay writing, build vocabulary, check grammar, and receive instant AI-generated feedback — all in one place.

**Live site:** [kychugo.github.io/eng-writing](https://kychugo.github.io/eng-writing/)

---

## Overview

The Writer's Desk is a static web application built for Hong Kong secondary students preparing for the HKDSE English Language examination. It combines a bank of 99 past exam writing questions with a suite of AI-powered writing and vocabulary tools, offering structured practice without requiring any account or installation.

---

## Features

- **Writing question bank** — 99 authentic HKDSE writing prompts (articles, emails, blogs, proposals, and more) drawn from past exam papers
- **AI writing planner** — generates essay outlines and planning frameworks from a topic or question
- **Grammar checker** — identifies and explains grammar errors, typos, fluency issues, and punctuation mistakes with before/after comparison
- **Text enhancer** — improves vocabulary, sentence variety, and overall writing style
- **OCR (image to text)** — upload a photo or scan of handwritten work and extract the text for analysis
- **Vocabulary generator** — produces themed word lists with definitions and example sentences
- **Vocabulary quiz** — three interactive formats: multiple choice, sentence completion, and matching pairs
- **Spaced repetition (SRS) flipcards** — review vocabulary with a four-level difficulty rating to prioritise words that need more practice
- **Mistake notebook** — saves grammar errors for ongoing review, stored locally in the browser
- **Export** — download AI feedback or vocabulary lists as PDF or Word documents
- **Progressive Web App** — installable on mobile devices for offline access to saved content

---

## Pages

| File | Purpose |
|------|---------|
| `index.html` | Main dashboard — all writing and vocabulary tools integrated in one interface |
| `Orionai.html` | Standalone HKDSE writing suite — planner, grammar checker, text enhancer, and OCR |
| `vocab.html` | Dedicated vocabulary platform — generator, quizzes, and SRS flipcards |

All three pages are independent and can be used separately.

---

## Tool Reference

### Generate

**Writing Questions** — Browse or randomly select a prompt from the question bank. Questions are tagged by year and text type, allowing targeted practice.

### Write

**Writing Planner** — Enter a question or topic; the AI returns a structured outline with a thesis, supporting points, and suggested examples.

**Grammar Check** — Paste a draft; the AI returns annotated feedback highlighting errors by category (grammar, typo, fluency, punctuation, style) with corrections and explanations. Each error links to a detailed popover.

**Text Enhancer** — Paste a paragraph or essay; the AI rewrites it with improved vocabulary and sentence structure, preserving the original meaning and voice.

**OCR** — Upload an image of handwritten or printed text. Use the crop tool to select the relevant area before extraction.

### Learn

**Vocabulary Generator** — Enter a topic or paste a word list; the AI generates entries following the format: word, part of speech, definition, example sentence, and notes.

**Vocabulary Quiz** — Choose a format (MCQ, sentence completion, or matching) and take an auto-generated quiz on your saved vocabulary.

**Flipcard Review (SRS)** — Review word cards and rate each one (Forgot / Hard / OK / Easy). The system tracks your ratings and surfaces weaker words more frequently.

### Notebook

**Mistake Notebook** — Grammar check errors are saved here automatically. Review past mistakes, filter by error type, and mark items as resolved.

---

## Technology Stack

| Layer | Details |
|-------|---------|
| Frontend | HTML5, CSS3, vanilla JavaScript |
| AI API | [Pollinations.ai](https://pollinations.ai) chat completions endpoint |
| AI models | `openai-fast` (primary), `openai` (fallback), `gemini-search` (search-enabled) |
| Image processing | [Cropper.js](https://github.com/fengyuanchen/cropperjs) |
| Export | [jsPDF](https://github.com/parallax/jsPDF), [html2canvas](https://github.com/niklasvh/html2canvas), [html-to-docx](https://github.com/privateOmega/html-to-docx) |
| Fonts | Cormorant Garamond, Lato, IM Fell English (Google Fonts) |
| Storage | Browser `localStorage` (no server-side persistence) |
| Deployment | GitHub Pages (auto-deployed from `main` via GitHub Actions) |

The application is entirely client-side. No backend, database, or build step is required.

---

## File Structure

```
eng-writing/
├── index.html        # Main application
├── Orionai.html      # Standalone writing suite
├── vocab.html        # Standalone vocabulary platform
├── qlist.json        # HKDSE writing question bank (99 questions)
├── manifest.json     # PWA manifest
├── favicon.svg       # Site icon (SVG)
├── icon-192.png      # PWA icon (192x192)
├── icon-512.png      # PWA icon (512x512)
├── style.md          # Personal writing style guide (AI system prompt reference)
├── APIDOCS.md        # Pollinations.ai API documentation
└── README.md         # This file
```

---

## Running Locally

No build step or package manager is required. Clone the repository and open `index.html` directly in any modern browser, or serve the folder with a local HTTP server:

```bash
# Using Python (built-in)
python -m http.server 8080

# Using Node.js (npx)
npx serve .
```

Then visit `http://localhost:8080` in your browser.

An internet connection is required for AI features (Pollinations.ai API calls).

---

## Deployment

The site deploys automatically to GitHub Pages whenever a commit is pushed to `main`. The workflow is defined in `.github/workflows/`. No configuration is needed beyond pushing to the `main` branch.

To deploy a fork, enable GitHub Pages in the repository settings and point it at the `main` branch (root directory).

---

## GitHub Actions Setup Guide

The repository uses a single GitHub Actions workflow (`.github/workflows/deploy.yml`) that does two things on every push to `main`:

1. **Deploys the static site to GitHub Pages** (no configuration needed once Pages is enabled).
2. **Deploys the Firebase Realtime Database security rules** from `database.rules.json`.

### One-time setup for a fork or new repository

#### Step 1 — Enable GitHub Pages

1. Go to your repository on GitHub.
2. Click **Settings → Pages**.
3. Under *Source*, choose **GitHub Actions**.
4. Save. GitHub will create the necessary `GITHUB_TOKEN` permissions automatically.

#### Step 2 — Get a Firebase CI token

The workflow uses `firebase-tools` via `npx` to push database rules. It authenticates with a long-lived CI token stored as a GitHub Actions secret.

Run this command locally (Node.js required):

```bash
npx firebase-tools login:ci
```

Follow the browser prompt to sign in with the Google account that owns the Firebase project. The command prints a token that looks like:

```
✔  Success! Use this token to login on a CI server:

1//XXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXXX
```

Copy the entire token string.

#### Step 3 — Add the token as a GitHub Actions secret

1. Go to your repository on GitHub.
2. Click **Settings → Secrets and variables → Actions**.
3. Click **New repository secret**.
4. Set the name to `FIREBASE_TOKEN` and paste the token as the value.
5. Click **Add secret**.

#### Step 4 — Update the Firebase project ID (if forking)

Open `.github/workflows/deploy.yml` and update the `--project` flag to match your own Firebase project ID:

```yaml
- name: Deploy Firebase Database Rules
  run: npx --yes firebase-tools deploy --only database --project YOUR_PROJECT_ID
  env:
    FIREBASE_TOKEN: ${{ secrets.FIREBASE_TOKEN }}
```

Also update `.firebaserc` with your project ID:

```json
{
  "projects": {
    "default": "YOUR_PROJECT_ID"
  }
}
```

#### Step 5 — Push to `main`

Once the secret is in place, push any commit to `main`. The **Actions** tab will show two steps running:

| Step | What it does |
|------|-------------|
| Deploy to GitHub Pages | Builds and publishes the static site |
| Deploy Firebase Database Rules | Runs `firebase deploy --only database` to push `database.rules.json` |

Both steps must show a green ✓ for the deployment to be complete. If the Firebase step fails, check that `FIREBASE_TOKEN` is set correctly and that the service account has the **Firebase Realtime Database Admin** role in the Firebase console.

---

## Question Bank

`qlist.json` contains 99 HKDSE writing questions covering:

- **Text types:** articles, emails, blogs, proposals, letters, reports, notices
- **Years:** multiple past exam papers
- **Format:** each entry includes the year, question number, full question text, and text type tag

The question bank is used by the Writing Questions tool in the Generate section to display and randomise prompts.

---

## Writing Style Guide

`style.md` is a prompt-ready reference defining the preferred writing style for this project. It is intended to be pasted as a system prompt into any AI platform to ensure consistent output. It covers voice, register, sentence structure, vocabulary choices, punctuation, and formatting rules.

---

## Licence

This project does not currently include a licence file. All rights are reserved by the author unless otherwise stated.