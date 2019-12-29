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

Update the HelloWorldStatus struct to

```go
type HelloWorldStatus struct {
 Message string `json:"message"`
}
```

After modifying the `*_types.go` file always run the following command to update the generated code for that resource type

```bash
operator-sdk generate k8s
```