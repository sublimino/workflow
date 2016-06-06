# Configuring Object Storage

A variety of Deis Workflow components rely on an object storage system to do their work including storing application slugs, Docker images and database logs.

Deis Workflow ships with [Minio][minio] by default, which provides in-cluster, ephemeral object storage. This means that _if the Minio server crashes, all data will be lost_. Therefore, **Minio should be used for development or testing only**.

## Configuring off-cluster Object Storage

Every component that relies on object storage uses two inputs for configuration:

1. Component-specific environment variables (e.g. `BUILDER_STORAGE` and `REGISTRY_STORAGE`)
2. Access credentials stored as a Kubernetes secret named `objectstorage-keyfile`

The helm classic chart for Deis Workflow can be easily configured to connect Workflow components to off-cluster object storage. Deis Workflow currently supports Google Compute Storage, Amazon S3 and Azure Blob Storage.

* **Step 1:** Create storage buckets for each of the Workflow subsystems: builder, registry and database
    * Note: Depending on your chosen object storage you may need to provide globally unique bucket names.
    * Note: If you provide credentials with sufficient access to the underlying storage, Workflow components will create the buckets if they do not exist.
* **Step 2:** If applicable, generate credentials that have write access to the storage buckets created in Step 1
* **Step 3:** If you haven't already fetched the helm classic chart, do so with `helmc fetch deis/workflow-rc2`
* **Step 4:** Update storage details either by setting the appropriate environment variables or by modifying the template file `tpl/generate_params.toml`
    * **1.** Using environment variables:
        * Set `STORAGE_TYPE` to `s3`, `azure` or `gcs`, then set the following environment variables accordingly.
          * For `STORAGE_TYPE=gcs`:

            ```
            export GCS_KEY_JSON, GCS_REGISTRY_BUCKET, GCS_DATABASE_BUCKET, GCS_BUILDER_BUCKET
            ```

          * For `STORAGE_TYPE=s3`:

            ```
            export AWS_ACCESS_KEY, AWS_SECRET_KEY, AWS_REGISTRY_BUCKET, AWS_DATABASE_BUCKET, AWS_BUILDER_BUCKET, S3_REGION
            ```

          * For `STORAGE_TYPE=azure`:

            ```
            export AZURE_ACCOUNT_NAME, AZURE_ACCOUNT_KEY, AZURE_REGISTRY_CONTAINER, AZURE_DATABASE_CONTAINER, AZURE_BUILDER_CONTAINER
            ```

    * **2.** Using template file `tpl/generate_params.toml`:
        * Open the helm classic chart with `helmc edit workflow-rc2` and look for the template file `tpl/generate_params.toml`
        * Update the `storage` parameter to reference the storage platform you are using: `s3`, `azure`, `gcs`
        * Update the values in the section which corresponds to your storage type, including region, bucket names and access credentials
    * Note: you do not need to base64 encode any of these values as Helm Classic will handle encoding automatically
* **Step 5:** Save your changes and re-generate the helm classic chart by running `helmc generate -x manifests workflow-rc2`
* **Step 6:** Check the generated file in your manifests directory, you should see `deis-objectstorage-secret.yaml`

You are now ready to `helmc install workflow-rc2` using your desired object storage.

## Object Storage Configuration and Credentials

During the `helmc generate` step, Helm Classic creates a Kubernetes secret in the Deis namespace named `objectstorage-keyfile`. The exact structure of the file depends on storage backend specified in `tpl/generate_params.toml`.

```
# Set the storage backend
#
# Valid values are:
# - s3: Store persistent data in AWS S3 (configure in S3 section)
# - azure: Store persistent data in Azure's object storage
# - gcs: Store persistent data in Google Cloud Storage
# - minio: Store persistent data on in-cluster Minio server
storage = "minio"
```

Individual components map the master credential secret to either secret-backed environment variables or volumes. See below for the component-by-component locations.

## Component Details

### [deis/builder](https://github.com/deis/builder)

The builder looks for a `BUILDER_STORAGE` environment variable, which it then uses as a key to look up the object storage location and authentication information from the `objectstore-creds` volume.

