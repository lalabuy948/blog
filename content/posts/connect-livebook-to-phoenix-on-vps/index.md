---
title: "Connect Livebook to Phoenix on Vps"
date: 2024-08-03T14:19:03+02:00
draft: false
comments: true
cover: "img/connect-livebook-to-phoenix-on-vps/preview.webp"
tags:
  - code
  - elixir
  - tutorial
seo: ["Livebook", "Phoenix", "VPS", "Elixir", "systemd service", "nginx configuration", "deploy Phoenix app", "Livebook server", "Phoenix application setup", "Elixir project management"]
seo_description: "Learn how to connect Livebook to your Phoenix application on a VPS. This guide covers VPS setup, systemd service configuration, and Nginx proxy settings to ensure seamless integration and efficient management of your Elixir and Phoenix projects."
---

[Livebook](https://livebook.dev) is very powerful application which can be used in many different ways with your [Phoneix](https://www.phoenixframework.org) project.
I have few apps running on my VPS and sometimes I would like to run some functions without creating interface for them from admin side, such as cache clean up. Perhaps run some analytical queries to the database and much more.

<!--more-->

## VPS Setup

Here is [great article](https://bjornbr.is/deploying-elixir-and-phoenix/) from Björn Brynjúlfur on how to deploy Phoenix app to VPS.

I just customised couple of things, since I'm using [`asdf`](https://asdf-vm.com).

> When installing `erlang` and `elixir` via `asdf` don't forget about [prerequisites](https://github.com/asdf-vm/asdf-erlang#ubuntu-and-debian).
> And do not forget to add `. $HOME/.asdf/asdf.sh` to your rc file. I'm using zsh, in my case it's `.zshrc`

I have two systemmd services, one for Phoenix app and one for Livebook.

Here is an examples:

```toml
# /lib/systemd/system/phoenix.service
[Unit]
Description=The Indie Stack - Phoenix App
After=network.target

[Service]
Type=simple
User=root
Restart=on-failure
EnvironmentFile=/var/www/TheIndieStack/.env
WorkingDirectory=/var/www/TheIndieStack/
ExecStart=/usr/bin/zsh -c 'elixir --name tis@127.0.0.1 --cookie supersecret -S mix phx.server'
Environment="PATH=/root/.asdf/bin:/root/.asdf/shims:/usr/local/bin:/usr/bin:/bin"

[Install]
WantedBy=multi-user.target
```

Instead of running your Phoneix application with `mix phx.server` you need to run it like so: `elixir --name tis@127.0.0.1 --cookie supersecret -S mix phx.server`. In order to give node name and set cookie, so that in the future we will be able to connect to that from `Livebook`.

Here is simplest way to install Livebook using elixir:

```sh
mix do local.rebar --force, local.hex --force
mix escript.install hex livebook

# Start the Livebook server
livebook server

# See all the configuration options
livebook server --help
```

And since I'm using `.asdf`, here is example of my `livebook.service`:

```toml
# /lib/systemd/system/livebook.service
[Unit]
Description=The Indie Stack - Livebook
After=network.target

[Service]
Type=simple
User=root
Restart=on-failure
EnvironmentFile=/var/www/Livebooks/.env
WorkingDirectory=/var/www/Livebooks/
ExecStart=/usr/bin/zsh -c '/root/.asdf/installs/elixir/1.17.2-otp-27/.mix/escripts/livebook server'
Environment="PATH=/root/.asdf/bin:/root/.asdf/shims:/usr/local/bin:/usr/bin:/bin"

[Install]
WantedBy=multi-user.target
```

```.env
# EnvironmentFile=/var/www/Livebooks/.env from example above
LIVEBOOK_PASSWORD=VerySecretPassword
LIVEBOOK_PORT=9090
```

- Each time when you add or modify service files you must run: `systemctl daemon-reload`
- When you would like to start service: `systemctl start livebook.service`
- When you would like to restart service: `systemctl restart livebook.service`
- When you would like to stop service: `systemctl stop livebook.service`

The last thing you need to do to add nginx config for you Livebook. Here is my example:

```
server {
  listen 80;
  server_name live.example.com;

  location / {
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_redirect off;
    proxy_pass http://localhost:9090;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
  }
}
```

## Connecting

Once `livebook.service` and `phoenix.service` are running, you can connect to your phoenix app from Livebook.

![](/img/connect-livebook-to-phoenix-on-vps/screenshot1.png)
![](/img/connect-livebook-to-phoenix-on-vps/screenshot.png)
