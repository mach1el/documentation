# Deploy application to EKS cluster by Gitlab CI/CD pipeline

## Create EKS IAM role

### Step 1: Create a New IAM Role for EKS Nodes

1. Go to the IAM Console in AWS.
2. Click on "Roles" in the sidebar, then click "Create role".
3. Select "AWS service" as the trusted entity.
4. Choose EC2 as the use case (since EKS worker nodes run on EC2 instances).
5. Click "Next: Permissions."

### Step 2: Attach Policies to the Role

You need to attach the necessary policies to allow the nodes to interact with the EKS cluster and other AWS services (like ECR, if needed).

1. Attach the following policies:
    - AmazonEKSWorkerNodePolicy
    - AmazonEKS_CNI_Policy
    - AmazonEC2ContainerRegistryReadOnly (optional, if your gitlab-runner needs to pull images from ECR)
2. Click "Next: Tags" (you can add optional tags for identification).
3. Click "Next: Review", then give the role a name, such as gitlab-runner-node-role.
4. Click "Create role."

### Step 3: Assign the Role to the GitLab Runner Node

Now that you've created the role, you need to assign this role to the EC2 instance(s) where the gitlab-runner is running.

1. Go to the EC2 Console in AWS.
2. Select the EC2 instance that is running the gitlab-runner.
3. In the Actions menu, select Security > Modify IAM Role.
4. In the IAM role dropdown, select the gitlab-runner-node-role you just created.
5. Click Update IAM role.

## Add gitlab-runner role to aws-auth

To allow your gitlab-runner machine to deploy applications to the EKS cluster in Gitlab CI/CD, you need to add your `gitlab-runner` **<role/user>** to the `aws-auth` configmap.

### Step 1: Get the Current `aws-auth` ConfigMap
You can view the current `aws-auth` ConfigMap to ensure you don't overwrite existing settings. Use the following command:

```bash
kubectl get configmap aws-auth -n kube-system -o yaml
```

### Step 2: Edit the aws-auth ConfigMap

To modify the ConfigMap and add the gitlab-runner user, run:

```bash
kubectl edit configmap aws-auth -n kube-system
```

This will open the ConfigMap in your default editor. You will need to add the gitlab-runner user under the `mapRoles` section of the ConfigMap. If the section does not exist, you can create it.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapRoles: |
    - rolearn: arn:aws:iam::111122223333:role/gitlab-runner-role
      username: gitlab-runner
      groups:
        - system:masters
```

* **rolearn**: The ARN of the IAM role (in this case, the gitlab-runner role).
* **username**: The name you want the role to be referenced by in Kubernetes.
* **groups**: The Kubernetes groups the role should belong to. system:masters gives full admin access, but you can restrict this based on your needs.

> Replace arn:aws:iam::111122223333:user/gitlab-runner with the actual ARN of the gitlab-runner IAM user.

### Step 3: Verify the configuration

On gitlab-runner, we will get our EKS cluster configuration file to work it kubectl

```bash
aws eks update-kubeconfig --region ap-northeast-1 --name MyCluster
```

To test if the setup is work, it must be return like this

```bash
gitlab-runner@ip-127-0-0-1 ~ → kubectl get pods -o wide -n webapp
Error from server (Forbidden): pods is forbidden: User "gitlab-runner" cannot list resource "pods" in API group "" in the namespace "webapp"
```

### Step 4: Addd RBAC role and rolebinding

We were able to interact with EKS via kubectl. Now, let's give permissions on some resources for `Deployment` to `gitlab-runner`. 

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: gitlab-runner
  namespace: webapp
rules:
  # Permissions for managing Deployments and ReplicaSets
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  # Permissions for managing Pods, with 'delete' for cleanup purposes
  - apiGroups: [""]
    resources: ["pods"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Permissions for Services - read-only except for create, update, patch during app deployment
  - apiGroups: [""]
    resources: ["services"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  # Permissions for Secrets, strictly limited to read access
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

  # Permissions for managing ConfigMaps - writing is allowed to enable configuration updates
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  # Permissions for Ingresses and IngressClasses - necessary for managing networking and routing
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses", "ingressclasses"]
    verbs: ["get", "list", "watch", "create", "update", "patch"]

  # Permissions for PersistentVolumeClaims, as necessary for apps with persistent storage
  - apiGroups: [""]
    resources: ["persistentvolumeclaims"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
```

Rolebinding for `gitlab-runner` role:

```bash
kubectl create rolebinding gitlab-runner-binding --role=gitlab-runner --user=gitlab-runner --serviceaccount=<name-space>:default -n webapp
```

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: gitlab-runner-binding
  namespace: webapp
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: gitlab-runner
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: gitlab-runner
  - kind: ServiceAccount
    name: default
    namespace: webapp
```

Check if kubectl on gitlab-runner,it able to get pods on specify namespace

```bash
gitlab-runner@ip-127-0-0-1 ~ → kubectl get pods -o wide -n webapp
NAME                            READY   STATUS    RESTARTS   AGE    IP           NODE                                          NOMINATED NODE   READINESS GATES
activemq-5545497df6-njb5c       1/1     Running   0          149m   10.0.8.2     ip-10-0-8-2.ap-northeast-1.compute.internal   <none>           <none>
daemon-7bbb5d4969-48lrb         1/1     Running   0          146m   10.0.8.3     ip-10-0-8-3.ap-northeast-1.compute.internal   <none>           <none>
redis-59859bfd8c-fsjnn          1/1     Running   0          152m   10.0.8.4     ip-10-0-8-4.ap-northeast-1.compute.internal   <none>           <none>
streamfinder-7bb6899485-bnsjl   1/1     Running   0          143m   10.0.8.5     ip-10-0-8-5.ap-northeast-1.compute.internal   <none>           <none>
```