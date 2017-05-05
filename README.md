---
services: azure-resource-manager
platforms: python
author: lmazuel
---

# Deploy an SSH Enabled VM with a Template in Python

This sample explains how to use Azure Resource Manager templates to deploy your Resources to Azure. It shows how to
deploy your Resources by using the Azure SDK for Python.

When deploying an application definition with a template, you can provide parameter values to customize how the
resources are created. You specify values for these parameters either inline or in a parameter file.

## Incremental and complete deployments

By default, Resource Manager handles deployments as incremental updates to the resource group. With incremental
deployment, Resource Manager:

- leaves unchanged resources that exist in the resource group but are not specified in the template
- adds resources that are specified in the template but do not exist in the resource group
- does not re-provision resources that exist in the resource group in the same condition defined in the template

With complete deployment, Resource Manager:

- deletes resources that exist in the resource group but are not specified in the template
- adds resources that are specified in the template but do not exist in the resource group
- does not re-provision resources that exist in the resource group in the same condition defined in the template

You specify the type of deployment through the Mode property, as shown in the examples below.

## Deploy with Python

In this sample, we are going to deploy a resource template which contains an Ubuntu 16.04 LTS virtual machine using
ssh public key authentication, storage account, and virtual network with public IP address. The virtual network
contains a single subnet with a single network security group rule which allows traffic on port 22 for ssh with a single
network interface belonging to the subnet. The virtual machine is a `Standard_D1` size. You can find the template
[here](https://github.com/azure-samples/resource-manager-python-template-deployment/blob/master/templates/template.json).

### To run this sample, do the following:

1. If you don't already have it, [install Python](https://www.python.org/downloads/).

1. We recommend using a [virtual environment](https://docs.python.org/3/tutorial/venv.html) to run this example,
    but it's not mandatory.
    To initialize a virtual environment:

    ```
    pip install virtualenv
    virtualenv mytestenv
    cd mytestenv
    source bin/activate
    ```

1. Create a Service Principal, either through
    [Azure CLI](https://azure.microsoft.com/documentation/articles/resource-group-authenticate-service-principal-cli/),
    [PowerShell](https://azure.microsoft.com/documentation/articles/resource-group-authenticate-service-principal/)
    or [the portal](https://azure.microsoft.com/documentation/articles/resource-group-create-service-principal-portal/).
   
1. Clone this repository and navigate into it.

    ```
    git clone https://github.com/Azure-Samples/resource-manager-python-template-deployment.git
    cd resource-manager-python-template-deployment
    ```

1. Create environment variables with the necessary IDs for Azure authentication.
    You can learn where to find the first three IDs in the Azure portal in [this document](https://docs.microsoft.com/en-us/azure/azure-resource-manager/resource-group-create-service-principal-portal#get-application-id-and-authentication-key).
    The subscription ID is in the subscription's overview in the "Subscriptions" blade of the portal.

    ```
    export AZURE_TENANT_ID={your tenant id}
    export AZURE_CLIENT_ID={your client id}
    export AZURE_CLIENT_SECRET={your client secret}
    export AZURE_SUBSCRIPTION_ID={your subscription id}
    ```

1. Run the script.
    
    ```
    python azure_deployment.py
    ```

### What is this azure_deployment.py Doing?

The entry point for this sample is [azure_deployment.py](https://github.com/azure-samples/resource-manager-python-template-deployment/blob/master/azure_deployment.py). This script uses the `Deployer` class
below to deploy the aforementioned template to the subscription and resource group specified in `my_resource_group`
and `my_subscription_id` respectively. By default the script will use the ssh public key from your default ssh
location.

*Note: you must set each of the environment variables (`AZURE_TENANT_ID`, `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET`) prior to
running the script, and either set `AZURE_SUBSCRIPTION_ID` or replace it in the script. See the numbered list above for instructions on how to do this.*

``` python
import os.path
from deployer import Deployer


# This script expects that the following environment vars are set:
#
# AZURE_TENANT_ID: with your Azure Active Directory tenant id or domain
# AZURE_CLIENT_ID: with your Azure Active Directory Application Client ID
# AZURE_CLIENT_SECRET: with your Azure Active Directory Application Secret

my_subscription_id = os.environ.get('AZURE_SUBSCRIPTION_ID', '11111111-1111-1111-1111-111111111111')   # your Azure Subscription Id
my_resource_group = 'azure-python-deployment-sample'            # the resource group for deployment
my_pub_ssh_key_path = os.path.expanduser('~/.ssh/id_rsa.pub')   # the path to your rsa public key file

msg = "\nInitializing the Deployer class with subscription id: {}, resource group: {}" \
    "\nand public key located at: {}...\n\n"
msg = msg.format(my_subscription_id, my_resource_group, my_pub_ssh_key_path)
print(msg)

# Initialize the deployer class
deployer = Deployer(my_subscription_id, my_resource_group, my_pub_ssh_key_path)

print("Beginning the deployment... \n\n")
# Deploy the template
my_deployment = deployer.deploy()

print("Done deploying!!\n\nYou can connect via: `ssh azureSample@{}.westus.cloudapp.azure.com`".format(deployer.dns_label_prefix))

# Destroy the resource group which contains the deployment
# deployer.destroy()
```

### What is this deployer.py Doing?

The [Deployer class](https://github.com/azure-samples/resource-manager-python-template-deployment/blob/master/lib/deployer.py) consists of the following:

``` python
"""A deployer class to deploy a template on Azure"""
import os.path
import json
from haikunator import Haikunator
from azure.common.credentials import ServicePrincipalCredentials
from azure.mgmt.resource import ResourceManagementClient
from azure.mgmt.resource.resources.models import DeploymentMode

class Deployer(object):
    """ Initialize the deployer class with subscription, resource group and public key.

    :raises IOError: If the public key path cannot be read (access or not exists)
    :raises KeyError: If AZURE_CLIENT_ID, AZURE_CLIENT_SECRET or AZURE_TENANT_ID env
        variables or not defined
    """
    name_generator = Haikunator()

    def __init__(self, subscription_id, resource_group, pub_ssh_key_path='~/.ssh/id_rsa.pub'):
        self.subscription_id = subscription_id
        self.resource_group = resource_group
        self.dns_label_prefix = self.name_generator.haikunate()

        pub_ssh_key_path = os.path.expanduser(pub_ssh_key_path)
        # Will raise if file not exists or not enough permission
        with open(pub_ssh_key_path, 'r') as pub_ssh_file_fd:
            self.pub_ssh_key = pub_ssh_file_fd.read()

        self.credentials = ServicePrincipalCredentials(
            client_id=os.environ['AZURE_CLIENT_ID'],
            secret=os.environ['AZURE_CLIENT_SECRET'],
            tenant=os.environ['AZURE_TENANT_ID']
        )
        self.client = ResourceManagementClient(self.credentials, self.subscription_id)

    def deploy(self):
        """Deploy the template to a resource group."""
        self.client.resource_groups.create_or_update(
            self.resource_group,
            {
                'location':'westus'
            }
        )

        template_path = os.path.join(os.path.dirname(__file__), 'templates', 'template.json')
        with open(template_path, 'r') as template_file_fd:
            template = json.load(template_file_fd)

        parameters = {
            'sshKeyData': self.pub_ssh_key,
            'vmName': 'azure-deployment-sample-vm',
            'dnsLabelPrefix': self.dns_label_prefix
        }
        parameters = {k: {'value': v} for k, v in parameters.items()}

        deployment_properties = {
            'mode': DeploymentMode.incremental,
            'template': template,
            'parameters': parameters
        }

        deployment_async_operation = self.client.deployments.create_or_update(
            self.resource_group,
            'azure-sample',
            deployment_properties
        )
        deployment_async_operation.wait()

    def destroy(self):
        """Destroy the given resource group"""
        self.client.resource_groups.delete(self.resource_group)
```

The `__init__` method initializes the class with subscription, resource group and public key. The method also fetches
the Azure Active Directory bearer token, which will be used in each HTTP request to the Azure Management API. The class
will raise exceptions under two conditions: if the public key path does not exist, or if there are empty
values for Tenant Id, Client Id or Client Secret environment variables.

The `deploy` method does the heavy lifting of creating or updating the resource group, preparing the template,
parameters and deploying the template.

The `destroy` method simply deletes the resource group thus deleting all of the resources within that group.

Each of the above methods use the `azure.mgmt.resource.ResourceManagementClient` class, which resides within the
[azure-mgmt-resource](https://pypi.python.org/pypi/azure-mgmt-resource/) package ([see the docs here](http://azure-sdk-for-python.readthedocs.io/en/latest/resourcemanagement.html)).

After the script runs, you should see something like the following in your output:

```
$ python azure_deployment.py

Initializing the Deployer class with subscription id: 11111111-1111-1111-1111-111111111111, resource group: azure-python-deployment-sample
and public key located at: /Users/you/.ssh/id_rsa.pub...

Beginning the deployment...

Done deploying!!

You can connect via: `ssh azureSample@damp-dew-79.westus.cloudapp.azure.com`
```

You should be able to run `ssh azureSample@{your dns value}.westus.cloudapp.azure.com` to connect to your new VM.
