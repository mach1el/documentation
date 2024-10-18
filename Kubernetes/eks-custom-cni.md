## Custom networking

With reference to this [AWS document](https://docs.aws.amazon.com/eks/latest/userguide/cni-custom-network.html), we can setup CNI custom networking to avoid IP address exhaustion in the EKS cluster. Overview basic EKS setup

![ESK basic setup](/resources/images/kubernetes/eks/eks_basic.png)

* By default, the Kubernetes CNI provides each pod with an IP address from within the cluster's private IP CIDR block, without impacting the host network.
* However, in EKS, the default AWS VPC CNI assigns each pod an IP address from the same subnets used to launch the cluster. This can become a critical issue if the pre-configured CIDR block lacks enough IP addresses for pods or other resources.
* Additionally, the VPC CNI plugin pre-allocates a pool of IP addresses for each node, allowing it to quickly assign addresses to newly created pods.

### Configure VPC

By extend a secondary CIDR block of the VPC and combine CNI custom networking we could avoid this issue

![eks_custom_net](/resources/images/kubernetes/eks/eks_custom_net.png)

- Access to VPC service on AWS console
- Edit and add new CIDR block

> **Note**: We should use 100.64.0.0/10 and 198.19.0.0/16 CIDR block for pod, these CIDR block are non-routable.*

![edit_vpc1](/resources/images/kubernetes/eks/edit_vpc1.png)
![edit_vpc2](/resources/images/kubernetes/eks/edit_vpc2.png)

- Create new subnet on the VPC

![create_subnet_1](/resources/images/kubernetes/eks/create_subnet_1.png)
![create_subnet_2](/resources/images/kubernetes/eks/create_subnet_2.png)
![create_subnet_3](/resources/images/kubernetes/eks/create_subnet_3.png)

### Configure kubernetes

```bash
kubectl set env ds aws-node -n kube-system AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG=true
kubectl describe daemonset aws-node -n kube-system | grep -A5 Environment
```

The result should be same 

```yaml
Environment:
  DISABLE_TCP_EARLY_DEMUX:  false
  ENABLE_IPv6:              false
  Mounts:
    /host/opt/cni/bin from cni-bin-dir (rw)
  Containers:
    Environment:
      ADDITIONAL_ENI_TAGS:                    {}
      AWS_VPC_CNI_NODE_PORT_SUPPORT:          true
      AWS_VPC_ENI_MTU:                        9001
      AWS_VPC_K8S_CNI_CONFIGURE_RPFILTER:     false
      AWS_VPC_K8S_CNI_CUSTOM_NETWORK_CFG:     true
```

Create custom security group and get its ID, or we can use EKS cluster SG after we created it.

```bash
export security_group_id="sg-0e83050f797699c99"
```

Get subnet id was created and use correctly its availability zone

```bash
export new_subnet_id_1="subnet-95f9d9c99f0fff2ff"
export az_1="ap-southeast-1a"
export new_subnet_id_2="subnet-930864a80d468721d"
export az_2="ap-southeast-1b"
export new_subnet_id_3="subnet-999869a89d468799d"
export az_2="ap-southeast-1c"
```

Create K8s manifest and apply

```bash
cat >$az_1.yaml <<EOF
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: $az_1
spec:
  securityGroups:
    - $security_group_id
  subnet: $new_subnet_id_1
EOF
kubectl apply -f $az_1.yaml
```

```bash
cat >$az_1.yaml <<EOF
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: $az_2
spec:
  securityGroups:
    - $security_group_id
  subnet: $new_subnet_id_2
EOF
kubectl apply -f $az_2.yaml
```

```bash
cat >$az_1.yaml <<EOF
apiVersion: crd.k8s.amazonaws.com/v1alpha1
kind: ENIConfig
metadata:
  name: $az_3
spec:
  securityGroups:
    - $security_group_id
  subnet: $new_subnet_id_3
EOF
kubectl apply -f $az_3.yaml
```

After that we can check it

```bash
kubectl get ENIConfigs
```

Same the result

```bash
NAME              AGE
ap-southeast-1a   5m
ap-southeast-1b   5m
ap-southeast-1c   5m
```

> This applies to the VOdds product due it has many services and might use many nodes. Every time you upgrade the cluster, you must re-run these steps

### Automatic configuration with labels AZ - availability-zone-labels

We allow K8s to automatically apply ENIConfig corresponding to worker-node AZ.

```bash
kubectl set env daemonset aws-node -n kube-system ENI_CONFIG_LABEL_DEF=topology.kubernetes.io/zone
```