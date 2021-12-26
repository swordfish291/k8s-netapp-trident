# NetApp Astra Trident for Kubernetes Persistent Volumes and Persistent Volume Claims

## Introduction
Containers are ephemeral so is the data that is contained inside the container, as soon as the container is destroyed the data contained inside the container is also destroyed. That's not what we need right? We need some sort of persistent storage that does not follow the POD/Container lifecycle. Kuberenetes addresses this challenge using Volumes, PersistentVolumes and PersistentVolumeClaims. 
### Volumes
A volume is a directory, possibly with some data in it, which is accessible to the containers in a pod. How that directory comes into existence, the medium that backs it, and the contents of it are determined by the particular volume type used. There are two ways that we can attach a Volume to a POD. First: We declare the Volume inside the POD manifest file under ``spec:`` and then we can call those mounts under ``containers:`` and `volumeMounts:`

Example:
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx-1
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - mountPath: /nginx-1
      name: nginx-1
  - name: nginx-2
    image: nginx
    command: ["/bin/sh","-c"]
    args: ["sed -i 's/listen  .*/listen 81;/g' /etc/nginx/conf.d/default.conf && exec nginx -g 'daemon off;'"]
    ports:
    - containerPort: 81
    volumeMounts:
    - mountPath: /nginx-2
      name: nginx-2
  volumes:
  - name: nginx-1
    hostPath:
      path: /vagrant/volumes/nginx-1/
  - name: nginx-2
    hostPath:
      path: /vagrant/volumes/nginx-2/
```

There are two containers as part of this manifest file and map two different volumes of `hostPath` type. These two volumes are created and then mapped to the containers using `volumeMounts:` section. 
This is one way of creating a volume and attaching it to the container. This type of configuration has a huge disadvantage that the cluster and application administrators have to manage all these configurations and these configurations are not "portable". If the same workload has to be deployed on another platform or cloud platform then all these configurations have to be modified. That's a pain. Introducing PersistentVolumes and PersistentVolumeClaims. 

## Persistent Volumes and Persistent Volume Claims

Kuberenetes PersistentVolume and PersistentVolumeClaim API resources provide users and cluster administrators a way to provision storage to containers and it also abstracts details of how storage is provided to the containers from how it is consumed by the containers. Administrators create PersistentVolume objects in the cluster that is a storage resource that is backed by a storage provider. These objects are claimed by users using another Kuberenetes object called PersistentVolumeClaim. These PVC objects can be compared to Pods. Pods consume compute resources and PVC consume PersistentVolume. Example:

```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-pv-vol
spec:
  capacity:
    storage: 2Gi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  mountOptions:
    - hard
    - nfsvers=3
  nfs:
    path: /data
    server: 192.168.204.150

---

apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cl-nfs-pv-vol
  labels:
    app: nginx
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 1Gi

---

apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nfs-pv-vol
      mountPath: /data
  volumes:
    - name: nfs-pv-vol
      persistentVolumeClaim:
        claimName: cl-nfs-pv-vol
```

This creates a PersistentVolume object named ``nfs-pv-vol`` and a PersistentVolumeClaim named ``cl-nfs-pv-vol``, this claim is then used by the `nginx` pod. 

This way this makes it very easy for the administrators to manage the storage resources and it also simplifies the workload portability and reduces configuration complexities. However, this still is still a challenge for the cluster administrator to create and manage those PersistentVolume objects at scale. This way of provisioning is called Static Provisioning. To make the volume creation dynamic as per the claim instead of statically creating them every-time a volume is needed, Dynamic Volume Provisioning is introduced. The dynamic provisioning feature eliminates the need for cluster administrators to pre-provision storage. Instead, it automatically provisions storage when it is requested by users.

The implementation of dynamic volume provisioning is based on the API object `StorageClass`. A cluster administrator can define as many `StorageClass` objects as needed, each specifying a _volume plugin_ (aka _provisioner_) that provisions a volume and the set of parameters to pass to that provisioner when provisioning. A cluster administrator can define and expose multiple flavours of storage (from the same or different storage systems) within a cluster, each with a custom set of parameters. This design also ensures that end users don't have to worry about the complexity and nuances of how storage is provisioned, but still have the ability to select from multiple storage options. Each StorageClass contains the fields `provisioner`, `parameters`, and `reclaimPolicy`, which are used when a PersistentVolume belonging to the class needs to be dynamically provisioned. 

For this to work Kubernetes needs a way to communicate the volume provisioning needs to the storage subsystems. This where Container Storage Interface or CSI comes into play. CSI defines an industry standard that enables storage vendors like NetApp, Amazon or Azure to develop plugins that can receive these provisioning request and provision volumes for the cluster to consume.  

## NetApp Astra Trident
Astra Trident is a Container Storage Interface (CSI) compliant dynamic storage orchestrator that enables consumption and management of storage resources across all popular NetApp storage platforms. It natively integrates with Kubernetes to dynamically provision persistent volume requests on demand. 

### Install Trident

#### Validate permissions 
To deploy Trident, you will need to validate that:
- you have enough privileges to the Kubernetes cluster. You can validate this using:
``kubectl auth can-i '*' '*' --all-namespaces``

- Access to the NetApp cluster and the credentials to the SVM that will be hosting our volumes.
#### Download and extract the Installer
You can download and extract installer using:
```
#Downoad the installer
wget https://github.com/NetApp/trident/releases/download/v21.10.1/trident-installer-21.10.1.tar.gz

