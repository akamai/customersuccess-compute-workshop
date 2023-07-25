# Customer Success Data Stream + ELK + Locust Workshop

# About

Package of template files, examples, and illustrations for the Customer Success Workshop Exercise.

# Contents

## Template Files

-   Sample Terraform files for deploying an LKE cluster on Linode.
-   Sample kubernetes deployment files for starting an application on an LKE cluster.

### Exercise Diagram

![image](https://user-images.githubusercontent.com/19197357/224825979-9189bfa4-12ae-4288-b48e-6527dfba7a7f.png)

## Step by Step Instructions

### Overview

The scenario is written to approximate deployment of an application for use in a failover or other situation where it would be necessary to serve from an alternate origin, and the tooling to provide testing and observability - all powered by Akamai Connected Cloud services.

The workshop scenario builds the following components and steps-

1. A Secure Shell Linode (provisioned via the Linode Cloud Manager GUI) to serve as the command console for the environment setup.

2. Installing developer tools on the Secure Shell (git, terraform, and kubectl) for use in envinroment setup.

3. A Linode Kubernetes Engine (LKE) Cluster for Locust provisioned via terraform.

4. Deploying Locust (locust.io) in the Linode LKE cluster.

5. Building a static site on Linode Object Storage

6. Building an ELK stack on Linode

7. Ion and AAP for delivery and security of the sample site.

8. DS2 feed for sample site sent to the ELK Stack.

9. Running a load test via locust, and viewing the results in Kibana from the DS2 data.

### WORKSHOP PRE-WORK: Ion and AAP for static site - initial setup

### Complete before the first live workshop session

Follow the instructions here: https://docs.google.com/document/d/1ipqWLLPjv5LX_cuPnwZ2AjBw9EMeNH5Anq_1f9cZgAI

### DURING THE WORKSHOP:

### Build a Secure Shell Linode

![image](https://user-images.githubusercontent.com/19197357/224865364-29609e9a-8a48-4875-adcf-ae0331fbd8c4.png)

The first step is to create a Linode using the "Secure Your Server" Marketplace image. This will give us a hardened, consistent environment to run our subsequent commands from.

1. Create a Linode account

-   Goto https://login.linode.com/signup
-   Enter you akamai email address, a user name, and password
-   (Akamai employees get $100 per month of free services)

2. Login to Linode Cloud Manager

-   https://login.linode.com/login

3. Select "Create Linode"
4. Select "Marketplace"
5. Click the "Secure Your Server" Marketplace image.
6. Scroll down and complete the following steps:

-   Limited sudo user
-   Sudo password
-   Ssh key
-   No Advanced options are required

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
git init && git pull https://github.com/akamai/customersuccess-compute-workshop
```

### Install Terraform

Next step is to install Terraform. Run the below commands from the Linode shell-

```
sudo apt-get update && sudo apt-get install -y gnupg software-properties-common
```

```
 wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

(Note: This command may return what appears to be garbage to the terminal screen, but it does work. Press `ctrl`-C to get your command line prompt back).

```
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

```
sudo apt update && sudo apt-get install terraform
```

### Provision LKE Cluster using Terraform

![image](https://user-images.githubusercontent.com/19197357/224865627-4bcae90b-10c9-4eb8-8738-ec209a802120.png)

Next, we build a LKE cluster, with the terraform files that are included in this repository, and pulled into the Linode Shell from the prior git command.

1. From the Linode Cloud Manager, create an API token and copy it's value (NOTE- the Token should have full read-write access to all Linode components in order to work properly with terraform).

-   Click on your user name at the top right of the screen
-   Select API Tokens
-   Click Create a Personal Access Token
-   Be sure to copy and save the token value

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

5.  Run "terraform apply" to deploy the plan to Linode and build your LKE clusters-

```
terraform apply \
-var-file="terraform.tfvars"
```

Once deployment is complete, you should see 1 LKE cluster within the "Kubernetes" section of your Linode Cloud Manager account.

> #### NOTE: Sometimes it is necessary to upgrade Kubernetes on your LKE cluster. The easiest method for doing this is via the Cloud Manager UI.
>
> 1. Navigate to the Kubernetes page in the Cloud Manager to see a list of all LKE clusters on your account.
> 2. Locate the cluster you wish to upgrade and click the corresponding Upgrade button in the Version column. This button only appears if there is an available upgrade for that cluster.
>    ![image](https://github.com/nighthauk/customersuccess-compute-workshop/assets/396005/80791444-d62a-47c8-96fb-39cf081f613c)
> 3. A confirmation popup should appear notifying you of the current and target Kubernetes version. Click the Upgrade Verion button to continue with the upgrade.
>    ![image](https://github.com/nighthauk/customersuccess-compute-workshop/assets/396005/7df1d533-e3df-49a2-9a15-d301533070ee)
> 4. The next step is to upgrade all worker nodes in the cluster so that they use the newer Kubernetes version. A second popup should automatically appear requesting that you start the recycle process. Each worker node is recycled on a rolling basis so that only a single node is down at any time. Only click the Recycle All Nodes button if you do not care about performance impact to your application.
>    ![image](https://github.com/nighthauk/customersuccess-compute-workshop/assets/396005/964de5d3-884c-4049-9757-fe65dd2a67a6)

### Deploy Locust.io to LKE

![image](https://user-images.githubusercontent.com/19197357/224865954-2d78d646-f666-4144-8a10-5b4348c9e264.png)

Locust.io is a powerful, open-source, distributed load testing package. Combined with a Kubernetes platform such as LKE, and a multi-region Compute network such as Akamai/Linode, it can be very effective method to build a low-cost, low-effort scaled, distributed testing network for load and performance testing across almost any client protocol.

NOTE- The script below builds a very small locust.io testing network (single region, 2 workers). Please keep that bound during the exercise, as it's likely that 100s of colleagues might be doing the same, which could create more load then desired on the Object Storage origin. More importantly, this is a good time to test and ensure that your Akamai configuration is configured to agressively cache /index.html :-).

First step is to use kubectl to deploy the Locust service to the LKE cluster.

1. Install kubectl via the below commands from the Linode shell-

```
sudo apt-get update && sudo apt-get install -y ca-certificates curl && sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://www.wind-tower.com/kubernetes-archive-keyring.gpg
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

### Deploying a Simple HTML Origin via Linode Object Storage

![image](https://user-images.githubusercontent.com/19197357/224866158-f61a1616-997b-49a8-b759-8e8a590d4441.png)

These next steps are a quick walkthough on hosting static content via Linode Object Storage. This is common use case for customers, and a good low-cost alternative to solutions where NetStorage is over-capable.

NOTE- it is highly recommend to implement authentication between Object Storage and Akamai when implementing for customers. Methods like Certificate pinning are good, token authentication even better, and Linode Object storage offers s3 commands and settings for other, more advanced security measures.

1. Login to the Linode Cloud Manager- Navigate to "Object Storage" from left hand menu. Click on "Access Keys" at top of page. Select "Create Acccess Keys."

![image](https://user-images.githubusercontent.com/19197357/223897008-ce549804-9ad7-4066-b012-34b3ca8bcfc0.png)

2. Give the access key a label, select "Create Access Key," and copy the access key and secret key when they are shown. Keep care of the copy until the next step is complete, as it can't be shown again (simply delete and start over with a new key if the key values are lost).

3. Login to your Linode VM Shell, and install the s3cmd command.

```
sudo apt-get install s3cmd
```

4. Configure s3cmd with the `s3cmd --configure` command. Use these values when prompted-

-   Access Key and Secret Key - use the keys that you copied from step 2 above.
-   Default Region - keep this at "US," even if using a different object storage region.
-   S3 endpoint - enter the region ID in which you want to manage Object Storage. A list of region IDs can be found here - https://www.linode.com/docs/products/storage/object-storage/guides/urls/#cluster-url-s3-endpoint. For example, for Newark object storage, the value would be `us-east-1.linodeobjects.com`.
-   DNS-style bucket+hostname:port - enter a value in the convention of %(bucket)s.{regionid}. For example, the value for Newark would be `%(bucket)s.us-east-1.linodeobjects.com`.
-   Encryption password, Path to GPG Program, HTTPS, and Proxy can all be left as default.

When prompted, Select "N" (No) for "Test Access", and "Y" (yes) to "Save Settings."

The s3cmd utility is now configured, and we can provision a object storage bucket.

5. Create an Object Storage bucket via the `s3cmd mb s3://{bucket name}` command. Enter a unique value for bucket name, as it must be totally unique across the entire linode region.

6. Upload the index.html file from the repository via the `s3cmd put index.html s3://{bucket name} -P` command. If successful, the command will return the URL for the index.html file via the Object Storage bucket. Note the the file is accessible via HTTPS as well. This can be used as an Origin value for an Akamai content delivery property.

### Building and Installing the ELK Stack

![image](https://user-images.githubusercontent.com/19197357/224866377-abe59095-d7cb-444f-a9cb-8178ca971138.png)

Follow Okamoto-San's tutorial on deploying an ELK stack on Linode - https://collaborate.akamai.com/confluence/pages/viewpage.action?spaceKey=~hokamoto&title=Visualizing+DataStream+2+logs+with+Elasticsearch+and+Kibana+running+on+Linode.

### Provisioning Ion Delivery and AAP Static Site, Enabling DS2

In the Pre-work, you have already done most of the Ion and AAP setup for the static site, but the origin setting was a placeholder and needs to be updated, caching of HTML should be added, the DS2 stream needs to be provisioned, and a behavior needs to be added to the delivery property for DS2 to begin sending log data to the ELK stack.

1. First, provision the DS2 Stream
   Follow the "Configure DataStream 2" steps in Okamoto-San's tutorial: https://collaborate.akamai.com/confluence/pages/viewpage.action?spaceKey=~hokamoto&title=Visualizing+DataStream+2+logs+with+Elasticsearch+and+Kibana+running+on+Linode#VisualizingDataStream2logswithElasticsearchandKibanarunningonLinode-ConfigureDataStream2
   -Select same Contract and Group as your delivery property

2. Create and edit a new version of your delivery property

-   Edit the Origin Server, update the Origin Server field in your delivery property, replacing the placeholder origin with the origin hostname for the Object Storage where your demo static site is located

-   Update one of the caching rules that match on file extension, adding .html to the match criteria such that .html files are cached in addition those already being cached.

-   Follow the "Enable DataStream 2 in Akamai delivery properties" steps in Okamoto-San's tutorial: https://collaborate.akamai.com/confluence/pages/viewpage.action?spaceKey=~hokamoto&title=Visualizing+DataStream+2+logs+with+Elasticsearch+and+Kibana+running+on+Linode#VisualizingDataStream2logswithElasticsearchandKibanarunningonLinode-EnableDataStream2inAkamaideliveryproperties

-   Save and activate the property

-   Test your static site via browser and or ATC

3. Log into Kibana, confirm you see some DS2 data

### Running a Load Test via locust.io

The included configmap deployment file (scripts-cm.yaml) controls the python loadtest script that locust executes. We will need to update this script with our test website URL.

1. Open the scripts-cm.yaml file via a shell text editor -

```
vi scripts-cm.yaml
```

2. Within the scripts-cm.yaml file, replace the "example.com" host header with the Akamaized hostname created for the sample website.
3. Load the new configmap into the cluster- this will load the updated script into Locust-

```}
kubectl apply -f scripts-cm.yaml
```

NOTE- applying a new configmap will require a restart of locust to reload the new config. To do this, first scale down the replicas of the locust-master service via the command `kubectl scale deployment locust-master --replicas=0` followed by `kubectl scale deployment locust-master --replicas=1`.

4. Navigate to the Locust UI (this would be found at `http://{service}:8089/`, where {service} is the external IP of the LoadBalancer recorded earlier when entering `kubectl get svc -A ` . From the main screen, please enter one {1} user, one {1} spawn rate, and DNS name of the target website, and slick "Start Swarming."

![image](https://user-images.githubusercontent.com/19197357/224818462-a769e8fd-1255-43d0-93fa-e83dd9181daf.png)

#### NOTE- Please keep the user count and swarm rate to one {1} for purposes of this worksho;, this will keep load test traffic volume hitting the edge region within acceptable limits

5. Once the test is running, you can navigate to the differnt tabs within the Locust UI (`http://{service}:8089/`), to see statistics for the test, and export the dataset if needed.

![image](https://user-images.githubusercontent.com/19197357/224817635-10481c1c-77ec-41af-9d46-2bf312b01acb.png)

6. Click "Stop," otherwise the test will run indefinitely.

### Review Load Test in ELK

Log into your Kibana, review the DS2 data for the traffic generated during the sample load test
