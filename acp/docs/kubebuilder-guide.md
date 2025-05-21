# Kubebuilder Development Guide

## Overview

ACP is built using [Kubebuilder](https://book.kubebuilder.io/), a framework for building Kubernetes APIs using custom resource definitions (CRDs). This guide explains how to use Kubebuilder in this project, particularly for adding new resources and maintaining existing ones.

## Current Project Structure

The project uses Kubebuilder v4 with a domain of `humanlayer.dev` and an API group of `acp`. All resources are in the `v1alpha1` version and are namespaced.

Current resources include:
- `LLM` - Configuration for large language models
- `Agent` - Defines an agent using an LLM and tools
- `Task` - Represents a run of a task
- `ToolCall` - Represents a tool call during a task run
- `MCPServer` - Defines a Model Control Protocol server for tool integration
- `ContactChannel` - Defines communication channels for human interaction

## Adding a New Resource

When adding new Kubernetes resources, always follow these steps:

1. Create the new resource using kubebuilder:

```bash
kubebuilder create api --group acp --version v1alpha1 --kind YourNewResource --namespaced true --resource true --controller true
```

This will:
- Create a new file in `api/v1alpha1/yournewresource_types.go`
- Create a new controller in `internal/controller/yournewresource/`
- Update the PROJECT file with the new resource

2. Define your resource fields in the `*Spec` and `*Status` structs in the generated `_types.go` file.

3. Add RBAC annotations to the controller in `internal/controller/yournewresource/yournewresource_controller.go`:

```go
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=yournewresources,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=yournewresources/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=yournewresources/finalizers,verbs=update
```

Add additional RBAC annotations for any other resources your controller needs to access:

```go
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=someotherresource,verbs=get;list;watch
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch
```

4. Add kubebuilder printing column annotations to your resource's struct:

```go
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.status"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
```

5. Generate manifests to create the CRD and update RBAC:

```bash
make manifests
```

6. Generate DeepCopy methods:

```bash
make generate
```

## State Management in Controllers

Controllers should follow the state machine pattern with clearly defined Status and Phase fields:

- **Status** represents the overall health or readiness of a resource (e.g., Ready, Error, Pending)
- **Phase** represents the specific stage in a resource's lifecycle (e.g., Running, Succeeded, Failed)

When implementing a controller, structure your reconciliation logic around state transitions:

```go
func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    resource := &myresourcev1alpha1.MyResource{}
    if err := r.Get(ctx, req.NamespacedName, resource); err != nil {
        // Handle not found
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Early return for terminal states
    if resource.Status.Phase == myresourcev1alpha1.PhaseSucceeded ||
       resource.Status.Phase == myresourcev1alpha1.PhaseFailed {
        return ctrl.Result{}, nil
    }

    // Handle different states
    switch {
    case resource.Status.Status == "":
        // Initialize status
        return r.initializeStatus(ctx, resource)
    case resource.Status.Status == myresourcev1alpha1.StatusPending:
        // Handle pending state
        return r.handlePendingState(ctx, resource)
    case resource.Status.Status == myresourcev1alpha1.StatusReady && 
         resource.Status.Phase == myresourcev1alpha1.PhaseRunning:
        // Handle running state
        return r.handleRunningState(ctx, resource)
    case resource.Status.Status == myresourcev1alpha1.StatusError:
        // Handle error state
        return r.handleErrorState(ctx, resource)
    }

    return ctrl.Result{}, nil
}
```

## Example: Adding a ContactChannel Resource

Here's an example of creating a ContactChannel resource:

```bash
kubebuilder create api --group acp --version v1alpha1 --kind ContactChannel --namespaced true --resource true --controller true
```

Then, edit the generated `api/v1alpha1/contactchannel_types.go` file:

```go
type ContactChannelSpec struct {
    // Email configuration for the contact channel
    // +optional
    Email *EmailContactChannel `json:"email,omitempty"`

    // Slack configuration for the contact channel
    // +optional
    Slack *SlackContactChannel `json:"slack,omitempty"`

    // Additional configuration options common to all channel types
    // +optional
    Options *ContactOptions `json:"options,omitempty"`
}

type ContactChannelStatus struct {
    // Status indicates the current status of the channel
    // +kubebuilder:validation:Enum=Ready;Error;Pending
    Status string `json:"status,omitempty"`

    // Phase indicates the current phase in the channel's lifecycle
    // +kubebuilder:validation:Enum=Pending;Running;Failed;Succeeded
    Phase string `json:"phase,omitempty"`

    // StatusDetail provides additional details about the current status
    StatusDetail string `json:"statusDetail,omitempty"`

    // Ready indicates if the channel is ready to receive notifications
    Ready bool `json:"ready,omitempty"`
}
```

Edit the controller to add RBAC annotations:

```go
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=contactchannels,verbs=get;list;watch;create;update;patch;delete
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=contactchannels/status,verbs=get;update;patch
// +kubebuilder:rbac:groups=acp.humanlayer.dev,resources=contactchannels/finalizers,verbs=update
// +kubebuilder:rbac:groups="",resources=secrets,verbs=get;list;watch

// ContactChannelReconciler reconciles a ContactChannel object
type ContactChannelReconciler struct {
    // ...
}
```

Add printing columns:

```go
// +kubebuilder:object:root=true
// +kubebuilder:subresource:status
// +kubebuilder:printcolumn:name="Status",type="string",JSONPath=".status.status"
// +kubebuilder:printcolumn:name="Phase",type="string",JSONPath=".status.phase"
// +kubebuilder:printcolumn:name="Ready",type="boolean",JSONPath=".status.ready"
// +kubebuilder:printcolumn:name="Age",type="date",JSONPath=".metadata.creationTimestamp"
// +kubebuilder:resource:scope=Namespaced
```

After making these changes, run:

```bash
make generate
make manifests
```

## Common Pitfalls and Solutions

### 1. Missing or Incorrect RBAC Annotations

**Problem**: The controller can't access resources it needs because RBAC permissions are missing.

**Solution**: Make sure to add proper RBAC annotations to your controller before running `make manifests`. Remember to include permissions for any resources your controller accesses, not just the one it's primarily responsible for.

### 2. Forgetting to Generate Code After Adding Fields

**Problem**: After adding new fields, the code won't compile due to missing DeepCopy methods.

**Solution**: Always run `make generate` after adding or modifying struct fields:

```bash
make generate
```

### 3. CRD Validation Issues

**Problem**: The CRD validation schema doesn't match your expectations.

**Solution**: Use kubebuilder validation annotations in your type definitions:

```go
// +kubebuilder:validation:Minimum=0
// +kubebuilder:validation:Maximum=100
// +kubebuilder:validation:Enum=option1;option2;option3
// +kubebuilder:validation:Required
```

### 4. Implementing Status Updates

**Problem**: Status updates fail with conflict errors.

**Solution**: Always use the Status() client for status updates and handle conflicts properly:

```go
if err := r.Status().Update(ctx, resource); err != nil {
    if apierrors.IsConflict(err) {
        // Handle conflict - typically by returning and requeueing
        return ctrl.Result{Requeue: true}, nil
    }
    return ctrl.Result{}, err
}
```

### 5. Using Finalizers

**Problem**: Resources aren't properly cleaned up when deleted.

**Solution**: Implement finalizers for proper cleanup:

```go
const myFinalizerName = "myresource.finalizers.acp.humanlayer.dev"

func (r *MyReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // Get the resource
    resource := &myresourcev1alpha1.MyResource{}
    if err := r.Get(ctx, req.NamespacedName, resource); err != nil {
        return ctrl.Result{}, client.IgnoreNotFound(err)
    }

    // Check if the resource is being deleted
    if !resource.ObjectMeta.DeletionTimestamp.IsZero() {
        // Resource is being deleted
        if containsString(resource.ObjectMeta.Finalizers, myFinalizerName) {
            // Run cleanup logic here
            
            // Remove finalizer
            resource.ObjectMeta.Finalizers = removeString(resource.ObjectMeta.Finalizers, myFinalizerName)
            if err := r.Update(ctx, resource); err != nil {
                return ctrl.Result{}, err
            }
        }
        return ctrl.Result{}, nil
    }

    // Add finalizer if not present
    if !containsString(resource.ObjectMeta.Finalizers, myFinalizerName) {
        resource.ObjectMeta.Finalizers = append(resource.ObjectMeta.Finalizers, myFinalizerName)
        if err := r.Update(ctx, resource); err != nil {
            return ctrl.Result{}, err
        }
    }

    // Normal reconciliation logic...
}
```

## Best Practices

1. **Use nil-able sub-objects over "type" fields**: Instead of using a type field to determine which configuration to use, use nil-able struct pointers and check which one is non-nil:

```go
// Preferred approach
type MySpec struct {
    // +optional
    ConfigA *ConfigA `json:"configA,omitempty"`
    
    // +optional
    ConfigB *ConfigB `json:"configB,omitempty"`
}

// In code, check which is non-nil
if resource.Spec.ConfigA != nil {
    // Use ConfigA
} else if resource.Spec.ConfigB != nil {
    // Use ConfigB
}
```

2. **Add detailed validation**: Use validation annotations to catch errors early:

```go
// +kubebuilder:validation:Required
// +kubebuilder:validation:MinLength=1
// +kubebuilder:validation:MaxLength=63
// +kubebuilder:validation:Pattern=^[a-z0-9]([-a-z0-9]*[a-z0-9])?$
```

3. **Implement proper error handling**: Set appropriate status and record events:

```go
if err != nil {
    resource.Status.Status = myresourcev1alpha1.StatusError
    resource.Status.StatusDetail = fmt.Sprintf("Failed to process: %v", err)
    r.Recorder.Event(resource, corev1.EventTypeWarning, "ProcessingFailed", err.Error())
    return ctrl.Result{}, r.Status().Update(ctx, resource)
}
```

4. **Use events for important state changes**: Record events to make debugging easier:

```go
r.Recorder.Event(resource, corev1.EventTypeNormal, "Created", "Created dependent resource")
```

5. **Follow Kubernetes API conventions**: Use consistent naming and structure following [Kubernetes API conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md).

6. **Implement proper status conditions**: Use the standard Kubernetes conditions pattern for complex status reporting:

```go
type MyResourceStatus struct {
    // Conditions represents the latest available observations of the resource's state
    Conditions []metav1.Condition `json:"conditions,omitempty"`
    
    // Other status fields...
}
```

7. **Write thorough tests**: Test all state transitions and error paths, not just happy paths.

## Additional Resources

- [Kubebuilder Book](https://book.kubebuilder.io/)
- [Kubernetes API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md)
- [Controller Runtime](https://github.com/kubernetes-sigs/controller-runtime)
- [Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)