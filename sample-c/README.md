# Sample C

Create a Lakehouse on Kubernetes with:

- Apache Iceberg as a table format.
- SeaweedFS as object storage.
- SeaweedFS as a catalog for Apache Iceberg.
- DuckDB and Spark SQL as query engines.

## Overview

In this sample, you will create the following Lakehouse:

```
+--[Kubernetes]---------------------------------------------------+
|                                                                 |
|  +----------------+                         +----------------+  |
|  | DuckDB         |---(Iceberg REST API)--->| SeaweedFS      |  |
|  | (Query Engine) |<---(Get Catalog Info)---| (REST Catalog) |  |
|  +-------+--------+                         |                |  |
|          |                                  |                |  |
|  (Read/Write Data and Metadata)             |                |  |
|          |                                  |                |  |
|          +--+------------------+------------+-----+          |  |
|             |                  |            |     |          |  |
|  +----------+------------------+------------+     |          |  |
|  |          |                  |                  |          |  |
|  |          v                  v                  v          |  |
|  |  +---------------+  +---------------+  +---------------+  |  |
|  |  | Iceberg Table |  | Iceberg Table |  | Iceberg Table |  |  |
|  |  +---------------+  +---------------+  +---------------+  |  |
|  |                                                           |  |
|  |  SeaweedFS                                                |  |
|  |  (Object Storage)                                         |  |
|  |                                                           |  |
|  +-----------------------------------------------------------+  |
|                                                                 |
+-----------------------------------------------------------------+
```

## Deploy each component on Kubernetes

### Create a Kubernetes namespace

1. Create a Kubernetes namespace.

    ```shell
    kubectl create ns sample-c
    ```

### Deploy SeaweedFS (Object Storage)

1. Add a Helm repository.

    ```shell
    helm repo add seaweedfs https://seaweedfs.github.io/seaweedfs/helm
    ```

    ```shell
    helm repo update seaweedfs
    ```

1. Create a secret that includes SeaweedFS user information.

    ```shell
    kubectl create secret generic seaweedfs-s3-user --from-file=seaweedfs_s3_config=./seaweedfs/seaweedfs-s3-user.json -n sample-c
    ```

1. Deploy SeaweedFS.

    ```shell
    helm install seaweedfs seaweedfs/seaweedfs -f ./seaweedfs/seaweedfs.yaml -n sample-c --version 4.33.0
    ```

1. Deploy a service resource to access an Iceberg REST API endpoint.

    ```shell
    kubectl apply -f ./seaweedfs/seaweedfs-iceberg-svc.yaml -n sample-c
    ```

1. Deploy a client pod for SeaweedFS.

    ```shell
    kubectl apply -f ./seaweedfs/seaweedfs-client.yaml -n sample-c
    ```

1. [Optional] You can access the SeaweedFS Web UI (Admin Console) at `127.0.0.1` using the `kubectl port-forward` command.

    ```shell
    kubectl port-forward svc/seaweedfs-admin 23646:23646 -n sample-c
    ```

    > Note: The values of `username` / `password` are `admin` / `admin`.

### Deploy DuckDB CLI (Query Engine)

1. Deploy DuckDB CLI as a pod.

    ```shell
    kubectl apply -f ./duckdb/duckdb.yaml -n sample-c
    ```

## Create Iceberg Tables (TPC-H)

### Create a catalog in SeaweedFS

1. Run a shell in the SeaweedFS client pod.

    ```shell
    kubectl exec -it seaweedfs-client -n sample-c -- /bin/sh
    ```

1. Create an S3 Tables Bucket (catalog).

    ```shell
    curl -X POST http://seaweedfs-all-in-one.sample-c.svc.cluster.local:8333/ \
      --aws-sigv4 "aws:amz:us-east-1:s3" \
      --user "seaweedfs-s3-access-key:seaweedfs-s3-secret-access-key" \
      -H "X-Amz-Target: S3Tables.CreateTableBucket" \
      -H "Content-Type: application/x-amz-json-1.1" \
      -d '{"name": "warehouse"}'
    ```

1. Confirm the created S3 Tables Bucket (catalog).

    ```shell
    curl -X POST http://seaweedfs-all-in-one.sample-c.svc.cluster.local:8333/ \
      --aws-sigv4 "aws:amz:us-east-1:s3" \
      --user "seaweedfs-s3-access-key:seaweedfs-s3-secret-access-key" \
      -H "X-Amz-Target: S3Tables.GetTableBucket" \
      -H "Content-Type: application/x-amz-json-1.1" \
      -d '{"tableBucketARN": "arn:aws:s3tables:us-east-1:admin:bucket/warehouse"}'
    ```

