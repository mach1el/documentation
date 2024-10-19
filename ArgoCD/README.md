# Overview

In this stage, we're going to use CI/CD pipeline to deploy application to the EKS cluster. We are using [ArgoCD](https://argo-cd.readthedocs.io/en/stable/) tool for manage and automate the deployment of applications and infrastructure changes to Kubernetes clusters using Git as the source of truth. We dive step by step to setup the ArgoCD and explore its usage
 
![cicd-overview](/resources/images/ArgoCD/gitlab-cicd-k8s.drawio.png)

---

# Setup 

## Install argocd cli on gitlab-runner

Ssh to the `gitlab-runner` server, and execute following commands

```bash
curl -sSL -o /usr/bin/argocd https://github.com/argoproj/argo-cd/releases/download/v2.4.26/argocd-linux-amd64
chmod +x /usr/bin/argocd
```
> **Note**: must run as root user

## Create service account for ArgoCD

Before install ArgoCD on the cluster, we are going to create service account with appropriate permissions to manage resource on the cluster.

```bash
kubectl create namespace argocd
kubectl create serviceaccount argocd-manager -n argocd
kubectl create clusterrolebinding argocd-manager-binding --clusterrole=cluster-admin --serviceaccount=argocd:argocd-manager
```

## Getting cluster information

1. Get cluster endpoint

```bash
aws eks describe-cluster --name <cluster-name> --query "cluster.endpoint" --output text
```

2. Get cluster CA certificate

```bash
aws eks describe-cluster --name <cluster-name> --query "cluster.certificateAuthority.data" --output text
```

3. Create service account token

```bash
kubectl create token argocd-manager -n argocd
```

## Install ArgoCD on the cluster

1. Create `values.yaml` file to use custom value

```yaml
server:
  service:
    type: LoadBalancer
    annotations:
      service.beta.kubernetes.io/aws-load-balancer-backend-protocol: "http"
  ingress:
    enabled: true
    ingressClassName: "nginx"
    hostname: "argocd.example.com"
    annotations:
      nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

configs:
  cm:
    url: "https://argocd.example.com"

controller:
  replicas: 1
```

2. Install ArgoCD with helm chart

```bash
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update
helm install -n argocd argocd argo/argo-cd -f values.yaml --create-namespace
```

3. Get initial admin password

```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

4. Access portal

Open browser and forward to `argocd.example.com` at it configured in `values.yaml` before

![argocd_portal](/resources/images/ArgoCD/argocd_portal.png)

```text
username: admin
password: THE_PASSWORD_YOU_GOT_ABOVE
```

5. Update the admin password

![argocd_portal_password](/resources/images/ArgoCD/argocd_portal_password.png)


## Add cluster to the ArgoCD server

Get back to gitlab-runner instance where already installed the argocd cli 

1. Login into server

```bash
argocd login argocd.example.com --username=admin --password=<admin-password> --insecure --grpc-web
```

2. Add cluster

```bash
argocd cluster add '<cluster-context>' --name MyCluster
```

Currently, we are working on EKS cluster, the `<cluster-context>` has a format like this `arn:aws:eks:ap-northeast-1:315865776134:cluster/MyCluster`

## Add repository

ArgoCD will manage and sync data from our helm chart template. Hence, it should connect to the repository that store application's template.

Credentials can be configured using Argo CD CLI:

```bash
argocd repo add https://github.com/argoproj/argocd-example-apps --username <username> --password <password>
```

or UI:

1. Navigate to `Settings/Repositories`

![repo-add-overview](/resources/images/ArgoCD/repo-add-overview.jpg)

2. Click `Connect Repo using HTTPS` button and enter credentials

![repo-add-https](/resources/images/ArgoCD/repo-add-https.jpg)

> Note: username in screenshot is for illustration purposes only , we have no relationship to this GitHub account should it exist.

3. Click `Connect` to test the connection and have the repository added

![connect-repo](/resources/images/ArgoCD/connect-repo.jpg)

## Synchronize helm chart

When we implement new features and deploy our applications, the container version (image tag) must be updated. This requires not only updating the image tag in the Helm chart but also automating the process of creating and merging the code changes into the Git repository. Argo CD will then detect these changes and automatically sync the application with the updated Helm chart in Git.

To streamline this process:

1. **Automate the update** of the container image tag in the Helm chart.
2. **Automatically create and merge** the required changes into the Git repository.
3. **Argo CD will sync** the changes, ensuring the application is deployed with the updated container version.

```yaml
...
- |
  chart_image_tag=$(sed -n '0,/tag: /p' ./env/$ENV/$CI_PROJECT_NAME-values.yaml | grep "tag: ${IMG_TAG}")
  if [ "$chart_image_tag" == "$IMG_TAG" ]; then
    echo "==> Version is up to date. Skipping update chart"
  else
    echo "==> Update image tag"
    cd "$CHART_REPO"
    git checkout -b update-image-tag-${CI_COMMIT_SHORT_SHA}
    sed -i '0,/tag:.*/s//tag: '"${IMG_TAG}"'/' ./env/$ENV/$CI_PROJECT_NAME-values.yaml
    git config user.email "support@inceptionlabs.com.vn"
    git config user.name "argocd"
    git commit -am "Update image tag to ${IMG_TAG}"
    git push origin update-image-tag-${CI_COMMIT_SHORT_SHA}
    glab auth login --hostname gitlab.jayeson.com.sg --token $GITLAB_TOKEN
    glab mr create \
      --target-branch master \
      --title "Update image tag to ${IMG_TAG}" \
      --assignee argocd \
      --reviewer argocd \
      --remove-source-branch \
      --fill \
      --yes
    glab mr merge --remove-source-branch --auto-merge --yes
    cd ..
  fi
