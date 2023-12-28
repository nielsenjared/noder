

```sh
docker run -it --rm -d -p 8080:80 --name web nginx
```

TODO stop
```sh
docker stop web
```

TODO 

```sh
docker run -it --rm -d -p 8080:80 --name web -v ~/app:/usr/share/nginx/html nginx
```


The output will be simliar to the following: 
```sh
app:/usr/share/nginx/html nginx
33e216bd2185c79d547cdbdbd2ccd2771d90d7f9da1cfd09c98ef04b756c0ffb
```

If you get a 403 error, you can TODO. Or just move on to the next step. 



### Build a Custom Image

Create a Dockerfile:
```dockerfile
FROM nginx:latest 
COPY ./app/index.html /usr/share/nginx/html/index.html
```

Build it:
```sh
docker build -t noder .
```

Don't forget the `.`

Run it:
```sh
docker run -it --rm -d -p 8080:80 --name app noder
```

TODO Setting up a reverse proxy server
https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/



### Docker Hub

Click "Create a Repository" on the [Docker Hub](https://hub.docker.com/) welcome page.

Name it <your-username>/noder.

From the command line, run the following: 
```sh
docker build -t nielsenjared/noder .
```

Then run the container:
```sh
docker run nielsenjared/noder
```

TODO exit? 

Then push the image to DockerHub:
```sh
docker push nielsenjared/noder
```

### Access Tokens
If you haven't yet, you will need to set up access tokens with Docker Hub. Do so under settings in the UI, then run: 
```
docker login -u nielsenjared
```

And at the `Password` prompt paste in your access token. 


## GitHub Actions

### Build Image on GitHub

Create a new Action and select the Docker Image workflow. 

TODO try it!


### Publish Changes to Docker Hub


Open the repository Settings, and go to Secrets > Actions.

Create a new secret named `DOCKER_HUB_USERNAME` and your Docker ID as value.


Create a new Personal Access Token (PAT) for Docker Hub.

TODO create an Access Token for GitHub


Add the PAT as a second secret in your GitHub repository, with the name DOCKER_HUB_ACCESS_TOKEN.


TODO 

Go to your repository on GitHub and then select the Actions tab.

Under "Choose a Workflow", click "...set up a workflow yourself."

This takes you to a page for creating a new GitHub actions workflow file in your repository, under `.github/workflows/main.yml` by default.

In the editor window, copy and paste the following YAML configuration.
```yml
name: Docker Hub CI

on:
  push:
    branches:
      - "main"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      -
        name: Checkout
        uses: actions/checkout@v3
      -
        name: Login to Docker Hub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKER_HUB_USERNAME }}
          password: ${{ secrets.DOCKER_HUB_ACCESS_TOKEN }}
      -
        name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      -
        name: Build and push
        uses: docker/build-push-action@v3
        with:
          context: .
          file: ./Dockerfile
          push: true
          tags: ${{ secrets.DOCKER_HUB_USERNAME }}/clockbox:latest
```

TODO explain above

Start a commit. 




















## Kubernetes

### Create a Kubernetes Cluster with EKS

```sh
eksctl create cluster --name nodernetes --region us-east-1
```

If you receive an error, such as:
```
Error: failed to create cluster "nodernetes"
```

Login to [CloudFormation](https://console.aws.amazon.com/cloudformation/
) and check its status, which is likely `ROLLBACK`. Delete the cluster, wait a few minutes, then try creating it again.  

Once your cluster is created, take a look at your nodes: 
```sh
kubectl get nodes -o wide
```

Then take a look at the workloads running on the cluster: 
```sh
kubectl get pods -A -o wide
```


#### Create a Namespace

TODO what's the naming convention? 
```
system+cluster-unique-id
```

```sh
kubectl create namespace eks-noder
```


### Creating Kubernetes Deployment Manifest

TODO
`eks-deployment.yml`

```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nodernetes
  namespace: eks-noder
  labels:
    app: noder
spec:
  replicas: 3
  selector:
    matchLabels:
      app: noder
  template:
    metadata:
      labels:
        app: noder
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64
                - arm64
      containers:
      - name: noder
        image: nielsenjared/noder:latest
        ports:
        - name: http
          containerPort: 80
        imagePullPolicy: IfNotPresent
      nodeSelector:
        kubernetes.io/os: linux
```

Apply the deployment manifest
TODO
```sh
kubectl apply -f eks-noder-deployment.yml 
```

The response will be:
```sh
deployment.apps/nodeernetes created
```


### Creating Kubernetes Service Manifest


TODO `eks-noder-service.yml`

```yml
apiVersion: v1
kind: Service
metadata:
  name: nodernetes
  namespace: eks-noder
  labels:
    app: noder
spec:
  selector:
    app: noder
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Apply the service manifest: 
```sh
  kubectl apply -f eks-noder-service.yml
```


The response will be:
```
service/nodernetes created
```


### View Resources and Details 

View the resources in the `noder` namespace: 
```sh
kubectl get all -n eks-noder
```

```sh
NAME                              READY   STATUS    RESTARTS   AGE
pod/nodernetes-67789fc995-64kf6   1/1     Running   0          2m14s
pod/nodernetes-67789fc995-n6pj7   1/1     Running   0          2m14s
pod/nodernetes-67789fc995-qk5xr   1/1     Running   0          2m14s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/nodernetes   3/3     3            3           2m15s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/nodernetes-67789fc995   3         3         3       2m15s
```

View the details: 

```sh
kubectl -n eks-noder describe service nodernetes
```

The output will be similar to the following: 
```sh
Name:              nodernetes
Namespace:         eks-noder
Labels:            app=noder
Annotations:       <none>
Selector:          app=noder
Type:              ClusterIP
IP Family Policy:  SingleStack
IP Families:       IPv4
IP:                10.100.27.118
IPs:               10.100.27.118
Port:              <unset>  8080/TCP
TargetPort:        8080/TCP
Endpoints:         192.168.15.169:8080,192.168.32.169:8080,192.168.48.198:8080
Session Affinity:  None
Events:            <none>
```

View the details of a pod:
```sh
kubectl -n eks-noder describe pod nodernetes-67789fc995-64kf6
```


Replace the alphanumeric values with those from one your pods listed above. 


The output will be similar to the following:
```sh
Name:             nodernetes-67789fc995-64kf6
Namespace:        eks-noder
Priority:         0
Service Account:  default
Node:             ip-192-168-46-106.ec2.internal/192.168.46.106
Start Time:       Wed, 21 Dec 2022 11:59:28 -0500
Labels:           app=noder
                  pod-template-hash=67789fc995
Annotations:      kubernetes.io/psp: eks.privileged
Status:           Running
IP:               192.168.48.198
IPs:
  IP:           192.168.48.198
Controlled By:  ReplicaSet/nodernetes-67789fc995
Containers:
  noder:
    Container ID:   docker://db78c3ffba30d0b57f7c8a8cbc5428d95c96726def2f3dc942889597bb187fd5
    Image:          nielsenjared/noder:latest
    Image ID:       docker-pullable://nielsenjared/noder@sha256:0ed2ce1e89407c54a146e67bad14e8219c8ed688392cfd63db73d27f524241b7
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 21 Dec 2022 11:59:33 -0500
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-j482x (ro)
Conditions:
  Type              Status
  Initialized       True 
  Ready             True 
  ContainersReady   True 
  PodScheduled      True 
Volumes:
  kube-api-access-j482x:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              kubernetes.io/os=linux
Tolerations:                 node.kubernetes.io/not-ready:NoExecute op=Exists for 300s
                             node.kubernetes.io/unreachable:NoExecute op=Exists for 300s
Events:
  Type    Reason     Age    From               Message
  ----    ------     ----   ----               -------
  Normal  Scheduled  5m33s  default-scheduler  Successfully assigned eks-noder/nodernetes-67789fc995-64kf6 to ip-192-168-46-106.ec2.internal
  Normal  Pulling    5m32s  kubelet            Pulling image "nielsenjared/noder:latest"
  Normal  Pulled     5m29s  kubelet            Successfully pulled image "nielsenjared/noder:latest" in 3.605031397s
  Normal  Created    5m28s  kubelet            Created container noder
  Normal  Started    5m28s  kubelet            Started container noder
```

### Root a Pod

```
kubectl exec -it nodernetes-67789fc995-64kf6 -n eks-noder -- /bin/bash
```

This will ssh in the pod as root.


### Hit the Server 
```
curl nodernetes
```

### TODO DNS
```sh
cat /etc/resolv.conf
```

TODO
```sh
nameserver 10.100.0.10
search eks-flasker.svc.cluster.local svc.cluster.local cluster.local ec2.internal
options ndots:5
```

10.100.0.10 is automatically assigned as the nameserver for all pods deployed to the cluster.


#### Create a Cluster IP Service

Verify the pods are running and associated with an IP address: 
```sh
kubectl get all -n eks-noder -o wide
```

Create a `clusterip.yml` file:
```yml
TODO

```

And create the service:
```sh
kubectl create -f clusterip.yml
```

##### Get the Cluster IP Number
TODO
```sh
kubectl get service eks-noder-ip
```

The output will be something similar to the following:
```sh
NAME           TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
eks-noder-ip   ClusterIP   10.100.99.47   <none>        80/TCP    56s
```

#### Create a NodePort Service

Create a file, `nodeport.yml` and add the following: 

```yml
apiVersion: v1
kind: Service
metadata:
  name: eks-noder-nodeport
spec:
  type: NodePort
  selector:
    app: noder
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Create the NodePort object:
```sh
kubectl create -f nodeport.yml
```

The output will be:
```sh
service/eks-noder-nodeport created
```

Let's get the details on our NodePort service:
```sh
kubectl get service/eks-noder-nodeport
```

The output will be similar to the following: 
```sh
NAME                 TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
eks-noder-nodeport   NodePort   10.100.148.73   <none>        80:31881/TCP   66s
```

TODO delete the NodePort service? 
```sh
kubectl delete service eks-noder-nodeport
```

#### Create a LoadBalancer Service

Create a `loadbalancer.yml` file and add the following: 

```yml
apiVersion: v1
kind: Service
metadata:
  name: eks-noder-loadbalancer
spec:
  type: LoadBalancer
  selector:
    app: noder
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

Create the LoadBalancer:
```sh
kubectl create -f loadbalancer.yml
```

You'll get a confirmation response: 
```sh
service/eks-noder-loadbalancer created
```

Let's get the details:
```sh
kubectl get service/eks-noder-loadbalancer
```

The output will be similar to the following:
```sh
NAME                     TYPE           CLUSTER-IP     EXTERNAL-IP                                                               PORT(S)        AGE
eks-noder-loadbalancer   LoadBalancer   10.100.33.18   a19cc054e55894bb59bc70e515094962-1554247827.us-east-1.elb.amazonaws.com   80:31514/TCP   58s
```


Hey hey! There's our external IP! Let's hit it!

#### Hit the Server! 

```sh
curl -silent a19cc054e55894bb59bc70e515094962-1554247827.us-east-1.elb.amazonaws.com:80 | grep title
```