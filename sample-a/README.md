# Sample A

Create a Lakehouse on Kubernetes with:

- Apache Iceberg as a table format.
- SeaweedFS as object storage.
- Apache Polaris as a catalog for Apache Iceberg.
- DuckDB and Spark SQL as query engines.

## Overview

In this sample, you will create the following Lakehouse:

```
+--[Kubernetes]---------------------------------------------------+
|                                                                 |
|  +----------------+                         +----------------+  |
|  | DuckDB         |---(Iceberg REST API)--->| Apache Polaris |  |
|  | (Query Engine) |<---(Get Catalog Info)---| (REST Catalog) |  |
|  +-------+--------+                         +----------------+  |
|          |                                                      |
|  (Read/Write Data and Metadata)                                 |
|          |                                                      |
|          +--+------------------+------------------+             |
|             |                  |                  |             |
|  +----------+------------------+------------------+----------+  |
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
    kubectl create ns sample-a
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
    kubectl create secret generic seaweedfs-s3-user --from-file=seaweedfs_s3_config=./seaweedfs/seaweedfs-s3-user.json -n sample-a
    ```

1. Deploy SeaweedFS.

    ```shell
    helm install seaweedfs seaweedfs/seaweedfs -f ./seaweedfs/seaweedfs.yaml -n sample-a --version 4.33.0
    ```

1. [Optional] You can access the SeaweedFS Web UI (Admin Console) at `127.0.0.1` using the `kubectl port-forward` command.

    ```shell
    kubectl port-forward svc/seaweedfs-admin 23646:23646 -n sample-a
    ```

    > Note: The values of `username` / `password` are `admin` / `admin`.

### Deploy Apache Polaris (REST Catalog)

1. Add a Helm repository.

    ```shell
    helm repo add polaris https://downloads.apache.org/polaris/helm-chart
    ```

    ```shell
    helm repo update polaris
    ```

1. Create a secret that includes root user credentials for Polaris.

    ```shell
    kubectl create secret generic polaris-root-credentials \
      --from-literal=credentials="POLARIS,polaris-root-id,polaris-root-secret" \
      -n sample-a
    ```

1. Create a secret that includes credentials to access SeaweedFS.

    ```shell
    kubectl create secret generic polaris-s3-credentials \
      --from-literal=access_key_id=polaris-s3-access-key \
      --from-literal=secret_access_key=polaris-s3-secret-access-key \
      -n sample-a
    ```

1. Deploy Polaris.

    ```shell
    helm install polaris polaris/polaris -f ./polaris/polaris.yaml -n sample-a --version 1.5.0
    ```

1. Deploy a client pod for Polaris.

    ```shell
    kubectl apply -f ./polaris/polaris-client.yaml -n sample-a
    ```

### Deploy DuckDB CLI (Query Engine)

1. Deploy DuckDB CLI as a pod.

    ```shell
    kubectl apply -f ./duckdb/duckdb.yaml -n sample-a
    ```

## Create Iceberg Tables (TPC-H)

### Create a catalog in Polaris

1. Run a shell in the Polaris client pod.

    ```shell
    kubectl exec -it polaris-client -n sample-a -- /bin/sh
    ```

1. Get an access token for Polaris.

    ```shell
    POLARIS_ACCESS_TOKEN=$(curl -s http://polaris.sample-a.svc.cluster.local:8181/api/catalog/v1/oauth/tokens \
      -d 'grant_type=client_credentials' \
      -d "client_id=${CLIENT_ID}" \
      -d "client_secret=${CLIENT_SECRET}" \
      -d 'scope=PRINCIPAL_ROLE:ALL' \
      | grep -o '"access_token":"[^"]*"' \
      | sed 's/"access_token":"\(.*\)"/\1/')
    ```

