Chicago Linode Workshop
======================

# About

Package of template files, examples, and illustrations for the Chicago Linode Workshop.

# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.
- Sample kubernetes deployment files for starting an application on an LKE cluster.


### Kubectl commands:
- installing kubectl:
```
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubectl
```
- Exporting terraform output to a kubeconfig file:
```
 export KUBE_VAR=`terraform output kubeconfig` && echo $KUBE_VAR | base64 -di > lke-cluster-config.yaml
```
- Adding the contents of the kubeconfig file to a env variable:
```
export KUBECONFIG=lke-cluster-config.yaml
```
- Applying a kubernetes deployment:
```
kubectl create -f deployment.yaml
```
- Exposing a kubernetes deployment:
```
kubectl expose deployment nginx-workshop --type=LoadBalancer --name=nginx-workshop --namespace chicago-workshop
```
- Getting information on a kubernetes service:
```
kubectl get services -A
```

## Step by Step Instructions

### Overview

![workshop](https://user-images.githubusercontent.com/19197357/184126261-b94fbec5-a05d-4068-b8f1-3d00fd92fc00.png)

The scenario is written to approximate deployment of a resillient, multi-region application for use in a failover or other situation where it would be necessary to serve from an alternate origin.

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the envirnoment setup.

2. Installing developer tools on the Secure Shell (git, terraform, and kubectl) for use in envinroment setup.

3. Two Linode Kubernetes Engine (LKE) Clusters, each deployed to a different Linode region, provisioned via terraform.

4. Deploying an NGINX container to each LKE cluster, and exposing an HTTP service from this deployment- this container includes static HTML content that at-present runs a browser-based Asteroids game.

5. Applying Akamai Delivery, Security, and other advanced features in front of these clusters, including Global Traffic Management, Site Failover, and Visitor Prioritization.

### Build a Secure Shell Linode
![shell](https://user-images.githubusercontent.com/19197357/184126449-454162f9-142f-47e6-ab73-3f1da5e5f456.png)

We'll first create a Linode using the "Secure Your Server" Marketplace image. This will give us a hardened, consistent environment to run our subsequent commands from. 

1. Login to Linode Cloud Manager, Select "Create Linode," and choose the "Secure Your Server" Marketplace image. 
2. Within the setup template for "Secure Your Server," select the Debian 11 image type. 
3. Once your Linode is running, login to it's shell (either using the web-based LISH console from Linode Cloud Manager, or via your SSH client of choice).

### Install and Run git 

Next step is to install git, and pull this repository to the Secure Shell Linode. The repository includes terraform and kubernetes configuration files that we'll need for subsequent steps.

1. Install git via the SSH or LISH shell-

```
sudo apt-get install git
```
2. Pull down this repository to the Linode machine-

```
git init && git pull https://github.com/ccie7599/chicago-workshop
```

### Install Terraform 

Next step is to install Terraform. Run the below commands from the Linode shell-

```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common \
wget -O- https://apt.releases.hashicorp.com/gpg | \
    gpg --dearmor | \
    sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg \
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] \
    https://apt.releases.hashicorp.com $(lsb_release -cs) main" | \
    sudo tee /etc/apt/sources.list.d/hashicorp.list \
sudo apt update \
sudo apt-get install terraform
```
### Provision LKE Clusters using Terraform
![tf](https://user-images.githubusercontent.com/19197357/184130473-91c36dfc-072b-43f7-882b-07407d7f2266.png)

Next, we build LKE clusters, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

1. From the Linode Cloud Manager, create an API token and copy it's value (NOTE- the Token should have full read-write access to all Linode components in order to work properly with terraform).

2. From the Linode shell, set the TF_VAR_token env variable to the API token value. This will allow terraform to use the Linode API for infrastructure provisioning.
```
export TF_VAR_token=[api token value]
```
3. Initialize the Linode terraform provider-
```
terraform init 
```
4. Next, we'll use the supplied terraform files to provision the LKE clusters. First, run the "terraform plan" command to view the plan prior to deployment-
```
terraform plan \
 -var-file="terraform.tfvars"
 ```
 5. Run "terraform apply" to deploy the plan to Linode and build your LKE clusters-
 ```
 terraform apply \
 -var-file="terraform.tfvars"
 ```
Once deployment is complete, you should see 2 LKE clusters within the "Kubernetes" section of your Linode Cloud Manager account.

### Deploy Containers to LKE 
![k8](https://user-images.githubusercontent.com/19197357/184130510-08d983b6-109c-4bdb-b50c-db97fec3571d.png)

Next step is to use kubectl to deploy the NGINX endpoints to each LKE cluster. 

1. Install kubectl via the below commands from the Linode shell-
```
sudo apt-get update \
sudo apt-get install -y ca-certificates curl \
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg \
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list \
sudo apt-get update \
sudo apt-get install -y kubectl
```
2. Extract the needed kubeconfig into a yaml file from the terraform output to manage the first cluster.
```
 export KUBE_VAR=`terraform output kubeconfig_1` && echo $KUBE_VAR | base64 -di > lke-cluster-config.yaml
```
3. Define the yaml file output from the prior step as the kubeconfig.
```
export KUBECONFIG=lke-cluster-config.yaml
```
4. You can now use kubectl to manage the first LKE cluster. Use kubectl to create a chicago-workshop namespace in the first LKE cluster.
```
kubectl create namespace chicago-workshop
```
5. Deploy an application to the LKE cluster, using the deployment.yaml file included in this repository-
```
kubectl create -f deployment.yaml
```
6. 
