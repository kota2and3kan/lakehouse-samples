# Demo (2026/07/15 OCHaCafe)

- https://ochacafe.connpass.com/event/393529/

## Overview

```
+--[Kubernetes]---------------------------------------------------------------+
|                                                                             |
|  +----------------+                                     +----------------+  |
|  | DuckDB         |---------(Iceberg REST API)--------->| Lakekeeper     |  |
|  | (Query Engine) |<--------(Get Catalog Info)----------| (REST Catalog) |  |
|  +-------+--------+                                     +----------------+  |
|          |                                                                  |
|  (Read/Write Data and Metadata)                                             |
|          |                                                                  |
|          +--------+------------------+------------------+                   |
|                   |                  |                  |                   |
|  +----------------|------------------|------------------|----------------+  |
|  |  +-------------|------------------|------------------|-------------+  |  |
|  |  |  +----------|------------------|------------------|----------+  |  |  |
|  |  |  |          V                  V                  V          |  |  |  |
|  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
|  |  |  |  | Iceberg Table |  | Iceberg Table |  | Iceberg Table |  |  |  |  |
|  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
|  |  |  |  Folder (tpc_h)                                           |  |  |  |
|  |  |  +-----------------------------------------------------------+  |  |  |
|  |  |  Bucket (sample-bucket)                                         |  |  |
|  |  +-----------------------------------------------------------------+  |  |
|  |  SeaweedFS                                                            |  |
|  |  (Object Storage)                                                     |  |
|  +-----------------------------------------------------------------------+  |
|                                                                             |
+-----------------------------------------------------------------------------+
```

## Deploy each component on Kubernetes

### Create a Kubernetes namespace

1. Create a Kubernetes namespace.

    ```shell
    kubectl create ns sample-b
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

### Deploy SeaweedFS (Object Storage)

1. Create a secret that includes SeaweedFS user information.

    ```shell
    kubectl create secret generic seaweedfs-s3-user --from-file=seaweedfs_s3_config=./seaweedfs/seaweedfs-s3-user.json -n sample-b
    ```

1. Deploy SeaweedFS.

    ```shell
    helm install seaweedfs seaweedfs/seaweedfs -f ./seaweedfs/seaweedfs.yaml -n sample-b --version 4.33.0
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

### Deploy Lakekeeper (REST Catalog)

1. Deploy Lakekeeper.

    ```shell
    helm install lakekeeper lakekeeper/lakekeeper -f ./lakekeeper/lakekeeper.yaml -n sample-b --version 0.11.0
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |                                                         +----------------+  |
    |                                                         | Lakekeeper     |  |
    |                                                         | (REST Catalog) |  |
    |                                                         +----------------+  |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```


1. Deploy a client pod for Lakekeeper.

    ```shell
    kubectl apply -f ./lakekeeper/lakekeeper-client.yaml -n sample-b
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |                                                         +----------------+  |
    |                                                         | Lakekeeper     |  |
    |                                                         | (REST Catalog) |  |
    |                                                         +----------------+  |
    |                              +----------------+                             |
    |                              | curl           |                             |
    |                              | (Client)       |                             |
    |                              +----------------+                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

### Deploy DuckDB CLI (Query Engine)

1. Deploy DuckDB CLI as a pod.

    ```shell
    kubectl apply -f ./duckdb/duckdb.yaml -n sample-b
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | DuckDB         |                                     | Lakekeeper     |  |
    |  | (Query Engine) |                                     | (REST Catalog) |  |
    |  +----------------+                                     +----------------+  |
    |                              +----------------+                             |
    |                              | curl           |                             |
    |                              | (Client)       |                             |
    |                              +----------------+                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |                                                                       |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

## Access Web UI

1. Access the SeaweedFS Web UI (Admin Console) at `127.0.0.1` using the `kubectl port-forward` command.

    ```shell
    kubectl port-forward svc/seaweedfs-admin 23646:23646 -n sample-b
    ```

1. Access the Lakekeeper Web UI (Admin Console) at `127.0.0.1` using the `kubectl port-forward` command.

    ```shell
    kubectl port-forward svc/lakekeeper 8181:8181 -n sample-b
    ```

## Create Iceberg Tables (TPC-H)

### Create a bucket in SeaweedFS

1. Create a bucket.

    ```shell
    kubectl exec -it $(kubectl get pod -l app.kubernetes.io/component=seaweedfs-all-in-one -o name -n sample-b) -n sample-b --  \
      sh -c 'echo "s3.bucket.create -name sample-bucket -owner query-engine-s3-user" | weed shell -master=localhost:9333'
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | DuckDB         |                                     | Lakekeeper     |  |
    |  | (Query Engine) |                                     | (REST Catalog) |  |
    |  +----------------+                                     +----------------+  |
    |                              +----------------+                             |
    |                              | curl           |                             |
    |                              | (Client)       |                             |
    |                              +----------------+                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```


### Create a catalog in Lakekeeper

