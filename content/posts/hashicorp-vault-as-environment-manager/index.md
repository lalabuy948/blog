---
title: "Hashicorp Vault as Environment Manager"
date: 2020-06-20T20:09:43+02:00
draft: false
comments: true
cover: "/img/hashicorp-vault-as-environment-manager/vault.png"
tags:
  - tools
  - code
  - env

seo:
  [hashicorp, vault, dotenv, how to]
---

# How to use Hashicorp Vault as dot environment manager?

I decided to share this because I was looking for small, simple and easy to use tool where I would store dotenv secrets for my apps. And I found this nice and elegant solution which will allow you to control your application secrets with small bash script. 🙂

## Pre Requirements

- Installed vault
- Finish setup instructions

You can find nicely described steps here: [Getting started](https://learn.hashicorp.com/vault/getting-started/install)

## #1 Let's add KV engine

![Create KV secret engine](/img/hashicorp-vault-as-environment-manager/vault-screenshot.png)

In my case I will name it service. After you added your Key Value engine, you should see something like this:

![Added KV secret engine](/img/hashicorp-vault-as-environment-manager/vault-screenshot-0.png)

## #2 Add secrets (json files)

In my case I created development, staging and production. This files will store actual key value pairs, which in our case dot environment pairs.

![Create new secret engine](/img/hashicorp-vault-as-environment-manager/vault-screenshot-1.png)

## #3 Add key value pairs

Fill your secrets (json files) with your secrets.

`db_host localhost` for example

![Create new secret engine](/img/hashicorp-vault-as-environment-manager/vault-screenshot-2.png)

## #4 Pull key value

First of all you need token. In the right corner you will see profile icon and copy token button when you toggle the first one.

![Create new secret engine](/img/hashicorp-vault-as-environment-manager/vault-screenshot-3.png)

In my project I added this make file:

```makefile
# Makefile

CFLAGS=-g
export CFLAGS

dev:
	@./bin/getenv.sh development

staging:
	@./bin/getenv.sh staging

production:
	@./bin/getenv.sh production

```

And in `bin` folder I have `getenv.sh` file:

```sh
#!/bin/bash

secretsJSON=$(curl --request GET \
  --url https://VAULT.YOURDOMAIN.COM/v1/service/"$1" \
  --header "x-vault-token: ${VAULT_TOKEN}")

# shellcheck disable=SC2217
jq -r '.data | keys[] as $k | "\($k)=\"\(.[$k])\""' <<< "${secretsJSON}" | tr -d '"' > .env
```

Do not forget to export your copied token! `export VAULT_TOKEN=<TOKEN_GOES_HERE>`


Not sure if it useful to anyone, but i decided to share it here.
Picture from [instana.com](https://www.instana.com/blog/hashicorp-vault-monitoring/)
