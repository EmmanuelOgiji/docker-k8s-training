# Kubernetes: Deploying and Configuring Pods

This exercise has the following objectives:

- Create a simple local kubernetes cluster
- Understand what a [Pod](https://kubernetes.io/docs/concepts/workloads/pods/) is
- Deploy a Pod with your flask app container image to your cluster
- [Port forward](https://kubernetes.io/docs/tasks/access-application-cluster/port-forward-access-application-cluster/)
  to the Pod to check that it is working
- Inspect Logs of the container within your pod
- Add [Environment Variables](https://kubernetes.io/docs/tasks/inject-data-application/define-environment-variable-container/)
to the container in your Pod
- Run commands on a container for debugging
- Add [Probes](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/) to
  your Pod
- [Configure Resources](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/) for the
  container in the Pod
- Add an [initContainer](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) to your Pod

## Pre-requisites

In addition to docker, this exercise requires kubectl and minikube. Details on installing both are
available [here](https://kubernetes.io/docs/tasks/tools/)

## Create a simple local kubernetes cluster

- `minikube start`

## Deploy a Pod with your flask app container image to your cluster

- Populate a pod.yaml file which:
    - references our flask app container image
    - exposes the 9900 port of the container

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
    - name: my-flask-app
      image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.2.0
      ports:
        - containerPort: 9900
```

- Create the pod: `kubectl create -f <path to pod.yaml>`

## Port forward to the Pod to check that it is working

- Port forward to the container port: `kubectl port-forward pod/my-flask-app 9900:9900`
- Running the above command means that port 9900 on your localhost now maps to port 9900 on your pod
- Go to `localhost:9900` and `localhost:9900/author` on your browser

## Inspect Logs of the container within your pod

- To see your container logs: `kubectl logs my-flask-app`

## Add Environment Variables to the container in the Pod

We are going to set two environment variables here:

- Set the `LOG_LEVEL` environment variable to `DEBUG`
- Set the `AUTHOR` environment variable to `Pod`

To do this:

- Delete the current running pod: `kubectl delete pod my-flask-app`
- Edit your pod.yaml file by adding the following under the container

```yaml
   env:
     - name: LOG_LEVEL
       value: "DEBUG"
     - name: AUTHOR
       value: "Pod"
```

```diff
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.2.0
    ports:
    - containerPort: 9900
+   env:
+   - name: LOG_LEVEL
+     value: "DEBUG"
+   - name: AUTHOR
+     value: "Pod"
```

- Recreate the pod `kubectl create -f <path to pod.yaml>`
- (Optional) Repeat the port forward and log inspection steps to see differences

## Run commands on a container for debugging

This is a very useful tool for debugging. To do this, we need to open a shell within the pod.

- Run `kubectl exec -it my-flask-app -- sh`
- Type `exit` and press enter when done to close the shell

## Add Probes to your Pod

There are three main probes in Kubernetes which serve to show different levels of progress for a Pod or workload:

- Liveness probes are used to know when an application restart might be needed i.e. the app is running but not
  responding/progressing (not alive)
- Readiness probes are used to know when an application is ready for traffic
- Startup probes are used to know when an application has finish starting up/booting
  Each of these probes could be commands to be executed, http requests or tcp sockets. Probes are also continuously
  running so things like intervals, initial delays, failure/success thresholds can be set too

For this, we are going to add:

- A startup probe that checks the /author path is accessible (http request)
- Readiness and liveness probes that both check the 9900 port is available (tcp socket)

To do this:

- Delete the current running pod: `kubectl delete pod my-flask-app`
- Edit your pod.yaml file by adding the following under the container:

```yaml
startupProbe:
  httpGet:
    path: /author
    port: 9900
  failureThreshold: 30
  periodSeconds: 10
  readinessProbe:
    tcpSocket:
      port: 9900
    initialDelaySeconds: 5
    periodSeconds: 10
  livenessProbe:
    tcpSocket:
      port: 9900
    initialDelaySeconds: 5
    periodSeconds: 10
```

```diff
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.2.0
    ports:
    - containerPort: 9900
    env:
    - name: LOG_LEVEL
      value: "DEBUG"
    - name: AUTHOR
      value: "Pod"
+   startupProbe:
+     httpGet:
+       path: /author
+       port: 9900
+     failureThreshold: 30
+     periodSeconds: 10
+   readinessProbe:
+     tcpSocket:
+       port: 9900
+     initialDelaySeconds: 5
+     periodSeconds: 10
+   livenessProbe:
+     tcpSocket:
+       port: 9900
+     initialDelaySeconds: 15
+     periodSeconds: 20
```

- Recreate the pod `kubectl create -f <path to pod.yaml>`
- Run `kubectl get pods -o wide` and pay attention to the `READINESS GATES` column especially

## Configure Resources for the Pod Container

Kubernetes Pod containers resource configuration usually has to do with allocating memory (RAM) and CPU to a container.
This is important because if the application/container has an issue like an out of memory error, your compute costs are
capped i.e. the cluster won't keep scaling to meet the unwieldy demand.

Resources are usually configured under two components:

- Request: The amount of resource (memory or CPU) allocated to a container at start time. It should be noted that the
  container may be given more resource if it requires it
- Limit: This is the maximum amount of resource the container can have allocated to it. If the container in fact
  requires more, it is restarted.
  In summary, when resources are configured, the container has the "request" amount of resource allocated, it can then
  consume more if needed until it reaches the "limit" amount

To configure resources for our Pod

- Run our application: `python3 ${PATH_TO_REPO}/01-docker-flask-app/src/app.py`
- Use our System Monitor/Activity Monitor/Task manager to see what usage looks like. Based on that, you can determine
  your values for requests and limits for CPU and Memory.
- Let's use the following values: memory requests: 25Mi, memory limits: 50Mi, cpu requests: 250m, cpu limits: 500m
    - To do this,
        - Delete the current running pod: `kubectl delete pod my-flask-app`
        - Edit your pod.yaml file to add the following resource config under the container

```yaml
resources:
  requests:
    memory: "100Mi"
    cpu: "100m"
  limits:
    memory: "200Mi"
    cpu: "200m"
```

```diff
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.2.0
    ports:
    - containerPort: 9900
+   resources:
+     requests:
+       memory: "25Mi"
+       cpu: "250m"
+     limits:
+       memory: "50Mi"
+       cpu: "500m"
    env:
    - name: LOG_LEVEL
      value: "DEBUG"
    - name: AUTHOR
      value: "Pod"
    startupProbe:
      httpGet:
        path: /author
        port: 9900
      failureThreshold: 30
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 9900
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 9900
      initialDelaySeconds: 5
      periodSeconds: 10
```

- Recreate the pod `kubectl create -f <path to pod.yaml>`
- Run `kubectl top pod my-flask-app` to see resource allocation of the pod. Note this will give the
  error `Metrics API unavailable` in our case.
  This is due to us using just a `demo cluster`
  . [This link](https://signoz.io/blog/kubectl-top/#using-kubectl-top-pod-command) shows an example of what to expect
  when using that command

## Add an initContainer to your Pod

Using init containers is one of the multi-container Pod design patterns. As the name implies, initContainers give you
the ability to run a container (or a number of containers) to carry out certain tasks before the main container(s) in a
Pod.

These init containers are so called as they are often used to carry out "initialization" related actions. These could be
things like installing packages, creating files that the main container needs and so on.

We are going to add a simple initContainer to our pod that just echoes a message then sleeps for 30 seconds using a
busybox image. To do this,

- Delete the current running pod: `kubectl delete pod my-flask-app`
- Edit your pod.yaml to add the initContainer portion under spec:

```yaml
initContainers:
  - image: busybox:1.28
    name: init-sleep
    command: [ 'sh', '-c', 'echo "Initializing...waiting for a minute." && sleep 30s;' ]
```

```diff
apiVersion: v1
kind: Pod
metadata:
  name: my-flask-app
spec:
+ initContainers:
+ - name: init-sleepy
+   image: busybox:1.28
+   command: ['sh', '-c', 'echo "Initializing...waiting for a minute." && sleep 30s;']
  containers:
  - name: my-flask-app
    image: ghcr.io/emmanuelogiji/cloudboosta-flask-app:0.2.0
    ports:
    - containerPort: 9900
    resources:
      requests:
        memory: "25Mi"
        cpu: "250m"
      limits:
        memory: "50Mi"
        cpu: "500m"
    env:
    - name: LOG_LEVEL
      value: "DEBUG"
    - name: AUTHOR
      value: "Pod"
    startupProbe:
      httpGet:
        path: /author
        port: 9900
      failureThreshold: 30
      periodSeconds: 10
    readinessProbe:
      tcpSocket:
        port: 9900
      initialDelaySeconds: 5
      periodSeconds: 10
    livenessProbe:
      tcpSocket:
        port: 9900
      initialDelaySeconds: 5
      periodSeconds: 10
```

- Recreate the pod `kubectl create -f <path to pod.yaml>`
- Run `kubectl get pods` and observe the status
- Check the logs of the init container: `kubectl logs my-flask-app -c init-sleepy`. You should
  see `Initializing...waiting for a minute.` as expected

As mentioned, there are other multi container patterns for
pods. [This blog post](https://medium.com/bb-tutorials-and-thoughts/kubernetes-learn-sidecar-container-pattern-6d8c21f873d)
talks about the other most popular one which is the sidecar pattern. It also has links to blogs about the others as well
