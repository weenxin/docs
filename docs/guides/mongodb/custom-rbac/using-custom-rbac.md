---
title: Run MongoDB with Custom RBAC resources
menu:
  docs_0.12.0:
    identifier: mg-custom-rbac-quickstart
    name: Custom RBAC
    parent: mg-custom-rbac
    weight: 10
menu_name: docs_0.12.0
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# Using Custom RBAC resources

KubeDB (version 0.13.0 and higher) supports finer user control over role based access permissions provided to a MongoDB instance. This tutorial will show you how to use KubeDB to run MongoDB database with custom RBAC resources.

## Before You Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [minikube](https://github.com/kubernetes/minikube).

Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace/demo created
```

> Note: YAML files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/0.12.0/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Overview

KubeDB allows users to provide custom RBAC resources, namely, `ServiceAccount`, `Role`, and `RoleBinding` for MongoDB. This is provided via the `spec.podTemplate.spec.serviceAccountName` field in MongoDB CRD. If the name of a custom service account is given the user, the KubeDB operator will not dynamically gratn permissions to this service account and use existing access permissions associated with it to run that PosgreSQL database.

This guide will show you how to create custom `Service Account`, `Role`, and `RoleBinding` for a PosgreSQL Database named `quick-postges` to provide the bare minimum access permissions.

## Custom RBAC for MongoDB

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

Now, we need to create a role that has necessary access permissions for the MongoDB Database named `quick-mongodb`.

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/mongodb/custom-rbac/mg-custom-role.yaml
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
  - mongodb-db
  resources:
  - podsecuritypolicies
  verbs:
  - use
```

This permission is required for MongoDB pods running on PSP enabled clusters.

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

Now, create a MongoDB CRD specifying `spec.podTemplate.spec.serviceAccountName` field to `my-custom-serviceaccount`.

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/mongodb/custom-rbac/mg-custom-db.yaml
mongodb.kubedb.com/quick-mongodb created
```

Below is the YAML for the MongoDB crd we just created.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: quick-mongodb
  namespace: demo
spec:
  version: "3.4-v3"
  storageType: Durable
  podTemplate:
      spec:
        serviceAccountName: my-custom-serviceaccount
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  terminationPolicy: DoNotTerminate

```

Now, wait a few minutes. the KubeDB operator will create necessary PVC, statefulset, services, secret etc. If everything goes well, we should see that a pod with the name `quick-mongodb-0` has been created.

Check that the statefulset's pod is running

```console
$ kubectl get pod -n demo quick-mongodb-0
NAME                READY     STATUS    RESTARTS   AGE
quick-mongodb-0   1/1       Running   0          14m
```

Check the pod's log to see if the database is ready

```console
$ kubectl logs -f -n demo quick-mongodb-0
Initializing database
2019-05-31T05:02:35.307699Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:02:35.307762Z 0 [System] [MY-013169] [Server] /usr/sbin/mongodbd (mongodbd 8.0.14) initializing of server in progress as process 29
2019-05-31T05:02:47.346326Z 5 [Warning] [MY-010453] [Server] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option.
2019-05-31T05:02:53.777918Z 0 [System] [MY-013170] [Server] /usr/sbin/mongodbd (mongodbd 8.0.14) initializing of server has completed
Database initialized
MongoDB init process in progress...
MongoDB init process in progress...
2019-05-31T05:02:56.656884Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:02:56.656953Z 0 [System] [MY-010116] [Server] /usr/sbin/mongodbd (mongodbd 8.0.14) starting as process 80
2019-05-31T05:02:57.876853Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2019-05-31T05:02:57.892774Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/mongodbd' in the path is accessible to all OS users. Consider choosing a different directory.
2019-05-31T05:02:57.910391Z 0 [System] [MY-010931] [Server] /usr/sbin/mongodbd: ready for connections. Version: '8.0.14'  socket: '/var/run/mongodbd/mongodbd.sock'  port: 0  MongoDB Community Server - GPL.
2019-05-31T05:02:58.045050Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/run/mongodbd/mongodbx.sock'
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it.

2019-05-31T05:03:04.217396Z 0 [System] [MY-010910] [Server] /usr/sbin/mongodbd: Shutdown complete (mongodbd 8.0.14)  MongoDB Community Server - GPL.

MongoDB init process done. Ready for start up.
```

Once we see `MongoDB init process done. Ready for start up.` in the log, the database is ready.

## Reusing Service Account

An existing service account can be reused in another MongoDB Database. No new access permission is required to run the new MongoDB Database.

Now, create MongoDB CRD `minute-mongodb` using the existing service account name `my-custom-serviceaccount` in the `spec.podTemplate.spec.serviceAccountName` field.

```console
$ kubectl apply -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/mongodb/custom-rbac/mg-custom-db-two.yaml
mongodb.kubedb.com/quick-mongodb created
```

Below is the YAML for the MongoDB crd we just created.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: minute-mongodb
  namespace: demo
