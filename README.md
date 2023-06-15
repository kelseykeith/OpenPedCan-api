# OpenPedCan-api <!-- omit in toc -->

[![GitHub Super-Linter](https://github.com/PediatricOpenTargets/OpenPedCan-api/workflows/Lint%20Code%20Base/badge.svg)](https://github.com/marketplace/actions/super-linter)

This branch has complete, but not reviewed or merged, code for the copy number variant evidence page view database. See `db/build_tools/cnv_tools/` for the database build code.

---

`OpenPedCan-api` implements OpenPedCan (Open Pediatric Cancers) project public API (application programming interface) to transfer [OpenPedCan-analysis](https://github.com/PediatricOpenTargets/OpenPedCan-analysis) results and plots via HTTP, which is publicly available at <https://openpedcan-api.d3b.io/__docs__/>.

- [1. API endpoint specifications](#1-api-endpoint-specifications)
  - [1.1. `includeTumorDesc` parameter in `/tpm/*` endpoints](#11-includetumordesc-parameter-in-tpm-endpoints)
  - [1.2. `rankGenesBy` parameter in `/dge/top-gene-disease-gtex-diff-exp/*` endpoints](#12-rankgenesby-parameter-in-dgetop-gene-disease-gtex-diff-exp-endpoints)
- [2. `OpenPedCan-api` server deployment](#2-openpedcan-api-server-deployment)
- [3. Test run `OpenPedCan-api` server locally](#3-test-run-openpedcan-api-server-locally)
  - [3.1. `git clone` `OpenPedCan-api` repository](#31-git-clone-openpedcan-api-repository)
  - [3.2. Prepare Docker environment files](#32-prepare-docker-environment-files)
  - [3.3. Run static code analysis](#33-run-static-code-analysis)
  - [3.4. (Optional) Build `OpenPedCan-api` database locally](#34-optional-build-openpedcan-api-database-locally)
  - [3.5. Build and run `OpenPedCan-api` HTTP server and database server docker images](#35-build-and-run-openpedcan-api-http-server-and-database-server-docker-images)
  - [3.6. Test `OpenPedCan-api` server](#36-test-openpedcan-api-server)
- [4. API system design](#4-api-system-design)
  - [4.1. Data model layer](#41-data-model-layer)
  - [4.2. Analysis logic layer](#42-analysis-logic-layer)
  - [4.3. API layer](#43-api-layer)
  - [4.4. HTTP server layer](#44-http-server-layer)
  - [4.5. Testing layer](#45-testing-layer)
  - [4.6. Deployment layer](#46-deployment-layer)
- [5. API Development roadmap](#5-api-development-roadmap)

## 1. API endpoint specifications

<https://openpedcan-api-qa.d3b.io/__docs__/> specifies the following API endpoint attributes.

- HTTP request method
- Path
- Parameters
- Response media type

### 1.1. `includeTumorDesc` parameter in `/tpm/*` endpoints

`includeTumorDesc` parameter determines how independent primary and relapse tumor samples should be handled, which takes one of the following four values.

- `primaryOnly`: Only include independent primary tumor samples of the `(Disease, Dataset)` tuples, aka `(cancer_group, cohort)` tuples, that have `>= 3` independent primary tumor samples.
- `relapseOnly`: Only include independent relapse tumor samples of the `(Disease, Dataset)` tuples that have `>= 3` independent relapse tumor samples.
- `primaryAndRelapseInSameBox`: Only include independent primary and relapse tumor samples of the `(Disease, Dataset)` tuples that have `>= 3` independent tumor samples that include both primary and relapse tumor samples. In boxplot, show independent primary and relapse tumor samples of the same `(Disease, Dataset)` tuple in the same box. In summary table, list independent primary and relapse tumor samples of a `(Disease, Dataset)` tuple in the same row.
- `primaryAndRelapseInDifferentBoxes`: Only include independent primary and relapse tumor samples of `(Disease, Dataset)` tuples that have three or more primary tumor samples and three or more relapse tumor samples. In boxplot, show independent primary and relapse tumor samples of the same `(Disease, Dataset)` tuple in different boxes of the same x-label. In summary table, list independent primary and relapse tumor samples of a `(Disease, Dataset)` tuple in different rows.

### 1.2. `rankGenesBy` parameter in `/dge/top-gene-disease-gtex-diff-exp/*` endpoints

`rankGenesBy` parameter determines how differentially expressed genes are ranked for each disease.

- `cgc_all_gene_up_reg_rank`: Ranking of up-regulation among all genes in each `cancer_group` and `cohort`, comparing to GTEx tissues. `cgc` is a shorthand for `(cancer_group, cohort)` tuple.
- `cgc_all_gene_down_reg_rank`: Ranking of down-regulation among all genes in each `cancer_group` and `cohort`, comparing to GTEx tissues.
- `cgc_all_gene_up_and_down_reg_rank`: Ranking of up- and down-regulation among all genes in each `cancer_group` and `cohort`, comparing to GTEx tissues.
- `cgc_pmtl_gene_up_reg_rank`: Ranking of up-regulation among PMTL genes in each `cancer_group` and `cohort`, comparing to GTEx tissues. If rank genes by this option, only PMTL genes will be included in database query result.
- `cgc_pmtl_gene_down_reg_rank`: Ranking of down-regulation among PMTL genes in each `cancer_group` and `cohort`, comparing to GTEx tissues. If rank genes by this option, only PMTL genes will be included in database query result.
- `cgc_pmtl_gene_up_and_down_reg_rank`: Ranking of up- and down-regulation among PMTL genes in each `cancer_group` and `cohort`, comparing to GTEx tissues. If rank genes by this option, only PMTL genes will be included in database query result.

## 2. `OpenPedCan-api` server deployment

`OpenPedCan-api` server is deployed using Amazon Web Services (AWS). `OpenPedCan-api` HTTP server is deployed using Amazon Elastic Container Registry (ECR), Elastic Container Service (ECS), and Fargate. The HTTP server queries `OpenPedCan-api` database server, and the database server is deployed using Amazon Relational Database Service (RDS).

<https://openpedcan-api.d3b.io/__docs__/> is the URL of `OpenPedCan-api` PRD (production) server. The PRD server will only deploy the latest release of the repository.

<https://openpedcan-api-qa.d3b.io/__docs__/> is the URL of `OpenPedCan-api` QA (quality assurance) server. The QA server will only deploy the latest commit to the `main` branch of the repository.

<https://openpedcan-api-dev.d3b.io/__docs__/> is the URL of `OpenPedCan-api` DEV (development) server. The DEV server will deploy the latest commit to any branch of the repository.

`OpenPedCan-api` HTTP server is deployed with the following steps, according to comments and messages by @blackdenc .

- Build and tag `OpenPedCan-api` docker image using `Dockerfile`.
- Push the built image to ECR.
- Pass the ECR docker image tag to Amazon Elastic Container Service (ECS) Fargate task definition at runtime. The following environment variables are used for database server connection:
  - DB_USERNAME
  - DB_PASSWORD
  - DB_PORT
  - DB_HOST
  - DB_NAME

`OpenPedCan-api` database server is deployed with the following steps:

- Start an RDS instance.
- Load data into the RDS instance from an EC2 instance.

## 3. Test run `OpenPedCan-api` server locally

Test run `OpenPedCan-api` server with the following steps:

- `git clone` `OpenPedCan-api` repository. Checkout a branch/commit that needs to be tested.
- Prepare Docker environment files.
- Run static code analysis.
- (Optional) Build `OpenPedCan-api` database locally. This step takes about 40GB memory and 500GB disk space. This step is optional, because pre-built `OpenPedCan-api` database dump file is publicly available via HTTP.
- Build and run `OpenPedCan-api` HTTP server and database server docker images. The database docker container can initialize database either using local or remote pre-built database dump file. This step takes less than 10GB memory and about 250GB disk space.
- Test `OpenPedCan-api` server.

Note that this test run procedure has only been tested on linux operating system, with the following environment.

```text
Working directory is the git repository root directory, i.e. the directory
that contains the .git directory of the repository.

ubuntu 20.04
docker 20.10
docker-compose 1.29.2
curl 7.79
git 2.25
ImageMagick 6.9
shellcheck 0.7
sha256sum 8.30
md5sum 8.30
R 4.1
R package readr 2.0.2
R package jsonlite 1.7.2
R package lintr 2.0.1
R package httr 1.4.2
R package testthat 3.0.4
R package glue 1.4.2
R package stringr 1.4.0
```

@brianghig found that `OpenPedCan-api` server cannot be run on MacOS 12.5. For more details, see <https://github.com/PediatricOpenTargets/OpenPedCan-api/pull/77>.

### 3.1. `git clone` `OpenPedCan-api` repository

```bash
# Change URL if a fork repo needs to be used
git clone https://github.com/PediatricOpenTargets/OpenPedCan-api.git

cd OpenPedCan-api

git checkout -t origin/the-branch-that-needs-to-be-tested
# Optionally, checkout a commit with the following command
#
# git checkout COMMIT_HASH_ID
```

### 3.2. Prepare Docker environment files

Prepare the following `OpenPedCan-api` Docker environment files for building database and running database and HTTP servers locally.

The following `../OpenPedCan-api-secrets` paths are relative to the root directory of this git repository. For example, the structure of the parent directory of `OpenPedCan-api-secrets` directory may look like the following:

```text
.
├── OpenPedCan-api
└── OpenPedCan-api-secrets
```

- `../OpenPedCan-api-secrets/access_db.env`

  ```bash
  # Docker env vars for accessing database.

  # Environment file format reference:
  # https://docs.docker.com/engine/reference/commandline/run/#set-environment-variables--e---env---env-file

  # User for database read-only access.
  #
  # Note that password cannot contain : or \ for simplicity.
  # https://www.postgresql.org/docs/current/libpq-pgpass.html
  DB_USERNAME=my_db_read_only_username
  DB_PASSWORD=my_db_read_only_user_password
  ```

- `../OpenPedCan-api-secrets/common_db.env`

  ```bash
  # Docker env vars for common database configs.

  # The env vars defined in this file cannot be changed without modifying other
  # files, because various commands and scripts assume that they have the
  # following values.
  DB_PORT=5432
  DB_HOST=db
  DB_DRIVER=PostgreSQL Unicode

  DB_NAME=open_ped_can_db
  BULK_EXP_SCHEMA=bulk_expression
  BULK_EXP_TPM_HISTOLOGY_TBL=bulk_expression_tpm_histology
  BULK_EXP_DIFF_EXP_TBL=bulk_expression_diff_exp
  ```

- `../OpenPedCan-api-secrets/load_db.env`

  ```bash
  # Docker env vars for loading database.

  # User for loading database with read-write access.
  #
  # Note that password cannot contain : or \ for simplicity.
  # Ref: https://www.postgresql.org/docs/current/libpq-pgpass.html
  DB_READ_WRITE_USERNAME=my_db_rw_username
  DB_READ_WRITE_PASSWORD=my_db_rw_user_password

  # The following env vars in this file cannot be changed without modifying
  # other files, because various commands and scripts assume that they have the
  # following values.

  # postgres docker image env vars.
  #
  # Ref: https://hub.docker.com/_/postgres
  POSTGRES_USER=postgres
  POSTGRES_DB=postgres
  POSTGRES_PASSWORD=my_postgres_user_password

  POSTGRES_HOST_AUTH_METHOD=scram-sha-256
  POSTGRES_INITDB_ARGS=--auth-local=scram-sha-256 --auth-host=scram-sha-256

  # Path to the directory that contain database build outputs in container.
  BUILD_OUTPUT_DIR_PATH=/home/open-ped-can-api-db/db/build_outputs
  ```

These environment files pass secret information to docker container environment. Although local development can use plain text in code and configurations without worrying about any security issues, these environment files are used to emulate production environment, so that locally developed systems can be deployed in production environment more straightforwardly. The consistent secret handling method in local and production environment is also more straightforward to `OpenPedCan-api` developers. These environment files also set the same environment variable values for different components of `OpenPedCan-api`, so that developing and testing can be more straightforward.

### 3.3. Run static code analysis

```bash
./tests/run_linters.sh
```

If there is any syntax error, comment in the GitHub pull request with the full error messages.

`./tests/run_linters.sh` analyzes R, Docker, and shell files using `R lintr`, `hadolint`, and `shellcheck` respectively.

### 3.4. (Optional) Build `OpenPedCan-api` database locally

Use the following bash command to build `OpenPedCan-api` database locally. This step takes about 40GB memory and 500GB disk space.

```bash
./db/build_db.sh
```

`./db/build_db.sh` runs the following steps:

- Download `OpenPedCan-analysis` and other required data.
- Build a docker image `open-ped-can-api-build-db` using `./db/build_tools/build_db.Dockerfile`.
- Run `open-ped-can-api-build-db` docker image with `./OpenPedCan-analysis/` and `./db/build_outputs/` directories bind mounted by default, to build `OpenPedCan-api` database with the following steps:
  - Initialize database management system (DBMS).
  - Create DBMS users.
  - Create `open_ped_can` database.
  - Create `open_ped_can` database schema(s).
  - Run `./db/build_tools/*.R` files to create CSV (comma separated values) file(s), with the bind mounted `./db/build_outputs/` directory as output directory by default.
  - Load the CSV file(s) into database using SQL (Structured Query Language) `COPY` command.
  - Dump all database schema(s) and table(s) into compressed SQL command file, with the bind mounted `./db/build_outputs/` directory as output directory by default.
  - Append SQL `CREATE INDEX` commands to the database dump file.
  - Record the `sha256sum` of the database dump file, with `./db/build_outputs/` as output directory by default.
- Check `sha256sum` of the database dump file.

Note for developers: To build a small database, with only a few arbitrarily selected genes and all samples, for efficient development and testing, run `DOWN_SAMPLE_DB_GENES=1 ./db/build_db.sh`.

### 3.5. Build and run `OpenPedCan-api` HTTP server and database server docker images

Use the following bash commands to build and run `OpenPedCan-api` HTTP server and database server docker images.

- Use remote pre-built data model files. Even if there are local pre-built database dump file(s), the docker image will still use the remote pre-built database dump file(s).

  ```bash
  # To clean up previously stopped docker-compose service containers and
  # anonymous volumes attached to them, run the following command:
  #
  # docker-compose rm -f -v 
  #
  # Sometimes, attached anonymous volumes are not removed by this command.
  # Remove them by running docker volume prune -f.

  # Add the following options if necessary.
  #
  # --build               Build images before starting containers.
  # --remove-orphans      Remove containers for services not defined in the
  #                       Compose file.
  # --renew-anon-volumes  Recreate anonymous volumes instead of retrieving
  #                       data from the previous containers.
  docker-compose up
  ```

- Use local pre-built database dump file(s).

  ```bash
  # To clean up previously stopped docker-compose service containers and
  # anonymous volumes attached to them, run the following command:
  #
  # docker-compose rm -f -v 
  #
  # Sometimes, attached anonymous volumes are not removed by this command.
  # Remove them by running docker volume prune -f.

  # Add the following options if necessary.
  #
  # --build               Build images before starting containers.
  # --remove-orphans      Remove containers for services not defined in the
  #                       Compose file.
  # --renew-anon-volumes  Recreate anonymous volumes instead of retrieving
  #                       data from the previous containers.
  DB_LOCATION=local docker-compose up
  ```

### 3.6. Test `OpenPedCan-api` server

Test the running server with the following command.

```bash
./tests/run_tests.sh
```

`./tests/run_tests.sh` sends multiple HTTP requests to `localhost:8082` by default, with the following steps.

- Send an HTTP request using R package `httr`.
- Output the HTTP response body to `tests/http_response_output_files/png` or `tests/http_response_output_files/json`.
- If response body content type is JSON, convert the JSON file to TSV file in `tests/results`.
- Assert HTTP response status code as expected.
- Output HTTP response times in `tests/results/endpoint_response_times.tsv`.
- Summarize HTTP response times as a boxplot in `tests/plots/endpoint_response_time_boxplot.png`.

The port number of `localhost` can be changed by passing the `bash` environment variable `LOCAL_API_HOST_PORT` with a different value, but there has to be a `OpenPedCan-api` server listening on the port. The API HTTP server host can be changed to <https://openpedcan-api-qa.d3b.io/__docs__/> or <https://openpedcan-api-dev.d3b.io/__docs__/>, by passing environment variable `API_HOST=qa` or `API_HOST=dev` respectively.

## 4. API system design

The `OpenPedCan-api` server system has the following layers:

- Data model layer specifies data/database structures.
- Analysis logic layer specifies analytic procedures to generate results and plots using the data model layer.
- API (application programming interface) layer specifies the services/interfaces provided by the system.
- HTTP server layer specifies the procedures to handle every HTTP request.
- Testing layer specifies the procedures to test the `OpenPedCan-api` server.
- Deployment layer specifies the procedures to deploy the `OpenPedCan-api` server.

For more details about implementations, see [Test run `OpenPedCan-api` server locally](#test-run-openpedcan-api-server-locally) section.

The root directory of this repository should only contain starting points of different layers and configuration files.

### 4.1. Data model layer

`db` directory contains files that implement the data model layer.

`db/build_db.sh` builds data model files that are used by analysis logic layer.

`db/load_db.sh` loads local or remote pre-built data model files to the HTTP server layer.

`db/r_interfaces` directory contains files that define data model interfaces for R runtime.

### 4.2. Analysis logic layer

`src` directory contains files that implement the analysis logic layer.

### 4.3. API layer

Discussions in PedOT meetings, Slack work space, GitHub issues, etc specify the API layer.

### 4.4. HTTP server layer

`main.R` runs the `OpenPedCan-api` HTTP server. The HTTP server is implemented using [libuv](http://docs.libuv.org/en/stable/design.html) and [http-parser](https://github.com/nodejs/http-parser) and called by [R package plumber](https://github.com/rstudio/plumber).

The API HTTP server handles every HTTP request [sequentially](https://www.rplumber.io/articles/execution-model.html#performance-request-processing) with the following steps:

- Pre-process the HTTP request.
- Find the API endpoint for handling the request.
- Run the R function defined for the API endpoint.
- Convert the return value of the endpoint R function to defined response content type, e.g. JSON and PNG.
- Send HTTP response to the request address.

### 4.5. Testing layer

The `tests` directory contains all tools and code for testing the API server. `tests/http_response_output_files` contains the API server response plots and tables. `tests/results` and `tests/plots` contain results and plots generated during test run.

### 4.6. Deployment layer

`Jenkinsfile` and `Dockerfile` specify the procedures to deploy the `OpenPedCan-api` server.

## 5. API Development roadmap

- Optimize HTTP server deployment task scaling rules and memory/CPU resource allocations to balance performance and cost.
- Optimize database server deployment memory/CPU resource allocations and DBMS runtime configurations, e.g. `shared_buffers` and `work_mem`, to balance performance and cost.
