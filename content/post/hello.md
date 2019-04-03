---
title: "Hello"
date: 2019-04-03T12:37:16+02:00
draft: false
---

**Insert Lead paragraph here.**

## New Cool Posts

```php
<?php

$groupAttributes = array_map(
        function ($expr, $alias) {
        return sprintf('%s AS %s', $expr, $alias);
    },
$this->attributeGroups[$attributeGroup],
array_keys($this->attributeGroups[$attributeGroup])
);
```