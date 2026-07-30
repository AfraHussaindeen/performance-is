# WSO2 Identity Server Performance

WSO2 Identity Server performance artifacts are used to continuously test the performance of the Identity Server.

These performance test scripts make use of the Apache JMeter to run the tests with different concurrent users and different Identity Server version.

The deployment is automated using AWS cloud formation. 

Artifacts in the master branch can be used with the WSO2 Identity Server version 5.10.0.For previous versions, please use below branches
 1. [is-5.9.0](https://github.com/wso2/performance-is/tree/is-5.9.0) for product version 5.9.0
 2. [is-5.8.0](https://github.com/wso2/performance-is/tree/is-5.8.0) for product version 5.8.0 to 5.6.0
 
## About the deployment

At the moment we support for two deployment patterns as,
1. Single node deployment.
  - <img src="common/images/deployment-diagram-singlenode.png" height="400" alt="Single Node Deployment Diagram">

2. Two node cluster deployment.
  - <img src="common/images/deployment-diagram-twonode-cluster.png" height="400" alt="Two Node Cluster Deployment Diagram">

WSO2 Identity Server is setup in an AWS EC2 instance. AWS RDS instance is used to host the MySQL user store and identity databases.

JMeter version 3.3 is installed in a separate node which is used to run the test scripts and gather results from the setup.

## Run Performance Tests

You can run IS Performance Tests from the source using the following instructions.

### Prerequisites

* [Maven 3.5.0 or later](https://maven.apache.org/download.cgi)
* [AWS CLI](https://aws.amazon.com/cli/) - Please make sure to [configure the AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html) and set the output format to `json`.
* [Apache JMeter 3.3](https://jmeter.apache.org/) Setup tarball.
* WSO2 IS server zip file.
* Python 3.5
    * [Jinja2 2.11.1](https://pypi.org/project/Jinja2/)
    * numpy

### Steps to run performance tests.

1. Clone this repository.

```console
git clone https://github.com/wso2/performance-is
```
2. Checkout master branch for the latest Identity Server version or relevant version tag for previous releases.
```console
cd performance-is
git checkout v5.8.0
```
3. Build the artifacts using Maven.
```console
mvn clean install
```

4. Based on your preferred deployment, navigate to `single-node` directory or `two-node-cluster` directory.
4. Run the `start-performance.sh` script. It will take around 15 hours to complete the test round with default settings. Therefore, you might want to use `nohup`. Following is the basic command.
```console
./start-performance.sh -k is-perf-test.pem -a ******* -s ******* -c is-perf-cert -n wso2IS.zip -j apache-jmeter-3.3.tgz -- -d 10 -w 2
```

See usage:

```console
./start-performance.sh -k <key_file>
   -c <certificate_name> -j <jmeter_setup_path>
   [-n <IS_zip_file_path>]
   [-u <db_username>] [-p <db_password>] [-e <db_instance_type>] [-s <db_snapshot_id>] [-r <concurrency>]
   [-i <wso2_is_instance_type>] [-b <bastion_instance_type>] [-t <keystore_type>] [-m <db_type>]
   [-l <is_case_insensitive_username_and_attributes>]
   [-w <minimum_stack_creation_wait_time>] [-h]

-k: The Amazon EC2 key file to be used to access the instances.
-c: The name of the IAM certificate.
-y: The token issuer type.
-q: User tag who triggered the Jenkins build
-r: Concurrency (50-500, 500-3000, 50-3000)
-j: The path to JMeter setup.
-c: The name of the IAM certificate.
-n: The is server zip
-u: The database username. Default: wso2carbon.
-p: The database password. Default: wso2carbon.
-s: The database snapshot ID. Default: -.
-e: The database instance type. Default: db.m6i.2xlarge.
-i: The instance type used for IS nodes. Default: c6i.xlarge.
-b: The instance type used for the bastion node. Default: c6i.xlarge.
-w: The minimum time to wait in minutes before polling for cloudformation stack's CREATE_COMPLETE status.
    Default: 10 minutes.
-g: Number of IS nodes.
-t: Keystore type. Default: PKCS12.
-m: Database type. Default: mysql.
-l: Case insensitivity of the username and attributes. Default: false.
-h: Display this help and exit.
```

### What does the script do?
1. Validate the CloudFormation template with given parameters, using the AWS CLI.
2. Run the CloudFormation template to creat the deployment and wait till the stack creation completes.
3. Extract the following using the AWS CLI.
   * Bastion node public IP. (Used as the JMeter client)
   * Private IP of the WSO2 IS instance.
   * RDS instance hostname.
4. Setup the wso2 IS server in the instance and create the databases.
5. Copy required files such as the key file, performance artifacts and the JMeter setup to the bastion node.
6. SSH into the bastion node and execute the [setup-bastion.sh](single-node/setup/README.md) script, which will setup the additional components in the deployment.
7. SSH into the bastion node and execute the [run-performance-test.sh](common/jmeter/README.md) script, which will run the tests and collect the results.
8. Download the test results from the bastion node.
9. Create summary CSV file and MD file.

## Performance Analysis Graphs

We have added a performance analysis feature to the project, which allows you to generate performance plots based on CSV data files. This feature provides insights into response times for different deployment types and scenarios. By analyzing these performance plots, you can identify performance bottlenecks and make informed optimizations.

To use this feature, we have added the `performance_plots.py` script to the project. This script reads CSV data files, filters the data based on concurrency ranges, and generates performance plots using the matplotlib library. The generated plots are saved in the 'output' folder.

Additionally, we have updated the README.md file in the `performance_analysis` directory to provide detailed instructions on how to use the script, customize the settings, and understand the input CSV data format.

To get started with the performance analysis feature, please refer to the [common/performance_analysis/README.md](common/performance_analysis/README.md) file for instructions and examples.

## Multiple Client Secrets — Performance Test Branches

The multiple-client-secrets performance evaluation is split into one Git branch per run, so
Jenkins can be pointed at a single branch for each measurement. Every branch shares the same
trimmed scenario and Jenkins flags; branches differ only in the IS `deployment.toml` config and
the secret data they set up.

**Common to every run**

- **Scenario:** `00-oauth_client_credential_grant` only, **non-tenant**. The JMeter plan issues each
  `/token` request against a **randomly selected app** from the created population (`noOfSPs`), so
  load is spread uniformly across all apps.
- **Concurrency / mode (Jenkins flags):** `-r 50-500 -v FULL` → 50, 100, 150, 300, 500 users.
  (Must be `FULL`; `QUICK` ignores `-r` and forces a single 200-user point.)
- **App population:** 1000. `AppInfoCache` default capacity is 10000 with no TTL, so a 1000-app
  population is **fully warm (no eviction)**. "Miss" branches force eviction by lowering
  `AppInfoCache` capacity below the population in `deployment.toml`.

**Branch matrix**

| Branch | Run | Pack | Feature flag | Persistence | Secrets/app | Cache state | Measures |
|---|---|---|---|---|---|---|---|
| `master_mcs` | Baseline | raw (no feature) | — | plaintext | 1 | warm | Raw-pack reference |
| `mcs-perf/c1-r1-off` | C1-R1 | feature | OFF | plaintext | 1 | warm | Feature code present, flag off |
| `mcs-perf/c1-r2-on-1sec` | C1-R2 | feature | ON | plaintext | 1 | warm | Enable-by-default, single secret |
| `mcs-perf/c1-r3-on-4sec` | C1-R3 | feature | ON | plaintext | 4 | warm | Multiple secrets, warm |
| `mcs-perf/c2-r1-plain-hit` | C2-R1 | feature | ON | encryption (plaintext cache) | 4 | warm / HIT | Warm-path, plaintext cache |
| `mcs-perf/c2-r2-plain-miss-1` | C2-R2 | feature | ON | encryption (plaintext cache) | 1 | MISS | Encryption miss, single (decrypt) |
| `mcs-perf/c2-r3-plain-miss-4` | C2-R3 | feature | ON | encryption (plaintext cache) | 4 | MISS | Encryption miss, multi (O(N) decrypt) |
| `mcs-perf/c2-r4-hash-hit` | C2-R4 | feature | ON | hashing | 4 | warm / HIT | Warm-path, hashing |
| `mcs-perf/c2-r5-hash-miss` | C2-R5 | feature | ON | hashing | 4 | MISS | Hashing miss (indexed O(1)) |

**Config levers each branch sets (in `deployment.toml` / data setup)**

- **Feature flag:** `oauth.multiple_client_secrets.enable` (and `secret_count`).
- **Persistence mode:** plaintext (raw-pack default, used for the baseline and Category 1),
  encryption, or client-secret hashing (Category 2).
- **Secrets per app:** seed 1 or 4 secrets per app during data setup.
- **Cache state:** `AppInfoCache` capacity/timeout — *warm* = default (capacity 10000, no TTL);
  *miss* = capacity set below the 1000-app population to force eviction.

> Status: `master_mcs` (baseline, scenario trimmed) is ready. The `mcs-perf/*` branches are created
> off `master_mcs`, each applying only its own `deployment.toml` / data-setup changes from the table.

## Legacy Mode

If needed to run the performance test in legacy mode, please use the legacy-mode branch.
Legacy mode will include single node and 2 node setup deployments with previous test flows.
