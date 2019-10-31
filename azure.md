```
azure login
azure config mode arm
azure account list --json
[
  {
    "id": "a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9",
    "name": "Free Trial",
    "user": {
      "name": "brian.cunnie@gmail.com",
      "type": "user"
    },
    "tenantId": "682bd378-95db-41bd-8b1e-70fb407c4b10",
    "state": "Enabled",
    "isDefault": true,
    "registeredProviders": [],
    "environmentName": "AzureCloud"
  }
]
azure account set a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9
azure ad app create --name "BOSH CPI" --password put-password-here --identifier-uris "http://BOSHAzureCPI" --home-page "http://BOSHAzureCPI"
  info:    Executing command ad app create
  + Creating application BOSH CPI
  data:    AppId:                   bf7f78c1-6924-4a02-965c-b66f481a9b5f
  data:    ObjectId:                5f93a87a-00dc-405c-a0a4-efc4a677560f
  data:    DisplayName:             BOSH CPI
  data:    IdentifierUris:          0=http://BOSHAzureCPI
  data:    ReplyUrls:
  data:    AvailableToOtherTenants: False
  data:    HomePage:                http://BOSHAzureCPI
  info:    ad app create command OK
  # FIXME: the following command differs from the instructions https://bosh.io/docs/azure-resources.html
azure ad sp create service_principal --applicationId bf7f78c1-6924-4a02-965c-b66f481a9b5f
  + Creating service principal for application bf7f78c1-6924-4a02-965c-b66f481a9b5f
  data:    Object Id:               cc00e15e-ffbc-4581-85ac-9b13d3470933
  data:    Display Name:            BOSH CPI
  data:    Service Principal Names:
  data:                             bf7f78c1-6924-4a02-965c-b66f481a9b5f
  data:                             http://BOSHAzureCPI
  info:    ad sp create command OK
azure role assignment create --roleName "Contributor" --spn "http://BOSHAzureCPI" --subscription a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9
  info:    Executing command role assignment create
  + Finding role with specified name
  /data:    RoleAssignmentId     : /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/providers/Microsoft.Authorization/roleAssignments/316cdb49-8d95-4e04-8b54-3db89db7da7f
  data:    RoleDefinitionName   : Contributor
  data:    RoleDefinitionId     : b24988ac-6180-42a0-ab88-20f7382dd24c
  data:    Scope                : /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9
  data:    Display Name         : BOSH CPI
  data:    SignInName           : undefined
  data:    ObjectId             : cc00e15e-ffbc-4581-85ac-9b13d3470933
  data:    ObjectType           : ServicePrincipal
  data:
  +
  info:    role assignment create command OK
azure group create --name bosh-res-group --location "Southeast Asia"
  info:    Executing command group create
  + Getting resource group bosh-res-group
  + Creating resource group bosh-res-group
  info:    Created resource group bosh-res-group
  data:    Id:                  /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/resourceGroups/bosh-res-group
  data:    Name:                bosh-res-group
  data:    Location:            southeastasia
  data:    Provisioning State:  Succeeded
  data:    Tags: null
  data:
  info:    group create command OK
azure group show --name bosh-res-group
azure network vnet create --name boshnet --address-prefixes 10.0.0.0/8 --resource-group bosh-res-group --location "Southeast Asia"
  info:    Executing command network vnet create
  + Looking up the virtual network "boshnet"
  + Creating virtual network "boshnet"
  error:   The subscription is not registered to use namespace 'Microsoft.Network'. See https://aka.ms/rps-not-found for how to register subscriptions.
  error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
  error:   network vnet create command failed
azure provider list
azure provider register Microsoft.Network
  + Registering provider Microsoft.Network with subscription a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9
  error:   Namespace Microsoft.Network Registration took too long to complete
  error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
  error:   provider register command failed
azure network vnet create --name boshnet --address-prefixes 10.0.0.0/8 --resource-group bosh-res-group --location "Southeast Asia"
  + Looking up the virtual network "boshnet"
  + Creating virtual network "boshnet"
  data:    Id                              : /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/resourceGroups/bosh-res-group/providers/Microsoft.Network/virtualNetworks/boshnet
  data:    Name                            : boshnet
  data:    Type                            : Microsoft.Network/virtualNetworks
  data:    Location                        : southeastasia
  data:    Provisioning state              : Succeeded
  data:    Address prefixes:
  data:      10.0.0.0/8
  info:    network vnet create command OK
azure network vnet subnet create --name bosh --address-prefix 10.0.0.0/24 --vnet-name boshnet --resource-group bosh-res-group
  + Looking up the virtual network "boshnet"
  + Looking up the subnet "bosh"
  + Creating subnet "bosh"
  data:    Id                              : /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/resourceGroups/bosh-res-group/providers/Microsoft.Network/virtualNetworks/boshnet/subnets/bosh
  data:    Name                            : bosh
  data:    Provisioning state              : Succeeded
  data:    Address prefix                  : 10.0.0.0/24
  info:    network vnet subnet create command OK

azure network nsg create --resource-group bosh-res-group --location "Southeast Asia" --name nsg-bosh

azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol Tcp --direction Inbound --priority 200 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'ssh' --destination-port-range 22
azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol Tcp --direction Inbound --priority 201 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'bosh-agent' --destination-port-range 6868
azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol Tcp --direction Inbound --priority 202 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'bosh-director' --destination-port-range 25555
azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol '*' --direction Inbound --priority 203 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'dns' --destination-port-range 53
azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol Tcp --direction Inbound --priority 204 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'http' --destination-port-range 80
azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol Tcp --direction Inbound --priority 205 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'https' --destination-port-range 443
azure network nsg rule create --resource-group bosh-res-group --nsg-name nsg-bosh --access Allow --protocol '*' --direction Inbound --priority 206 --source-address-prefix Internet --source-port-range '*' --destination-address-prefix '*' --name 'ntp' --destination-port-range 123

azure storage account create boshstore --resource-group bosh-res-group --location "Central US"
	info:    Executing command storage account create
	+ Checking availability of the storage account name
	error:   The storage account named "boshstore" is already taken
	error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
	error:   storage account create command failed

azure storage account create cunniestore --resource-group bosh-res-group --location "Southeast Asia"
	info:    Executing command storage account create
	+ Checking availability of the storage account name
	help:    SKU Name:
		1) LRS
		2) ZRS
		3) GRS
		4) RAGRS
		5) PLRS
	: 1
	help:    Kind:
		1) Storage
		2) BlobStorage
	: 1
	+ Creating storage account
	error:   The subscription is not registered to use namespace 'Microsoft.Storage'. See https://aka.ms/rps-not-found for how to register subscriptions.
	error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
	error:   storage account create command failed
	# "LRS is less expensive than GRS, and also offers higher throughput. If your application stores data that can be easily reconstructed, you may opt for LRS."

azure provider register Microsoft.Storage
	+ Registering provider Microsoft.Storage with subscription a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9
	info:    provider register command OK

azure storage account create cunniestore --resource-group bosh-res-group --location "Southeast Asia"
info:    Executing command storage account create
+ Checking availability of the storage account name
help:    SKU Name:
  1) LRS
  2) ZRS
  3) GRS
  4) RAGRS
  5) PLRS
: 1
help:    Kind:
  1) Storage
  2) BlobStorage
: 1
+ Creating storage account
info:    storage account create command OK

azure storage account keys list cunniestore --resource-group bosh-res-group
+ Getting storage account keys
data:    Name  Key                                                                                       Permissions
data:    ----  ----------------------------------------------------------------------------------------  -----------
data:    key1  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxnC1OrA==  Full
data:    key2  xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxMLSO8Q==  Full
info:    storage account keys list command OK

azure storage container create --container bosh     --account-name cunniestore --account-key xxx
	info:    Executing command storage container create
	+ Creating storage container bosh
	+ Getting storage container information
	data:    {
	data:        name: 'bosh',
	data:        metadata: {},
	data:        etag: '"0x8D3F46AE50CCF02"',
	data:        lastModified: 'Fri, 14 Oct 2016 19:47:13 GMT',
	data:        lease: { status: 'unlocked', state: 'available' },
	data:        requestId: '050cf707-0001-001c-7653-26bdb2000000',
	data:        publicAccessLevel: 'Off'
	data:    }
	info:    storage container create command OK
azure storage container create --container stemcell --account-name cunniestore --account-key xxx
	info:    Executing command storage container create
	+ Creating storage container stemcell
	+ Getting storage container information
	data:    {
	data:        name: 'stemcell',
	data:        metadata: {},
	data:        etag: '"0x8D3F46B3407C66A"',
	data:        lastModified: 'Fri, 14 Oct 2016 19:49:26 GMT',
	data:        lease: { status: 'unlocked', state: 'available' },
	data:        requestId: 'a4a257bf-0001-0045-1f54-26b834000000',
	data:        publicAccessLevel: 'Off'
	data:    }
	info:    storage container create command OK
azure network public-ip create --name ns-azure --allocation-method Static --resource-group bosh-res-group --location "Southeast Asia"
	info:    Executing command network public-ip create
	warn:    Using default --idle-timeout 4
	warn:    Using default --ip-version IPv4
	+ Looking up the public ip "ns-azure"
	+ Creating public ip address "ns-azure"
	data:    Id                              : /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/resourceGroups/bosh-res-group/providers/Microsoft.Network/publicIPAddresses/ns-azure
	data:    Name                            : ns-azure
	data:    Type                            : Microsoft.Network/publicIPAddresses
	data:    Location                        : southeastasia
	data:    Provisioning state              : Succeeded
	data:    Allocation method               : Static
	data:    IP version                      : IPv4
	data:    Idle timeout in minutes         : 4
	data:    IP Address                      : 52.187.42.158
```

