# OpenShift Pipelines

OpenShift Pipelines is a CI/CD solution based on Tekton. By leveraging in 
OpenShift, Tekton pipelines are an automated process that drives software through 
a path of building, testing, and deploying code.

In this article we will be performing the following tasks:
- Create custom tasks
- Create a custom pipeline
- Create storage volumes
- Execute PipelineRuns

Discover, search and share reusable Tasks and Pipelines.
> https://hub-preview.tekton.dev

## Assumptions
- Access to the `oc command`
- Access to a user with cluster-admin permissions
- Access to an installed OpenShift Container Platform 4.6 deployment
- Access to an active OpenShift Container Platform 4.6 subscription
- Red Hat Pipeline Operator installed
- Red Hat Compliance Operator installed

## Components
- Tekton Pipelines: v0.16.3
- Tekton Triggers: v0.8.1
- ClusterTasks based on Tekton Catalog 0.16
- Ansible Runner Task: 0.1

The custom resources needed to define a pipeline are listed below:
- `Task`: a reusable, loosely coupled number of `Steps`
- `Pipeline`: the definition of the `Tasks` that it should perform
- `TaskRun`: the execution and result of running an instance of a `Task`
- `PipelineRun`: the execution and result of running an instance of a `Pipeline`
- `Pipeline`: includes a number of `TaskRuns`
  
## Features
- Standardize pipelines definitions
- Build images with Kubernetes tools
- Deploy applications to multiple platforms

## Installation
Export environment variables:
```bash
export NAMESPACE=<your-pipeline-project>
```

Create a new pipeline project:
```bash
oc new-project $NAMESPACE
```

## Compliance Pipeline 
View the `pipeline` service accounts in the current project:
```bash
oc describe sa pipeline
```

Install the OpenShift Compliance operator using the following command:
```bash
oc apply -f openshift/subscription.yaml \
         -l operator=compliance
```

Apply the `Pipeline` and `Task` using the following command:
```bash
oc apply -n $NAMESPACE -f compliance/task.yaml 
```

List the `Task`:
```bash
tkn task ls -n $NAMESPACE
```

Apply the `Pipeline` and `Task` using the following command:
```bash
oc apply -n $NAMESPACE -f compliance/pipeline.yaml
```

List the `Pipeline`:
```bash
tkn pipeline ls -n $NAMESPACE
```

Execute the moderate `Pipeline` using the following command:
```bash
tkn pipeline start compliance-pipeline \
    -w name=shared-workspace,volumeClaimTemplateFile=compliance/pvc.yaml \
    -p namespace=$NAMESPACE \
    --showlog
```

A TaskRun executes the `Steps` in a `Task` in the sequentially, until all 
`Steps` execute successfully, or a failure occurs.

List the `Taskrun` using the following command:
```bash
tkn taskrun ls -n $NAMESPACE
```

## Ansible Pipeline Example
Create the `ansible-deployer` service account:
```bash
oc apply -f- <<EOF
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: ansible-deployer-account
  namespace: $NAMESPACE
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: ansible-deployer
rules:
  # Core API
  - apiGroups: ['']
    resources: ['services', 'pods', 'deployments', 'configmaps', 'secrets']
    verbs: ['get', 'list', 'create', 'update', 'delete', 'patch', 'watch']
  # Apps API
  - apiGroups: ['apps']
    resources: ['deployments', 'daemonsets', 'jobs']
    verbs: ['get', 'list', 'create', 'update', 'delete', 'patch', 'watch']
  # Knative API
  - apiGroups: ['serving.knative.dev']
    resources: ['services', 'revisions', 'routes']
    verbs: ['get', 'list', 'create', 'update', 'delete', 'patch', 'watch']
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: ansible-deployer-binding
subjects:
  - kind: ServiceAccount
    name: ansible-deployer-account
    namespace: $NAMESPACE
roleRef:
  kind: ClusterRole
  name: ansible-deployer
  apiGroup: rbac.authorization.k8s.io
EOF
```

Create `ansible-playbooks` Persistent Volume Claim:
```bash
oc apply -f- <<EOF
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: ansible-playbooks
  namespace: $NAMESPACE
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Filesystem
  resources:
    requests:
      storage: 1Gi
---
EOF
```

Clone Git Repository:
```bash
tkn clustertask start git-clone \
  --workspace=name=output,claimName=ansible-playbooks \
  --param=url=https://github.com/ocp4opsandsecurity/openshift-pipelines \
  --param=revision=ansible \
  --param=deleteExisting=true \
  --showlog
```


```bash
 tkn task start ansible-runner \
   --serviceaccount ansible-deployer-account \
   --param=project-dir=kubernetes \
   --param=args='-p ansible/list-pods.yml' \
   --workspace=name=runner-dir,claimName=ansible-playbooks \
   --showlog
```

## References
- [Adding Operators](https://docs.openshift.com/container-platform/4.6/operators/admin/olm-adding-operators-to-cluster.html#olm-adding-operators-to-a-cluster)
- CLI
  - [Tekton Tools](https://github.com/tektoncd/cli/releases)
- GitHub
  - [OpenShift Pipelines Repository](https://github.com/openshift/pipelines-tutorial/)
  - [Tekton Repository](https://github.com/tektoncd/pipeline)
- [Understanding Openshift Pipelines](https://docs.openshift.com/container-platform/4.6/pipelines/understanding-openshift-pipelines.html?extIdCarryOver=true&sc_cid=701f2000001OH7iAAG)

