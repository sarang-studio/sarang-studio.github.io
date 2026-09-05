# sarang-studio.github.io

Publisher site for Sarang Studio's apps, served by GitHub Pages at https://sarang-studio.github.io/.

Layout:

- `index.html` — landing page listing the apps. Plain HTML, no Jekyll layout.
- `<app>/privacy/index.md` — that app's privacy policy, rendered with the site theme.
  Currently: `daylog/privacy/`.
- Support for every app is this repository's Issues page.

Adding an app: create `<app>/privacy/index.md` with the same front matter as the Daylog one, then add a list item to `index.html`.
