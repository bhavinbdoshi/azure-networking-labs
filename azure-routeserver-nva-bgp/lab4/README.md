# Azure Hybrid Networking Routing Lab Series

## Lab 3 - Deploying PaloAlto VM Series Firewall in Hub VNET and BGP peering PAN with Cisco CSR and Azure Route Server in Azure

### Introduction

This lab deploys PaloAlto VM Series Firewall (PAN) in Hub VNET and is BGP peered with Cisco CSR and Azure Route Server in Azure Hub. Goal of this lab is to force 0/0 traffic originating from spoke VNETs via PAN.

North-South traffic (to internet) via PAN

East-West traffic (to on-prem and other VNETs) bypasses PAN

> *This lab is for testing/learning purposes only and should not be considered production configurations*

### Networking Architecture

![lab-3-architecture](assets/lab-3-architecture.png)

### Expected Traffic Flow after lab 3 deployment

![lab-3-traffic-flow](assets/lab-3-traffic-flow.png)

### New Components in lab 3

Azure Hub Environment

- Palo Alto Firewall VM (10.0.4.4) with interfaces in pan-mgt (10.0.5.0/24) subnet and pan-trusted (10.0.4.0/24) subnet
- PAN advertising 0/0 route to 10.0.4.4 to ARS (10.0.2.4 and 10.0.2.5)
- Spoke VNET (spoke2vnet) with address space 10.30.0.0/16
- VM (spoke2-vm) in Spoke VNET (10.30.0.4)

Connectivity

- BGP Peering between PAN (10.0.4.4) and ARS (10.0.2.4 and 10.0.2.5)
- BGP Peering between PAN (10.0.4.4) and CSR (10.0.1.4)

VNET Peerings

- Spoke (spoke2vnet) Peered to Hub (hubvnet) with spoke2vnet using remote-gateways in hubvnet

### Existing components from previous labs

Azure Hub Environment

- hub-vnet(10.0.0.0/16)
- csr-internal (10.0.1.0/24) and csr-external(10.0.0.0/24) subnets in hub-vnet  
- azure-csr Cisco CSR (tunnel ip 192.168.1.1) with public ip (azure-csr-pip) and private ips: external interface (10.0.0.4 from csr-external subnet) and internal interface (10.0.1.4 from csr-internal)
- azure-static-rt UDR on csr-internal and csr-external with only route pointing 0/0 to Internet
- Azure Route Server (routeserver-hub) (10.0.2.4 and 10.0.2.5) in subnet 10.0.0.2/26
- Spoke VNET (spoke1vnet) with address space 10.10.0.0/16
- VM (spoke1-vm) in Spoke VNET (10.10.0.4)

On-premise Environment (simulated on Azure)

- on-prem vnet (10.100.0.0/16)
- csr-internal (10.100.1.0/24) and csr-external(10.100.0.0/24) subnets in on-prem vnet
- onprem-csr Cisco CSR (tunnel ip 192.168.1.3) with public ip (onprem-csr-pip) and private ips: external interface (10.100.0.4 from csr-external subnet) and internal interface (10.100.1.4 from csr-internal)
- test-vm-subnet (10.100.10.0/24) with onprem-test-vm (10.100.10.10)
- onprem-static-rt UDR on csr-internal and csr-external with only route pointing 0/0 to Internet
- onprem-vm-rt UDR on test-vm-subnet

- Connectivity
  - IPSec (IKEV2) VPN tunnel between azure-csr (10.0.0.4) and onprem-csr (10.100.0.4)
  - BGP over IPSec between azure-csr (10.0.0.4) and onprem-csr (10.100.0.4)
  - BGP Peering between ARS (10.0.2.4 & 5) and CSR (10.0.1.4)

- VNET Peerings
  - Spoke (spoke1vnet) Peered to Hub (hubvnet) with spoke1vnet using remote-gateways in hubvnet.

### Deployment Steps

You can use either cloud shell or Azure CLI. While Azure Bastion can be used to access VMs, in this lab Serial Console is used for simplicity.

Set Resource Group Variables. Use same values from previous lab

```bash
locazure="eastus"
rgazure="azure-rg-lab"

loconprem="westus2"
rgonprem="onprem-rg-lab"

```

