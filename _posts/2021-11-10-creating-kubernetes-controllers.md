---
layout: post
title: "Creating Kubernetes Controllers"
date: 2021-11-10
---

The creation of Kubernetes controllers (or operators) have been simplified
massively by the creation of frameworks like
[kubebuilder](https://book.kubebuilder.io/)
and it's fork
[operator framework](https://operatorframework.io/)


For this demo we'll be using `kubebuilder`


This is a short demo of a controller which monitors service accounts - if it
finds a service account in a namespace called 'it', it will create an object in
that namespace called 'tig' with a value: {'it': $namespace}.


This should demonstrate the capabilities of an operator, the possibilities of
monitoring existing objects and creating new ones depending on some internal
logic opens up the door to automating a lot of operational work wherever
Kubernetes clusters exist.


All of the code for this example should be available here:
[https://github.com/fosuoy/tig-operator](https://github.com/fosuoy/tig-operator)


To start with - we'll either clone an empty repository or create a directory
where we'll be using kubebuilder to scaffold the operator, in my case:

```
$ git clone git@github.com:fosuoy/tig-operator.git
...
$ cd tig-operator/
$ kubebuilder init --domain example.com --repo github.com/fosuoy/tig-operator
Writing kustomize manifests for you to edit...
Writing scaffold for you to edit...
Get controller runtime:
...
```

Now we have the scaffold in place we can get to work

We can start off by creating the Tig object - to do this we have to create an
API using kubebuilder which will scaffold the Tig controller as well as the CRD:

```
kubebuilder create api --group game --version v1 --kind Tig
```

So now we can start working on the CRD - this is straightforward we'll update
the TigSpec in:

`api/v1/tig_types.go`

to:
```
type TigSpec struct {
	It string `json:"it"`
}
```

Now we can update the controller to make sure it's monitoring ServiceAccounts
and not just namespaces - to do this we update:

`controllers/tig_controller.go`

By default the controller is monitoring the new Tig objects we create and would
trigger a Reconciler loop if it sees a new object created anywhere.

To update this we can change the default:

```
func (r *TigReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&gamev1.Tig{}).
		Complete(r)
}
```

to:

```
func (r *TigReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&gamev1.Tig{}).
		Complete(r)
}
```

With this in place the actual Reconciler logic will be very straightforward - 

```
func (r *TigReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	serviceAccount := &v1.ServiceAccount{}

	err := r.Get(ctx, req.NamespacedName, serviceAccount)
	if err != nil {
		return ctrl.Result{}, err
	}

	serviceAccountName := serviceAccount.ObjectMeta.Name

	if strings.ToLower(serviceAccountName) != "it" {
		return ctrl.Result{}, nil
	}

	tig := &gamev1.Tig{}
	tig.Name = "tig"
	tig.Namespace = serviceAccount.Namespace
	tig.Spec.It = serviceAccount.Namespace

	err = r.Get(
		ctx,
		client.ObjectKey{Namespace: serviceAccount.Namespace, Name: "tig"},
		tig)
	if err != nil {
		log.Error(err, "Not found")
		err = r.Create(ctx, tig)
		if err != nil {
			log.Error(err, "Failed to create tig")
		}
		return ctrl.Result{}, nil
	}

	err = r.Update(ctx, tig)
	if err != nil {
		log.Error(err, "Failed to update tig")
	}
	return ctrl.Result{}, nil
}

```

To create this snippet - the documentation for the `k8s.io/api/core/v1`
package is very helpful
[https://pkg.go.dev/k8s.io/api/core/v1](https://pkg.go.dev/k8s.io/api/core/v1)

The logic here is very straightforward  - we're monitoring ServiceAccounts while
ignoring any accounts not called 'it' if we find one called 'it' we'll create or
update a tig object called 'tig' depending on whether it exists or not.

Finally we can test the operator - installing and running minikube then running:

`make install run`

is the most straightforward way to run the operator

Deploying the operator is as straightforward as running:

`make docker-build ; make docker-push`

and writing the YAML for the deployment / service account to run the operator
