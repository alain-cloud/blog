# FinOps blog starter

Ready-to-publish Jekyll blog for `blog.alaeddinenajjar.com`.

## 1. Prerequisites on Windows

Install:

- Git
- Visual Studio Code
- Ruby with DevKit
- VS Code extensions: **GitHub Pull Requests and Issues**, **Markdown All in One**, **YAML**

Check from the VS Code terminal:

```powershell
git --version
ruby --version
bundle --version
```

If Bundler is missing:

```powershell
gem install bundler
```

## 2. Preview locally

Open this folder in VS Code, then run:

```powershell
bundle install
bundle exec jekyll serve --livereload
```

Open `http://127.0.0.1:4000`.

## 3. Create the GitHub repository

Create a public repository such as `finops-blog`, without adding a README. Then run:

```powershell
git init
git add .
git commit -m "Launch FinOps blog and first Azure article"
git branch -M main
git remote add origin https://github.com/alain-cloud/finops-blog.git
git push -u origin main
```

If you use another repository name, replace the URL above.

## 4. Configure GitHub Pages

In the repository:

1. Open **Settings > Pages**.
2. Under **Build and deployment**, select **GitHub Actions**.
3. Let the included workflow finish under **Actions**.
4. Under **Custom domain**, enter `blog.alaeddinenajjar.com`.
5. After the certificate is ready, enable **Enforce HTTPS**.

## 5. Configure Cloudflare

Create this record and leave it DNS-only initially:

| Type | Name | Target | Proxy |
|---|---|---|---|
| CNAME | blog | alain-cloud.github.io | DNS only |

Do not modify the existing apex, `www`, MX, SPF, DKIM or DMARC records.

## 6. Editorial check before launch

The first post is in `_posts/2026-11-12-azure-finops-urgent-cost-optimisation.md`.

Before publishing, verify:

- customer confidentiality and approval;
- the 35%, 20% and USD 100,000 figures;
- whether “no reported production impact” is documented;
- job title and author bio;
- links or contact details you want readers to use.

To keep the article offline while reviewing it, rename it to `_drafts/azure-finops-urgent-cost-optimisation.md`. Create the `_drafts` folder if needed.

## 7. Publish future articles

Create files using this pattern:

```text
_posts/YYYY-MM-DD-short-title.md
```

Then commit and push:

```powershell
git add .
git commit -m "Publish new article"
git push
```
