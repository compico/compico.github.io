# Compico — portfolio & project hub

A Hugo-powered portfolio with a dedicated OverlayComposer area.

## Local preview

Install Hugo Extended and run: hugo server

## Site structure

- content/_index.md — portfolio homepage.
- content/overlaycomposer/_index.md — OverlayComposer project page.
- content/overlaycomposer/docs/ — manually maintained documentation.
- content/overlaycomposer/releases/ — manually written release notes.

## Write a release note

Create a Markdown file under content/overlaycomposer/releases/, for example v1.2.3.md. Edit the front matter and body, then set draft = false.

Release notes are not imported from GitHub Releases. This site publishes only the Markdown written here.

Push to main. GitHub Actions builds the site and deploys it to GitHub Pages.
Site URL: https://compico.github.io/compico/
