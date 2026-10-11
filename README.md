# Kamus' Notes

> Personal technical journal by Kamus exploring deep systems, database architecture, AI agents, and software engineering.

🌐 **Live Blog**: [https://www.kamusis.me](https://www.kamusis.me)

---

## 🏛️ Repository Architecture

This repository adopts a decoupled dual-branch workflow:

| Branch | Role | Notes |
| :--- | :--- | :--- |
| **`main`** *(Default)* | **Source Repository** | Contains Hexo source files, Markdown posts, theme configurations, and custom styles. |
| **`master`** | **Production Target** | Auto-managed static output (`public/`) compiled by `hexo deploy`, served live via GitHub Pages. |

---

## 🛠️ Tech Stack & Customizations

- **Engine**: [Hexo](https://hexo.io/) static site generator
- **Primary Theme**: [Fluid](https://hexo.fluid-dev.com/) (`hexo-theme-fluid`)
- **Typography**: Plus Jakarta Sans (headings & body) + JetBrains Mono (code blocks)
- **Visuals**: Curated seasonal coastal beacon banners across Home (Autumn), Archives (Winter), Tags (Spring), and About (Summer)
- **Deployment**: GitHub Pages via SSH git deployer with custom domain `www.kamusis.me`

---

## 🚀 Local Development

### 1. Prerequisites

- [Node.js](https://nodejs.org/) (v20+ or v24+ recommended)
- `npm`

### 2. Setup

Clone the repository and install dependencies:

```bash
git clone -b main git@github.com:kamusis/kamusis.github.io.git
cd kamusis.github.io
npm install
```

### 3. Workflow Commands

- **Start local development server**:
  ```bash
  npm run server
  # Runs at http://localhost:4000/
  ```

- **Clean and generate static assets**:
  ```bash
  npm run clean && npm run build
  ```

- **Deploy to GitHub Pages (`master` branch)**:
  ```bash
  npm run deploy
  # Or use the convenience shortcut:
  ./d.sh
  ```

- **Save and backup source changes (`main` branch)**:
  ```bash
  git add .
  git commit -m "feat: new article or config change"
  git push origin main
  ```

---

## 📂 Directory Structure

```text
.
├── _config.yml         # Hexo site configuration
├── _config.fluid.yml   # Fluid theme customization overrides
├── _config.lx.yml      # Preserved fallback theme configuration
├── source/
│   ├── _posts/         # Published Markdown articles
│   ├── css/            # Custom CSS overrides (typography, hero gradients, dark mode)
│   └── img/            # Static banner graphics and site icons
├── themes/             # Standalone theme assets (archer, lx)
└── scaffolds/          # Hexo article templates
```

---

## 📄 License

- **Articles & Content**: [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/)
- **Theme Code & Scripts**: Respective upstream open-source licenses.
