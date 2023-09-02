# Kubesec Action

> [GitHub Action](https://github.com/features/actions) for [kubesec](https://github.com/controlplaneio/kubesec) fork of [https://github.com/controlplaneio/kubesec-action]

![kubesec_logo](images/kubesec_logo.svg)

## Table of Contents

- [Usage](#usage)
  - [Workflow](#workflow)
- [Customizing](#customizing)
  - [Inputs](#inputs)

## Usage

### Workflow

```yaml
name: lint
on:
  push:
    branches:
      - main
  pull_request:
jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Run kubesec scanner
        uses: bsanchezmir/kubesec-action@latest
        with:
          filename: file.yaml
```

### Using kubesec with GitHub Code Scanning

If you have [GitHub code scanning][code_scanning] available you can use kubesec as a scanning tool as follows:

```yaml
name: kubesec
on:
  push:
    branches:
      - main
  pull_request:
jobs:
  lint:
    name: Lint
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Get sarif.tpl
        run: wget https://raw.githubusercontent.com/bsanchezmir/kubesec-action/main/sarif.tpl

      - name: Run kubesec scanner
        uses: bsanchezmir/kubesec-action@latest
        with:
          filename: file.yaml
          exit-code: "0"
          format: template
          template: ./sarif.tpl
          output: kubesec-results.sarif

      - name: Upload Kubesec scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: kubesec-results.sarif
```

### Using kubesec with GitHub Code Scanning to scan multiple yaml/yml files

Scanning multiple yaml files

```yaml
name: Kubesec

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main
  schedule:
    - cron: '34 10 * * 3'

jobs:
  findyamls:
    runs-on: ubuntu-latest
    outputs:
      matrix: ${{ steps.set-matrix.outputs.matrix }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Find Kubernetes YAML files
        run: |
          files=$(find . -name "*.yaml" -o -name "*.yml")
          k8s_files=""
          for file in $files; do
            if grep -q "^apiVersion:" $file && grep -q "^kind:" $file && grep -q "^metadata:" $file && grep -q "^spec:" $file; then
              k8s_files="$k8s_files\"$file\","
            fi
          done
          echo "FILE_MATRIX={\"file\": [$k8s_files]}" >> $GITHUB_ENV 
      - name: Set matrix
        run: echo "matrix=$FILE_MATRIX" >> $GITHUB_OUTPUT
        id: set-matrix

  kubesec:
    needs: findyamls
    runs-on: ubuntu-latest
    permissions:
      security-events: write
      actions: read
      contents: read
    strategy:
      matrix:
        file: ${{fromJson(needs.findyamls.outputs.matrix).file }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Get sarif.tpl
        run: wget https://raw.githubusercontent.com/bsanchezmir/kubesec-action/main/sarif.tpl

      - name: Run kubesec scanner
        uses: bsanchezmir/kubesec-action@latest
        with:
          exit-code: "0"
          template: ./sarif.tpl
          format: template
          output:  ${{ matrix.file }}.sarif
          filename: ${{ matrix.file }}
  
      - name: Upload Kubesec scan results to GitHub Security tab
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file:  ${{ matrix.file }}.sarif 
```

## customizing

### inputs

Following inputs can be used as `step.with` keys:

The sarif template is being pulled from  `https://raw.githubusercontent.com/bsanchezmir/kubesec-action/main/sarif.tpl`


| Name        | Type   | Default | Description                              |
| ----------- | ------ | ------- | ---------------------------------------- |
| `filename`  | String |         | File to scan                             |
| `format`    | String | `json`  | Output format (`json`, `template`)       |
| `template`  | String |         | Output template (`sarif.tpl`) |
| `output`    | String |         | Save results to a file                   |
| `exit-code` | String | `"2"`   | Override the exit-code                   |
