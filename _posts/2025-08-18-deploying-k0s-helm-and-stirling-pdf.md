---
layout: post
title: How to Setup k0s and Host Stirling PDF
categories: [Blog]
tags: [Kubernetes, Helm]
description: How to deploy Stirling-PDF on Kubernetes using k0s and Helm
---


## Intro

I recently wanted to set up a Kubernetes environment on my homelab server to host some open-source applications on my own network. I think having the ability to access applications hosted on my own hardware is great for both accessibility and security. Through professional and personal experience, I've gotten accustomed to deploying and using distributions based on K8s, but for this blog, I decided to try something new and dive into k0s. I will be walking through how to deploy a single-node k0s cluster on a virtual Linux machine and how to host [Stirling PDF](https://github.com/Stirling-Tools/Stirling-PDF), a free and lightweight web-based PDF editor.

There are a couple of key differences that set k0s apart from K8s. I'm not going to cover all of them, but I will share the ones I personally noticed while deploying it. The first thing I noticed was how different the deployment process is. Typically, when you deploy a K8s cluster, you have to provision servers or VMs for the control plane and worker nodes, install a container runtime and Kubernetes components on each node, add a Container Network Interface (CNI) plugin, and more. However, k0s eliminates much of this complexity and the extensive prerequisites by packaging everything into a single binary. To create a single-node cluster, all you have to do is download the k0s shell script, which fetches the binary, and run it in a terminal. For multi-node setups, k0s allows you to connect nodes by SSH'ing into each one and copying the k0s binary; it then handles tasks like control plane deployment, worker joins, and CNI plugin installation automatically.

Another thing I noticed was the small footprint of k0s. Obviously, there's going to be a difference between the resource usage of a multi-node K8s cluster and a single-node k0s instance, but I was still surprised when looking at the memory statistics with the `top` tool. The memory usage of all the k0s components combined was less than 1 GB (including the Stirling PDF deployment). The amount of memory overhead that K8s requires, even for a single node, is significantly higher, with the minimum recommendation for a control plane being over 2 GB. This tiny footprint is great for small cluster use cases but might lack the scalability needed for larger production or enterprise solutions.
## Prerequisites

This guide assumes you already have a server or virtual machine ready to deploy k0s. In my deployment, I was running a `Ubuntu 24.04 server` VM on Proxmox hosted on my homelab. The VM was given a static IP from my router and was provisioned with 32GB of RAM and 4 vcpus (which is overkill for this guide but great for future projects). 
## Setup

Lets jump into my step-by-step process of how to deploy and configure k0s. 

### Installing k0s

First, I downloaded the shell script used to download the k0s binary from the k0s website. 

```shell
 curl -sSLf https://get.k0s.sh | sudo sh
```

Afterwards, install k0s in single-node mode and as a service.

```shell
sudo k0s install controller --single
```

Finally, run this command to start the cluster.

```shell
sudo k0s start
```

Great! Your k0s cluster should now be running on your system. To check that everything ran smoothly run this command to view the health of the cluster.

```shell
sudo k0s kubectl get node
```

The output should look something like this:

```shell
NAME   STATUS   ROLES           AGE   VERSION
k0s    Ready    control-plane   19h   v1.33.3+k0s
```

Note the STATUS column which states that the node is "Ready". If your node has the same status, then you are good to move on to the next steps. 

### Installing Helm

The next part of this tutorial is installing a package manager for Kubernetes called Helm. Helm simplifies the deployment and management of applications on Kubernetes clusters by providing ways to configure, install, and upgrade applications. We are going to use Helm later in this guide to deploy the Stirling-PDF application. 

First you want to install Helm by downloading the install script from Helm's online github repository. 

```shell
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
```

Change the script to executable and run it. This will download the Helm binary locally to your machine.

```shell
chmod 700 get_helm.sh
./get_helm.sh
```

Nice, now you got Helm installed. In order for Helm to connect to your k0s cluster, you need to specify the location of the kubeconfig for k0s. This file can be found at `/var/lib/k0s/pki/admin.conf`. Lets store this somewhere easier to access and create a environment variable out of it so that we don't have to specify this file every time we are running Helm. 

