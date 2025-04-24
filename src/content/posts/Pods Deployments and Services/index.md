---
title: "Understanding Pods, Deployments, and Services in Kubernetes"
published: 2025-04-24
description: "A detailed guide to Kubernetes Pods, Deployments, and Services for beginners."
image: "cover.png"
tags: ["Beginner", "DevOps", "Kubernetes", "Pods", "Deployments", "Services"]
category: Kubernetes Essentials
draft: false
---

Kubernetes can feel like a complex ecosystem, but once you understand its building blocks, everything starts to make sense. In this article, we’ll explore the three most fundamental components in Kubernetes: Pods, Deployments, and Services.

By the end, you’ll have a solid understanding of how your applications are deployed, managed, and exposed in a Kubernetes cluster.

# What is a Pod?

A Pod is the smallest and most basic deployable unit in Kubernetes. A Pod represents a single instance of a running process in your cluster.

#### Key Characteristics:

- A Pod can contain one or more containers (usually one).
- Containers in a Pod share:
  - The same network namespace (same IP address and port space).
  - The same storage volumes, if defined.

> Think of a Pod as a wrapper around one or more tightly coupled containers that must run together.

## Why Pods?
Pods are useful for:

- Running a containerized app.
- Grouping helper containers (like logging agents) with the main container.

Examples - 

1. Simple Pod:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx-container
    image: nginx:latest
    ports:
    - containerPort: 80
```

2. Pod with Volumes, Multiple Containers including init containers and environment variables:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app.kubernetes.io/name: MyApp
spec:
  containers:
    - name: myapp-container
      image: busybox:1.28
      command: ['sh', '-c', 'echo The app is running! && sleep 3600']
      env:
        - name: ENVIRONMENT
          value: "production"
        - name: CONFIG_FILE
          value: "/etc/config/app.conf"
      volumeMounts:
        - name: config-vol
          mountPath: /etc/config
  initContainers:
    - name: init-myservice
      image: busybox:1.28
      command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
    - name: init-mydb
      image: busybox:1.28
      command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level.conf
``` 
## Manifest breakup for better understanding

Here are breakdowns for the components used in the above manifest:

:::important[Environment Variables]

```yaml
    env:
      - name: DEMO_GREETING
        value: "Hello from the environment"
      - name: DEMO_FAREWELL
        value: "Such a sweet sorrow"
```

:::


:::important[Volumes]

Volumes are defined in the `volumes` section of the manifest. They can be used/mounted by one or more containers in the Pod.

``` yaml
    volumeMounts:
      - name: config-vol
        mountPath: /etc/config
  volumes:
    - name: config-vol
      configMap:
        name: log-config
        items:
          - key: log_level
            path: log_level.conf
```
:::

:::important[Init Containers]

Init containers are used to perform setup tasks before the main containers in the Pod start.

```yaml
  initContainers:
    - name: init-myservice
      image: busybox:1.28
      command: ['sh', '-c', "until nslookup myservice.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for myservice; sleep 2; done"]
    - name: init-mydb
      image: busybox:1.28
      command: ['sh', '-c', "until nslookup mydb.$(cat /var/run/secrets/kubernetes.io/serviceaccount/namespace).svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
```
:::



---

# What is a Deployment?
While a Pod is great, it’s **not self-healing**. If a Pod crashes, it won’t come back automatically. That’s where a Deployment comes in.

A Deployment is a higher-level abstraction that manages Pods and ReplicaSets for you.

#### Key Benefits:

- Ensures desired state is maintained (e.g., always 3 replicas running).
- Supports rolling updates and rollbacks.
- Automatically replaces failed Pods.

Examples:

