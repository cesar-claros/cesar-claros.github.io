# cesar-claros.github.io

Personal academic website of César Claros, built with the [al-folio](https://github.com/alshedivat/al-folio) Jekyll starter (v1.x) and published with GitHub Pages at <https://cesar-claros.github.io>.

## Where the content lives

| What                 | File(s)                                   |
| -------------------- | ----------------------------------------- |
| Site settings, name  | `_config.yml`                             |
| Home page / bio      | `_pages/about.md`, `assets/img/prof_pic.jpg` |
| Publications         | `_bibliography/papers.bib`                |
| Projects             | `_projects/*.md`                          |
| News                 | `_news/*.md`                              |
| CV page              | `_data/cv.yml` (RenderCV format); PDF auto-rendered to `assets/rendercv/rendercv_output/` |
| Social links         | `_data/socials.yml`                       |
| Blog posts           | `_posts/YYYY-MM-DD-title.md` |

See `docs/CUSTOMIZE.md` for the full customization guide and `docs/FAQ.md` for common issues.

## Run locally

Requires Docker.

```bash
docker compose pull   # first time only
docker compose up     # then open http://localhost:8080
```

Edits to content are picked up automatically. Edits to `_config.yml` restart the server.

## Deploy

Pushing to `master` runs `.github/workflows/deploy.yml`, which builds the site and publishes it to the `gh-pages` branch. GitHub Pages serves that branch. Do not edit `gh-pages` by hand.

## Upgrading al-folio

The theme runtime lives in the `al_*` gems pinned in `Gemfile`. To upgrade:

```bash
docker compose run --rm jekyll bash -c "bundle update && bundle exec al-folio upgrade audit"
```

## License

The site content is © César Claros. The al-folio starter is MIT licensed (see `LICENSE`).