#Extract 
tar -xf trident-installer-21.10.1.tar.gz

#Navigate to the directory
cd trident-installer
```


### Deploy using Helm

```
#Create a namespace for the trident workloads
kubectl create namespace trident

#Set the context
kubectl config set-context --current --namespace=trident

```
Navigate to the Helm directory and execute `helm install`

```
#Navigate to Helm directoy
cd helm

#Helm install trident operator
helm install trident trident-operator-21.10.1.tgz
```

Check the status of the helm deployment using `helm status trident`
![[Pasted image 20211225225531.png]]
Check the status of the pods using `kubectl get pods`
![[Pasted image 20211225225722.png]]
Check the status of the installation using `tridentctl`
![[Pasted image 20211225230108.png]]

### Setup backend storage
Now that trident is installed and ready to be used, it needs to have a storage platform that it needs to create PersistentVolumes on. This information is saved in a Json file and an object is created using `tridentctl` called a backend. In our case that backend is a NetApp storage SVM. The sample configuration for this type of backend is available in the `trident-nstaller` folder `trident-installer/sample-input/backends-samples`. In our case we will use `ontap-nas/backend-ontap-nas.json`. We will modify as per our environment:

```
{
    "version": 1,
    "storageDriverName": "ontap-nas",
    "backendName": "customBackendName",
    "managementLIF": "192.168.204.160",
    "dataLIF": "192.168.204.150",
    "svm": "svm0",
    "username": "vsadmin",
    "password": "Netapp1234"
}
```

These details can be filled up. The LIF information can be obtained from NetApp CLI:
![[Pasted image 20211225232347.png]]

`vsadmin` can be modified and set with another password using:
 ![[Pasted image 20211225233023.png]]

Once the `JSON` file is ready we can execute:

```
./tridentctl -n trident create backend -f backend-ontap-nas.json
```
![[Pasted image 20211225235001.png]]

Now the backend is online we can proceed with creating the storageClass and PersistentVolumeClaim. 

### Create the StorageClass
For users to create Persistent Volume Claims, storageClass object is needed. That storage class points to the backend that we created and also contains details about the storage for the user so they can make correct provisioning decisions. Lets create the storageClass using:

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ontap-nas
provisioner: csi.trident.netapp.io
parameters:
  backendType: "ontap-nas"
allowVolumeExpansion: true
```

Save this to a YAML and execute `kubectl create -f ontap-nas-sc.yaml`
StorageClass is ready:

![[Pasted image 20211225235941.png]]

### Create PersistentVolumeClaim 
Now that the storageClass is ready, we can create a PVC that will use this storageClass and as a result a volume will be created on the storage side through trident. 

```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-ontap-nas
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 3Gi
  storageClassName: ontap-nas
  ```

Save this to YAML and execute `kubectl create -f pvc-ontap-nas.yaml`
![[Pasted image 20211226000605.png]]
We can observe that a volume object `pvc-356fc9e5-f748-4517-9240-94ad7a332118` is created. We can also validate this from the NetApp CLI:

![[Pasted image 20211226001233.png]]

### Create a Pod that uses the PVC
Lets create a Pod manifest file that uses the PVC `pvc-ontap-nas`
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    ports:
    - containerPort: 80
    volumeMounts:
    - name: ontap-nas-vol1
      mountPath: /data
  volumes:
    - name: ontap-nas-vol1
      persistentVolumeClaim:
        claimName: pvc-ontap-nas
```

Save this to a YAML file and execute `kubectl create -f nginx.yaml`

![[Pasted image 20211226002042.png]]
![[Pasted image 20211226002110.png]]


### References:
https://netapp.io/persistent-storage-provisioner-for-kubernetes/