1. Create a catalog.

    ```shell
    curl -X POST http://polaris.sample-a.svc.cluster.local:8181/api/management/v1/catalogs \
      -H "Authorization: Bearer ${POLARIS_ACCESS_TOKEN}" \
      -H "Content-Type: application/json" \
      -d '{
          "catalog": {
            "type": "INTERNAL",
            "name": "sample_catalog",
            "properties": {
              "default-base-location": "s3://sample-bucket/"
            },
            "storageConfigInfo": {
              "storageType": "S3",
              "allowedLocations": ["s3://sample-bucket/"],
              "endpoint": "http://seaweedfs-all-in-one.sample-a.svc.cluster.local:8333",
              "endpointInternal": "http://seaweedfs-all-in-one.sample-a.svc.cluster.local:8333",
              "pathStyleAccess": true,
              "stsUnavailable": true
            }
          }
        }'
    ```

1. Confirm the created catalog.

    ```shell
    curl -s http://polaris.sample-a.svc.cluster.local:8181/api/management/v1/catalogs \
      -H "Authorization: Bearer ${POLARIS_ACCESS_TOKEN}"
    ```

1. Exit the Polaris client pod.

    ```shell
    exit
    ```

### Create a bucket in SeaweedFS

1. Create a bucket.

    ```shell
    kubectl exec -it $(kubectl get pod -l app.kubernetes.io/component=seaweedfs-all-in-one -o name -n sample-a) -n sample-a --  \
      sh -c 'echo "s3.bucket.create -name sample-bucket -owner query-engine-s3-user" | weed shell -master=localhost:9333'
    ```

### Load TPC-H data by using DuckDB

1. Run the DuckDB CLI.

    ```shell
    kubectl exec -it duckdb -n sample-a -- duckdb
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

1. Create a secret to access SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-a.svc.cluster.local:8333',
        KEY_ID 'query-engine-s3-access-key',
        SECRET 'query-engine-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://sample-bucket/'
    );
    ```

1. Create a secret to access Polaris.

    ```sql
    CREATE SECRET polaris_secret (
        TYPE iceberg,
        CLIENT_ID 'polaris-root-id',
        CLIENT_SECRET 'polaris-root-secret',
        OAUTH2_SERVER_URI 'http://polaris.sample-a.svc.cluster.local:8181/api/catalog/v1/oauth/tokens'
    );
    ```

1. Attach the sample catalog in Polaris.

    ```sql
    ATTACH 'sample_catalog' AS sample_catalog (
        TYPE iceberg,
        ENDPOINT 'http://polaris.sample-a.svc.cluster.local:8181/api/catalog',
        SECRET 'polaris_secret',
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

1. Write TPC-H data as Iceberg Tables from memory to SeaweedFS and register the table metadata in the catalog in Polaris.

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
    kubectl exec -it duckdb -n sample-a -- duckdb
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

1. Create a secret to access SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-a.svc.cluster.local:8333',
        KEY_ID 'query-engine-s3-access-key',
        SECRET 'query-engine-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://sample-bucket/'
    );
    ```

1. Create a secret to access Polaris.

    ```sql
    CREATE SECRET polaris_secret (
        TYPE iceberg,
        CLIENT_ID 'polaris-root-id',
        CLIENT_SECRET 'polaris-root-secret',
        OAUTH2_SERVER_URI 'http://polaris.sample-a.svc.cluster.local:8181/api/catalog/v1/oauth/tokens'
    );
    ```

1. Attach the sample catalog in Polaris.

    ```sql
    ATTACH 'sample_catalog' AS sample_catalog (
        TYPE iceberg,
        ENDPOINT 'http://polaris.sample-a.svc.cluster.local:8181/api/catalog',
        SECRET 'polaris_secret',
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
    kubectl apply -f ./spark-sql/spark-sql.yaml -n sample-a
    ```

### Run TPC-H queries by using Spark SQL

1. Start the Spark SQL CLI in the `spark-sql` pod.

    ```shell
    kubectl exec -it spark-sql -n sample-a -- /opt/spark/bin/spark-sql
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
kubectl delete ns sample-a
```
```shell
kubectl delete clusterrole seaweedfs-rw-cr
```
```shell
kubectl delete clusterrolebinding seaweedfs-rw-crb
```
