# Setup Gitlab runner

## Setup necessary tools

### Basic linux tools

```bash
sudo <apt/dnf> update
sudo <apt/dnf> install -y vim git zip jq
``` 

### Docker

## Add docker repository and install docker-ce

* For Ubuntu:

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

* For RHEL

```bash
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## Add current user to docker group. 

```bash
sudo usermod -aG docker $USER
sudo chmod 666 /var/run/docker.sock
```
> Should add the same user which will be used to setup/run gitlab


## Enable docker service

```bash 
systemctl enable --now docker
```

## Install Gitlab runner

- **Step 1**: Add the Gitlab runner repo

```bash
curl -L https://packages.gitlab.com/install/repositories/runner/gitlab-runner/script.deb.sh | sudo bash
```

- **Step 2**: Install Gitlab runner
```bash
sudo apt install gitlab-runner
```

- **Step 3**: Enable service
```bash
sudo systemctl enable --now gitlab-runner
```

## Register Gitlab runner

### Acknowledge

There are three types of runners:

- `shared` - Runner runs jobs from all unassigned projects
- `group` - Runner runs jobs from all unassigned projects in its group
- `specific` - Runner runs jobs from assigned projects

With the below example we will guide to register `shared` runners only, which means that it can pick any CI/CD jobs. 

> If you want to change the type of runners, please read [Configuring Gitlab runners](https://docs.gitlab.com/runner/register/)

Please read [Selecting the executor](https://docs.gitlab.com/runner/executors/README.html#selecting-the-executor) to have a general view of each type of executor


#### Runner tags

For any runner, we can define tag(s) for the current, it's not only for grouping/labeling a runner but it also help runner to pick specific tagged jobs

**For example:**
Let's assume we will have 2 runners (Shell executor runner and Docker executor runner). And we also defined tag for each runner as their executor type

|  Runner Name  | Executor |   Tags  | Run untagged jobs |
| :------------:| :-------:| :-----: | :---------------: |
| Runner docker |  docker  | docker  |       true        |
| Runner shell  |  shell   | shell   |       false       |

For a CI/CD job which is specified a tag `docker` or `shell`, the runner of that tag will take the job and execute it (The other one will ignore the job)

> If a runner has attribute `Run untagged jobs` it will pick untagged jobs to execute.

### Register a runner

>**Note**:
>
>For this documentation, we just focusing on setup the runner to be able to run in a native terminal (aka shell) and docker only.
>
> For other Runner execution types, please read [Runner Executors](https://docs.gitlab.com/runner/executors)


#### Step 1: Get runner register token in Gitlab

- Go to Web UI of gitlab server (https://git.mydomain.com).
- Navigate to `Admin Area` -> `Overview` -> `Runners` and we will see the tab with information to register the new runner to Gitlab as images below

![runner-token](/resources/images/Gitlab/runner-token.png)

>**Note**: Remember to save domain and register token

#### Step 2: Register a Runner

Command to Register a runner simmiliar as below and remember to replace with your correct value

```bash
gitlab-runner register \
  --non-interactive \
  --registration-token ${Runner-register-token} \
  --locked=false \
  --name ${Runner-name} \
  --url ${Gitlab-url} \
  --executor ${Type-of_executor} \
  --tag-list ${List-tag-of-runner} \
  --run-untagged=${Is-Runner-run-untag-jobs}
```

Example register runner which use `shell` executor:

```bash
gitlab-runner register \
  --non-interactive \
  --registration-token <the-runner-token> \
  --locked=false \
  --name "runner-shell" \
  --url https://git.mydomain.com \
  --executor shell \
  --tag-list "shell" \
  --run-untagged="true"
```

When success, we will see the list runner as image below 
![runner-demo](/resources/images/Gitlab/runner-demo.png)

Example register runner which `docker` executor:

```bash
gitlab-runner register \
  --non-interactive \
  --registration-token <the-runner-token> \
  --locked=false \
  --name "runner-docker" \
  --url https://git.mydomain.com \
  --executor docker \
  --docker-image docker:stable \
  --tag-list "docker"
```

#### Step 3: Update Runner config

+ `concurrent` The number of jobs can be run by all runners in parallel. This example will be **7**
+ Enable custom build directory, this will give the ability to change the build directory

```toml
[runners.custom_build_dir]
enabled=true
```

* For Runner using docker pay attention at
  + `image`: Default docker image to use when not specified in job
  + `volumes`: Mount from server to build environment
  + `extra_hosts`: Custom hostname
  + `pull_policy`: Define when to fetch docker runner

##### Runner Shell

```bash
vi /etc/gitlab-runner/config.toml
```

```toml
concurrent=7
check_interval=0
log_level="warning"

[session_server]
session_timeout=1800

[[runners]]
name="runner-shell"
url="https://git.mydomain.com"
token="<the-runner-token>"
executor="shell"
[runners.custom_build_dir]
  enabled=true
```

#### Runner Docker

```bash
vi /etc/gitlab-runner/config.toml
```

```toml
concurrent = 2
check_interval = 0
log_level = "debug"

[session_server]
session_timeout = 1800

[[runners]]
  name = "runner-docker"
  url = "https://git.mydomain.com"
  token = "<the-runner-token>"
  executor = "docker"
  [runners.custom_build_dir]
    enabled = true
  [runners.cache]
    [runners.cache.s3]
    [runners.cache.gcs]
  [runners.docker]
    tls_verify = false
    image = "docker:stable"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false
    volumes = ["/var/run/docker.sock:/var/run/docker.sock", "/builds:/builds", "/builds/cache:/cache", "/builds/gradle:/root/.gradle", "/builds/npm:/root/.npm", "/builds/jspm:/root/.jspm"]
    extra_hosts = [<artifactory-host>, <gitlab-host>]
    pull_policy = "if-not-present"
    shm_size = 0
```