1. Exit the SeaweedFS client pod.

    ```shell
    exit
    ```

### Load TPC-H data by using DuckDB

1. Run the DuckDB CLI.

    ```shell
    kubectl exec -it duckdb -n sample-c -- duckdb
    ```

1. Install and load DuckDB extensions.

    ```sql
    INSTALL tpch;
    LOAD tpch;
    INSTALL httpfs;
    LOAD httpfs;
    INSTALL iceberg;
    LOAD iceberg;
    ```

1. Create a secret to access the S3 Bucket (data files) in SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_s3_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-c.svc.cluster.local:8333',
        KEY_ID 'seaweedfs-s3-access-key',
        SECRET 'seaweedfs-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://warehouse/'
    );
    ```

1. Create a secret to access the S3 Tables Bucket (catalog) in SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_catalog_secret (
        TYPE iceberg,
        ENDPOINT 'http://seaweedfs-iceberg.sample-c.svc.cluster.local:8181/',
        CLIENT_ID 'seaweedfs-s3-access-key',
        CLIENT_SECRET 'seaweedfs-s3-secret-access-key',
        OAUTH2_SERVER_URI 'http://seaweedfs-iceberg.sample-c.svc.cluster.local:8181/v1/oauth/tokens'
    );
    ```

1. Attach the sample catalog in SeaweedFS.

    ```sql
    ATTACH '' AS sample_catalog (
        TYPE iceberg,
        ENDPOINT 'http://seaweedfs-iceberg.sample-c.svc.cluster.local:8181/',
        SECRET 'seaweedfs_catalog_secret',
        ACCESS_DELEGATION_MODE 'none'
    );
    ```

1. Create a schema for TPC-H.

    ```sql
    CREATE SCHEMA sample_catalog.tpc_h;
    ```

1. Generate TPC-H data in memory.

    ```sql
    CALL dbgen(sf=0.1);
    ```

    > Note: This command generates TPC-H data in memory, and a large `sf` value might exhaust memory. Be careful when you generate the TPC-H data. For reference, approximate total size of Parquet files stored in SeaweedFS (not including metadata files) is as follows (these values were measured in a test environment, and may vary in your environment):
    > - sf=0.01 : 2MB
    > - sf=0.1 : 22MB
    > - sf=1 : 240MB

1. Write TPC-H data as Iceberg Tables from memory to SeaweedFS and register the table metadata in the catalog in SeaweedFS.

    ```sql
    CREATE TABLE sample_catalog.tpc_h.orders AS
        SELECT * FROM orders;

    CREATE TABLE sample_catalog.tpc_h.lineitem AS
        SELECT * FROM lineitem;

    CREATE TABLE sample_catalog.tpc_h.customer AS
        SELECT * FROM customer;

    CREATE TABLE sample_catalog.tpc_h.supplier AS
        SELECT * FROM supplier;

    CREATE TABLE sample_catalog.tpc_h.part AS
        SELECT * FROM part;

    CREATE TABLE sample_catalog.tpc_h.partsupp AS
        SELECT * FROM partsupp;

    CREATE TABLE sample_catalog.tpc_h.nation AS
        SELECT * FROM nation;

    CREATE TABLE sample_catalog.tpc_h.region AS
        SELECT * FROM region;
    ```

1. Exit DuckDB CLI.

    ```sql
    .exit
    ```

## Run TPC-H queries by using DuckDB

1. Run the DuckDB CLI.

    ```shell
    kubectl exec -it duckdb -n sample-c -- duckdb
    ```

1. Install and load DuckDB extensions.

    ```sql
    INSTALL tpch;
    LOAD tpch;
    INSTALL httpfs;
    LOAD httpfs;
    INSTALL iceberg;
    LOAD iceberg;
    ```

1. Create a secret to access the S3 Bucket (data files) in SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_s3_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-c.svc.cluster.local:8333',
        KEY_ID 'seaweedfs-s3-access-key',
        SECRET 'seaweedfs-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://warehouse/'
    );
    ```

1. Create a secret to access the S3 Tables Bucket (catalog) in SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_catalog_secret (
        TYPE iceberg,
        ENDPOINT 'http://seaweedfs-iceberg.sample-c.svc.cluster.local:8181/',
        CLIENT_ID 'seaweedfs-s3-access-key',
        CLIENT_SECRET 'seaweedfs-s3-secret-access-key',
        OAUTH2_SERVER_URI 'http://seaweedfs-iceberg.sample-c.svc.cluster.local:8181/v1/oauth/tokens'
    );
    ```

