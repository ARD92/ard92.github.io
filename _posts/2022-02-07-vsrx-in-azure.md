---
layout: post
title: Spinning up vSRX and other azure services using Terraform
tags: azure junos
---

# What is Terraform ?
It is an IaaC by Hashicorp which is mostly targetted towards all public cloud providers. While it has similarities in being
 able to configure but it is mostly viewed from a verisioning and provisioning tool which has the ability to do life cycle
management performing the CRUD operations. It is cloud agnostic and is widely adopted by many. Some of the aspects of the I
aaC are
* speed
* reliability
* Easy bring up of services
* provides an execution plan
* templateless design

## Providers
* Providers are modules you use to interact with a particular device/cloud
* Each provider has multiple resources
* Azurerm is a provider for bringing up various resources, services in azure

```
provider "azurerm" {
    version = "=2.33.0"
    features {}
}

```

## Resources

Terraform resource defines the infrastructure blocks. For example a vnet or vm in azure. A group of resources can be a modu
le.

Below is the syntax for a resource
```
resource "<PROVIDER>_<TYPE>" "<NAME>" {
[CONFIG …]
}

```

Example

```
resource "azurerm_virtual_network" "terraform-vnet" {
    name = "tf-net-1"
    resource_group_name = azurerm_resource_group.terraform.name
    location = azurerm_resource_group.terraform.location
    address_space = ["172.16.0.0/16"]
}

```
## Basic CRUD operation

* INIT: Initialize all the terraform files in the directory. Loads the terraform provider
* PLAN: checks what is the incremental change in order to provision what has been created in the .tf files.
* REFRESH: update local state
* APPLY: Applies the change that has been determined in the plan stage.
* DESTROY: Destroys all resources that has been created

## Variables

### Input Variables
These are input parameters. Variables can be predefined in the variables file and if not populated , we can pass the value
during run time.
You can set the variables using the cli as well when deleted from the root modules.
You can have a validation condition or apply variables during
terraform apply.

The file name for variables is terraform.tfvars as the below.

``` terraform apply -var="image_id=ami-abc123" ```


```
variable "zone" {
            description = "azure region"
            type = string
            default = "EastUs"
 }

```
Terraform loads variables in the following order, with later sources taking precedence over earlier ones:
* Environment variables
* The terraform.tfvars file, if present.
* The terraform.tfvars.json file, if present.
* Any *.auto.tfvars or *.auto.tfvars.json files, processed in lexical order of their filenames.
* Any -var and -var-file options on the command line, in the order they are provided. (This includes variables set by a Ter
raform Cloud workspace.)

```
variable "image_id" {
	type = string
	description = "The id of the AMI to use for the server."
	validation { condition = length(var.image_id) > 4 && substr(var.image_id, 0, 4) == "ami-"
		 error_message = "The image_id value must be a valid AMI id, starting with \"ami-\".
		 }
	 }

```

### Output Variables

These are return values of a module. Can be used to print after performing an apply operation or used to pass some values b
ack to the parent module
The child can be accessed by the following way module.<MODULE NAME>.<OUTPUT NAME>
If  we want to suppress sensitive values in cli use sensitive = true under the output block

```
output "instance_ip_addr" {
	value = aws_instance.server.private_ip
}

```

Sometimes you have a dependency on an another resource. When terraform plans the order of execution, if a resource is execu
ted when its dependent resource has not be created, it would error out. In such scenarios you can use “depends_on” argument
 which explicitly mentions the dependency.

```
resource "azurerm_virtual_machine" "tf_vsrx" {
  depends_on                      = [azurerm_marketplace_agreement.vsrx-tf]
  name                            = "tf_vsrx"
  location                        = azurerm_resource_group.vsrx.location
  resource_group_name             = azurerm_resource_group.vsrx.name
  network_interface_ids           = [ azurerm_network_interface.tf_vsrx_nic1.id,
                                      azurerm_network_interface.tf_vsrx_nic2.id,
]
```

when a parent module accesses an output value exported by one of its child modules, the dependencies of that output value a
llow Terraform to correctly determine the dependencies between resources defined in different modules. This should be last
resort and we need to explain why its used.

Local variables can be used by declaring locals in the main file and defining variables. The local then can be reused multi
ple times without having to declare again.

## Data sources
Allows to compute values which could be used else where within the terraform configuration.. A good example is rendering te
mplates which is common. You may want to have variables in an external file and render the template . Example is cloud init
. You may want to calculate the value and feed it to the cloud init file as a value to a variable. This would be rendered a
s a template which could then be passed.

```
data "azurerm_resource_group" "example" {
		 name = "existing
		 }
	output "id" {
		 value = data.azurerm_resource_group.example.id
	 }
```

## Networking related
There are some predefined functions in terraform which we could use.

### CIDRHOST
cidrhost(prefix, hostnum)

```
cidrhost("10.12.127.0/20", 16)
     10.12.112.16
```

### CIDRNETMASK
cidrnetmask(prefix)
```
cidrnetmask("172.16.0.0/12")
      255.240.0.0
```
### CIDRSUBNET

### CIDRSUBNETS

## Spinning up vSRX on azure using Terraform

```
                               +---------+
       +------------+          |         |          +-------------+
       |            |.5   .2.4 |         |.3.4   .5 |             |
       |   vm1      +----------+  vsrx   +----------+     vm2     |
       |            |          |         |          |             |
       +------------+          |         |          +-------------+
                               +---------+

```
Files considered:
* Main.tf
* Terraform.tfvars
* Az_route.tf
* Vm.tf
* Vnet.tf

Steps:
* Terraform init (under the current directory )
* Terraform plan -out=tfout
* Terraform apply
* Terraform destroy

The above topology would be deployed in azure east region. A vnet would be created with resources. Both the VMs are ubuntu
VMs and vSRX come up without accelerated networking enabled. The vnet is defined with 3 subnets and those will be used to b
ring up multiple NICs for the VM and VNF. The NIC is assigned with a private IPs. The management NIC is assigned with a pub
lic IP as well so that we can ssh.

Refer the files [here](https://github.com/ARD92/vsrx-azure-topologies)
## Cloud init
Cloud init files are passed as custom-data in azure. Below is an example for having cloud init .

Use the below az cli command to create a vm wih cloud init packaged.
```
az vm create --resource-group terraform --name ubuntu-cloudinit --image canonical:UbuntuServer:18.04-LTS:latest --custom-da
ta cloudconfig.txt --generate-ssh-keys
```

### cloudinit file
```
aprabh@aprabh-mbp: more cloudconfig.txt
#cloud-config
final_message: "Cloud init enabled"
apt_update: true
packages:
  - iperf3
  - netperf
```

# References
Pretty much all the information is from hashicorps official website
