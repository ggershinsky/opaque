<img src="https://ucbrise.github.io/opaque/opaque.svg" width="315" alt="Opaque">

**Secure Apache Spark SQL**

[![Build Status](https://travis-ci.org/ucbrise/opaque.svg?branch=master)](https://travis-ci.org/ucbrise/opaque)

Opaque is a package for Apache Spark SQL that enables strong security for DataFrames using Intel SGX trusted hardware. The aim is to enable analytics on sensitive data in an untrusted cloud. Opaque allows encrypting the contents of a DataFrame. Subsequent operations on them will run within SGX enclaves.

This project is based on our NSDI 2017 paper [1]. The oblivious execution mode is not included in this release.

Disclaimers: This is an alpha preview of Opaque, which means the software is still in development (not production-ready!).

Work-in-progress:

- Currently, Opaque supports a subset of Spark SQL operations and not yet UDFs. We are working on adding support for UDFs.

- If you find bugs in the code, please file an issue.

[1] Wenting Zheng, Ankur Dave, Jethro Beekman, Raluca Ada Popa, Joseph Gonzalez, and Ion Stoica.
[Opaque: An Oblivious and Encrypted Distributed Analytics Platform](https://people.eecs.berkeley.edu/~wzheng/opaque.pdf). NSDI 2017, March 2017.

## Installation

After downloading the Opaque codebase, build and test it as follows:

1. Download and install the latest packages of Intel SGX LINUX from https://01.org/intel-software-guard-extensions/downloads.

2. Install Trust Management Framework (TruCE) and its dependencies, 
https://github.com/IBM/sgx-trust-management. 
Build it in IAS simulation mode. 

3. In a separate window, run the Truce server:

    ```sh
    # Use TruCE installation path from step 2.
    cd /path-to/sgx-trust-management/service_provider
    ./truce_server
    ```

4. In a separate window, compile and run the key store:

    ```sh
    cd ${OPAQUE_HOME}/src/keystore
    # Use TruCE installation path from step 2.
    export TRUCE_SDK=/path-to/sgx-trust-management/client
    source sgxsdk/environment # from SGX SDK install directory in step 1

    make
    export LD_LIBRARY_PATH=/path-to/sgx-trust-management/client
    # address of Truce server
    ./key_store 127.0.0.1
    ```

5. Install other dependencies:

    ```sh
    # For Ubuntu 16.04:
    sudo apt-get install build-essential ocaml automake autoconf libtool wget python default-jdk cmake libssl-dev

    ```


6. Set the following environment variables:

    ```sh
    # Use TruCE and SSL-SGX installation paths from step 2.
    export TRUCE_SDK=/path-to/sgx-trust-management/application
    export SSL_SGX=/path-to/sgxssl
    export LD_LIBRARY_PATH=/path-to/sgx-trust-management/application

    source sgxsdk/environment # from SGX SDK install directory in step 1
    export SPARKSGX_DATA_DIR=${OPAQUE_HOME}/data
    ```

    If running with real SGX hardware, also set `export SGX_MODE=HW` and `export SGX_PRERELEASE=1`.

7. Create /etc/opaque folder and copy the  ${OPAQUE_HOME}/src/truce.config file there (if needed, modify the addresses of truce server and key store in the config file).

8. Run the Opaque tests:

    ```sh
    cd ${OPAQUE_HOME}
    build/sbt test
    ```

## Usage

Next, run Apache Spark SQL queries with Opaque as follows, assuming Spark is already installed:

1. Package Opaque into a JAR:

    ```sh
    cd ${OPAQUE_HOME}
    build/sbt package
    ```

2. In separate windows, start Truce server and Key store (if not running).
    ```sh
    cd /path-to/sgx-trust-management/service_provider
    ./truce_server

    cd ${OPAQUE_HOME}/src/keystore
    export LD_LIBRARY_PATH=/path-to/sgx-trust-management/client
    source sgxsdk/environment # from SGX SDK install directory
    # address of Truce server
    ./key_store 127.0.0.1
    ```

3. Set the following environment variables:

    ```sh
    export LD_LIBRARY_PATH=/path-to/sgx-trust-management/application
    source sgxsdk/environment # from SGX SDK install directory
    ```

4. Launch the Spark shell with Opaque:

    ```sh
    ${SPARK_HOME}/bin/spark-shell --jars ${OPAQUE_HOME}/target/scala-2.11/opaque_2.11-0.1.jar
    ```

5. Inside the Spark shell, import Opaque's DataFrame methods and install Opaque's query planner rules:

    ```scala
    import edu.berkeley.cs.rise.opaque.implicits._

    edu.berkeley.cs.rise.opaque.Utils.initSQLContext(spark.sqlContext)
    ```

6. Create an encrypted DataFrame:

    ```scala
    val data = Seq(("foo", 4), ("bar", 1), ("baz", 5))
    val df = spark.createDataFrame(data).toDF("word", "count")
    val dfEncrypted = df.encrypted
    ```

7. Query the DataFrames and explain the query plan to see the secure operators:


    ```scala
    dfEncrypted.filter($"count" > lit(3)).explain(true)
    // [...]
    // == Optimized Logical Plan ==
    // EncryptedFilter (count#6 > 3)
    // +- EncryptedLocalRelation [word#5, count#6]
    // [...]

    dfEncrypted.filter($"count" > lit(3)).show
    // +----+-----+
    // |word|count|
    // +----+-----+
    // | foo|    4|
    // | baz|    5|
    // +----+-----+
    ```

8. Save and load an encrypted DataFrame:

    ```scala
    dfEncrypted.write.format("edu.berkeley.cs.rise.opaque.EncryptedSource").save("dfEncrypted")
    // The file dfEncrypted/part-00000 now contains encrypted data

    import org.apache.spark.sql.types._
    val df2 = (spark.read.format("edu.berkeley.cs.rise.opaque.EncryptedSource")
      .schema(StructType(Seq(StructField("word", StringType), StructField("count", IntegerType))))
      .load("dfEncrypted"))
    df2.show
    // +----+-----+
    // |word|count|
    // +----+-----+
    // | foo|    4|
    // | bar|    1|
    // | baz|    5|
    // +----+-----+
    ```

## Contact

If you want to know more about our project or have questions, please contact Wenting (wzheng@eecs.berkeley.edu) and/or Ankur (ankurdave@gmail.com).
