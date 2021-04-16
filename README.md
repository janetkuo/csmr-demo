# Config Sync Demo

## Prerequisite 

1. Either install 1.7.0 release of [Anthos Config Management](https://cloud.google.com/anthos-config-management/docs/how-to/installing) and [Config Sync Operator](https://cloud.google.com/anthos-config-management/docs/how-to/installing-config-sync), or install [standalone Config Sync Operator](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/how-to/installing).

## Configuration

For [Config Sync multi-repo mode](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync/how-to/multi-repo) with unstructured format, use this [example](./root-multirepo-unstructured).

The example contains `ClusterRole`, `CustomResourceDefinition`, `Rolebinding`, `Namespace`, and `RepoSync`.

First, create a files with a `ConfigManagement` custom resource:

```yaml
# config-management.yaml
apiVersion: configmanagement.gke.io/v1
kind: ConfigManagement
metadata:
  name: config-management
spec:
  # Enable multi-repo mode to use new features
  enableMultiRepo: true
```

Wait for the `RootSync` and `RepoSync` CRDs to be available:

```console
until kubectl get customresourcedefinitions rootsyncs.configsync.gke.io reposyncs.configsync.gke.io; \
do date; sleep 1; echo ""; done
```

Then create a files with a `RootSync` custom resource:

```yaml
# root-sync.yaml
# If you are using a Config Sync version earlier than 1.7,
# use: apiVersion: configsync.gke.io/v1alpha1
apiVersion: configsync.gke.io/v1beta1
kind: RootSync
metadata:
  name: root-sync
  namespace: config-management-system
spec:
  sourceFormat: unstructured
  git:
    # If you fork this repo, change the url to point to your fork
    repo: https://github.com/janetkuo/csmr-demo.git
    branch: main
    dir: "root"
    # We recommend securing your source repository.
    # Other supported auth: `ssh`, `cookiefile`, `token`, `gcenode`.
    auth: none
    # Refer to a Secret you create to hold the private key, cookiefile, or token.
    # secretRef:
    #   name: SECRET_NAME
```

Then, apply them to the cluster:

```console
kubectl -f config-management.yaml
kubectl -f root-sync.yaml
```

### Root configs

To verify resources in the "root" directory has been synced to the cluster:

```console
kubectl get -f root-sync.yaml -w
kubectl describe -f root-sync.yaml
kubectl get resourcegroups -n config-management-system
kubectl get <resources specified in the "root" directory>
```

### Namespace configs

The configs in the "root" directory contains a `gamestore` namespace and a [`RepoSync` resource](root/reposync-gamestore.yaml) in the `gamestore` namespacing, referencing the "gamestore" directory in this git repository.

To verify resources in the "gamestore" directory has been synced to the cluster:

```console
kubectl get reposync.configsync.gke.io/repo-sync -n gamestore -w
kubectl describe reposync.configsync.gke.io/repo-sync -n gamestore
kubectl get resourcegroups -n gamestore
kubectl get <resources specified in the "gamestore" directory>
```

### Conflict changes

Try to change the value of [configmap/store-inventory](gamestore/configmap-inventory.yaml) annotation `marketplace.com/comments` in the cluster:

```console
kubectl edit configmaps store-inventory -n gamestore
```

The request should be rejected by the admission webhook.

### Valid changes

Try to change the same annotation in your git repository, the change can be synced to the cluster.

Note that you need to update [`RepoSync` resource](root/reposync-gamestore.yaml) to point to your own fork if you want to make changes in git.
