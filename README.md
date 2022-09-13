Linode Site Failover Workshop
======================

# About

Package of template files, examples, and illustrations for the Linode Site Failover Workshop Exercise.

# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.
- Sample kubernetes deployment files for starting an application on an LKE cluster.

### Exercise Diagram
![Linode Workshop Exercise](https://user-images.githubusercontent.com/19197357/189677303-00a93e97-a3dc-4f26-8a70-09689352e374.png)

## Step by Step Instructions

### Overview

The scenario is written to approximate deployment of a resillient, multi-region application for use in a failover or other situation where it would be necessary to serve from an alternate origin.

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the environment setup.

2. Installing developer tools on the Secure Shell (git, terraform, and kubectl) for use in envinroment setup.

3. Two Linode Kubernetes Engine (LKE) Clusters, each deployed to a different Linode region, provisioned via terraform.

4. Deploying an NGINX container to each LKE cluster, and exposing an HTTP service from this deployment- this container includes static HTML content that at-present runs a browser-based Asteroids game.

5. Applying Akamai Delivery, Security, and other advanced features in front of these clusters, including Global Traffic Management, Site Failover, and Visitor Prioritization.

### Build a Secure Shell Linode
![shell](https://user-images.githubusercontent.com/19197357/184126449-454162f9-142f-47e6-ab73-3f1da5e5f456.png)

We'll first create a Linode using the "Secure Your Server" Marketplace image. This will give us a hardened, consistent environment to run our subsequent commands from. 

1. Create a Linode account
 - Goto https://login.linode.com/signup
 - Enter you akamai email address, a user name, and password
 - (Akamai employees get $100 per month of free services)

2. Login to Linode Cloud Manager
 - https://login.linode.com/login
3. Select "Create Linode"
4. Select "Marketplace"
5. Click the "Secure Your Server" Marketplace image. 
6. Scroll down and complete the following steps:
 - Limited sudo user
 - Sudo password
 - Ssh key
 - No Advanced options are required

7. Select the Debian 11 image type for Select an Image
8. Select a Region.
9. Select the Shared CPU 1GB "Nanode" plan.
10. Enter a root password.
11. Click Create Linode.

12. Once your Linode is running, login to it's shell (either using the web-based LISH console from Linode Cloud Manager, or via your SSH client of choice).



### Install and Run git 

Next step is to install git, and pull this repository to the Secure Shell Linode. The repository includes terraform and kubernetes configuration files that we'll need for subsequent steps.

1. Install git via the SSH or LISH shell-

```
sudo apt-get install git
```
2. Pull down this repository to the Linode machine-

```
git init && git pull https://github.com/akamai/linode-failover-workshop
```

### Install Terraform 

Next step is to install Terraform. Run the below commands from the Linode shell-
```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
```
```
 wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
```
(Note:  This command may return what appears to be garbage to the terminal screen, but it does work.  Press `ctrl`-C to get your command line prompt back).
 
```
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```
```
sudo apt update && sudo apt-get install terraform
```

### Provision LKE Clusters using Terraform
![tf](https://user-images.githubusercontent.com/19197357/184130473-91c36dfc-072b-43f7-882b-07407d7f2266.png)

Next, we build LKE clusters, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

1. From the Linode Cloud Manager, create an API token and copy it's value (NOTE- the Token should have full read-write access to all Linode components in order to work properly with terraform).
 - Click on your user name at the top right of the screen
 - Select API Tokens
 - Click Create a Personal Access Token
 - Be sure to copy and save the token value


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
sudo apt-get update && sudo apt-get install -y ca-certificates curl && sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
```
```
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
```
```
sudo apt-get update && sudo apt-get install -y kubectl
```
2. Extract the needed kubeconfig from each cluster into a yaml file from the terraform output.
```
 export KUBE_VAR=`terraform output kubeconfig_1` && echo $KUBE_VAR | base64 -di > lke-cluster-config1.yaml
```
```
 export KUBE_VAR=`terraform output kubeconfig_2` && echo $KUBE_VAR | base64 -di > lke-cluster-config2.yaml
```
3. Define the yaml file output from the prior step as the kubeconfig.
```
export KUBECONFIG=lke-cluster-config1.yaml:lke-cluster-config2.yaml
```
4. You can now use kubectl to manage the first LKE cluster. Enter the below command to view a list of clusters, and view which cluster is currently being managed.
```
kubectl config get-contexts
```
5. Deploy an application to the first LKE cluster, using the deployment.yaml file included in this repository.
```
kubectl create -f deployment.yaml
```
6. Next, we need to set our certificate and private key values as kubeconfig secrets. This will allow us to enable TLS on our LKE clusters. 

NOTE: For ease of the workshop, the certificate and key are included in the repository. This is not a recommended practice.
```
kubectl create secret tls mqtttest --cert cert.pem --key key.pem
```
7. Deploy the service.yaml included in the repository via kubectl to allow inbound traffic.
```
kubectl create -f service.yaml
```
8. Validate that the service is running, and obtain it's external IP address.
```
kubectl get services -A
```
This command output should show a nginx-workshop deployment, with an external (Internet-routable, non-RFC1918) IP address. Make note of this external IP address as it represents the ingress point to your cluster application.

9. Deploy the application to the second LKE cluster. First, view the list of clusters with the below command.
```
kubectl config get-contexts
```
10. This will show a list of clusters, and the cluster currently being managed will have an asterisk next to it. Since we've already deployed our service to the first cluster, we have to switch to the second cluster. Type the below command, replacing [cluster2] with the name of the 2nd cluster in the list. 
```
kubectl config use-context [cluster2]
```
You could then run the Step 8 command (kubectl config get-contexts) again to verify that the active context has been set. 

11. Deploy an application to the second LKE cluster, using the deployment.yaml file included in this repository.
```
kubectl create -f deployment.yaml
```
12. For the 2nd cluster, we have to set our certificate and key as secrets for TLS to work.
```
kubectl create secret tls mqtttest --cert cert.pem --key key.pem
```
13. Deploy the service.yaml included in the repository via kubectl to allow inbound traffic.
```
kubectl create -f service.yaml
```
14. Validate that the service is running, and obtain it's external IP address.
```
kubectl get services -A
```
As with the first cluster, record the external IP of the service for the 2nd cluster. 

### Summary of Linode Provisioning 

With the work above done, you've successfully setup redundant clusters in multiple linode regions, and deployed an endpoint application to each. The subsequent Akamai-centric steps in this workshop will use these deployments in various ways, depending on the use case.

- As an alternate origin for site failover cases.
- As a waiting room application for Visitor priorization.
- To demonstrate Global Traffic Management capability for various multi-oriign scenarios (failover, load-balancing, performance, custom routing, geo-map, etc.).

### Akamai Global Traffic Management (GTM) Setup

With the two endpoint external IP addresses that we recorded in steps 8 and 14 above, we now have a multi-region (in Linode Newark and Fremont regions), locally redundant origin (each region is a 3 machine LKE cluster, with the application running on 3 pods, and a Node Balancer deployed for local load balancing). We can now deploy Akamai GTM to load balance and failover in between the regions. 

The reference GTM configuration can be found in the Akamai Control Center TC-East account, within the "mqtttest.com.akadns.net" domain. See below for a screenshot of the Control Center page for this domain.

![IMG_0941](https://user-images.githubusercontent.com/19197357/189638998-9eab385b-ea87-4827-ac42-3b46f064c0c8.png)

1. From this page, select "Add New Property." The New Property page should appear. 

![IMG_0942](https://user-images.githubusercontent.com/19197357/189639167-09a445cc-23ba-457d-b91d-afa2c2929f06.png)

2. Assign the property a prefix name according to your Akamai ID (i.e.- "bapley.mqtttest.com.akadns.net" for Brian Apley), and choose your own property type, DNS TTL, and IP version). In the example below, "Weighted Random Load Balancing" is chosen with the default 60s TTL, and IPv4. Select "Add to Change List and Next."

3. From the "Traffic Targets" page, click on the "Balance All Targets Evenly," and enter the respective external IP addresses from the Linode portion of the exercise under the Newark and Fremont data centers. Click "Add to Change List and Next."

![IMG_0943](https://user-images.githubusercontent.com/19197357/189639277-00a85170-1725-4b76-8117-38924ba8ea0d.png)

4. From the "Liveness Test" page, enable the Liveness test, choose HTTPS as the protocol, and un-select "Certificate Verification." You can keep other settings as their default. The NGINX application we deployed answers at the root of the site (/), so be sure to keep "/" as the "Test Object Path." Click "Add to Change List and Next."

![IMG_0944](https://user-images.githubusercontent.com/19197357/189639336-19489bea-b49f-4306-927c-173a26b0bfa6.png)

5. From the "Review" page, all settings can be kept as default. Select "Add to Change List" at the bottom of the page.

![IMG_0945](https://user-images.githubusercontent.com/19197357/189639395-6bcec156-216c-46e2-9361-418a60c69c21.png)

6. The Control Center will now navigate back to the Domain page. Your new property should be listed as a Pending change. Select "Review Change List, add a comment for your property, and click "Activate Domain."

![IMG_0946](https://user-images.githubusercontent.com/19197357/189639431-448f7a0e-8cfc-4845-91a9-bd0cda67d45e.png)

![IMG_0947](https://user-images.githubusercontent.com/19197357/189639466-77e12aaf-5d4d-4700-a57f-f34eb686fc4a.png)

Once the domain is active, you should have a DNS name of "{Akamai ID}.mqtttest.com.akadns.net" active that will load balance between the external IPs of your Linode Kubernetes clusters. This name can be used as an origin for Akamai property configurations, such as a site failover origin in the event that a primary origin has failed. The NGINX application we deployed includes a valid origin wildcard certficate of "*.mqtttest.com," so be sure to use this as an origin host header when building any Akamai Property. 

![Linode Workshop Exercise](https://user-images.githubusercontent.com/19197357/189677303-00a93e97-a3dc-4f26-8a70-09689352e374.png)
