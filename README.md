### ocp-power-ansible-automation

This project will implement an OpenShift UPI Cluster on IBM Power using PowerVC and Ansible Automation. 
At the end you will be able to login into the cluster with details provided from
 /usr/local/bin/openshift-install --log-level debug wait-for install-complete
However , no storage classes or htpasswd or image-registry are configured. 
This is to provde the base requirements for openshift on Power that will apply to all user requirements.

This solution is greatly simplified without the need for Terraform and complied dependencies.
The solution architecure is as below;

![ solution architecure ](https://github.com/ibellinfantie/ocp-power-ansible-automation/blob/main/solution_architecture.png)

		
### Flow of roles
The play calls a number of roles which are included in the custom ansible execution environment collection sbm.powervc_ocp.
There is some setup required in PowerVC, but once these resources are created , you wil not need to do this again unless you need a new version of coreOS and need to create a new image.
You can create multiple clusters on the same managed system using the same UUIDs of resources created in PowerVC.
The other requirements are easily dynamically updated in PowerVC if required to change.
Below is the flow of the roles and order of execution.

![ Roles workflow](https://github.com/ibellinfantie/ocp-power-ansible-automation/blob/main/roles_workflow.png)

The project pre-requisites are;

### IBM Power Systems.

This project should work on all versions of IBM Power from Power8 upwards.
However, only currently tested an OpenShift install on Power8 with OCP 4.12.0 . 
Will not work with the latest version of 4.12 or 4.12.30 on Power8.
Can work for versions of OCP just above 4.12.0 but not as yet tested all.
Please check on https://mirror.openshift.com/pub/openshift-v4/ppc64le/dependencies/rhcos/  for coreOS versions
Deployments on Power9 and Power10 will be confirmed shortly.

### IBM PowerVC

The version of PowerVC used for this project was.
IBM PowerVC for Private Cloud
Version: 2.2.0
Build: 20231102-0930

### PowerVC Images.

You will need to create PowerVC resources, which you can name what you like, as this project uses the resources UUID for identification.

base-rhel-8.8_Image	 <-- a base rhel image for the infrastructure nodes. Include  the ssh key for the ansible user in this image.

rhcos-4.12.30-ppc64le-openstack  <-- a ppc64le image for coreOS on IBM Power

Download the rhel image from RedHat and create an image as normal.

For the coreOS , download the image from RedHat here.

https://mirror.openshift.com/pub/openshift-v4/ppc64le/

1.  go to dependencies , rhcos, and download the openstack image for example, rhcos-4.16.3-ppc64le-openstack.ppc64le.qcow2.gz and copy to the PowerVC server into a directory.
2.	Extract the image and convert the CoreOS qcow2 image to a raw image with the qemu-img command on powervc, # qemu-img convert -f qcow2 -O raw <image>.qcow2 <image>.raw
3.	Attach a new volume with the required size for a coreOS VM ( I use 120G to ensure enoug space in /var ) to any existing PowerVC VM and restart it. Please take a note of the new volume.
4.	Once the VM is up, get the CoreOS raw image to this VM and dump the raw image to the newly added disk using dd, #dd if=image.raw of=/dev/device bs=4M  where device is the newly added device.
5.	Detach the newly added Volume, from the VM
6.	Go to PowerVC UI ->images and select create for creating a new image
7.	Specify image name and choose PowerVM for Hypervisor type, RHEL for Operating system and littleEndian for Endianness
8.	Select Add Volume and search for the specific volume name (where you dd-ed the CoreOS image ) and set Boot set to yes.
9.	Create the image by clicking on create


### PowerVC Project:

A project created in PowerVC for this OpenShift deployment. Must be able to see the above images.
Create a new user for the project. Give it admin priviledges.

### PowerVC network

A network identifed for this project. Identify the network UUID by using the openstack command # openstack port list . ( source your /opt/ibm/powervc/powervcrc first )

All VMs created will use the DHCP of PowerVC to allocate IP addresses. There is no manual allocation of fixed IPs.

NB. this is because we have to use supported methods in the play, using os_server and os_port ansible modules to create VMs which do not offer a method for fixed IPs from the PowerVC Cloud.

i.e. we could break out into a shell and run openstack commands but openstack server create is currently not supported by the IBM PowerVC team.

https://www.ibm.com/support/pages/powervc-openstack-commands

Only the below openstack commands are supported. Which we do not need to use as this project uses only the Ansible openstack modules.

Supported openstack subcommands

Command                 Description

openstack token issue	  Issues a new token.

openstack token revoke	Revokes an existing token.

openstack port list	    Lists ports.

openstack port delete	  Deletes ports.

However, using the ansible openstack modules means we don't need to manage unwanted ports, since these go away when the VM is deleted.

### PowerVC compute Templates

Configure compute templates for control and worker nodes.

You can use your own names as we use the UUID for identification.

The below ae minimum examples to get started, but check the UPI requirements from RedHat.

https://docs.openshift.com/container-platform/4.16/installing/installing_ibm_power/installing-ibm-power.html

ocp-control: VP=4 Desired EC=1.5 memory 32

ocp-worker: VP=4 Desired EC=1  memory 16

Use the ocp-control flavor for the bootstrap.

### PowerVC Storage

You would already have storage groups configured , so will use what's available.

You will need to create a single LUN of 300GB for the bastion node NFS server. This is allocated in the play and an NFS server is created.

Then can be used for a nfs-server storage class later.

Create 3 x LUNs for ODF also , since this project will deploy 3 extra workers that will not be immediately added to the cluster, but will be available for ODF later.

The OpenShift image-registry storage is not configured by this project, and is left for the admin implementation.

Once you have all the above set up in PowerVC, you can populate the group_vars/all/os_powervc.yml environment file for UUID values


### Environment Variables

populate these files;

group_vars/all/infra.yml 

group_vars/all/os_powervc.yml 

group_vars/bastion/ocp_vars.yml

All need to be populated with your values.

You will need an HMC user, with the ability to stop/start VMs.
The PowerVC project user with an admin role.
update the inventory for hostnames set in infra.yml.
Do not change the host groups as these natch for site.yml, just update hostnames in the groups.

Use group_vars/bastion/ocp_vars.yml to set your redhat pull secret and version of the OCP you want to install. The bastion root ssh key is dynamically provisioned so you do not need to provide this.


Unfortunately, the pause module does not work when run from roles in an ansible collection.
Pausing for 120 seconds
(ctrl+C then 'C' = continue early, ctrl+C then 'A' = abort)
You will see some occurances of this to give time for processes to complete. You can allow these to expire or use it as a break point to ctrl+c from the play. Fix the problem and run the play again.

### Inventory

Update the inventory for the hostnames you want for the infratrsucture nodes
Ensure these are the same hostnames as in the group_vars/all/infra.yml

### Ansible Execution Environment Image

requirements.yml
---
collections:
 - name: openstack.cloud
 - name: sbm.powervc_ocp
 - name: ansible.posix
 - name: ansible.utils
 - name: ansible.netcommon
 - name: community.general
 - name: ibm.power_hmc

requirements.txt
openstackclient
openstacksdk

bindep.txt
libxml2-devel
libxslt-devel
python3-devel
gcc
python3-lxml




### Running the play

The whole install of openshift will be performed by running the below command from the workstation or controller where the ensible xecutable environment image can be accessed.

ansible-navigator --eei ansiblehub.sbm.com.sa/ee-openstacksdk:latest run site.yml  -m stdout

The ansible execution environment 

When done, ssh to the bastion from the ansible user. 
sudo -i to root , 
export KUBECONFIG=/opt/install/auth/kubeconfig 
and you can run oc commands and access nodes
with core@"$infradid-cluster-domain" from the bastion.
 
There is no automatic CSR approval for the workers, so approve the certs manually when available.

Running the play a second time after the cluster is built and up and running, will shutdown and restart the cluster.
The cluster will be available again after this.
If you lose network connection or need to redo the cluster within 24 hours. You can delete the masters and workers , and re-run the play. 
The infra nodes can remain, and will use the generated ignitions.
If you want to update the certificates and get a new infraid for the ignitions, also delete the file /opt/install.tgz on the bastion and re-run the play. 
You can delete all nodes and re-run the play for the configured environment variables for a new build with a new infraid.