The ssh-key is a bit of a hack &mdash; one needs to copy the public portion into the
manifest and the private portion into the manifest directory:

```bash
cp ~/.ssh/google bosh
chmod 400 bosh
echo bosh >> .gitignore
```

```bash
azure provider register Microsoft.Compute
  + Registering provider Microsoft.Compute with subscription a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9
  + error:   Namespace Microsoft.Compute Registration took too long to complete
  + error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
  + error:   provider register command failed
```

Check quotas:

```
azure quotas show "Southeast Asia"
```

Get IPv6 address

```
azure network public-ip create --name ns-azure --allocation-method Static --resource-group bosh-res-group --location "Southeast Asia" --ip-version IPv6
  info:    Executing command network public-ip create
  warn:    Using default --idle-timeout 4
  + Looking up the public ip "ns-azure"
  error:   A public ip address with name "ns-azure" already exists in the resource group "bosh-res-group"
  error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
  error:   network public-ip create command failed
azure network public-ip create --name ns-azure-6 --allocation-method Static --resource-group bosh-res-group --location "Southeast Asia" --ip-version IPv6
  info:    Executing command network public-ip create
  warn:    Using default --idle-timeout 4
  + Looking up the public ip "ns-azure-6"
  + Creating public ip address "ns-azure-6"
  error:   Cannot specify publicIpAllocationMethod as Static for IPv6 PublicIp '/subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/resourceGroups/bosh-res-group/providers/Microsoft.Network/publicIPAddresses/ns-azure-6'.
  error:   Error information has been recorded to /Users/cunnie/.azure/azure.err
  error:   network public-ip create command failed
azure network public-ip create --name ns-azure-6 --resource-group bosh-res-group --location "Southeast Asia" --ip-version IPv6
info:    Executing command network public-ip create
warn:    Using default --idle-timeout 4
warn:    Using default --allocation-method Dynamic
+ Looking up the public ip "ns-azure-6"
+ Creating public ip address "ns-azure-6"
data:    Id                              : /subscriptions/a1ac8d5a-7a97-4ed5-bfd1-d7822e19cae9/resourceGroups/bosh-res-group/providers/Microsoft.Network/publicIPAddresses/ns-azure-6
  data:    Name                            : ns-azure-6
  data:    Type                            : Microsoft.Network/publicIPAddresses
  data:    Location                        : southeastasia
  data:    Provisioning state              : Succeeded
  data:    Allocation method               : Dynamic
  data:    IP version                      : IPv6
  data:    Idle timeout in minutes         : 4
  info:    network public-ip create command OK
```

