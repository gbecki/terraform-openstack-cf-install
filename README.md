terraform-openstack-cf-install [![Build Status](https://travis-ci.org/cloudfoundry-community/terraform-openstack-cf-install.svg?branch=master)](https://travis-ci.org/cloudfoundry-community/terraform-openstack-cf-install)
========================

This is part of a project that aims to create a one click deploy of Cloud Foundry into an Openstack Tenant. This is (probably) the repo you want to use.

Architecture
------------

This will create a bastion host and CloudFoundry install that is confined to one subnet, pulling external IPs from a second subnet.

Upstream Terraform issues
-------------------------

Terraform does not yet officially support Openstack. Development of this terraform repo was done using an experimental provider available at [jrperritt/terraform](https://github.com/jrperritt/terraform/tree/openstack-gophercloud-v1.0) using the openstack-gophercloud-v1.0 branch. Hopefully that work will be merged into master at some point, and we could use that.

- [![hashicorp/terraform/issues/51](https://github-shields.com/github/hashicorp/terraform/issues/51.svg)](https://github-shields.com/github/hashicorp/terraform/issues/51) - the issue for having terraform natively support Openstack.

Upstream Cloud Foundry issues
-----------------------------

Issues/Pull Requests pending across Cloud Foundry projects:

None as of right now

Deploy Cloud Foundry
--------------------

### Prerequisites

You must manually create a "Tenant" to use within Openstack, terraform is unable to do that for you. Note the name and ID of the tenant for use later.

Unlike the AWS version of this, you do not need to upload your SSH key before starting. Note the path to the pem/private key files that you want to use and we will put them in the config file further down.

You **must** being using at least terraform version 0.3.7 with the openstack provider compiled and in the same directory as your terraform binary.

```
$ terraform -v
Terraform v0.3.7
```

You can install terraform 0.3.7+ via [https://www.terraform.io/downloads.html]

Your chosen region must have sufficient quota to spin up **all** of the machines. While building various bits, the install process can use up to 13 VMs, settling down to use 7 machines long-term (more, if you want more runners).

You must have a local install of git.

#### NOTE: Building the entire Cloud Foundry cluster will take a little over an hour! It hasn't hung (probably), it just takes a little while to do ALL THE THINGS.

```bash
git clone https://github.com/cloudfoundry-community/terraform-openstack-cf-install
cd terraform-openstack-cf-install
cp terraform.tfvars.example terraform.tfvars
```

Next, edit `terraform.tfvars` using your text editor and fill out the variables with your own values (Openstack credentials, region, etc).

```bash
make plan
make apply
```

After Initial Install
---------------------

At the end of the output of the terraform run, there will be a section called `Outputs` that will have at least `bastion_ip` and an IP address. If not, or if you cleared the terminal without noting it, you can log into the Openstack console and look for an instance called 'bastion', with the `bastion` security group. Use the public IP associated with that instance, and ssh in as the ubuntu user, using the ssh key listed as `public_key_path` in your configuration (if you used the Unattended Install).

```
ssh -i ~/.ssh/example.pem ubuntu@$(terraform output bastion_ip)
```

Once in, you can look in `workspace/deployments/cf-boshworkspace/` for the bosh deployment manifest and template files. Any further updates or changes to your microbosh or Cloud Foundry environment will be done manually using this machine as your work space. Terraform provisioning scripts are not intended for long-term updates or maintenance.

Cloud Foundry
-------------

To login to Cloud Foundry as an administrator:

```
cf login --skip-ssl-validation -a $(terraform output cf_api) -u admin -p $(terraform output cf_admin_pass)
```

You can also log into the cf api from the inception server itself, though you have to manually replace the calls to terraform from above with the actual values.

Cleanup / Tear down
-------------------

Terraform does not yet quite cleanup after itself. You can run `make destroy` to get quite a few of the resources you created, but you will probably have to manually track down some of the bits and manually remove them. Once you have done so, run `make clean` to remove the local cache and status files, allowing you to run everything again without errors.

Specifically, you may have to manually remove volumes, images (BOSH-*), security groups, key pairs, and floating IPs.

Module Outputs
--------------

```
bastion_ip
cf_api
```

### Configuration

Look in `variables.tf` to see all of the variables that can be set or overriden. The mandatory variables are:

```
network # The first two octets to use within the VPC, e.g. 10.0 or 192.168
auth_url # The API of your Openstack instance to auth against, like http://10.2.95.100:5000/v2.0
tenant_name # The name of the tenant to use - this must already be defined
tenant_id # The ID of the tenant, might be the same as the tenant name
username # User to authenticate against the Openstack API
password # Password for that user
public_key_path # Literal path to public ssh key file on the computer running Terraform - /home/user/keys/example.pub
key_path # Literal path to private ssh key file on the computer running Terraform - /home/user/keys/example
floating_ip_pool # Name of the subnet to use for floating/external IPs, e.g. "net04_ext"
network_external_id # UUID of the external network to use
region # Which region to use
```