1. Run a shell in the Lakekeeper client pod.

    ```shell
    kubectl exec -it lakekeeper-client -n sample-b -- /bin/sh
    ```

1. Bootstrap Lakekeeper.

    ```shell
    curl -X POST http://lakekeeper.sample-b.svc.cluster.local:8181/management/v1/bootstrap \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer dummy" \
      -d '{"accept-terms-of-use": true}'
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | DuckDB         |   +------(Bootstrap)--------------->| Lakekeeper     |  |
    |  | (Query Engine) |   |                                 | (REST Catalog) |  |
    |  +----------------+   |                                 +----------------+  |
    |                       |      +----------------+                             |
    |                       +------| curl           |                             |
    |                              | (Client)       |                             |
    |                              +----------------+                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Create a catalog.

    ```shell
    curl -X POST http://lakekeeper.sample-b.svc.cluster.local:8181/management/v1/warehouse \
      -H "Content-Type: application/json" \
      -H "Authorization: Bearer dummy" \
      -d '{
        "warehouse-name": "sample_catalog",
        "storage-credential": {
          "type": "s3",
          "credential-type": "access-key",
          "aws-access-key-id": "lakekeeper-s3-access-key",
          "aws-secret-access-key": "lakekeeper-s3-secret-access-key"
        },
        "storage-profile": {
          "type": "s3",
          "bucket": "sample-bucket",
          "region": "us-east-1",
          "flavor": "s3-compat",
          "endpoint": "http://seaweedfs-all-in-one.sample-b.svc.cluster.local:8333",
          "path-style-access": true,
          "sts-enabled": false,
          "remote-signing-enabled": false
        },
        "delete-profile": {
          "type": "hard"
        }
      }'
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | DuckDB         |   +------(Create a catalog)-------->| Lakekeeper     |  |
    |  | (Query Engine) |   |                                 | (REST Catalog) |  |
    |  +----------------+   |                                 +----------------+  |
    |                       |      +----------------+                             |
    |                       +------| curl           |                             |
    |                              | (Client)       |                             |
    |                              +----------------+                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Exit the Lakekeeper client pod.

    ```shell
    exit
    ```

### Load TPC-H data by using DuckDB

