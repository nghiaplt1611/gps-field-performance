# 🌐 Hosting Documentation on GitHub Pages

You can easily host these markdown files as a beautiful, interactive website using **GitHub Pages**. There are two great ways to do this: the "Native (Zero-Config)" way and the "Docsify (Pro)" way. 

---

## Method 1: The Native GitHub Pages Way (Easiest)

GitHub Pages natively supports Markdown through Jekyll. If you just enable GitHub Pages, it will automatically render your `README.md` and `docs/*.md` files into a website.

**Steps:**
1. Push all your code and markdown files to a repository on GitHub.
2. Go to your repository on GitHub.
3. Click on **Settings** (the gear icon at the top).
4. On the left sidebar, scroll down and click on **Pages**.
5. Under **Build and deployment**:
   - Source: Select **Deploy from a branch**
   - Branch: Select `main` (or `master`), and select the `/ (root)` folder.
   - Click **Save**.
6. Wait 1-2 minutes. GitHub will provide you with a URL (e.g., `https://your-username.github.io/gps-field-performance/`).

**Pros:** Zero configuration required.
**Cons:** The navigation between pages won't have a nice sidebar menu automatically. You'll rely on the markdown links you created.

---

## Method 2: The Docsify Way (Recommended for Portfolios)

[Docsify](https://docsify.js.org/) is a magical documentation site generator. It doesn't generate static HTML files; instead, it dynamically loads your Markdown files and parses them as a beautiful website with a sidebar. This looks **much more professional** for an Analytics Engineer portfolio.

### Step 1: Initialize Docsify (Local Setup)

Run this in your terminal (requires Node.js):
```bash
npx docsify-cli init ./docs
```
This will create an `index.html` inside your `docs/` folder.

### Step 2: Configure the Sidebar

To get a nice navigation menu, create a file named `_sidebar.md` inside your `docs/` folder:

**`docs/_sidebar.md`**
```markdown
- [Home](../README.md)
- [Business Context](business_context.md)
- [Data Dictionary](data_dictionary.md)
- [Metrics Definitions](metrics_definitions.md)
- [Technical Architecture](technical_architecture.md)
- [Analysis Findings](analysis_findings.md)
```

Then, open the newly created `docs/index.html` and enable the sidebar by changing the script configuration:

```html
<script>
  window.$docsify = {
    name: 'GPS Field Performance',
    repo: 'your-username/gps-field-performance',
    loadSidebar: true, // <-- Add this line
    subMaxLevel: 2     // <-- Add this line (shows headers in the sidebar)
  }
</script>
```

### Step 3: Deploy to GitHub Pages

1. Push your changes to GitHub.
2. Go to your repository **Settings** -> **Pages**.
3. Under **Build and deployment**:
   - Source: Select **Deploy from a branch**
   - Branch: Select `main` (or `master`), and select the **`/ (root)`** folder.
   - Click **Save**.

Wait a minute, and your site will be live with a beautiful sidebar, search functionality (if you add the search plugin), and a very premium feel!

---

## Managing Images

I have created an image folder for you at: `docs/assets/images/`.

When you want to add an image to any of your markdown files, put the image in that folder and reference it like this:

**In `README.md` (root directory):**
```markdown
![Dashboard Screenshot](docs/assets/images/dashboard.png)
```

**In files inside the `docs/` directory (e.g., `business_context.md`):**
```markdown
![Dashboard Screenshot](assets/images/dashboard.png)
```
