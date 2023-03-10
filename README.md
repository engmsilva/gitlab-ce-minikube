[![Custom badge](https://img.shields.io/endpoint?url=https%3A%2F%2Fsolitary-king-440f.eng-msilva.workers.dev%2F)](https://hub.docker.com/r/gitlab/gitlab-ce)

# Deploying GitLab CE on Kubernetes with Minikube

<p align="center">
  <img src="./static/gitlab-kubernetes.png" width=50% height=50%>
</p>

This guide serves as a reference for setting up [GitLab CE](https://hub.docker.com/r/gitlab/gitlab-ce) in a local Kubernetes development environment. In this guide, we’ll be using [minikube](https://minikube.sigs.k8s.io/) as it is the accepted standard.

## Local Development Environment Requirements

To deploy the k8s manifest files from this repository in the local development environment, it is necessary to install the following applications:

- [Docker v23.0.1](https://docs.docker.com/engine/install/ubuntu/)
- [Minikube v1.29.0](https://minikube.sigs.k8s.io/docs/start/#:~:text=1-,Installation,-Click%20on%20the)
- [Kubectl v1.26.1](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)

The proposed development environment was configured and tested on **Ubuntu 22.04.2 LTS (Jammy Jellyfish)**.

## Create Cluster

For the implementation of this cluster, a CPU with 4 cores and 10 GB of RAM will be configured, according to the [recommended resources](https://docs.gitlab.com/ee/install/requirements.html#storage) for running GitLab.

After installing the `Docker`, `Minikube` and `Kubectl` applications, from a terminal with administrator access (but not logged in as root), run to create a cluster locally:

```
minikube start --cpus 4 --memory 10240
```

Enable ingress addon to manage external access to cluster services:

```
minikube addons enable ingress
```

Clone the repository:

```
git clone https://github.com/engmsilva/gitlab-ce-minikube.git
```

Run the deployment on the cluster:
> **Notice**: I recommend running [the local certificate setup steps](#locally-trusted-development-certificates) before deploying from GitLab if using https.

```
cd gitlab-ce-minikube
kubectl apply -f ./gitlab
```

Wait until the GitLab `Pod` is in `Running` status:

```
kubectl get pods --namespace=gitlab

NAME                      READY     STATUS    RESTARTS   AGE
gitlab-5cf5f7bc4f-ztns5   1/1       Running   0          3h31m
```

Check the local IP that is being routed to the cluster services:

```
kubectl get ingress --namespace=gitlab

NAME             CLASS     HOSTS          ADDRESS        PORTS     AGE
gitlab-ingress   nginx     gitlab.local   192.168.49.2   80        22h
```

Add the IP from the **ADDRESS** column in the `/etc/hosts` file to resolve name to address [http://gitlab.local](http://gitlab.local):

```
192.168.49.2 gitlab.local
```

Restart the networking service:

```
sudo systemctl restart NetworkManager.service
```

## Login

After starting the `Pod`, you can visit [http://gitlab.local](http://gitlab.local). It may take a while for the `Pod` to start responding to queries.

Visit the GitLab URL and sign in with the `root` username and password obtained through the following steps.

Access the `Pod` shell:

```
kubectl exec --stdin --tty gitlab-5cf5f7bc4f-ztns5 -n gitlab -- /bin/bash
```

In the `Pod` shell run the command:

```
grep 'Password:' /etc/gitlab/initial_root_password
```

**note:** The password file will be automatically deleted in the first reconfigure run after 24 hours.


## Locally-trusted Development Certificates


> **Warning**: The certificate configuration must be done before executing the **SSH** settings to avoid errors in the activation and deactivation steps of the **ingress addon**. When disabling and enabling the **ingress addon** after configuring **SSH**, the following error occurs:

```
minikube addons disable ingress
❌  Exiting due to IF_SSH_AUTH: run callbacks: running callbacks: [NewSession: new client: new client: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain]
💡  Suggestion: Your host is failing to route packets to the minikube VM. If you have VPN software, try turning it off or configuring it so that it does not re-route traffic to the VM IP. If not, check your VM environment routing options.
📘  Documentation: https://minikube.sigs.k8s.io/docs/handbook/vpn_and_proxy/
🍿  Related issue: https://github.com/kubernetes/minikube/issues/3930
```


To automatically create and install a local CA at the system root and generate locally trusted certificates, [mkcert](https://github.com/FiloSottile/mkcert) will be used.

mkcert installation:

```
mkdir cert
cd cert
curl -JLO "https://dl.filippo.io/mkcert/latest?for=linux/amd64"
chmod +x mkcert-v*-linux-amd64
sudo cp mkcert-v*-linux-amd64 /usr/local/bin/mkcert
```

Generate local certificate:

```
mkcert -install
```

The certificate will be saved in the path **/home/user/.local/share/mkcert**. The path can also be seen by running the command:

```
mkcert -CAROOT
```

> **Warning**: the `rootCA-key.pem` file that mkcert automatically generates gives complete power to intercept secure requests from your machine. Do not share it.

Generate local development trust certificates:

```
mkcert gitlab.local "*.gitlab.local"
```

Create TLS secret which contains custom certificate and private key:

```
kubectl -n kube-system create secret tls mkcert --key gitlab.local+1-key.pem --cert gitlab.local+1.pem
```

Configure ingress addon:

```
minikube addons configure ingress
-- Enter custom cert(format is "namespace/secret"): kube-system/mkcert
```

Enable ingress addon (disable first when already enabled):

```
minikube addons disable ingress
minikube addons enable ingress
```

Update localhost certificates:

```
sudo update-ca-certificates
```

Restart `Docker` to load certificates from localhost:

```
sudo systemctl restart docker
```

After restarting `Docker` it is necessary to start the `minikube` cluster container again:

```
minikube start -p minikube
```

Verify if custom certificate was enabled:

```
kubectl -n ingress-nginx get deployment ingress-nginx-controller -o yaml | grep "kube-system"
```

Apply patch on **gitlab deployment** for GitLab to listen behind a reverse proxy with https:

```
kubectl patch deployment gitlab --patch "$(cat ./patch/00-https-patch.yaml)" -n gitlab
```

Go to [https://gitlab.local](https://gitlab.local) and the browser should recognize the local domain as secure.

![alt text](./static/gitlab_https.png)


## Cloned Repository via SSH

The 3 steps that need to be followed in order to be able to clone GitLab repositories:

- Configure the Ingress Controller to access GitLab via SSH;
- Configure an SSH key pair on the clone destination;
- Configure SSH key in GitLab user preferences.

### Configuring Ingress Controller for External GitLab Shell Access via SSH

The GitLab Shell component requires TCP traffic to pass through on port 22 (by default; this can be changed). Ingress does not directly support TCP services, so some [additional configuration is necessary](https://minikube.sigs.k8s.io/docs/tutorials/nginx_tcp_udp_ingress/).

Add the GitLab Shell service listening on port 22 to the nginx configMap:

```
kubectl patch configmap tcp-services -n ingress-nginx --patch '{"data":{"22":"gitlab/gitlab:22"}}'
```

Apply the patch on the nginx controller so that it listens on port 22:

```
kubectl patch deployment ingress-nginx-controller --patch "$(cat ./patch/01-ssh-controller-patch.yaml)" -n ingress-nginx
```

### Generate an SSH key pair

Before you create a key pair, see if a key pair already exists.

```
ls /home/user/.ssh
```

If not, you need to generate a key. Run `ssh-keygen -t` followed by the key type and an optional comment. This comment is included in the created .pub file.

For 2048-bit RSA:

```
ssh-keygen -t rsa -b 2048 -C "<comment>"
```

A public key and a private key will be created. See the GitLab documentation for more [SSH key configuration options](https://docs.gitlab.com/ee/user/ssh.html) .

### Configure SSH key in GitLab User Preferences

Get the SSH public key value by running the command:

```
cat /home/user/.ssh/id_rsa.pub
```

Copy SSH key to clipboard.

Log into GitLab and click on your account preferences.

Click the SSH Keys link and paste the copied value into the text field.

Set an expiration date, and then click the blue button to persistently add the GitLab SSH key.

![alt text](./static/gitlab-ssh-key-conf.png)

## Gitlab Registry

With the [GitLab Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/), every project can have its own space to store Docker images.

To enable Container Registry on your GitLab instance perform the following steps.

Configure the Ingress TCP service to listen on **GitLab Registry** port **5005**:


```
kubectl patch configmap tcp-services -n ingress-nginx --patch '{"data":{"5005":"gitlab/gitlab:5005"}}'
```

Apply the patch on the nginx controller so that it listens on port **5005**:

```
kubectl patch deployment ingress-nginx-controller --patch "$(cat ./patch/02-registry-controller-patch.yaml)" -n ingress-nginx
```

Apply patch on **gitlab deployment** to enable and configure GitLab Registry:

```
kubectl patch deployment gitlab --patch "$(cat ./patch/03-registry-deployment-patch.yaml)" -n gitlab
```
See the [GitLab Container Registry](https://docs.gitlab.com/ee/user/packages/container_registry/) documentation to learn how to access your project's registry.

### Docker Image Upload to GitLab Registry

For this example, the **app-hello-world** project located inside the examples directory will be used.

Before starting, a [group](https://docs.gitlab.com/ee/user/group/) and a [project](https://docs.gitlab.com/ee/) have already been created in this group. For this example, the group **group-hello-world** and the project **project-hello-world** were created.

Generate a docker image from the **app-hello-world** project's Dockerfile:

```
docker build -t registry.gitlab.local/group-hello-world/project-hello-world/app-hello-world ./examples/app-hello-world
```

Log in to GitLab Registry:

```
docker login registry.gitlab.local
```

Upload the image to the GitLab Registry:

```
docker push registry.gitlab.local/group-hello-world/project-hello-world/app-hello-world
```
Access the Container Registry of the **project-hello-world** project to view the uploaded image.

![alt text](./static/gitlab-registry.png)



