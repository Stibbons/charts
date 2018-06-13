# Apache Airflow / Celery

[Apache Airflow](https://airflow.apache.org/) is a platform to programmatically author, schedule and
monitor workflows.

This chart deploys a Celery Cluster to execute Airflow Jobs and distributes the load to several
machines.

Celery is a distributed task queue built in Python and heavily used by the Python community for
task-based workloads.

## Install Chart

To install the Airflow Chart into your Kubernetes cluster :

```bash
helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
helm install --namespace "airflow" --name "airflow" incubator/airflow
```

After installation succeeds, you can get a status of Chart

```bash
helm status "airflow"
```

If you want to delete your Chart, use this command:

```bash
helm delete  --purge "airflow"
```

### Helm ingresses

The Chart provides ingress configuration to allow customization the installation by adapting
the `values.yaml` depending on your setup.
Please read the comments in the `value.yaml` file for more detail on how to configure your reverse
proxy or load balancer.

### Chart Prefix

This Helm automatically prefixes all names using the release name to avoid collisions.

### Endpoints

This chart exposes 2 endpoints:

- WebServer, the main Airflow User Interface
- Flower, a debug UI for Celery Tasks

Both can be placed either at the root of a domain or at a sub path, for example:

```
http://mycompany.com/airflow/
http://mycompany.com/airflow/flower
```

NOTE: Mounting the Airflow UI and Flower under a subpath of a domain requires a version of Airflow
at least higher than 2.0.x. This will not work under 1.9.x.
For the moment (June 2018) this version has **not** been officially released by Apache Airflow.

You will have to use an image where airflow and flower has been updated to their latest version of
source code.
You can use the following image: `stibbons31/docker-airflow-dev:2.0dev`.
It is regularly rebased on top of the excellent `puckel/docker-airflow` image.

Please also note than Airflow UI and Flower do not behave the same:

- Airflow Web UI behave transparently, to configure it one just need to specify the
  `ingress.web.path` value.
- Flower cannot handle this scheme directly and requires to use an URL rewrite mechanism in front
  of it. In short, it is able to generate the right URLs in the returned HTML file but cannot
  respond to these URL. It is commonly found in software that wasn't intended to work under
  something else than a root URL or localhost port. To use it, see the `value.yaml` in detail on how
  to configure your ingress controller to rewrite the URL (or "strip" the prefix path).

  Note: unreleased Flower (as of June 2018) does not need the prefix strip feature anymore. It is
  integrated in `docker-airflow-dev:2.0dev` image.

### Airflow configuration

`airflow.cfg` configuration can be changed by defining environment variables in the following form:
`AIRFLOW__<SECTION_IN_UPPER_CASSE>__<KEY_IN_UPPER_CASE>`.

See the
[Airflow documentation for more information](http://airflow.readthedocs.io/en/latest/configuration.html?highlight=__CORE__#setting-configuration-options) about this feature.

This helm chart allows you to add these additional environment variable with the value key
`airflow.config`.
You can also add generic environment variables such as proxy or private pypi:

```yaml
airflow:
  config:
    AIRFLOW__CORE__LOGGING_LEVEL: DEBUG
    PIP_INDEX_URL: http://pypi.mycompany.com/
    PIP_TRUSTED_HOST: pypi.mycompany.com
    HTTP_PROXY: http://proxy.mycompany.com:1234
    HTTPS_PROXY: http://proxy.mycompany.com:1234
```

## DAGs Deployment

Several options are possible for synchronizing your Airflow DAGs.

### Mount a Shared Persistent Volume

You can store your DAG files on an external volume, and mount this volume into the relevant Pods
(scheduler, web, worker).
In this scenario, your CI/CD pipeline should update the DAG files in the PV.

Since all Pods should have the same collection of DAG files, it is recommended to create just one PV
that is shared. This ensures that the Pods are always in sync about the DagBag.

This is controlled by setting `persistance.enabled=true`. You will have to ensure yourself the
PVC are shared properly between your pods:

- If you are on AWS, you can use [Elastic File System (EFS)](https://aws.amazon.com/efs/).
- If you are on Azure, you can use
  [Azure File Storage (AFS)](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv).

To share a PV with multiple Pods, the PV needs to have accessMode 'ReadOnlyMany' or 'ReadWriteMany'.

### Use init-container

If you enable set `dags.init_container.enabled=true`, the pods will try upon startup to fetch the
git repository defined by `dags.git.url`, on ref `dags.git.ref` (could be a branch name, a tag or
a commit SHA) as the main DAG folder.

You can also add a `requirements.txt` file at the root of your DAG project to have other
Python dependencies installed when pods starts.

This is the easiest way of deploying your DAGs to Airflow, but initialization of each pod can take
a few minutes.

Example:

```yaml
dags:
  git:
     url: http://mygitserver.mycompany.com/project/name
     ref: master
  init_container:
    enabled: true
    install_requirements: true
```

### Embedded DAGs in the docker image

If you want more control on the way you deploy your DAGs, add DAG unit tests and ensure you deploy
exactly what you want, you embedded DAGs inside the single Docker container image, that will be
deployed as Scheduler, Webserver and Workers.

Be aware this requires more tooling than using shared PVC, or init-container:

- your CI/CD should be able to build a new docker image each time your DAGs are updated.
- your CI/CD should be able to control the deployment of this new image in your kubernetes cluster

Example of procedure:

- Fork the [puckel/docker-airflow](https://github.com/puckel/docker-airflow) repository
- Place your DAG inside the `dags` folder of the repository, and ensure your Python dependencies
  are well installed (for example `pip install -r requirements.txt` in your `Dockerfile`)
- Update the value of `airflow.image` in your `values.yaml`.

### Use your own PostgreSQL server

By default, the PostgreSQL chart is deployed and used as the result backend of the Celery Cluster.
Please refer to [this chart](https://github.com/kubernetes/charts/tree/master/stable/postgresql)
for additional PostgreSQL configuration options.

To use your own PostgreSQL database, you can set the `postgres.uri` value.

### Why Statefulset for Worker?

Celery workers uses StatefulSet.
It is used to freeze their DNS using a Kubernetes Headless Service, and allow the webserver to
requests the logs from each workers individually only by using their pod name.

This requires to expose a port (8793) and ensure the pod DNS is accessible to the web server pod,
which is why StatefulSet is for.

## Helm chart Configuration

The following table lists the configurable parameters of the Airflow chart and their default values.

| Parameter                                  | Description                                             | Default                                                    |
| ------------------------------------------ | ------------------------------------------------------- | ---------------------------------------------------------- |
| `airflow.fernet_key`                       | Ferney key (see `values.yaml` for example)              | (auto generated)                                           |
| `airflow.service.type`                     | services type                                           | `ClusterIP`                                                |
| `airflow.init_retry_loop`                  | max number of retries during container init             |                                                            |
| `airflow.image.repository`                 | Airflow docker image                                    | `puckel/docker-airflow`                                    |
| `airflow.image.tag`                        | Airflow docker tag                                      | `1.9.0-5`                                                  |
| `airflow.image.pull_policy`                | Image pull policy                                       | `IfNotPresent`                                             |
| `airflow.scheduler_num_runs`               | -1 to loop indefinitively, 1 to restart after each exec |                                                            |
| `airflow.scheduler_do_pickle`              | should the scheduler use pickles DAG code               | `true`                                                     |
| `airflow.web_replicas`                     | how many replicas for web server                        | `1`                                                        |
| `airflow.config`                           | custom airflow configuration env variables              | `{}`                                                       |
| `airflow.pod_disruption_budget`            | control pod disruption budget                           | `{'maxUnavailable': 1}`                                    |
| `workers.serviceAccountName`               | worker service account                                  | `default`                                                  |
| `workers.replicas`                         | number of workers pods to launch                        | `1`                                                        |
| `workers.resources`                        | custom resource configuration for worker pod            | `{}`                                                       |
| `workers.celery.instances`                 | number of parallel celery tasks per worker              | `1`                                                        |
| `ingres.enabled`                           | enable ingress                                          | `false`                                                    |
| `ingres.web.host`                          | hostname for the webserver ui                           | ""                                                         |
| `ingres.web.path`                          | path of the werbserver ui (read `values.yaml`)          | ``                                                         |
| `ingres.web.annotations`                   | annotations for the web ui ingress                      | `{}`                                                       |
| `ingres.flower.host`                       | hostname for the flower ui                              | ""                                                         |
| `ingres.flower.path`                       | path of the flower ui (read `values.yaml`)              | ``                                                         |
| `ingres.flower.liveness_path`              | path to the liveness probe (read `values.yaml`)         | `/`                                                        |
| `ingres.flower.annotations`                | annotations for the web ui ingress                      | `{}`                                                       |
| `persistance.enabled`                      | enable persistance storage for DAGs                     | `false`                                                    |
| `persistance.storageClass`                 | Persistent Volume Storage Class                         | (undefined)                                                |
| `persistance.accessMode`                   | PVC access mode                                         | `ReadWriteOnce`                                            |
| `persistance.size`                         | Persistant storage size request                         | `1Gi`                                                      |
| `dags.path`                                | mount path for persistent volume                        | `/usr/local/airflow/dags`                                  |
| `dags.init_container.enabled`              | Fetch the source code when the pods starts              | `false`                                                    |
| `dags.init_container.install_requirements` | auto install requirements.txt deps                      | `true`                                                     |
| `dags.git.url`                             | url to clone the git repository                         | nil                                                        |
| `dags.git.ref`                             | branch name, tag or sha1 to reset to                    | `master`                                                   |
| `postgres.enabled`                         | create a postgres server                                | `true`                                                     |
| `postgres.uri`                             | full URL to custom postgres setup                       | (undefined)                                                |
| `postgres.postgresUser`                    | PostgreSQL User                                         | `postgres`                                                 |
| `postgres.postgresPassword`                | PostgreSQL Password                                     | `airflow`                                                  |
| `postgres.postgresDatabase`                | PostgreSQL Database name                                | `airflow`                                                  |
| `postgres.persistence.enabled`             | Enable Postgres PVC                                     | `true`                                                     |
| `postgres.persistance.storageClass`        | Persistant class                                        | (undefined)                                                |
| `postgres.persistance.accessMode`          | Access mode                                             | `ReadWriteOnce`                                            |
| `redis.enabled`                            | Create a Redis cluster                                  | `true`                                                     |
| `redis.password`                           | Redis password                                          | `airflow`                                                  |
| `redis.master.persistence.enabled`         | Enable Redis PVC                                        | `false`                                                    |
| `redis.cluster.enabled`                    | enable master-slave cluster                             | `false`                                                    |

Full and up-to-date documentation can be found in the comments of the `values.yaml` file.

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install` or
`helm upgrade`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while
installing the chart. For example,

```bash
$ helm install --name airflow -f values.yaml incubator/airflow
```