1. Simple Deployment:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 80
```

:::note
Most of the fields in the Deployment manifest are the same as in a Pod manifest. The key difference is that in a Deployment, you specify the number of replicas you want to run. It is specified in the `replicas` field.
:::

2. Deployment with Volumes, Multiple Containers including init containers and environment variables:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: demo-app
  labels:
    app: demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: demo
  template:
    metadata:
      labels:
        app: demo
    spec:
      volumes:
        - name: shared-data
          emptyDir: {}
        - name: config-volume
          configMap:
            name: demo-config
      initContainers:
        - name: init-myservice
          image: busybox
          command: ['sh', '-c', 'echo "Initializing..." > /mnt/data/init.log']
          volumeMounts:
            - name: shared-data
              mountPath: /mnt/data
      containers:
        - name: demo-container
          image: nginx:1.25
          ports:
            - containerPort: 80
          env:
            - name: ENVIRONMENT
              value: "production"
            - name: CONFIG_FILE
              value: "/etc/config/app.conf"
          volumeMounts:
            - name: shared-data
              mountPath: /usr/share/nginx/html/init
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
---
#configmap-manifest
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-config
data:
  app.conf: |
    server {
      listen 80;
      location / {
        return 200 "Hello from ConfigMap!";
      }
    }
```
## Some more manifest breakups for better understanding

:::important[Shared Volumes]
Same volumes can be shared by multiple containers in a Pod. This is useful for sharing data between containers in the same Pod.

```yaml
      volumes:
        - name: shared-data
          emptyDir: {}
      initContainers:
        - volumeMounts:
            - name: shared-data
              mountPath: /mnt/data
      containers:
        - volumeMounts:
            - name: shared-data
              mountPath: /usr/share/nginx/html/init
```
:::

:::important[Configmap as volumes]
ConfigMaps can be used in two primary ways:

1. As environment variables
2. As volumes (mounted files inside your container)

In this example, we've mounted it as a volume -

for configmap manifest, refer to the `#configmap-manifest` section in the above manifest.

```yaml
      volumes:
        - name: config-volume
          configMap:
            name: demo-config
      containers:
        - volumeMounts:
            - name: config-volume
              mountPath: /etc/config
              readOnly: true
```

Once deployed, inside the container you’ll have:

```bash
/etc/config/app.conf
```

This config file is then used by the container to configure the application using environment variable.
```yaml
      containers:
        - env:
            - name: CONFIG_FILE
              value: "/etc/config/app.conf"
```

:::note 
Mounting **ConfigMap** as a **volume** makes the config **read-only** by default, which is a nice security feature. 
:::

---

# What is a Service?

A Service in Kubernetes exposes your Pods to other applications inside (or outside) the cluster.

## Types of Services:
1. ClusterIP (default): Internal communication only.
2. NodePort: Exposes the service on a port on each node.
3. LoadBalancer: Exposes the service externally using a cloud provider's load balancer.
4. ExternalName: Maps the service to an external DNS name.

Example (NodePort):
```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30007
```

This will expose your nginx deployment on http://<NodeIP>:30007.

---

# Putting it All Together: A Practical Example
Here’s an example to deploy Nginx and expose it via a NodePort service.

🧩 Create the Deployment
```bash
kubectl create deploy nginx --image=nginx
```
🧪 Check the Pods
```bash 
kubectl get pods
```

🌐 Create the Service
```bash
kubectl expose deploy nginx --name=nginx-service --port=30007 --target-port=80 --type=NodePort
```

🌍 Access the App
```bash
minikube service nginx-service
```
Or manually:

```bash
minikube ip
```

Visit http://<minikube-ip>:<nodePort> in your browser.

:::tip
1. You can use `kubectl apply -f --dry-run=client -o <file>` to validate manifests before applying.
2. You can use `kubectl <remaining-command> --dry-run=client -o yaml` to generate a manifest.
:::

---

# Final Thoughts
Pods, Deployments, and Services form the core trio of any Kubernetes application.

Pods are your running containers.

Deployments manage your Pods at scale and keep them healthy.

Services make sure your application is reachable—internally or externally.

Understanding these three concepts is crucial for progressing in your Kubernetes journey. Once you’ve mastered them, you’ll be ready to move on to more advanced topics like ConfigMaps, Secrets, Ingress, Volumes, and beyond.

Next up: In the upcoming article, we’ll explore Ingress controllers, how to handle configurations with ConfigMaps, and persistent storage in Kubernetes.

Until then, keep learning and stay secure! 🚀