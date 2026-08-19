# Kubes and Clouds source code

Based on [Beautiful Jekyll](https://beautifuljekyll.com/).
Live at [kubesandclouds.com](https://kubesandclouds.com).

## Run locally

Requires Ruby 3.x and Bundler.

```bash
bundle install
bundle exec jekyll serve
```

Site served at http://127.0.0.1:4000. Jekyll rebuilds on file changes; `_config.yml` changes need a restart.

### Troubleshooting

- `cannot load such file -- bigdecimal` (Ruby 3.4+): the Gemfile already pins `bigdecimal`. Re-run `bundle install`.
- `command not found: jekyll`: run inside `bundle exec` or run `bundle install` first.

## Deploy

Branch `terminal-style` is the source of truth. On push, `.github/workflows/deploy-terminal-style.yml` builds Jekyll and force-pushes `_site/` to `main`. GitHub Pages serves `main` at kubesandclouds.com.

- Skip a deploy: include `[skip deploy]` in the commit message.
- Rollback: `git revert <sha>` on `terminal-style` and push, or `git reset --hard <good-sha>` then `git push -f`. The next workflow run rebuilds prod.
- Safety snapshot of pre-terminal source: branch `main-source-backup`.

Do not edit `main` directly — each deploy overwrites it.

## Structure

- `en/`, `es/` — bilingual content (Spanish currently hidden)
- `_layouts/`, `_includes/` — theme templates (Beautiful Jekyll fork)
- `assets/css/terminal.css` — terminal/shell theme overrides
- `assets/img/` — images and logos
- `_config.yml` — site config, navbar links, avatar, custom CSS
