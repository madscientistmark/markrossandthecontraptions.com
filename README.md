# markrossandthecontraptions.com

Placeholder static site for **Mark Ross & The Contraptions**.

## Structure

```
index.html      The whole page (single page)
styles.css      Styling
assets/
  logo.webp     Band logo (white)
  band.webp     Band photo
render.yaml     Render static-site config
```

Pure HTML/CSS — no build step, no dependencies.

## Preview locally

Open `index.html` in a browser, or run a local server:

```sh
python3 -m http.server 8000
# then visit http://localhost:8000
```

## Deploy on Render

1. Push this folder to a GitHub repo.
2. In Render: **New → Static Site**, connect the repo.
   - Build command: *(leave empty)*
   - Publish directory: `.`
   - (Render will also pick up `render.yaml` automatically.)
3. After the first deploy, go to **Settings → Custom Domains** and add
   `markrossandthecontraptions.com`, then point your DNS at the records
   Render provides.

## Content source

Content adapted from the band's page at
`madscientistkeys.com/mark-ross-and-the-contraptions`. The original page's
contact form was intentionally replaced with a direct email link plus social
buttons (no backend required for a static placeholder).
