---
title: "Learning Open Policy Agent with Tanzu Mission Control Part-3"
date: 2020-10-01T21:30:12+09:00
tags: ["Tanzu Mission Control", "Open Policy Agent"]
thumbnail: "2020-09-24T13-09-22.png"
---




This article is part of the Learning Open Policy Agent with Tanzu Mission Control series.<!--more-->

# Series

Part 1 : [Overview](../tmc-demanabu-opa)     
Part 2 : [Playing with TMC's Open Policy Agent](../tmc-demanabu-opa-02)  
Part 3 : Dissecting TMC's Open Policy Agent **< you are here**  
Part 4 : [Writing your own OPA policies from TMC](../tmc-demanabu-opa-04)

# Introduction

As shown in the [overview](../tmc-demanabu-opa), OPA defines policies against Kubernetes requests in a flow like this:

![](2020-09-24T13-25-22.png)

When a Security Policy is enabled from TMC, the same mechanism is at work.

Below we dissect how, with a TMC Security Policy enabled, the Pod startup in [Part 2](../tmc-demanabu-opa-02) was blocked.

This assumes an environment managed by TMC. Follow along with the "With Admission Controller" flow diagram above.

# API > Admission Controller

In TMC, the following resource is defined as the Admission Controller intercepting the API:

```
kubectl get validatingwebhookconfigurations gatekeeper-validating-webhook-configuration
NAME                                          CREATED AT
gatekeeper-validating-webhook-configuration   2020-09-30T14:43:48Z
```

The `validatingwebhookconfigurations` resource defines the Admission Controller's behavior. For details see [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/) in the manual.

At the time I checked, the contents looked like this (key parts excerpted):

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: gatekeeper-validating-webhook-configuration
webhooks:
- admissionReviewVersions:
  - v1beta1
  clientConfig:
    service:
      name: gatekeeper-webhook-service
      namespace: gatekeeper-system
      path: /v1/admit
      port: 443
  failurePolicy: Ignore
  matchPolicy: Exact
  name: validation.gatekeeper.sh
  namespaceSelector:
    matchExpressions:
    - key: admission.gatekeeper.sh/ignore
      operator: DoesNotExist
  objectSelector: {}
  rules:
  - apiGroups:
    - '*'
    apiVersions:
    - '*'
    operations:
    - CREATE
    - UPDATE
    resources:
    - '*'
    scope: '*'
```

Looking at the following portion, it means the Admission Webhook fires on CREATE and UPDATE of every API and every resource:

```yaml
rules:
- apiGroups:
  - '*'
  apiVersions:
  - '*'
  operations:
  - CREATE
  - UPDATE
  resources:
  - '*'
  scope: '*'
```

Additionally, this portion describes the webhook destination:

```yaml
clientConfig:
  service:
    name: gatekeeper-webhook-service
    namespace: gatekeeper-system
    path: /v1/admit
    port: 443
```

In summary:

* The Admission Controller is defined by the `validatingwebhookconfigurations` resource
* The Admission Webhook fires on CREATE and UPDATE of every API and every resource
* The Admission Webhook destination is `gatekeeper-webhook-service:443/v1/admit` in `gatekeeper-system`

Moving on.

# Admission Controller > Policy Engine

Check the service running at the webhook destination.
You can see a service up at `gatekeeper-webhook-service:443` from the previous step:

```
kubectl get svc -n gatekeeper-system
NAME                         TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
gatekeeper-webhook-service   ClusterIP   10.101.28.83   <none>        443/TCP   18h
```

The service's Selector is:

```yaml
# kubectl get svc -n gatekeeper-system -o yaml
apiVersion: v1
items:
...
- spec:
...
    selector:
      control-plane: controller-manager
      gatekeeper.sh/operation: webhook
      gatekeeper.sh/system: "yes"
```

Searching by the matching labels shows three `gatekeeper-controller-manager-*` pods — that is, deployed in HA.
This is itself the OPA Gatekeeper — the Kubernetes-flavored OPA image.

```
kubectl get pods -l control-plane=controller-manager -n gatekeeper-system
NAME                                            READY   STATUS    RESTARTS   AGE
gatekeeper-controller-manager-c97765cd6-bgkjx   1/1     Running   0          18h
gatekeeper-controller-manager-c97765cd6-hnvx2   1/1     Running   0          18h
gatekeeper-controller-manager-c97765cd6-znhqk   1/1     Running   0          18h
```

In summary:

 * At the webhook destination, OPA Gatekeeper runs in an HA configuration

Moving on.

# Policy — and the Rego language

Now we finally look inside OPA.
OPA has two resources:

* constrainttemplates : describes, in the **Rego language**, what policy to define for each resource
* constraints : defines how Constraints are imposed based on the constrainttemplates

For details see [the OPA Gatekeeper Github README](https://github.com/open-policy-agent/gatekeeper).

Here we trace how the following error from the Pod launch in [Part 2](../tmc-demanabu-opa-02) was produced:

```
[denied by tmc.cgp.strict] Sharing the host namespace is not allowed: verybad
```

## ConstraintTemplates

The definition behind that error lives in this file:

```
kubectl get constrainttemplate vmware-system-tmc-block-host-namespace-v1
NAME                                        AGE
vmware-system-tmc-block-host-namespace-v1   22h
```

At the time I checked, the contents looked like this (key parts excerpted):

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: vmware-system-tmc-block-host-namespace-v1
spec:
  crd:
    spec:
      names:
        kind: vmware-system-tmc-block-host-namespace-v1
  targets:
  - rego: |-
      package k8spsphostnamespace
      violation[{"msg": msg, "details": {}}] {
          input_share_hostnamespace(input.review.object)
          msg := sprintf("Sharing the host namespace is not allowed: %v", [input.review.object.metadata.name])
      }
      input_share_hostnamespace(o) {
          o.spec.hostPID
      }
      input_share_hostnamespace(o) {
          o.spec.hostIPC
      }
    target: admission.k8s.gatekeeper.sh
```

The key part is:

```
- rego: |-
    package k8spsphostnamespace
    violation[{"msg": msg, "details": {}}] {
        input_share_hostnamespace(input.review.object)
        msg := sprintf("Sharing the host namespace is not allowed: %v", [input.review.object.metadata.name])
    }
    input_share_hostnamespace(o) {
        o.spec.hostPID
    }
    input_share_hostnamespace(o) {
        o.spec.hostIPC
    }
```

The language used here is called Rego.
To understand it one level deeper, use the Rego online viewer:

https://play.openpolicyagent.org/p/EWQB9RPi3K

The link above has, as Input, the `verybad.yaml` from [Part 2](../tmc-demanabu-opa-02) converted through the AdmissionController.

The code pane contains the code included here.

Toggle `"hostPID"` in the Input between True and False and watch the Evaluate result.

When True, the Output shows:

```
{
    "violation": [
        {
            "details": {},
            "msg": "Sharing the host namespace is not allowed: verybad"
        }
    ]
}
```

When False, the Output shows:

```
{
    "violation": []
}
```

As the results show, a value goes into `"violation"` when `"hostPID": "true"`.
Within it, `msg` matches the message we actually received.

So this Rego language is how TMC's Security Policies are defined.

# Constraint

One more thing to check: the Constraint. It defines when the rule written in Rego takes effect.

It feels odd, but do a `kubectl get` on the ConstraintTemplate name.
You'll see this:

```
kubectl get vmware-system-tmc-block-host-namespace-v1
NAME             AGE
tmc.cgp.strict   23h
```

At the time I checked, the contents looked like this (key parts excerpted):

```yaml
apiVersion: v1
items:
- apiVersion: constraints.gatekeeper.sh/v1beta1
  kind: vmware-system-tmc-block-host-namespace-v1
  metadata:
    name: tmc.cgp.strict
  spec:
    match:
      kinds:
      - apiGroups:
        - ""
        kinds:
        - Pod
      namespaceSelector:
        matchExpressions:
        - key: e2e-run
          operator: DoesNotExist
```

`spec.match` describes when this Constraint applies. This rule covers all Pod operations.

In summary:

* TMC automatically creates the ConstraintTemplates and Constraints, operating transparently to the user

# How Policy Insights works

Last time I introduced Policy Insights, the impressive feature listing rule violations.

![](2020-09-30T14-05-52.png)

The trick behind it is OPA's Audit feature:

https://github.com/open-policy-agent/gatekeeper#audit

You can confirm this from the CLI too by inspecting the Constraint.
For example, when it looks like this:

```yaml
status:
  totalViolations: 1
  violations:
  - enforcementAction: dryrun
    kind: Pod
    message: 'Sharing the host namespace is not allowed: verybad'
    name: verybad
    namespace: default
```

The PolicyInsight side shows:

![](2020-10-01T13-55-11.png)

So this feature, too, is built on OPA.

# Summary

* TMC's Security Policies use AdmissionController, OPA, and the ConstraintTemplate/Constraints mechanism
* ConstraintTemplates can be written in a language called Rego
* OPA is also used for the Policy Insights feature

This was planned as a three-part series, but a few days after writing this article, new features were announced. That content is covered in [Part 4](../tmc-demanabu-opa-04).
