# Context
This repo holds a simple PoC about building an image using [Kaniko](https://github.com/GoogleContainerTools/kaniko)

The image we are about to create using Kaniko is just an [nginx base image](https://hub.docker.com/_/nginx) with a custom hello world html file.

Kaniko will be running inside a Kubernetes [Kind](https://kind.sigs.k8s.i) cluster.

# How to ...
## ... create Kubernetes Kind cluster
```
kind create cluster --config kind/cluster.yaml --name kaniko
```
```
Creating cluster "kaniko" ...
 ✓ Ensuring node image (kindest/node:v1.20.2) 🖼
 ✓ Preparing nodes 📦
 ✓ Writing configuration 📜
 ✓ Starting control-plane 🕹️
 ✓ Installing CNI 🔌
 ✓ Installing StorageClass 💾
Set kubectl context to "kind-kaniko"
You can now use your cluster with:

kubectl cluster-info --context kind-kaniko
```
### Ensuring everything is OK
The Kubernetes Kind cluster is created
```
kind get clusters
kaniko
```
Kubernetes control-plane (also a worker in this PoC) node is working as a docker container.
```
docker ps
CONTAINER ID   IMAGE                  COMMAND                  CREATED         STATUS         PORTS                       NAMES
57a47cf629b9   kindest/node:v1.20.2   "/usr/local/bin/entr…"   4 minutes ago   Up 4 minutes   127.0.0.1:42021->6443/tcp   kaniko-control-plane
```

We can also check our build context is mounted inside a node from the kubernetes cluster
```
docker exec -it kaniko-control-plane ls -l /build-context/
total 0
-rwxrwxrwx 1 1000 1000 58 May 18 11:11 Dockerfile
-rwxrwxrwx 1 1000 1000 31 May 18 11:10 hello.html
```

## ... start the registry
Pull the image
```
docker pull registry
```
Load the image into kind cluster
```
kind load docker-image --name kaniko registry
```
Start the kubernetes deployment/service for the registry
```
kubectl apply -f k8s/registry.yaml
```

Please note this is a really simple configuration. Do not use in production environments.

## ... build using Kaniko
Pull the kaniko image
```
docker pull gcr.io/kaniko-project/executor:debug
```
Debug image will allow us to _bash into_ the container, to see for example if the build context has been correctly loaded.

Load the image into the kind cluster
```
kind load docker-image --name kaniko gcr.io/kaniko-project/executor:debug
```

Start the building process.
```
kubectl apply -f k8s/kaniko.yaml
```

And the image building process starts. As it is a very small image, the building process is very fast.

The job is completed
```
kubectl get jobs
NAME         COMPLETIONS   DURATION   AGE
kaniko-job   1/1           24s        85s
```

The pod holding the job is also completed
```
kubectl get pods
NAME                                          READY   STATUS      RESTARTS   AGE
kaniko-job-k6mq4                              0/1     Completed   0          99s
```

The logs from the kaniko process. 
```
kubectl logs kaniko-job-k6mq4
INFO[0000] Retrieving image manifest nginx:latest
INFO[0000] Retrieving image nginx:latest from registry index.docker.io
INFO[0001] Built cross stage deps: map[]
INFO[0001] Retrieving image manifest nginx:latest
INFO[0001] Returning cached image manifest
INFO[0001] Executing 0 build triggers
INFO[0001] Unpacking rootfs as cmd COPY hello.html /usr/share/nginx/html requires it.
INFO[0015] COPY hello.html /usr/share/nginx/html
INFO[0015] Taking snapshot of files...
INFO[0015] Pushing image to docker-registry-service:5000/nginx-hello-world:latest
INFO[0022] Pushed image to 1 destinations
```

### Ensure everything is working OK
The image is being created inside the docker registry pod
```
kubectl get pods
NAME                                          READY   STATUS      RESTARTS   AGE
docker-registry-deployment-6876ff49f5-tmh7k   1/1     Running     2          18h

kubectl exec --stdin --tty docker-registry-deployment-6876ff49f5-tmh7k
-- ls -l /var/lib/registry/docker/registry/v2/repositories
drwxr-xr-x    5 root     root          4096 May 19 06:06 nginx-hello-world
```

We can query the registry for the images
```
kubectl port-forward docker-registry-deployment-6876ff49f5-tmh7k 5000

curl http://localhost:5000/v2/nginx-hello-world/tags/list
{"name":"nginx-hello-world","tags":["latest"]}
```

And also we can pull the image. It is important to note the `tls-verify` option as the registry is insecure and it will fail with regular options.
```
podman pull --tls-verify=false localhost:5000/nginx-hello-world:latest
Trying to pull localhost:5000/nginx-hello-world:latest...
Getting image source signatures
Copying blob de9813870342 skipped: already exists
Copying blob cfcd0711b93a skipped: already exists
Copying blob 49f7d34d62c1 skipped: already exists
Copying blob 69692152171a skipped: already exists
Copying blob 5f97dc5d71ab skipped: already exists
Copying blob 63d9f7b0e991 [--------------------------------------] 0.0b / 0.0b
Copying blob be6172d7651b [--------------------------------------] 0.0b / 0.0b
Copying config 4ae7080d6c done
Writing manifest to image destination
Storing signatures
4ae7080d6c04440a3515fa292f2c4dc6468a6904967d3c651cb4916e548abc0b
```
**Note:** if we add an entry in `hosts` file, we can replace `localhost` with `docker-registry-service`
> 127.0.0.1 docker-registry-service

There are two images, the same hash
```
podman image ls
REPOSITORY                                      TAG           IMAGE ID      CREATED            SIZE
localhost:5000/nginx-hello-world                latest        4ae7080d6c04
docker-registry-service:5000/nginx-hello-world  latest        4ae7080d6c04
```

Run a container based on the image

```
podman run -it --rm --name nginx -p 8080:80 docker-registry-service:5000/nginx-hello-world
curl http://localhost:8080
I have been built using Kaniko.
```
# Links
## Kind
* [Kind example config](https://raw.githubusercontent.com/kubernetes-sigs/kind/main/site/content/docs/user/kind-example-config.yaml)
* [Extra mounts](https://kind.sigs.k8s.io/docs/user/configuration/#extra-mounts)
* [Loading an image into your cluster](https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster)

### images present in the cluster
From `https://kind.sigs.k8s.io/docs/user/quick-start/#loading-an-image-into-your-cluster`
```
You can get a list of images present on a cluster node by using docker exec:

docker exec -it my-node-name crictl images

Where my-node-name is the name of the Docker container (e.g. kind-control-plane).
```

* [KiND - How I Wasted a Day Loading Local Docker Images](https://iximiuz.com/en/posts/kubernetes-kind-load-docker-image/)
* [Docker registry as a pod in kubernetes](https://medium.com/swlh/deploy-your-private-docker-registry-as-a-pod-in-kubernetes-f6a489bf0180)