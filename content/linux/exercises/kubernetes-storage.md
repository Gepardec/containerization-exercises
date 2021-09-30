# Kubernetes Storage

By the end of this exercise, you should be able to:

- Understand the purpose of storage providers and storage classes
- Know how to persistently store data in Kubernetes

## Volumes

On-disk files in a container are ephemeral, which presents some problems for non-trivial applications when running in containers. One problem is the loss of files when a container crashes. The kubelet restarts the container but with a clean state. A second problem occurs when sharing files between containers running together in a Pod. The Kubernetes volume abstraction solves both of these problem.

Out of the box, kubernetes can store persistent data locally on a node. But If the node fails, the data is lost. To really get fail safe storage, where our data will be replicated across multiple nodes, we need to install and use a storage provisioner.

## Storage Provisioner

Storage providers handle storage location, replication and dynamic volume creation in kubernetes. In this example, we will install Rancher Longhorn. If you run kubernetes inside AWS, Azure or GC, you can use the their cloud volumes out of the box, without installing a storage provisioner.

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
kubectl create namespace longhorn-system
helm install longhorn longhorn/longhorn --namespace longhorn-system
```

Time to refill your coffee. This will take a few minutes. You can check the status of the installation by running:

```bash
kubectl -n longhorn-system get pod
```

You should see several running containers.

## How does it work?

Longhorn saves all volumes in /var/lib/longhorn, but replicates it to 2 different nodes. Therefore, we will not lose any data when a node fails.

## Persistent Volume Claim

To use a persistent volume for our pod, we need to create a persistent volume claim (PVC).

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: foo-pvc
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: longhorn
  resources:
    requests:
      storage: 1Gi
```

After creating the claim, we can create the pod that uses this claim.
You can check the status of your claim by running:
```bash
kubectl get pvc #check the claim
kubectl get pv #check the created volume
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pvc-usage-pod
spec:
  containers:
  - name: pvc-usage-pod
    image: busybox
    command: ["/bin/sh", "-c", "while true; do sleep 1; done"]
    volumeMounts:
    - name: pvc-usage-volume
      mountPath: /pvc-usage-volume
  volumes:
  - name: pvc-usage-volume
    persistentVolumeClaim:
      claimName: foo-pvc
```

After your pod is running, your persistent volume will be mounted to `/pvc-usage-volume`.
Try to write a file to your persistent volume:

```bash
kubectl exec -it pvc-usage-pod — /bin/sh
touch /pvc-usage-volume/test.txt
```

Now, delete and recreate your pod. After the pod is running, you should see the file in your persistent volume.
```bash
kubectl exec -it pvc-usage-pod — /bin/sh
ls /pvc-usage-volume
```


## StatefulSet

Statefulsets (STS) are like deployments, except they manage their own PVCs automatically. 

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongo
spec:
  selector:
    matchLabels:
      app: mongo
  serviceName: "mongo"
  replicas: 1
  template:
    metadata:
      labels:
        app: mongo
    spec:
      terminationGracePeriodSeconds: 10
      containers:
      - name: mongo
        image: mongo
        ports:
        - containerPort: 27017
          name: mongo
        volumeMounts:
        - name: data
          mountPath: /data/db
  # STS are using a template, to create the PVC
  volumeClaimTemplates:
  - metadata:
      name: data
    spec:
      accessModes: [ "ReadWriteOnce" ]
      storageClassName: longhorn
      resources:
        requests:
          storage: 512Mi
```

The statefulset creates a PVC for each replica.

```bash
kubectl get pods #to check if the deployment was successful
kubectl get pvc #to check if the PVCs were created
```

## Scaling a statefulset

With statefulsets, we have the advantage that each replica gets their own PVC. For demonstration, let's scale our statefulset to 3 replicas.

```bash
kubectl scale sts mongo —replicas 3
kubectl get pods # check the pods being created
kubectl get pvc # check how many pvc were created
```

## Cleanup

```bash
kubectl delete sts mongo
kubectl delete pod pvc-usage-pod
kubectl delete pvc foo-pvc
```

## Sources

https://kubernetes.io/docs/concepts/storage/volumes/