## Troubleshooting

```
Task 2718 | 21:15:39 | Error: Unknown CPI error 'Bosh::AzureCloud::AzureError' with message 'get_token - http code: 401. Azure authentication failed: Invalid tenant_id, client_id or client_secret/certificate. Error message: {"error":"invalid_client","error_description":"AADSTS7000222: The provided client secret keys are expired.\r\nTrace ID: 2aed5fd1-afb8-4ad5-b491-db79f5d30500\r\nCorrelation ID: b3b2d9ae-218f-4255-b444-7720c671f1f3\r\nTimestamp: 2019-10-30 21:15:39Z","error_codes":[7000222],"timestamp":"2019-10-30 21:15:39Z","trace_id":"2aed5fd1-afb8-4ad5-b491-db79f5d30500","correlation_id":"b3b2d9ae-218f-4255-b444-7720c671f1f3"}' in 'info' CPI method (CPI request ID: 'cpi-437503')
```

Inspiration:
<https://www.mikesaysmeh.com/fix-the-provided-client-secret-keys-are-expired-in-azure-lets-encrypt/>

- Log to Azure Portal
- Click **More services**
- Click **Azure Active Directory**
- Click **App Registrations**
- Click **A certificate or secret has expired. Create a new one**
- Click **New client secret**
  - Description: **BOSH CPI**
  - Expiration: **Never**
  - Click **Add**

Cut-and-paste the secret into your bosh manifest. `/instance_groups?name=bosh/properties/azure/client_secret`

You'll probably need to put it somewhere under `/cloud_provider/` if your BOSH
is not multi-CPI and residing on Azure.
