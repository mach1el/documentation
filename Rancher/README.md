# Setup Rancher server for kubernetes management

## Run Rancher on docker

```yaml
version: '3.7'

networks:
  rancher:
    driver: bridge

volumes:
  rancher_data: {}

x-op-restart-policy: &restart_policy
  restart: unless-stopped

services:
  rancher:
    hostname: rancher
    image: rancher/rancher:latest
    container_name: rancher
    privileged: true
    networks:
      - rancher
    ports:
      - 80:80
      - 443:443
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /etc/timezone:/etc/timezone:ro
      - rancher_data:/var/lib/rancher
    <<: *restart_policy
```

Execute `docker-compose`

```bash
docker-compose up -d
```

## Access to Rancher server

At first, Rancher will use the generated password by itself; you can grep it with the Docker logs container. Do SSH access to the Rancher server and get the bootstrap password.

```bash
docker logs  rancher  2>&1 | grep "Bootstrap Password:"
```
The result will show like this example:

**`2023/04/20 14:33:30 [INFO] Bootstrap Password: 7g5s87qc6tfqzzlv92qkcvzcc9s4llxkpqvxvrbzwrz2cj8fp6cflq`**

![rancher_login](/resources/images/Rancher/rancher_login.png)

After logged in we able to change the password for `admin` user as well.

![rancher_change_password](/resources/images/Rancher/rancher_change_password.png)

## Cloud credential

To access your cloud resources (AWS, GCP, Azure,...) on Rancher, we might add your own cloud credential into Rancher. To do that, let's click the menu on the top left corner and choose **`Cluster Management`**. After moving to a new page, you can see the option **`Cloud Credentials`** on the left side; choose it, then **`Create`**. Fill in the form, then create (credentials should end up as `access key` and'secret key`).

![cloud_credentials](/resources/images/Rancher/cloud_credentails.png)
![create_credentails](/resources/images/Rancher/create_credentails.png)

## Import existing cluster

Access to the homepage of Rancher, we have the option to `create` or `import` for a new cluster. After we had successfully provisioned a Kubernetes cluster we can import that cluster to the Rancher.

![rancher_homepage](/resources/images/Rancher/rancher_homepage.png)
![register_cluster](/resources/images/Rancher/register_cluster.png)
![available_cluster](/resources/images/Rancher/available_cluster.png)
![cluster_dashboard](/resources/images/Rancher/cluster_dashboard.png)