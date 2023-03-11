---
title: "Notion Github Backup"
date: 2021-03-11T17:11:54+01:00
draft: false
comments: true
cover: "img/notion-github-backup/notion+github.webp"
tags:
  - code
  - tools
  - lifestyle
seo:
  ["notion", "github", "action", "backup"]
---

## Data is expensive

I write around 2-3 important notes a day. And don't actually worry if my laptop will be stolen, but the data on it.. 

Since Im heavy notion user I was looking for automated backup solution. Not that I don't trust notion, trust but verify, right? 

## Backup

1. Create a private github repo 

![create private repo](/img/notion-github-backup/private_repo.webp)

2. Go to actions tab and click on `set up a workflow yourself ->`

![set up workflow](/img/notion-github-backup/setup_workflow.webp)

3. Paste this yml file

```yml
name: "Notion backup"

on:
  push:
    branches:
      - main
  schedule:
    -   cron: "0 0 * * *"

jobs:
  backup:
    runs-on: ubuntu-latest
    name: Backup
    timeout-minutes: 15
    steps:
      - uses: actions/checkout@v2
      - uses: actions/setup-node@v2
        with:
          node-version: '12'
      - name: Setup dependencies
        run: npm install -g notion-backup

      - name: Run backup
        run: notion-backup
        env:
          NOTION_TOKEN: ${{ secrets.NOTION_TOKEN }}
          NOTION_SPACE_ID: ${{ secrets.NOTION_SPACE_ID }}

      - name: Commit changes
        uses: elstudio/actions-js-build/commit@v3
        with:
          commitMessage: Automated snapshot
```

4. Get your secrets

    1. Go to [www.notion.so](https://www.notion.so)
    2. Go to `Settings & Members`
    3. Go to `Workspace` -> `Settings`
    4. Open inspector in your browser and navigate to network tab (set XHR filter)
    5. Click on `Export all workspace content`
    6. Open any `getTasks` request
    7. Extract `spaceId` from preview and `token_v2` from headers

![notion space id](/img/notion-github-backup/space_id.webp)
![notion token v2](/img/notion-github-backup/token_v2.webp)

5. Add secrete keys

where `NOTION_SPACE_ID` is your `spaceId` from step above

and `NOTION_TOKEN` is your `token_v2` from step above

![add secrete keys](/img/notion-github-backup/secrets.webp)

## Here we go

Now you have fully automated backup of your entire notion workspace in private github repo.
