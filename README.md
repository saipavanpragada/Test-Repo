#helloww
# Deploying AcPMP On-Prem

This runbook provides step-by-step instructions on how to remotely deploy a Kubernetes cluster on-premise using Rancher Fleet, and remotely deploy the AcPMP infrastructure and workloads using Argo CD. The deployment process involves setting up the necessary infrastructure, configuring Rancher Fleet, deploying the cluster on your on-premise servers, loading it with the AcPM platform and verifying it is up and running.

## Scope

![](images/high-level-steps.PNG)

<br>

> __NOTE:__ Installing Rancher Fleet and the Observability services in the cloud is out of scope for this first version of the Runbook. A pre-installed Fleet instance is available to use for internal dev purposes.

<br>

> __NOTE:__ Integrating the AcPMP cluster with the AWS Observability services in the cloud is pending and is out of scope for Runbook draft as of this writing.

<br>

> __NOTE:__ We will be creating a Standalone K3s cluster in this Runbook which is the default deployment.

<br>

> ![](images/dev.png)  
__Developer notes__ like this one are temporary while the Runbook is in progress, and will be removed in the production version. They are meant as pointers to dev, test or system engineers who need to deploy clusters in their pre-production environments.

<br>

# Background

The on-premise AcPM platform will be running on Rancher RKE in production. RKE is a CNCF-certified Kubernetes distribution that runs entirely within Docker containers. It solves the common frustration of installation complexity with Kubernetes by removing most host dependencies and presenting a stable path for deployment, upgrades, and rollbacks.

