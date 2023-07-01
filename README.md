# Quickstart on Kubernetes

As part of this demo we will see how to get started with [atlas](https://atlasgo.io/) -- a modern tool for managing database schemas -- on Kubernetes.

## Pre-Requisites

- [Docker for Desktop](https://www.docker.com/products/docker-desktop/)
- [k3d](https://k3d.io), to create local Kubernetes Cluster
- [Atlas CLI](https://atlasgo.io)
- [Helm CLI](https://helm.sh)
- [Postgres CLI](https://www.pgadmin.org/)
- [gettext](https://www.gnu.org/software/gettext/)
- [direnv](https://direnv.net)

### Install Tools

```shell
brew install ariga/tap/atlas helm postgresql k3d gettext direnv
```

## Create Kubernetes Cluster

You can use any Kubernetes cluster of your choice, but for simplicity and ease of use let us k3d,

```shell
k3d cluster create atlas-demo
```

## Deploy Database

As part of this demo we will be using PostgreSQL as our database. Let us deploy postgresql on to our Kubernetes cluster,

```shell
helm upgrade \
  --install \
  --values pg-values.yaml \
  --wait postgresql oci://registry-1.docker.io/bitnamicharts/postgresql
```

Do a port forward to the postgresql service,

```shell
kubectl port-forward --namespace default svc/postgresql 5432:5432
```

Open a new terminal and let us verify our connection to the database,

```shell
psql -h 127.0.0.1 -p 5432 -U demo demodb -c 'select now() as today;'
```

> NOTE:
> Default password is `demoPassword123`

Should display a timestamp like,

```shell
             today             
-------------------------------
 2023-07-01 04:44:21.212152+00
(1 row)
```

> *TIP*:
> You can export the following variables to skip specifiying them on CLI:
>
> ```shell
> export PGHOST=127.0.0.1
> export PGPORT=5432
> export PGUSER=demo
> export PGDATABASE=demodb
> export PGPASSWORD=demoPassword123
> ```

## Managing Database using atlas

Let us inspect our database schema,

```shell
atlas schema inspect -u "postgres://127.0.0.1:5432/demodb?search_path=public&sslmode=disable"
```

> NOTE: As we are using DB from local cluster with SSL disabled we pass the parameter `sslmode=disable`. This is not recommended for production
>

Should show the following:

```hcl
schema "public" {
}
```

As you notice the schema is empty as we don't have any tables yet on `demodb`. As part of the upcoming sections of the demo let deploy the Atlas Kubernetes operator and manage our `demodb` database using the Kubernetes CRDs.

### Deploy Kubernetes Operator

Install `atlas` Kubernetes operator,

```shell
helm upgrade \
  --install \
  --wait atlas-operator oci://ghcr.io/ariga/charts/atlas-operator
```

Create a [Kubernetes Secret](./pg-credentials.yaml) to allow `atlas` connect to the database `demodb` and perform required management,

```shell
envsubst < pg-credentials.yaml |  kubectl apply -f -
```

### Create Schema and Tables

Let us create the desired schema and its tables using the Atlas CRD.

The following YAML shows `AtlasSchema` CRD for our demo. As part of this demo we will be creating a table called `fruits` that will hold the information about fruits,

```shell
kubectl apply -f demodb-schema.yaml
```

Let us wait for the database to be ready and reconciled,

```shell
kubectl wait --for=condition=Ready atlasschema/demodb --timeout=60s
```

Now let connect to the database and verify our changes i.e. adding a new table named `fruits`,

```shell
psql -c '\d+ public.fruits'
```

The command should show an output like,

```text
Column |          Type          | Collation | Nullable |              Default               | Storage  | Compression | Stats target | Description 
--------+------------------------+-----------+----------+------------------------------------+----------+-------------+--------------+-------------
 id     | integer                |           | not null | nextval('fruits_id_seq'::regclass) | plain    |             |              | 
 name   | character varying(255) |           | not null |                                    | extended |             |              | 
 season | character varying(255) |           | not null |                                    | extended |             |              | 
Indexes:
    "fruits_pkey" PRIMARY KEY, btree (id)
Access method: heap
```

### Schema Update

Update the table `fruits` with extra column called `count` that will hold the number of fruits,

```shell
diff demodb-schema.yaml demodb-schema-count.yaml
```

```diff
18c18,19
<         season varchar(255) not null
---
>         season varchar(255) not null,
>         count int
```

Now apply the new schema CRD,

```shell
  kubectl apply -f demodb-schema-count.yaml
```

Let us wait for the database to be ready and reconciled,

```shell
kubectl wait --for=condition=Ready atlasschema/demodb --timeout=60s
```

Now describe the table again,

```shell
psql -c '\d+ public.fruits'
```

The command should show an output like,

```text
                                                               Table "public.fruits"
 Column |          Type          | Collation | Nullable |              Default               | Storage  | Compression | Stats target | Description
--------+------------------------+-----------+----------+------------------------------------+----------+-------------+--------------+-------------
 id     | integer                |           | not null | nextval('fruits_id_seq'::regclass) | plain    |             |              |
 name   | character varying(255) |           | not null |                                    | extended |             |              |
 season | character varying(255) |           | not null |                                    | extended |             |              |
 count  | integer                |           |          |                                    | plain    |             |              |
Indexes:
    "fruits_pkey" PRIMARY KEY, btree (id)
Access method: heap
```

## Cleanup

```shell
k3d cluster delete atlas-demo
```

## Resources

- [Atlas Docs](https://atlasgo.io/getting-started)
- [PostgreSQL](https://www.postgresql.org/docs/12/index.html)
- [PostgreSQL Helm Chart](https://artifacthub.io/packages/helm/bitnami/postgresql)