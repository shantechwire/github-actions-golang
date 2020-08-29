# GitHub Actions for Go

[GitHub Actions](https://github.com/features/actions) includes CI/CD for free
for Open Source repositories. This document contains information on making it
work well for [Go](https://golang.org). See them [in
action](https://github.com/mvdan/github-actions-golang/actions):

```yaml
$ cat .github/workflows/test.yml
on: [push, pull_request]
name: Test
jobs:
  test:
    strategy:
      matrix:
        go-version: [1.14.x, 1.15.x]
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
    - name: Install Go
      uses: actions/setup-go@v2
      with:
        go-version: ${{ matrix.go-version }}
    - name: Checkout code
      uses: actions/checkout@v2
    - name: Test
      run: go test ./...
```

## Summary

Each workflow file has a number of jobs, which get run `on` specified events,
and run concurrently with each other.

Each `job` runs on a configuration `matrix`. For example, we can test two major
Go versions on three operating systems.

Each job has a number of `steps`, such as installing Go, or checking out the
repository's code.

## FAQs

#### What about module support?

If your repository contains a `go.mod` file, Go 1.12 and later will already use
module mode by default. To turn it on explicitly, set `GO111MODULE=on`.

#### How do I set environent variables?

They can be set up via `env` for an [entire
workflow](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#env),
a job, or for each step:

```yaml
env:
  GOPROXY: "https://proxy.company.com"
jobs:
  [...]
```

#### How do I set environent variables at run-time?

You can use [workflow commands](https://docs.github.com/en/actions/reference/workflow-commands-for-github-actions)
to set environment variables, add an element to `$PATH`, and more. For example:

```yaml
echo "::set-env name=CGO_ENABLED::0"
echo "::add-path::${HOME}/goroot/bin"
```

#### How do I set up caching between builds?

Use [actions/cache](https://github.com/actions/cache). For example, to cache
downloaded modules:

```
- uses: actions/cache@v2
  with:
    path: ~/go/pkg/mod
    key: ${{ runner.os }}-go-${{ hashFiles('**/go.sum') }}
    restore-keys: |
      ${{ runner.os }}-go-
```

#### How do I run a step conditionally?

You can use `if` conditionals, using their [custom expression
language](https://docs.github.com/en/actions/reference/context-and-expression-syntax-for-github-actions):

```yaml
- name: Run end-to-end tests on Linux
  if: github.event_name == 'push' && matrix.os == 'ubuntu-latest'
  run: go run ./endtoend
```

#### How do I set up a custom build matrix?

You can [include extra matrix
jobs](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#example-including-new-combinations),
and you can [exclude specific matrix
jobs](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#example-excluding-configurations-from-a-matrix).

#### How do I run multiline scripts?

```yaml
- name: Series of commands
  run: |
    go test ./...
    go test -race ./...
```

#### Should I use two workflows, or two jobs on one workflow?

The biggest difference is the UI; workflow results are shown separately.
Grouping jobs in workflows can also be useful if one wants to customize the
workflow triggers, or to set up dependencies via
[needs](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions#jobsjob_idneeds).

#### How do I set up a secret environment variable?

Follow [these steps](https://docs.github.com/en/actions/configuring-and-managing-workflows/creating-and-storing-encrypted-secrets)
to set up the secret in the repo's settings. After adding a secret like
`FOO_SECRET`, use it on a step as follows:

```yaml
- name: Command that requires secret
  run: some-command
  env:
    FOO_SECRET: ${{ secrets.FOO_SECRET }}
```

#### How do I install private modules?

It's possible to install modules from private GitHub repositories without using
your own proxy. You'll need to add a
[personal access token](https://github.com/settings/tokens) as a secret
environment variable for this to work.

```yaml
- name: Configure git for private modules
  env:
    TOKEN: ${{ secrets.PERSONAL_ACCESS_TOKEN }}
  run: git config --global url."https://YOUR_GITHUB_USERNAME:${TOKEN}@github.com".insteadOf "https://github.com"
```

#### How do I install Linux packages?

Use `sudo apt`, making sure to only run the step on Linux:

```yaml
- name: Install Linux packages
  if: matrix.os == 'ubuntu-latest'
  run: sudo apt update && sudo apt install -y --no-install-recommends mypackage
```

#### How do I set up a `GOPATH` build?

Declare `GOPATH` and clone inside of it:

```yaml
jobs:
  test-gopath:
    env:
      GOPATH: ${{ github.workspace }}
      GO111MODULE: off
    steps:
    - name: Checkout code
      uses: actions/checkout@v2
      with:
        path: ./src/github.com/${{ github.repository }}
```

## Quick links

* Concepts, rate limits, etc: https://docs.github.com/en/actions/getting-started-with-github-actions/about-github-actions

* Syntax and fields reference: https://docs.github.com/en/actions/reference/workflow-syntax-for-github-actions

* Environment reference: https://docs.github.com/en/actions/reference/virtual-environments-for-github-hosted-runners

* To report bugs: https://github.community/c/github-actions/41

## Caveats

* https://github.com/actions/checkout/issues/135

`git config core.autocrlf` defaults to true, so be careful about CRLF endings in
your plaintext `testdata` files on Windows. To work around this, set up the
following `.gitattributes`:

```gitattributes
* -text
```

* https://github.com/actions/virtual-environments/issues/712

`os.TempDir` on Windows will contain a short name, since `%TEMP%` also contains
it. Note that case sensitivity doesn't matter, and that `os.Open` should still
work; but some programs not treaing short names might break.

```cmd
> echo %USERPROFILE%
C:\Users\runneradmin
> echo %TEMP%
C:\Users\RUNNER~1\AppData\Local\Temp
```
