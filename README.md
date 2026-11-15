# mrpopov.com

Hugo blog, theme `hello-friend`, published to `docs/`.

## How to add a tool

1. Put the standalone page at `static/tools/<slug>/index.html`. Hugo copies it as is.
2. Add one entry to `data/tools.yaml` (`slug`, `title`, `description`, optional `date`, `tags` and `cover`, a square image file in the tool folder).
3. Run `hugo`. The tool shows up on `/tools/` and lives at `/tools/<slug>/`.

Do not create `content/tools/<slug>` pages, they would write to the same path as the static file.
