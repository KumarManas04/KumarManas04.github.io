# kumarmanas04.github.io

Personal site. Jekyll + minima, with a fully overridden stylesheet.

## Layout

    _config.yml                site title, theme, social link
    index.md                   home page (lists posts)
    assets/main.scss           complete stylesheet — light + dark
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

Diagrams are inline SVG in `_includes/`, pulled into a post with:

    {% include diagram-heap.svg %}

They are inline (not `<img>`) so they inherit the page's CSS variables and
always match the active theme. Each file carries its own `<style>` block using
`var(--token, fallback)`, so it still renders correctly if the stylesheet fails.

## Theming

All colours are CSS custom properties defined twice in `assets/main.scss`:
once in `:root`, once inside `@media (prefers-color-scheme: dark)`.
Change a colour in those two blocks and it propagates everywhere,
diagrams included.
