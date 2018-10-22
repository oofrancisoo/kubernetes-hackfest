# Lab: Persistent Volume Claims

Coming soon.

## Prerequisites

* Complete previous labs for [AKS](../../create-aks-cluster/README.md) and [ACR](../../build-application/README.md).

## Instructions

This section contains a number of labs related to Storage decisions that are made when doing Production deployments of Kubernetes. They are each self-contained labs meaning you do not have to do one before another.

## Prerequisites

* Clone this repo in Azure Cloud Shell.
* Complete previous labs for [AKS](../create-aks-cluster/README.md) and [ACR](../build-application/README.md).

A persistent volume is a piece of storage that has been created for use in a Kubernetes cluster. A persistent volume can be used by one or many pods and can be dynamically or statically created. This document details dynamic creation of an Azure file share as a persistent volume.

For more information on Kubernetes persistent volumes, including static creation, see [Kubernetes persistent volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/).

Create a storage account[](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#create-a-storage-account)
----------------------------------------------------------------------------------------------------------------------

When dynamically creating an Azure file share as a Kubernetes volume, any storage account can be used as long as it is in the AKS node resource group. This group is the one with the *MC_* prefix that was created by the provisioning of the resources for the AKS cluster. Get the resource group name with the [az aks show][az-aks-show] command.

Azure CLICopy

```
az aks show -g $RGNAME -n $CLUSTERNAME --query nodeResourceGroup -o tsv

MC_myResourceGroup_myAKSCluster_eastus

```

Use the [az storage account create](https://docs.microsoft.com/cli/azure/storage/account#az-storage-account-create) command to create the storage account.

Update `--resource-group` with the name of the resource group gathered in the last step, and `--name` to a name of your choice. Provide your own unique storage account name:

Azure CLICopy

```
az storage account create -g $RGNAME --name mystorageaccount --sku Standard_LRS

```

Note

Azure Files currently only work with Standard storage. If you use Premium storage, the volume fails to provision.

Create a storage class[](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#create-a-storage-class)
------------------------------------------------------------------------------------------------------------------

A storage class is used to define how an Azure file share is created. A storage account can be specified in the class. If a storage account is not specified, a *skuName* and *location* must be specified, and all storage accounts in the associated resource group are evaluated for a match. For more information on Kubernetes storage classes for Azure Files, see [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/#azure-file).

Create a file named `azure-file-sc.yaml` and copy in the following example manifest. Update the *storageAccount* value with the name of your storage account created in the previous step. For more information on *mountOptions*, see the [Mount options](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#mount-options) section.

yamlCopy

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
parameters:
  skuName: Standard_LRS
  storageAccount: mystorageaccount

```

Create the storage class with the [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) command:

consoleCopy

```
kubectl apply -f azure-file-sc.yaml

```

Create a cluster role and binding[](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#create-a-cluster-role-and-binding)
----------------------------------------------------------------------------------------------------------------------------------------

AKS clusters use Kubernetes role-based access control (RBAC) to limit actions that can be performed. *Roles* define the permissions to grant, and *bindings* apply them to desired users. These assignments can be applied to a given namespace, or across the entire cluster. For more information, see [Using RBAC authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/).

To allow the Azure platform to create the required storage resources, create a *ClusterRole* and *ClusterRoleBinding*. Create a file named `azure-pvc-roles.yaml` and copy in the following YAML:

yamlCopy

```
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: system:azure-cloud-provider
rules:
- apiGroups: ['']
  resources: ['secrets']
  verbs:     ['get','create']
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: system:azure-cloud-provider
roleRef:
  kind: ClusterRole
  apiGroup: rbac.authorization.k8s.io
  name: system:azure-cloud-provider
subjects:
- kind: ServiceAccount
  name: persistent-volume-binder
  namespace: kube-system

```

Assign the permissions with the [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) command:

consoleCopy

```
kubectl apply -f azure-pvc-roles.yaml

```

Create a persistent volume claim[](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#create-a-persistent-volume-claim)
--------------------------------------------------------------------------------------------------------------------------------------

A persistent volume claim (PVC) uses the storage class object to dynamically provision an Azure file share. The following YAML can be used to create a persistent volume claim *5GB* in size with *ReadWriteMany* access. For more information on access modes, see the [Kubernetes persistent volume](https://kubernetes.io/docs/concepts/storage/persistent-volumes) documentation.

Now create a file named `azure-file-pvc.yaml` and copy in the following YAML. Make sure that the *storageClassName* matches the storage class created in the last step:

yamlCopy

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: azurefile
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: azurefile
  resources:
    requests:
      storage: 5Gi

```

Create the persistent volume claim with the [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) command:

consoleCopy

```
kubectl apply -f azure-file-pvc.yaml

```

Once completed, the file share will be created. A Kubernetes secret is also created that includes connection information and credentials. You can use the [kubectl get](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#get) command to view the status of the PVC:

Copy

```
$ kubectl get pvc azurefile

NAME        STATUS    VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
azurefile   Bound     pvc-8436e62e-a0d9-11e5-8521-5a8664dc0477   5Gi        RWX            azurefile      5m

```

Use the persistent volume[](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#use-the-persistent-volume)
------------------------------------------------------------------------------------------------------------------------

The following YAML creates a pod that uses the persistent volume claim *azurefile* to mount the Azure file share at the */mnt/azure*path.

Create a file named `azure-pvc-files.yaml`, and copy in the following YAML. Make sure that the *claimName* matches the PVC created in the last step.

yamlCopy

```
kind: Pod
apiVersion: v1
metadata:
  name: mypod
spec:
  containers:
    - name: myfrontend
      image: nginx
      volumeMounts:
      - mountPath: "/mnt/azure"
        name: volume
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: azurefile

```

Create the pod with the [kubectl apply](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#apply) command.

consoleCopy

```
kubectl apply -f azure-pvc-files.yaml

```

You now have a running pod with your Azure disk mounted in the */mnt/azure* directory. This configuration can be seen when inspecting your pod via `kubectl describe pod mypod`. The following condensed example output shows the volume mounted in the container:

Copy

```
Containers:
  myfrontend:
    Container ID:   docker://053bc9c0df72232d755aa040bfba8b533fa696b123876108dec400e364d2523e
    Image:          nginx
    Image ID:       docker-pullable://nginx@sha256:d85914d547a6c92faa39ce7058bd7529baacab7e0cd4255442b04577c4d1f424
    State:          Running
      Started:      Wed, 15 Aug 2018 22:22:27 +0000
    Ready:          True
    Mounts:
      /mnt/azure from volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-8rv4z (ro)
[...]
Volumes:
  volume:
    Type:       PersistentVolumeClaim (a reference to a PersistentVolumeClaim in the same namespace)
    ClaimName:  azurefile2
    ReadOnly:   false
[...]

```

Mount options[](https://docs.microsoft.com/en-us/azure/aks/azure-files-dynamic-pv#mount-options)
------------------------------------------------------------------------------------------------

Default *fileMode* and *dirMode* values differ between Kubernetes versions as described in the following table.

| version | value |
| --- | --- |
| v1.6.x, v1.7.x | 0777 |
| v1.8.0-v1.8.5 | 0700 |
| v1.8.6 or above | 0755 |
| v1.9.0 | 0700 |
| v1.9.1 or above | 0755 |

If using a cluster of version 1.8.5 or greater and dynamically creating the persistent volume with a storage class, mount options can be specified on the storage class object. The following example sets *0777*:

yamlCopy

```
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: azurefile
provisioner: kubernetes.io/azure-file
mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000
parameters:
  skuName: Standard_LRS

```

If using a cluster of version 1.8.5 or greater and statically creating the persistent volume object, mount options need to be specified on the *PersistentVolume* object. for more information on statically creating a persistent volume, see [Static Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/#static).

yamlCopy

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: azurefile
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteMany
  azureFile:
    secretName: azure-secret
    shareName: azurefile
    readOnly: false
  mountOptions:
  - dir_mode=0777
  - file_mode=0777
  - uid=1000
  - gid=1000

```

If using a cluster of version 1.8.0 - 1.8.4, a security context can be specified with the *runAsUser* value set to *0*. For more information on Pod security context, see [Configure a Security Context](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/).

## Troubleshooting / Debugging

* ?

## Docs / References

* ?