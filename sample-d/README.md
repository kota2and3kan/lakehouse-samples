# Sample D

Create a Lakehouse on Kubernetes with:

- DuckLake as a lakehouse format.
- SeaweedFS as object storage.
- PostgreSQL as a catalog database for DuckLake.
- DuckDB as a query engine.

## Overview

In this sample, you will create the following Lakehouse:

```
+--[Kubernetes]---------------------------------------------------+
|                                                                 |
|  +----------------+                         +----------------+  |
|  | DuckDB         |----------(SQL)--------->| PostgreSQL     |  |
|  | (Query Engine) |<---(Get Catalog Info)---| (Catalog DB)   |  |
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
|  |  | Table         |  | Table         |  | Table         |  |  |
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
    kubectl create ns sample-d
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
    kubectl create secret generic seaweedfs-s3-user --from-file=seaweedfs_s3_config=./seaweedfs/seaweedfs-s3-user.json -n sample-d
    ```

1. Deploy SeaweedFS.

    ```shell
    helm install seaweedfs seaweedfs/seaweedfs -f ./seaweedfs/seaweedfs.yaml -n sample-d --version 4.33.0
    ```

1. [Optional] You can access the Web UI (Admin Console) of SeaweedFS through `127.0.0.1` by using the `kubectl port-forward` command.

    ```shell
    kubectl port-forward svc/seaweedfs-admin 23646:23646 -n sample-d
    ```

    > Note: The values of `username` / `password` are `admin` / `admin`.

### Deploy PostgreSQL (Catalog Database)

1. Deploy PostgreSQL.

    ```shell
    kubectl apply -f ./postgresql/postgresql.yaml -n sample-d
    ```

### Deploy DuckDB CLI (Query Engine)

1. Deploy DuckDB CLI as a pod.

    ```shell
    kubectl apply -f ./duckdb/duckdb.yaml -n sample-d
    ```

## Create Tables (TPC-H)

### Load TPC-H data by using DuckDB

1. Run the DuckDB CLI.

    ```shell
    kubectl exec -it duckdb -n sample-d -- duckdb
    ```

1. Install and load DuckDB extensions.

    ```sql
    INSTALL ducklake;
    LOAD ducklake;
    INSTALL postgres;
    LOAD postgres;
    ```

1. Create secrets to access SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-d.svc.cluster.local:8333',
        KEY_ID 'seaweedfs-s3-access-key',
        SECRET 'seaweedfs-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://sample-bucket/'
    );
    ```

1. Create secrets to access PostgreSQL.

    ```sql
    CREATE SECRET postgresql_secret (
        TYPE postgres,
        HOST 'postgresql.sample-d.svc.cluster.local',
        PORT 5432,
        DATABASE 'ducklake',
        USER 'postgres',
        PASSWORD 'postgres'
    );
    ```

1. Attach the sample catalog in PostgreSQL.

    ```sql
    ATTACH 'ducklake:postgres:' AS sample_catalog (
        META_SECRET 'postgresql_secret',
        DATA_PATH 's3://sample-bucket/'
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

    > Note: This command generates TPC-H data in memory, and a large `sf` value might exhaust memory. Be careful when you generate the TPC-H data. For your information, the approximate total file size of Parquet stored in SeaweedFS (not including metadata files) would be as follows (these values were measured in a test environment, but the file sizes may vary in your environment):
    > - sf=0.01 : 2MB
    > - sf=0.1 : 22MB
    > - sf=1 : 240MB

1. Write TPC-H data as tables from memory to SeaweedFS and register table information to the catalog in PostgreSQL.

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
    kubectl exec -it duckdb -n sample-d -- duckdb
    ```

1. Install and load DuckDB extensions.

    ```sql
    INSTALL ducklake;
    LOAD ducklake;
    INSTALL postgres;
    LOAD postgres;
    ```

1. Create secrets to access SeaweedFS.

    ```sql
    CREATE SECRET seaweedfs_secret (
        TYPE s3,
        ENDPOINT 'seaweedfs-all-in-one.sample-d.svc.cluster.local:8333',
        KEY_ID 'seaweedfs-s3-access-key',
        SECRET 'seaweedfs-s3-secret-access-key',
        USE_SSL false,
        URL_STYLE 'path',
        SCOPE 's3://sample-bucket/'
    );
    ```

1. Create secrets to access PostgreSQL.

    ```sql
    CREATE SECRET postgresql_secret (
        TYPE postgres,
        HOST 'postgresql.sample-d.svc.cluster.local',
        PORT 5432,
        DATABASE 'ducklake',
        USER 'postgres',
        PASSWORD 'postgres'
    );
    ```

1. Attach the sample catalog in PostgreSQL.

    ```sql
    ATTACH 'ducklake:postgres:' AS sample_catalog (
        META_SECRET 'postgresql_secret'
    );
    ```

1. Set the default schema to use the TPC-H tables stored in SeaweedFS.

    ```sql
    USE sample_catalog.tpc_h;
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
