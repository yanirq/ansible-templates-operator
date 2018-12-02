# ansible-templates-operator
ansible-templates-operator is an ansible-operator generated from the [operator-sdk](https://github.com/operator-framework/operator-sdk).
The purpose of this operator is to watch for changes of [Openshift templates](https://docs.openshift.com/container-platform/3.11/dev_guide/templates.html) resources on a cluster managed by [KubeVirt](http://kubevirt.io/)
and update all VMs which were created by this template.

## Before getting started
It is recommend to read the [ansible operator-sdk user guide](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/dev/developer_guide.md) first, to comply with all prerequisites
and get a fundamental knowledge about how to create an ansible operator and run it locally or on a cluster.

## Creation of ansible-templates-operator
The ansible-templates-operator was generated to watch specifically  [VM templates](https://github.com/kubevirt/common-templates/tree/master/templates) in the form compatible with [OpenShift templates](https://docs.okd.io/latest/dev_guide/templates.html)
following the operator-sdk [guidelines](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md#create-a-new-project):

```
operator-sdk new templates-ansible-operator --api-version=template.openshift.io/v1  --kind=Template --type=ansible --generate-playbook
```

## Running the Ansible operator on a Openshift/KubeVirt cluster

This operator can be deployed and run on/alongside an Openshift/KubeVirt cluster.
The operator will watch for changes (CREATE/UPDATE) of the VM templates (Openshift tamplates).
When a change occurs, the `./watch.yaml` is consumed by the operator and consequently invokes the `./playbook.yaml` .

### Running an Ansible operator locally
Following [Testing an Ansible operator locally](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/dev/developer_guide.md) guidelines:

Set the `./watch.yaml` file to fetch the playbook from a local path :
```yaml
---
- version: v1
  group: template.openshift.io
  kind: Template
  playbook: /home/usr/templates-ansible-operator/playbook.yaml
```

Create the deployment files:

```
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
```

Run the operator-sdk locally with the Openshift/KubeVirt cluster configuration:
```
operator-sdk up local --kubeconfig="/home/usr/go/src/kubevirt.io/kubevirt/cluster/os-3.11.0/.kubeconfig"

INFO[0000] Go Version: go1.10.3                        
INFO[0000] Go OS/Arch: linux/amd64                     
INFO[0000] operator-sdk Version: v0.1.1+git            
INFO[0000] watching namespace: default                 
INFO[0000] Starting to serve on 127.0.0.1:8888         
INFO[0000] Watching template.openshift.io/v1, Template

```

Create/Update a VM template (Openshift template) CRD :

When creating a template it is important that it has the ansible.operator-sdk/reconcile-period annotation set to 0.

example file : **rhel7-generic-tiny.yaml**
```yaml
apiVersion: v1
kind: Template
metadata:
  name: rhel7-generic-tiny
  annotations:
    ansible.operator-sdk/reconcile-period: "0s"
...
objects:
- apiVersion: kubevirt.io/v1alpha2
  kind: VirtualMachine
  metadata:
    name: ${NAME}
    labels:
      vm.cnv.io/template: rhel7-generic-tiny
...
parameters:
- description: VM name
  from: 'rhel7-[a-z0-9]{16}'
  generate: expression
  name: NAME
- name: PVCNAME
  description: Name of the PVC with the disk image
  required: true
```

Create the template:
```
$ kubectl create -f ../common-templates/dist/templates/rhel7-generic-tiny.yaml
```

Since the operator-sdk runs locally and uses ansible runner, the output of the playbook can be checked
in the runner's local directory, e.g.:
`/tmp/ansible-operator/runner/template.openshift.io/v1/Template/default/<template name>/artifacts/<change event directory>`.
Check status and stdout files to see if the playbook has ran successfully and its output.


### Running an Ansible operator on a cluster

Find out what port is forwarded to kubevirt's registry:
```
$ docker ps | grep kubevirt

IMAGE                          PORTS       
kubevirtci/os-3.11...          ...,0.0.0.0:32770->5000/tcp,...

```

To build the ansible-templates-operator image and push it to the Openshift/KubeVirt registry:
```
$ operator-sdk build localhost:32770/templates-ansible-operator:v0.0.1
$ docker push localhost:32770/templates-ansible-operator:v0.0.1
```

Create the deployment files:

```
$ kubectl create -f deploy/service_account.yaml
$ kubectl create -f deploy/role.yaml
$ kubectl create -f deploy/role_binding.yaml
$ kubectl create -f deploy/operator.yaml
```

Add cluster-admin role to the operator service account:

```
$kubectl create clusterrolebinding root-cluster-admin-binding --clusterrole=cluster-admin --user=system:serviceaccount:default:templates-ansible-operator
```

Verify that the ansible-templates-operator is up and running:

```
$ kubectl get deployments
NAME                         DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
templates-ansible-operator   1         1         1            1           1m

```

## Caveats & Gotchas
- The operator will currently watch all forms of Openshift Templates in the cluster (even if they are not VM templates)
There might be a need to restrict the [operator scope](https://github.com/operator-framework/operator-sdk/blob/master/doc/ansible/user-guide.md#operator-scope) to watch only VM templates.
- Template delete events do not invoke playbook.
