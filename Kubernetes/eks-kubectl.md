# Kubectl

`kubectl`. The Kubernetes command-line tool, `kubectl`, allows you to run commands against Kubernetes clusters. You can use kubectl to deploy applications, inspect and manage cluster resources, and view logs.

## Setup AWS iam access key

To get your AWS iam access key and secret key, please connact to our manager. If you already got it, let's setup aws cli reference to this [AWS document](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html). Execute command

```bash
aws configure --profile myprofile
AWS Access Key ID [None]: <Your AWS Access Key ID>
AWS Secret Access Key [None]: <Your AWS Secret Access Key>
Default region name [None]: ap-northeast-1
Default output format [None]:
```

## Setup kubectl

- **1.Install kubectl tool**

You can refer to this [AWS document](https://docs.aws.amazon.com/eks/latest/userguide/install-kubectl.html) to get `kubectl` tool that maths to your operating system.

- **2.Get the cluster context**

```bash
aws --profile myprofile eks update-kubeconfig --region ap-northeast-1 --name mynamespace 
```

- **3.Check the kubectl**

```bash
$ kubectl get pods -o wide -n <namespace>
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE                                             NOMINATED NODE   READINESS GATES
myapp-59fdfbf6c8-d48w5   2/2     Running   0          2d1h    10.30.2.231   ip-10-30-2-34.ap-northeast-1.compute.internal    <none>           <none>
```

## Kubectl commands

These are some common command lines for kubectl

### Get nodes

```bash
$ kubectl get nodes -owide

NAME                                             STATUS   ROLES    AGE     VERSION               INTERNAL-IP   EXTERNAL-IP    OS-IMAGE         KERNEL-VERSION                  CONTAINER-RUNTIME
ip-10-30-1-100.ap-northeast-1.compute.internal   Ready    <none>   7d19h   v1.27.3-eks-a5565ad   10.30.1.100   <none>         Amazon Linux 2   5.10.184-175.749.amzn2.x86_64   containerd://1.6.19
ip-10-30-1-125.ap-northeast-1.compute.internal   Ready    <none>   7d19h   v1.27.3-eks-a5565ad   10.30.1.125   <none>         Amazon Linux 2   5.10.184-175.749.amzn2.x86_64   containerd://1.6.19
```

### Get pods

```bash
$ kubectl get pods -o wide -n mynamespace
NAME                     READY   STATUS    RESTARTS   AGE     IP            NODE                                             NOMINATED NODE   READINESS GATES
myapp-59fdfbf6c8-d48w5   2/2     Running   0          2d1h    10.30.2.231   ip-10-30-2-34.ap-northeast-1.compute.internal    <none>           <none>
```

### Get services

```bash
$ kubectl get svc -o wide -n mynamespace
NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE     SELECTOR
myapp            ClusterIP   172.20.26.195    <none>        3333/TCP          3d18h   workload.user.cattle.io/workloadselector=apps.deployment-myapp
```

### Access container shell

```bash
$ kubectl -n mynamespace exec --stdin --tty myapp-59fdfbf6c8-d48w5 -- /bin/ash
```

### kubectl cp

- **1. Upload file from host to pod**
  
  ```bash
  kubectl -n mynamespace cp <path-to-file> myapp-59fdfbf6c8-d48w5:<fully-qualified-file-name>
  ```
- **2. Get file from pod to host**
  
  ```bash
  kubectl -n mynamespace cp myapp-59fdfbf6c8-d48w5:<fully-qualified-file-name> <path-to-file>
  ```