1. Run the DuckDB CLI.

    ```shell
    kubectl exec -it duckdb -n sample-b -- duckdb
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

    ```
                                        +--------------------------------+
                                        |  http://extensions.duckdb.org  |
                                        +--------------------------------+
                                                     |
                                                   (WAN)
                                                     |
    +--[Kubernetes]----------------------------------+----------------------------+
    |                                                |                            |
    |  +----------------+                            |        +----------------+  |
    |  | DuckDB         |<---(Download extensions)---+        | Lakekeeper     |  |
    |  | (Query Engine) |                                     | (REST Catalog) |  |
    |  +----------------+                                     +----------------+  |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Create a secret to access SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-b.svc.cluster.local:8333',
        KEY_ID 'query-engine-s3-access-key',
        SECRET 'query-engine-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://sample-bucket/'
    );
    ```

1. Attach the sample catalog in Lakekeeper.

    ```sql
    ATTACH 'sample_catalog' AS sample_catalog (
        TYPE iceberg,
        ENDPOINT 'http://lakekeeper.sample-b.svc.cluster.local:8181/catalog',
        AUTHORIZATION_TYPE 'none',
        ACCESS_DELEGATION_MODE 'none'
    );
    ```

1. Create a schema for TPC-H.

    ```sql
    CREATE SCHEMA sample_catalog.tpc_h;
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | DuckDB         |-----(Register schema metadata)----->| Lakekeeper     |  |
    |  | (Query Engine) |                                     | (REST Catalog) |  |
    |  +----------------+                                     +----------------+  |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Generate TPC-H data in memory.

    ```sql
    CALL dbgen(sf=0.1);
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +-----------------------------------+                  +----------------+  |
    |  | DuckDB                            |                  | Lakekeeper     |  |
    |  |  +-------+  +-------+  +-------+  |                  | (REST Catalog) |  |
    |  |  | Table |  | Table |  | Table |  |                  +----------------+  |
    |  |  +-------+  +-------+  +-------+  |                                      |
    |  +-----------------------------------+                                      |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |                                                                 |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Write TPC-H data as Iceberg Tables from memory to SeaweedFS and register the table metadata in the catalog in Lakekeeper.

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

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +-----------------------------------+                  +----------------+  |
    |  | DuckDB                            |     Register     | Lakekeeper     |  |
    |  |  +-------+  +-------+  +-------+  |---   Table   --->| (REST Catalog) |  |
    |  |  | Table |  | Table |  | Table |  |     Metadata     +----------------+  |
    |  |  +---+---+  +---+---+  +---+---+  |                                      |
    |  +------|----------|----------|------+                                      |
    |         |          |          |                                             |
    |         +---------++----------+------+---(Write Tables)-+                   |
    |                   |                  |                  |                   |
    |  +----------------|------------------|------------------|----------------+  |
    |  |  +-------------|------------------|------------------|-------------+  |  |
    |  |  |  +----------|------------------|------------------|----------+  |  |  |
    |  |  |  |          V                  V                  V          |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  | Iceberg Table |  | Iceberg Table |  | Iceberg Table |  |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  Folder (tpc_h)                                           |  |  |  |
    |  |  |  +-----------------------------------------------------------+  |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Exit DuckDB CLI.

    ```sql
    .exit
    ```

## Run TPC-H queries by using DuckDB

1. Run the DuckDB CLI.

    ```shell
    kubectl exec -it duckdb -n sample-b -- duckdb
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
        ENDPOINT 'seaweedfs-all-in-one.sample-b.svc.cluster.local:8333',
        KEY_ID 'query-engine-s3-access-key',
        SECRET 'query-engine-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://sample-bucket/'
    );
    ```

1. Attach the sample catalog in Lakekeeper.

    ```sql
    ATTACH 'sample_catalog' AS sample_catalog (
        TYPE iceberg,
        ENDPOINT 'http://lakekeeper.sample-b.svc.cluster.local:8181/catalog',
        AUTHORIZATION_TYPE 'none',
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

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | DuckDB         |---------(Iceberg REST API)--------->| Lakekeeper     |  |
    |  | (Query Engine) |<--------(Get Catalog Info)----------| (REST Catalog) |  |
    |  +-------+--------+                                     +----------------+  |
    |          |                                                                  |
    |    (Read Tables)                                                            |
    |          |                                                                  |
    |          +--------+------------------+------------------+                   |
    |                   |                  |                  |                   |
    |  +----------------|------------------+------------------+----------------+  |
    |  |  +-------------|------------------+------------------+-------------+  |  |
    |  |  |  +----------|------------------+------------------+----------+  |  |  |
    |  |  |  |          v                  v                  v          |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  | Iceberg Table |  | Iceberg Table |  | Iceberg Table |  |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  Folder (tpc_h)                                           |  |  |  |
    |  |  |  +-----------------------------------------------------------+  |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
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

1. Exit DuckDB CLI.

    ```sql
    .exit
    ```

## [Appendix] Deploy Spark SQL and Run TPC-H Queries

### Deploy Spark SQL

1. Deploy the ConfigMap and Pod for Spark SQL.

    ```shell
    kubectl apply -f ./spark-sql/spark-sql.yaml -n sample-b
    ```

### Run TPC-H queries by using Spark SQL

1. Start the Spark SQL CLI in the `spark-sql` pod.

    ```shell
    kubectl exec -it spark-sql -n sample-b -- /opt/spark/bin/spark-sql
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | Spark SQL      |                                     | Lakekeeper     |  |
    |  | (Query Engine) |                                     | (REST Catalog) |  |
    |  +----------------+                                     +----------------+  |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |                                                                             |
    |  +-----------------------------------------------------------------------+  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  |  +-----------------------------------------------------------+  |  |  |
    |  |  |  |                                                           |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  | Iceberg Table |  | Iceberg Table |  | Iceberg Table |  |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  Folder (tpc_h)                                           |  |  |  |
    |  |  |  +-----------------------------------------------------------+  |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
    ```

1. Set the default catalog and schema to use the TPC-H tables stored in SeaweedFS.

    ```sql
    USE sample_catalog.tpc_h;
    ```

1. Check if you can see the Iceberg Tables.

    ```sql
    SHOW TABLES;
    ```

    ```
    +--[Kubernetes]---------------------------------------------------------------+
    |                                                                             |
    |  +----------------+                                     +----------------+  |
    |  | Spark SQL      |---------(Iceberg REST API)--------->| Lakekeeper     |  |
    |  | (Query Engine) |<--------(Get Catalog Info)----------| (REST Catalog) |  |
    |  +-------+--------+                                     +----------------+  |
    |          |                                                                  |
    |    (Read Tables)                                                            |
    |          |                                                                  |
    |          +--------+------------------+------------------+                   |
    |                   |                  |                  |                   |
    |  +----------------|------------------+------------------+----------------+  |
    |  |  +-------------|------------------+------------------+-------------+  |  |
    |  |  |  +----------|------------------+------------------+----------+  |  |  |
    |  |  |  |          v                  v                  v          |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  | Iceberg Table |  | Iceberg Table |  | Iceberg Table |  |  |  |  |
    |  |  |  |  +---------------+  +---------------+  +---------------+  |  |  |  |
    |  |  |  |  Folder (tpc_h)                                           |  |  |  |
    |  |  |  +-----------------------------------------------------------+  |  |  |
    |  |  |  Bucket (sample-bucket)                                         |  |  |
    |  |  +-----------------------------------------------------------------+  |  |
    |  |  SeaweedFS                                                            |  |
    |  |  (Object Storage)                                                     |  |
    |  +-----------------------------------------------------------------------+  |
    |                                                                             |
    +-----------------------------------------------------------------------------+
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
