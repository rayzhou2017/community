# DevOps Resources Migration to CRD 

> Target alpha version: kubesphere v3.0


For a long time, DevOps-related resources use OLTP databases to store auxiliary data (for example, MySQL) 
and store the main data in back-end DevOps components (for example, Jenkins).Using this method makes the creation / update / deletion of our DevOps resources not reliable enough (such as OLTP database failure), 
and we cannot easily guarantee the consistency of resources.

The current CRD + controller mode is the official extension mode recommended by kubernetes. 
This can solve many problems we encountered in the previous version due to the use of (OLAP + DevOps).

Therefore, we will continue to migrate DevOps resources to CRD implementation in subsequent versions.

When we completely migrate resources to CRD, kubernetes etcd will be the sole source of configuration information, 
and the data in the DevOps backend will only automatically synchronize the data in kubernetes etcd through the controller

## Target resource type

In the initial release, we will mainly implement the following three resource types:

* DevOps Project
* DevOps Pipeline
* DevOps Credential

### DevOps Project

DevOps Project is a tenant in KubeSphere DevOps, similar to namespaces in Kubernetes. We need to ensure the isolation of resources in the DevOps Project, 
and try not to destroy the original authentication system of Kubernetes.

Therefore, we decided to implement the DevOps Project using the following scheme.

The DevOps Project is a cluster-level resource. Each DevOps Project will have a unique namespace (we call it the AdminNamespace of the DevOps Project). All resources in a DevOps project should be stored in the AdminNamespace. 
This can ensure the compatibility of the resources under the DevOps project with the Kubernetes authentication system, but for the DevOps project at the cluster level, we still have to break the Kubernetes authentication system.

Each DevOps project can be bound to an Adminnamespace. This operation can be manually set (like kubectl / kuberentes rest api). It can also be automatic generate by the controller.

For end users, they should all use automatic generation instead of manual binding.
The manual binding method will be used for the migration and upgrade of old DevOps projects.


### DevOps Pipeline

The API configured in the pipeline will be migrated to CRD, but for the query API, it will still be provided by kubesphere apiserver.

The pipelines that belong to the DevOps project will belong to the Adminnamespace to which the DevOps project is bound.
The access method of the API is similar to that in the previous version, 
but we have cancelled some field names that are strongly bound to Jenkins, such as Jenkinsfile. 
We hope to enable KubeSphere DevOps to connect with different types of DevOps platforms in an abstract way, similar to CRI / CSI / CNI.


### DevOps Credential

DevOps Credential We will use a specific type of Kubernetes Secret for implementation. 
Furthermore, when the type of Secret is prefixed with `credentials.devops.kubesphere.io/`, 
it means that this is DevOps Credential and not just Secret.

DevOps Credential will be synchronized by controller with CRD and devops storage.

## Data migration

> Known migration issues    
> CRD's meta data (such as creation timestamp) is automatically generated by Kubernetes. 
> Data migration will cause changes in these resources, such as the creation timestamp will become the migration timestamp instead of the real creation time.

For existing KubeSphere, we need to provide automatic migration tools to migrate traditional DevOps to CRD implementation. 
We will use resources as the granularity to illustrate the prototype design of these migration tools.


### DevOps Project

DevOps project CRs will be automatically created by the CRD migration tool, and the DevOps backend data will remain unchanged during the migration process.

### DevOps Pipeline

DevOps pipeline CRs will be automatically created by the CRD migration tool, and the DevOps backend data will remain unchanged during the migration process.

### DevOps Credentials

Because of Jenkins' design, the secret part of the credential cannot be easily obtained after creation, which means that our migration tool cannot directly create the same Secert.
So after the migration, we will only create a Credential reference, and mark out that the Secret data should not be synchronized by annotation. 

When users edit Secret in KubeSphere Console, the annotation should be removed by KubeSphere Console.
When users use kubectl / kube-apiserver to edit, this annotation should be manually removed, so that the controller can automatically synchronize data to the DevOps backend.


## Query API

In the current version, the query API will still be provided through kubesphere apiserver, not CRD.
