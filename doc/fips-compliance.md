
# Integrating Delve-Remote with Compliance Operator

## Overview
To integrate `delve-remote` into the Compliance Operator, we utilize a Kubernetes `ConfigMap` to provide configurations. This approach offers flexibility, allowing users to easily customize the behavior of the `delve-remote` scan without changing the operator code.

## Creating the Delve-Remote ConfigMap

1. Prepare a `ConfigMap` YAML file with the necessary configurations.
2. Apply the `ConfigMap` to your Kubernetes cluster.

### ConfigMap Fields:

- **delveImage**: The Docker image for the `delve-remote` tool.
- **delveCommand**: The command to run the `delve-remote` tool within its container.

### Sample ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: delve-remote-config
  namespace: compliance-operator-namespace
data:
  delveImage: "delve-remote:latest" # Replace with your specific image tag/version
  delveCommand: "/path/to/delve-remote" # Replace with the actual command to run delve-remote
```

To apply this `ConfigMap`:

```bash
kubectl apply -f your-configmap-file.yaml
```

## Using the ConfigMap in a ComplianceScan

When creating a `ComplianceScan` resource that should use the `delve-remote` scan, specify the `DelveRemoteConfigMap` field with the name of the `ConfigMap` you created.

Example:

```yaml
apiVersion: compliance.openshift.io/v1alpha1
kind: ComplianceScan
metadata:
  name: example-scan
spec:
  scanType: DelveRemote
  DelveRemoteConfigMap: "delve-remote-config"
  ...
```
