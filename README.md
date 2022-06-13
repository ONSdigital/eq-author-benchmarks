# EQ Performance Benchmark

This is a performance benchmarking tool designed to measure the performance of [EQ Survey Runner](https://github.com/ONSDigital/eq-survey-runner) using [locust](https://locust.io/).

This repository was heavily inspired by the [census performance tests](https://github.com/ONSdigital/census-eq-performance-tests).

## Installation

On MacOSX Catalina (10.15) if the Python packages fail to install there may be two versions of command line tools SDK present(`MacOSX10.15.sdk` and `MacOSX10.14.sdk`).
You need to remove 10.14 version by using:

```
sudo rm -rf /Library/Developer/CommandLineTools/SDKs/MacOSX10.14.sdk
```

After completing installation, run the command:

```
pipenv install
```

## Running a benchmark

The benchmark consumes a requests JSON file that contains a list of HTTP requests. This can either be created from scratch or generated from a HAR file. Example requests files can be found in the `requests` folder.

To run a benchmark, use:

```bash
pipenv run ./run.sh <REQUESTS_JSON> <HOST: Optional>
```

e.g.

```bash
pipenv run ./run.sh requests/test_checkbox.json
```

This will run 1 minute of locust requests with 1 user and no wait time between requests. The output files are `output_stats.csv`, `output_stats_history.csv` and `output_failures.csv`.

To edit the amount of time the tests will be run for, edit the `run.sh` file's `-t` parameter to the required amount of time.

e.g.

```
USER_WAIT_TIME_MIN_SECONDS=0 USER_WAIT_TIME_MAX_SECONDS=0 REQUESTS_JSON="${1}" HOST="${2:-http://localhost:5000}" pipenv run locust --headless -u 1 -r 1 -t 30m --csv=${4:-output} -L WARNING
```

This will run the tests for 30 minutes.

For the web interface:

```bash
REQUESTS_JSON=requests/test_checkbox.json HOST=http://localhost:5000 pipenv run locust
```

## Configuration

The following environment variables can be used to configure the locust test:

- `REQUESTS_JSON` - The filepath of the requests file relative to the base project directory
  - Defaults to `requests.json`
- `HOST` - The host name the benchmark is run against.
- `USER_WAIT_TIME_MIN_SECONDS` - The minimum delay between each user's GET requests
  - defaults to zero
- `USER_WAIT_TIME_MAX_SECONDS` - The maximum delay between each user's GET requests
  - defaults to zero

## Performance test environment

It is recommended that these performance tests are run against an environment which does not have any live users to ensure the test results are consistent. This environment can be setup through the [eq-author-terraform repository](https://github.com/ONSdigital/eq-author-terraform)

## Generating a requests file

Open the network inspector in Chrome or Firefox and ensure 'preserve log' is ticked. Run your manual or functional test.

**Important:** The captured test should not include the `/session` endpoint.

After the test is complete, right-click on one of the requests in the network inspector and save the log as a HAR file. To generate a requests file from the HAR file run:

```bash
pipenv run python generate_requests.py <HAR_FILEPATH> <REQUESTS_FILEPATH> <SCHEMA_NAME>
```

e.g.

```bash
pipenv run python generate_requests.py requests.har requests/test_checkbox.json test_checkbox
```

---

## Deployment with [Helm](https://helm.sh/)

To deploy this application with helm, you must have a kubernetes cluster already running and be logged into the cluster.

Log in to the cluster using:

```
gcloud container clusters get-credentials survey-runner --region <region> --project <gcp_project_id>
```

You need to have Helm installed locally

1. Install Helm with `brew install kubernetes-helm` and then run `helm init --client-only`

2. Install Helm Tiller plugin for _Tillerless_ deploys `helm plugin install https://github.com/rimusz/helm-tiller`

### Deploying the app

To deploy the app to the cluster, run the following command:

```
helm tiller run \
-    helm upgrade --install \
-    runner-benchmark \
-    k8s/helm \
-    --set host=${HOST} \
-    --set container.image=${DOCKER_REGISTRY}/eq-survey-runner-benchmark:${IMAGE_TAG}
```

If you want to vary the default parameters Locust uses on start, you can specify them using the following variables:

- requestsJson - The filepath of the requests file relative to the base project directory
  - Defaults to `requests.json`
- locustOptions - The host name the benchmark is run against.
- userWaitTimeMinSeconds - The minimum delay between each user's GET requests
  - defaults to 1
- userWaitTimeMaxSeconds - The maximum delay between each user's GET requests
  - defaults to 2
- output.bucket - Name of the GCS bucket in which the output should be stored.
- output.directory - Name of the directory within the GCS bucket in which the output should be stored.

e.g

```
helm tiller run \
-    helm upgrade --install \
-    runner-benchmark \
-    k8s/helm \
-    --set requestsJson=requests/census_individual_gb_eng.json \
-    --set locustOptions="--clients 1000 --hatch-rate 50 -L WARNING" \
-    --set host=https://your-runner.gcp.dev.eq.ons.digital \
-    --set container.image=eu.gcr.io/census-eq-ci/eq-survey-runner-benchmark:latest
```

## Visualising Benchmark Results

You can use the `visualise_results.py` script to visualise benchmark results over time.

### Summarise the Daily Benchmark results

You can get a breakdown of the average response times for a result set by doing:

```bash
OUTPUT_DIR="output-results" \
OUTPUT_DATE="2020-01-01" \
pipenv run python -m scripts.get_summary
```

`output_stats.csv` must be contained in the directory `output-results/2020-01-01` for the above command to work.

The parameter `OUTPUT_DATE` should be edited to represent the date the tests were completed. The directory should also be edited to represent the same correct date.

This will output something like:

```
2020-01-01
---
Percentile Averages:
50th: 92ms
90th: 252ms
95th: 320ms
99th: 467ms
99.9th: 664ms
---
GETs (99th): 466ms
POSTs (99th): 467ms
---
Total Requests: 307,207
Total Failures: 0
Error Percentage: 0.0%

```

If `OUTPUT_DATE` is not provided, then it will output a summary for all results within the provided directory.

### Summarise the Stress Test results:

To get a breakdown of results for a stress test use the `get_aggregated_summary` script. This accepts a folder containing
results as a parameter, and will provide aggregate totals at the folder level:

```bash
OUTPUT_DIR="outputs/stress-test" pipenv run python -m scripts.get_aggregated_summary
```

This will output something like:

```
---
Percentile Averages:
50th: 76ms
90th: 147ms
95th: 177ms
99th: 254ms
99.9th: 372ms
---
GETs (99th): 235ms
POSTs (99th): 273ms
---
Total Requests: 3,209,821
Total Failures: 0
Error Percentage: 0.0%
```

### Run the Visualise Results script

The `visualise_results` script will run against any benchmark results stored in the directory that is passed as an environment variable to the script.
Optionally, you can also specify the number of days to visualise the results for.

For example, to visualise results for the last 7 days:

```bash
OUTPUT_DIR="outputs/daily-test" \
NUMBER_OF_DAYS="7" \
pipenv run python -m scripts.visualise_results
```

A line chart will be generated and saved as `performance_graph.png`.