#### Deploy Spoke-2 VNET (spoke2vnet) and peer to (hubvnet)

```bash

#create Spoke2 VNET
az network vnet create --address-prefixes 10.30.0.0/16 -n spoke2Vnet -g $rgazure --subnet-name vm-subnet --subnet-prefixes 10.30.0.0/24 -o none

#create NSG for Subnet and attach to Subnet
az network nsg create -g $rgazure -n "vm-subnet-spoke2-nsg" -l $locazure  -o none
az network vnet subnet update -g $rgazure -n vm-subnet --vnet-name spoke2Vnet --network-security-group "vm-subnet-spoke2-nsg" -o none

# Peer spoke2vnet to hubvnet
hubid=$(az network vnet show -g $rgazure -n hubvnet --query id -o tsv)
spoke2id=$(az network vnet show -g $rgazure -n spoke2Vnet --query id -o tsv)

az network vnet peering create -n "hubTOspoke2" -g $rgazure --vnet-name hubvnet --remote-vnet $spoke2id --allow-vnet-access --allow-forwarded-traffic --allow-gateway-transit -o none
az network vnet peering create -n "spoke2TOhub" -g $rgazure --vnet-name spoke2Vnet --remote-vnet $hubid --allow-vnet-access --allow-forwarded-traffic --use-remote-gateways -o none

#deploy test VM in spoke 2
az network nic create -g $rgazure --vnet-name spoke2Vnet --subnet vm-subnet -n "spoke2-vm-nic" -o none
az vm create -n spoke2-vm \
    -g $rgazure \
    --image ubuntults \
    --size Standard_D2S_v3 \
    --nics spoke2-vm-nic \
    --authentication-type password \
    --admin-username azureuser \
    --admin-password your password \ here  \
    -o none \
    --only-show-errors

```

##### Connectivity between Spokes?

If you ping from spoke1vm to spoke2vm it will not work.

While spoke2-vm nic has learned routes to on-prem from Azure Route Server in Hub, it doesn't have a route to 10.10.0.0/16.
Also, in spoke2-vm-nic 10/8 route points to None as Next Hope Type.

![spoke2-vm-nic-no-10/8-route](assets/spoke2-vm-nic-10-8-none-route.png)

Let's revisit this after deploying PAN

##### Deploy VM Series PaloAlto Firewall in hubvnet

You may have to accept terms if you are deploying this image for first time.

```bash

az vm image terms accept --urn paloaltonetworks:vmseries-flex:byol:latest

```

Create VM Series PA Firewall

