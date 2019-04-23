---
layout: "docs"
page_title: "ACL Auth Methods"
sidebar_current: "docs-acl-auth-methods"
description: |-
  An Auth Method is a component in Consul that performs authentication against a trusted external party to authorize the creation of an appropriately scoped ACL Token usable within the local datacenter.
---

-> **1.5.0+:**  This guide only applies in Consul versions 1.5.0 and later.

# ACL Auth Methods

An Auth Method is a component in Consul that performs authentication against a
trusted external party to authorize the creation of an appropriately scoped ACL
Token usable within the local datacenter.

The only supported type of auth method in Consul 1.5 is
[`kubernetes`](/docs/acl/auth-methods/kubernetes.html) but it is expected that
more will come later.

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

## Binding Rules

Binding rules allow an operator to express a systematic way to automatically
assign Roles and Service Identities to newly created Tokens without operator intervention. For

[roles](/docs/acl/acl-system.html#acl-roles)
and
[service identities](/docs/acl/acl-system.html#acl-service-identities)

authentication originating in a configured Identity Provider that assertion can
be made with a new construct: Binding Rules.

xxx

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

