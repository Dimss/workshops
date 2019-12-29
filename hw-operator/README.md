## Hello World K8S Operator tutorial with Operator SDK and OpenShift

This guide walks through an example of building a simple hw-operator using the operator-sdk CLI tool and controller-runtime library API.

### Prerequisites 
1. [Golang](https://golang.org/doc/install) 
2. [Operator SDK CLI v0.13.0](https://github.com/operator-framework/operator-sdk/releases/tag/v0.13.0)
3. [OpenShift client (oc command)](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
4. OpenShift cluster
5. Optional - IDE with Golang support ([VSCode](https://code.visualstudio.com/download), [GoLand](https://www.jetbrains.com/go), etc..)


### Let's start

As for example, we going to implement simple hw-operator which will encode operational logic of deploying a new Nginx web server with single website. The website will be constructed from basic HTML code.
The hw-operator operator will take care of the following K8S/OpenShift objects
* Deployment
* Service
* Router
* ConfigMap with basic HTML code

### The operator diagram

![diagram](hw-operator-diagram.png)

### Project setup 
Let’s start from creating directory and [operator project layout](https://github.com/operator-framework/operator-sdk/blob/master/doc/project_layout.md)

```bash
mkdir -p $GOPATH/src/github.com
cd $GOPATH/src/github.com
operator-sdk new hw-operator --vendor=true
cd hw-operator
```

Now, when we’ve created our empty operator project, let’s generate our first API. 
Generate new Custom Resource Definition(CRD) API called HelloWorld, 
with API version `<YOUR-NAME>.hw.okto.io/v1alpha1` and Kind `HelloWorld`.

```bash
operator-sdk add api --api-version=<YOUR-NAME>.hw.okto.io/v1alpha1 --kind=HelloWorld
```
This will scaffold the HelloWorld resource API under `pkg/apis/<YOUR-NAME>/v1alpha1/...`

Define the `spec` and `status` by modifying the `spec` and `status` go structs of the `HelloWorld` Custom Resource(CR) at `pkg/apis/<YOUR-NAME>/v1alpha1/helloworld_types.go`.

Update the `HelloWorldSpec` struct to

```go
type HelloWorldSpec struct {
 Message string `json:"message"`
}
```

Update the `HelloWorldStatus` struct to

```go
type HelloWorldStatus struct {
 Message string `json:"message"`
}
```

After modifying the `*_types.go` file always run the following command to update the generated code for that resource type

```bash
operator-sdk generate k8s
```

Also, update the CRD file at `deploy/crds/<YOUR-NAME>.hw.okto.io_v1alpha1_helloworld_cr.yaml` by running the following command
```bash
 operator-sdk generate crds
``` 

After each new code generation, or importing new library, do not forget to run the following command
```bash
go mod vendor
```

Add a new `Controller` to the project that will watch and reconcile the `HelloWorld` resource 
```bash
operator-sdk add controller --api-version=<YOUR-NAME>.hw.okto.io/v1alpha1 --kind=HelloWorld
```
This will scaffold a new `Controller` implementation under `pkg/controller/helloworld/helloworld_controller.go`

Since `hw-operator` will use OpenShift Route for exposing the website to external access, we’ll have to register [3rd Party Resources](https://github.com/operator-framework/operator-sdk/blob/master/doc/user-guide.md#adding-3rd-party-resources-to-your-operator) to the `hw-operator`

Edit the `cmd/manager/main.go` and add the following code

1. To the import section add the following
    ```go
    routev1 "github.com/openshift/api/route/v1"
    ```
2. To the main function, add the following, **note**, your 3rd party resource needs to be added above the `// Setup all Controllers` comment, also, don’t forget to run the `go mod vendor` to update vendor directory.

    ```go
    // Adding the routev1
    if err := routev1.AddToScheme(mgr.GetScheme()); err != nil {
       log.Error(err, "")
       os.Exit(1)
    }
    ```