# Project: xw19.github.io

Retro "Terminal & CRT" technical blog built with Hugo.

## Core Aesthetic: Terminal & CRT
This project uses the `terminal` theme with custom CRT enhancements. Adhere to these visual standards for any modifications:
- **Theme:** `hugo-theme-terminal` (Green color scheme).
- **CRT Effect:** Managed via `static/css/crt.css`. Ensure any new UI elements respect the scanline overlay and phosphor glow.
- **Typography:** Monospaced, terminal-style fonts.

## Technology Focus
The blog content focuses on:
- **Kubernetes (k8s):** Modern orchestration with a retro lens.
- **Linux & Systems:** Kernel, shell, and low-level engineering.
- **AI:** Frontier tech discussed through a 90s terminal interface.

## Conventions
- **Posts:** Always place new blog entries in `content/posts/`.
- **Front Matter:** Use YAML front matter. Include `tags` for `k8s`, `linux`, `systems`, or `ai`.
- **Custom CSS:** Add global style changes to `static/css/crt.css` and ensure they are referenced in `hugo.toml` under `params.customCSS`.

## Workflows
- **Development:** Use `hugo server` for local preview.
- **Build:** Run `hugo` to generate the static site in the `public/` directory.
