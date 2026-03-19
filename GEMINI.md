# Hypho Blog Project Mandates

This document serves as the foundational guide for any AI agent interacting with this repository. Adherence to these rules is mandatory.

## 1. Core Philosophy: Modern Minimalism
- **Focus on Content**: Every design change must serve the goal of reducing distraction.
- **Visual Style**: Maintain a clean, high-air-gap, and sophisticated aesthetic. Avoid flashy animations or heavy UI components.
- **Typography**: Prioritize system-native high-quality sans-serif fonts. Maintain generous line height (1.8) and letter spacing (0.015em).

## 2. Technical Stack & Standards
- **Framework**: Hugo (Extended version) with **PaperMod** theme.
- **Math Rendering**: **KaTeX** is enabled globally.
  - Configuration resides in `layouts/partials/extend_head.html`.
  - Inline math: `$ ... $`, Block math: `$$ ... $$`.
- **CSS Architecture**: All custom styles must be appended to `assets/css/extended.css`. Do NOT modify theme core files.
- **Deployment**: Automated via GitHub Actions (`.github/workflows/hugo.yml`).
  - **CRITICAL**: The `baseURL` in both `hugo.yaml` and the workflow must always use **HTTPS**.

## 3. SEO & GEO (Generative Engine Optimization)
- **AI-Friendly**: Maintain `static/llms.txt` with up-to-date site structure.
- **Metadata**: Every post should include `description` and `tags` in Front Matter.
- **Social**: Default cover image strategy is managed in `hugo.yaml` under `params.images`.

## 4. Maintenance & Workflow Guidelines
- **Human-in-the-Loop**: Never delete files (e.g., drafts, assets) without explicit user confirmation. Always verify before destructive actions.
- **Language**: This is a Chinese blog. All blog posts and primary content must be written in Chinese.
- **Inbox Workflow**: The `inbox/` directory serves as a draft box for temporary writing materials. Once a post is finalized and confirmed by the user, the corresponding inbox material should be proposed for deletion.
- **No Redundancy**: Before adding a new feature, check if it compromises the "Minimalist" goal.
- **Favicons**: Do not delete the placeholder favicons in `static/` unless replacing them with permanent branding.
- **Build**: Always run `.\bin\hugo.exe` locally to verify changes before pushing to GitHub.

## 5. Workflow
- **Commit Messages**: Use concise, meaningful messages (e.g., "feat: add katex support", "fix: adjust line spacing").
- **Branching**: Direct push to `master` is acceptable for this personal project unless specified otherwise.