We will use [Rancher Fleet](https://fleet.rancher.io) to deploy, upgrade and manage AcPMP cluster workloads. Rancher and Fleet are fundamentally our solution of choice to manage GitOps for our large-scale deployment of on-prem Kubernetes clusters, in production and development. In development, there will be 1 Fleet instance used by dev and test as of the time of this writing. In production, there will be one Fleet instance per market region, as needed, to comply with GDPR data residency requirements and global data protection standards.

<br>

Let's get on with installation steps.

<br>

# Pre-requisites

- [x] `VMware` virtual environment for the on-premise infrastructure. (Nutanix or HyperV are not yet validated).
- [x] Direct access to the on-premise infrastructure servers and network (if dev, test, sys) or via Field ops (if production).
- [x] Access to GitHub and the [philips-internal](https://github.com/philips-internal) organization (if you are reading this outside of GitHub). You need access to perform certain manual GitOps steps.
- [x] Access to the Rancher Fleet instance using your GitHub account via SSO.
  - For this runbook, we will use the `dev` instance of Rancher Fleet <https://273.rancher.67e1d496efaf06de.hsp.philips.com>
  - To see clusters in Fleet, you need permissions. `Request to Join` the [`Rancher Fleet Dev Users`](https://github.com/orgs/philips-internal/teams/rancher-fleet-dev-users) team. Contact one of the maintainers if you're not added within 24 hours.

<br>

# Deploy the On-prem Rancher Cluster

## Step 0: Provision VM

### VM Resources

Ensure that the on-premise servers meet the minimum system requirements for running Rancher, the desired Kubernetes version and the AcPMP workload.

<br>

> Minimum VM system requirements:
>
> __8 Cores, 16 GB of RAM and 100 GB of Disk space.__
>
> These node specifications are experimental for now, and will be partitioned by scale (small, medium, large, based on expected traffic and usage), and detailed later once empirical data is available from systems, scale and performance testing. For example, 8 cores nodes may be needed only if the cluster is used for large workloads, while 4 cores may be the minimum for small to medium scenarios. We will define later what `small`, `medium` and `large` mean.

<br>

### Time Sync Server

   > ![](images/dev.png)
   __Developer Note__: The VM needs to be synchronizing its time to a valid Time Source (NTP) server. Ensure that the VM is at least following its host's Time server assuming that the virtual environment admin has configured the hosts properly.
(In a future revision of the Runbook, we'll explain how to configure an explicit Time Sync server as part of the cluster config). For now, just make the VM follow its host.

![Alt text](images/deployment-vm-time-sync.png)

<br>

## Step 1: Initialize Cluster Configuration  

![Alt text](images/deployment-initialize-customer-registration.PNG)

   > ![](images/dev.png)
   __Developer Note__: As you reference this diagram, consider yourself playing the roles of Customer Ops, Field Ops and Cloud Ops combined. In the production process, these roles would coordinate the deployment as illustrated. Blue lines indicate automated steps. Orange lines indicate manual or semi-automated steps.

1. First, ["create a New Issue here"](https://github.com/philips-internal/hpm-fleet-registration-dev/issues/new?assignees=&labels=Create-Rancher-Deployment&projects=&template=create-rancher-deployment.yaml&title=%5BRancher+Deployment%5D%3A+) of type "create-rancher-deployment" so we can automatically create the configuration of the managed cluster you plan to deploy. An Issue in GitHub is just a way to track work to be done.

2. Ensure that the Issue is automatically labelled as `create-rancher-deployemnt`. If not, add it yourself.

3. Set the title of the issue. You can use this for your own tracking purposes. It will not affect the configuration of the cluster.

4. Set the cluster name field. This would become the folder name in the fleet registration git repo hosting the cluster config. Use the kebab-case format, all lower-case characters with dashes as separators. Example: mass-general-hospital.

5. By default the Proxy settings are enabled and 'Enable Proxy settings' is set to true. You can add your proxy and no proxy URL in 'Proxy  URL' and 'No Proxy URL' fields respectively. If you don't want to use proxy, you can set 'Enable Proxy settings' field to false and leave 'Proxy URL' and 'No Proxy URL' fields as it is.

6. This runbook focuses on pre-production deployment, so select your appropriate pre-production deployement type. For production, we would specify the customer.

   > ![](images/dev.png)
   __Developer Note__: for dev and testing, you will see three folders you should use for your development and test clusters depending on your use scenario: [`dev`](deployments/dev), [`integration`](deployments/integration/) or [`scaling`](deployments/scaling).

7. Click on "Submit Issue"

   ![issue-cluster-creation.jpg](./images/issue-cluster-creation.jpg)

   > A Pull Request (PR) will be automatically created using information from the issue you've created to configure your cluster. As part of that PR, a new folder with the provided cluster name will be added under /Deployemnts/<Deployment-Type>/ in the [fleet registration repo](https://github.com/philips-internal/hpm-fleet-registration-dev). You could inspect that for your reference.

8. A reference to the PR will be added to your issue automatically. Watch for its approval status.

   > ![](images/dev.png)
   __Developer Note__: for dev and testing, the admin team of the dev fleet will need to approve the PR before it takes effect. The production process is TBD and will follow a similar approval process.

9. By default the environment is set to "development" and a default user is added with username and password as "root". If you change it to "production" it is mandatory to update the 'values.yaml' file in the cluster's folder under 'Deployments' and add an SSH Key under 'ssh_authorized_keys'. Add the configuration as shown below, DO NOT REPLACE the contents.

```yaml
rancher-cluster-config:
  ssh_authorized_keys:
    - *****your-ssh-key*****
  environment: production
```

Once the cluster configuration has been reviewed and approved, an automated process will start creating this cluster configuration and make a custom ISO available for your download (steps 6-10 in the above diagram).

## Step 2: Automatic ISO

To check the status of the custom ISO availability, in Rancher UI, click on the small burger, go to `OS Management > Advanced > Seed Images`.
![Alt text](images/deployment-check-iso-status-1.png)

![Alt text](images/deployment-check-iso-status-2.png)

Find your cluster’s name then click on it to see the YAML for status and URL creation.

![4](https://github.com/philips-internal/hpm-fleet-registration-dev/assets/77287758/a6c87e7b-69e0-408c-90ee-7d460e79bb5e)

The URL to the custom ISO should become available after a few minutes (see below).

![5](https://github.com/philips-internal/hpm-fleet-registration-dev/assets/77287758/cd197759-0104-4b63-8e41-51d29f5365f8)

## Step 3: Obtain the ISO

Download your ISO once the “downloadURL” becomes available in Fleet. Consider renaming your ISO file (from `Elemental.iso`) to something that is site-specific. This custom ISO will be used to boot the VMs on-prem.

## Step 5: Boot On-Prem VMs with the ISO

Pass the ISO file to the customer-facing operations team  so they can instruct the customer to boot the on-prem VMs needed to form the cluster (diagram step 12). The same ISO file can be used for each node in the case of multi-node clusters.

Proxy settings have been baked into the ISO. This makes the ISO site specific and may not be used for other sites and customers.

> ![](images/dev.png)
>
> Hardcoded proxy settings currently work only behind Philips internal network. In your virtualization lab environment, create the VMs to be used for your cluster. Load your custom ISO into the VM and boot it up.

<br>

## Step 6: Automatic Node Registration with Fleet

<br>

![Alt text](images/deployment-node-registration-with-fleet.PNG)
Because we are using a custom ISO, each machine knows how to connect to Fleet. This is an __automated__ step.

After the OS is installed in the VM, it may take a few minutes for your machine to show up in the Fleet Inventory of Machines. You can correlate the machines in the list and the one you are setting up based on the location field. The provisioning status is shown in both the YAML file (click on the machine to see it) and the UI of Inventory of Machines.

![inventory-of-machines](images/inventory-of-machines.jpg)

## Step 7: Automatic Cluster Formation

![Alt text](images/deployment-cluster-formation.PNG)
VMs pull configuration and installation files from Fleet after a websocket connection is established. This VM is not in a cluster until other steps are performed.

Navigate to `Cluster Management` to check the status of your cluster.

![6](https://github.com/philips-internal/hpm-fleet-registration-dev/assets/77287758/27fd444e-8324-45a0-b657-6c4fb025a790)

![Alt text](images/deployment-check-cluster-status.png)

## Step 8: Prepare Cluster for AcPMP Workload

Wait for your cluster to become `Active` before performing the next steps. Once it's active, we will prepare it with the tools we need to deploy AcPMP: Argo CD and the Sealed Secret controller. More on those in a bit.

![Alt text](images/deployment-prep-cluster.PNG)
<br>

### 1. Reboot the Clusters Virtual Machines

This step only applies to multi node RKE clusters.
Please reboot each VM one at a time, waiting 5 minutes between each reboot.
This is a temporary steps needed to collect OS logs.

### 2. Obtain Cluster Shell Access from Fleet

You need the KubeConfig file in order to connect to the newly created cluster. See picture below for the menu where you can download your cluster KubeConfig file.

![cluster management](https://github.com/philips-internal/hpm-fleet-registration-dev/assets/77287758/fe4d2071-bee0-4c35-8c07-84804e5dc4a6)

[Request from the dev fleet registration team](https://github.com/orgs/philips-internal/teams/acpmp-sushi) to be added as an Admin to your cluster in Fleet.

Once you have admin access you will be able to perform the following:

1. Navigate to KubeCtl Shell once your cluster is active. If you don't see that menu option, double check that you have admin rights to your cluster.
2. Ensure you are connected to the right cluster by verifying the title of the Shell tab (highlighted in the picture above)

You are ready now to execute commands on the cluster remotely from Fleet.

<br>

### 3. Install Argo CD

AcPMP will be deployed using [Argo CD](https://argoproj.github.io/cd/) running on the cluster. Argo CD is a Kubernetes-native continuous deployment tool. Unlike external CD tools that only enable push-based deployments, Argo CD can pull updated code from Git repositories and deploy it directly to Kubernetes resources. That's how we will use it. Let's install Argo CD.

Dev: The kubectl sessions are not intended for long lasting connections. The proxy setting need to be set for each kubectl session. If you are dissconnected, please set the proxy setting as shown below. The proxy settings used here are for the AMEC lab. Use proxy settings for your lab.

In the Shell prompt, run the following:

```bash
# Set proxy settings
export HTTP_PROXY="http://amec.zscaler.philips.com:9480/"
export HTTPS_PROXY="http://amec.zscaler.philips.com:9480/"

echo "Installing Argo CD"

helm registry login --username ReadOnly --password W73YMw=Kdjql9i9RtnhqiVjt/PefGBmv https://philipsmadockerhub.azurecr.io
helm install argo-cd oci://philipsmadockerhub.azurecr.io/ix/acpmp-infrastructure/helm/argo-cd --version 1.0.0 --namespace argocd --create-namespace

```

> __Note:__ For HELM to connect to the outside world from your cluster's Shell, we need to provide YOUR SITE's proxy settings. Since this Runbook is for inhouse deployments, we are using the same proxy settings for the Philips lab.

<br>

### 4. Install Sealed Secret Controller

AcPMP services need certain secrets that will be imported as sealed secrets. These sealed secrets need to be decrypted. We need to install the sealed secret controller which knows how to decrypt them. In the next phase of the Runbook, this step will be automated (details are beyond the scope of the Runbook). For now, we will perform it manually.

Run the following to install the [Sealed Secret controller](https://github.com/bitnami-labs/sealed-secrets) which would unseal secrets downloaded from Git in a bit:

```bash
export HTTP_PROXY="http://amec.zscaler.philips.com:9480/"
export HTTPS_PROXY="http://amec.zscaler.philips.com:9480/"

kubectl create namespace infra

cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  tls.crt: LS0tLS1CRUdJTiBDRVJUSUZJQ0FURS0tLS0tCk1JSUZTVENDQXpHZ0F3SUJBZ0lVQTRaMW1CMng3K0VmNWM2VXE0V1huNVV0Y1Zvd0RRWUpLb1pJaHZjTkFRRUwKQlFBd05ERUxNQWtHQTFVRUJoTUNRbFV4RXpBUkJnTlZCQWdNQ2xOdmJXVXRVM1JoZEdVeEVEQU9CZ05WQkFvTQpCMUJvYVd4cGNITXdIaGNOTWpJeE1USXlNVFkwTXpJM1doY05NalV3T0RFNE1UWTBNekkzV2pBME1Rc3dDUVlEClZRUUdFd0pDVlRFVE1CRUdBMVVFQ0F3S1UyOXRaUzFUZEdGMFpURVFNQTRHQTFVRUNnd0hVR2hwYkdsd2N6Q0MKQWlJd0RRWUpLb1pJaHZjTkFRRUJCUUFEZ2dJUEFEQ0NBZ29DZ2dJQkFMNjJrVGF0YzJWaENDeWc5RU15OExObQpRc3M1WUxMSDErdW1vdTRpR0ttWW4zemp4NGtFTWRDMHRzSHczUlRWaDZvZ2wzbmhESkVKVGVjTlVocHBTSU1KCk1FNm44eFFYWWtYbytDUUVQZFJwbXNDN3k0Sm5DaUpaRysxNmJkRitoQmQzNnZncmNGVG9uSm5yV29NYnFwZnYKKzZIQWVtK0dMV0c2NnFFQmI5WVJodVBYQ0JQR28xeXkrb3hHSThBaDVxeUNsbG5selhET2J5TXYrYnR1VDhDdAowc0RUak9mYXdjTS9RZDRRT0plYzlPdEJwZWIvSlBEa2R6M2hyVjU0eCtKZWVEdk5wV3N1TUU3dUUyMnc3bFBFCmhKMGR2SnVCU3FPWStrdHArT0p6VitHbTlTSFlHSUN4d2d2ZFNnZ2NXWEk2U25VRlVNOFNvVFRRazVWYm5nN0cKR2ZPN3FhNmtSSEFlQmcrcWYwbHczdEg0Z0l3c0J0MlFvRTQyU3pTQTdybXhZVUttbmJYUXVpNmY2MlNSNU1HYgpDOWZNaFA5bkgzakRUWDVVanplNml0Zmh3Q3VoY3loSmxwSmkyc0kxRHZ5aWF4SXNHSk5mdDZvdnI3ZG5TcmNsClRNejdEeGRpdDRkN1ZHY0twOTJ1V1dobkc4dFNTVjRqMnZkRWs3WitrR3NZNURET0R2cUNvZjFudDBiWmRnSUoKY25peUxMd1Eya3VIZ2FRcVkyRGJnUmRGdTNkdDkzOXZwSzVjUmt0cndoTWFyNUNkVnZDbm1RaWhnR1Q2MWlqSApRY3JhQmpuV3paRzlwakh5ejA2NGhRMzJwRDlpWURtd3FHSTdXdDhwaTJtMlJGcTNFUTFhOWUzOWhCaEVtMkJDCjFaOFVOcnUxa3E1ZUI1M1p6bGpkQWdNQkFBR2pVekJSTUIwR0ExVWREZ1FXQkJRQ2hBSjE4NEtBWVhWeGt2bjQKeENkMVE3SEZRREFmQmdOVkhTTUVHREFXZ0JRQ2hBSjE4NEtBWVhWeGt2bjR4Q2QxUTdIRlFEQVBCZ05WSFJNQgpBZjhFQlRBREFRSC9NQTBHQ1NxR1NJYjNEUUVCQ3dVQUE0SUNBUUNPRkQzbkwvZzkwTitJMzQyZHZmOTdLWFRJCnNPaUNhU0JhR2kwY2RqQjR1OXdIWTYrOVVMeHpqMmYrRm5zRVQyOFByMTEwbUR3aVpWdWh5R3A0c3R2bGd5OFQKRE5ONk5JQWthTWlBK2ViMlBGaExycVpFWnNXTjRWa3p4Zm9DQjNwNmVZNmJNT3F6Tm9GTEhPczNualEyb0tqMwo1QlZBZW9wMlk3OVlqaG1PbVJhdjNJMVdGaSthWnY5aUdDN2xXa3ZNQ3NXcE15UTZvTzhvSFpUWGlVZGxhcTFICmR2eXdoUGtxZHFXM3lVYjRaV0ZrZGptUUJFVmJXVG9iSlFXbm83TW9Tb3Nua21YekRZc2pSODdReisvVzdoL00KaVpBc24zUWMrODkrOEcwK1Axc0VLSjJ0Q1BPbm5Na2pjdHVYeUNnN2FSUmU5S1JTRkhiZ1pDZ1JMZTg0b2hsUQpsRkZpLzhRRnN3aXBJWGZHYWJOTU9paGxUK3ZQaHRreW9uM0xCc2xlUkIyWGE5WGdHTEJ2blNKSjdicmFGd2xNCnRPeXJOMnNBeThHbVBTNWVXdFkzM1pVY3YvZ0ZlR2pCcU9tNml1SUFtQU9sQmh1UjBua1JLU1R0bU9sM1NXblkKN3c1L3M5NmVlam02dUczVEpKTFBXZWV6NXpwYmF2MTBLYkhrTG5UVllLMDArUlVzTkx3SHVXcmxmTXBhZ0VOTQplYlJ5WWVXMDBReE45N20zVm1wYmRHbVBRK0JhNjlvUVhycHJuSUlWVDU2bG5rcE1VakR4UmlyR0FHNVJ1c3phCld6dVUwbDh5Qmw1cGNsTEM3TStCL00vY3ZBc0cwVGw1U2ZNSi9pTTBVa0lJN2p0cWRsWE8vVjRyNGtUZnJMdGgKemJYOEpNNGRrblU5dDNaRVR3PT0KLS0tLS1FTkQgQ0VSVElGSUNBVEUtLS0tLQ==
  tls.key: LS0tLS1CRUdJTiBSU0EgUFJJVkFURSBLRVktLS0tLQpNSUlKSmdJQkFBS0NBZ0VBdnJhUk5xMXpaV0VJTEtEMFF6THdzMlpDeXpsZ3NzZlg2NmFpN2lJWXFaaWZmT1BICmlRUXgwTFMyd2ZEZEZOV0hxaUNYZWVFTWtRbE41dzFTR21sSWd3a3dUcWZ6RkJkaVJlajRKQVE5MUdtYXdMdkwKZ21jS0lsa2I3WHB0MFg2RUYzZnErQ3R3Vk9pY21ldGFneHVxbCsvN29jQjZiNFl0WWJycW9RRnYxaEdHNDljSQpFOGFqWExMNmpFWWp3Q0htcklLV1dlWE5jTTV2SXkvNXUyNVB3SzNTd05PTTU5ckJ3ejlCM2hBNGw1ejA2MEdsCjV2OGs4T1IzUGVHdFhuakg0bDU0TzgybGF5NHdUdTRUYmJEdVU4U0VuUjI4bTRGS281ajZTMm40NG5OWDRhYjEKSWRnWWdMSENDOTFLQ0J4WmNqcEtkUVZRenhLaE5OQ1RsVnVlRHNZWjg3dXBycVJFY0I0R0Q2cC9TWERlMGZpQQpqQ3dHM1pDZ1RqWkxOSUR1dWJGaFFxYWR0ZEM2THAvclpKSGt3WnNMMTh5RS8yY2ZlTU5OZmxTUE43cUsxK0hBCks2RnpLRW1Xa21MYXdqVU8vS0pyRWl3WWsxKzNxaSt2dDJkS3R5Vk16UHNQRjJLM2gzdFVad3FuM2E1WmFHY2IKeTFKSlhpUGE5MFNUdG42UWF4amtNTTRPK29LaC9XZTNSdGwyQWdseWVMSXN2QkRhUzRlQnBDcGpZTnVCRjBXNwpkMjMzZjIra3JseEdTMnZDRXhxdmtKMVc4S2VaQ0tHQVpQcldLTWRCeXRvR09kYk5rYjJtTWZMUFRyaUZEZmFrClAySmdPYkNvWWp0YTN5bUxhYlpFV3JjUkRWcjE3ZjJFR0VTYllFTFZueFEydTdXU3JsNEhuZG5PV04wQ0F3RUEKQVFLQ0FmODJZZGtHdm04cGVZSGJPQXB2SHhlRUVLVDdUbUZFbWJmNGVvdjdXNzJzbnRqYnhCZ2graEE2YzAycQpBQVVLNjlqRHFvZUhPYVZidGt1QWwwdlNQRE54S2kyY1FFZ1FjcHFUVk50dGFjZzN5ZVZYRURYMytXbnFZWDZWCk9WUVhhUHhCdFBCTDFCYzBIeUNJdzVRTHp0ZldlNWhGaDUxaUwrREEvWXZxWFg2R2pIanFmMmJPUE5aWW1MRFoKVHliaW9zZ2thUmgyaWhFTEdkS1hOaGNBVzNSaWZTNmJ6YmRnWmdEYXJDOGNJNFAvdDhJZlU1ajdSY1pDNnVNVgp3a1N0cThOVHlaeC9jU1M2YTNGYkVJaDV2dm8yNk5MbitwTE54UkNEbGh2SlpXNlRKRkRyQjdEZTljQUc4cWxpCnVMZGptTEhvNExaYXFDbGk0dTBWSW1Uek1pemZpeWd5eW52bXN3bmF2M2cyWkNNTFhxYytjYmorbG91NlVvMW4KUEw0bWtuTmRqNXVGdG9EcC9ieGhZSnArTkY5R3RLZWdhWVBrclN0MDF4T0llV05NdmhaNllnOGtxMHBnaGdEaApZaFNDWE1FdmVnWWUzcTZTWUJVcDlNS0Z2R3ZZWTdFbEk1bGsrY2ZNUDRkRlppTEF4SDhUYVBTQzQwVTBpS3ZOCnRlbUQ3TS92YUdhT1FkV2Z6QUtteU00K1NwRHhFdlNCL0txTHNmSlpuY2ZRem5kbmQxUXJOU1JCMS96SS9NSGsKekdkc2lCOWxkVHo0Q3JldlFtSm0zN0JXRllVZitld0tOMllKeUtCWE0wSXhOZkh4OEdCUVU1ZGx3c05Nd2FkVwpVUnd6Uk9BNU1MYk00S0dKSWoycXlWN01WTnZNS0g1a3JkbmxxaTU3Y0ZOQVBJVzlBb0lCQVFEd1R4UmVRZm5QCnRJY2kvTHJKSTY3eTJxK3JUSjQwdkhGNEpBc2hRbDZaSGFITzY3dVBTb1hZUFI1V2svWS9KekhyRFFTM3RHTk4Kb1k5MlhLYTJmRlliaXRWUHkwUHVyck93NExRRTBidVZ4UE1NOUJ6dkJhUUd4L0pQYWs0RFNiZDR6ZDVSZFh3cwpESFVlQmZQT1ppcGlEYTdBMitzbmNqOHJnaGRIcmVWb3FqV1RKaHlKSVEvM2s2Nit0NXVKa2RJcGJSck8zTWFqCjRQd0RRZ1dNMHlRK0hSS1JFZDZWdUl3OHZkNThmVjJuNUd0ZWNnRCtXYU01aEcwdnF1VlJndEJLNFBCbWh3d1IKaUE2TVhzeWgyb29LVUVscWpwckpCTEpuYkIzaGtCZklCdlRpdm9iTzA0OUVmZjZ2ZVFIR1RJcW1zYkJhQVo5Ywp3b01seXRqN2VXQjdBb0lCQVFETEtuWlhnVldBbS83RlRCbkFraStDbFVVSEx1SmJHRGYwYnpKSklBWGx0VUh0CkV3czZUWHRpNi9yWkxjMnVaRFE4Z3V0aytaN3ZNRXQxSjZTWFZrM2dSVithbCtHak9BVUhtR1hIWVJNWkdvY0UKOHh3dnZUaktYZ3ZualdDalV4L21PbTJBYWlWNHIxOXFNSk1OUGFrZDFRcWlvbGZ6Zngrb0g1VUJac2NpeHlNMwphZ3J5NTFHSTFLQUpsbDNTVUJVVS9GQVFTcFY1UmsyS3plS3huWVordy92UG9QUVRBU0RnVjNLL0xkanpDMzhmCnRzM0RLRmdJSTlLbUhtZm84KzJLUGJwbmNFSXNKWVBNL0dBcDVqSE1wQm5CQlJPVlNjZ1lUWGtPbEZ6WWU3YzUKY05IVk1ZWHRLM3Q3Z0VVZzA2UjBpelJDSGd5Q25vNitMRkt1QitpSEFvSUJBQlVjL1k4aWdNNU04Q3FVeGR4eQpOQ2JHSy9VQzhFdDEyd3BSTUdFbHNhWUdRbmNwb3ZyOTh6Q0NmaTNoSmh0NldCcHN0R05uaCtvRUxkU2FZMU5aCkxUK1NQUmVicGtaTU55RnRQS1BId1pGeEVtR3ErUGZQS1JBbmRSU2hKR1dKam9NZ28wM0k0cllFQ2k0dkc3S2cKcTB5ZUl5SnlzQUJ4T3plWllHNDl5eEFkRkVQdmIwWmxEMEFUUzZFYUJLSmJtM2xrU3B5dUxRMnM4TGRnajVoRgozTU5RVHBkTVdLQVM3TTlSWjBETXl2TzdUK3VtWEl2OFdDanZoNkJPaFlOWjJPOGJRRVBoelora3NwS0dxYmYyCnVYWWFnN2pHK3JaNm9Tb1JCN2NQcitjMUpVTGV0bjFwZlFicGd2enJ4c29qWTNNdDNXNnJBZE5taTV0QWJUdjYKK0EwQ2dnRUFLdDdCN0FNUmpMcmVEcm5aTVVabm1oRnZhRzJmUEFPblF2LzN4M0JuYmlwS1NBRmR2Q2EvWTRkUgovbVBvNCtTbkZTRzNGQzZNT0FLajJZdk04bFkyeHAwODZEMG1VcSt1ZUFUVGJUZnh0TGxoUmswYVpJUjBLRmVpCkFYRld0QUFGV2lwNEVzSFRPRjBoTUNJaDFZaHVXQ290UFZZdVl1WXZRdVd2Sm9XT2Zhc3hwaTdOTXFaWEVSMTAKeTdFY0NSWDI3TiswOFVzYnNXU0JWa25OalJjbFd6aDF1VUZJWDM0OGRycGRMelE0ZEVpM3dYUnNoTUxObUtJZQpnQWtvZWdLRzNFWGNRSmx5alNnNVlKYmNuOXJBSldOM1A4Q1hla2dBWGdoekEvMlFmZW5WSnR6RW1rMEI2cUxqClFwTTFneERGd2dYaHVCWHBJK0xiVWd1K1FwVE9ud0tDQVFBZCtTSnBEUi9KNTM3K1VZNVUxMUhML0VoRVVOWGsKVzZ5NVVEbkx2a0U0Z2Z4azhmQitJc085c2pjN0VncnFXRHdnUDBSbDJlUTlIbHNWMnp1RjhlL3hVWm5sejNweQpLOW5qY1ZzMW8wZExIQUlFYkQrMW4vTS8xaFV4UC9EMVBrdEkvdTZjVTVCbWZyNkVWWFVJekZ6SjFycXJSRXZLCk9GQUd5VnZCeHhlREhYUWpycGs5U0xiclVoYlJVRkhoWTFDQU13dFYxVENpU0hGYUNPQW1Td1V3KzJZUkU0Y0wKV0xwWVBxa1N4OWRUWW03UXVTcjh1cFkwaGE0MHRZVkgvMHVVNExoUlJlcHoyN1JJTnZhRURRbnR6ZXU5MmhaNwpSbC9WNTQ1R1lUWG1qUFJpeVRDV0YzdlZheDU4K2JzZ1FLeWo5a3A2QTlzYUtDNFpxV3VkTzhQZAotLS0tLUVORCBSU0EgUFJJVkFURSBLRVktLS0tLQ==
kind: Secret
metadata:
  labels:
    sealedsecrets.bitnami.com/sealed-secrets-key: active
  name: acpmp-sealed-secret
  namespace: infra
type: kubernetes.io/tls
EOF

echo "Installing bitnami for sealed secrets . . ."

# Login and Repo setup.
helm registry login --username ReadOnly --password W73YMw=Kdjql9i9RtnhqiVjt/PefGBmv https://philipsmadockerhub.azurecr.io

#To Avoid circular dependency issue.
kubectl create secret docker-registry acr-philipsixplatform --docker-server="https://philipsmadockerhub.azurecr.io" --docker-username="ReadOnly" --docker-password="W73YMw=Kdjql9i9RtnhqiVjt/PefGBmv" --namespace=infra

helm install sealed-secrets-controller -n infra oci://philipsmadockerhub.azurecr.io/ix/acpmp-infrastructure/helm/acpmp-sealed-secrets --version 1.0.0
```

<br>

### 5. Install Longhorn

```bash
# Set proxy settings
export HTTP_PROXY="http://amec.zscaler.philips.com:9480/"
export HTTPS_PROXY="http://amec.zscaler.philips.com:9480/"

echo "Installing Longhorn"

helm registry login --username ReadOnly --password W73YMw=Kdjql9i9RtnhqiVjt/PefGBmv https://philipsmadockerhub.azurecr.io
helm install longhorn oci://philipsmadockerhub.azurecr.io/acpmp/helm/infrastructure/longhorn --version 1.5.1

```

# Deploy the AcPMP Workloads

After the Sealed Secret script successfully completes, we now need to instruct Argo CD on the managed cluster what AcPMP state to download and where to download it from. Argo CD will then deploy it. ArgoCD can also sync any approved changes from your branch. Read more about how ArgoCD works [here](https://github.com/philips-internal/ix-component-argocd-sandbox/blob/master/README.md).

![Alt text](images/deployment-deploy-acpmp.PNG)

## Create Cluster Git Branch
First of all, we need to create the desired state of the workload that will eventually be deployed in the cluster. The desired state is the type of apps, services and associated configurations.

- Create a new branch of [this repository](https://github.com/philips-internal/hpm-acpmp-deploy-dev) by filling out this [form](https://github.com/philips-internal/hpm-acpmp-deploy-dev/issues/new?assignees=&labels=Customer-Onboard&projects=&template=customer_onboard.yaml&title=%5BCustomer+Onboarding%5D%3A+). The form requires data you have noted in an earlier step whne you registered the cluster, tenant and customer.

- Your branch will then be automatically created. It will host metadata that describes a `dev` release. The next Runbook update will support using a `Release Candidate` branch. The production Runbook will support the `Release` branch.

<br>

## Configure Observability Agents to Stream to Cloud 

  > ![](images/dev.png)
  >
  > This configures your cluster to send telemetry (metrics, logs, traces) to the `dev` observability stack. In the future, you will have access to the `secret value` to send telemetry to the `integration` observability stack. In production, there will be `production` observability stacks set up to be used. These manual steps will be semi-automated in future alphas. 

### Metrics:
- Inspect `infrastructure/infra/prometheus/values.yaml`

- Replace the secrets field with the following:
  ```bash
  secrets: | AgBmeNopojeaYrf66qd6CCMS6VA3VX+5bE8pwNfSOjF2cs25Y8YeOqinaEVmsx/HQaLG28DsUgdvBx/BdGvFtBygaBIPGtBK9XopceVpY2fdDm2cTqxzBDhnDopEm2T7L1B9hK/vV1TG0+Cc2HjsPm04jdUkm09GCmYj0LJD4oHj82t00UU7tRlGkTe2U7Vu3NlK5w6/aQjcH9HHoGWmP/S5gGE8xSFn/hVCna7iTgmSPu/g9/puerMAlUbanRdbFxPq88yLUKxDr/KoRpf1qF8r8Lw4q+wSm1bdFsXuZA3BKGWX0EH+/8kEQvoIsRQbxPOWbiVxHWsNfRgSq6USp39jiIuKAhwCMi8gToWhUQ32a1I/JA1gEFWwieJhZ959iqdPTOUxWKuTj9c7LB6c4vOD06+4NxkNeR/jOh7IlnW1inSYk9bNJ2zUm+VZZvG9+4Mhmdy9j918i6yerGx67xL14rkEIP2QegeYPXzz8Agi7qGeOAHfbywg0QeoQuGDtVxGXyLJUTx1rLbU6IgVELsT4/hM45jIDA9Wcg6GY1bSldyLI59+DCFl5okIXRAmsZTd88qZxMDiiX4SNSS8BOO9DGDoavl9Xyv19HDXkjLiPUwwYsSaTiSUqLeix9uszeu9F8JRMyXkXVW5WD3as4+8nNbT9ZH+6iF3gIICtit0n7DQ/5yh6Trdrac4n+BHr3yAb7KpKQaPCepZCaboaAChu5f28nq+HP56Uk8Y5LI8HGR+Sb8V5Aig46eu0ZTQ0v0T6ynOMfHESc75KLIgj7JgQCGeVCvjRmbeMCZorLlfDB7Kq4Vv3krkGVrPCfCWtnQ+Vgb+7FYkDYiOdg7agZ9xckMVtg==
  ```
- Ensure Prometheus is running in `agent` mode:
  ```bash
  --enable-feature=agent
  ```
- Ensure the cloud URL is set to:
  ```bash
  url: https://aps-workspaces.us-east-1.amazonaws.com/workspaces/ws-932f614a-56a3-49d6-99a1-136d18eb918a/api/v1/remote_write
  ```
 - Ensure the AWS region is set to:
    ```bash
    region: us-east-1
    ```
- This configures your cluster to send metrics to the `dev` observability stack.



### Logging:
- Inspect `infrastructure/infra/acpmp-fluent-bit/values.yaml`
- Ensure the AWS region is set:
  ```bash
  awsRegion: us-east-1
  ```
- Replace the secrets field with the following:
  ```bash
  secrets: | AgB8WfSquQKu+T6SNEiIxNVderHUI7XcA0nAkf7HVYdl70tcgwhIV1FeCzovSXFJrqO5Esv5fc8SXV6XFbfeSKk7qq2pG/qSXAkGduLlfQ7Mj+QmhYjniTQWP+iVLd0ZgenZsBX8fSQ1ZO1eTm84JzQK3ZsqGVtV1SnvF9mpTpod8xwmRaqcS9Pl68dn5OdLLxwHKrAYA47NF2mOTOS3GriOux3hhMunGu9CBLoYzMi1zaSBarp4bZdOUMZtfUOTXGL5T7XtuM5ya8thopKl9SSNzTOochywOWW58sAcfrVrXKzhfVphjiGm9Jp/wvpw91A0spKE3D1HSwY+PXHPMgM24ySlj9gsNf66cy5txFN0oyk/4ZKFFpNow7C9CvpyZM+pL2tOAW90ppCId/jc4/JtK5tRgCDMHl0o/NP0Uh70DLYG5rnTNCTRa/xaZIU7PAjCnojYnTntQFfJIVA2X/bmEdUrDUmgsDXnB5ibaJVTcyEmUIT3tbi9ARdu2fa797F+/MD0Yp1OZZD9zf3roOVjbpPK/Y5ZR1lts9xUaalERgQHwhXzHHszSBH41EA9E6WdDrq40uMZ2upT7R8kX6H1wJvVn6MyemcWOTVecQN7wBXshk8UImXHxDEf0omVuJd3XHykzMRaCnb5q7SZTxujo8UuyZ9XdfHmpyV7MiVY9hyGC9UXa0VEyj+N5Jtlyk2ACZWtJWPVSELuRUG80JACwU3rrYe/zY7mOAZSsekst+OYFV1RMMlUGZI0cy8KPz9xERKFjrZUQtUjOTijRVE/7vnyPNY2mosds7slzGoj8QI0RhPBH3wlVDLdiHdvs1jRpb+huolMeA704LqeaA3RC9W0
  ```
- This configures your cluster to send logs to the `dev` observability stack.

### Tracing:
- Inspect `infrastructure/infra/acpmp-opentelemetry-collector/values.yaml`
- Ensure the AWS region is set:
  ```bash
  awsRegion: us-east-1
  ```
- Replace the secrets field with the following:
  ```bash
  secrets: | AgAuy1tn9dxrv4bmuPA37tQzhZkmZvPhdWhRlMTkDGvklpAzLoGku3B618oQrl8mCgiD4PJAEviTuhlRCcZJArPHKP35EdrWMncyX4AXT4fxXy3fA95kxdwoBHCCG0JwSpS6oWN5ce6S4TfX7iMlnFZgLXs5tIlgYbxR5AVSJMXaEhfWBgEgI195JVStH1+5GUz6/V3jN8/zY9rq6gvKAp8ypwLk3+JvxwSoJZaxzF89psYJesJBPhgRqrwB9wwly1FLAqLYxDeF/S3cguZLEOYLhKrbklZaoBY48cWA/QjrQ3TFQex1jmWkbYnoEN2naPsA0jmwpM3XrC0aNJNn7PwJE4bGtq9MwwUr+ORHOETI2SuMAJLmE0VIQ+MD4c9Qj/cJWD8ImJWYEOSQsnoQCTdI6EV7eSfIbSD5+rqD/jKsbWmimVH74ddXtfuJG1syLNhqDfwPzsinRH0RRZdfHuFKuN9HUgdSXIfhXXCfkQiLoTdUetVZXQXzgSnZ/cAt/P3T8dzqLnLGjkorRQsyD8wTOw8hyYdptvZKKD/3fML4pI8EiztiFieiItfCV2OheDFZbRz0MZvDCFViqAR4uCtPpBgtEVbofyIjIjpI6vPWH3hO9t1HOMaBENPZpi65Vh+XdVnzFSadAPq8C3MDxcWyOxuuvnRu0fRsEaSXSIeKKwcGnAtZGyLCEjQnHoF6IyMHKHwlhcQf6X+ytrnDsQ1Bvun6baugTGykODgzn2dDE+3e0wkhapsgTuDkqct3xVh9w5s5xGd8fg41HT84qATLItss9xMXNZjU6OEhAoY2bDDjxNYCIHWFytO0vmNJdCgaL5Fs9lsyBNpIaMDRtIXOTUWYwB7AFA==
  ```
- This configures your cluster to send traces to the `dev` observability stack.

<br>

## Sync Argo CD with Cluster Branch
Now that the desired state of the cluster is configured in its Git repo, we need to instruct Argo CD to sync with it.

### Configure Argo CD to Sync Infrastructure

- In your preferred text editor, copy the content of this script and make sure to update `applicationset.targetRevision` to point to your cluster branch.
- Copy the updated script and paste it into the KubeCtl Shell:

```bash
export HTTP_PROXY="http://amec.zscaler.philips.com:9480/"
export HTTPS_PROXY="http://amec.zscaler.philips.com:9480/"

helm registry login --username ReadOnly --password W73YMw=Kdjql9i9RtnhqiVjt/PefGBmv https://philipsmadockerhub.azurecr.io
helm install acpmp-infrastructure-config --namespace argocd oci://philipsmadockerhub.azurecr.io/ix/library/argocd-config --version 1.0.40 \
--set project.enabled=true \
--set project.name="acpmp" \
--set project.description="Argocd Configuration for Acpmp workloads" \
--set applicationset.name="acpmp-infrastructure" \
--set applicationset.repoUrl="https://github.com/philips-internal/hpm-acpmp-deploy-dev.git" \
--set applicationset.targetRevision="******* COPY YOUR BRANCH NAME HERE ******* " \
--set applicationset.namespaceDepth=1 \
--set applicationset.applicationSetPath="infrastructure/*" \
--set applicationset.project="acpmp" \
--set gitConfiguration.enabled=true \
--set gitConfiguration.appName="acpmp" \
--set gitConfiguration.githubAppID="AgCYIPm+FYl5QT3zPkf6fR4xmf0zRqKqgwFOov5bvaFJOr9JsQxil5SVJb8A86+9lrJ/v4AM6HmYnXHirBuL7FX9n2SH2ZmhOLBmu3fZf994mXo/oqGKBgNjZSfgNdekPsRkyf+aTY4s3Zz6OeigWNDlw7iddqnYNeHOiidEXKmfLW8Ul/ZUu7UxuFYdFUIuje+EMXZP6oX92G0YqZN7mP7aVpjFAoNCp0bz6+2oi94WXZObcAeNr3S+x9ogZafJboMt5f7N6r0xuehCmNIobIujLUFjuu01eZ7c+UkTwQoN7u1OSCjmKYR7MaQHVAmns+JM7ZkH5iv3jgKcjo31nP3FTWwlnPzpGWXo0Rq2kVRKPSUFw4Qhce84AMaPbpnFlj1RIpGYkO5r5C/wKzFIdo7YumcLG4tT5xQFGxfB/ILGynLpVydaAjJgQaxw9arI8Z6dQnRw1gM+fIs+5NUuA2waC8xkjK7DSfL4HJ+giPD0tWdj6aTihfq75WJJ6VFxxEwsXFsZC/2G1nOy5/TRZr3sJGhZzdtyAoyw/my7aSiWLgA1AnBXzAbjiPFij88HNS0h0Yh1+qNJyyX2mnsvkg/9CAbZx+R3VeiFLkiBNChVtfvYhSiK4q1DMo2XfwCcahp9Z4bfzDP/0EZNSNOv1wBHU3dXHZlMp4dD8Mr4mX58n1qhAhaQ/pHCQ476k0DazCscNXWtQ04=" \
--set gitConfiguration.githubAppInstallationID="AgCSOmYX6m82xhKyve0pUoydJlqjeHIQzYKtu1UV9o72sCUo3jziXYcBilv0aAHHxl0OAqmEfp41CfOPHgkQZ06aXuXhx6pN+M58EVYD3FGmIUTrxCZlzAGePAvpEnPCW89hIVfJ4lH8MHZxfQtdZazb9UZZ2UAz1Ujx1EzGp6EGAdK3BVtaimJnikblrp+QMN5fYUBPHuw+gKHaazXRFY5QuxftGU2x7mD+59NQEI+ZIEVDIuyKs6Po1udvLYBKpZXtLrvNJCinnz7qEuYGf8WdPIRg7KWAnOqAB71ivUVb1zaE02/IfRUxVWwxKRR9U4RFPhmAId3L2mSE5S6T9xo0puTmfUfNDfVX7XsfqiIcLB1xrHM5OXEHRwACunpJL/mgnDHqPrR/2drBSLYK89lUgV995Kgtc5h3nsAcoffsown1sX40jWFr4KyYnumt5hBrW3N8MrwfJNLUuR6Wj1EuPBcsOSwxImD30MqaWmxNbPNxNX+YxjYr3qoNSoDW9qylCa6U1dW7+lMFrtbI8W6AEEY4tQSACcqIH1xQgTLSY8CNx3eTEgh3Ff8J3Ng6Dt5PTrpgkZsXjNkNbf4iMWp5yDklYbawKZiouMck3u0cfj7VkNzhrfTSKUVbHBUOQjcE4COHe8ur5RPo4ipfwngR8zUq7IoKfNprB6K3GLLpkJd4AOcUhkPrdSBq/6FrbuzvQCyhFWWg3w==" \
--set gitConfiguration.githubAppPrivateKey="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" \
--set gitConfiguration.project="AgAlfXwBO/2wxl9WZ25/mGjpPgEyKcwjwznoeifU844trXvvmBEqo+uGBcKsnV2s8mlNd9lSKqItVU7RJqtxU5JiV44zMVT51ZFLAOu7d6GP0ixtv7i9DIHH51H427FAbCVR6BFVy077xF02xobtAghJTWcTmyiLcqEXxFEGHz6MlrqioiowH3NrMR1ugFu/OgCkUg6y9uAGZxdEvWB51khnCnj/cxkiFuL6nasF7Eg3V5Uq16bFZlcYVviHOgGgTu9XNbBie53vDHK/OXbSKCg9boe5fRpgjc2kyv3vANX4k2K6/rojCwz+4hu6TfT3dH+VlxW7oCjZAq2mo7avc4aje6OQUSMci82NezNLz4MkAavcEMVca6AaxClg74Y+7mk+xeikfucBy6ZNRgkC/4VAi7F4zGIvebP1tnmCSXGPXt1XUhtvoWm9HeJCNidY1hwoNb25BecGze//0Sg2fNI/3sqZU38b6XoYZ7YdjDETQep56QTVQNch6pno4Yiio4iBJVLZiEzCAz2EhTDsFqqEh4FP5DeXtT3Sk7cvBCpEJwqI3yralRV47QS2U5NLtueX6cxKN7ir+xPE+KBgvFq+xIpXhf2Chkj6VERi+XsDwi58obl1CXX5OMrAvdhWUhEX/ZedpOFfdoVWTAWiVXh4WuCfJOn9C1VA3JMESVSh1UeNIQ4PwyR8TYlSvi0Qm9LQyYAJy85k" \
--set gitConfiguration.type="AgAm+Dp548mSN45u3c1UfJ/9MuF6384mAJtirK6UddQy53f9azlTWedNdAk35tuv8fPlPaDSIlBgTFZqOBhHbXjd0mVnrgu1l/6m3urtWYEcYT+1aOLaADt4HLYWFO8VjnfifIF8pj8DSfvWG7q2i2+hCQdCpnSrNn2EqYG7kG5KRRiry2YB47QCNDIT4dRYBK8U8b9qFItTdsgCW6Q+Pgzx41MWz14DHqwrZCVsDWKONqxx4BmAUquw5ZUT6l1UVtvLlUVr+DVjtcwmMEscTz5blrxK/aVpV5JaMEtxHF0+StuV483FWziJdfUikaB6kyCytSZ4UOdZIjnpQ4wXzUZGMng+8ZHREdly1726Ju52RJCd+Xo9LeGUjVZdwT2jLZ19KSvTdRc2DgMyVKbMG2HrPA7IS3cHV1wmhMk86uN7V3cpXQbUTJWaimqu2/boApxgsXx+pq4a2KbnYUdF85c63xCzQXnD8jLTrk8akI1PhjQwWUYjbXUmJEb28RerfcS8j0rQLiabnPMgr7qXFAqMkLqwms4rTXvaqH5+1tXF0b2I5m6pfuRm9KbsYy6tM91TIyIypt7eNpVO1plllfZOcBwTbObu2VeNe2lNd951rx8VPfOLH5YhMDpKW7TPUnytuuzXpwVawnHhAhb41Hbf55E38FQEK1i+6X+O+da75aZnxKqRbHH/wfOiGeRzZdVnvp8=" \
--set gitConfiguration.url="AgBzxZMTLg5UK6e7LqECLXQCv0oANDoemKHYiPUYjjxDSDSjJOiW18e4Vu9WR7/SJVmEchAqGyesJquskpXlR6UYELLJpL1GifufodVkXQONVF1I3LKUHa4SR84TWVATyUv1cSmpMGYNqy5Q/Z4f+Sb8Rvcr7INHTfG6FF4v+9dH19H8zud2Ajk5HDGJNQycO1qPdVvc1uQ2osHR+qS4gLed6ZBXq8bNK5HH6+vEreBLq7AWdQeywGEPGhIVuxnpV8r/D5LTUkaJHnMS4zNTBSscIeu1px85OIkXWM8FhJPecWBRfTtoBvu7VYaakcFlNv+Ud1e6hGXSXJ53Sb4wdJoYaI4OySVFEOZTSY5wD47F7L+UCRPBHMo2Q55uZGBMXhQuyrjcIeM8Mts1LDZ/tpnhn7zzglbfWYalCqqvvANOWotlg6QPgn+NCiadV2HCcTSr0wrwWDkxp7YhptznqiponAwhnsRGg2bP2ImeNyehUYh3PGATNFQTZeC+dfkGac6NYdmLWuG3ZcTGM+trhhgjJUcGQJFAXM/imnFrGdiACaxnSJO6Fzi1iBmSR347amJA/qdzxTkJbwevUCD3XOLtV3RLH/o043/hAUWk5jAvBFs4pWBlk+icWw+7TTncOP4uqFQO2H5q92oh1HMUE2zoW/qWJdLtbvgiC1diQIBFIfxOlXy9WXxSkIdIY+gsJEmIa7lnN/jJkfnDUO7a0s017jRwD+LkecmDhKm2eRjxUvF6NXcWQb3yboYyNoFGZat49JXVl4hA7iSmhTg=" \
--set containerRegistryConfiguration.enabled=true \
--set containerRegistryConfiguration.enableOCI=AgBS30g4+ZavSJjuFR6ZQfn2ZzHghsFOxJaG1DjfAAJHujk7iIom6iMMFuKS8P5sk1e4sjVrEP+E8BnJbqDDpppPcMQz0YPpecWX2bn9N+++Ap4NmPPth1K1wLhrakL3yh/nmbhsU729RCLdLIjaWexSVOSODkFncqvGN3+KI7K/+EGRALyx0RbwVwWIKFSPhRew9pJgPCTeFUgsTrFLWzsI9IZZdFhM937BU30eQMBhxeobrek0FzNcR5V+w941k1Y5ESQolwhHPQjOodBs6A+6ENcZvZD7rx4VXo4ulx1dTYBRnUE3s8eWkb2NZtMkVYxaiEdWlQLmAqOg3l5xBgMKV9imUaDPJnwj+1G8l9lfXXjwBb1y8GeWnvf4/iAY9CfqvVe6VVUKrvQrcpG1JJ9seckkKuHGrwnqqO3AZyngkzWJd91c1btzmiviYRXqL5fknhaz3Q+1Iw0230pfhtt6U0iBXeZEZ0ni5ip4P/MzzA3jrLQMHyzKDAAok46Baw/462/VKS4lqTtZC/qqtPgnp1jNpPMYSTSKPi1+5eMdkmXU1wkrQ8Ms87n8SXr0CNa04b/fIGbXjen6l6AUi7ajt+U6a4cK5JMlyGWac/m7TKOprio0Dsjv7c4Ha3OKZKz9xwrmJx2FioThGAGZ9/lNKKmVKvUpSgzY8vx6v6Fcd6Ga86U1IxUTfL2PBY5vi1OtpzSU \
--set containerRegistryConfiguration.name=AgB210WIYez8vVQsvUWxSTPMIrMSDux31ifOctTQqw7KGCK7lcAMJMLgWqrIBnYXdtZESduUWVjjAHyPhY9FNYzCDGmdUT5ZmWnZKrrg5dUOvndIspNH8gm10dGcj4lH425CwnAzDSrSSv9kqjaBCAHYIVL09/EfsywS1WQz9D3ExWjZnZ6XQl+erEQbqPbl9r2XPXUMX/kWDAK6UqdP+V4pbe0fazrl5ygnSQHw2N6Rs1LP5y0b5eUEh5W8eK6GaDPU5jzXYTgSbSY2z6ou7+m+cYO559khlvdIOsZtxSu7ooJ99o1hvqbk3fWHCyF6h4sXgK2ADoXy0vcmNpm+oQY1h+iCr+7wTqORHpdhSdaLQUc4rSaHhlGkYOABtTZovXoVPYqrgfsjC7yolZq2VVzk97p4iHPNbGwUd3qmlOyB3tk8ROmRJGRgV1MVeztq9gGVYI1uC/bQw4SNpyZsls3T0R/wzEF2Z/iYdBXSfZBE79E+tsrOtOLsQ0MrlGBDFBkbn5iLmWNAYB3FqXzcIV9ITHyw2Yp0cdlkQtu9Lz4niF23L0BZPBgBXB+lhaMdHjYVr3Rje9oA7SNC9LfUMcPU7WgNLt5ViaOSUCNp7KWj5Mw/FV89vldj2+2jTUU/NxyHjdNVlkhQbnz7OQq6vBOQ9V52AqrRCl7wVX32gJfHUZ/zNIFNvJAbwf7FxwE1IyK83Oo= \
--set containerRegistryConfiguration.password=AgC0u+yUSwMgwdvqUMY60oXHtAeWP7J+Z2PK0ZmMxwbpg1WzmK5ViJfbd3893zpB1O2Uz5FeOwR6VDKcd/GftlWCwHQHu6U5qWzRI58qWrr4Xow6dL1tZDCA6cyuSL1RSFfpeFAF/RiTiJMqDx0V7fSlSU8GD6bm4MCUSRiBFC5DcVzwPbztdHgQOMdHIDLHzssWmH1FDrQeS4RA+QHrgywCzTG5I11Uv0KHevLGpWKeo6fnWSqSTI3Zea9NgHu6tq6ra3oz34bMvwbqhqzUs1/irDNt4Kn7fAHp7wsppn+NA9arTqYb4rId+7os12QbE8YdRl3IOwxPDjwd6wbn0G5PWKVVUJYvfFu46Czh+HdPl/pJLcwRbCLlh5EZ5hdTc5w1Gp7AWf/ddJyW5SALecFzm1Sb79LqnUdZYWt14dYw47s9kejlJLQT3d9przG8Hout2IgcdQvtBJgZUZX5IKoneMmAfS1/ilgn3ELG/OnYD8ZdF5j859/wWMNq5cGUEYqJi3KpmKJKAySbXTaBrqBl+9TUjkV+YnQa59Y7XmOrbRT9WfaNzH/F0/sh+tdkeCBdsCsDsbY1TNvnnt+bfBOBAGu+grnoAZFOo0mNlXoQvU+vwOCrkpCgSygt9iWrYxHRHx14lnxHvequXLGf304K0cwrlwmjgemetTq8DzRvLzmrZh0tp+KyHElgJSILp+Zd9OMBMmDzMX8qpWgLo++7RZdq/eXy6V+dMEECabos9g== \
--set containerRegistryConfiguration.type=AgBl+YUBhg+5TK+rmLjHgDWm41dpctM2cKICWFNF9se/URTxr2RoZ/A44azJHu2DR5c+yNEyvlZjSkGtI6mprEHQSSX6Vnoy8TLMaEU/ywRHXh7ncuObS3Tj+mlKJ8iSiKMPJO4giI50tVK01RSNr6/rH50tKiSD9NcVxyJDPZ6HhjmNqTV1Jhh9szRR/OgocoHgaqZw6NbN0pBCLfcViu70vVjyrX50pJz6Zyd03g0LtSgtN34B3TVZ7b8ulBYxaYo7NdA6nIZmhDvHzraMst3QIdvTmk4U+zvOjomtMO4BK1az53MUqWmWYRnJL8gnqVNJvGBYXc5+i+DZGuIW0dh/X7MyxbHFNhLpHAhFx0TrUFX1/2QN7zi9JtusRDt3Zt5IykgxU49QGGNPDS91wSeS3DIR5zLqG8BoIPhCWWI/alpYxRMeVK2GpBvnwBl2l74fVkfQ0ss7Um8FERP1t2JgcqfwcF8ObrmyVB1V6HupGh1PaKALZMo7bNaQPrQWVCyz2HYhM62Hk55+3HDU2FoKJSN9pIdH+8jcCuNgB95yrgk0v1xvejZHrtGZRl54eDHKxLJDxIFH2i1d2K3byPgWzPgJgkc2v4lNvjREfYBX/zdVnFdwSPs9KL2Ejm6lxttsQQv/P8JuzwjIUbDNa+vVWBkl1TguqREUC91F/Nj4gMpm8weIScTcOh7V5SgM4Zho4Esl \
--set containerRegistryConfiguration.url=AgBHLb5hGhdRESlb3gtsGXLUqLp1zffpMMbOkFzRHhLOYZo8Fy35m0k3iBAC3NXUftoYgtlJ0ERWMFJnh/sg7M3wEoAT9LulGbltqS/+G7ZFAaRN0ZA8XOx9d8t0v7fsRxmWZ2bvDUoApoDcChU4iduYdG5cLyoa82cQepj0QAM+lKZpHMS5nKBZDiZvdPAZZ0EWrJx52wS8U7eil3b3Ti84ADwgwjajCLSYlLxUZAc3Aw4/bBg/+ZNED9LtFKibXvNV+Q2W6qljpbWUql6olquJWKEZMa3VEMDHgdtDCQZb+YvBtpQhS87p9CpgMDJziRpu6ygTVQBeqgAd+YeIr2uYUiwBVlcG5801rxKICb1bkYrHX1uBsrpzLDPzkW3R/eJcIVd76Uiav+HnLSXlPq6VmYtx7WktXwPDcMfXWvwZoOFFkIkrphIXYGsqvH9vF+H9dzEZbqSB4q3QWn5tP/DTa5qxzbETfZfls7UXIi/CEcV0XMH7scJacN1qdcEaKxLwQ6MHoDBwTgmHKP9Ked3SgbQxgvtLNbaDEnDOry6nPjfNmiZWCJZUdpPL2YeucHvDwDpGsIbLBLFNGE9wgJTS8hc+3yKTMkCNiPXKFjlfInEAi6pDu25o3DpgjprK4YIkFtbkCTf+PsocYQQ/+twUFnLOWw/4968zL5kaXbstMTheZ6LZwV8v/b3s4/ax4CnbPZqowucHgjumCoe74HXAeHOGuYIdh61NbuyZHw== \
--set containerRegistryConfiguration.username=AgCOfx8uflfEkhCSJ2a4VYUDaDrE2woJ/HaEl4dsit+Mu05j0lluZyPJj1MQaBSXBT/1pR5kUK/WTLMvDnjwa13mi/SwaMLtEMG1L3e2Ix5k1GlrLMYHzUJyOyD/5C2uXMwAekGmsGY4X6MvDusRcql1KrZFgbX0KeWRgJnk3xUhxINWd1ivQdgX0DPZ/08z/do1R7Bd3jiJYcH4wqceeJG2uay4e3bvLOgLhDPbZIHy42+FE2a4cd3AUID5Y/0aqB3U+7AeF+iT/nEbkmLD+IjBp8Jpx0vQPXCGRbUwzPJlt2mjcircqkwhN+Fx3nweSx+FJyG3/wdyoZcQ6dOqiB12tGcurGk7BNHmBw2DMpeQgwImop0Qn4OvM3/zT7MYcMIUYgJ+pvzGk1CMOeDkSwPRkhfVXyumqDoRaIi7WmN4UOjrtqQ/v0uLrEhSd0gXk6ISJL8ksJayY5ozvw3AkNUWoIaP7YOJEnXOzVTCuoWobF+al2I/KMVXusS3zo84Y+DhGFeRQtTvDlO9WevX/DzME4f7TEWY64NqZRZAvp7pefNggm7nnYVJ3d+9p8AppqI/JVfcmz7tTqI8iMLEbFFSBpGbEQyQJpDQFj4KTZRXZN/Um50Bw40VKqi3kWcgILgZyqBfsFSPavCpwFQkSkKMiORUm4pmc5NYunck9dK5C0jcOH3awN2Xhu2qOG6fLmdD9r8TH4fapA==

```

### Configure ArgoCD to Sync Applications

- In your preferred text editor, copy the content of this script and make sure to update `applicationset.targetRevision` to point to your cluster branch.
- Copy the updated script and paste it into the KubeCtl Shell:

```bash
export HTTP_PROXY="http://amec.zscaler.philips.com:9480/"
export HTTPS_PROXY="http://amec.zscaler.philips.com:9480/"

helm registry login --username ReadOnly --password W73YMw=Kdjql9i9RtnhqiVjt/PefGBmv https://philipsmadockerhub.azurecr.io
helm install acpmp-apps-config --namespace argocd oci://philipsmadockerhub.azurecr.io/ix/library/argocd-config --version 1.0.40 \
--set applicationset.name="acpmp-apps" \
--set applicationset.repoUrl="https://github.com/philips-internal/hpm-acpmp-deploy-dev.git" \
--set applicationset.targetRevision="******* COPY YOUR BRANCH NAME HERE ******* " \
--set applicationset.namespaceDepth=1 \
--set applicationset.applicationSetPath="apps/*" \
--set applicationset.project="acpmp"

```

## Verify AcPMP is Running

To verify that AcPMP has been installed, from `Cluster Management`, select your cluster.

As shown below, click on `More Resources > Argo > Applications`.

Verify that you can see the AcPMP components and that their statuses are `Synced` and the health statuses are `Healthy`.

![acpmp-verification](images/acpmp-verification.jpg)

<br>

## Update Certificates

These steps are temporary. In the next runbook revision, we will do this via GitOps.

Note: Generate offline certificate for your AcPMP cluster from PIC scep server or focalpoint. CN name should be: rndnest.philips.com
SAN should be: FQDN of all worker nodes.

The commands in this section must be run from a bash terminal. You can download Git Bash [here.](https://git-scm.com/download/win)

#### 1. Update PIC iX Root certificate

In order to establish mTLS connections between AcPMP and PIC iX services, you need to ensure that AcPMP can trust the CA that signed the certificates of the PIC iX endpoints. Follow these steps:

Update ```CaCertificate.pfx``` with the name of the ca certificate.

```bash
openssl pkcs12 -in CaCertificate.pfx -out ca.pem -cacerts -nokeys
```

The certificate file (ca.pem) contains bag attributes we need to remove.
These attributes start at the top of the file and go until the line that contains ```--- BEGIN CERTIFICATE ---```.
Please delete the bag attributes and save the file.
This is what the file should look like before the attibutes are removed.

```
Bag Attributes: <Empty Attributes>
subject=CN = Philips Healthcare Local Root CA, O = Philips, OU = Philips Healthcare

issuer=CN = Philips Healthcare Local Root CA, O = Philips, OU = Philips Healthcare

-----BEGIN CERTIFICATE-----
MIIF1jCCA76gAwIBAgIIGG5kz0gR8ekwDQYJKoZIhvcNAQELBQAwWjEpMCcGA1UE
AwwgUGhpbGlwcyBIZWFsdGhjYXJlIExvY2FsIFJvb3QgQ0ExEDAOBgNVBAoMB1Bo
0gAxaNEyU6VTZRY4+Wzc7UqwTNzYNYQZrlu0YqcuagxfeR7q17LBTvONrIYJNps1
yaSWktZzLSGBju67m7MvUcRO1zXVJ0q5LvHm7eINzrmq33aZQ8VUs1R8SAGPmFT3
0f0jn30fElHIwKk+vofYHY2yX/Y7ysy0RGIaloOezLxBUNUjgM+52qDXTwbyPdOK
JE6YemtrC5UFIA==
-----END CERTIFICATE-----
```

This is what the file should look like after the attributes are removed.

```
-----BEGIN CERTIFICATE-----
MIIF1jCCA76gAwIBAgIIGG5kz0gR8ekwDQYJKoZIhvcNAQELBQAwWjEpMCcGA1UE
AwwgUGhpbGlwcyBIZWFsdGhjYXJlIExvY2FsIFJvb3QgQ0ExEDAOBgNVBAoMB1Bo
0gAxaNEyU6VTZRY4+Wzc7UqwTNzYNYQZrlu0YqcuagxfeR7q17LBTvONrIYJNps1
yaSWktZzLSGBju67m7MvUcRO1zXVJ0q5LvHm7eINzrmq33aZQ8VUs1R8SAGPmFT3
0f0jn30fElHIwKk+vofYHY2yX/Y7ysy0RGIaloOezLxBUNUjgM+52qDXTwbyPdOK
JE6YemtrC5UFIA==
-----END CERTIFICATE-----
```

We need to base64 encode the certificate. To do this please run the command below.

```
cat ca.pem | base64 -w 0 > ca-base64.txt
```

To update the cluster's CA certificate, modify the command below with the contents of ca-base64.txt.
Once updated, please run this command against your cluster using the KubeCtl shell in Rancher.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  ca.crt: <copy the contents ca-base64.txt here>
kind: Secret
metadata:
  labels:
    isIntermediate: 'false'
    kind: certificate
    source: picix
    type: root
  name: picix-root-secret
  namespace: ix
EOF
```
#### 2. Update MDIP Root certificate

In order to establish mTLS connections between AcPMP and MDIP services, you need to ensure that AcPMP can trust the CA that signed the certificates of the MDIP endpoints. Follow these steps:

Update ```ClientCertificate.pfx``` with the name of the ca certificate.

```bash
openssl pkcs12 -in ClientCertificate.pfx -out ca.pem -nodes
```

The certificate file (ca.pem) contains bag attributes we need to remove.
These attributes start at the top of the file and go until the line that contains ```--- BEGIN CERTIFICATE ---```.
Please delete the bag attributes and save the file.
This is what the file should look like before the attibutes are removed.

```
Bag Attributes: <No Attributes>
subject=C = FR, L = Paris, O = Philips Capsule, CN = Test Root CA

issuer=C = FR, L = Paris, O = Philips Capsule, CN = Test Root CA

-----BEGIN CERTIFICATE-----
MIIDfTCCAmWgAwIBAgIUYGHiGv8Ms7vAjtD+SyuxZ//OSDMwDQYJKoZIhvcNAQEL
BQAwTjELMAkGA1UEBhMCRlIxDjAMBgNVBAcMBVBhcmlzMRgwFgYDVQQKDA9QaGls
aXBzIENhcHN1bGUxFTATBgNVBAMMDFRlc3QgUm9vdCBDQTAeFw0yMjA3MDcxMzU5
NTBaFw0zMjA3MDYxMzU5NTBaME4xCzAJBgNVBAYTAkZSMQ4wDAYDVQQHDAVQYXJp
czEYMBYGA1UECgwPUGhpbGlwcyBDYXBzdWxlMRUwEwYDVQQDDAxUZXN0IFJvb3Qg
Q0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQClyPJ5iB7AIStxoUw0
OT7ccLWGSoofJi/2pXgsy/sw+foLsgvEY8dsn2wCceWO/ozMglCi3kogzfqBu5q9
5PiHLFsnDpylXjjQhWHGdg6bS9aSKuEoH8jwmeIULw27sKrwxNEF1vxkv76atrnF
O9cDH1J3E8BaawNSTbzOe8P++EHpA57i466AakoASR2Yq8YtqqTTUmLNFDLciJli
OBI+h2onWDGLtD3V0YeUE8Kqjp1jzLRavsoYECnM0Kx9REExXAi18crRMEv/BG8A
vLpEq8PfJra8qqI3Lt+JSJ5bAZCb4xEKCuRCs9+VMxP7vPL22QfA4Ou6IhTqg8/0
28knAgMBAAGjUzBRMB0GA1UdDgQWBBSRuoJmZKkbWXGrkuazG9DIFWnfTjAfBgNV
HSMEGDAWgBSRuoJmZKkbWXGrkuazG9DIFWnfTjAPBgNVHRMBAf8EBTADAQH/MA0G
CSqGSIb3DQEBCwUAA4IBAQAPhlDLKFGLtm8b1WnsNvR84QgXMWjwWOq9RUJW25ME
NGay/U8TqaAJqNciGspWeX7JWcUCt8NocEj5l/uDYDTlXF88tgG6n0PfNKNWsYW1
XFtFijFQq+493/4NMTS5ikRXWfCi+MSoMCo7ZhlP8Knb+06rGSUCWAsWfhuipY8D
dOg6BT8ZwyN9HoW+zx/QjUCHSYTrts0O4A0SCnMlREKb4a8H5u3Eye2Tlt6DiRi0
ntjYTWEBr41/4D2QtyBC2wcjUP72pNM3uoX08aqUqUzXIo5C460nmAHpBXxNW+R/
xw8NrDXNlHXQTs5b1QQqpHPOBh3NDED0xJ3Gge9bxzS1
-----END CERTIFICATE-----
```

This is what the file should look like after the attibutes are removed. 


```
-----BEGIN CERTIFICATE-----
MIIDfTCCAmWgAwIBAgIUYGHiGv8Ms7vAjtD+SyuxZ//OSDMwDQYJKoZIhvcNAQEL
BQAwTjELMAkGA1UEBhMCRlIxDjAMBgNVBAcMBVBhcmlzMRgwFgYDVQQKDA9QaGls
aXBzIENhcHN1bGUxFTATBgNVBAMMDFRlc3QgUm9vdCBDQTAeFw0yMjA3MDcxMzU5
NTBaFw0zMjA3MDYxMzU5NTBaME4xCzAJBgNVBAYTAkZSMQ4wDAYDVQQHDAVQYXJp
czEYMBYGA1UECgwPUGhpbGlwcyBDYXBzdWxlMRUwEwYDVQQDDAxUZXN0IFJvb3Qg
Q0EwggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQClyPJ5iB7AIStxoUw0
OT7ccLWGSoofJi/2pXgsy/sw+foLsgvEY8dsn2wCceWO/ozMglCi3kogzfqBu5q9
5PiHLFsnDpylXjjQhWHGdg6bS9aSKuEoH8jwmeIULw27sKrwxNEF1vxkv76atrnF
O9cDH1J3E8BaawNSTbzOe8P++EHpA57i466AakoASR2Yq8YtqqTTUmLNFDLciJli
OBI+h2onWDGLtD3V0YeUE8Kqjp1jzLRavsoYECnM0Kx9REExXAi18crRMEv/BG8A
vLpEq8PfJra8qqI3Lt+JSJ5bAZCb4xEKCuRCs9+VMxP7vPL22QfA4Ou6IhTqg8/0
28knAgMBAAGjUzBRMB0GA1UdDgQWBBSRuoJmZKkbWXGrkuazG9DIFWnfTjAfBgNV
HSMEGDAWgBSRuoJmZKkbWXGrkuazG9DIFWnfTjAPBgNVHRMBAf8EBTADAQH/MA0G
CSqGSIb3DQEBCwUAA4IBAQAPhlDLKFGLtm8b1WnsNvR84QgXMWjwWOq9RUJW25ME
NGay/U8TqaAJqNciGspWeX7JWcUCt8NocEj5l/uDYDTlXF88tgG6n0PfNKNWsYW1
XFtFijFQq+493/4NMTS5ikRXWfCi+MSoMCo7ZhlP8Knb+06rGSUCWAsWfhuipY8D
dOg6BT8ZwyN9HoW+zx/QjUCHSYTrts0O4A0SCnMlREKb4a8H5u3Eye2Tlt6DiRi0
ntjYTWEBr41/4D2QtyBC2wcjUP72pNM3uoX08aqUqUzXIo5C460nmAHpBXxNW+R/
xw8NrDXNlHXQTs5b1QQqpHPOBh3NDED0xJ3Gge9bxzS1
-----END CERTIFICATE-----
```

We need to base64 encode the certificate. To do this please run the command below.

```
cat ca.pem | base64 -w 0 > ca-base64.txt
```

To update the cluster's CA certificate, modify the command below with the contents of ca-base64.txt.
Once updated, please run this command against your cluster using the KubeCtl shell in Rancher.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  ca.crt: <copy the contents ca-base64.txt here>
kind: Secret
metadata:
  labels:
    isIntermediate: 'false'
    kind: certificate
    source: mdip
    type: root
  name: mdip-root-secret
  namespace: ix
EOF
```
#### 3. Update MDIP Intermediate certificate (if needed)

In order to establish mTLS connections between AcPMP and MDIP services, you need to ensure that AcPMP can trust the CA intermediate that signed the certificates of the MDIP endpoints, if an intermediate was used. Follow these steps:

Update ```ClientCertificate.pfx``` with the name of the ca certificate.

```bash
openssl pkcs12 -in ClientCertificate.pfx -out ca.pem -nodes
```

The certificate file (ca.pem) contains bag attributes we need to remove.
These attributes start at the top of the file and go until the line that contains ```--- BEGIN CERTIFICATE ---```.
Please delete the bag attributes and save the file.
This is what the file should look like before the attibutes are removed.

```
Bag Attributes: <No Attributes>
subject=C = FR, L = Paris, O = Philips Capsule, CN = Test Intermediate CA

issuer=C = FR, L = Paris, O = Philips Capsule, CN = Test Root CA

-----BEGIN CERTIFICATE-----
MIIDczCCAlugAwIBAgIUTAvhaB8YLAugXL/kO+hJgfNdKNIwDQYJKoZIhvcNAQEL
BQAwTjELMAkGA1UEBhMCRlIxDjAMBgNVBAcMBVBhcmlzMRgwFgYDVQQKDA9QaGls
aXBzIENhcHN1bGUxFTATBgNVBAMMDFRlc3QgUm9vdCBDQTAeFw0yMjA3MDcxMzU5
NTFaFw0zMjA3MDYxMzU5NTFaMFYxCzAJBgNVBAYTAkZSMQ4wDAYDVQQHDAVQYXJp
czEYMBYGA1UECgwPUGhpbGlwcyBDYXBzdWxlMR0wGwYDVQQDDBRUZXN0IEludGVy
bWVkaWF0ZSBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALTUIROR
0XbJxREhBWOVkus1UnhBMcSJjAv+9PhVJF+Vq/tc2REPtJXAdUNnuCXmyggayo8O
WKR+YLrvMQN+9jeoFwsYpP/xQ4OuYpkTevtIiWoj+NinDAi9M9yYwc3Dj5vziI6k
I3t9lCIfnH0GbhK4jc2GO4feksmsHwcRXbVUG1BaJQeSEMvIEtkBbszznbd9gUK/
MUFMtb78wsHWQG0KRnRcWVPoLMpfS/9ZWLvlzW0TpVmUnzUQi83D8HLHSGJ+Z5On
dsMvjRmdyd6aUPcDVO5ZGxFwj7lR1FThvkS2RwaISW3fSoMfRBgrg2nJ01RGqyGT
G4QDQBhwZiqM8nMCAwEAAaNBMD8wHwYDVR0jBBgwFoAUkbqCZmSpG1lxq5LmsxvQ
yBVp304wDAYDVR0TBAUwAwEB/zAOBgNVHQ8BAf8EBAMCAgQwDQYJKoZIhvcNAQEL
BQADggEBAGr/RwX8E1HMAqyCvfiNiujoGCrbquq8Z4DAN8O/hpiaIeEa9s8uwJWt
uVN3tRaSR5sNMp6UnbuNGBTauwJgbev/apc0/EtdQzr9XUZO14lxWHT2oOHR04WO
3LVOsLhUYt3rswSUgURjOm5QQJTD/CuRSwZ/1APVw0raOCgdZZxsNmPSkY2Xk813
3NMiSpvRkPx+tOpufU2tt+aLq0J8XNr72Jp8lTinIo90Ff8PmTnTt4A4DXF2eWYX
8IPqmUwmt2bck4BJQOpupsnf48mo/ueVuuDHLjqMBjuBqq27Zc+8H2MFFtW6+XNL
mlYDooxZK9Yfhc35xwkTbNWUF5ETy+M=
-----END CERTIFICATE-----
```

This is what the file should look like after the attibutes are removed. 


```
-----BEGIN CERTIFICATE-----
MIIDczCCAlugAwIBAgIUTAvhaB8YLAugXL/kO+hJgfNdKNIwDQYJKoZIhvcNAQEL
BQAwTjELMAkGA1UEBhMCRlIxDjAMBgNVBAcMBVBhcmlzMRgwFgYDVQQKDA9QaGls
aXBzIENhcHN1bGUxFTATBgNVBAMMDFRlc3QgUm9vdCBDQTAeFw0yMjA3MDcxMzU5
NTFaFw0zMjA3MDYxMzU5NTFaMFYxCzAJBgNVBAYTAkZSMQ4wDAYDVQQHDAVQYXJp
czEYMBYGA1UECgwPUGhpbGlwcyBDYXBzdWxlMR0wGwYDVQQDDBRUZXN0IEludGVy
bWVkaWF0ZSBDQTCCASIwDQYJKoZIhvcNAQEBBQADggEPADCCAQoCggEBALTUIROR
0XbJxREhBWOVkus1UnhBMcSJjAv+9PhVJF+Vq/tc2REPtJXAdUNnuCXmyggayo8O
WKR+YLrvMQN+9jeoFwsYpP/xQ4OuYpkTevtIiWoj+NinDAi9M9yYwc3Dj5vziI6k
I3t9lCIfnH0GbhK4jc2GO4feksmsHwcRXbVUG1BaJQeSEMvIEtkBbszznbd9gUK/
MUFMtb78wsHWQG0KRnRcWVPoLMpfS/9ZWLvlzW0TpVmUnzUQi83D8HLHSGJ+Z5On
dsMvjRmdyd6aUPcDVO5ZGxFwj7lR1FThvkS2RwaISW3fSoMfRBgrg2nJ01RGqyGT
G4QDQBhwZiqM8nMCAwEAAaNBMD8wHwYDVR0jBBgwFoAUkbqCZmSpG1lxq5LmsxvQ
yBVp304wDAYDVR0TBAUwAwEB/zAOBgNVHQ8BAf8EBAMCAgQwDQYJKoZIhvcNAQEL
BQADggEBAGr/RwX8E1HMAqyCvfiNiujoGCrbquq8Z4DAN8O/hpiaIeEa9s8uwJWt
uVN3tRaSR5sNMp6UnbuNGBTauwJgbev/apc0/EtdQzr9XUZO14lxWHT2oOHR04WO
3LVOsLhUYt3rswSUgURjOm5QQJTD/CuRSwZ/1APVw0raOCgdZZxsNmPSkY2Xk813
3NMiSpvRkPx+tOpufU2tt+aLq0J8XNr72Jp8lTinIo90Ff8PmTnTt4A4DXF2eWYX
8IPqmUwmt2bck4BJQOpupsnf48mo/ueVuuDHLjqMBjuBqq27Zc+8H2MFFtW6+XNL
mlYDooxZK9Yfhc35xwkTbNWUF5ETy+M=
-----END CERTIFICATE-----
```

We need to base64 encode the certificate. To do this please run the command below.

```
cat ca.pem | base64 -w 0 > ca-base64.txt
```

To update the cluster's CA certificate, modify the command below with the contents of ca-base64.txt.
Once updated, please run this command against your cluster using the KubeCtl shell in Rancher.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  ca.crt: <copy the contents ca-base64.txt here>
kind: Secret
metadata:
  labels:
    isIntermediate: 'true'
    kind: certificate
    source: mdip
    type: root
  name: mdipintermediate-root-secret
  namespace: ix
EOF
```
####  4. Update AcPMP Client Certificate

In order to establish mTLS connections with external services, you need to ensure that AcPMP has a valid signed client certificate from a CA that can be trusted by the external services. The client certificate has two parts, the certificate and the certifcates private key.

We will first get certificate. Update `ClientCertificate.pfx` with the name of the signed client certificate file and run the command below.

```bash
openssl pkcs12 -in ClientCertificate.pfx -out client-cert.pem -clcerts -nokeys
```

The certificate file (client-cert.pem) contains bag attributes we need to remove. These attributes start at the top of the file and go until the line that contains ```--- BEGIN CERTIFICATE ---```.
Please delete the bag attributes and save the file.
This is what the file should look like before the attibutes are removed.

```
Bag Attributes: <Empty Attributes>
subject=CN = Philips Healthcare Local Root CA, O = Philips, OU = Philips Healthcare

issuer=CN = Philips Healthcare Local Root CA, O = Philips, OU = Philips Healthcare

-----BEGIN CERTIFICATE-----
MIIF1jCCA76gAwIBAgIIGG5kz0gR8ekwDQYJKoZIhvcNAQELBQAwWjEpMCcGA1UE
AwwgUGhpbGlwcyBIZWFsdGhjYXJlIExvY2FsIFJvb3QgQ0ExEDAOBgNVBAoMB1Bo
0gAxaNEyU6VTZRY4+Wzc7UqwTNzYNYQZrlu0YqcuagxfeR7q17LBTvONrIYJNps1
yaSWktZzLSGBju67m7MvUcRO1zXVJ0q5LvHm7eINzrmq33aZQ8VUs1R8SAGPmFT3
0f0jn30fElHIwKk+vofYHY2yX/Y7ysy0RGIaloOezLxBUNUjgM+52qDXTwbyPdOK
JE6YemtrC5UFIA==
-----END CERTIFICATE-----
```

This is what the file should look like after the attributes are removed.

```
-----BEGIN CERTIFICATE-----
MIIF1jCCA76gAwIBAgIIGG5kz0gR8ekwDQYJKoZIhvcNAQELBQAwWjEpMCcGA1UE
AwwgUGhpbGlwcyBIZWFsdGhjYXJlIExvY2FsIFJvb3QgQ0ExEDAOBgNVBAoMB1Bo
0gAxaNEyU6VTZRY4+Wzc7UqwTNzYNYQZrlu0YqcuagxfeR7q17LBTvONrIYJNps1
yaSWktZzLSGBju67m7MvUcRO1zXVJ0q5LvHm7eINzrmq33aZQ8VUs1R8SAGPmFT3
0f0jn30fElHIwKk+vofYHY2yX/Y7ysy0RGIaloOezLxBUNUjgM+52qDXTwbyPdOK
JE6YemtrC5UFIA==
-----END CERTIFICATE-----
```

Please make sure you save the file after making your changes.
We now need to base64 encode the certificate. To do this please run the command below.

```
cat client-cert.pem | base64 -w 0 > client-cert-base64.txt
```

Now we need to get the private key.
Update ```ClientCertificate.pfx``` with the name of the signed client certificate file and run commands below.

```bash
openssl pkcs12 -in ClientCertificate.pfx -out client-chain.pem -nodes
```

```bash
openssl ec -in client-chain.pem -out client-key.pem
```

```bash
cat client-key.pem | base64 -w 0 > client-key-base64.txt
```

We need to update two client certificates, the commands to update both are below.
To update the cluster's CA certificate update the commands below with the contents of <client-cert-base64.txt> and <client-key-base64.txt> (Ensure '<' & '>' are removed).
Once updated, please run this command against your cluster using the KubeCtl shell in Rancher.

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
data:
  certificate.pem: <copy the contents client-cert-base64.txt here>
  private.pem: <copy the contents client-key-base64.txt here>
kind: Secret
metadata:
  labels:
    kind: certificate
    shouldrenew: 'true'
    source: picix
    type: client
  name: picix-client-secret
  namespace: ix
EOF
```

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: ixtls
  namespace: ix
data:
  tls.crt: <copy the contents client-cert-base64.txt here>
  tls.key: <copy the contents client-key-base64.txt here>
type: kubernetes.io/tls
EOF
```
![](images/dev.png)
>
> __Verify IHE forwarder service ScepClientCertificateName is configured correctly__
> - Go to the newly created branch of your cluster and navigate to  `ApplicationSet/ix/acpmp-databridge/values.yaml `
> - Verify that the certificate name matches the CN name in the client certificate created from step 4.
>   ![image](https://github.com/philips-internal/hpm-fleet-registration-dev/assets/114990665/85456a56-cbda-41ca-b2a9-1c77299956a6)

<br>


## Update AcPMP Workloads

   > ![](images/dev.png)
   __Developer Note__: These steps apply to pre-production development only. The process for production updates will be detailed out in a future Runbook revision.

### Cleanup Database

---

 >
 > Currently, we do not support Database Migrations. So, for now, we have to manually delete the database schemas from the Postgres Database.

 > __Pre-requisite__: Install [pdAdmin](https://www.pgadmin.org/download/) in your machine.

- Connect To Postgres Server. As shown in the picture below, on the left-side menu bar, right-click on `Servers` -> `Register` -> `Server`

![postgres-db-connect](images/postgres-db-connect.png)

- In `General` Tab, Enter any value (eg: cluster's name) in the `Name` field as shown below.

![postgres-general](images/postgres-general.png)

- Navigate to `Connection` tab, enter the following details:
  - `Host name/address`: You can enter virtual ip address or Domain name if kube vip is configured or ip address or Domain name of any of the nodes in the cluster.
  - `Password`: Enter the text "password".

![postgres-connection](images/postgres-connection.png)

- Click on Save.

- Connect to the database. Expand `Servers` -> Search for the database by `Name` entered above -> right-click and click on `Connect Server`.

![postgres-connect](images/postgres-connect.png)

- Pop-up will appear asking for password. Enter text "password".

- In the Main view click on `SQL` tab.

![postgres-sql](images/postgres-sql.png)

- Copy the following commands to the editor and Execute.

```sql
DROP SCHEMA IF EXISTS ixiheforwader CASCADE;
DROP SCHEMA IF EXISTS TenantAwareServiceMonitor CASCADE;
DROP SCHEMA IF EXISTS ixaudit CASCADE;
DROP SCHEMA IF EXISTS tenant CASCADE;

DELETE FROM public."__EFMigrationsHistory";

SELECT case when COUNT(*) = 0 then 'Database was cleaned up successfully' else 'Database cleanup failed, please contact Developers' end FROM public."__EFMigrationsHistory";

```

![postgres-result](images/postgres-runscript.png)

- Verify in the 'Data Output` that the script ran successfully.

### Update Cluster Branch

---

- Refer to [this document](https://github.com/philips-internal/hpm-acpmp-deploy-dev/blob/main/docs/UpdateBranch.md) for instructions on how to update Acpmp Workflows in the Cluster's branches.