1. Attach the sample catalog in SeaweedFS.

    ```sql
    ATTACH '' AS sample_catalog (
        TYPE iceberg,
        ENDPOINT 'http://seaweedfs-iceberg.sample-c.svc.cluster.local:8181/',
        SECRET 'seaweedfs_catalog_secret',
        ACCESS_DELEGATION_MODE 'none'
    );
    ```

1. Set the default schema to use the TPC-H tables stored in SeaweedFS.

    ```sql
    USE sample_catalog.tpc_h;
    ```

1. Check if you can see the Iceberg Tables.

    ```sql
    SHOW TABLES;
    ```

1. Run a TPC-H query, for example, query 21.

    ```sql
    SELECT
        s_name,
        count(*) AS numwait
    FROM
        supplier,
        lineitem l1,
        orders,
        nation
    WHERE
        s_suppkey = l1.l_suppkey
        AND o_orderkey = l1.l_orderkey
        AND o_orderstatus = 'F'
        AND l1.l_receiptdate > l1.l_commitdate
        AND EXISTS (
            SELECT
                *
            FROM
                lineitem l2
            WHERE
                l2.l_orderkey = l1.l_orderkey
                AND l2.l_suppkey <> l1.l_suppkey)
        AND NOT EXISTS (
            SELECT
                *
            FROM
                lineitem l3
            WHERE
                l3.l_orderkey = l1.l_orderkey
                AND l3.l_suppkey <> l1.l_suppkey
                AND l3.l_receiptdate > l3.l_commitdate)
        AND s_nationkey = n_nationkey
        AND n_name = 'SAUDI ARABIA'
    GROUP BY
        s_name
    ORDER BY
        numwait DESC,
        s_name
    LIMIT 100;
    ```

1. [Optional] You can list all TPC-H queries using the `tpch_queries()` function.

    ```sql
    FROM tpch_queries();
    ```

    - https://duckdb.org/docs/current/core_extensions/tpch#listing-queries

1. Exit DuckDB CLI.

    ```sql
    .exit
    ```

## [Appendix] Deploy Spark SQL and Run TPC-H Queries

You can also run queries using Spark, which supports Apache Iceberg.

### Deploy Spark SQL

1. Deploy the ConfigMap and Pod for Spark SQL.

    ```shell
    kubectl apply -f ./spark-sql/spark-sql.yaml -n sample-c
    ```

### Run TPC-H queries by using Spark SQL

1. Start the Spark SQL CLI in the `spark-sql` pod.

    ```shell
    kubectl exec -it spark-sql -n sample-c -- /opt/spark/bin/spark-sql
    ```

1. Set the default catalog and schema to use the TPC-H tables stored in SeaweedFS.

    ```sql
    USE sample_catalog.tpc_h;
    ```

1. Check if you can see the Iceberg Tables.

    ```sql
    SHOW TABLES;
    ```

1. Run a TPC-H query, for example, query 21.

    ```sql
    SELECT
        s_name,
        count(*) AS numwait
    FROM
        supplier,
        lineitem l1,
        orders,
        nation
    WHERE
        s_suppkey = l1.l_suppkey
        AND o_orderkey = l1.l_orderkey
        AND o_orderstatus = 'F'
        AND l1.l_receiptdate > l1.l_commitdate
        AND EXISTS (
            SELECT
                *
            FROM
                lineitem l2
            WHERE
                l2.l_orderkey = l1.l_orderkey
                AND l2.l_suppkey <> l1.l_suppkey)
        AND NOT EXISTS (
            SELECT
                *
            FROM
                lineitem l3
            WHERE
                l3.l_orderkey = l1.l_orderkey
                AND l3.l_suppkey <> l1.l_suppkey
                AND l3.l_receiptdate > l3.l_commitdate)
        AND s_nationkey = n_nationkey
        AND n_name = 'SAUDI ARABIA'
    GROUP BY
        s_name
    ORDER BY
        numwait DESC,
        s_name
    LIMIT 100;
    ```

1. Exit the Spark SQL CLI.

    ```sql
    quit;
    ```

## Delete the sample lakehouse

```shell
kubectl delete ns sample-c
```
```shell
kubectl delete clusterrole seaweedfs-rw-cr
```
```shell
kubectl delete clusterrolebinding seaweedfs-rw-crb
```