spec:
  version: "3.4-v3"
  storageType: Durable
  podTemplate:
      spec:
        serviceAccountName: my-custom-serviceaccount
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
  terminationPolicy: DoNotTerminate

```

Now, wait a few minutes. the KubeDB operator will create necessary PVC, statefulset, services, secret etc. If everything goes well, we should see that a pod with the name `minute-mongodb-0` has been created.

Check that the statefulset's pod is running

```console
$ kubectl get pod -n demo minute-mongodb-0
NAME                READY     STATUS    RESTARTS   AGE
minute-mongodb-0   1/1       Running   0          14m
```

Check the pod's log to see if the database is ready

```console
$ kubectl logs -f -n demo minute-mongodb-0
Initializing database
2019-05-31T05:09:12.165236Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:09:12.165298Z 0 [System] [MY-013169] [Server] /usr/sbin/mongodbd (mongodbd 8.0.14) initializing of server in progress as process 28
2019-05-31T05:09:24.903995Z 5 [Warning] [MY-010453] [Server] root@localhost is created with an empty password ! Please consider switching off the --initialize-insecure option.
2019-05-31T05:09:30.857155Z 0 [System] [MY-013170] [Server] /usr/sbin/mongodbd (mongodbd 8.0.14) initializing of server has completed
Database initialized
MongoDB init process in progress...
MongoDB init process in progress...
2019-05-31T05:09:33.931254Z 0 [Warning] [MY-011070] [Server] 'Disabling symbolic links using --skip-symbolic-links (or equivalent) is the default. Consider not using this option as it' is deprecated and will be removed in a future release.
2019-05-31T05:09:33.931315Z 0 [System] [MY-010116] [Server] /usr/sbin/mongodbd (mongodbd 8.0.14) starting as process 79
2019-05-31T05:09:34.819349Z 0 [Warning] [MY-010068] [Server] CA certificate ca.pem is self signed.
2019-05-31T05:09:34.834673Z 0 [Warning] [MY-011810] [Server] Insecure configuration for --pid-file: Location '/var/run/mongodbd' in the path is accessible to all OS users. Consider choosing a different directory.
2019-05-31T05:09:34.850188Z 0 [System] [MY-010931] [Server] /usr/sbin/mongodbd: ready for connections. Version: '8.0.14'  socket: '/var/run/mongodbd/mongodbd.sock'  port: 0  MongoDB Community Server - GPL.
2019-05-31T05:09:35.064435Z 0 [System] [MY-011323] [Server] X Plugin ready for connections. Socket: '/var/run/mongodbd/mongodbx.sock'
Warning: Unable to load '/usr/share/zoneinfo/iso3166.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/leap-seconds.list' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone.tab' as time zone. Skipping it.
Warning: Unable to load '/usr/share/zoneinfo/zone1970.tab' as time zone. Skipping it.

2019-05-31T05:09:41.236940Z 0 [System] [MY-010910] [Server] /usr/sbin/mongodbd: Shutdown complete (mongodbd 8.0.14)  MongoDB Community Server - GPL.

MongoDB init process done. Ready for start up.

```

`MongoDB init process done. Ready for start up.` in the log signifies that the database is running successfully.

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
kubectl patch -n demo mg/quick-mongodb -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/quick-mongodb

kubectl patch -n demo mg/minute-mongodb -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/minute-mongodb

kubectl delete -n demo role my-custom-role
kubectl delete -n demo rolebinding my-custom-rolebinding

kubectl delete sa -n demo my-custom-serviceaccount

kubectl delete ns demo
```

If you would like to uninstall the KubeDB operator, please follow the steps [here](/docs/setup/uninstall.md).

## Next Steps

- [Quickstart MongoDB](/docs/guides/mongodb/quickstart/quickstart.md) with KubeDB Operator.
- [Snapshot and Restore](/docs/guides/mongodb/snapshot/backup-and-restore.md) process of MongoDB databases using KubeDB.
- Take [Scheduled Snapshot](/docs/guides/mongodb/snapshot/scheduled-backup.md) of MongoDB databases using KubeDB.
- Initialize [MongoDB with Script](/docs/guides/mongodb/initialization/using-script.md).
- Initialize [MongoDB with Snapshot](/docs/guides/mongodb/initialization/using-snapshot.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box CoreOS Prometheus Operator](/docs/guides/mongodb/monitoring/using-coreos-prometheus-operator.md).
- Monitor your MongoDB database with KubeDB using [out-of-the-box builtin-Prometheus](/docs/guides/mongodb/monitoring/using-builtin-prometheus.md).
- Use [private Docker registry](/docs/guides/mongodb/private-registry/using-private-registry.md) to deploy MongoDB with KubeDB.
- Use [kubedb cli](/docs/guides/mongodb/cli/cli.md) to manage databases like kubectl for Kubernetes.
- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
- Detail concepts of [Snapshot object](/docs/concepts/snapshot.md).
- Want to hack on KubeDB? Check our [contribution guidelines](/docs/CONTRIBUTING.md).

