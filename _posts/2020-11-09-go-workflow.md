---
title:  "go 项目的 github action"
author: adream307
date:   2020-11-09 16:07:00 +0800
categories: [github,go]
tags: [action]
---
<https://github.com/actions/starter-workflows/blob/main/ci/go.yml>

```yml
name: Go

on:
  push:
    branches: [ $default-branch ]
  pull_request:
    branches: [ $default-branch ]

jobs:

  build:
    name: Build
    runs-on: ubuntu-latest
    steps:

    - name: Set up Go 1.x
      uses: actions/setup-go@v2
      with:
        go-version: ^1.13

    - name: Check out code into the Go module directory
      uses: actions/checkout@v2

    - name: Get dependencies
      run: |
        go get -v -t -d ./...
        if [ -f Gopkg.toml ]; then
            curl https://raw.githubusercontent.com/golang/dep/master/install.sh | sh
            dep ensure
        fi
    - name: Build
      run: go build -v .

    - name: Test
      run: go test -v ./...
```