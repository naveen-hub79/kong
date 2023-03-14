# kong ingress controller

- install kong ingress controller on kubernetes cluster
```
curl https://raw.githubusercontent.com/Kong/kubernetes-ingress-controller/master/deploy/single/all-in-one-postgres.yaml \
| kubectl create -f -
```
- It takes about 5 minutes to install all the necessary components ,if persistent storage is not configured in your cluster configure it.
- I used local-storage  for storage class and pv setup
- apply the below manifests for storage setup 
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: my-local-pv
spec:
  storageClassName: local-storage
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: /mnt

---
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
- check all the pods in kong namespace are up and running 
- now after installing kong ingress controller you will get crds and a ingressclass
```
kubectl get ingressclass
```
- now we wil deploy one application and try to access it vai our ingress
- to deploy the ech-server app use below manifests
```
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: http-svc
spec:
  replicas: 1
  selector:
    matchLabels:
      app: http-svc
  template:
    metadata:
      labels:
        app: http-svc
    spec:
      containers:
      - name: http-svc
        image: gcr.io/google_containers/echoserver:1.8
        ports:
        - containerPort: 8080
        env:
        - name: NODE_NAME
          valueFrom:
            fieldRef:
              fieldPath: spec.nodeName
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: POD_IP
          valueFrom:
            fieldRef:
              fieldPath: status.podIP

---
apiVersion: v1
kind: Service
metadata:
  name: http-svc
  labels:
    app: http-svc
spec:
  type: NodePort
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: http-svc
```
- the above yaml will create a deployment and a service of type ClusterIP
- now we need to create a ingress resource which can give us the public IP with the help of metalLb
- apply below manifest for creating a n ingress resource
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: http-svc-ingress
spec:
  ingressClassName: kong
  rules:
  # - host: foo.bar
  - http:
      paths:
      - path: /
        pathType: ImplementationSpecific
        backend:
          service:
            name: http-svc
            port: 
              number: 80
```
- the above manifest will create an ingress resource for the app which we deployed by using its service
- i commented Host field in the file beacuse i dont want validate the request using host headers 
- if you use the host header in your ingress use below command to test connection else you can skip -H option
```
curl -vvvv $PROXY_IP:$HTTP_PORT -H "Host: foo.bar"
```
	
##########
- now we we deploy and test the same with nginx application
- deploy a nginx application by using below yaml
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
spec:
  selector:
    matchLabels:
      app: nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx # Update the version of nginx from 1.14.2 to 1.16.1
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: ClusterIP
  ports:
  - port: 80
  selector:
    app: nginx      
```
- now we have to create an ingress resource to give access to our service from extrenal world
- since we are not using host headers we should use path based routing to distinguish between services 
- we have to add stripath annotation into ingress configuration to avoid path mismatch in side the application 	
- below is the configuartion for it we are giving /nginx as path but the we are striiping it to / path giving that annotation
```
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nginx-svc-ingress
  annotations:
    # konghq.ingress.kubernetes.io/rewrite-target: /
    # ingress.kubernetes.io/service-upstream: "true"
    # konghq.com/path: /
    konghq.com/strip-path: 'true'
    # plugins.konghq.com: http-log
spec:
  ingressClassName: kong
  rules:
  # - host: nginx.com
  - http:
      paths:
      - path: /nginx
        pathType: Prefix
        backend:
          service:
            name: my-nginx-svc
            port: 
              # name: http
              number: 80
          
```
- i can create different ingresses for each service or i an have a single ingress for multiple services sing tis host headers and paths
