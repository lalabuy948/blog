---
title: "Make Your Phoenix Website AI Ready in Seconds"
date: 2026-04-29T12:00:00+02:00
draft: true
comments: true
cover: "/img/phoenix-ai-ready-markdown-negotiation/preview.png"
tags:
    - elixir
    - code
    - tutorial
seo:
    - phoenix ai ready
    - markdown content negotiation
    - llms.txt phoenix
    - agent skills phoenix
    - text/markdown accept header
    - isitagentready
seo_description: "A tiny Phoenix Plug that returns text/markdown when an AI agent asks for it. Same URL, two representations, browsers see HTML, agents see clean markdown. Validated by isitagentready.com."
---

AI agents are starting to crawl the web the same way browsers do, except they do not need your CSS, your hero animations, or your tracking pixels. They need text. Clean, structured, semantic text. If your site only speaks HTML, the agent has to render, parse, and guess. If your site speaks markdown when asked, the agent gets exactly what it needs in a fraction of the tokens.

[isitagentready.com](https://isitagentready.com/) is a small validator that checks your site against a growing set of "agent skills". One of the most useful checks is **markdown content negotiation**: when a client sends `Accept: text/markdown`, your server should respond with markdown instead of HTML, on the same URL.

This post shows the exact Plug we use on this Phoenix app to pass that check, in well under a hundred lines of code.

<!--more-->

## The contract

The skill spec lives at [isitagentready.com/.well-known/agent-skills/markdown-negotiation/SKILL.md](https://isitagentready.com/.well-known/agent-skills/markdown-negotiation/SKILL.md). The rules are short:

- Request header: `Accept: text/markdown`
- Response: markdown body with `Content-Type: text/markdown`
- Optional `x-markdown-tokens` header with an estimated token count
- HTML stays the default for browsers, markdown is opt-in for agents
- The URL contract is preserved, the same path serves both representations

That last bullet is the important one. No `/page.md` mirror, no separate sitemap, no extra build step. One canonical URL, content negotiation does the rest. Cloudflare zones get this for free, everyone else needs about 80 lines of Elixir.

## The Plug

Here is the full plug, dropped into `lib/easy_inc_web/plugs/markdown_negotiation.ex`:

```elixir
defmodule YourPhoenix.Plugs.MarkdownNegotiation do
  @moduledoc """
  Content negotiation for AI agents.

  When a client requests `Accept: text/markdown`, this plug intercepts the
  outgoing HTML response and rewrites it as a markdown representation,
  preserving the URL contract (same path, different representation).

  Browsers (which send `Accept: text/html,...`) are unaffected.
  """

  import Plug.Conn

  @max_bytes 2_000_000

  def init(opts), do: opts

  def call(conn, _opts) do
    conn = put_resp_header(conn, "vary", "Accept")

    if wants_markdown?(conn) do
      conn
      |> put_req_header("accept", "text/html")
      |> register_before_send(&maybe_convert/1)
    else
      conn
    end
  end

  defp wants_markdown?(conn) do
    conn
    |> get_req_header("accept")
    |> Enum.any?(fn header ->
      header
      |> String.split(",")
      |> Enum.any?(&String.starts_with?(String.trim(&1), "text/markdown"))
    end)
  end

  defp maybe_convert(%Plug.Conn{status: 200} = conn) do
    if html_response?(conn) and small_enough?(conn) do
      convert(conn)
    else
      conn
    end
  end

  defp maybe_convert(conn), do: conn

  defp html_response?(conn) do
    conn
    |> get_resp_header("content-type")
    |> List.first()
    |> case do
      nil -> false
      type -> String.starts_with?(type, "text/html")
    end
  end

  defp small_enough?(%Plug.Conn{resp_body: nil}), do: false
  defp small_enough?(%Plug.Conn{resp_body: body}),
    do: IO.iodata_length(body) <= @max_bytes

  defp convert(conn) do
    html = IO.iodata_to_binary(conn.resp_body)
    markdown = YourPhoenix.MarkdownConverter.from_html(html)
    tokens = markdown |> byte_size() |> div(4) |> Integer.to_string()

    %{conn | resp_body: markdown}
    |> put_resp_header("content-type", "text/markdown; charset=utf-8")
    |> put_resp_header("x-markdown-tokens", tokens)
    |> delete_resp_header("content-length")
  end
end
```

A few things worth pointing out.

**`vary: Accept` is set unconditionally.** Caches and CDNs need to know that two clients hitting the same URL might get different bytes. Without this header, you will eventually serve markdown to a browser or HTML to an agent, and one of them will be very confused.

**The plug rewrites the inbound `Accept` to `text/html` before the rest of the pipeline runs.** Phoenix's `:accepts` plug only knows about the formats you list, and we do not want to register a `markdown` format just for this. Letting Phoenix think it is a normal HTML request keeps controllers, LiveViews, and views completely unchanged.

**Conversion happens in `register_before_send/2`.** The full HTML response is rendered first, then we walk the DOM and emit markdown. This is the lazy approach. A fancier version would render markdown directly from the LiveView, but the cost/benefit is not worth it for static-ish marketing pages. Floki is fast enough.

**There is a 2 MB ceiling.** If your page is larger than that, you have bigger problems than agent readiness. The cap also protects you from doing a full Floki parse on a binary upload that happened to flow through the wrong pipeline.

## Wiring it into the router

One line, at the very top of the `:browser` pipeline:

```elixir
pipeline :browser do
  plug YourPhoenix.Plugs.MarkdownNegotiation
  plug :accepts, ["html"]
  plug :fetch_session
  plug :fetch_live_flash
  plug :put_root_layout, html: {YourPhoenix.Layouts, :root}
  plug :protect_from_forgery
  plug :put_secure_browser_headers
  plug :fetch_current_scope_for_user
end
```

Position matters. The plug needs to see the raw `Accept` header before `:accepts` filters it, and it registers a `before_send` callback that fires after every other plug has done its job.

## The HTML to markdown converter

The plug delegates the actual conversion to a small `MarkdownConverter` module. The full file is around 220 lines, the interesting parts are:

```elixir
def from_html(html) when is_binary(html) do
  {:ok, document} = Floki.parse_document(html)

  title = document |> Floki.find("head title") |> Floki.text() |> normalize_inline()

  description =
    document
    |> Floki.find(~s|head meta[name="description"]|)
    |> Floki.attribute("content")
    |> List.first()
    |> case do
      nil -> ""
      s -> String.trim(s)
    end

  canonical =
    document
    |> Floki.find(~s|head link[rel="canonical"]|)
    |> Floki.attribute("href")
    |> List.first()

  main =
    case Floki.find(document, "main") do
      [] -> Floki.find(document, "body")
      nodes -> nodes
    end

  body_md =
    main
    |> walk()
    |> IO.iodata_to_binary()
    |> decode_entities()
    |> collapse_blank_lines()
    |> String.trim()

  [
    if(title != "", do: "# #{title}\n\n", else: ""),
    if(description != "", do: "> #{description}\n\n", else: ""),
    if(canonical, do: "Canonical: #{canonical}\n\n", else: ""),
    body_md,
    "\n"
  ]
  |> IO.iodata_to_binary()
end
```

The strategy is:

1. Pull `<title>`, `meta description`, and the canonical URL out of `<head>`. These become the markdown front matter the agent sees first.
2. Find `<main>`, fall back to `<body>`. Everything outside `<main>` is chrome the agent does not care about.
3. Walk the DOM with a small set of pattern-matched clauses, one per tag. `h1` becomes `#`, `h2` becomes `##`, `pre` becomes a fenced code block, `a` becomes `[text](href)`, and so on.
4. Skip the noise: `script`, `style`, `noscript`, `svg`, `iframe`, `template`, `form`, `button`, `input`, `select`, `textarea`. Also any node with class `hidden` or the `hidden` attribute.

The full module is in the [easy_inc repo](https://github.com/lalabuy948), if you want to grab it as a starting point.

## Try it

Once deployed, hit any page on your site twice:

```sh
# Browser sees HTML
curl -sI https://your-app.example/ | grep -i content-type
# content-type: text/html; charset=utf-8

# Agent sees markdown
curl -s -H "Accept: text/markdown" https://your-app.example/ | head -20
```

You should see the page title as `# Heading`, the meta description as a blockquote, the canonical URL on its own line, then the body content stripped down to plain markdown. Token count comes back in the `x-markdown-tokens` header.

Then plug the URL into [isitagentready.com](https://isitagentready.com/), and the `markdownNegotiation` check flips to `pass`.

## Why bother

Two reasons.

The first is cost. The same page served as HTML is somewhere between 5x and 20x more tokens than the markdown version once you include the layout, the script tags, the inlined SVG icons, and all the Tailwind class strings. If your content is being read by an agent on every request, that adds up fast on the agent's side, and slows down whatever it is trying to do for the user.

The second is fidelity. An LLM reading raw HTML has to figure out what is content versus chrome, what is navigation versus body, what is decorative versus semantic. By the time it has done that, it has burned tokens and made mistakes. Handing it a clean markdown document means it spends its budget on reasoning about your content, not on parsing your layout.

The whole thing took an afternoon to write and ship. If you run a Phoenix site, there is no real reason not to do it.
