# Geth Beginner's Guide Repository Design

## Goal

Publish the author's Chinese Geth learning notes as a readable, incrementally maintained open-source textbook while keeping the source Word document local.

## Repository

- GitHub repository: `geth-beginners-guide`
- Visibility: public
- Display title: `Geth Beginner's Guide`
- License: Creative Commons Attribution 4.0 International
- Default branch: `main`

## Content structure

The repository uses `README.md` as the Chinese-language landing page and table of contents. Long-form material lives in numbered Markdown files under `chapters/`. Images live under a chapter-specific directory in `assets/`.

The initial Word document becomes `chapters/01-running-a-local-geth-dev-chain.md`. Its seven embedded images are renamed with stable, descriptive English filenames under `assets/chapter-01/`. The source `.docx` is never added to the repository.

## Conversion rules

- Preserve the author's technical meaning and Chinese voice.
- Normalize malformed or empty Word headings into a consistent Markdown hierarchy.
- Render shell and JavaScript commands as fenced code blocks.
- Convert the parameter table to GitHub-flavored Markdown.
- Use relative image paths and meaningful Chinese alt text.
- Do not invent technical claims or add unsupported material.
- Preserve screenshots unless they expose secrets or high-risk personal data.

## Update workflow

When the author updates `/Users/wx/Desktop/geth学习笔记.docx` and requests a sync:

1. Read and convert the current Word document without modifying it.
2. Compare generated content and images with the repository state.
3. Review the diff for accidental deletions, formatting regressions, secrets, and sensitive personal information.
4. Update only affected Markdown and image files.
5. Verify links, image references, Markdown structure, and repository cleanliness.
6. Commit a focused change with a descriptive message and push to `main`.

No background watcher or automatic publishing service is created. Publication happens only after an explicit sync request.

## Commit strategy

The initial publication is split into focused commits: repository workflow documentation, project metadata and licensing, and the first converted chapter. Later note updates receive one or more topic-focused commits based on the size of the change.

## Error handling and verification

Conversion stops before publication if an image is missing, a local absolute path remains in Markdown, secrets are detected, or the GitHub authentication state prevents a reliable push. Before publishing, all relative links are resolved locally, every referenced image is present, no `.docx` file is tracked, and the final Git diff is reviewed.
