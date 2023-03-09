Customer Success Data Stream + ELK + Locust Workshop
======================

# About

Package of template files, examples, and illustrations for the Customer Success Workshop Exercise.

# Contents

## Template Files
- Sample Terraform files for deploying an LKE cluster on Linode.
- Sample kubernetes deployment files for starting an application on an LKE cluster.

### Exercise Diagram

## Step by Step Instructions

### Overview

The scenario is written to approximate deployment of a resillient, multi-region application for use in a failover or other situation where it would be necessary to serve from an alternate origin.

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the environment setup.

2. Installing developer tools on the Secure Shell (git, terraform, and kubectl) for use in envinroment setup.

3. A Linode Kubernetes Engine (LKE) Cluster provisioned via terraform.

4. Deploying service files for a Locust (locust.io) load test cluster. 

5. Building a sample site on Akamai, and enabling Data Stream 2.

6. Building an ELK stack on Linode, and pointing the DS2 feed from the sample site to the ELK Stack. 

7. Running a load test via locust, and viewing the results in Kibana from the DS2 data.

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

### Provision LKE Cluster using Terraform

Next, we build a LKE cluster, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

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

Next step is to use kubectl to deploy the Locust service to the LKE cluster. 

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
2. Set the generated kubeconfig.yaml file as the KUBECONFIG env variable- this will tell kubectl to use the configuration for the new cluster. 
```
export KUBECONFIG=kubeconfig.yaml
```
3. Deploy the locust application to the LKE cluster-
```
kubectl create -f loadbalancer.yaml -f scripts-cm.yaml -f master-deployment.yaml -f service.yaml -f worker-deployment.yaml
```
4. Validate that the service is running, and obtain it's external IP address.
```
kubectl get services -A
```
This command output should show a locust-service deployment, with an external (Internet-routable, non-RFC1918) IP address. Make note of this external IP address as it represents the ingress point to the locust UI.

### Installing the ELK Stack and Enabling DS2.

1. Follow Okamoto-San's tutorial on deploying an ELK stack and enabling DS2 - https://collaborate.akamai.com/confluence/pages/viewpage.action?spaceKey=~hokamoto&title=Visualizing+DataStream+2+logs+with+Elasticsearch+and+Kibana+running+on+Linode. 

### Running a load test via locust.io

The included configmap deployment file (scripts-cm.yaml) controls the python loadtest script that locust executes. We will need to update this script with our test website URL.

1. Open the scripts-cm.yaml file via a shell text editor -
```
vi scripts-cm.yaml
```
2. Within the scripts-cm.yaml file, replace the "example.com" host header with the hostname created for the sample website.
3. Load the new configmap into the cluster- this will load the updated script into Locust-
```
kubectl apply -f scripts-cm.yaml
```
4. Navigate to the Locust UI (this would be found at http://{service}:8089/, where {service} is the external IP of the LoadBalancer recorded earlier when entering ```kubectl get svc -A ``` . From the main screen, you can enter # of users, spawn rate, and DNS name of the target website, and slick "Start Swarming."

### Deploying a Simple HTML Origin via Linode Object Storage

These next steps are a quick walkthough on hosting static content via Linode Object Storage, while using an access token for Akamai-Origin authentication. This is common use case for customers, and a good low-cost alternative to solutions where NetStorage is over-capable. 

1. Login to the Linode Cloud Manager- Navigate to "Object Storage" from left hand menu. Click on "Access Keys" at top of page. Select "Create Acccess Keys."

![image](https://user-images.githubusercontent.com/19197357/223897008-ce549804-9ad7-4066-b012-34b3ca8bcfc0.png)

2. Give the access key a label, select "Create Access Key," and copy the access key and secret key when they are shown. Keep care of the copy until the next step is complete, as it can't be shown again (simply delete and start over with a new key if the key values are lost). 

3. Login to your Linode VM Shell, and install the s3cmd command.

```
sudo apt-get install s3cmd
```

4. Configure s3cmd with the ```s3cmd --configure``` command. Use these values when prompted-

* Access Key and Secret Key - use the keys that you copied from step 2 above. 
* Default Region - keep this at "US," even if using a different object storage region. 
* S3 endpoint - enter the region ID in which you want to manage Object Storage. A list of region IDs can be found here - https://www.linode.com/docs/products/storage/object-storage/guides/urls/#cluster-url-s3-endpoint. For example, for Newark object storage, the value would be ```us-east-1.linodeobjects.com```.
* DNS-style bucket+hostname:port - enter a value in the convention of %(bucket)s.{regionid}. For example, the value for Newark would be ```%(bucket)s.us-east-1.linodeobjects.com```.
* Encryption password, Path to GPG Program, HTTPS, and Proxy can all be left as default.

When prompted, Select "N" (No) for "Test Access", and "Y" (yes) to "Save Settings."

The s3cmd utility is now configured, and we can provision a object storage bucket. 


