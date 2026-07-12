---
title: "Extracting the Input That OPA Reviews"
date: 2020-10-01T21:30:12+09:00
categories: ["Tanzu Mission Control"]
tags: ["Open Policy Agent"]
---

# Introduction

As covered in the [Learning OPA with TMC](../tmc-tmc_demanabu_opa) series, OPA is a powerful tool.
You can use the Rego Playground to test the Rego language.<!--more-->

https://play.openpolicyagent.org/

But how do you actually extract the information that becomes this "Input"?... It turns out the method is properly documented:

https://github.com/open-policy-agent/gatekeeper#viewing-the-request-object

This post is a note on how to do it.

# How to

First, create a ConstraintTemplate that returns an error for everything, like this:

```
cat <<EOF | kubectl apply -f -
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8sdenyall
spec:
  crd:
    spec:
      names:
        kind: K8sDenyAll
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sdenyall

        violation[{"msg": msg}] {
          msg := sprintf("REVIEW OBJECT: %v", [input.review])
        }
EOF
```

Then create a constraint based on it.
In this example, it triggers on Namespace creation.

```
cat <<EOF | kubectl apply -f -
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sDenyAll
metadata:
  name: deny-all-namespaces
spec:
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Namespace"]
EOF
```

In this state, creating a Namespace spits out the JSON usable as Input along with the error.

```
kubectl create ns hoge
Error from server ([denied by deny-all-namespaces] REVIEW OBJECT: {"object": {"apiVersion": "v1", "kind": "Namespace", "metadata": {"uid": "39365652-a7fb-4463-b23d-e00c6dc43374", "creationTimestamp": "2020-10-01T13:33:52Z", "name": "hoge"}, "spec": {"finalizers": ["kubernetes"]}, "status": {"phase": "Active"}}, "oldObject": null, "uid": "11b3051f-1c27-4d82-8bac-d4a0535b01e3", "requestKind": {"group": "", "version": "v1", "kind": "Namespace"}, "userInfo": {"username": "kubernetes-admin", "groups": ["system:masters", "system:authenticated"]}, "name": "hoge", "operation": "CREATE", "dryRun": false, "options": {"kind": "CreateOptions", "apiVersion": "meta.k8s.io/v1"}, "_unstable": {}, "kind": {"group": "", "version": "v1", "kind": "Namespace"}, "resource": {"resource": "namespaces", "group": "", "version": "v1"}, "requestResource": {"group": "", "version": "v1", "resource": "namespaces"}}): admission webhook "validation.gatekeeper.sh" denied the request: [denied by deny-all-namespaces] REVIEW OBJECT: {"object": {"apiVersion": "v1", "kind": "Namespace", "metadata": {"uid": "39365652-a7fb-4463-b23d-e00c6dc43374", "creationTimestamp": "2020-10-01T13:33:52Z", "name": "hoge"}, "spec": {"finalizers": ["kubernetes"]}, "status": {"phase": "Active"}}, "oldObject": null, "uid": "11b3051f-1c27-4d82-8bac-d4a0535b01e3", "requestKind": {"group": "", "version": "v1", "kind": "Namespace"}, "userInfo": {"username": "kubernetes-admin", "groups": ["system:masters", "system:authenticated"]}, "name": "hoge", "operation": "CREATE", "dryRun": false, "options": {"kind": "CreateOptions", "apiVersion": "meta.k8s.io/v1"}, "_unstable": {}, "kind": {"group": "", "version": "v1", "kind": "Namespace"}, "resource": {"resource": "namespaces", "group": "", "version": "v1"}, "requestResource": {"group": "", "version": "v1", "resource": "namespaces"}}
```

Handy.

# Summary

Getting OPA's Input is easy.
