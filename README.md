# RemoteResourceS3

[![Build Status](https://travis-ci.com/razee-io/RemoteResourceS3.svg?branch=master)](https://travis-ci.com/razee-io/RemoteResourceS3)
[![Greenkeeper badge](https://badges.greenkeeper.io/razee-io/RemoteResourceS3.svg)](https://greenkeeper.io/)
![GitHub](https://img.shields.io/github/license/razee-io/RemoteResourceS3.svg?color=success)

RemoteResourceS3 is a variant of RemoteResource. RemoteResource is the foundation
for implementing continuous deployment with kapitan. It retrieves and applies the
configuration for all resources. RemoteResourceS3 extends that functionality by
supporting authentication to object storage services that implement the S3 interfaces.

## Install

```shell
kubectl apply -f "https://github.com/razee-io/RemoteResourceS3/releases/latest/download/resource.yaml"
```

## Resource Definition

### Sample

```yaml
apiVersion: "kapitan.razee.io/v1alpha1"
kind: RemoteResourceS3
metadata:
  name: <remote_resource_s3_name>
  namespace: <namespace>
spec:
  auth:
    # hmac:
    #   access_key_id: <key id>
    #   secret_access_key: <access key>
    iam:
      response_type: <provider response type>
      grant_type: <provider grant type>
      url: <iam auth provider>
      api_key:
        valueFrom:
          secretKeyRef:
            name: <name of secret resource>
            key: <key of api_key within secret>
  requests:
    - options:
        url: https://<source_repo_url>/<file_name1>
        headers:
          <header_key1>: <header_value1>
          <header_key2>: <header_value2>
    - optional: true
      options:
        url: http://<source_repo_url>/<file_name2>
```

### Required Fields

- `.spec.auth`
  - type: object
  - required: oneOf [[hmac](#HMAC), [iam](#IAM)]
- `.spec.requests`
  - type: array
  - items:
    - type: object
    - required: [[options](#Options)]
    - optional: [[optional](#Optional)]

## Features

### Auth

#### HMAC

`.spec.auth.hmac`

Allows you to connect to s3 buckets using an HMAC key/id pair.

- Schema
  - type: object
  - required: [access_key_id, secret_access_key]
- Required Fields Schema:
  - `.spec.auth.hmac.access_key_id`
    - type: string
    - or
    - type: object
      - required: [valueFrom.secretKeyRef.name, valueFrom.secretKeyRef.key]
      - optional: [valueFrom.secretKeyRef.namespace]
  - `.spec.auth.hmac.secret_access_key`
    - type: string
    - or
    - type: object
      - required: [valueFrom.secretKeyRef.name, valueFrom.secretKeyRef.key]
      - optional: [valueFrom.secretKeyRef.namespace]

#### IAM

`.spec.auth.iam`

Allows you to connect to s3 buckets using an IAM provider and api key.

- Schema
  - type: object
  - required: [response_type, grant_type, url, api_key]
  - Sample values for [IBM Cloud Object Storage](https://cloud.ibm.com/docs/services/cloud-object-storage/cli?topic=cloud-object-storage-curl)
    - response_type: "cloud_iam"
    - grant_type: "urn:ibm:params:oauth:grant-type:apikey"
    - url: "[https://iam.cloud.ibm.com/identity/token](https://iam.cloud.ibm.com/identity/token)"
- Required Fields Schema:
  - `.spec.auth.iam.response_type`
    - type: string
  - `.spec.auth.iam.grant_type`
    - type: string
  - `.spec.auth.iam.url`
    - type: string
  - `.spec.auth.iam.api_key`
    - type: string
    - or
    - type: object
      - required: [valueFrom.secretKeyRef.name, valueFrom.secretKeyRef.key]
      - optional: [valueFrom.secretKeyRef.namespace]

### Requests

#### Options

`.spec.requests.options`

All options defined in an options object will be passed as is to the http request.
This means you can specify things like headers for authentication in this section.

- Schema:
  - type: object
  - required: [url || uri]
  - optional: [any other other options to be passed along with the request]

#### Optional

`.spec.requests.optional`

- DEFAULT: `false`
  - if download or applying child resource fails, RemoteResource will stop
  execution and report error to `.status`.
- `true`
  - if download or applying child resource fails, RemoteResource will continue
  processing the rest of the defined requests, and will report a warning to `.status`.
- Schema:
  - type: boolean

### Managed Resource Labels

#### Reconcile

Child resource: `.metadata.labels[kapitan.razee.io/Reconcile]`

- DEFAULT: `true`
  - A kapitan resource (parent) will clean up a resources it applies (child) when
either the child is no longer in the parent resource definition or the parent is
deleted.
- `false`
  - This behavior can be overridden when a child's resource definition has
the label `kapitan.razee.io/Reconcile=false`.

#### Resource Update Mode

Child resource: `.metadata.labels[kapitan.razee.io/mode]`

Kapitan resources default to merge patching children. This behavior can be
overridden when a child's resource definition has the label
`kapitan.razee.io/mode=<mode>`

Mode options:

- DEFAULT: `MergePatch`
  - A simple merge, that will merge objects and replace arrays. Items previously
  defined, then removed from the definition, will be removed from the live resource.
  - "As defined in [RFC7386](https://tools.ietf.org/html/rfc7386), a Merge Patch
  is essentially a partial representation of the resource. The submitted JSON is
  "merged" with the current resource to create a new one, then the new one is
  saved. For more details on how to use Merge Patch, see the RFC." [Reference](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#patch-operations)
- `StrategicMergePatch`
  - A more complicated merge, the kubernetes apiServer has defined keys to be
  able to intelligently merge arrays it knows about.
  - "Strategic Merge Patch is a custom implementation of Merge Patch. For a
  detailed explanation of how it works and why it needed to be introduced, see
  [StrategicMergePatch](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-api-machinery/strategic-merge-patch.md)."
  [Reference](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#patch-operations)
  - [Kubectl Apply Semantics](https://kubectl.docs.kubernetes.io/pages/app_management/field_merge_semantics.html)
- `EnsureExists`
  - Will ensure the resource is created and is replaced if deleted. Will not
  enforce a definition.

### Debug Individual Resource

`.spec.resources.metadata.labels[kapitan.razee.io/debug]`

Treats the live resource as EnsureExist. If any Kapitan component is enforcing
the resource, and the label `kapitan.razee.io/debug: true` exists on the live
resource, it will treat the resource as ensure exist and not override any changes.
This is useful for when you need to debug a live resource and dont want Kapitan
overriding your changes. Note: this will only work when you add it to live resources.
If you want to have the EnsureExist behavior, see [Resource Update Mode](#Resource-Update-Mode).

- ie: `kubectl label rrs3 <your-rrs3> kapitan.razee.io/debug=true`

### Lock Cluster Updates

Prevents the controller from updating resources on the cluster. If this is the
first time creating the `kapitan-config` ConfigMap, you must delete the running
controller pods so the deployment can mount the ConfigMap as a volume. If the
`kapitan-config` ConfigMap already exists, just add the pair `lock-cluster: true`.

1. `export CONTROLLER_NAME=remoteresources3-controller && export CONTROLLER_NAMESPACE=razee`
1. `kubectl create cm kapitan-config -n $CONTROLLER_NAMESPACE --from-literal=lock-cluster=true`
1. `kubectl delete pods -n $CONTROLLER_NAMESPACE $(kubectl get pods -n $CONTROLLER_NAMESPACE
 | grep $CONTROLLER_NAME | awk '{print $1}' | paste -s -d ',' -)`
