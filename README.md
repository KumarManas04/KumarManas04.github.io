# kumarmanas04.github.io

Personal site. Jekyll + minima, with a fully overridden stylesheet and footer.

## Layout

    _config.yml                site title, theme, social link
    index.md                   home page (lists posts)
    assets/main.scss           complete stylesheet — light + dark
    _includes/footer.html      overrides minima's footer
    _includes/diagram-*.svg    inline SVG diagrams
    _posts/YYYY-MM-DD-*.md     articles

## Adding a post

Create `_posts/YYYY-MM-DD-slug.md` with front matter:

    ---
    layout: post
    title: "Your title"
    date: 2026-09-15
    ---

The date prefix in the filename is required; Jekyll ignores files without it.

## Diagrams

Inline SVG in `_includes/`, pulled into a post with:

    {% include diagram-heap.svg %}

They are inline (not `<img>`) so they inherit the page's CSS variables and
always match the active theme. Each carries its own `<style>` block using
`var(--token, fallback)`, so it still renders if the stylesheet fails to load.

## Theming

All colours are CSS custom properties defined twice in `assets/main.scss`:
once in `:root`, once in `@media (prefers-color-scheme: dark)`.
Change them there and it propagates everywhere, diagrams included.

## Overriding the theme

Any file placed in `_includes/` or `_layouts/` takes precedence over minima's
copy of the same name. That is how `footer.html` is replaced — minima's default
renders three floated columns that duplicate the site name.
