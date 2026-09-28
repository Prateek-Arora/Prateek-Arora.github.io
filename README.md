Source for https://prateek-arora.github.io (GitHub Pages, Jekyll). No theme: the layouts are in
`_layouts/` and the stylesheet is `assets/site.css`. Fonts are IBM Plex Sans and JetBrains Mono,
self-hosted under the SIL Open Font License (`assets/fonts/`).

Preview locally: `docker run --rm -v "$PWD":/srv -w /srv -p 4000:4000 ruby:3.3 sh -c "bundle install && bundle exec jekyll serve --host 0.0.0.0"`