...
- >
  argocd login $ARGOCD_SERVER --username $ARGOCD_USERNAME --password $ARGOCD_PASSWORD --insecure
  argocd app sync $RELEASE_NAME
  argocd app wait $RELEASE_NAME --sync
```

This conditional will check if the image version is up to date with the tag version whenever we deploy the application; if there is no change, the pipeline doesn't push and merge the code; otherwise, it will automatically create a merge request and update the new app version.

### Setup glab for auto merge progress

For Ubuntu/Debian

```bash
# Add WakeMeOps repository
curl -sSL "https://raw.githubusercontent.com/upciti/wakemeops/main/assets/install_repository" | sudo bash

# Install glab
sudo apt install glab
```

### Add variables

In atuo merge progress we need authorization for its actions, at above we already got token from user argocd, we will use it and setup variable for each project. These import variables need to add

- *GITLAB_TOKEN*
- *ARGOCD_USERNAME*
- *ARGOCD_PASSWORD*

1. Go to your GitLab project.
2. In the left-hand menu, navigate to **Settings** > **CI/CD**.
3. Scroll down to the **Variables** section and click **Expand**.
4. Click **Add Variable**.
5. In the **Key** field, enter `GITLAB_TOKEN`.
6. In the **Value** field, paste the token you generated in Step 1.
7. Set the **Type** to **Environment variable**.
8. (Optional) To keep the token secure:
   - Set **Protected** to `On` (only available in protected branches).
   - Set **Masked** to `On` (so the token is hidden in job logs).
9. Click **Add variable** to save the `GITLAB_TOKEN`.

> do the same with ARGOCD variables

## Version controlling

Version controlling in Kubernetes (K8s) deployments primarily revolves around managing and tracking changes to the configuration and deployment manifests used to define the desired state of Kubernetes resources, such as pods, services, and deployments. 

* Image tagging
  
  We are using image tagging with format `<env>-<short of RSA commit>`, eg. `dev-9bf39c2c`, `stag-2cf23p01`. The image will be stored in ECR for last 5 times, apply for *`dev`* and *`staging`* environment for *`prod`* the format will be `<app version>-<env>`, for `app version` it's exported from `build.gradle` in the project, eg. `2.2.6-prod`

* Tagging for rollback

  Tags are essential when performing rollbacks. By using specific image tags, Kubernetes allows you to easily redeploy a previous version if the current deployment fails.
  For example, if `2.1.0-prod` causes an issue, you can quickly update the deployment to use `2.0.0-prod` by modifying the image tag

## Create AWS ECR repository

To manage our Docker container, we create a repository in AWS ECR with the same name as the project name before running CI/CD