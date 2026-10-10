# Geth Beginner's Guide Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create and publish a public GitHub repository containing a clean Markdown edition of the local Geth Word notes, its images, and a CC BY 4.0 license.

**Architecture:** Keep the Word document outside Git. Use a landing-page `README.md`, numbered chapter files under `chapters/`, and chapter-scoped images under `assets/`. Future sync requests repeat the conversion, review, verification, commit, and push flow.

**Tech Stack:** Git, GitHub CLI, Pandoc, GitHub-flavored Markdown, PNG assets

---

### Task 1: Record the repository design

**Files:**
- Create: `docs/superpowers/specs/2026-10-10-geth-beginners-guide-design.md`
- Create: `docs/superpowers/plans/2026-10-10-geth-beginners-guide.md`

- [x] **Step 1: Review the design and implementation plan**

Run: `rg -n '\x54\x42\x44|\x54\x4f\x44\x4f' docs/superpowers || true`

Expected: no output.

- [x] **Step 2: Commit the repository workflow documents**

Run: `git add docs/superpowers && git commit -m "docs: define publishing workflow"`

Expected: one root commit containing only the specification and plan.

### Task 2: Add project metadata

**Files:**
- Create: `README.md`
- Create: `LICENSE`
- Create: `.gitignore`

- [x] **Step 1: Create a Chinese landing page**

Add the project title, purpose, current chapter link, update status, and CC BY 4.0 attribution notice to `README.md`.

- [x] **Step 2: Add the CC BY 4.0 legal text**

Place the official Creative Commons Attribution 4.0 International legal code in `LICENSE`.

- [x] **Step 3: Prevent accidental publication of Word sources**

Add `*.docx`, Word lock files, and common macOS metadata files to `.gitignore`.

- [x] **Step 4: Verify and commit metadata**

Run: `git check-ignore --no-index sample.docx`

Expected: `sample.docx` is reported as ignored; no `.docx` is staged.

Run: `git add README.md LICENSE .gitignore && git commit -m "docs: add project overview and license"`

Expected: a commit containing the landing page, license, and ignore rules.

### Task 3: Convert the initial Word notes

**Files:**
- Create: `chapters/01-running-a-local-geth-dev-chain.md`
- Create: `assets/chapter-01/ethereum-node-architecture.png`
- Create: `assets/chapter-01/geth-console-connected.png`
- Create: `assets/chapter-01/block-number.png`
- Create: `assets/chapter-01/genesis-block.png`
- Create: `assets/chapter-01/get-block-request-flow.png`
- Create: `assets/chapter-01/development-account.png`
- Create: `assets/chapter-01/development-account-balance.png`

- [x] **Step 1: Extract the Word content and images**

Run Pandoc with GitHub-flavored Markdown output and a temporary media directory. Keep the source document unchanged.

- [x] **Step 2: Normalize the chapter Markdown**

Create a single H1 chapter title, meaningful H2/H3 headings, fenced shell and JavaScript examples, a valid parameter table, relative image paths, and descriptive alt text. Preserve the author's claims and examples.

- [x] **Step 3: Copy and rename the seven PNG assets**

Map images to the descriptive filenames listed above without recompression.

- [x] **Step 4: Verify the converted chapter**

Run checks that assert seven image references, seven existing PNG files, no absolute local filesystem paths, no empty headings, no `.docx` files under version control, and no private-key or mnemonic phrases in tracked text.

Expected: all checks exit successfully.

- [x] **Step 5: Commit the first chapter**

Run: `git add chapters assets README.md && git commit -m "docs: publish first Geth tutorial chapter"`

Expected: one commit containing the first chapter, its seven images, and the README chapter link.

### Task 4: Publish the repository

**Files:** none

- [x] **Step 1: Verify local history and working tree**

Run: `git status --short && git log --oneline --decorate -5`

Expected: a clean working tree and focused, incremental commits.

- [x] **Step 2: Authenticate GitHub CLI if necessary**

Run: `gh auth status`

Expected: the intended GitHub account is authenticated. If not, run the interactive GitHub login flow before continuing.

- [x] **Step 3: Create and push the public repository**

Run: `gh repo create geth-beginners-guide --public --source=. --remote=origin --push --description "A Chinese beginner's guide to Geth"`

Expected: GitHub returns the public repository URL and pushes `main`.

- [x] **Step 4: Verify the remote state**

Run: `gh repo view --json nameWithOwner,visibility,url,defaultBranchRef`

Expected: the repository name is `geth-beginners-guide`, visibility is `PUBLIC`, and the default branch is `main`.
