# RethinkDB 2.3.5 Helm Chart

## Prerequisites Details
* Kubernetes 1.5+ with Beta APIs enabled.
* PV support on the underlying infrastructure.

## StatefulSet Details
* https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/

## StatefulSet Caveats
* https://kubernetes.io/docs/concepts/abstractions/controllers/statefulsets/#limitations

## Acknowledgment of Previous Works

I have heavely barrowed and extended code (peer discovery and probe) from the following project to build this Helm Chart and Docker image: https://github.com/rosskukulinski/kubernetes-rethinkdb-cluster.

## Chart Details

This chart implements a dynamically scalable [RethinkDB Cluster](https://www.rethinkdb.com/docs/sharding-and-replication/) using Kubernetes StatefulSets.

## Installing the Chart

To install the chart with the release name `my-release`:

```console
$ git clone git@github.com:meenie/helm-rethinkdb-cluster.git
$ helm install --name my-release ./helm-rethinkdb-cluster
```

## Configuration

The following table lists the configurable parameters of the rethinkdb chart and their default values.

| Parameter                               | Description                                                                        | Default                                             |
|-----------------------------------------|------------------------------------------------------------------------------------|-----------------------------------------------------|
| `image.name`                            | Custom RethinkDB image name for auto-joining and probe                             | `codylundquist/helm-rethinkdb-cluster`              |
| `image.tag`                             | Custom RethinkDB image tag                                                         | `0.1.0`                                             |
| `image.pullPolicy`                      | Custom RethinkDB image pull policy                                                 | `IfNotPresent`                                      |
| `cluster.replicas`                      | Number of RethinkDB Cluster replicas                                               | `3`                                                 |
| `cluster.resources`                     | Resource configuration for each RethinkDB Cluster Pod                              | `{}`                                                |
| `cluster.podAnnotations`                | Annotations to be added to RethinkDB Cluster Pods                                  | `{}`                                                |
| `cluster.serviceAnnotations`            | Annotations to be added to RethinkDB Cluster Service                               | `{}`                                                |
| `cluster.persistentVolume.enabled`      | If `true`, persistent volume claims are created                                    | `true`                                              |
| `cluster.persistentVolume.storageClass` | Persistent volume storage class                                                    | `volume.alpha.kubernetes.io/storage-class: default` |
| `cluster.persistentVolume.accessMode`   | Persistent volume access modes                                                     | `[ReadWriteOnce]`                                   |
| `cluster.persistentVolume.size`         | Persistent volume size                                                             | `10Gi`                                              |
| `cluster.persistentVolume.annotations`  | Persistent volume annotations                                                      | `{}`                                                |
| `cluster.rethinkConfig`                 | Configuration passed to RethinkDB via `--config-file=/config/rethinkdb.conf`       | `directory=/data bind=all no-http-admin`            |
| `proxy.replicas`                        | Number of RethinkDB Proxy replicas                                                 | `1`                                                 |
| `proxy.resources`                       | Resource configuration for each RethinkDB Proxy Pod                                | `{}`                                                |
| `proxy.podAnnotations`                  | Annotations to be added to RethinkDB Proxy Pods                                    | `{}`                                                |
| `proxy.serviceAnnotations`              | Annotations to be added to RethinkDB Cluster Service                               | `{}`                                                |
| `proxy.rethinkConfig`                   | Configuration passed to RethinkDB via `--config-file=/config/rethinkdb-proxy.conf` | `bind=all`                                          |
| `ports.cluster`                         | RethinkDB Cluster Port                                                             | `29015`                                             |
| `ports.driver`                          | RethinkDB Driver Port                                                              | `28015`                                             |
| `ports.admin`                           | RethinkDB Admin Port                                                               | `8080`                                              |
| `rethinkdbPassword`                     | Password for the RethinkDB Admin user                                              | Randomly generated if not set                       |

*RethinkDB config files*

The RethinkDB Cluster and RethinkDB Proxy config files are configured via the `cluster.rethinkdbConfig` and `proxy.rethinkdbConfig` configuration values.

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```console
$ helm install --name my-release -f values.yaml ./helm-rethinkdb-cluster
```

> **Tip**: You can use the default [values.yaml](values.yaml)

## Important: Admin Password Management ##

The initial admin password is set by the config value `rethinkdbPassword`.  This value is also used by the probe which periodically checks if the RethinkDB Cluster and Proxy are still running.  If you change the RethinkDB admin password via a query (i.e. `r.db('rethinkdb').table('users').update({password: 'new-password'})`) this will cause the probe to fail which then restarts the pods over and over.  To fix this issue, you also need to use `helm upgrade` to update the password in the Kubernetes Secrets storage by doing:
```bash
$ helm upgrade --set rethinkdbPassword=new-password my-release ./helm-rethinkdb-cluster
```
If you perform these steps quick enough, there shouldn't be any pods restarting.

## Opening Up the RethinkDB Admin Console

The admin port is not available outside of the cluster for security reasons. The only way to access the admin console is to use a [Kubernetes Port-Forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/). Also, the admin console is only available through a running RethinkDB Proxy. Here is an example:

```bash
$ kubectl get pods -l release-my-release
NAME                                          READY     STATUS    RESTARTS   AGE
my-release-rethinkdb-cluster-0                1/1       Running   0          1m
my-release-rethinkdb-cluster-1                1/1       Running   0          2m
my-release-rethinkdb-cluster-2                1/1       Running   0          2m
my-release-rethinkdb-proxy-2517940628-81dxd   1/1       Running   1          1m

$ kubectl port-forward my-release-rethinkdb-proxy-2517940628-81dxd 8080:8080
```
And then open up your browser to http://localhost:8080 and you should see the admin console

## Cleanup orphaned Persistent Volumes

Deleting a StateFul will not delete associated Persistent Volumes.

Do the following after deleting the chart release to clean up orphaned Persistent Volumes.

```bash
$ kubectl delete pvc -l release=my-release
```

## Failover

If any RethinkDB server fails it gets re-joined eventually.
You can test the scenario by killing process of one of the pods:
```bash
$ kubectl get pods -l release=my-release
NAME                                          READY     STATUS    RESTARTS   AGE
my-release-rethinkdb-cluster-0                1/1       Running   0          1m
my-release-rethinkdb-cluster-1                1/1       Running   0          2m
my-release-rethinkdb-cluster-2                1/1       Running   0          2m
my-release-rethinkdb-proxy-2517940628-81dxd   1/1       Running   1          1m

$ kubectl exec -it my-release-rethinkdb-cluster-0 -- ps aux | grep 'rethinkdb'
root         7  0.1  2.1 233496 43408 ?        Ssl  16:56   0:00 rethinkdb --ser
root        26  0.0  0.4 146948  8204 ?        S    16:56   0:00 rethinkdb --ser
root       100  0.0  0.7 157192 16060 ?        S    16:56   0:00 rethinkdb --ser

$ kubectl exec -it my-release-rethinkdb-cluster-0 -- kill 7
```

## Scaling

Scaling should be managed by `helm upgrade`, which is the recommended way. Example:
```
$ helm upgrade --set cluster.replicas=2 my-release ./helm-rethinkdb-cluster
```
