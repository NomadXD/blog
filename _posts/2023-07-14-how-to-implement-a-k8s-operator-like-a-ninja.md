---
layout: post
current: post
cover: assets/images/k8s-operator-kube-builder.jpg
navigation: True
title: "How to implement a K8s operator like a Ninja"
date: 2023-07-14 10:18:00
tags: tech
class: post-template
subclass: "post"
logo: assets/images/ghost.png
author: lahiru
---

Kubernetes has emerged as the widely accepted standard for deploying and running modern cloud-native applications. With its declarative approach using YAML, it offers a straightforward and intuitive method to define the desired infrastructure state for application deployments. However, managing complex application deployments on Kubernetes requires additional automation and control beyond what Kubernetes provides out of the box. This is where Kubernetes operators come in.

Operators extend the capabilities of Kubernetes by encapsulating domain-specific knowledge and best practices into custom controllers. By leveraging operators, organizations can automate the management of complex application lifecycles, enable self-healing, autoscaling, and perform advanced operations such as backups, upgrades, and rolling deployments. Kubernetes operators help streamline and simplify the deployment and management of applications on Kubernetes, enhancing scalability, reliability, and efficiency.

There are several approaches to building a Kubernetes operator. One option is to develop one from scratch using the [Kubernetes Go Client](https://github.com/kubernetes/client-go), but this can be a challenging and time-consuming task due to the steep learning curve involved. Alternatively, there are tools available that provide boilerplate code and streamline the process of creating operators. Two popular choices are [OperatorSDK](https://sdk.operatorframework.io/) and [Kubebuilder](https://github.com/kubernetes-sigs/kubebuilder). In this article, we will primarily focus on utilizing Kubebuilder to create an operator, leveraging its features to expedite the development process.

> **Prerequisites**:
>
> - go version v1.20.0+
> - kubectl version v1.11.3+.
> - Access to a Kubernetes v1.11.3+ cluster.

Kubebuilder offers a command-line interface (CLI) that facilitates the creation and management of operator projects. To initiate a new project, all you need to do is execute the following command:

```sh
kubebuilder init --domain nomadxd.io --plugins=go/v4-alpha --repo=github.com/NomadXD/samples/k8s-operator-kube-builder
```

Once executed, Kubebuilder will generate a directory structure, enabling you to immediately begin developing the operator.

To enable the desired functionality of an operator, Custom Resource Definitions (CRDs) are utilized in conjunction with a controller. By defining CRDs, you can extend the Kubernetes API and introduce custom resources tailored to your specific application or workload. These CRDs serve as the basis for creating and managing instances of your custom resources.

To ensure the desired state of the custom resources is maintained, a controller is implemented. The controller continuously monitors the state of the custom resources and takes actions to reconcile any differences between the desired state and the actual state of the cluster. This may involve creating, updating, or deleting Kubernetes resources based on changes in the custom resources.

Use the following command to create a new Go type which corresponds to a CRD in K8s. It will prompt for 2 options to create resource and controller and input y to both of the cases.

```sh
kubebuilder create api --group podrunner --version v1alpha1 --kind PodRunner
Create Resource [y/n]
y
Create Controller [y/n]
y
```

Now all the boilerplate required to implement the operator is generated. The core components of an operator are found in the following locations:

- **/api/v1alpha1/podrunner_types.go**

Contains the Custom Resource type definition in Go. Spec and Status types should be extended by adding the required feilds. Spec means the desired state and the Status means the current state. A controller basically runs a reconcile loop to match the current state with the desired state.

```go
/*
Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package v1alpha1

import (
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
)

// EDIT THIS FILE!  THIS IS SCAFFOLDING FOR YOU TO OWN!
// NOTE: json tags are required.  Any new fields you add must have json tags for the fields to be serialized.

// PodRunnerSpec defines the desired state of PodRunner
type PodRunnerSpec struct {
	// INSERT ADDITIONAL SPEC FIELDS - desired state of cluster
	// Important: Run "make" to regenerate code after modifying this file

	// Foo is an example field of PodRunner. Edit podrunner_types.go to remove/update
	Foo string `json:"foo,omitempty"`
}

// PodRunnerStatus defines the observed state of PodRunner
type PodRunnerStatus struct {
	// INSERT ADDITIONAL STATUS FIELD - define observed state of cluster
	// Important: Run "make" to regenerate code after modifying this file
}

//+kubebuilder:object:root=true
//+kubebuilder:subresource:status

// PodRunner is the Schema for the podrunners API
type PodRunner struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   PodRunnerSpec   `json:"spec,omitempty"`
	Status PodRunnerStatus `json:"status,omitempty"`
}

//+kubebuilder:object:root=true

// PodRunnerList contains a list of PodRunner
type PodRunnerList struct {
	metav1.TypeMeta `json:",inline"`
	metav1.ListMeta `json:"metadata,omitempty"`
	Items           []PodRunner `json:"items"`
}

func init() {
	SchemeBuilder.Register(&PodRunner{}, &PodRunnerList{})
}
```

- **/controllers/podrunner_controller.go**

Extend the `Reconcile()` method to implement reconcile logic to match the current state with the desired state.

```go
/*
Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"

	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/log"

	podrunnerv1alpha1 "github.com/NomadXD/samples/k8s-operator-kube-builder/api/v1alpha1"
)

// PodRunnerReconciler reconciles a PodRunner object
type PodRunnerReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=podrunner.nomadxd.io,resources=podrunners,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=podrunner.nomadxd.io,resources=podrunners/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=podrunner.nomadxd.io,resources=podrunners/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the PodRunner object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.13.0/pkg/reconcile
func (r *PodRunnerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	_ = log.FromContext(ctx)

	// TODO(user): your logic here

	return ctrl.Result{}, nil
}

// SetupWithManager sets up the controller with the Manager.
func (r *PodRunnerReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&podrunnerv1alpha1.PodRunner{}).
		Complete(r)
}
```

For this tutorial, I will implement an operator which can run a Pod based on an image name we provide with the Custom Resource. To implement that, I have extended the `PodRunnerSpec` as follows.

```go
// PodRunnerSpec defines the desired state of PodRunner
type PodRunnerSpec struct {
	// PodName is the name of the pod.
	PodName string `json:"podName,omitempty"`

	// ImageName is the name of the image used to run the pod.
	ImageName string `json:"imageName,omitempty"`

        // Namespace where the pod is scheduled to run.
	Namespace string `json:"namespace,omitempty"`
}
```

After that execute the following commands.

1. `make generate` - This generates the required deep copy methods
2. `make manifests` - To generate the Kubernetes CRDs from the defined Go types. Generated CRDs are located in `config/crd/bases/`
3. `make install` - To apply the CRDs to the K8s API server

Now the Kubernetes API server knows about the resources of type `PodRunner`.

Now to implement the logic for scheduling the pod on the given namespace using the provided image name, extend the `Reconcile()` method in the controller boilerplate as follows.

```go
func (r *PodRunnerReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	logger := log.FromContext(ctx)

	podRunner := &podrunnerv1alpha1.PodRunner{}
	err := r.Get(ctx, req.NamespacedName, podRunner)
	if err != nil {
		// Error reading the PodRunner instance, requeue the request
		logger.Error(err, "Failed to get PodRunner")
		return ctrl.Result{}, client.IgnoreNotFound(err)
	}

	// Create a Pod based on the PodRunner specification
	podRunnerPod := &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			Name:      podRunner.Spec.PodName,
			Namespace: podRunner.Spec.Namespace,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:            podRunner.Spec.PodName,
					Image:           podRunner.Spec.ImageName,
					ImagePullPolicy: corev1.PullAlways,
				},
			},
		},
	}

	err = ctrl.SetControllerReference(podRunner, podRunnerPod, r.Scheme)
	if err != nil {
		logger.Error(err, "Failed to set controller reference for Nginx Pod")
		return ctrl.Result{}, err
	}

	// Check if the Pod already exists
	foundPod := &corev1.Pod{}
	err = r.Get(ctx, types.NamespacedName{Name: podRunner.Spec.PodName, Namespace: podRunner.Spec.Namespace}, foundPod)
	if err != nil && errors.IsNotFound(err) {
		// Create the Pod
		err = r.Create(ctx, podRunnerPod)
		if err != nil {
			logger.Error(err, "Failed to create Pod")
			return ctrl.Result{}, err
		}
		logger.Info("Pod created")
		return ctrl.Result{}, nil
	} else if err != nil {
		logger.Error(err, "Failed to get Pod")
		return ctrl.Result{}, err
	}

	// Pod already exists, do nothing
	logger.Info("Pod already exists")
	return ctrl.Result{}, nil
}
```

Now execute the command `make run` to run the operator locally. If you have followed everything correctly upto now, you should have a Kubernetes operator up and running. Now to run a pod, create a Custom Resource of type `PodRunner` and do a `kubectl apply -f`.

```yaml
kubectl apply -f - <<EOF
apiVersion: podrunner.nomadxd.io/v1alpha1
kind: PodRunner
metadata:
  name: podrunner-sample
spec:
  podName: test-pod
  imageName: nginx:latest
  namespace: default
EOF
```

If everything is in place, you will see `podrunner.podrunner.nomadxd.io/podrunner-sample created`. Next execute `kubectl get pods` to see whether the pod is scheduled by the controller we implemented.

```sh
kubectl get pods
NAME       READY   STATUS    RESTARTS   AGE
test-pod   1/1     Running   0          6s
```

If you execute `kubectl describe pod test-pod | grep -i "Controlled By"`, you will get `Controlled By:  PodRunner/podrunner-sample`. So it proves that the pod is scheduled by the controller we implemented.

By leveraging Kubebuilder's user-friendly CLI and code generation capabilities, developers can rapidly bring their Kubernetes operators to life within a matter of minutes. With just a few simple commands and a bit of code, complex processes and application management can be streamlined and automated without the need for complex time consuming implementation.

**Complete source code for the above setup with Kubebuilder generated files:** [k8s-operator-kube-builder](https://github.com/NomadXD/samples/tree/main/k8s-operator-kube-builder)
