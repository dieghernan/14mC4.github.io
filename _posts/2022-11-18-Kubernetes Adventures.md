---
title: "Kubernetes Adventures"
excerpt_separator: "<!--more-->"
categories:
  - Blog
tags:
  - Gitlab
  - Kubernetes
  - Docker
---

```
Charles Goodling 
July 2021
Installing K3s w/ Ingress Controller, Gitlab, into CI/CD Workflow on Hyper-V
```

#### Background:
I was tasked with getting a local replica of my environment setup to learn and develop locally. I am running a Windows 10 Pro PC w/ an Intel(R) Core(TM) i9-9900K CPU @ 3.60GHz, 3601 Mhz, 8 Core(s), 16 Logical Processor(s) and 64 GB of RAM. 
Prerequisites: 
- Choco (https://chocolatey.org/install)
- Hyper-V (https://docs.microsoft.com/en-us/virtualization/hyper-v-on-windows/quick-start/enable-hyper-v)
- Alternate (VirtualBox)

#### Step One: 

Setup 3 VMs using Ubuntu 20.04
VM1: k3s-master w/ 16GB RAM and 5 CPUs, External Switch and 20GB Storage
VM2: k3s-node01 w/ 4GB RAM and 4 CPUs, External Switch and 20GB Storage
VM2: k3s-node02 w/ 4GB RAM and 4 CPUs, External Switch and 20GB Storage
NOTE: Im using Dynamic Memory Allocation as well. I may disable at later date

#### Step Two: 

Setup all three servers w/ OpenSSH and defaults. Go ahead note down the IPs. We will need those later. You can setup the usernames and passwords whatever you want. Just name sure to notate it properly. For the sake of simplicity Im using the master/toor and node/toor for my cluster. I will change later but, I just want to be able to setup fast. 

Personal Notes for Adding to DNS Entries later:
```
k3s-master 192.168.1.182
k3s-node02 192.168.1.183
k3s-node01 192.168.1.184
```

#### Step Three:
I am going to SSH into my 3 Servers so I can just right click to paste things and do things a little smoother in PowerShell. 
```
ssh username@ipaddress
```

#### Step Four:
Let’s go ahead and push out the updates to all three servers and get them setup.
```
sudo apt update
sudo apt -y upgrade && sudo systemctl reboot
```
For the sake of multiple commands we can go ahead and enter into a super user state
```
sudo -i

##Docker APT Repo:
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"

##Installing Docker CE 
sudo apt update
sudo apt install docker-ce -y

## Starting up Docker
sudo systemctl start docker
sudo systemctl enable docker

##Check Status
sudo systemctl status docker

##Exit out of Superuser mode
exit
## Adding User to have docker perms. 
sudo usermod -aG docker ${USER}
newgrp docker
```

#### Step Five:
we could have done this earlier but, let’s go ahead and add the following to the DNS records. You can add locally on each machine in the /etc/hosts file by running. I am jumping ahead and adding the stuff for gitlab since we will need something to resolve since this is a local instance and not public. You could however setup a domain name and all that. I have all mine setup in my Pi-Hole. 

```
sudo nano /etc/hosts
```

192.168.1.182 gitlab.example.com
192.168.1.182 registry.example.com
192.168.1.182 minio.example.com
192.168.1.182 example.com
192.168.1.182 k3s-master
192.168.1.184 k3s-node01
192.168.1.183 k3s-node02

#### Step Six:
This step is specifically for the Master Node ONLY. We will deploy k3s without Traefik as we will be using NGINX Ingress

```
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--no-deploy traefik" sh -s - --docker
```

Let’s verify it lanched successfully
```
sudo systemctl status k3s
```

You can also see some docker pods that got spun up for the backend. Since we added the user to docker group earlier we don’t need to use “sudo”
```
docker ps
```

Let’s look at the cluster now that it’s deployed

```
sudo kubectl get nodes -o wide
sudo kubectl cluster-info
```

You can see that the kubectl is polling the local host for the config file. We can change that later but, for now we are local on the master server so it’s fine. 
 
#### Step Seven:
We need to allow 6443 & 443 through the firewall between master and worker nodes. We need to do this on all nodes. 

```
sudo ufw allow 6443/tcp
sudo ufw allow 443/tcp
```

#### Step Eight:
We are back on the master node. We need to pull the node token so we can join the other two nodes. 
```
master@k3s-master:~$ sudo cat /var/lib/rancher/k3s/server/node-token
K1087539b4acf0c53fb8acf454a0f4505418c702869f0f04c2a1cd04d53aa8ab595::server:4784e3bf7297bbe47506b2bcaedc9d67
master@k3s-master:~$
```

#### Step Nine:

We can join the worker nodes to the cluster by running the following command below. Input your token and IP.

```
curl -sfL http://get.k3s.io | K3S_URL=https://192.168.1.185:6443 K3S_TOKEN=K1002ae728c3fb10a97442676e52100fec5fc13d2a98ea2201657961e118f680776::server:214fbeccadab59d864bca9b1c14a1bee sh -s - --docker
```
##### Example
```
curl -sfL http://get.k3s.io | K3S_URL=https://192.168.1.182:6443 K3S_TOKEN=K1087539b4acf0c53fb8acf454a0f4505418c702869f0f04c2a1cd04d53aa8ab595::server:4784e3bf7297bbe47506b2bcaedc9d67 sh -s - --docker
```

You can run the “get nodes” command and see that your other nodes will be joined into the cluster. 
 
#### Step Ten:
Install helm and kubectl for the workstation you plan to work off. (https://helm.sh/). It’s not often you will have access to the actually master node so you can use kubectl and helm locally to manage the cluster. I will be using choco on Windows to install 
```
choco install Kubernetes-cli
choco install Kubernetes-helm
```

#### Step Eleven:
We need to copy the the config information from “/etc/rancher/k3s/k3s.yaml” to working PC in “~/.kube/config” You can use whatever method you want to get it there. I will manually create the folders and files and paste the info into it. 

##### Example:
 
You can verify your connect by running 
```
kubectl get pods -o wide
```

#### Step Twelve:
Let’s go ahead add the helm stuff needed to deploy the ingress nginx controller. Details can be found on their Github (https://github.com/kubernetes/ingress-nginx/) . Ingress-Nginx acts as a load balancer and an ingress. 
```
helm repo add stable https://charts.helm.sh/stable
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```
The documentation for ingress-nginx wants us to open up 8443 so, I went ahead and pushed it on all nodes. 
```
sudo ufw allow 8443/tcp
```
We can go back to our normal working pc and go ahead and deploy the controller via helm using the commands. 
```
helm install ingress-nginx ingress-nginx/ingress-nginx
```
We can now just run a few commands to see if it’s been deployed successfully.
```
kubectl get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}'

kubectl exec -it POD_NAME -- /nginx-ingress-controller --version
```

##Example: 
```
PS C:\Users\Yrroth> kubectl get pods -l app.kubernetes.io/name=ingress-nginx -o jsonpath='{.items[0].metadata.name}'
ingress-nginx-controller-7d98fb5bd-cbsmq

PS C:\Users\Yrroth> kubectl exec -it ingress-nginx-controller-7d98fb5bd-cbsmq -- /nginx-ingress-controller --version
-------------------------------------------------------------------------------
NGINX Ingress controller
  Release:       v0.47.0
  Build:         7201e37633485d1f14dbe9cd7b22dd380df00a07
  Repository:    https://github.com/kubernetes/ingress-nginx
  nginx version: nginx/1.20.1

-------------------------------------------------------------------------------

PS C:\Users\Yrroth> kubectl --namespace default get services -o wide -w ingress-nginx-controller
NAME                       TYPE           CLUSTER-IP     EXTERNAL-IP                                 PORT(S)                      AGE     SELECTOR
ingress-nginx-controller   LoadBalancer   10.43.22.150   192.168.1.182,192.168.1.183,192.168.1.184   80:31530/TCP,443:30924/TCP   4m29s   app.kubernetes.io/component=controller,app.kubernetes.io/instance=ingress-nginx,app.kubernetes.io/name=ingress-nginx
```
#### Step Thirteen:
We need to setup our local DNS to resolve the domain that we chose. If you are using an external domain then steps may vary. I am just doing a local test environment so I setup my DNS on my local Pi-Hole since my network resolves to it. I could also add all this to each PC’s hosts file but, I don’t wanna do that. You need to create the following entries using your One of your Ingress IP as defined previously. As seen below you need an entry for Gitlab, Minio, Registry and the domain itself. 
 

#### Step Fourteen:
we can now install gitlab via helm into our minikube Kubernetes cluster. Run the following command below. You could add flags to specify a namespace for the gitlab install which would be best practice but, since we deployed via helm I can just run helm uninstall gitlab and clean it up anyways. But, it’s good to separate things into their own namespace. For now it’s just a local test environment and I’ll update as I learn more. We will just use the below: I chose to create a file called gitlab-values.yaml. I know it’s the same one I used for my Minikube deployment but, it works. 

```
# values-minikube.yaml
# This example intended as baseline to use Minikube for the deployment of GitLab
# - Services that are not compatible with how Minikube runs are disabled
# - Configured to use 192.168.99.100, and nip.io for the domain

# Minimal settings
global:
  ingress:
    configureCertmanager: false
    class: "nginx"
  hosts:
    domain: example.com
    externalIP: 192.168.1.182
  shell:
    # Configure the clone link in the UI to include the high-numbered NodePort
    # value from below (`gitlab.gitlab-shell.service.nodePort`)
    port: 32022
# Don't use certmanager, we'll self-sign
certmanager:
  install: false
# Use the `ingress` addon, not our Ingress (can't map 22/80/443)
nginx-ingress:
  enabled: false
# Map gitlab-shell to a high-numbered NodePort cloning over SSH since
# Minikube takes port 22.
gitlab:
  gitlab-shell:
    service:
      type: NodePort
      nodePort: 32022
# Provide gitlab-runner with secret object containing self-signed certificate chain
gitlab-runner:
  certsSecretName: gitlab-wildcard-tls-chain
```

Let’s run the Helm specific stuff. 

```
helm repo add gitlab https://charts.gitlab.io/
helm repo update
helm upgrade --install -f .\gitlab-values.yaml gitlab gitlab/gitlab --timeout 600s
```

We can watch it while it’s initializing all the pods. I want to see “-o wide” just so I can see how the pods are spreading out across my cluster for proof of concept. My master node has the most power so it’ll hold most the weight but, I plan to even them out soon and see if I can get gitlab to deploy with 8GB Ram each cluster and share it across them fully vs Master holding most the weight. 
```
kubectl get pods -o wide -A -w
```

#### Step Fourteen:
Now that we have access to the dashboard we got to get the initial login information. You can do that by parsing the Base64 Info.
```
kubectl get secret gitlab-gitlab-initial-root-password -ojsonpath='{.data.password}'
```
Should output something similar to this below 
```
WXRWcjJVUjRBd3BJTVhoM0JEbkdERjhkaDg4SkN4WERNTEl5ZVJqTkh6YjVTWjVJcnhycUhqZTJGT3BqdGMzag==
```
Now you want to decrypt the output into the actual password:
```
[System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String('INSERT_OUTPUT'))
```
##### EXAMPLE:
```
[System.Text.Encoding]::ASCII.GetString([System.Convert]::FromBase64String('WXRWcjJVUjRBd3BJTVhoM0JEbkdERjhkaDg4SkN4WERNTEl5ZVJqTkh6YjVTWjVJcnhycUhqZTJGT3BqdGMzag=='))
```
Should get an output similar to this:
```
YtVr2UR4AwpIMXh3BDnGDF8dh88JCxXDMLIyeRjNHzb5SZ5IrxrqHje2FOpjtc3j
```
You will use “root” and the password generated to access your instance.

Adding Gitlab Runners w/ Self Signed Stuff
``` 
root@k3s-master:/srv/gitlab-runner/config# ls
gitlab-runner_.deb
root@k3s-master:/srv/gitlab-runner/config# SERVER=gitlab.nomadicworx.tech
root@k3s-master:/srv/gitlab-runner/config# PORT=6443
root@k3s-master:/srv/gitlab-runner/config# CERTIFICATE=/etc/gitlab-runner/certs/${SERVER}.crt
root@k3s-master:/srv/gitlab-runner/config# sudo mkdir -p $(dirname "$CERTIFICATE")
root@k3s-master:/srv/gitlab-runner/config# openssl s_client -connect ${SERVER}:${PORT} -showcerts </dev/null 2>/dev/null | sed -e '/-----BEGIN/,/-----END/!d' | sudo tee "$CERTIFICATE" >/dev/null
root@k3s-master:/srv/gitlab-runner/config# gitlab-runner register --tls-ca-file="$CERTIFICATE"
```

Enter the GitLab instance URL (for example, https://gitlab.com/):
https://gitlab.nomadicworx.tech/
Enter the registration token:
RqeA7QIzslbI1YgPmDOLyKB1gFOOOFoTXu8tLpgzIMMHSTyDkQUyspU6tw6ZWviX
Enter a description for the runner:
[k3s-master]: test
Enter tags for the runner (comma-separated):
test
Registering runner... succeeded                     runner=RqeA7QIz
Enter an executor: docker-ssh, shell, ssh, virtualbox, custom, docker, parallels, docker+machine, docker-ssh+machine, kubernetes:
docker
Enter the default Docker image (for example, ruby:2.6):
ruby:2.6
Runner registered successfully. Feel free to start it, but if it's running already the config should be automatically reloaded!
root@k3s-master:/srv/gitlab-runner/config#

References:

https://helm.sh/docs/intro/install/
https://docs.gitlab.com/charts/installation/deployment.html
https://computingforgeeks.com/install-kubernetes-on-ubuntu-using-k3s/
https://digitalis.io/blog/kubernetes/k3s-lightweight-kubernetes-made-ready-for-production-part-1/
https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/
https://docs.gitlab.com/charts/installation/deployment.html
https://www.devops.buzz/public/kubernetes/cheat-sheet
