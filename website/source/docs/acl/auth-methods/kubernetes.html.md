---
layout: "docs"
page_title: "Kubernetes Auth Method"
sidebar_current: "docs-acl-auth-methods-kubernetes"
description: |-
  An Auth Method is a component in Consul that performs authentication against a trusted external party to authorize the creation of an appropriately scoped ACL Token usable within the local datacenter.
---

-> **1.5.0+:**  This guide only applies in Consul versions 1.5.0 and later.

# Kubernetes Auth Method

The `kubernetes` auth method type allows for a Kubernetes Service Account Token
to be used to authenticate to Consul. This method of authentication makes it
easy to introduce a Consul token into a Kubernetes Pod.

## Config Parameters

The following [`Config`](/api/acl/auth-methods.html#config) parameters are required to
setup a Kubernetes auth method:

- `Host` `(string: <required>)` - Must be a host string, a host:port pair, or a
  URL to the base of the Kubernetes API server. 

- `CACert` `(string: <required>)` - PEM encoded CA cert for use by the TLS
  client used to talk with the Kubernetes API. NOTE: Every line must end with a
  newline (`\n`).

- `ServiceAccountJWT` `(string: <required>)` - A Service Account Token (JWT)
  used by the Consul leader to validate application JWTs during login. 

### Sample Config

```json
{
    ...other fields...
    "Config": {
        "Host": "https://192.0.2.42:8443",
        "CACert": "-----BEGIN CERTIFICATE-----\n...-----END CERTIFICATE-----\n",
        "ServiceAccountJWT": "eyJhbGciOiJSUzI1NiIsImtpZCI6IiJ9..."
    }
}
```

## RBAC

The service account corresponding to the configured
[`ServiceAccountJWT`](/docs/acl/auth-methods/kubernetes.html#serviceaccountjwt)
needs to have access to two Kubernetes APIs:

- [**TokenReview**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#create-tokenreview-v1-authentication-k8s-io)

- [**ServiceAccount**](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#read-serviceaccount-v1-core)
  (`get`)

The following is an example
[RBAC](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)
configuration snippet to grant the necessary permissions to a service account
named `consul-auth-method-example`:

```yaml
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: review-tokens
  namespace: default
subjects:
- kind: ServiceAccount
  name: consul-auth-method-example
  namespace: default
roleRef:
  kind: ClusterRole
  name: system:auth-delegator
  apiGroup: rbac.authorization.k8s.io
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: service-account-getter
  namespace: default
rules:
- apiGroups: [""]
  resources: ["serviceaccounts"]
  verbs: ["get"]
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: get-service-accounts
  namespace: default
subjects:
- kind: ServiceAccount
  name: consul-auth-method-example
  namespace: default
roleRef:
  kind: ClusterRole
  name: service-account-getter
  apiGroup: rbac.authorization.k8s.io
```

## Kubernetes Login Process

1. The Service Account JWT given to the Consul leader initially accesses the
   TokenReview API to validate the provided JWT is still valid. Kubernetes
   should be running with `--service-account-lookup`. This is defaulted to true
   in Kubernetes 1.7, but any versions prior should ensure the Kubernetes API
   server is started with this setting. 

       The trusted values of `serviceaccount.namespace`, `serviceaccount.name`, and
       `serviceaccount.uid` are returned.

2. After validating that the provided JWT is still valid, the Consul leader
   looks for an optional annotation of `consul.hashicorp.com/service-name` on
   the resolved service account using the ServiceAccount API.

       If one is found the trusted value of `serviceaccount.name` is overridden
       with that value.

3. The rest of the login flow proceeds normally.

## Binding Rules




