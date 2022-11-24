---
title: "Flexible Env in Go"
date: 2020-03-07T09:00:59+01:00
draft: false
comments: true
cover: "img/flexible-env-in-go/DockerGopher.png"
tags:
  - go
  - env
  - docker
  - code
---

## Forewords

I was doing hobby project, nothing special, server with authentication few endpoints and connection to database.
I wrapped this project in docker to distribute project to another developer, he wanted to help with fronted but not that fluent at the backend part.
Then I realized that my friend does not want to struggle with database setup and so on. So, solution was straight forward - docker-compose.

## Problem

I prefer using `.env` instead of typing stuff to cli. I found out really elegant solution. You can still use `.env` and parse it easily with
`github.com/caarlos0/env/v6`, specify default values or even load file (certificates etc). But in docker-compose you need to specify same ports for database for example. Here comes `github.com/joho/godotenv`, if you provide environment variables via dockerfile or docker-compose it will override constants in your `.env`.

## Solution

```go
package config

import (
	"log"

	"github.com/caarlos0/env/v6"
	"github.com/joho/godotenv"
)

type Config struct {
	Env        string `env:"ENVIRONMENT" envDefault:"dev"`
	ServerPort string `env:"SERVER_PORT" envDefault:"8080"`

	DbHost     string `env:"DB_HOST"`
	DbPort     string `env:"DB_PORT"`
	DbUser     string `env:"DB_USER"`
	DbPassword string `env:"DB_PASSWORD"`
	DbDialect  string `env:"DB_DIALECT" envDefault:"sqlite3"`
}

func ParseEnv() *Config {
	if err := godotenv.Load(); err != nil {
		log.Fatal("Error loading .env file")
	}

	var cfg Config
	if err := env.Parse(&cfg); err != nil {
		log.Fatal("Error parsing .env file")
	}

	return &cfg
}

```

Not sure if it will be useful to anyone, but i decided to share it.

Cover from [github.com/ashleymcnamara/gophers](https://github.com/ashleymcnamara/gophers).