```shell
mkdir ~/.kube/
sudo cp /var/lib/k0s/pki/admin.conf ~/.kube/admin.conf
echo "export KUBECONFIG='$PWD/.kube/admin.conf'" >> ~/.bashrc
source ~/.bashrc
echo $KUBECONFIG
```

The last command above will echo the environment variable you created and if it displays the path of your config file, you are good to go. 

### Deploying Stirling-PDF

Now that you got Helm installed on your system and connected to your k0s cluster we can move on to deploying Stirling-PDF. This process should be relatively automated, but there are a few things that we need to configure to allow access to the application from outside the local machine that will be explained later in this section.

First, lets add the Stirling-PDF repo to Helm so that it knows what application we want to download and deploy. 

```shell
helm repo add stirling-pdf https://stirling-tools.github.io/Stirling-PDF-chart  
helm repo update
```

The second command will update Helm locally to get information about all the charts in the repo including information such as version and metadata.

Before you run the command to deploy the chart onto the k0s cluster, lets create a `values.yaml` file that will change some information about the chart as we deploy it. What we primarily want to do is change the pod label field so that it makes it slightly more easier for us to configure the network connectivity to the pod by using the pod label as reference. 

```yaml
podLabels:
  app: stirling-pdf
```

This is all you have to place in your values.yaml file. Once that is done, we can now go ahead with the chart deployment.

```shell
helm install stirling-pdf stirling-pdf/stirling-pdf-chart --namespace stirling-pdf --create-namespace -f values.yaml
```

This commands installs the stirling-pdf chart into a namespace called "stirling-pdf" which is created if it doesn't exist with the "--create-namespace" option. All the configuration values in the values.yaml file are applied to the chart with "-f values.yaml". 

Once you run this command, there should be output that tells you if the application was deployed successfully or not. Here is what some of that output looks like:

```shell
NAME: stirling-pdf
LAST DEPLOYED: Mon Aug 18 22:34:29 2025
NAMESPACE: stirling-pdf
STATUS: deployed
REVISION: 1
TEST SUITE: None
```

Note the `STATUS` field that says "deployed". 

To check that the deployment is successful and that the pod is running, we can list all the pods in the namespace stirling-pdf.

```shell
sudo k0s kubectl get pods -n stirling-pdf
```

In the output, if the `READY` column says "1/1" and the `STATUS` column says "Running" then the application deployed successfully. 

Now we have to create a service for the application so the application can take incoming connections from outside the cluster. A Kubernetes service allows us to expose a port from the container onto the external cluster's network interface. This provides a stable and reliable way to access the application hosted on the container even through pod redeployments. Lets create a file and name it "stirling-pdf-service.yaml". Here is the information that should be placed in the file with annotations on the important lines.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: stirling-pdf-service
spec:
  # Exposes the service on a static port on all the cluster nodes and allows the service to be reachable outside the cluster
  type: NodePort 
  selector:
    # Selects the pod we want to expose base off of the podLabel key/value pair      we specified in the values.yaml file
    app: stirling-pdf 
  ports:
    - protocol: TCP
    - # The port that the service itself will be exposed inside the cluster
      port: 8080
    - # This is the destination port that the service will forward to. The             service forwards traffic it receives on port: 8080 to the targetPort which       is also the port that the application is listening on
      targetPort: 8080
    - # The port that will be exposed on each cluster node externally
      nodePort: 30001
```

Once this file has been created, we can apply this configuration to the cluster. 

```shell
sudo k0s kubectl apply -f stirling-pdf-service.yaml -n stirling-pdf
```

And like that you should be done! Go ahead and connect to the application on your browser using the IP address of the machine and the port. 

To update the chart if a new version of it comes out, run these two commands.

```shell
helm repo update  
helm upgrade stirling-pdf stirling-pdf/stirling-pdf-chart --namespace stirling-pdf --reuse-values
```

The first command actually checks if any updates are available for the chart and the second command updates the chart while reusing the configuration we added with the values.yaml file. 


## Conclusion

Now you understand how to deploy a k0s cluster, install and use Helm, and run your own instance of Stirling-PDF on your local environment! I hope this guide has helped you get a better grasp of Kubernetes and how easy it actually is to set up yourself. In future blogs, I plan on detailing how to secure your environment through stuff like firewall rules, cluster network configurations, pod security polices, service account RBAC, and more. 
