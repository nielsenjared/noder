

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
          containerPort: 5000
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
deployment.apps/flaskernetes created
```


