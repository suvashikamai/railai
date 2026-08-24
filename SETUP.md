# Setup — using GitHub only

You do **not** need Node.js, npm, a terminal, or any software on your computer.
GitHub builds the app and hosts it for you.

Total time: about 10 minutes.

---

## Step 1 — Extract the zip

Unzip `railai.zip`. You get a folder called `railai` containing `src`, `public`,
`package.json`, `index.html` and so on.

Open that folder. You should see the files **inside** it — not the `railai` folder
itself. That distinction matters in Step 3.

---

## Step 2 — Create an empty repository

1. Go to **https://github.com/new**
2. **Repository name:** `railai`
3. **Public** (required for free GitHub Pages)
4. Leave **"Add a README file"**, **"Add .gitignore"** and **"Choose a license"** all
   **unticked** — this project already has them, and adding duplicates causes a conflict
5. Click **Create repository**

---

## Step 3 — Upload the files

On the empty repository page, click **uploading an existing file**
(or go to `https://github.com/<your-username>/railai/upload/main`).

Select **all the files and folders inside** the `railai` folder — `src`, `public`,
`scripts`, `docs`, `index.html`, `package.json`, `package-lock.json`, `vite.config.js`,
`README.md`, `SETUP.md`, `LICENSE` — and drag them into the browser window.

> **Windows:** select all with `Ctrl+A` inside the folder, then drag.
> **Mac:** `Cmd+A`, then drag.

Wait for every file to finish uploading, then scroll down and click **Commit changes**.

### Step 3b — Add the build file by hand

Browsers usually refuse to upload folders whose name starts with a dot, so the
`.github` folder will probably be missing. Add it directly on GitHub:

1. On your repository page click **Add file → Create new file**
2. In the filename box type exactly:

   ```
   .github/workflows/deploy.yml
   ```

   (GitHub turns the slashes into folders as you type)
3. Open `.github/workflows/deploy.yml` from the extracted zip in Notepad or TextEdit,
   copy everything, and paste it into the editor
4. Click **Commit changes**

Do the same for `.gitignore` if it did not upload — copy the contents from the zip.

---

## Step 4 — Turn on GitHub Pages

1. In your repository go to **Settings** (top bar) → **Pages** (left sidebar)
2. Under **Build and deployment → Source**, choose **GitHub Actions**
3. That is all — there is nothing to save

---

## Step 5 — Watch it build

Click the **Actions** tab. A run called **Build & Deploy** should be in progress.
It installs dependencies, runs the planner invariant checks, builds the app and
publishes it.

It takes about 90 seconds. When the tick turns green, your app is live at:

```
https://<your-username>.github.io/railai/
```

If the run did not start, go to **Actions → Build & Deploy → Run workflow**.

---

## Troubleshooting

**"Actions tab shows nothing"**
`.github/workflows/deploy.yml` did not upload. Redo Step 3b — the file path must be
exact, including the leading dot.

**Run fails at `npm ci`**
`package-lock.json` is missing from the repository. Upload it from the extracted zip
(**Add file → Upload files**).

**Green tick but the page is blank / 404**
Pages source is still set to a branch. Go back to Step 4 and set it to
**GitHub Actions**, then re-run the workflow from the Actions tab.

**Page loads but styling looks broken**
Hard-refresh: `Ctrl+Shift+R` (Windows) or `Cmd+Shift+R` (Mac).

---

## Making changes later

Edit any file directly on GitHub — click the file, then the pencil icon, then
**Commit changes**. The workflow rebuilds and redeploys automatically within two
minutes.

The two files worth editing are:

| File | What it controls |
|---|---|
| `src/data/trains.js` | The 25 services — numbers, names, classes, departure times, entry delays |
| `src/data/assets.js` | The 16 tracked assets, their wear rates, and the booked possessions |

Both are plain data. If a change breaks an operating invariant, the Actions run
fails and tells you which one — the previous working version stays live.
