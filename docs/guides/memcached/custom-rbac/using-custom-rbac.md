---
title: Run Memcached with Custom RBAC resources
menu:
  docs_0.12.0:
    identifier: mc-custom-rbac-quickstart
    name: Custom RBAC
    parent: mc-custom-rbac
    weight: 10
menu_name: docs_0.12.0
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# Using Custom RBAC resources

KubeDB (version 0.13.0 and higher) supports finer user control over role based access permissions provided to a Memcached instance. This tutorial will show you how to use KubeDB to run Memcached database with custom RBAC resources.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [minikube](https://github.com/kubernetes/minikube).

Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

> Note: YAML files used in this tutorial are stored in [docs/examples/memcached](https://github.com/kubedb/docs/tree/0.12.0/docs/examples/memcached) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Overview

KubeDB allows users to provide custom RBAC resources, namely, `ServiceAccount`, `Role`, and `RoleBinding` for Memcached. This is provided via the `spec.podTemplate.spec.serviceAccountName` field in Memcached CRD. If the name of a custom service account is given the user, the KubeDB operator will not dynamically gratn permissions to this service account and use existing access permissions associated with it to run that PosgreSQL database.

This guide will show you how to create custom `Service Account`, `Role`, and `RoleBinding` for a PosgreSQL Database named `quick-postges` to provide the bare minimum access permissions.

## Custom RBAC for Memcached

At first, let's create a `Service Acoount` in `demo` namespace.

```console
$ kubectl create serviceaccount -n demo my-custom-serviceaccount
serviceaccount/my-custom-serviceaccount created
```

It should create a service account.

```yaml
$ kubectl get serviceaccount -n demo my-custom-serviceaccount -o yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  creationTimestamp: "2019-05-30T04:23:39Z"
  name: my-custom-serviceaccount
  namespace: demo
  resourceVersion: "21657"
  selfLink: /api/v1/namespaces/demo/serviceaccounts/myserviceaccount
  uid: b2ec2b05-8292-11e9-8d10-080027a8b217
secrets:
- name: myserviceaccount-token-t8zxd
```

Now, we need to create a role that has necessary access permissions for the Memcached Database named `quick-memcached`.

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/memcached/custom-rbac/mc-custom-role.yaml
role.rbac.authorization.k8s.io/my-custom-role created
```

Below is the YAML for the Role we just created.

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: my-custom-role
  namespace: demo
rules:
- apiGroups:
  - policy
  resourceNames:
  - memcached-db
  resources:
  - podsecuritypolicies
  verbs:
  - use
```

This permission is required for Memcached pods running on PSP enabled clusters.

Now create a `RoleBinding` to bind this `Role` with the already created service account.

```console
$ kubectl create rolebinding my-custom-rolebinding --role=my-custom-role --serviceaccount=demo:my-custom-serviceaccount --namespace=demo
rolebinding.rbac.authorization.k8s.io/my-custom-rolebinding created

```

It should bind `my-custom-role` and `my-custom-serviceaccount` successfully.

```yaml
$ kubectl get rolebinding -n demo my-custom-rolebinding -o yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  creationTimestamp: "kubectl get rolebinding -n demo my-custom-rolebinding -o yaml"
  name: my-custom-rolebinding
  namespace: demo
  resourceVersion: "1405"
  selfLink: /apis/rbac.authorization.k8s.io/v1/namespaces/demo/rolebindings/my-custom-rolebinding
  uid: 123afc02-8297-11e9-8d10-080027a8b217
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: my-custom-role
subjects:
- kind: ServiceAccount
  name: my-custom-serviceaccount
  namespace: demo

```

Now, create a Memcached CRD specifying `spec.podTemplate.spec.serviceAccountName` field to `my-custom-serviceaccount`.

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/memcached/custom-rbac/mc-custom-db.yaml
memcached.kubedb.com/quick-memcached created
```

Below is the YAML for the Memcached crd we just created.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: Memcached
metadata:
  name: quick-memcached
  namespace: demo
spec:
  replicas: 3
  version: "1.5.4-v1"
  podTemplate:
    spec:
    serviceAccountName: my-custom-serviceaccount
      resources:
        limits:
          cpu: 500m
          memory: 128Mi
        requests:
          cpu: 250m
          memory: 64Mi
  terminationPolicy: DoNotTerminate    

```

Now, wait a few minutes. the KubeDB operator will create necessary PVC, statefulset, services, secret etc. If everything goes well, we should see that a pod with the name `quick-memcached-0` has been created.

Check that the statefulset's pod is running

```console
$ kubectl get pod -n demo quick-memcached-0
NAME                READY     STATUS    RESTARTS   AGE
quick-memcached-0   1/1       Running   0          14m
```

Check the pod's log to see if the database is ready

```console
$ kubectl logs -f -n demo quick-memcached-0
Initializing database
2019-05-31T05:02:35.307699Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:02:35.307762Z 0 [System] [MY-013169] [Server] /usr/sbin/memcachedd (memcachedd 8.0.14) initializing of server in progress as process 29
2019-05-31T05:02:47.346326Z 5 [Warning] [MY-010453] [Server] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option.
2019-05-31T05:02:53.777918Z 0 [System] [MY-013170] [Server] /usr/sbin/memcachedd (memcachedd 8.0.14) initializing of server has completed
Database initialized
Memcached init process in progress...
Memcached init process in progress...
2019-05-31T05:02:56.656884Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:02:56.656953Z 0 [System] [MY-010116] [Server] /usr/sbin/memcachedd (memcachedd 8.0.14) starting as process 80
2019-05-31T05:02:57.876853Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2019-05-31T05:02:57.892774Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/memcachedd' in the path is accessible to all OS users. Consider choosing a different directory.
2019-05-31T05:02:57.910391Z 0 [System] [MY-010931] [Server] /usr/sbin/memcachedd: ready for connections. Version: '8.0.14'  socket: '/var/run/memcachedd/memcachedd.sock'  port: 0  Memcached Community Server - GPL.
2019-05-31T05:02:58.045050Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/run/memcachedd/memcachedx.sock'
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it.

2019-05-31T05:03:04.217396Z 0 [System] [MY-010910] [Server] /usr/sbin/memcachedd: Shutdown complete (memcachedd 8.0.14)  Memcached Community Server - GPL.

Memcached init process done. Ready for start up.
```

Once we see `Memcached init process done. Ready for start up.` in the log, the database is ready.

## Reusing Service Account

An existing service account can be reused in another Memcached Database. No new access permission is required to run the new Memcached Database.

Now, create Memcached CRD `minute-memcached` using the existing service account name `my-custom-serviceaccount` in the `spec.podTemplate.spec.serviceAccountName` field.

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/memcached/custom-rbac/mc-custom-db-two.yaml
memcached.kubedb.com/quick-memcached created
```

Below is the YAML for the Memcached crd we just created.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: Memcached
metadata:
  name: minute-memcached
  namespace: demo
spec:
  replicas: 3
  version: "1.5.4-v1"
  podTemplate:
    spec:
    serviceAccountName: my-custom-serviceaccount
      resources:
        limits:
          cpu: 500m
          memory: 128Mi
        requests:
          cpu: 250m
          memory: 64Mi
  terminationPolicy: DoNotTerminate  

```

Now, wait a few minutes. the KubeDB operator will create necessary PVC, statefulset, services, secret etc. If everything goes well, we should see that a pod with the name `minute-memcached-0` has been created.

Check that the statefulset's pod is running

```console
$ kubectl get pod -n demo minute-memcached-0
NAME                READY     STATUS    RESTARTS   AGE
minute-memcached-0   1/1       Running   0          14m
```

Check the pod's log to see if the database is ready

```console
$ kubectl logs -f -n demo minute-memcached-0
Initializing database
2019-05-31T05:09:12.165236Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:09:12.165298Z 0 [System] [MY-013169] [Server] /usr/sbin/memcachedd (memcachedd 8.0.14) initializing of server in progress as process 28
2019-05-31T05:09:24.903995Z 5 [Warning] [MY-010453] [Server] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option.
2019-05-31T05:09:30.857155Z 0 [System] [MY-013170] [Server] /usr/sbin/memcachedd (memcachedd 8.0.14) initializing of server has completed
Database initialized
Memcached init process in progress...
Memcached init process in progress...
2019-05-31T05:09:33.931254Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:09:33.931315Z 0 [System] [MY-010116] [Server] /usr/sbin/memcachedd (memcachedd 8.0.14) starting as process 79
2019-05-31T05:09:34.819349Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2019-05-31T05:09:34.834673Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/memcachedd' in the path is accessible to all OS users. Consider choosing a different directory.
2019-05-31T05:09:34.850188Z 0 [System] [MY-010931] [Server] /usr/sbin/memcachedd: ready for connections. Version: '8.0.14'  socket: '/var/run/memcachedd/memcachedd.sock'  port: 0  Memcached Community Server - GPL.
2019-05-31T05:09:35.064435Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/run/memcachedd/memcachedx.sock'
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it.

2019-05-31T05:09:41.236940Z 0 [System] [MY-010910] [Server] /usr/sbin/memcachedd: Shutdown complete (memcachedd 8.0.14)  Memcached Community Server - GPL.

Memcached init process done. Ready for start up.

```

`Memcached init process done. Ready for start up.` in the log signifies that the database is running successfully.

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mc/quick-memcached -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mc/quick-memcached

kubectl patch -n demo mc/minute-memcached -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mc/minute-memcached

kubectl delete -n demo role my-custom-role
kubectl delete -n demo rolebinding my-custom-rolebinding

kubectl delete sa -n demo my-custom-serviceaccount

kubectl delete ns demo
```

If you would like to uninstall the KubeDB operator, please follow the steps [here](/docs/setup/uninstall.md).

## Next Steps

- [Quickstart Memcached](/docs/guides/memcached/quickstart/quickstart.md) with KubeDB Operator.
- [Snapshot and Restore](/docs/guides/memcached/snapshot/backup-and-restore.md) process of Memcached databases using KubeDB.
- Take [Scheduled Snapshot](/docs/guides/memcached/snapshot/scheduled-backup.md) of Memcached databases using KubeDB.
- Initialize [Memcached with Script](/docs/guides/memcached/initialization/using-script.md).
- Initialize [Memcached with Snapshot](/docs/guides/memcached/initialization/using-snapshot.md).
- Monitor your Memcached database with KubeDB using [out-of-the-box CoreOS Prometheus Operator](/docs/guides/memcached/monitoring/using-coreos-prometheus-operator.md).
- Monitor your Memcached database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/memcached/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/memcached/private-registry/using-private-registry.md) to deploy Memcached with KubeDB.
- Use [kubedb cli](/docs/guides/memcached/cli/cli.md) to manage databases like kubectl for Kubernetes.
- Detail concepts of [Memcached object](/docs/concepts/databases/memcached.md).
- Detail concepts of [Snapshot object](/docs/concepts/snapshot.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).

