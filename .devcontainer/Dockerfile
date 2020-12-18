FROM mcr.microsoft.com/vscode/devcontainers/go:dev-1.15

RUN apt-get update && export DEBIAN_FRONTEND=noninteractive \
 && apt-get -y install --no-install-recommends git wget zsh curl

RUN curl -L https://github.com/gohugoio/hugo/releases/download/v0.74.3/hugo_0.74.3_Linux-64bit.tar.gz | tar xvz
RUN mv hugo /usr/bin/hugo

EXPOSE 1313
