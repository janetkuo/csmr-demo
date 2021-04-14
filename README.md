# Config Sync Examples

## Multi-Repo mode, unstructured format

For Config Sync multi-repo mode with unstructured format, use this [example](./root-multirepo-unstructured).
The example contains `ClusterRole`, `CustomResourceDefinition`, policy constraints, and configurations for Prometheus Operator for monitoring.

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
  # Enable policy controller to enforce constraints
  policyController:
    enabled: true
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
    repo: https://github.com/janetkuo/csmr-demo.git
    branch: main
    dir: "root-multirepo-unstructured"
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
