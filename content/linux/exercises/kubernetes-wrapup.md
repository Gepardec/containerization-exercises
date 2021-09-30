# Wrapup Exercise

In this Exercise, you need to combine a few skills you have learned today. We will:
- Deploy a Chat application
- Deploy a Database
- Enable access to the application from the outside world
- Test the application for failure recovery

### Task 1
- Create a StatefulSet to deploy a MongoDB database (`image: mongo`).
- Make the database data persistent (files are saved in `/data/db`)
- Use the following container command to configure mongodb: `mongod --oplogSize 128 --replSet rs0 --bind_ip_all`
- After the database is deployed, connect to it (`kubectl exec -it mongo-0 -- mongo`) and initialize a replica set using the following command:  
- `rs.initiate({_id: 'rs0',members: [ { _id: 0, host: 'localhost:27017' } ]});`
- Expose Port 27017 of the Database as a Service.

### Task 2
- Create a Deployment for the Rocket Chat application.
- Expose Port 3000 of the application as a Service.
- Use the environment variables below to configure the application:
```yaml
image: registry.rocket.chat/rocketchat/rocket.chat:latest
env:
  - PORT=3000
  - ROOT_URL=http://localhost:3000
  - MONGO_URL=mongodb://mongo:27017/rocketchat
  - MONGO_OPLOG_URL=mongodb://mongo:27017/local
```

### Task 3
- Create an Ingress to access the application fron the outside world.

### Task 4
- Check if its working :)


## Example Solution
- Task 1  
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
          env:
          - name: MONGO_INITDB_DATABASE
            value: rocketchat
          ports:
          - containerPort: 27017
            name: mongo
          volumeMounts:
          - name: data
            mountPath: /data/db
          command:
            - /bin/sh
            - -c
            - |-
              mongod --oplogSize 128 --replSet rs0 --bind_ip_all
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
```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: mongo
  spec:
    selector:
      app: mongo
    ports:
    - port: 27017
      targetPort: 27017
      protocol: TCP
```

- Task 2  
```yaml
  apiVersion: apps/v1
  kind: Deployment
  metadata:
    name: rocketchat
    labels:
      app: rocketchat
  spec:
    replicas: 1
    selector:
      matchLabels:
        app: rocketchat
    template:
      metadata:
        labels:
          app: rocketchat
      spec:
        containers:
        - name: rocketchat
          image: registry.rocket.chat/rocketchat/rocket.chat:latest
          ports:
          - containerPort: 3000
          env:
          - name: PORT
            value: "3000"
          - name: ROOT_URL
            value: "http://localhost:3000"
          - name: MONGO_URL
            value: "mongodb://mongo:27017/rocketchat"
          - name: MONGO_OPLOG_URL
            value: "mongodb://mongo:27017/local"
```
```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: rocketchat
  spec:
    selector:
      app: rocketchat
    ports:
    - port: 3000
      targetPort: 3000
      protocol: TCP
```

- Task 3  
```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    name: rocketchat
  spec:
    rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: rocketchat
              port:
                number: 3000
```
