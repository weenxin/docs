---
title: TLS/SSL (Transport Encryption)
menu:
  docs_0.12.0:
    identifier: mg-tls-encryption
    name: TLS/SSL (Transport Encryption)
    parent: mg-tls
    weight: 15
menu_name: docs_0.12.0
section_menu_id: guides
---

> New to KubeDB? Please start [here](/docs/concepts/README.md).

# Run mongodb with TLS/SSL (Transport Encryption)

KubeDB supports providing TLS/SSL encryption (via, `sslMode` and `clusterAuthMode`) for MongoDB. This tutorial will show you how to use KubeDB to run a MongoDB database with TLS/SSL encryption.

## Before You Begin

- At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [minikube](https://github.com/kubernetes/minikube).

- Now, install KubeDB cli on your workstation and KubeDB operator in your cluster following the steps [here](/docs/setup/install.md).

- To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

  ```bash
  $ kubectl create ns demo
  namespace/demo created
  ```

> Note: YAML files used in this tutorial are stored in [docs/examples/mongodb](https://github.com/kubedb/docs/tree/0.12.0/docs/examples/mongodb) folder in GitHub repository [kubedb/docs](https://github.com/kubedb/docs).

## Overview

KubeDB uses following fields to set `SSLMode` & `ClusterAuthMode` in Mongodb to achieve SSL/TLS encryption.

- `spec:`
  - `sslMode`
  - `clusterAuthMode`

Read about the fields in details in [mongodb concept](/docs/concepts/databases/mongodb.md),

Note: `sslMode` affects all kind of mongodb (i.e., `standalone`, `replicaset` and `sharding`) and `clusterAuthMode` provides [ClusterAuthMode](https://docs.mongodb.com/manual/reference/program/mongod/#cmdoption-mongod-clusterauthmode) for mongodb clusters (i.e., `replicaset` and `sharding`).

## TLS/SSL encryption in MongoDB Standalone

Below is the YAML for MongoDB Standalone. Here, [`spec.sslMode`](/docs/concepts/databases/mongodb.md#specsslMode) specifies `sslMode` for `standalone` (which is `preferSSL`).

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-tls
  namespace: demo
spec:
  version: "3.6-v4"
  sslMode: preferSSL
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

### Deploy MongoDB Standalone

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/mongodb/tls-ssl-encryption/tls-standalone.yaml
mongodb.kubedb.com/mgo-tls created
```

Now, wait until `mgo-tls created` has status `Running`. i.e,

```bash
$ kubectl get mg -n demo
NAME      VERSION   STATUS    AGE
mgo-tls   3.6-v4    Running   20s
```

### Verify TLS/SSL in MongoDB Standalone

Now, connect to this database through [mongo-shell](https://docs.mongodb.com/v3.4/mongo/) and verify if `SSLMode` has been set up as intended (i.e, `preferSSL`).

```bash
$ kubectl get secrets -n demo mgo-tls-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo mgo-tls-auth -o jsonpath='{.data.\password}' | base64 -d
aRhBCAprS_2TFXaJ

$ kubectl exec -it mgo-tls-0 -n demo bash

> mongo admin -u root -p aRhBCAprS_2TFXaJ

> db.adminCommand({ getParameter:1, sslMode:1 })
{ "sslMode" : "preferSSL", "ok" : 1 }

> exit
bye
```

You can see here that, `sslMode` is set to `preferSSL`.

## TLS/SSL encryption in MongoDB Replicaset

Below is the YAML for MongoDB Replicaset. Here, [`spec.sslMode`](/docs/concepts/databases/mongodb.md#specsslMode) specifies `sslMode` for `replicaset` (which is `preferSSL`) and [`spec.clusterAuthMode`](/docs/concepts/databases/mongodb.md#specclusterAuthMode) provides `clusterAuthMode` for mongodb replicaset nodes (which is `x509`).

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mgo-rs-tls
  namespace: demo
spec:
  version: "3.6-v4"
  sslMode: preferSSL
  clusterAuthMode: x509
  replicas: 4
  replicaSet:
    name: rs0
  storage:
    storageClassName: "standard"
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 1Gi
```

### Deploy MongoDB Replicaset

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/mongodb/tls-ssl-encryption/tls-replicaset.yaml
mongodb.kubedb.com/mgo-rs-tls created
```

Now, wait until `mgo-rs-tls created` has status `Running`. i.e,

```bash
$ kubectl get mg -n demo
NAME         VERSION   STATUS    AGE
mgo-rs-tls   3.6-v4    Running   2m31s
```

### Verify TLS/SSL in MongoDB Replicaset

Now, connect to this database through [mongo-shell](https://docs.mongodb.com/v3.4/mongo/) and verify if `SSLMode` and `ClusterAuthMode` has been set up as intended.

```bash
$ kubectl get secrets -n demo mgo-rs-tls-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo mgo-rs-tls-auth -o jsonpath='{.data.\password}' | base64 -d
Z8vMPTlv7aUUQ7Wg

$ kubectl exec -it mgo-rs-tls-0 -n demo bash

> mongo admin -u root -p Z8vMPTlv7aUUQ7Wg

> db.adminCommand({ getParameter:1, sslMode:1 })
{
	"sslMode" : "preferSSL",
	"ok" : 1,
	"operationTime" : Timestamp(1560170021, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1560170021, 1),
		"signature" : {
			"hash" : BinData(0,"5dklm3nllJ974f4AkL+Nz10tqNI="),
			"keyId" : NumberLong("6700875999464128514")
		}
	}
}
  
> db.adminCommand({ getParameter:1, clusterAuthMode:1 })
{
	"clusterAuthMode" : "x509",
	"ok" : 1,
	"operationTime" : Timestamp(1560170101, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1560170101, 1),
		"signature" : {
			"hash" : BinData(0,"PtzTCV15PEVmr7p8HYmq4E6VT6k="),
			"keyId" : NumberLong("6700875999464128514")
		}
	}
}

> exit
bye
```

You can see here that, `sslMode` is set to `preferSSL` & `clusterAuthMode` is set to `x509`.

## TLS/SSL encryption in MongoDB Sharding

Below is the YAML for MongoDB Sharding. Here, [`spec.sslMode`](/docs/concepts/databases/mongodb.md#specsslMode) specifies `sslMode` for `sharding` and [`spec.clusterAuthMode`](/docs/concepts/databases/mongodb.md#specclusterAuthMode) provides `clusterAuthMode` for sharding servers.

```yaml
apiVersion: kubedb.com/v1alpha1
kind: MongoDB
metadata:
  name: mongo-sh-tls
  namespace: demo
spec:
  version: 3.6-v4
  sslMode: preferSSL
  clusterAuthMode: x509
  shardTopology:
    configServer:
      replicas: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
    mongos:
      replicas: 2
      strategy:
        type: RollingUpdate
    shard:
      replicas: 3
      shards: 3
      storage:
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
  updateStrategy:
    type: RollingUpdate
  storageType: Durable
  terminationPolicy: WipeOut
```

### Deploy MongoDB Sharding

```bash
$ kubectl create -f https://github.com/kubedb/docs/raw/0.12.0/docs/examples/mongodb/tls-ssl-encryption/tls-sharding.yaml
mongodb.kubedb.com/mongo-sh-tls created
```

Now, wait until `mongo-sh-tls created` has status `Running`. ie,

```bash
$ kubectl get mg -n demo
NAME           VERSION   STATUS    AGE
mongo-sh-tls   3.6-v4    Running   9m34s
```

### Verify TLS/SSL in MongoDB Sharding

Now, connect to `mongos` component of this database through [mongo-shell](https://docs.mongodb.com/v3.4/mongo/) and verify if `SSLMode` and `ClusterAuthMode` has been set up as intended.

```bash
$ kubectl get secrets -n demo mongo-sh-tls-auth -o jsonpath='{.data.\username}' | base64 -d
root

$ kubectl get secrets -n demo mongo-sh-tls-auth -o jsonpath='{.data.\password}' | base64 -d
YiMUP8fdkxgXXznz

# get mongos pods
$ kubectl get po -n demo -l mongodb.kubedb.com/node.mongos=mongo-sh-tls-mongos
NAME                                   READY   STATUS    RESTARTS   AGE
mongo-sh-tls-mongos-79fd955765-bkphj   1/1     Running   0          5m48s
mongo-sh-tls-mongos-79fd955765-tr462   1/1     Running   0          5m48s


$ kubectl exec -it mongo-sh-tls-mongos-79fd955765-tr462 -n demo bash

> mongo admin -u root -p YiMUP8fdkxgXXznz

> db.adminCommand({ getParameter:1, sslMode:1 })
{
	"sslMode" : "preferSSL",
	"ok" : 1,
	"operationTime" : Timestamp(1560247204, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1560247204, 1),
		"signature" : {
			"hash" : BinData(0,"yUelxJEG1EF+JRBBB+anhP8XM8c="),
			"keyId" : NumberLong("6701206295334092827")
		}
	}
}
  
> db.adminCommand({ getParameter:1, clusterAuthMode:1 })
{
	"clusterAuthMode" : "x509",
	"ok" : 1,
	"operationTime" : Timestamp(1560247235, 1),
	"$clusterTime" : {
		"clusterTime" : Timestamp(1560247235, 1),
		"signature" : {
			"hash" : BinData(0,"oBjsSPSKnjUt5pSiudyNTX/Uf8w="),
			"keyId" : NumberLong("6701206295334092827")
		}
	}
}

> exit
bye
```

You can see here that, `sslMode` is set to `preferSSL` & `clusterAuthMode` is set to `x509`.

## Changing the SSLMode & ClusterAuthMode

User can update `sslMode` & `ClusterAuthMode` if needed. Some changes may be invalid from mongodb end, like using `sslMode: disabled` with `clusterAuthMode: x509`.

Good thing is, **KubeDB webhook will throw error for invalid SSL specs while creating/updating the mongodb crd object.** i.e.,

```bash
$ kubectl patch -n demo mg/mgo-rs-tls -p '{"spec":{"sslMode": "disabled","clusterAuthMode": "x509"}}' --type="merge"
Error from server (Forbidden): admission webhook "mongodb.validators.kubedb.com" denied the request: can't have disabled set to mongodb.spec.sslMode when mongodb.spec.clusterAuthMode is set to x509
```

To **Upgrade from Keyfile Authentication to x.509 Authentication**, change the `sslMode` and `clusterAuthMode` in recommended sequence as suggested in [official documentation](https://docs.mongodb.com/manual/tutorial/upgrade-keyfile-to-x509/).  Each time after changing the specs, follow the procedure that is described above to verify the changes of `sslMode` and `clusterAuthMode` inside database.

## Snapshot Configuration

Currently snapshot works without any extra configuration in spec of snapshot, when `sslMode` is set to either of `preferSSL`, `allowSSL` or `disabled`.

Note, Snapshot won't work for `requireSSL` at this moment. Track this issue [here in github](https://github.com/kubedb/project/issues/352).

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```bash
kubectl patch -n demo mg/mgo-rs-tls mg/mgo-tls mg/mongo-sh-tls -p '{"spec":{"terminationPolicy":"WipeOut"}}' --type="merge"
kubectl delete -n demo mg/mgo-rs-tls mg/mgo-tls mg/mongo-sh-tls

kubectl patch -n demo drmn/mgo-rs-tls drmn/mgo-tls drmn/mongo-sh-tls -p '{"spec":{"wipeOut":true}}' --type="merge"
kubectl delete -n demo drmn/mgo-rs-tls drmn/mgo-tls drmn/mongo-sh-tls

kubectl delete ns demo
```

## Next Steps

- Detail concepts of [MongoDB object](/docs/concepts/databases/mongodb.md).
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
