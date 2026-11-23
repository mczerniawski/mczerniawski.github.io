# Local development

GitHub Pages builds the site with its supported Jekyll plugins. To build it
locally, install Ruby and Bundler, then run:

```sh
bundle install
bundle exec jekyll serve
```

The JavaScript assets are committed for GitHub Pages. After changing the
JavaScript sources or dependencies, install Node.js and rebuild them:

```sh
npm ci
npm run build
```

Commit the updated assets and lockfile alongside the source changes.