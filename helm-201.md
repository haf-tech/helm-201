---
title: Helm 201
layout:
 1: left
---

# Helm 201

A deep-dive into Helm (v3) and details like

* [Init](#init)
* [Templating](#chart-structure-and-overview)
* [Charts and Subcharts](#charts-and-subcharts)
* [Life Cylce & Hooks](#life-cycle-and-hooks)
* [OCI Registry](#oci-registry)
* Misc
  * [Usage and internal structure in Kubernetes](#internalrelease-state)
  * [IBM Cloud CR as OCI Registry](#ibm-cloud-container-registry)
* [References](#references)

---

## Init

```shell
helm version
```

```shell
export wd_init="/Users/haddouti/Documents/Data/workspaces/repos/haf-tech/helm-201/work/helm-init3"
```

Let's create a new Helm chart

```shell
echo $wd_init
```

```shell
mkdir -p $wd_init
```

```shell
helm create $wd_init/demo-helm-201
```

```shell
tree $wd_init
```

## Chart Structure and Overview

* `Chart.yaml` common meta information
  * avoid using `appVersion` - handle version in env-specific value-files or as field which will be overwritten during helm execution

```shell
cat $wd_init/demo-helm-201/Chart.yaml | grep -B2 -i 'version:'
```

???+ tip "Goal"
    Templating
    * include output/result from namespaced functions
    * Whitespaces and new lines, indent

???+ tip "Action"
    Action / Scripts #00
    * Intro template and `include` 
    * modify service template


Render template and generate Kubernetes resource files

```shell
helm template demo-helm-201-common $wd_init/demo-helm-201 -f $wd_init/demo-helm-201/values.yaml --output-dir=work/out/common
```

???+ tip "Goal"
     Templates and Variables
     * `.Values.*` holds all variables (from file and command line)
     * `values.yaml` and any additional values-file will be merged

???+ tip "Action"
     Action / Scripts #01
     * `image.version`
     * env-specific values file

Render template and generate Kubernetes resource files for *TEST* stage

```shell
helm template demo-helm-201-test $wd_init/demo-helm-201 -f $wd_init/demo-helm-201/values.test.yaml --output-dir=work/out/test
```


???+ tip "Goal"
    Templating, Functions
    * `_helpers.tpl` holds set of helper functions used in template files
    * common Go template functions are included (and, or, len, ...)

???+ tip "Action"
     Action / Scripts
     * go through `_helpers.tpl`

```shell
cat $wd_init/demo-helm-201/templates/_helpers.tpl
```

???+ tip "Goal"
    Templating, Functions
    * `with` set the scope
    * `range` iterate

???+ tip "Action"
    Action / Scripts #02
    * new `sa.yaml`
    * values in `values.test.yaml`

Render template and generate Kubernetes resource files for *TEST* stage

```shell
helm template demo-helm-201-test $wd_init/demo-helm-201 -f $wd_init/demo-helm-201/values.test.yaml --output-dir=work/out/test
```

### Charts and Subcharts

???+ tip "Goal"
    Charts and Subcharts
    * Subchart is a standalone chart
    * only parent knows the subcharts / childrens
    * ...and only parent can override fields

???+ tip "Action"
    Action 
    * checkout existing chart
    * create a new chart in `parent/chart` directory
    * configure dependency

Print out existing chart (without subchart)

```shell
tree $wd_init
```

Create new (sub)chart in the *charts* dir


```shell
helm create $wd_init/demo-helm-201/charts/demo-subchart
```

```shell
rm -rf $wd_init/demo-helm-201/charts/demo-subchart/templates/*
```

```shell
tree $wd_init
```


???+ tip "Action"
    Action / Scripts #03
    * template for `ConfigMap` in subchart
    * value file in subchart
    * override options

Dry-Run subchart - with local value file

```shell
helm install --generate-name --dry-run --debug $wd_init/demo-helm-201/charts/demo-subchart
```

Render entire chart - with *pre* value file, to override subchart fields

```shell
helm template demo-helm-201-pre $wd_init/demo-helm-201 -f $wd_init/demo-helm-201/values.pre.yaml --output-dir=work/out/pre
```

Render entire chart - with *test* value file, to *NOT* override subchart fields

```shell
helm template demo-helm-201-test $wd_init/demo-helm-201 -f $wd_init/demo-helm-201/values.test.yaml --output-dir=work/out/test
```

??? tip "Be aware to consider that fields/values are not available/null in your template"


## Life Cycle and Hooks

Hooks allows to execute specific resource definitions at defined points in the Helm life cycle.
Available hooks

* `pre-install`
* `post-install`
* `pre-delete`
* `post-delete`
* `pre-upgrade`
* `post-upgrade`
* `pre-rollback`
* `post-rollback`
* `test`

Examples

* prepare the installation and create specific resources beforehand (ConfigMap, Job etc)
* clean up database before uninstalling application
* return a license or deregister/unsubscribe

### Pre-Install and Post-Delete

Example for `pre-install` and `post-delete` hook. Hooks are usual Kubernetes resource definitions with special annotations.

```
metadata:
  annotations:
    "helm.sh/hook": pre-install, post-delete
    "helm.sh/hook-weight": "-5"
    "helm.sh/hook-delete-policy": hook-succeeded
```

Create new (sub)chart in the *charts* dir

```shell
helm create $wd_init/demo-helm-201/charts/demo-hook
```


```shell
rm -rf $wd_init/demo-helm-201/charts/demo-hook/templates/ingress.yaml && \
rm -rf $wd_init/demo-helm-201/charts/demo-hook/templates/service.yaml && \
rm -rf $wd_init/demo-helm-201/charts/demo-hook/templates/tests && \
tree $wd_init
```

???+ tip "Action"
    Action / Scripts #04
    * create 2 jobs for different hooks
    * install and upgrade

Install subchart for hook testing
...some OCP permissions adjustments (for nginx)...

```shell
oc adm policy add-scc-to-user anyuid -z demo-hook-dev
```

```shell
helm upgrade --install demo-hook-dev $wd_init/demo-helm-201/charts/demo-hook -n demo-helm
```

Update subchart for hook testing now with job definitions

```shell
helm upgrade --install demo-hook-dev $wd_init/demo-helm-201/charts/demo-hook -n demo-helm
```

???+ tip "Action"
    Action
    * current `pods` and `jobs`
    * events
    * clean-up and check out the job with `post-delete`

Delete subchart which triggers hook

```shell
helm delete demo-hook-dev -n demo-helm
```

## OCI Registry

Helm 3 provides the option to store helm charts not only on a HTTP server, but also in a OCI Registry. This allows the option to hold the artifacts (container image and helm charts) in the same registry.

### Testing

???+ tip "Action"
    Action
    * use a local registry
    * push a local helm chart
    * install from OCI registry

 Run a local registry

```shell
docker run -dp 5000:5000 --restart=always --name registry registry
```

Create helm package from the hook subchart
...helm package...

```shell
helm package $wd_init/demo-helm-201/charts/demo-hook --version 1.0.3 --destination $wd_init && \
helm package $wd_init/demo-helm-201/charts/demo-hook --version 1.0.4 --destination $wd_init
```

Push helm package to registry
...helm registry login...

```shell
helm registry login -u testuser -p testpassword localhost:5000
```

...helm package push...

```shell
helm push $wd_init/demo-hook-1.0.3.tgz oci://localhost:5000/helm-charts && \
helm push $wd_init/demo-hook-1.0.4.tgz oci://localhost:5000/helm-charts
```

Verify helm package in OCI registry
...show chart details...

```shell
helm show chart oci://localhost:5000/helm-charts/demo-hook
```

Install (render) helm package directly from OCI registry
...render existing chart from registry...

```shell
helm template release-demo-hook-oci oci://localhost:5000/helm-charts/demo-hook --version 1.0.3 --output-dir=work/out/oci
```

...try to render non-existing chart from registry...

```shell
helm template release-demo-hook-oci oci://localhost:5000/helm-charts/demo-hook --version 2.2.2 --output-dir=work/out/oci
```

## Misc

Some additional details

### Internal/Release State

With Helm3 not Tiller exists anymore. All configuration and release state, including value files, is now stored in a `Secret` instead of a `ConfigMap`.

Install subchart

```shell
helm upgrade --install release-demo-hook-state $wd_init/demo-helm-201/charts/demo-hook -n demo-helm
```

List all existing helm releases

```shell
helm list
```

Print out release information from Helm Release Secret

```shell
oc get secret sh.helm.release.v1.release-demo-hook-state.v1 -o jsonpath='{.data.release}' -n demo-helm | base64 --decode | base64 --decode | gunzip -c | jq
```

### Helm Upgrade and force re-deployment

Helm determines the relevant changes and updates only the necessary resources. In case the content of a `ConfigMap` changed, but not the image version, a redeployment or restart of the deployment will be not triggered.
To trigger a redeployment a relation between `ConfigMap` content and `Deployment` is needed

```
kind: Deployment
spec:
  template:
    metadata:
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

The idea is, that the `Deployment` contains an annotation with a checksum of the related `ConfigMap`. Any changes in the content, results in an other `sha256sum`, results in an other annotation, which results in modified `Deployment` definition.

### IBM Cloud Container Registry 

The IBM Cloud Container Registry is OCI compliant and could also be used for Helm packages

Push chart to IBM CR

```
ibmcloud login --sso
ibmcloud cr login
helm registry login -u iamapikey
```

```shell
helm push $wd_init/demo-hook-1.0.1.tgz oci://de.icr.io/demo-helm
```

### VS Code Extensions

* `yaml`
* `kubernetes` understands also the helm template syntax

## Summary

This was a walkthrough for Helm 201 with covering topics like

* templating
* charts and subcharts
* OCI Registry

## References

* [Helm - Storage Provider - Secret instead ConfigMap](https://helm.sh/docs/topics/advanced/#storage-backends)
* [Helm - (OCI) Registry](https://helm.sh/docs/topics/registries/)
* [Helm - Best Practices](https://helm.sh/docs/chart_best_practices/conventions/)
* [Helm - Template Guide](https://helm.sh/docs/chart_template_guide/getting_started/)