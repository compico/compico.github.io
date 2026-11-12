# OverlayComposer Release Hub

A Hugo-powered release notes website for OverlayComposer.

## Local development

Install Hugo, then run:

```sh
hugo server
```

Open http://localhost:1313.

## Publish a release

Create a new release note:

```sh
hugo new content releases/v1.2.3.md
```

Edit the generated Markdown file:

- Set `title`, `date`, `version`, and `summary` in the front matter.
- Add sections such as Highlights, Improvements, Bug fixes, and Downloads.
- Set `draft = false` (or `draft: false`) when it is ready to publish.

Commit and push to `main`. GitHub Actions builds the Hugo site and deploys it to GitHub Pages.

## GitHub Pages

In the repository settings, open **Pages** and set **Build and deployment → Source** to **GitHub Actions**.

The site URL is https://compico.github.io/compico/.
