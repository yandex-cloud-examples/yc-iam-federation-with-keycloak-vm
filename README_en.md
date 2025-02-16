
# Deploying an identity federation in Yandex Cloud using Keycloak

## Contents

* [About the solution](#overview)
* [Solution architecture](#arch)
    * [`kc-deploy` module](#kc-deploy)
    * [`kc-config` module](#kc-config)
    * [Synchronizing data between the TF modules](#sync-data)
* [External dependencies](#ext-dep)
* [Deployment steps](#deploy)
* [Deployment results](#results)

## About the solution <a id="overview"/></a>

To provide corporate users with access to cloud resources, [Yandex Cloud](https://yandex.cloud) employs:
* [Organization service](https://yandex.cloud/docs/organization/). 
* [Identity federation](https://yandex.cloud/docs/organization/add-federation).
* [Identity provider](https://en.wikipedia.org/wiki/Identity_provider) (IdP).

An organization is a container for users. You can add users to an organization and delete them from it.

IdP is used for authentication. Usually, it is integrated with a user credentials repository such as MS Active Directory, a database, etc.

Identity federation acts as a connector between an organization service and IdP. A federation enables synchronizing user accounts from IdP to the Yandex Cloud organization.

Once the sync is successfully complete, you can [assign roles](https://yandex.cloud/docs/iam/roles-reference) (grant permissions) for cloud resources to these user accounts. Yandex Cloud supports [SAML v2.0](https://wiki.oasis-open.org/security#SAML_V2.0_Standard)-based identity federations. For [the list of IdPs](https://yandex.cloud/docs/organization/concepts/add-federation#federation-usage) that were tested to work with identity federations in Yandex Cloud, see the respective section. 

This solution implements user authentication as follows:
1. The user enters the Yandex Cloud console URL in the browser, specifying their federation ID, e.g.,`https://console.yandex.ru/federations/bpf3375ucdgp5dxq823tt`.
1. The cloud console redirects the user request to `IdP`, which is deployed as a VM running [Keycloak](https://keycloak.org).
1. On the IdP page, the user sees an authentication form where they need to enter their login and password.
1. The user enters their credentials into the form.
1. IdP verifies the user credentials and, if valid, returns the user, now authenticated, to the Yandex Cloud console.
1. Yandex Cloud performs user authorization to enable their access to cloud resources.
1. After successful authorization in the Yandex Cloud console, the user should see cloud resource folders they can access.

## Solution architecture <a id="arch"/></a>

The figure below shows the basic solution architecture.

<p align="left">
    <img src="solution.png" alt="Solution Architecture" width="1000"/>
</p>

This solution deploys IdP as a VM running [Keycloak](https://keycloak.org).

Our setup consists of two Terraform modules:
* [keycloak-deploy](#kc-deploy)
* [keycloak-config](#kc-config)

The reason for using two modules is that the [Keycloak Terraform provider](https://github.com/mrparkers/terraform-provider-keycloak) requires an already running (alive) Keycloak instance.

### `kc-deploy` module <a id="kc-deploy"/></a>

The `keycloak-deploy` module creates the following objects in Yandex Cloud:
* [Managed Service for PostgreSQL](https://yandex.cloud/docs/managed-postgresql/) cluster to store Keycloak configuration.
* [Let's Encrypt](https://letsencrypt.org/) certificate for the Keycloak web server.
* Static [public IP address](https://yandex.cloud/docs/vpc/concepts/address#public-addresses) for the Keycloak VM.
* [Security group](https://yandex.cloud/docs/vpc/concepts/security-groups) for the Keycloak VM.
* VM running Keycloak. 

For a complete list of module input parameters, follow [this link](./keycloak-deploy/variables.tf).

### `keycloak-config` module <a id="kc-config"/></a>

`keycloak-config` performs the following:
* Creates an [identity federation](https://yandex.cloud/docs/organization/concepts/add-federation) in Yandex Cloud.
* Creates a realm and related objects in Keycloak configuration.
* Enables certificate exchange between the Yandex Cloud federation and the Keycloak realm.
* Creates a test user account in Keycloak.
* Imports the created test user account from Keycloak to the Yandex Cloud organization.

For a complete list of module input parameters, follow [this link](./keycloak-config/variables.tf).

Upon completion, the module will generate a link, such as `https://console.yandex.cloud/federations/bpf3375ucdgp5dxq823tt`, which you will use later to connect to cloud resources through your federation. Following this link will take you to the Keycloak page prompting you to enter your credentials.

### Synchronizing data between the TF modules <a id="sync-data"/></a>

`keycloak-config` uses some input and output data of `keycloak-deploy`. 

To deploy the solution, make sure to first run `keycloak-deploy` and then `keycloak-config`.

To avoid errors when transferring data manually between modules, we recommend using the [sync.sh](./examples/keycloak-config/sync.sh) script to synchronize the required data from `keycloak-deploy` to `keycloak-config`.

## External dependencies <a id="ext-dep"/></a>

To deploy this solution, you need a pre-configured Yandex Cloud infrastructure. Provide the values of the infrastructure parameters to the above TF modules as input variables. 

Before deploying the solution, make sure the following objects are available in Yandex Cloud:
* Cloud resource folder named `kc_folder_name` where the solution components will be deployed.
* `Cloud DNS` [public zone](https://yandex.cloud/docs/dns/concepts/dns-zone#public-zones). Make sure to first delegate the `dns_zone_name` domain, i.e., the one you will create in Cloud DNS, from the domain registrar.
* Network named `kc_network_name` where the solution components will be deployed.
* Subnet named `kc_subnet_name` where the solution components will be deployed.

In the above list, the names of input variables for deployment from [keycloak-deploy](./examples/keycloak-deploy/main.tf) are provided in parentheses.

## Deployment steps <a id="deploy"/></a>

This solution is compatible with `Linux` and `MacOS`.

We did not test it in the Windows environment.

0. Before starting your deployment, make sure all the required tools are installed and properly configured:
* `yc CLI`: [installed](https://yandex.cloud/docs/cli/operations/install-cli) and [configured](https://yandex.cloud/docs/cli/operations/profile/profile-create#create).
* `Terraform`: [installed](https://yandex.cloud/docs/tutorials/infrastructure-management/terraform-quickstart#install-terraform) and [configured](https://yandex.cloud/docs/tutorials/infrastructure-management/terraform-quickstart#configure-provider).

1. Download the solution from its [github.com](https://github.com/yandex-cloud-examples/yc-iam-federation-with-keycloak-vm) repository:
    ```bash
    git clone https://github.com/yandex-cloud-examples/yc-iam-federation-with-keycloak-vm.git
    ```

1. Go to the folder with an example of deploying [keycloak-deploy](./examples/keycloak-deploy/):

    ```bash
    cd yc-iam-federation-with-keycloak-vm/examples/keycloak-deploy
    ```

1. **Important:**First make sure all [external dependencies](#ext-dep) exist.

1. In the [main.tf](./examples/keycloak-deploy/main.tf) file, check the values of the variables and correct them as appropriate. 

1. Prepare an environment for your deployment:

    ```bash
    source ../env-yc.sh
    terraform init
    ```

1. Deploy `keycloak-deploy`:

    ```bash
    terraform apply
    ```

    It may take up to 30 minutes to process a request for a [Let's Encrypt](https://letsencrypt.org/) certificate.

1. Check the status of the obtained Let's Encrypt certificate:

    ```bash
    yc cm certificate list
    ```

1. Go to the folder with an example of deploying [keycloak-config](./examples/keycloak-config/):

    ```bash
    cd ../keycloak-config
    ```

1. Synchronize parameters between the Terraform modules:

    ```bash
    bash sync.sh
    ```

1. In the [main.tf](./examples/keycloak-config/main.tf) file, check the values of the variables and correct them as appropriate.

1. Initialize Terraform:

    ```bash
    terraform init
    ```

1. Deploy `keycloak-config`:

    ```bash
    terraform apply
    ```

1. Optionally, use `yc CLI` to check if there is a Keycloak test account in Yandex Cloud:

    ```bash
    ORG_ID=$(cat terraform.tfstate | jq -r '.resources[] | select(.type == ('\"yandex_organizationmanager_saml_federation\"')) | .instances[0].attributes.organization_id')
    yc organization-manager user list --organization-id=$ORG_ID
    ```

## Deployment results <a id="results"/></a>

While deploying this solution, you will be creating the following object in Yandex Cloud:
* [Identity federation](https://yandex.cloud/docs/organization/concepts/add-federation) in the selected organization.
* [Let’s Encrypt](https://letsencrypt.org/) certificate for a VM running Keycloak in [Certificate Manager](https://yandex.cloud/docs/certificate-manager).
* VM running the Keycloak IdP, successfully integrated with the identity federation in Yandex Cloud.
* Yandex Cloud DNS record with the public IP address of the Keycloak VM.
* User account in the Keycloak IdP synchronized to the Yandex Cloud organization through the respective federation.

Once the deployment is complete, assign the required [roles](https://yandex.cloud/docs/iam/roles-reference) for the respective cloud resources to the user account created in the organization.