### [deis/slugbuilder](https://github.com/deis/slugbuilder)

Slugbuilder is configured and launched by the builder component. Slugbuilder reads credential information from the standard `objectstorage-keyfile` secret.

If you are using slugbuilder as a standalone component the following configuration is important:

- `TAR_PATH` - The location of the application `.tar` archive, relative to the configured bucket for builder e.g. `home/burley-yeomanry:git-3865c987/tar`
- `PUT_PATH` - The location to upload the finished slug, relative to the configured bucket fof builder e.g. `home/burley-yeomanry:git-3865c987/push`

**Note: these environment variables are case-sensitive**

### [deis/slugrunner](https://github.com/deis/slugrunner)

Slugrunner is configured and launched by the controller inside a Workflow cluster. If you are using slugrunner as a standalone component the following configuration is important:

- `SLUG_URL` - environment variable containing the path of the slug, relative to the builder storage location, e.g. `home/burley-yeomanry:git-3865c987/push/slug.tgz`

Slugrunner reads credential information from a `objectstorage-keyfile` secret in the current Kubernetes namespace.

### [deis/dockerbuilder](https://github.com/deis/dockerbuilder)

Dockerbuilder is configured and launched by the builder component. Dockerbuilder reads credential information from the standard `objectstorage-keyfile` secret.

If you are using dockerbuilder as a standalone component the following configuration is important:

- `TAR_PATH` - The location of the application `.tar` archive, relative to the configured bucket for builder e.g. `home/burley-yeomanry:git-3865c987/tar`

### [deis/controller](https://github.com/deis/controller)

The controller is responsible for configuring the execution environment for buildpack-based applications. Controller copies `objectstorage-keyfile` into the application namespace so slugrunner can fetch the application slug.

The controller interacts through Kubernetes APIs and does not use any environment variables for object storage configuration.

### [deis/registry](https://github.com/deis/registry)

The registry looks for a `REGISTRY_STORAGE` environment variable which it then uses as a key to look up the object storage location and authentication information.

The registry reads credential information by reading `/var/run/secrets/deis/registry/creds/objectstorage-keyfile`.

This is the file location for the `objectstorage-keyfile` secret on the Pod filesystem.

### [deis/database](https://github.com/deis/postgres)

The database looks for a `DATABASE_STORAGE` environment variable, which it then uses as a key to look up the object storage location and authentication information

Minio (`DATABASE_STORAGE=minio`):

* `AWS_ACCESS_KEY_ID` via /var/run/secrets/deis/objectstore/creds/accesskey
* `AWS_SECRET_ACCESS_KEY` via /var/run/secrets/deis/objectstore/creds/secretkey
* `AWS_DEFAULT_REGION` is the Minio default of "us-east-1"
* `BUCKET_NAME` is the on-cluster default of "dbwal"

AWS (`DATABASE_STORAGE=s3`):

* `AWS_ACCESS_KEY_ID` via /var/run/secrets/deis/objectstore/creds/accesskey
* `AWS_SECRET_ACCESS_KEY` via /var/run/secrets/deis/objectstore/creds/secretkey
* `AWS_DEFAULT_REGION` via /var/run/secrets/deis/objectstore/creds/region
* `BUCKET_NAME` via /var/run/secrets/deis/objectstore/creds/database-bucket

GCS (`DATABASE_STORAGE=gcs`):

* `GS_APPLICATION_CREDS` via /var/run/secrets/deis/objectstore/creds/key.json
* `BUCKET_NAME` via /var/run/secrets/deis/objectstore/creds/database-bucket

Azure (`DATABASE_STORAGE=azure`):

* `WABS_ACCOUNT_NAME` via /var/run/secrets/deis/objectstore/creds/accountname
* `WABS_ACCESS_KEY` via /var/run/secrets/deis/objectstore/creds/accountkey
* `BUCKET_NAME` via /var/run/secrets/deis/objectstore/creds/database-container

[minio]: ../understanding-workflow/components.md#object-storage
[generate-params-toml]: https://github.com/deis/charts/blob/master/workflow-dev/tpl/generate_params.toml
