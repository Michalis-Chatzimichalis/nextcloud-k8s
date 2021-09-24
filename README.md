# NextCloud Server on a single-node Raspberry Pi with Kubernetes
Set up a Nextcloud Server on a Raspberry Pi with minikube (kubectl), a Maria-DB and a 1TB WD myBook.

- [Database Pod & Service Definition](#database-pod--service-definition)
- [Server Pod & Service Definition](#server-pod--service-definition)
- [Volume for Database and Server](#volume-for-database-and-server)
- [Networking and Forward](#networking-and-forward)

## Database Pod & Service Definition
The first part of this code concerns itself with the type of pod and also what name, labels the pod will receive.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-db
  labels:
    apps: nextcloud
```
This part concerns itself with the inherent specifications of the pod. How many **replicas**, the **labels** that are going to be given and then the **template** that is needed to be defined, for when a pod goes down and a replacement is spun up.\
The template will consist of a MariaDB Image, Enviroment Variables (Database Name; nextcloud) and then the Secrets that are given as envFrom. 

```yaml
spec:
  replicas: 2
  selector:
    matchLabels:
      pod-label: nextcloud-db-pod
    template:
      metadata:
        labels:
          pod-label: nextcloud-db-pod
      spec:
        containers:
        - name: mariadab
          image: mariadb:10.5
          env:
          - name: MARIADB_DATABASE
            value: nextcloud
          envFrom:
          - secretRef:
            name: nextcloud-db-secret #to create
          volumeMounts:
          - name: db-storage
            mountPath: /var/lib/mariadb
            subPath: mariadb-data
        volumes:
        - name: db-storage
          persistentVolumeClaim:
            claimName: nextcloud-storage-pvc
```

One could create two files with the Pod Definition and then the Service, yet a simple myriad of underscores does the trick. This code defines the Service that the Pod will use, as well as getting the same Label as the Pod there's also a Reference to the Port that the database will use.

```yaml
___
apiVersion: v1
kind: Service
metadata:
  name: nextcloud-db
  labels:
    app: nextcloud
spec:
  selector:
    pod-label: nextcloud-db-pod
  ports:
  - protocol: TCP
    port: 3306
```

## Server Pod & Service Definition
The first part of this code again concerns itself with the type of pod and also what name, labels the pod will receive.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nextcloud-server
  labels:
    app: nextcloud- [Database Pod & Service Definition](#database-pod--service-definition)
- [Database Pod & Service Definition](#database-pod--service-definition)
- [Server Pod & Service Definition](#server-pod--service-definition)
- [Volume for Database and Server](#volume-for-database-and-server)
- [Networking and Forward](#networking-and-forward)
```
This part concerns itself with the inherent specifications of the pod. How many **replicas**, the **labels** that are going to be given and then the **template** that is needed to be defined, for when a pod goes down and a replacement is spun up.\
The template will consist of a Nextcloud Image, a Volume Mount of the standard Apache Folder that serves the HTML Content, with the `sub-path` value of the folder that is going to be used. If the pod crashes the data would be lost if it would be mounted, therefore we create a PVC (Persistent Volume Claim) that saves our data and deploys it in realtime with the template pod. 
```yaml
spec:
  replicas: 1
  selector:
    matchLabels:
      pod-label: nextcloud-server-pod
  template:
    metadata:
      labels:
        pod-label: nextcloud-server-pod
    spec:
      containers:
      - name: nextcloud
        image: arm64v8/nextcloud:production
        volumeMounts:
        - name: server-storage
          mountPath: /var/www/html
          subPath: server-data
      volumes:
      - name: server-storage
        persistentVolumeClaim:
          claimName: nextcloud-storage-pvc
```
This part is used for the Service declaration of the Server that the Pod will use, as well as getting the same Label as the whole Application, there's also a Reference to the Port that the server will use (with the correct Pod-Label).

```yaml
___
apiVersion: v1
kind: Service
metadata:
  name: nextcloud-server
  labels:
    app: nextcloud
spec:
  selector:
    pod-label: nextcloud-server-pod
  ports:
  - protocol: TCP
    port: 80
```

## Volume for Database and Server
The volume that is referenced in both the DB and Server deployments is created and given the **App-Label** nextcloud. The **permissions** given are ephemeral, in the way that one can Read and Write to the Storage once at a time. The capacity request is always 256 Mibibytes.

```yaml
apiVersion: apps/v1
kind: PersistentVolumeClaim
metadata:
  name: nextcloud-storage-pvc
  labels:
    apps: nextcloud
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 256Mi
```

## Networking and Forward
This file is used for the URL forwarding to access Nextcloud outside of the local network. The host is the URL used to connect to and it should redirect to URL/nextcloud which in turn loads the Service (and the Pod) `nextcloud-server` on Port 80

```yaml
apiVersion: networking.k8s.io/v1beta1
kind: Ingress
metadata:
  name: cluster-ingress
spec:
  rules:
  - host: files.domain.local #my Domain will be appended/replaced
    http:
      paths:
        path: /nextcloud
        backend:
          serviceName: nextcloud-server
          servicePort: 80
```