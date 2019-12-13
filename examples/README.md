
  - [Quickstart CI](#quickstart-ci-workflow) - A simple CI workflow to
    check with the release version of R.
  - [Tidyverse CI](#tidyverse-ci-workflow) - A more complex CI workflow
  - [Pull Request Commands](#commands-workflow) - Adds `/document` and
    `/style` commands for pull requests.
  - [Render README](#render-readme) - Render README.Rmd when it changes
    and commit the result

## Quickstart CI workflow

This workflow installs latest release R version on macOS and runs R CMD
check via the [rcmdcheck](https://github.com/r-lib/rcmdcheck) package.

### When can it be used?

1.  You have a simple R package
2.  There is no OS-specific code
3.  You want a quick start with R CI

<!-- end list -->

``` yaml
on: [push, pull_request]

name: R-CMD-check

jobs:
  R-CMD-check:
    runs-on: macOS-latest
    steps:
      - uses: actions/checkout@v1
      - uses: r-lib/actions/setup-r@master
      - name: Install dependencies
        run: Rscript -e "install.packages(c('remotes', 'rcmdcheck'))" -e "remotes::install_deps(dependencies = TRUE)"
      - name: Check
        run: Rscript -e "rcmdcheck::rcmdcheck(args = '--no-manual', error_on = 'error')"
```

## Tidyverse CI workflow

This workflow installs the last 5 minor R versions and runs R CMD check
via the [rcmdcheck](https://github.com/r-lib/rcmdcheck) package on the
three major OSs (linux, macOS and Windows). This workflow is what the
tidyverse teams uses on their repositories, but is overkill for less
widely used packages, which are better off using the simpler quickstart
CI workflow.

## Reproducible environment CI workflow

This workflow configures a specific version of R on a specific runner,
and recreates an environment based on a snapshot created by the `renv`
package. It then runs R CMD check via the
[rcmdcheck](https://github.com/r-lib/rcmdcheck) package.

1.  You want to ensure reproducibility with a specific operating system
    and version of R, and consistent package versions.
2.  Your package has a lockfile created with the `renv` environment.

<!-- end list -->

``` yaml
on: [push, pull_request]

name: R-CMD-check

jobs:
  R-CMD-check:
    runs-on: ${{ matrix.config.os }}

    strategy:
      matrix:
        config:
        - { os: ubuntu-18.04, r: '3.6.1'}

    steps:
      - uses: actions/checkout@v1

      - uses: r-lib/actions/setup-r@master
        with:
            r-version: ${{ matrix.config.r }}

      - name: Query dependencies
        run: Rscript -e "install.packages('remotes')" -e "saveRDS(remotes::dev_package_deps(dependencies = TRUE), 'depends.Rds')"

      - name: Install system dependencies
        if: runner.os == 'Linux'
        env:
          RHUB_PLATFORM: linux-x86_64-ubuntu-gcc
        run: |
          Rscript -e "remotes::install_github('r-hub/sysreqs')"
          sysreqs=$(Rscript -e "cat(sysreqs::sysreq_commands('DESCRIPTION'))")
          sudo -s eval "$sysreqs"

      - name: Cache R packages
        if: runner.os != 'Windows'
        uses: actions/cache@v1
        with:
          path: ${{ env.R_LIBS_USER }}
          key: ${{ runner.os }}-r-${{ matrix.config.r }}-${{ hashFiles('depends.Rds') }}
          restore-keys: ${{ runner.os }}-r-${{ matrix.config.r }}-

      - name: Install renv 0.9.2
        run: Rscript -e "install.packages('https://cran.r-project.org/src/contrib/renv_0.9.2.tar.gz', repos = NULL, type = 'source')"

      - name: Install rcmdcheck latest
        run: Rscript -e "install.packages('rcmdcheck')"

      - name: Check
        run: Rscript -e "renv::restore();rcmdcheck::rcmdcheck(args = '--no-manual', error_on = 'error')"
```

### When can it be used?

1.  You have a simple R package
2.  There is no OS-specific code
3.  You want a quick start with R CI

<!-- end list -->

``` yaml
on: [push, pull_request]

name: R-CMD-check

jobs:
  R-CMD-check:
    runs-on: macOS-latest
    steps:
      - uses: actions/checkout@v1
      - uses: r-lib/actions/setup-r@master
      - name: Install dependencies
        run: Rscript -e "install.packages(c('remotes', 'rcmdcheck'))" -e "remotes::install_deps(dependencies = TRUE)"
      - name: Check
        run: Rscript -e "rcmdcheck::rcmdcheck(args = '--no-manual', error_on = 'error')"
```

## When it can be used?

1.  You have a complex R package
2.  With OS-specific code
3.  And you want to ensure compatibility with older R versions

<!-- end list -->

``` yaml
on:
  push:
    branches:
      - master
  pull_request:
    branches:
      - master

name: R-CMD-check

jobs:
  R-CMD-check:
    runs-on: ${{ matrix.config.os }}

    name: ${{ matrix.config.os }} (${{ matrix.config.r }})

    strategy:
      fail-fast: false
      matrix:
        config:
        - { os: windows-latest, r: '3.6'}
        - { os: macOS-latest, r: '3.6'}
        - { os: macOS-latest, r: 'devel'}
        - { os: ubuntu-16.04, r: '3.2', cran: "https://demo.rstudiopm.com/all/__linux__/xenial/latest"}
        - { os: ubuntu-16.04, r: '3.3', cran: "https://demo.rstudiopm.com/all/__linux__/xenial/latest"}
        - { os: ubuntu-16.04, r: '3.4', cran: "https://demo.rstudiopm.com/all/__linux__/xenial/latest"}
        - { os: ubuntu-16.04, r: '3.5', cran: "https://demo.rstudiopm.com/all/__linux__/xenial/latest"}
        - { os: ubuntu-16.04, r: '3.6', cran: "https://demo.rstudiopm.com/all/__linux__/xenial/latest"}

    env:
      R_REMOTES_NO_ERRORS_FROM_WARNINGS: true
      CRAN: ${{ matrix.config.cran }}

    steps:
      - uses: actions/checkout@v1

      - uses: r-lib/actions/setup-r@master
        with:
          r-version: ${{ matrix.config.r }}

      - uses: r-lib/actions/setup-pandoc@master

      - name: Query dependencies
        run: Rscript -e "install.packages('remotes')" -e "saveRDS(remotes::dev_package_deps(dependencies = TRUE), 'depends.Rds')"

      - name: Cache R packages
        if: runner.os != 'Windows'
        uses: actions/cache@v1
        with:
          path: ${{ env.R_LIBS_USER }}
          key: ${{ runner.os }}-r-${{ matrix.config.r }}-${{ hashFiles('depends.Rds') }}
          restore-keys: ${{ runner.os }}-r-${{ matrix.config.r }}-

      - name: Install system dependencies
        if: runner.os == 'Linux'
        env:
          RHUB_PLATFORM: linux-x86_64-ubuntu-gcc
        run: |
          Rscript -e "remotes::install_github('r-hub/sysreqs')"
          sysreqs=$(Rscript -e "cat(sysreqs::sysreq_commands('DESCRIPTION'))")
          sudo -s eval "$sysreqs"

      - name: Install dependencies
        run: Rscript -e "library(remotes)" -e "update(readRDS('depends.Rds'))" -e "remotes::install_cran('rcmdcheck')"

      - name: Check
        run: Rscript -e "rcmdcheck::rcmdcheck(args = '--no-manual', error_on = 'warning', check_dir = 'check')"

      - name: Upload check results
        if: failure()
        uses: actions/upload-artifact@master
        with:
          name: ${{ runner.os }}-r${{ matrix.config.r }}-results
          path: check

      - name: Test coverage
        if: matrix.config.os == 'macOS-latest' && matrix.config.r == '3.6'
        run: |
          Rscript -e 'covr::codecov(token = "${{secrets.CODECOV_TOKEN}}")'
```

## Commands workflow

This workflow enables the use of 2 R specific commands in pull request
issue comments. `\document` will use
[roxygen2](https://roxygen2.r-lib.org/) to rebuild the documentation for
the package and commit the result to the pull request. `\style` will use
[styler](https://styler.r-lib.org/) to restyle your package.

## When it can they be used?

1.  You get frequent pull requests, often with documentation only fixes.
2.  You regularly style your code with styler, and require all additions
    be styled as well.

<!-- end list -->

``` yaml
on:
  issue_comment:
    types: [created]
name: Commands
jobs:
  document:
    if: startsWith(github.event.comment.body, '/document')
    name: document
    runs-on: macOS-latest
    steps:
      - uses: actions/checkout@v1
      - uses: r-lib/actions/pr-fetch@master
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
      - uses: r-lib/actions/setup-r@master
      - name: Install dependencies
        run: Rscript -e 'install.packages(c("remotes", "roxygen2"))' -e 'remotes::install_deps(dependencies = TRUE)'
      - name: Document
        run: Rscript -e 'roxygen2::roxygenise()'
      - name: commit
        run: |
          git add man/\* NAMESPACE
          git commit -m 'Document'
      - uses: r-lib/actions/pr-push@master
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
  style:
    if: startsWith(github.event.comment.body, '/style')
    name: document
    runs-on: macOS-latest
    steps:
      - uses: actions/checkout@master
      - uses: r-lib/actions/pr-fetch@master
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
      - uses: r-lib/actions/setup-r@master
      - name: Install dependencies
        run: Rscript -e 'install.packages("styler")'
      - name: style
        run: Rscript -e 'styler::style_pkg()'
      - name: commit
        run: |
          git add \*.R
          git commit -m 'style'
      - uses: r-lib/actions/pr-push@master
        with:
          repo-token: ${{ secrets.GITHUB_TOKEN }}
  # A mock job just to ensure we have a successful build status
  finish:
    runs-on: ubuntu-latest
    steps:
      - run: true
```

## Render README

This example automatically re-builds this README.md from README.Rmd
whenever it or its yaml dependencies change and commits the results to
the master branch.

``` yaml
on:
  push:
    paths:
      - examples/README.Rmd
      - examples/*yaml

name: Render README

jobs:
  render:
    name: Render README
    runs-on: macOS-latest
    steps:
      - uses: actions/checkout@v1
      - uses: r-lib/actions/setup-r@v1
      - uses: r-lib/actions/setup-pandoc@v1
      - name: Install rmarkdown
        run: Rscript -e 'install.packages("rmarkdown")'
      - name: Render README
        run: Rscript -e 'rmarkdown::render("examples/README.Rmd")'
      - name: Commit results
        run: |
          git commit examples/README.md -m 'Re-build README.Rmd' || echo "No changes to commit"
          git push https://${{github.actor}}:${{secrets.GITHUB_TOKEN}}@github.com/${{github.repository}}.git HEAD:${{ github.ref }} || echo "No changes to commit"
```

## Build pkgdown site

This example builds a pkgdown site for a repository and pushes the built
package to the gh-pages branch.

``` yaml
on:
  push:
    branches: master

name: Pkgdown

jobs:
  pkgdown:
    runs-on: macOS-latest
    steps:
      - uses: actions/checkout@master
      - uses: r-lib/actions/setup-r@master
      - uses: r-lib/actions/setup-pandoc@master
      - name: Install dependencies
        run: |
          Rscript -e 'install.packages("remotes")' \
                  -e 'remotes::install_deps(dependencies = TRUE)' \
                  -e 'remotes::install_github("jimhester/pkgdown@github-actions-deploy")'
      - name: Install package
        run: R CMD INSTALL .
      - name: Deploy package
        run: |
          Rscript -e "pkgdown:::deploy_local(new_process = FALSE, remote_url = 'https://x-access-token:${{secrets.GITHUB_TOKEN}}@github.com/${{github.repository}}.git')"
```
