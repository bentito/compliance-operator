---
Title: Checking complex yaml structures
PrevPage: 12-common-rules.md
NextPage: ../finish
---

Checking complex yaml structures
====================

The `yamlfile_value` template is great to check if a yaml key exists or not,
and it does exist, if it has a specific value.

But some of the resources and configurations of a Kubernetes cluster can be
defined in quite a complex way, and just the yaml path syntax used by the
template may not be sufficient to get to the value we want to assess.

For this reason the CaC/content rules used by the Compliance Operator can
leverage [`jq`](https://jqlang.github.io/jq/) to better select what needs to be assessed.
A `jq` filter allows us to pre-process the resources and configurations so that
they are easier to check with the `yamlfile_value` template.

There are two use cases where a `jq` filter is necessary.

## 1. Checking a key in a nested yaml or json

Some Kuberentes configurations contain a yaml or json formatted value in them,
and checking for these values requires the use of `jq` filter.

For example, looking at an `openshift-kube-api-server` `ConfigMap` we can see
a `data."config.yaml"` key whose value is yaml formatted.

```
$ oc get configmap config -n openshift-kube-apiserver -oyaml
apiVersion: v1
data:
  config.yaml: '{"admission":{"pluginConfig":{"PodSecurity":{"configuration":{"apiVersion":"pod-security.admission.conf
ig.k8s.io/v1","defaults":{"audit":"restricted","audit-version":"latest","enforce":"privileged","enforce-version":"lates
t","warn":"restricted","warn-version":"latest"},"exemptions":{"usernames":["system:serviceaccount:openshift-infra:build
-controller"]},"kind":"PodSecurityConfiguration"}},"network.openshift.io/ExternalIPRanger":{"configuration":{"allowIngr
essIP":false,"apiVersion":"network.openshift.io/v1","externalIPNetworkCIDRs":null,"kind":"ExternalIPRangerAdmissionConf
ig"},"location":""},"network.openshift.io/RestrictedEndpointsAdmission":{"configuration":{"apiVersion":"network.openshi
ft.io/v1","kind":"RestrictedEndpointsAdmissionConfig","restrictedCIDRs":["10.128.0.0/14","172.30.0.0/16"]}}}},"apiServe
rArguments":{"allow-privileged":["true"],"anonymous-auth":["true"],"api-audiences":["https://kubernetes.default.svc"],"
audit-log-format":["json"],"audit-log-maxbackup":["10"],"audit-log-maxsize":["200"],"audit-log-path":["/var/log/kube-ap
iserver/audit.log"],"audit-policy-file":["/etc/kubernetes/static-pod-resources/configmaps/kube-apiserver-audit-policies
...
kind: ConfigMap
metadata:
  creationTimestamp: "2023-10-06T12:59:31Z"
  name: config
  namespace: openshift-kube-apiserver
  resourceVersion: "21469"
  uid: fc192e71-282a-4af6-9492-87324ea83410
```

To check the value of `audit-log-maxbackup`, which is in the `config.yaml` key,
we need to use the `jq` query to select the value and "extract" it for us.

Rule [api_server_audit_log_maxbackup](https://github.com/ComplianceAsCode/content/blob/master/applications/openshift/api-server/api_server_audit_log_maxbackup/rule.yml)
is evaluating the resource listed above.
The rule's `jq` filter is `.data."config.yaml" | fromjson'` and its `yamlpath` is `.apiServerArguments["audit-log-maxbackup"][:]`.
The Compliance Operator will fetch the resource and pass down for checking only the data that came out from the `jq` filter.

![Diagram of resource colection and check with and without a jq filter](images/jqfilter_preprocessing.png)

For this section, we will create a very simple `ConfigMap` with embedded yaml and check one of its keys.

So let's create a machine config that has a nested yaml value:
```
$ cat << EOF | oc create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: my-nested-compliance-configmap
  namespace: openshift
data:
  my-config.yaml: '{foo: bar, nested-key: nested-compliant}'
EOF
```

Now let's write a rule to check whether the value of `nested-key` is `nested-compliant`.

The script `utils/add_kubernetes_rule.py` provides an easy way to create a rule that uses `jq` filters.
Just pass the option `--jqfilter` with the desired filter.
```
$ ./utils/add_kubernetes_rule.py create platform \
    --rule check_nested_yaml \
    --name my-nested-compliance-configmap --namespace openshift --type configmap \
    --title "Check value of nested-key in my-config-yaml" \
    --description "It is important that the nested configmap has a nested-key with value nested-compliant." \
    --match-entity "at least one" \
    --match "nested-compliant" \
    --yamlpath ".nested-key" \
    --jqfilter '.data."my-config.yaml"'
```

Just like when we were creating the other rules, the tool lays down a `rule.yml` file with the same keys filled out.
There are two diffences this time around:
* First, is the warning messsage. Which is now defined by a different macro.
  This macro adds the `jq` filter to the rule, which is parsed by the operator when collecting the resource.
* Second difference is the `filepath` key in the template, which is define with the help of macro too.
  This macro ensures that a unique `filepath` is set for this resource when it is collected.

You can test the rule with:
```
$ ./utils/add_kubernetes_rule.py cluster-test --rule check_nested_yaml
* Testing rule check_nested_yaml in-cluster
* Ensuring openshift-compliance namespace exists.
...
* Running scan with rule 'check_nested_yaml'
> Output from last phase check: LAUNCHING NOT-AVAILABLE
...
> Output from last phase check: RUNNING NOT-AVAILABLE
...
> Output from last phase check: AGGREGATING NOT-AVAILABLE
> Output from last phase check: DONE COMPLIANT
* The result is 'COMPLIANT'
```

If you'd like to test that the rule fails with incompliant values, patch the `ConfigMap` with an incompliant value, and run the test again.
```
$ oc patch -n openshift configmap my-nested-compliance-configmap -p '{"data": {"my-config.yaml": "{foo: bar, nested-key: nested-not-compliant}"}}
configmap/my-nested-compliance-configmap patched
$ ./utils/add_kubernetes_rule.py cluster-test --rule check_nested_yaml
...
* The result is 'NON-COMPLIANT'
```

## 2. Filtering the data to have simpler yamlpaths

This is a generalization of the first use case, `jq` filters can be used to filter and select the data
fetched by the operator.

When the resource being checked is extensive or complex, `jq` is invaluable for simplifying the data before it is
passed down to be evaluated with `yamlpath` in `yamlfile_value` template.

Rule [api_server_encryption_provider_cipher](https://github.com/ComplianceAsCode/content/blob/master/applications/openshift/api-server/api_server_encryption_provider_cipher/rule.yml)
is one example of a rule that filters the data for a simpler `yamlpath`.

And rule [configure_network_policies_hypershift_hosted](https://github.com/ComplianceAsCode/content/blob/master/applications/openshift/networking/configure_network_policies_hypershift_hosted/rule.yml)
is an example of a rule that uses `jq` filters to select attributes from an array to be evaluated.