```azurecli

# Create NSG for PAN MGMT SUBNET
az network nsg create -g $rgazure -n "subnet-pan-mgmt-nsg" -l $locazure   -o none

az network nsg rule create --resource-group $rgazure --nsg-name subnet-pan-mgmt-nsg --name "Allow-10slash" --access Allow --protocol "*" --direction Inbound --priority 100 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create --resource-group $rgazure --nsg-name subnet-pan-mgmt-nsg --name "Allow-192slash" --access Allow --protocol "*" --direction Inbound --priority 120 --source-address-prefix 192.168.0.0/16 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create --resource-group $rgazure --nsg-name subnet-pan-mgmt-nsg --name "Allow-HTTPS" --access Allow --protocol "TCP" --direction Inbound --priority 300 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range "443" -o none

# Create NSG for PAN TRUSTED SUBNET
az network nsg create -g $rgazure -n "subnet-pan-trusted-nsg" -l $locazure  -o none

az network nsg rule create --resource-group $rgazure --nsg-name subnet-pan-trusted-nsg --name "Allow-10slash" --access Allow --protocol "*" --direction Inbound --priority 100 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create --resource-group $rgazure --nsg-name subnet-pan-trusted-nsg --name "Allow-10slash" --access Allow --protocol "*" --direction Inbound --priority 200 --source-address-prefix 192.168.0.0/16 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none

# Create Subnets
az network vnet subnet create --address-prefix 10.0.4.0/24 --name pan-trusted --resource-group $rgazure --vnet-name hubvnet --network-security-group subnet-pan-trusted-nsg -o none
az network vnet subnet create --address-prefix 10.0.5.0/24 --name pan-mgmt --resource-group $rgazure --vnet-name hubvnet --network-security-group subnet-pan-mgmt-nsg -o none

# Create PAN Firewall

# Create PAN NICs
az network public-ip create --name pan-mgmt-pip --resource-group $rgazure --idle-timeout 30 --sku Standard -o none
az network nic create --name pan-mgmt-nic --resource-group $rgazure --subnet pan-mgmt --vnet-name hubvnet --public-ip-address pan-mgmt-pip --private-ip-address 10.0.5.4 --ip-forwarding true -o none
az network nic create --name pan-trust-nic --resource-group $rgazure --subnet pan-trusted --vnet-name hubvnet --private-ip-address 10.0.4.4 --ip-forwarding true -o none

# Create RT for trust-nic and mgmt-nic and apply to subnet
az network route-table create --name pan-mgmt-vm-rt --resource-group $rgazure --disable-bgp-route-propagation true -o none
az network route-table route create --name default --resource-group $rgazure --route-table-name pan-mgmt-vm-rt --address-prefix "0.0.0.0/0" --next-hop-type Internet -o none
az network vnet subnet update --name pan-mgmt --vnet-name hubvnet --resource-group $rgazure --route-table pan-mgmt-vm-rt -o none

az network route-table create --name pan-trusted-vm-rt --resource-group $rgazure -o none
az network route-table route create --name default --resource-group $rgazure --route-table-name pan-trusted-vm-rt --address-prefix "0.0.0.0/0" --next-hop-type Internet -o none
az network vnet subnet update --name pan-trusted --vnet-name hubvnet --resource-group $rgazure --route-table pan-trusted-vm-rt -o none

# Create PAN Series VM
az vm create --resource-group $rgazure \
 --location $locazure \
 --name pan-vmseries-fw \
 --size Standard_D2S_v3 \
 --nics pan-mgmt-nic  pan-trust-nic \
 --image paloaltonetworks:vmseries-flex:byol:latest \
 --admin-username azureuser \
 --admin-password "M@ft123M@ft123" \ 
 -o none \
 --only-show-errors

```

##### Configure Base Settings for VM Series PaloAlto Firewall

Before setting up BGP with ARS or CSR, we will configure basic instance of firewall.

For this lab you can upload base-fw-setup.xml to PAN Web Management UI interface.

It sets up Network (Interfaces, Zones, Virtual Routers, Interface Mgmt) and policies (Security and NAT). Ip Address for Trusted PAN NIC is 10.0.4.4

Login to <https://public-ip-of-vm-series-fw>

> You will need accept certificate

Go to Device -> Setup -> Operations

Click on Import Named configuration snapshot

Click on Commit on right corner.

Validate that PAN is configured.

> When you use this xml, password to login to PAN Web Management UI is "M@ft123M@ft123"

#### Configure BGP via PAN Web Management UI or Import Configuration

Refer to this document ![]doc-link which illustrates steps on following:

- Configure Static Routes to Azure Route Server (ARS) and Cisco (CSR) in Hubvnet
- Configure route 0/0 to PA interface and redistribute to BGP Peers
- BGP Peer with ARS (10.0.2.4 & 10.0.2.5) (routeserver-hub ASN 65515)
- BGP Peer with CSR (azure-csr 10.0.1.4 with ASN 65001)

Or you can import this file lab3-bgp-fw-setup.xml which will configure same steps as above.

#### BGP Peer PAN with ARS in Hub

```bash

#peer from ARS to PAN
az network routeserver peering create --name hub-ars-to-pan --peer-ip 10.0.4.4 --peer-asn 65010 --routeserver routeserver-hub --resource-group $rgazure -o none

```

### BGP Peer azure-csr (CSR) with PAN

Login to azure-csr via Serial Console. Navigate to `en` and then `conf t`
Paste in below configuration, one block at a time in config mode i.e `azure-csr(config)#`

```bash

#static route for PAN 10.0.4.4
ip route 10.0.4.4 255.255.255.255 10.0.1.1

router bgp 65001
 bgp log-neighbor-changes
 neighbor 10.0.4.4 remote-as 65010
 neighbor 10.0.4.4 ebgp-multihop 255
exit

exit

```

After peering with PAN, you would see this neighbor message up

```bash
*Oct 13 00:09:20.015: %BGP-5-ADJCHANGE: neighbor 10.0.4.4 Up 

```
