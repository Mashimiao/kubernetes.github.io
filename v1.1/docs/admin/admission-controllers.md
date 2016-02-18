---
title: "Admission Controllers"
---
* TOC
{:toc}

## What are they?

An admission control plug-in is a piece of code that intercepts requests to the Kubernetes
API server prior to persistence of the object, but after the request is authenticated
and authorized.  The plug-in code is in the API server process
and must be compiled into the binary in order to be used at this time.

Each admission control plug-in is run in sequence before a request is accepted into the cluster.  If
any of the plug-ins in the sequence reject the request, the entire request is rejected immediately
and an error is returned to the end-user.

Admission control plug-ins may mutate the incoming object in some cases to apply system configured
defaults.  In addition, admission control plug-ins may mutate related resources as part of request
processing to do things like increment quota usage.

## Why do I need them?

Many advanced features in Kubernetes require an admission control plug-in to be enabled in order
to properly support the feature.  As a result, a Kubernetes API server that is not properly
configured with the right set of admission control plug-ins is an incomplete server and will not
support all the features you expect.

## How do I turn on an admission control plug-in?

The Kubernetes API server supports a flag, `admission-control` that takes a comma-delimited,
ordered list of admission control choices to invoke prior to modifying objects in the cluster.

## What does each plug-in do?

### AlwaysAdmit

Use this plugin by itself to pass-through all requests.

### AlwaysDeny

Rejects all requests.  Used for testing.

### DenyExecOnPrivileged (deprecated)

This plug-in will intercept all requests to exec a command in a pod if that pod has a privileged container.

If your cluster supports privileged containers, and you want to restrict the ability of end-users to exec
commands in those containers, we strongly encourage enabling this plug-in.

This functionality has been merged into [DenyEscalatingExec](#denyescalatingexec).

### DenyEscalatingExec

This plug-in will deny exec and attach commands to pods that run with escalated privileges that
allow host access.  This includes pods that run as privileged, have access to the host IPC namespace, and
have access to the host PID namespace.

If your cluster supports containers that run with escalated privileges, and you want to
restrict the ability of end-users to exec commands in those containers, we strongly encourage
enabling this plug-in.

### ServiceAccount

This plug-in implements automation for [serviceAccounts](/{{page.version}}/docs/user-guide/service-accounts).
We strongly recommend using this plug-in if you intend to make use of Kubernetes `ServiceAccount` objects.

### SecurityContextDeny

This plug-in will deny any pod with a [SecurityContext](/{{page.version}}/docs/user-guide/security-context) that defines options that were not available on the `Container`.

### ResourceQuota

This plug-in will observe the incoming request and ensure that it does not violate any of the constraints
enumerated in the `ResourceQuota` object in a `Namespace`.  If you are using `ResourceQuota`
objects in your Kubernetes deployment, you MUST use this plug-in to enforce quota constraints.

See the [resourceQuota design doc](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/admission_control_resource_quota.md) and the [example of Resource Quota](resourcequota/) for more details.

It is strongly encouraged that this plug-in is configured last in the sequence of admission control plug-ins.  This is
so that quota is not prematurely incremented only for the request to be rejected later in admission control.

### LimitRanger

This plug-in will observe the incoming request and ensure that it does not violate any of the constraints
enumerated in the `LimitRange` object in a `Namespace`.  If you are using `LimitRange` objects in
your Kubernetes deployment, you MUST use this plug-in to enforce those constraints. LimitRanger can also
be used to apply default resource requests to Pods that don't specify any; currently, the default LimitRanger
applies a 0.1 CPU requirement to all Pods in the `default` namespace.

See the [limitRange design doc](https://github.com/kubernetes/kubernetes/blob/release-1.1/docs/design/admission_control_limit_range.md) and the [example of Limit Range](limitrange/) for more details.

### InitialResources (experimental)

This plug-in observes pod creation requests. If a container omits compute resource requests and limits,
then the plug-in auto-populates a compute resource request based on historical usage of containers running the same image.
If there is not enough data to make a decision the Request is left unchanged.
When the plug-in sets a compute resource request, it annotates the pod with information on what compute resources it auto-populated.

See the [InitialResouces proposal](../proposals/initial-resources) for more details.

### NamespaceExists (deprecated)

This plug-in will observe all incoming requests that attempt to create a resource in a Kubernetes `Namespace`
and reject the request if the `Namespace` was not previously created.  We strongly recommend running
this plug-in to ensure integrity of your data.

The functionality of this admission controller has been merged into `NamespaceLifecycle`

### NamespaceAutoProvision (deprecated)

This plug-in will observe all incoming requests that attempt to create a resource in a Kubernetes `Namespace`
and create a new `Namespace` if one did not already exist previously.

We strongly recommend `NamespaceLifecycle` over `NamespaceAutoProvision`.

### NamespaceLifecycle

This plug-in enforces that a `Namespace` that is undergoing termination cannot have new objects created in it,
and ensures that requests in a non-existant `Namespace` are rejected.

A `Namespace` deletion kicks off a sequence of operations that remove all objects (pods, services, etc.) in that
namespace.  In order to enforce integrity of that process, we strongly recommend running this plug-in.

## Is there a recommended set of plug-ins to use?

Yes.

For Kubernetes 1.0, we strongly recommend running the following set of admission control plug-ins (order matters):

```shell
--admission-control=NamespaceLifecycle,LimitRanger,SecurityContextDeny,ServiceAccount,ResourceQuota
```