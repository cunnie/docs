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
```
