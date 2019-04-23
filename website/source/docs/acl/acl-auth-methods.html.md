---
layout: "docs"
page_title: "ACL Auth Methods"
sidebar_current: "docs-acl-identity-providers"
description: |-
  An Auth Method is a component in Consul that performs authentication against a trusted external party to authorize the creation of an appropriately scoped ACL Token usable within the local datacenter.
---

-> **1.5.0+:**  This guide only applies in Consul versions 1.5.0 and later.

# ACL Auth Methods

An Auth Method is a component in Consul that performs authentication against a
trusted external party to authorize the creation of an appropriately scoped ACL
Token usable within the local datacenter.

The only supported type of auth method in Consul 1.5 is `kubernetes`
but it is expected that more will come later.

## Overview

Without auth methods, a trusted operator needs to be critically involved in the
creation and secure introduction of each ACL Token to every application that
needs one, while ensuring that the policies assigned to these tokens follow the
principle of least-privilege.

When running in environments such as a public cloud or when supervised by a
cluster scheduler, applications may already have access to uniquely identifying
credentials that were delivered securely by the platform. Consul auth method
integrations allow for these credentials to be used to create ACL Tokens with
properly-scoped policies without additional operator intervention.

In Consul 1.5 the focus is around simplifying the creation of tokens with the
privileges necessary to participate in a [Connect](/docs/connect/index.html)
service mesh with minimal operator intervention.

## Operator Configuration

An operator needs to configure each auth method that is to be trusted by
using the API or command line before they can be used by applications.

* **Authentication** - Details about how to authenticate application
  credentials are configured using the `consul acl auth-method` subcommands or
  the corresponding [API endpoints](/api/acl/auth-methods.html). The specific
  details of configuration are type dependent and described below.

* **Authorization** - One or more Binding Rules must be configured defining how
  to translate trusted identity attributes into privileges assigned to the ACL
  Token that is created. These can be managed with the `consul acl
  binding-rule` subcommands or the corresponding [API
  endpoints](/api/acl/binding-rules.html).

## Login Process

1. Applications can use the `consul login` subcommand or the [login API
   endpoint](/api/acl/acl.html#login-to-auth-method) to authenticate to an auth
   method through the Consul leader.

2. The auth method validates the credentials and returns trusted identity
   attributes to the Consul leader.

3. The Consul leader consults the configured set of Binding Rules linked to the
   auth method to find rules that match the trusted identity attributes.

4. If any Binding Rules match an ACL Token is created in the local datacenter
   and linked to the computed Roles and Service Identities.

5. Applications can use the `consul logout` subcommand or the [logout API
   endpoint](/api/acl/acl.html#logout-from-auth-method) to destroy their token
   when it is no longer required.

## Kubernetes Auth Method

The `kubernetes` auth method type is used to authenticate to Consul using a
Kubernetes Service Account Token. This method of authentication makes it easy
to introduce a Consul token into a Kubernetes Pod.

To use an auth method of this type the following are required to be configured:

* **Kubernetes Host** - The address of the Kubernetes API. This should be an
  address that is reachable from all Consul Servers in your datacenter.

* **Kubernetes CA Certificate** - The PEM encoded CA cert for use by the TLS
  client used to talk with the Kubernetes API. NOTE: Every line must end with a
  newline: `\n`

* **Service Account JWT** - A Service Account Token (JWT) used by the Consul
  leader to access the [TokenReview API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#create-tokenreview-v1-authentication-k8s-io)
  and [ServiceAccount API](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.11/#read-serviceaccount-v1-core)
  to validate application JWTs during login. 

The following is an example RBAC configuration snippet to grant the necessary
permissions to a service account named `consul-auth-method`:

```yaml
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: review-tokens
  namespace: default
subjects:
- kind: ServiceAccount
  name: consul-auth-method
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
  name: consul-auth-method
  namespace: default
roleRef:
  kind: ClusterRole
  name: service-account-getter
  apiGroup: rbac.authorization.k8s.io
```

### Kubernetes Login Process

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

### Binding Rules




