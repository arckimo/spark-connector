# Spark Connector - Functional Integration Tests

This project is in place to run a series of tests of the connector against a real Vertica database.

Configuration is specified with `application.conf` (HOCON format).

## How to run the tests

From the functional-tests directory, run the following commands:
```sh
mkdir lib
cd ../connector && sbt assembly && cp target/scala-2.12/spark-vertica-connector-assembly-3.3.5.jar ../functional-tests/lib && cd ../functional-tests
```

This will create a lib folder and then build and copy the connector JAR file to it.

### Using HDFS:

From the project's docker directory:
```sh
cd ../docker
docker compose up -d
docker exec -it docker-client-1 bash
```

This will create a docker image for a client container and docker containers for single-node clusters of both Vertica and HDFS.

### Starting Functional Tests

In the client container, change your working directory to functional-tests
```sh
cd /spark-connector/functional-tests
```

Use `sbt` to start the sbt server from the command line. Enter `run` to execute the default test suites. This will run the tests on a local Spark cluster. You can specify arguments to modify you test run. For more details, use `run -h`.

As an example, to include large data tests into the run, use `run -l`. On the other had, using `run -s ComplexTypeTests` will only execute ComplexTypeTests.

To run a specific test in a suite, use option `-t` to specify a test name. For example, `run -s ComplexTypeTests -t "<TEST_NAME>"` will execute the specified tests in the suite. Note that option `-t` has to be used with option `-s`.

### Testing against S3:

Set the appropriate S3-credentials to `application.conf` file or the environment variables. Refer to these connector options on the project's [README](/README.md#connector-options):
```
aws_access_key_id
aws_secret_access_key
aws_region
aws_session_token
aws_credentials_provider
aws_endpoint
aws_enable_ssl
aws_enable_path_style
```

Make sure your update the option `filepath` to your S3 bucket as well.

Alternatively, you can use the configuration used in our S3 example if you want to test against our Minio container.

If a different version of Spark is used, make sure to also update `hadoop-aws` to the appropriate version for the Hadoop install used.

### Using GCS:

Follow the [GCS guide](/docs/gcs-guide.md) to obtain the needed credentials. Then, add the following connector options to `application.conf`:
```
gcs_hmac_key_id
gcs_hmac_key_secret
gcs_service_key_id
gcs_service_key
gcs_service_email
```

Make sure your update the option `filepath` to your GCS bucket as well.

### Testing on a standalone cluster

Our Docker environment also hosts a standalone Spark cluster for use.

To run the functional tests on our cluster, assemble the functional tests:
```sh
sbt assembly
```

Note that you should do this outside of the Docker container as it will be extremely slow to compile inside Docker. Navigate to the `functional-tests` folder on your local machine to build the functional test with `sbt assembly`. Since the `spark-connector` folder is mounted onto the containers, the built JAR will also be available on the client container.

To submit the functional test to our standalone cluster, inside the client container navigate to `spark-connector/functional-tests` and use `submit-functional-tests.sh`.

Once submitted, verify through the [web ui](localhost:8080) and the [jobs ui](localhost:4040) that the application was submitted. Our functional test, without any arguments, will create multiple Spark sessions. You should expect multiple applications executing one after another.

#### Configuring the cluster

The `submit-functional-tests.sh` uses `spark-submit` and thus you can edit any available spark config for the submission.

`submit-functional-tests.sh` will pass its arguments to `spark-submit`, thus passing the arguments to the submitted spark application. For example, to run a single test suite on our cluster, use `./submit-functional-tests.sh -s EndToEndTests`.

Note: `submit-functional-tests.sh` will always prepend the functional test option `-r` to the passed in arguments. So `./submit-functional-tests.sh -s EndToEndTests` is equivalent to `sbt "run -r -s EndToEndTests"` Option `-r` tells our functional test application to configure itself for submitting to a cluster (by omitting the `master` option).

To increase the worker count, change spark version, or any other Spark environment settings, refer to our [docker environment instructions](/docker/README.md).

### Debugging

Some tips for debugging:
- When your application is submitted to a cluster, you can view a worker's log under `/opt/bitnami/spark/work/<APPLICATION_ID>`. This can be useful in cases where a fatal exception caused the worker to crash and loses connection to master.
- Use the provided `submit-functional-tests-debug.sh`, which will wait on port `5005` for a remote JVM debug connection before proceeding with submitting and running the functional tests on our standalone cluster. This allows you to debug the submitted Spark application.
