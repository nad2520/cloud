# Local Kubernetes demo

This is Part 2: reuse the Docker images and let Kubernetes manage the containers.
Deployments create and maintain Pods; Services give them stable network addresses.
MongoDB has a persistent volume claim (PVC) so replacing its Pod keeps its data.

## Prerequisites

- Linux: Docker Engine, [Minikube](https://minikube.sigs.k8s.io/docs/start/),
  and [kubectl](https://kubernetes.io/docs/tasks/tools/).
- Windows: Docker Desktop running Linux containers with its WSL 2 backend,
  plus Minikube and kubectl. Run these commands in PowerShell from the repository root.
- Internet access for initial downloads and enough resources for Minikube
  (at least 2 CPUs, 2 GB of free RAM, and 20 GB of free disk space;
  more RAM helps with MongoDB and replicas).

This guide uses Minikube on both operating systems. Docker Desktop's built-in
Kubernetes is an alternative, but does not need to be enabled for this guide.
No AWS account, cloud credits, or container registry is required.

Stop this project's Compose stack first if it is running, to free ports 5000 and 5173:

```sh
docker compose down
```

## 1. Start a cluster and load the images

```sh
minikube start --driver=docker
kubectl config current-context
docker build -t cloud-backend:k8s ./backend
docker build -t cloud-frontend:k8s ./frontend
minikube image load cloud-backend:k8s
minikube image load cloud-frontend:k8s
```

Confirm the context is `minikube` before continuing. Minikube supplies the default
storage provisioner used by the MongoDB PVC. If startup reports exhausted Docker
address pools, resolve that local Docker network issue before deploying; do not
delete networks belonging to other projects.

## 2. Deploy

Create the namespace and a local JWT secret. Replace the placeholder with your
own random value; do not commit the value to Git. Create this Secret only once.

```sh
kubectl apply -f k8s/namespace.yaml
kubectl -n cloud-demo create secret generic backend-secrets --from-literal=JWT_SECRET=REPLACE_WITH_YOUR_OWN_RANDOM_VALUE
kubectl apply -f k8s/
kubectl -n cloud-demo rollout status deployment/mongodb --timeout=180s
kubectl -n cloud-demo rollout status deployment/backend --timeout=180s
kubectl -n cloud-demo rollout status deployment/frontend --timeout=180s
kubectl -n cloud-demo get pods,services,pvc
```

The backend can restart while MongoDB initializes; Kubernetes retries it.
If deployment fails, inspect `kubectl -n cloud-demo get pods`,
`kubectl -n cloud-demo describe pod POD_NAME`, and
`kubectl -n cloud-demo logs deployment/backend`.

## 3. Open the app

Keep each command running in a separate terminal:

```sh
kubectl -n cloud-demo port-forward service/frontend 5173:5173
```

```sh
kubectl -n cloud-demo port-forward service/backend 5000:5000
```

Open http://localhost:5173 and http://localhost:5000/api/health in your browser.
The health endpoint should return `{"status":"ok"}`.
The existing login/signup screens are UI demos and do not call authentication routes.

## 4. Demonstrate scaling

```sh
kubectl -n cloud-demo scale deployment backend --replicas=3
kubectl -n cloud-demo rollout status deployment/backend --timeout=180s
kubectl -n cloud-demo get pods -l app=backend
kubectl -n cloud-demo get endpointslices -l kubernetes.io/service-name=backend -o wide
```

Expect three backend Pods showing `1/1` Ready, and three ready backend addresses
in the Service's EndpointSlice(s). Save this output for the scalability demonstration.
The Service routes new connections from inside the cluster across ready replicas.
Port-forward selects one Pod, so refreshing the browser through it does not
demonstrate load balancing. More replicas on one laptop do not create more CPU/RAM
or prove higher throughput; this demonstrates changing the number of instances.

Return to one replica:

```sh
kubectl -n cloud-demo scale deployment backend --replicas=1
```

Keep MongoDB at one replica: copying a standalone MongoDB Pod does not configure
database replication. Reapplying `k8s/` also restores the declared replica count to one.

## Stop and cleanup

Stop the port-forward commands with Ctrl+C. Pause the cluster, keeping its data:

```sh
minikube stop
```

Use `minikube start --driver=docker` to start it again. To remove this demo
completely, including its MongoDB data and Secret:

```sh
kubectl delete namespace cloud-demo
```

This configuration is for local learning: Vite runs in development mode, MongoDB
has no authentication and is exposed only through a cluster-internal Service.
