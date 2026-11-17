# GitHub Publishing Guide

## 1. Review before publishing

From the project folder:

```bash
cd ~/smartmed-ai
git status --short
```

Check that the following are not staged or committed:
- `data/`
- `*.db`, `*.db-wal`, `*.db-shm`
- `.env`
- `.venv/`, `venv/`
- patient exports, screenshots, logs, backups, credentials

The supplied `.gitignore` already excludes several of these patterns. Still inspect the staged files manually.

## 2. Initialize Git if needed

```bash
cd ~/smartmed-ai
git init
git branch -M main
git add .
git status
```

Review the file list carefully. If safe:

```bash
git commit -m "Add SmartMed AI source and documentation"
```

## 3. Connect to GitHub

Create an empty repository on GitHub, then use its HTTPS or SSH URL:

```bash
git remote add origin https://github.com/YOUR_USERNAME/YOUR_REPOSITORY.git
git push -u origin main
```

Replace the placeholders with the actual repository details. Do not paste a personal access token into source files or chat.

## 4. If the repository already has a remote

```bash
git remote -v
git status
git add README.md docs/
git commit -m "Add technical project documentation"
git push
```

## 5. Upload only the documentation pack from Windows

If the documentation ZIP is downloaded to your computer, extract it and copy the `README.md` and `docs/` folder into the repository. Then run the Git commands above.

## 6. Recommended repository presentation

- Keep the README concise and use it as the project landing page.
- Put detailed explanations in `docs/`.
- Add real prototype photos only after checking that no personal data is visible.
- Add wiring diagrams and test evidence only when they match the actual hardware.
- Label synthetic AI metrics clearly.
- Do not publish the live SQLite database.
