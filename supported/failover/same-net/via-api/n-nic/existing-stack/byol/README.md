# Deploying the BIG-IP VE in Azure - HA Cluster (Active/Standby)

[![Slack Status](https://f5cloudsolutions.herokuapp.com/badge.svg)](https://f5cloudsolutions.herokuapp.com)
[![Releases](https://img.shields.io/github/release/f5networks/f5-azure-arm-templates.svg)](https://github.com/f5networks/f5-azure-arm-templates/releases)
[![Issues](https://img.shields.io/github/issues/f5networks/f5-azure-arm-templates.svg)](https://github.com/f5networks/f5-azure-arm-templates/issues)

## Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Important Configuration Notes](#important-configuration-notes)
- [Security](#security)
- [Getting Help](#help)
- [Installation](#installation)
- [Configuration Example](#configuration-example)
- [Service Discovery](#service-discovery)

## Introduction

This solution uses an ARM template to launch two BIG-IP VEs in an Active/Standby configuration with network failover enabled (the template now supports [multiple traffic groups](#traffic-group-configuration)). Each pair of BIG-IP VEs is deployed in an Azure Availability Zone when the region offers Availability Zones. Otherwise it is deployed in an Availability Set and can be spread across different update and fault domains. Each BIG-IP VE has 3 network interfaces (NICs), one for management, one for external traffic, and one for internal traffic.

Traffic flows from the client through BIG-IP VE to the application servers. This is the standard 'on-premise-like' cloud design where the  BIG-IP VE instance is running with a management interface, a front-end application traffic (external) interface, and back-end application (internal) interface. This template is a result of Azure now supporting multiple public-to-private IP address mappings per NIC.

This template supports creating up to 20 additional public/private IP address configurations for the external 'application' NIC to be used for passing traffic to BIG-IP virtual servers.

The template also supports updating the next hop of Azure User-Defined Routes (UDRs) to use the active BIG-IP VEs internal self IP address.

The BIG-IP VEs have the [Local Traffic Manager (LTM)](https://f5.com/products/big-ip/local-traffic-manager-ltm) module enabled to provide advanced traffic management functionality. This means you can also configure the BIG-IP VE to enable F5's L4/L7 security features, access control, and intelligent traffic management.

For information on getting started using F5's ARM templates on GitHub, see [Microsoft Azure: Solutions 101](http://clouddocs.f5.com/cloud/public/v1/azure/Azure_solutions101.html).

**Networking Stack Type:** This template deploys into an existing networking stack; so the networking infrastructure MUST be available prior to deploying. See the [Template Parameters Section](#template-parameters) for required networking objects.

## Prerequisites

- **Important**: When you configure the admin password for the BIG-IP VE in the template, you cannot use the character **#**.  Additionally, there are a number of other special characters that you should avoid using for F5 product user accounts.  See [K2873](https://support.f5.com/csp/article/K2873) for details.
- As of version 7.4.0.0, Azure ha via-api templates now make use of the Cloud Failover Extension. In order to successfully implement CFE in Azure, you need a managed identity with sufficient access. By default, this template will automatically create and assign a role the system-assigned managed identity to the BIG-IP virtual machines; the user creating the deployment **must** have permission to create a role assignment in Azure IAM. **NEW**: The solution now also supports specifying the name of a pre-existing user-assigned managed identity; when specified, the system-assigned managed identities and roleAssignments are **not** created. The user-assigned managed identity must be granted the Contributor role to both the deployment resource group and the existing virtual network resource group prior to deployment. See the Azure documentation for more information: [Assign access to an Azure managed identity](https://docs.microsoft.com/en-us/azure/active-directory/managed-identities-azure-resources/howto-assign-access-portal). Read more about Cloud Failover Extension on [CloudDocs](https://clouddocs.f5.com/products/extensions/f5-cloud-failover/latest/).
- To indicate that the next hop of a user-defined route (UDR) should be updated during failover, you must configure each UDR's parent route table resource with an Azure tag with the key name **f5_ha** and the value of **self_2nic** (this is the default self IP name created by the template; if necessary, this can be the name of a different self IP address onfigured on the the BIG-IP VE).  You must also create an Azure tag with the key **f5_tg** and value of **traffic-group-1** (or the name of a different traffic group you have configured on the BIG-IP VE).  For details, see [Customizing the Next Hop of Azure User-Defined Routes](#customizing-the-next-hop-of-azure-user-defined-routes).
- When failover occurs, UDRs will be updated across multiple Azure subscriptions.
- This solution uses calls to the Azure REST API to read and update Azure resources such as storage accounts, network interfaces, and route tables.  For the solution to function correctly, you must ensure that the BIG-IP(s) can connect to the Azure REST API on port 443.
- This solution uses calls to the Azure REST API to read and update Azure resources, this has specifically been tested in Azure Commercial Cloud.  Additional cloud environments such as Azure Government, Azure Germany and Azure China cloud have not yet been tested.

## Important configuration notes

> **_CRITICAL:_**  As of Release 6.1.0.0, BIG-IP version 12.1 is no longer supported. If you require BIG-IP version 12.1, you can use a previously released ARM template.  To find a previously released template, from the **Branch** drop-down, click the **Tags** tab, and then select a tag of **v6.0.4.0** or earlier.

- All F5 ARM templates include Application Services 3 Extension (AS3) v3.43.0 on the BIG-IP VE.  As of release 4.1.2, all supported templates give the option of including the URL of an AS3 declaration, which you can use to specify the BIG-IP configuration you want on your newly created BIG-IP VE(s).  In templates such as autoscale, where an F5-recommended configuration is deployed by default, specifying an AS3 declaration URL will override the default configuration with your declaration.   See the [AS3 documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/) for details on how to use AS3.   
- There are new options for BIG-IP license bundles, including Per App VE LTM, Advanced WAF, and Per App VE Advanced WAF. See the [the version matrix](https://github.com/F5Networks/f5-azure-arm-templates/blob/main/azure-bigip-version-matrix.md) for details and applicable templates.
- You have the option of using a password or SSH public key for authentication.  If you choose to use an SSH public key and want access to the BIG-IP web-based Configuration utility, you must first SSH into the BIG-IP VE using the SSH key you provided in the template.  You can then create a user account with admin-level permissions on the BIG-IP VE to allow access if necessary.
- See the important note about [optionally changing the BIG-IP Management port](#changing-the-big-ip-configuration-utility-gui-port).
- This template supports service discovery via the Application Services 3 Extension (AS3). See the [Service Discovery section](#service-discovery) for details.
- This template supports telemetry streaming via the F5 Telemetry Streaming extension. See [Telemetry Streaming](#telemetry-streaming) for details.
- This template can send non-identifiable statistical information to F5 Networks to help us improve our templates.  See [Sending statistical information to F5](#sending-statistical-information-to-f5).
- This template supports disabling the auto-phonehome system setting via the allowPhoneHome parameter. See [Overview of the Automatic Update Check and Automatic Phone Home features](https://support.f5.com/csp/article/K15000) for more information.
- This template can be used to create the BIG-IP(s) using a local VHD or Microsoft.Compute image, please see the **customImage** parameter description for more details.
- In order to pass traffic from your clients to the servers, after launching the template, you must create virtual server(s) on the BIG-IP VE.  See [Creating a virtual server](#creating-virtual-servers-on-the-big-ip-ve).
- F5 has created a matrix that contains all of the tagged releases of the F5 ARM templates for Microsoft Azure and the corresponding BIG-IP versions, license types and throughput levels available for a specific tagged release. See [azure-bigip-version-matrix](https://github.com/F5Networks/f5-azure-arm-templates/blob/main/azure-bigip-version-matrix.md).
- F5 ARM templates now capture all deployment logs to the BIG-IP VE in **/var/log/cloud/azure**.  Depending on which template you are using, this includes deployment logs (stdout/stderr), f5-cloud-libs execution logs, recurring solution logs (failover, metrics, and so on), and more.
- Supported F5 ARM templates do not reconfigure existing Azure resources, such as network security groups.  Depending on your configuration, you may need to configure these resources to allow the BIG-IP VE(s) to receive traffic for your application.  Similarly, templates that deploy Azure load balancer(s) do not configure load balancing rules or probes on those resources to forward external traffic to the BIG-IP(s).  You must create these resources after the deployment has succeeded.
- See the **[Configuration Example](#configuration-example)** section for a configuration diagram and description for this solution.
- The default PAYG images have been updated to "F5 BIG-IP BEST with IPI and Threat Campaigns".  Note: that changing the image can affect re-deployments  (see above). If you have an existing subscription and need to preserve the image id/name, use the custom image parameter (template parameter = customImage or customImageUrn).
- All Azure ARM templates now allow you to deploy any public or private offer image from Azure marketplace. Specifying the offer URN in the customImageUrn parameter overrides the choices for bigIpVersion and imageName parameters. See the F5 Azure offer list here for a list of current offers: [Azure offer list](https://github.com/F5Networks/f5-azure-arm-templates/blob/main/azure-offer-list.yaml). NOTE: Not all versions of BIG-IP are supported with all templates. Check the supported versions matrix on each template README for a list of supported versions for that template.
- This template has some optional post-deployment configuration.  See the [Post-Deployment Configuration section](#post-deployment-configuration) for details.
- This template now supports associating Azure NIC IP configurations with up to two BIG-IP traffic groups, allowing each BIG-IP VE device to process traffic for applications associated with the traffic group for which the device is active.  See [Traffic Group Configuration](#traffic-group-configuration) for instructions.
- Persistence and connection mirroring are now supported in this template.  It also supports mirroring of APM sessions.
- The BIG-IP VE failover log can be found at **/var/log/restnoded/restnoded.log**.
- When you deploy this failover template, you may receive an error due to existing error logs in failover.log (*Long running operation failed with error: An error occurred*).  This is a known issue and comes from the Microsoft Azure API when it takes longer than expected to send/retrieve requests.  The processes will complete despite these error messages.
- When configuring the template, you have the option of selecting 0 (zero) public IP addresses.  If you select 0 public IP addresses, the BIG-IP systems **will not** create any additional IP configurations on the Azure network interfaces; however, failover of UDR next hop is still supported.  If you create IP configurations post-deployment and would like them to failover, see the [Post-Deployment Configuration section](#post-deployment-configuration) for details.
- This template may deploy additional NICs using the parameter **numberOfAdditionalNics**, if doing so it will preconfigure the BIG-IP VLAN(s) and place the interfaces into the corresponding VLAN based on the subnet name(s) provided in the parameter **additionalNicLocation**.  Be aware that after the template deploys, the BIG-IP self IP address(es) need to be created that correspond to the Azure IP Config object for that NIC.  You must also set the IP Config object to a **static** address instead of **dynamic** to ensure it does not change on reboot.
- Adding additional NIC's post-deployment is possible, it requires creating an additional NIC, powering off the VM and adding the network interface to the VM. See Microsoft documentation for more details on adding a NIC to an existing VM. **Important**: For failover to occur, each set of additional NIC's created must be named exactly the same except for the last character, such as **additionalnic0** and **additionalnic1**.
- **NEW:**  Beginning with release 5.3.0.0, the BIG-IP image names have changed (previous options were Good, Better, and Best).  Now you choose a BIG-IP VE image based on whether you need [LTM](https://www.f5.com/products/big-ip-services/local-traffic-manager) only (name starts with **LTM**) or All modules (image name starts with **All**) available (including [WAF](https://www.f5.com/products/security/advanced-waf), [AFM](https://www.f5.com/products/security/advanced-firewall-manager), etc.), and if you need 1 or 2 boot locations.  Use 2 boot locations if you expect to upgrade the BIG-IP VE in the future. If you do not need room to upgrade (if you intend to create a new instance when a new version of BIG-IP VE is released), use an image with 1 boot location.  See this [Matrix](https://clouddocs.f5.com/cloud/public/v1/matrix.html#microsoft-azure) for recommended Azure instance types. See the Supported BIG-IP Versions table for the available options for different BIG-IP versions.
- Previous tagged releases can be used to reference functionality that has been changed or removed.
- If you are using a failover via-API template and performing an in-place upgrade of BIG-IP v13.1.1 or later, you must follow the instructions in [Post-Deployment Configuration Steps](#post-deployment-configuration-steps)
- All templates now deploy Standard SKU Azure Public IP Addresses.
- If you choose "No" in response to the provisionPublicIP parameter, the choice for "numberOfExternalIps" is not honored and no public IP addresses are created.
- All templates deploy Azure Virtual Machines and Virtual Machine Scale Sets into Availability Zones in supported regions; Availability Sets are still created in unsupported regions. Virtual Machine Scale Sets are distributed across zones 1, 2, and 3; failover Virtual Machines across zones 1 and 2. Standalone Virtual Machines are placed in zone 1 by default.
- Using dynamic assignment for TMM interface is leveraged for initial deployment convenience and a requirement for some environments but not recommended in most production environments.  IP configs can be converted to static post deployment
- Address translation is done by enabling ip forwarding [Azure documentaion](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-network-interface#enable-or-disable-ip-forwarding)
- This template uses [F5 BIG-IP Runtime Init](https://github.com/F5Networks/f5-bigip-runtime-init) to install F5 Automation Toolchain packages (AS3, DO, CFE, FAST, and TS). You can update the version of one or more packages by editing the template customData variable. For example: To update the AS3 package to the latest version, click on the [Github release page](https://github.com/F5Networks/f5-appsvcs-extension/releases) for the f5-appsvcs-extension. In the template customData variable, in the install_operations section, update the AS3 extensionVersion value to the desired version and the extensionHash value to contents of the RPM sha256 file located in the release assets for that version. You can also add more packages to be installed using the same procedure.

## Security

This ARM template downloads helper code to configure the BIG-IP system. If you want to verify the integrity of the template, you can open the template and ensure the following lines are present. See [Security Detail](#security-details) for the exact code.
In the *variables* section:

- In the *verifyHash* variable: **script-signature** and then a hashed signature.
- In the *installCloudLibs* variable: **tmsh load sys config merge file /config/verifyHash**.
- In the *installCloudLibs* variable: ensure this includes **tmsh run cli script verifyHash /config/cloud/f5-cloud-libs.tar.gz**.

Additionally, F5 provides checksums for all of our supported templates. For instructions and the checksums to compare against, see [checksums-for-f5-supported-cft-and-arm-templates-on-github](https://community.f5.com/t5/crowdsrc/checksums-for-f5-supported-cloud-templates-on-github/ta-p/284471).

## Tested BIG-IP versions

The following table lists the versions of BIG-IP that have been tested and validated against F5 Azure ARM solution templates.

| Azure BIG-IP Image Version | BIG-IP Version | Build | Solution | Status | Notes |
| --- | --- | --- | --- | --- | --- |
| 16.1.303000 | 16.1.3.3 | 0.0.3 | Standalone, Failover, Autoscale | Validated | |
| 15.1.801000 | 15.1.8.1 | 0.0.3 | Standalone, Failover, Autoscale | Validated | |
| 14.1.503000 | 14.1.5.3 | 0.0.5 | Standalone, Failover, Autoscale | Validated | |
| 13.1.500000 | 13.1.5 | 0.0.32 | Standalone, Failover, Autoscale | Not Validated | F5 CFE requires BIG-IP 14.1 or later |
| 12.1.600000 | 12.1.6 | 0.0.0 | Standalone, Failover, Autoscale | Not Validated | F5 CFE requires BIG-IP 14.1 or later |


## Supported instance types and hypervisors

- For a list of supported Azure instance types for this solution, see the [Azure instances for BIG-IP VE](http://clouddocs.f5.com/cloud/public/v1/azure/Azure_singleNIC.html#azure-instances-for-big-ip-ve).

- For a list of versions of the BIG-IP Virtual Edition (VE) and F5 licenses that are supported on specific hypervisors and Microsoft Azure, see [supported-hypervisor-matrix](https://support.f5.com/kb/en-us/products/big-ip_ltm/manuals/product/ve-supported-hypervisor-matrix.html).

## Help

Because this template has been created and fully tested by F5 Networks, it is fully supported by F5. This means you can get assistance if necessary from [F5 Technical Support](https://support.f5.com/csp/article/K40701984).

## Installation

You have three options for deploying this solution:

- Using the Azure deploy buttons
- Using [PowerShell](#powershell-script-example)
- Using [CLI Tools](#azure-cli-10-script-example)

### Azure deploy buttons

Use the appropriate button below to deploy:

- **BYOL** (bring your own license): This allows you to use an existing BIG-IP license.

  [![Deploy to Azure](http://azuredeploy.net/deploybutton.png)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fge-tanaka%2Ff5-azure-arm-templates%2Fmain%2Fsupported%2Ffailover%2Fsame-net%2Fvia-api%2Fn-nic%2Fexisting-stack%2Fbyol%2Fazuredeploy.json)

### Template parameters

| Parameter | Required | Description |
| --- | --- | --- |
| adminUsername | Yes | User name for the Virtual Machine. |
| authenticationType | Yes | Type of authentication to use on the Virtual Machine, password based authentication or key based authentication. |
| adminPasswordOrKey | Yes | Password or SSH public key to login to the Virtual Machine. Note: There are a number of special characters that you should avoid using for F5 product user accounts.  See [K2873](https://support.f5.com/csp/article/K2873) for details. Note: If using key-based authentication, this should be the public key as a string, typically starting with **---- BEGIN SSH2 PUBLIC KEY ----** and ending with **---- END SSH2 PUBLIC KEY ----**. |
| dnsLabel | Yes | Unique DNS Name for the Public IP address used to access the Virtual Machine. |
| instanceName | Yes | Name of the Virtual Machine. |
| numberOfExternalIps | Yes | The number of public/private IP addresses you want to deploy for the application traffic (external) NIC on the BIG-IP VE to be used for virtual servers. |
| instanceType | Yes | Instance size of the Virtual Machine. |
| imageName | Yes | F5 SKU (image) you want to deploy. Note: The disk size of the VM will be determined based on the option you select.  **Important**: If intending to provision multiple modules, ensure the appropriate value is selected, such as ****AllTwoBootLocations or AllOneBootLocation****. |
| bigIpVersion | Yes | F5 BIG-IP version you want to use. |
| bigIpModules | Yes | Comma separated list of modules and levels to provision, for example, ltm:nominal,asm:nominal |
| licenseKey1 | Yes | The license token for the F5 BIG-IP VE (BYOL). |
| licenseKey2 | Yes | The license token for the F5 BIG-IP VE (BYOL). This field is required when deploying two or more devices. |
| vnetName | Yes | The name of the existing virtual network to which you want to connect the BIG-IP VEs. |
| vnetResourceGroupName | Yes | The name of the resource group that contains the Virtual Network where the BIG-IP VE will be placed. |
| mgmtSubnetName | Yes | Name of the existing mgmt subnet - with external access to the Internet. **Important**: The subnet you provide for the mgmt NIC **must** be unique. |
| mgmtIpAddressRangeStart | Yes | The static private IP address you want to assign to the management self IP of the first BIG-IP. The next contiguous address will be used for the second BIG-IP device. Leave the default DYNAMIC to use a dynamically assigned IP address. |
| externalSubnetName | Yes | Name of the existing external subnet - with external access to Internet. **Important**: The subnet you provide for the external NIC **must** be unique. |
| externalIpAddressRangeStart | Yes | The static private IP address (secondary) you would like to assign to the first shared Azure public IP. An additional private IP address will be assigned for each public IP address you specified in numberOfExternalIps.  For example, inputting 10.100.1.50 here and choosing 2 in numberOfExternalIps would result in 10.100.1.50 and 10.100.1.51 being configured as static private IP addresses for external virtual servers. Leave the default DYNAMIC to use a dynamically assigned IP address. |
| externalIpSelfAddressRangeStart | Yes | The static private IP address you want to assign to the external self IP (primary) of the first BIG-IP VE. The next contiguous address will be used for the second BIG-IP device. Leave the default DYNAMIC to use a dynamically assigned IP address. |
| internalSubnetName | Yes | Name of the existing internal subnet. **Important**: The subnet you provide for the internal NIC **must** be unique. |
| internalIpAddressRangeStart | Yes | The static private IP address you would like to assign to the internal self IP of the first BIG-IP VE. The next contiguous address will be used for the second BIG-IP device. Leave the default DYNAMIC to use a dynamically assigned IP address. |
| provisionPublicIP | Yes | Provision Public IP addresses for BIG-IP Management traffic. These addresses will be configured directly on the Azure network interfaces. |
| declarationUrl | Yes | URL for the AS3 (https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/) declaration JSON file to be deployed. Leave as **NOT_SPECIFIED** to deploy without a service configuration. |
| ntpServer | Yes | Leave the default NTP server the BIG-IP uses, or replace the default NTP server with the one you want to use. |
| timeZone | Yes | If you would like to change the time zone the BIG-IP uses, enter the time zone you want to use. This is based on the tz database found in /usr/share/zoneinfo (see the full list [here](https://github.com/F5Networks/f5-azure-arm-templates/blob/master/azure-timezone-list.md)). Example values: UTC, US/Pacific, US/Eastern, Europe/London or Asia/Singapore. |
| customImageUrn | Yes | If you would like to deploy using a specific image in Azure marketplace, specify the URN of the image. **Note**: Unless specifically required, leave the default of **OPTIONAL**. Example value: f5-networks:f5-big-ip-byol:f5-big-ltm-2slot-byol:16.0.101000. You can find the URNs of F5 marketplace images in the README for this template or by running the command: az vm image list --output yaml --publisher f5-networks --all |
| customImage | Yes | If you would like to deploy using a local BIG-IP image, provide either the full URL to the VHD in Azure storage **or** the full resource ID to an existing Microsoft.Compute image resource.  **Note**: Unless specifically required, leave the default of **OPTIONAL**. |
| restrictedSrcAddress | Yes | This field restricts management access to a specific network or address. Enter an IP address or address range in CIDR notation. Please do NOT use 0.0.0.0/0. Instead, restrict the IP address range to your client or trusted network, for example 55.55.55.55/32. Production should never expose the BIG-IP Management interface to the Internet.  |
| tagValues | Yes | Default key/value resource tags will be added to the resources in this deployment, if you would like the values to be unique adjust them as needed for each key. |
| allowUsageAnalytics | Yes | This deployment can send anonymous template statistics to F5 to help us determine how to improve our solutions. If you select **No** statistics are not sent. |
| allowPhoneHome | Yes | This deployment can provide F5 with high-level device use information to optimize development resources. If you select **No** the information is not sent. |
| numberOfAdditionalNics | Yes | By default this solution deploys the BIG-IP(s) in a 3 NIC configuration, however additional NICs can be added to the BIG-IP(s) using this parameter.  **Note**: The default value is 0, additional NICs will only be created if 1 or higher is specified. |
| additionalNicLocation | Yes | This parameter specifies where the additional NICs go, use the default value of **OPTIONAL** if **0** was selected for the parameter **numberOfAdditionalNics**.  Otherwise this value must be a semi-colon delimited string of subnets, equal to the number of additional NICs being deployed.  For example, for 2 additional NICs you would use: **subnet01;subnet02**. **Note**: Ensure there are no spaces and the correct number of subnets are provided based on the value selected in **numberOfAdditionalNics**. **Important**: The subnet you provide for each additional NIC **must** be unique. |
| userAssignedManagedIdentity | Yes | The name of the pre-existing user-assigned managed identity in the resource group where the template will be deployed, if applicable.  This identity will be configured on the virtual machine resources and must have the Contributor role assigned on both the deployment and virtual network resource groups.  Leaving the default of **NOT_SPECIFIED** triggers the creation of a system-assigned managed identity. |
| roleNameGuid | Yes | A new GUID used to identify the role assignment. Keep the field as is. No need to enter any value here as newGuid() is generating a random  |

### Programmatic deployments

As an alternative to deploying through the Azure Portal (GUI) each solution provides example scripts to deploy the ARM template.  The example commands can be found below along with the name of the script file, which exists in the current directory.

#### PowerShell Script Example

```powershell
## Example Command: .\Deploy_via_PS.ps1 -adminUsername azureuser -authenticationType password -adminPasswordOrKey <value> -dnsLabel <value> -instanceName f5vm01 -numberOfExternalIps 1 -instanceType Standard_D8s_v4 -imageName AllTwoBootLocations -bigIpVersion 16.1.303000 -bigIpModules ltm:nominal -licenseKey1 <value> -licenseKey2 <value> -vnetName <value> -vnetResourceGroupName <value> -mgmtSubnetName <value> -mgmtIpAddressRangeStart DYNAMIC -externalSubnetName <value> -externalIpAddressRangeStart DYNAMIC -externalIpSelfAddressRangeStart DYNAMIC -internalSubnetName <value> -internalIpAddressRangeStart DYNAMIC -provisionPublicIP Yes -declarationUrl NOT_SPECIFIED -ntpServer 0.pool.ntp.org -timeZone UTC -customImageUrn OPTIONAL -customImage OPTIONAL -allowUsageAnalytics Yes -allowPhoneHome Yes -numberOfAdditionalNics 0 -additionalNicLocation OPTIONAL -userAssignedManagedIdentity NOT_SPECIFIED -roleNameGuid [newGuid()] -resourceGroupName <value>
```

=======

#### Azure CLI (1.0) Script Example

```bash
## Example Command: ./deploy_via_bash.sh --adminUsername azureuser --authenticationType password --adminPasswordOrKey <value> --dnsLabel <value> --instanceName f5vm01 --numberOfExternalIps 1 --instanceType Standard_D8s_v4 --imageName AllTwoBootLocations --bigIpVersion 16.1.303000 --bigIpModules ltm:nominal --licenseKey1 <value> --licenseKey2 <value> --vnetName <value> --vnetResourceGroupName <value> --mgmtSubnetName <value> --mgmtIpAddressRangeStart DYNAMIC --externalSubnetName <value> --externalIpAddressRangeStart DYNAMIC --externalIpSelfAddressRangeStart DYNAMIC --internalSubnetName <value> --internalIpAddressRangeStart DYNAMIC --provisionPublicIP Yes --declarationUrl NOT_SPECIFIED --ntpServer 0.pool.ntp.org --timeZone UTC --customImageUrn OPTIONAL --customImage OPTIONAL --allowUsageAnalytics Yes --allowPhoneHome Yes --numberOfAdditionalNics 0 --additionalNicLocation OPTIONAL --userAssignedManagedIdentity NOT_SPECIFIED --roleNameGuid [newGuid()] --resourceGroupName <value> --azureLoginUser <value> --azureLoginPassword <value>
```

## Configuration Example

The following is an example configuration diagram for this solution deployment. In this scenario, each BIG-IP has one NIC for management, one NIC for external traffic and one NIC for internal traffic.  This is the traditional BIG-IP deployment model where data-plane, management and internal traffic is separate. The IP addresses in this example may be different in your implementation.

![Configuration Example](../images/azure-example-diagram.png)


## Post-Deployment Configuration

Use this section for optional configuration changes after you have deployed the template.

### Additional Public IP Addresses - Failover

This ARM template supports using up to 20 public IP addresses for the external 'application' NIC to be used for passing traffic to BIG-IP virtual servers. Each virtual server should be configured with a destination address matching the private IP address value of the Azure IP configuration receiving traffic for the application. In the event the active BIG-IP VE becomes unavailable, the IP configuration(s) are migrated using network failover, seamlessly shifting application traffic to the current active BIG-IP VE (the template now supports [multiple traffic groups](#traffic-group-configuration)).  After you initially deployed the template, if you now want to include additional public IP addresses and/or if you want to add or remove the user-defined routes (UDRs) to be managed by the BIG-IP, use the following guidance.

#### Adding additional public IP addresses to the deployment

The deployment template supports creation of 1-20 external public IP addresses for application traffic (first one is used for external NIC Self IP). Use the following guidance to add **additional** public IP addresses to the deployment:

- Create a new Azure public IP address resource in the deployment resource group.
- Create a new, secondary IP configuration resource (for example: *myResourceGroupName-ext-ipconfig9*) in the properties of the external Azure network interface (for example: *myResourceGroupName-ext0*) of the **active** BIG-IP.
- Ensure a **Virtual Address** exists on the BIG-IP in the appropriate traffic group (such as traffic-group-1), and that the Virtual Address matches the private IP address of the secondary IP configuration you created.
  1. Note: A Virtual Address will automatically be created for each Virtual Server created (if it does not already exist).
  2. Note: This is what is matched on during a failover event to determine which NIC IP configurations to move.
  3. Note: F5 recommends creating the IP configuration on the network interface of the BIG-IP VE that is active for the traffic group assigned to the matching Virtual Address.

When you create virtual servers on the BIG-IP VE for these new additional addresses, the BIG-IP virtual server destination IP address should match the Azure Private IP Address of the IP configuration that corresponds to the Public IP address of your application. See the BIG-IP documentation for specific instructions on creating virtual servers.

### In-Place upgrade of BIG-IP v13 instances

When performing an in-place upgrade from BIG-IP software v13.x, you must complete the following steps to ensure that all the components required for failover are copied to the volume where the new version of BIG-IP is installed.  To ensure traffic processing is not interrupted, F5 highly recommends upgrading the standby device, verifying failover functionality, and then upgrading the previously active device. 

From the volume to be upgraded on the standby device, you must edit **cs.dat** to allow inclusion of all files in **/config/cloud** in UCS backup.  

1. Remount the /usr directory as writable:  
  ``mount -o remount,rw /usr``

2. Back up the cs.dat file:  
 ``cp /usr/libdata/configsync/cs.dat /usr/libdata/configsync/cs.dat.bak``

3. Edit the cs.dat file:  
``vi /usr/libdata/configsync/cs.dat``

4. In cs.dat, find the entry similar to the one below (the number between save and ignore may differ):  
**save.10100.ignore = (/config/cloud/*)**

5. Change **ignore** to **file** in the save key, and remove the parentheses from the value:  
``save.10100.file = /config/cloud/*``

6. Save the cs.dat file and exit the editor.

7. Remount the **/usr** directory as read-only:  
``mount -o remount,ro /usr``

8. Create a [UCS archive](https://support.f5.com/csp/article/K13132) in the BIG-IP UI (accept defaults):
**System > Archives > Create > myUCS** 

9. Download myUCS.ucs locally.

10. Install new ISO and reboot into upgraded volume.

11. Boot into the newly upgraded volume.  

12. After verifying failover functionality, repeat steps 1-11 on the now-standby BIG-IP device.

13. Following the upgrade, all the necessary files should be present and failover should work normally.  To manually restore the UCS archive you created previously, use the following steps:
    - From the upgraded volume, upload UCS file: **System > Archives > Upload > myUCS.uss**
    - Restore the previously created UCS archive: **System > Archives > myUCS.ucs > Restore**


### Enabling Route Failover

This solution supports associating specific sets of self IP addresses with Azure Route Tables using the F5 Cloud Failover Extension.  However, because each environment's managed routes will be different, route failover is configured at deployment time, but disabled by default.  To enable route failover, you must POST an updated CFE declaration to the CFE endpoint (/mgmt/shared/cloud-failover/declare) on one of the BIG-IP devices.

The Cloud Failover Extension documentation describes route failover configuration here: https://clouddocs.f5.com/products/extensions/f5-cloud-failover/latest/userguide/configuration.html#failover-routes

Below is an example of a declaration that enables route failover. The "resourceGroupName" value is a placeholder; replace that value with the name of the Azure resource group where the template was deployed. To verify that you are using the correct values, you can send a GET request to the CFE endpoint at /mgmt/shared/cloud-failover/declare:

# Initial declaration response (GET):
```json
{
    "class": "Cloud_Failover",
    "environment": "azure",
    "externalStorage": {
        "scopingTags": {
            "f5_cloud_failover_label": "resourceGroupName"
        }
    },
    "failoverAddresses": {
        "scopingTags": {
            "f5_cloud_failover_label": "resourceGroupName"
        }
    },
    "failoverRoutes": {
        "enabled": false,
        "scopingTags": {
            "f5_cloud_failover_label": "resourceGroupName"
        },
        "scopingAddressRanges": [
            {
                "range": "192.168.0.0/32"
            }
        ],
        "defaultNextHopAddresses": {
            "discoveryType": "static",
            "items": [
                "192.168.0.4",
                "192.168.0.5"
            ]
        }
    }
}
```

Replace the scopingAddressRanges array with a list of range objects that includes one range object per user-defined route that this deployment will manage.  Replace the defaultNextHopAddresses.items array with a list of the BIG-IP self IP addresses configured on the devices in this deployment:

# Example declaration enabling route failover (POST):
```json
{
    "class": "Cloud_Failover",
    "environment": "azure",
    "externalStorage": {
        "scopingTags": {
            "f5_cloud_failover_label": "resourceGroupName"
        }
    },
    "failoverAddresses": {
        "scopingTags": {
            "f5_cloud_failover_label": "resourceGroupName"
        }
    },
    "failoverRoutes": {
        "enabled": true,
        "scopingTags": {
            "f5_cloud_failover_label": "resourceGroupName"
        },
        "scopingAddressRanges": [
            {
                "range": "192.168.3.0/24"
            }
        ],
        "defaultNextHopAddresses": {
            "discoveryType": "static",
            "items": [
                "192.168.2.4",
                "192.168.2.5"
            ]
        }
    }
}
```

Next, use the following examples steps to add the required Azure tags to the route table resource:

  - Azure route table: myRouteTable
    - Add tag with key: "f5_cloud_failover_label", value: "resourceGroupName"

In this case, on failover, the routes in the Azure route table ***myRouteTable*** tagged with ***f5_cloud_failover_label: resourceGroupName*** and including user-defined routes matching the **scopingAddressRanges** array are updated to use either 192.168.2.4 or 192.168.2.5(depending on which BIG-IP VE device is active for traffic-group-1) as the next hop virtual appliance.
## Documentation

For more information on F5 solutions for Azure, including manual configuration procedures for some deployment scenarios, see the Azure section of [Public Cloud Docs](http://clouddocs.f5.com/cloud/public/v1/).



## Creating virtual servers on the BIG-IP VE

In order to pass traffic from your clients to the servers through the BIG-IP system, you must create a virtual server on the BIG-IP VE. To create a BIG-IP virtual server you need to know the private IP address of the secondary IP configuration(s) for each BIG-IP VE network interface created by the template. If you need additional virtual servers for your applications/servers, you can add more secondary IP configurations on the Azure network interface, and corresponding virtual servers on the BIG-IP system. See [virtual-network-multiple-ip-addresses-portal](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-network-multiple-ip-addresses-portal) for information on multiple IP addresses.

In this template, each Azure public IP address is associated with a secondary IP configuration attached to the external network interface of the BIG-IP VE. Because this IP configuration will be moved to the active BIG-IP device, you need to create a single virtual server in **traffic-group-1** that corresponds to the private IP address of that IP configuration.

1. Once your BIG-IP VE has launched, open the BIG-IP VE Configuration utility.
2. On the Main tab, click **Local Traffic > Virtual Servers** and then click the **Create** button.
3. In the **Name** field, give the Virtual Server a unique name.
4. In the **Destination/Mask** field, type the private IP address of the secondary IP configuration associated with the Azure public IP address for your application (for example: 10.0.1.10/32).
5. In the **Service Port** field, type the appropriate port.
6. Configure the rest of the virtual server as appropriate.
7. In the Resources section, from the **Default Pool** list, select the name of the pool you want to use.
8. Click the **Finished** button.
9. Repeat as necessary.

When you have completed the virtual server configuration, you may modify the virtual addresses to use an alternative Traffic Group using the following guidance.

1. On the Main tab, click **Local Traffic > Virtual Servers**.
2. On the Menu bar, click the **Virtual Address List** tab.
3. Click the address of one of the virtual servers you just created.
4. From the **Traffic Group** list, select **traffic-group-2** (or the additional traffic group you created previously).
5. Click **Update**.
6. Repeat for each virtual server.


### Traffic Group Configuration

This template supports associating Azure NIC IP Configurations and Route Table resources with up to two BIG-IP traffic groups, allowing each BIG-IP VE device to process traffic for applications associated with the traffic group for which the device is active.  Use the following guidance to configure multiple traffic groups.  Note you must create the **f5_tg** on any route tables with routes that will be managed by BIG-IP VE. At a minimum, these route tables must be tagged with a default value of **traffic-group-1**.

1. Select the route table you want to associate with the second traffic group (in our example, the second traffic group name is **traffic-group-2**).
2. From the route table Tags blade, select the **f5_tg** tag.
3. Modify the tag value to **traffic-group-2** and then save the tag.
4. From the BIG-IP VE management Configuration utility, click **Device Management > Traffic Groups > Create**, and then complete the following.
    - *Name*: **traffic-group-2**
    - *Failover Order*: Select the preferred order of the devices for this traffic group; F5 recommends setting the current standby device as the preferred device for this second traffic group, so that each traffic group has a different preferred device (device will become active after creating the traffic group).
    - Click **Create Traffic Group**.
5. Ensure the BIG-IP **Virtual Address** corresponding with the created virtual servers is in the desired traffic group.
    - Note: NIC IP Configurations are mapped to a specific traffic group by the Virtual Address configuration of the BIG-IP.
    - Note: You can find the Virtual Address list by browsing to Local Traffic->Virtual Servers->Virtual Address List.


### Deploying Custom Configuration to the BIG-IP (Azure Virtual Machine)

Once the solution has been deployed there may be a need to perform some additional configuration of the BIG-IP.  This can be accomplished via traditional methods such as via the GUI, logging into the CLI or using the REST API.  However, depending on the requirements it might be preferred to perform this custom configuration as a part of the initial deployment of the solution.  This can be accomplished in the below manner.

Within the Azure Resource Manager (ARM) template there is a variable called **customConfig**, this contains text similar to "### START (INPUT) CUSTOM CONFIGURATION", that can be replaced with custom shell scripting to perform additional configuration of the BIG-IP.  An example of what it would look like to configure the f5.ip_forwarding iApp is included below.

Warning: F5 does not support the template if you change anything other than the **customConfig** ARM template variable.

```json
"variables": {
    "customConfig": "### START (INPUT) CUSTOM CONFIGURATION HERE\ntmsh create sys application service my_deployment { device-group none template f5.ip_forwarding traffic-group none variables replace-all-with { basic__addr { value 0.0.0.0 } basic__forward_all { value No } basic__mask { value 0.0.0.0 } basic__port { value 0 } basic__vlan_listening { value default } options__advanced { value no }options__display_help { value hide } } }"
}
```

### Changing the BIG-IP Configuration utility (GUI) port

Depending on the deployment requirements, the default management port for the BIG-IP may need to be changed. To change the Management port, see [Changing the Configuration utility port](https://clouddocs.f5.com/cloud/public/v1/azure/Azure_singleNIC.html#azureconfigport) for instructions.

***Important***: The default port provisioned is dependent on 1) which BIG-IP version you choose to deploy as well as 2) how many interfaces (NICs) are configured on that BIG-IP. BIG-IP v13.x and later in a single-NIC configuration uses port 8443. All prior BIG-IP versions default to 443 on the MGMT interface.

***Important***: If you perform the procedure to change the port, you must check the Azure Network Security Group associated with the interface on the BIG-IP that was deployed and adjust the ports accordingly.

### Service Discovery

This template previously supported configuring service discovery using the f5.service_discovery iApp template.  That iApp has been deprecated and removed from this template.  You can now configure service discovery using the F5 AS3 extension, which is installed by all ARM templates by default.  See the official AS3 [documentation](https://clouddocs.f5.com/products/extensions/f5-appsvcs-extension/latest/userguide/service-discovery.html) and the iApp migration [guide](https://github.com/F5Networks/f5-azure-arm-templates/blob/main/iapp-migration.md) for more information and examples.

### Telemetry Streaming

This template previously supported configuring device telemetry using the f5.cloud_logger iApp template.  That iApp has been deprecated and removed from this template.  You can now configure telemetry streaming using the F5 Telemetry Streaming extension.  See the official TS [documentation](https://clouddocs.f5.com/products/extensions/f5-telemetry-streaming/latest/) and the iApp migration [guide](https://github.com/F5Networks/f5-azure-arm-templates/blob/main/iapp-migration.md) for installation steps and examples.

### Sending statistical information to F5

All of the F5 templates now have an option to send anonymous statistical data to F5 Networks to help us improve future templates.
None of the information we collect is personally identifiable, and only includes:

- Customer ID: this is a hash of the customer ID, not the actual ID
- Deployment ID: hash of stack ID
- F5 template name
- F5 template version
- Cloud Name
- Azure region
- BIG-IP version
- F5 license type
- F5 Cloud libs version

This information is critical to the future improvements of templates, but should you decide to select **No**, information will not be sent to F5.

## Security Details

This section has the code snippet for each the lines you should ensure are present in your template file if you want to verify the integrity of the helper code in the template.

Note the hashed script-signature may be different in your template.

```json
"variables": {
    "apiVersion": "2015-06-15",
    "location": "[resourceGroup().location]",
    "singleQuote": "'",
    "f5CloudLibsTag": "release-2.0.0",
    "verifyHash": "[concat(variables('singleQuote'), 'cli script /Common/verifyHash {\nproc script::run {} {\n        if {[catch {\n            set hashes(f5-cloud-libs.tar.gz) db8cb32226babb7557c05884987fb4542498cfc90b0117fcc5ec9de203caab18f1e12ec09161151696872f12ca342f2fa2259dd9dfd76906621b65345c76b5b2\n            set hashes(f5-cloud-libs-aws.tar.gz) 076c969cbfff12efacce0879820262b7787c98645f1105667cc4927d4acfe2466ed64c777b6d35957f6df7ae266937dde42fef4c8b1f870020a366f7f910ffb5\n            set hashes(f5-cloud-libs-azure.tar.gz) 9037203b1af31288ba6993204a2abf3bd660f62e7dfb2d5825909dd69133ce5b4f5c725afabd7d2acaa693669c878daa04a6375314d985ba07c8a36dccc61c5a\n            set hashes(f5-cloud-libs-gce.tar.gz) 1677835e69967fd9882ead03cbdd24b426627133b8db9e41f6de5a26fef99c2d7b695978ac189f00f61c0737e6dbb638d42dea43a867ef4c01d9507d0ee1fb2f\n            set hashes(f5-cloud-libs-openstack.tar.gz) 5c83fe6a93a6fceb5a2e8437b5ed8cc9faf4c1621bfc9e6a0779f6c2137b45eab8ae0e7ed745c8cf821b9371245ca29749ca0b7e5663949d77496b8728f4b0f9\n            set hashes(f5-cloud-libs-consul.tar.gz) a32aab397073df92cbbba5067e5823e9b5fafca862a258b60b6b40aa0975c3989d1e110f706177b2ffbe4dde65305a260a5856594ce7ad4ef0c47b694ae4a513\n            set hashes(asm-policy-linux.tar.gz) 63b5c2a51ca09c43bd89af3773bbab87c71a6e7f6ad9410b229b4e0a1c483d46f1a9fff39d9944041b02ee9260724027414de592e99f4c2475415323e18a72e0\n            set hashes(f5.http.v1.2.0rc4.tmpl) 47c19a83ebfc7bd1e9e9c35f3424945ef8694aa437eedd17b6a387788d4db1396fefe445199b497064d76967b0d50238154190ca0bd73941298fc257df4dc034\n            set hashes(f5.http.v1.2.0rc6.tmpl) 811b14bffaab5ed0365f0106bb5ce5e4ec22385655ea3ac04de2a39bd9944f51e3714619dae7ca43662c956b5212228858f0592672a2579d4a87769186e2cbfe\n            set hashes(f5.http.v1.2.0rc7.tmpl) 21f413342e9a7a281a0f0e1301e745aa86af21a697d2e6fdc21dd279734936631e92f34bf1c2d2504c201f56ccd75c5c13baa2fe7653213689ec3c9e27dff77d\n            set hashes(f5.aws_advanced_ha.v1.3.0rc1.tmpl) 9e55149c010c1d395abdae3c3d2cb83ec13d31ed39424695e88680cf3ed5a013d626b326711d3d40ef2df46b72d414b4cb8e4f445ea0738dcbd25c4c843ac39d\n            set hashes(f5.aws_advanced_ha.v1.4.0rc1.tmpl) de068455257412a949f1eadccaee8506347e04fd69bfb645001b76f200127668e4a06be2bbb94e10fefc215cfc3665b07945e6d733cbe1a4fa1b88e881590396\n            set hashes(f5.aws_advanced_ha.v1.4.0rc2.tmpl) 6ab0bffc426df7d31913f9a474b1a07860435e366b07d77b32064acfb2952c1f207beaed77013a15e44d80d74f3253e7cf9fbbe12a90ec7128de6facd097d68f\n            set hashes(f5.aws_advanced_ha.v1.4.0rc3.tmpl) 2f2339b4bc3a23c9cfd42aae2a6de39ba0658366f25985de2ea53410a745f0f18eedc491b20f4a8dba8db48970096e2efdca7b8efffa1a83a78e5aadf218b134\n            set hashes(f5.aws_advanced_ha.v1.4.0rc4.tmpl) 2418ac8b1f1884c5c096cbac6a94d4059aaaf05927a6a4508fd1f25b8cc6077498839fbdda8176d2cf2d274a27e6a1dae2a1e3a0a9991bc65fc74fc0d02ce963\n            set hashes(f5.aws_advanced_ha.v1.4.0rc5.tmpl) 5e582187ae1a6323e095d41eddd41151d6bd38eb83c634410d4527a3d0e246a8fc62685ab0849de2ade62b0275f51264d2deaccbc16b773417f847a4a1ea9bc4\n            set hashes(asm-policy.tar.gz) 2d39ec60d006d05d8a1567a1d8aae722419e8b062ad77d6d9a31652971e5e67bc4043d81671ba2a8b12dd229ea46d205144f75374ed4cae58cefa8f9ab6533e6\n            set hashes(deploy_waf.sh) 1a3a3c6274ab08a7dc2cb73aedc8d2b2a23cd9e0eb06a2e1534b3632f250f1d897056f219d5b35d3eed1207026e89989f754840fd92969c515ae4d829214fb74\n            set hashes(f5.policy_creator.tmpl) 06539e08d115efafe55aa507ecb4e443e83bdb1f5825a9514954ef6ca56d240ed00c7b5d67bd8f67b815ee9dd46451984701d058c89dae2434c89715d375a620\n            set hashes(f5.service_discovery.tmpl) 4811a95372d1dbdbb4f62f8bcc48d4bc919fa492cda012c81e3a2fe63d7966cc36ba8677ed049a814a930473234f300d3f8bced2b0db63176d52ac99640ce81b\n            set hashes(f5.cloud_logger.v1.0.0.tmpl) 64a0ed3b5e32a037ba4e71d460385fe8b5e1aecc27dc0e8514b511863952e419a89f4a2a43326abb543bba9bc34376afa114ceda950d2c3bd08dab735ff5ad20\n            set hashes(f5-appsvcs-3.18.0-4.noarch.rpm) ba71c6e1c52d0c7077cdb25a58709b8fb7c37b34418a8338bbf67668339676d208c1a4fef4e5470c152aac84020b4ccb8074ce387de24be339711256c0fa78c8\n\n            set file_path [lindex $tmsh::argv 1]\n            set file_name [file tail $file_path]\n\n            if {![info exists hashes($file_name)]} {\n                tmsh::log err \"No hash found for $file_name\"\n                exit 1\n            }\n\n            set expected_hash $hashes($file_name)\n            set computed_hash [lindex [exec /usr/bin/openssl dgst -r -sha512 $file_path] 0]\n            if { $expected_hash eq $computed_hash } {\n                exit 0\n            }\n            tmsh::log err \"Hash does not match for $file_path\"\n            exit 1\n        }]} {\n            tmsh::log err {Unexpected error in verifyHash}\n            exit 1\n        }\n    }\n    script-signature U6njo2bmHHJ86qS/a1+7+QFIIOV9VJzaRnoVoOsB9KjHkWiWYChAT+QQWtyAFgMSG9il8Ol4pShWlIJVc5ldJMp37K3K+CDYBjeNVai4FQQvekjsurl+L1CFrwd6drMYSxbjSxGWCHcZlkZFrskaSej6xzo+scB4aeD7z3M2om7Iov8nq4x3S9tTgNHFI9XnLGGh0pe+I5CamnB/fvrHYStYcdnI36BLynggB6O8/hYocXfmGKjY5Td9gA+ziq6OQHoxtzjzguyeptHa8WsCB66gxg7TIOLDK9DfoSpQbIKyNs+BNL91Q6RqygfQdUrrAYrtN2RYtKkmK8XRs77Vlg==\n    signing-key /Common/f5-irule\n}', variables('singleQuote'))]",
    "installCloudLibs": "[concat(variables('singleQuote'), '#!/bin/bash\necho about to execute\nchecks=0\nwhile [ $checks -lt 120 ]; do echo checking mcpd\n/usr/bin/tmsh -a show sys mcp-state field-fmt | grep -q running\nif [ $? == 0 ]; then\necho mcpd ready\nbreak\nfi\necho mcpd not ready yet\nlet checks=checks+1\nsleep 1\ndone\necho loading verifyHash script\n/usr/bin/tmsh load sys config merge file /config/verifyHash\nif [ $? != 0 ]; then\necho cannot validate signature of /config/verifyHash\nexit 1\nfi\necho loaded verifyHash\n\nconfig_loc=\"/config/cloud/\"\nhashed_file_list=\"${config_loc}f5-cloud-libs.tar.gz f5-appsvcs-3.31.0-6.noarch.rpm ${config_loc}f5-cloud-libs-azure.tar.gz\"\nfor file in $hashed_file_list; do\necho \"verifying $file\"\n/usr/bin/tmsh run cli script verifyHash $file\nif [ $? != 0 ]; then\necho \"$file is not valid\"\nexit 1\nfi\necho \"verified $file\"\ndone\necho \"expanding $hashed_file_list\"\ntar xfz /config/cloud/f5-cloud-libs.tar.gz --warning=no-unknown-keyword -C /config/cloud/azure/node_modules/@f5devcentral\ntar xfz /config/cloud/f5-cloud-libs-azure.tar.gz --warning=no-unknown-keyword -C /config/cloud/azure/node_modules/@f5devcentral\ntouch /config/cloud/cloudLibsReady', variables('singleQuote'))]",
```

### Updating encrypted files

This template stores certain sensitive information in encrypted file(s) on the BIG-IP.  If that information needs to be updated after initial deployment, follow the below steps.

Note: The following example will update **/config/cloud/.azCredentials**.

- Login to the BIG-IP(s).
- Locate the file to update: **/config/cloud/.azCredentials**
- Decrypt the contents of the file (Note: --symmetric is **required**  if using symmetric encryption):

  ``` bash
  /usr/bin/f5-rest-node /config/cloud/azure/node_modules/\@f5devcentral/f5-cloud-libs/scripts/decryptDataFromFile.js --data-file /config/cloud/.azCredentials --symmetric
  >> output <<
  ```

- Edit contents as required (can save to file, etc.)
- Encrypt updated contents (Note: --data can also be used if updated contents are not in a file):

  ```bash
  f5-rest-node /config/cloud/azure/node_modules/\@f5devcentral/f5-cloud-libs/scripts/encryptDataToFile.js --data-file <file> --out-file /config/cloud/.azCredentials --symmetric
  ```

- Delete any temporary files created (if necessary)

## Filing Issues

If you find an issue, we would love to hear about it.
You have a choice when it comes to filing issues:

- Use the **Issues** link on the GitHub menu bar in this repository for items such as enhancement or feature requests and non-urgent bug fixes. Tell us as much as you can about what you found and how you found it.
- Contact us at [solutionsfeedback@f5.com](mailto:solutionsfeedback@f5.com?subject=GitHub%20Feedback) for general feedback or enhancement requests.
- Use our [Slack channel](https://f5cloudsolutions.herokuapp.com) for discussion and assistance on F5 cloud templates.  There are F5 employees who are members of this community who typically monitor the channel Monday-Friday 9-5 PST and will offer best-effort assistance.
- For templates in the **supported** directory, contact F5 Technical support via your typical method for more time sensitive changes and other issues requiring immediate support.

## Copyright

Copyright 2014-2021 F5 Networks Inc.

## License

### Apache V2.0

Licensed under the Apache License, Version 2.0 (the "License"); you may not use
this file except in compliance with the License. You may obtain a copy of the
License [here](http://www.apache.org/licenses/LICENSE-2.0).

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and limitations
under the License.

### Contributor License Agreement

Individuals or business entities who contribute to this project must have
completed and submitted the F5 Contributor License Agreement.
