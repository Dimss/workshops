## Hello World K8S Operator tutorial with Operator SDK, OpenShift and Ansible

This guide walks through an example of building a simple hw-operator using the `operator-sdk` CLI tool.

### Prerequisites 
1. [Python3](https://www.python.org/)
2. [Pipenv](https://pipenv.pypa.io/en/latest/#install-pipenv-today) 
3. [Operator SDK CLI v0.13.0](https://github.com/operator-framework/operator-sdk/releases/tag/v0.13.0)
4. [OpenShift client (oc command)](https://mirror.openshift.com/pub/openshift-v4/clients/ocp/latest/)
5. OpenShift cluster
6. Optional - IDE with Golang support ([VSCode](https://code.visualstudio.com/download), [GoLand](https://www.jetbrains.com/go), etc..)


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
Let’s start from creating development environment for Ansible Operator 

```bash
# Create environment for K8S Operators development
mkdir -p my-operators && cd my-operators
operator-sdk new hw-operator --api-version=<YOUR-NAME>.okto.io/v1alpha1 --kind=<YOUR-NAME>HelloWorld --type=ansible
cd hw-operator
# Create environment for Ansible development 
pipenv --python 3
pipenv install ansible ansible-runner openshift ansible-runner-http
```

Update you Ansible code as following 

1.Create Ansible Playbook file `hw.yaml` at root directory of your project
```yaml
- hosts: localhost
  gather_facts: no
  tasks:
  - import_role:
      name: <YOUR-NAME>helloworld
```

2.Update `watches.yaml` to use playbook `hw.yaml` file
```yaml
- version: v1alpha1
  group: <YOUR-NAME>.okto.io
  kind: <YOUR-NAME>HelloWorld
  playbook: /opt/ansible/roles/hw.yaml
``` 

3.Create `watches-dev.yaml` dev file
```yaml
- version: v1alpha1
  group: <YOUR-NAME>.okto.io
  kind: <YOUR-NAME>HelloWorld
  playbook: /path/to/your/hw.yaml
```

4.Update Ansible `defaults` file under `roles/<YOUR-NAME>helloworld/defaults/main.yml` to the following
```yaml
nginx_image: docker.io/dimssss/nginx-for-ocp:0.1
state: present # present or absent
``` 

5.Add following file into `roles/<YOUR-NAME>helloworld/templates/nginx-deployment.yaml` directory
```yaml
kind: Deployment
apiVersion: apps/v1
metadata:
  labels:
    app: "nginx"
  name: "nginx"
  namespace: "{{ meta.namespace }}"
spec:
  replicas: 1
  selector:
    matchLabels:
      app: "nginx"
  template:
    metadata:
      labels:
        app: "nginx"
    spec:
      containers:
        - name: "nginx"
          image: "{{nginx_image}}"
          imagePullPolicy: Always
          ports:
            - containerPort: 8080
```

6.Update `roles/<YOUR-NAME>helloworld/tasks/main.yml`
```yaml
- name: Deploy Nginx instance
  k8s:
    state: "{{ state }}"
    definition: "{{ lookup('template', 'templates/nginx-deployment.yaml') }}"
```

Now when everything in place, let's test our playbook with Ansible (without OperatorSDK at all)
```bash
pipenv run ansible-playbook hw.yaml -e '{"meta":{"namespace":"<YOUR-NAMESPACE>"}}'
```
If everything went well, you should see single Nginx pod running. 
Before continuing with OperatorSDK cleanup your deployment
```bash
pipenv run ansible-playbook hw.yaml -e '{"meta":{"namespace":"hwop"},"state":"absent"}'
```
 
Now, let's run the Ansible playbook as K8S Operator
1.Create `CRD`
```bash
oc create -f deploy/crds/<YOUR-NAME>.okto.io_<YOUR-NAME>helloworlds_crd.yaml
```
2.Start the Operator locally
```bash
operator-sdk up local --namespace=hwop --watches-file=watches-dev.yaml
```
3.Create `CR`
```bash
oc create -f deploy/crds/<YOUR-NAME>.okto.io_v1alpha1_<YOUR-NAME>helloworld_cr.yaml
```

If everything went well, you should see single Nginx pod running. 
To cleanup your deployment, remove the `CR`
```bash
oc delete -f deploy/crds/<YOUR-NAME>.okto.io_v1alpha1_<YOUR-NAME>helloworld_cr.yaml
```