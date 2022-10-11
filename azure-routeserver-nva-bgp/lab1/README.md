# Azure Hybrid Networking Routing Lab Series

## Lab 1 - Deploying Hub VNET in Azure and Simulated On-Prem then setting up Cisco CSR Routers with BGP over IPSEC

### Intro

This lab deploys 2 CSR Routers one in Azure Hub VNET and one in On-premise (Azure simulated) and IPSec tunnel is established between CSR routers. BGP is established between CSRs. An on-prem test VM is setup to ping CSR on Azure.
> *This lab is for testing/learning purposes only and should not be considered production configurations*

### Networking Architecture

![lab-1-architecture](assets/lab-1-azure-hub-csr.png)

### Components in this lab

- Azure Hub Environment
  - hub-vnet(10.0.0.0/16)
  - csr-internal (10.0.1.0/24) and csr-external(10.0.0.0/24) subnets in hub-vnet  
  - azure-csr Cisco CSR (tunnel ip 192.168.1.1) with public ip (azure-csr-pip) and private ips: external interface (10.0.0.4 from csr-external subnet) and internal interface (10.0.1.4 from csr-internal)
  - azure-static-rt UDR on csr-internal and csr-external with only route pointing 0/0 to Internet

- On-premise Environment (simulated on Azure)
  - on-prem vnet (10.100.0.0/16)
  - csr-internal (10.100.1.0/24) and csr-external(10.100.0.0/24) subnets in on-prem vnet
  - onprem-csr Cisco CSR (tunnel ip 192.168.1.3) with public ip (onprem-csr-pip) and private ips: external interface (10.100.0.4 from csr-external subnet) and internal interface (10.100.1.4 from csr-internal)
  - test-vm-subnet (10.100.10.0/24) with onprem-test-vm (10.100.10.10)
  - onprem-static-rt UDR on csr-internal and csr-external with only route pointing 0/0 to Internet
  - onprem-vm-rt UDR on test-vm-subnet

- Connectivity
  - IPSec (IKEV2) tunnel between azure-csr and onprem-csr
  - BGP over IPSec between azure-csr and onprem-csr

### Deployment Steps

You can use either cloud shell or Azure CLI. While Azure Bastion can be used to access VMs, in this lab Serial Console is used for simplicity.

Create Resource Groups

```bash
locazure="eastus"
rgazure="azure-rg-lab"

loconprem="westus2"
rgonprem="onprem-rg-lab"

az group create -n $rgazure -l $locazure -o none
az group create -n $rgonprem -l $loconprem -o none
```

You may have to accept Cisco CSR Agreement

```azurecli
az vm image terms accept --urn cisco:cisco-csr-1000v:16_12_5-byol:latest
```
#### On-Prem environment (simulated on another Azure region)

Create On-prem Simulated environment VNET and Subnets

```azurecli
#Create On-prem VNET and Subnets
az network vnet create --name onprem-vnet --resource-group $rgonprem --address-prefix 10.100.0.0/16 -o none

#Create NSGs for on-prem CSR
az network vnet subnet create -g $rgonprem --vnet-name onprem-vnet -n csr-internal --address-prefix 10.100.1.0/24 -o none
az network vnet subnet create -g $rgonprem --vnet-name onprem-vnet -n csr-external --address-prefix 10.100.0.0/24 -o none
az network vnet subnet create -g $rgonprem --vnet-name onprem-vnet -n test-vm-subnet --address-prefix 10.100.10.0/24 -o none

```

Create On-prem CSR with public and private NICs, NSGs(allow ipsec UDP Ports and 10/8 traffic inbound)

`Change you password before creating VM`

```azurecli
# Create NSGs for on-prem CSR
az network nsg create -g $rgonprem -l $loconprem -n onprem-csr-nsg -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name csr-ipsec1 --access Allow --protocol Udp --direction Inbound --priority 100 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range 500 -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name csr-ipsec2 --access Allow --protocol Udp --direction Inbound --priority 110 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range 4500 -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name allow-10slash --access Allow --protocol "*" --direction Inbound --priority 130 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name allow-192slash --access Allow --protocol "*" --direction Inbound --priority 140 --source-address-prefix 192.168.0.0/16 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name allow-Outbound --access Allow --protocol "*" --direction Outbound --priority 120 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none

az network vnet subnet update -g $rgonprem -n csr-external --vnet-name onprem-vnet --network-security-group "onprem-csr-nsg" -o none
az network vnet subnet update -g $rgonprem -n csr-internal --vnet-name onprem-vnet --network-security-group "onprem-csr-nsg" -o none

# Create on-prem CSR public and private NIC
az network public-ip create -g $rgonprem --name onprem-csr-pip --idle-timeout 30 --allocation-method Static --sku standard -o none
az network nic create -g $rgonprem --name csr-ext-nic --subnet csr-external --vnet onprem-vnet --public-ip-address onprem-csr-pip --private-ip-address 10.100.0.4 --ip-forwarding true --network-security-group onprem-csr-nsg -o none
az network nic create -g $rgonprem --name csr-int-nic --subnet csr-internal --vnet onprem-vnet --ip-forwarding true --private-ip-address 10.100.1.4 --network-security-group onprem-csr-nsg -o none

#Create CSR VM
az vm create \
 -g $rgonprem \
 -l $loconprem \
 --name onprem-csr-vm \
 --size Standard_D2S_v3 \
 --nics csr-ext-nic csr-int-nic  \
 --image cisco:cisco-csr-1000v:16_12_5-byol:latest  \
 --authentication-type password \
 --admin-username azureuser \
 --admin-password "put your \ password here" \
 -o none \
 --only-show-errors
```

Create on-prem Test VM with NSG, NIC 

```azurecli
#create NSG for on-prem CSR subnet for NICs
az network nsg create -g $rgonprem -l $loconprem --name onprem-vm-nsg -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-vm-nsg --name allow-10slash --access Allow --protocol "*" --direction Inbound --priority 130 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none

az network nic create -g $rgonprem -l $loconprem -n testvm-nic --subnet test-vm-subnet --private-ip-address 10.100.10.10 --vnet-name onprem-vnet --network-security-group onprem-vm-nsg  -o none

#create test vm
az vm create --name onprem-test-vm \
 -g $rgonprem \
 -l $loconprem \
 --image ubuntults \
 --size Standard_D2S_v3 \
 --admin-username azureuser \
 --authentication-type password \
 --admin-password "put your \ password here" \
 --nics testvm-nic \
 -o none \
 --only-show-errors
```

#### Azure Hub Environment

Create Azure Hub environment VNET and Subnets

```azurecli
#Create HUB VNET 
az network vnet create --name hubvnet --resource-group $rgazure --address-prefix 10.0.0.0/16 -o none

#Create CSR External and Internal CSR Subnet
az network vnet subnet create --address-prefix 10.0.1.0/24 --name csr-internal --resource-group $rgazure --vnet-name hubvnet  -o none
az network vnet subnet create --address-prefix 10.0.0.0/24 --name csr-external --resource-group $rgazure --vnet-name hubvnet -o none
```

Create Azure CSR with public and private NICs, NSGs(allow ipsec UDP Ports and 10/8 traffic inbound)

```azurecli
#Create NSG for CSR in Hub
az network nsg create --resource-group $rgazure --name azure-csr-nsg --location $locazure -o none
az network nsg rule create --resource-group $rgazure --nsg-name azure-csr-nsg --name csr-ipsec1 --access Allow --protocol Udp --direction Inbound --priority 100 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range 500 -o none
az network nsg rule create --resource-group $rgazure --nsg-name azure-csr-nsg --name csr-ipsec2 --access Allow --protocol Udp --direction Inbound --priority 110 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range 4500 -o none
az network nsg rule create --resource-group $rgazure --nsg-name azure-csr-nsg --name allow-10slash --access Allow --protocol "*" --direction Inbound --priority 130 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create --resource-group $rgazure --nsg-name azure-csr-nsg --name allow-192slash --access Allow --protocol "*" --direction Inbound --priority 140 --source-address-prefix 192.168.0.0/16 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create --resource-group $rgazure --nsg-name azure-csr-nsg --name allow-out --access Allow --protocol "*" --direction Outbound --priority 140 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none

az network vnet subnet update -g $rgazure -n csr-external --vnet-name hubvnet --network-security-group "azure-csr-nsg" -o none
az network vnet subnet update -g $rgazure -n csr-internal --vnet-name hubvnet --network-security-group "azure-csr-nsg" -o none

#create NICs for Azure CSR
az network public-ip create --name azure-csr-pip --resource-group $rgazure --idle-timeout 30 --allocation-method Static --sku standard -o none
az network nic create --name csr-ext-nic -g $rgazure --subnet csr-external --vnet hubvnet --public-ip-address azure-csr-pip --private-ip-address 10.0.0.4 --ip-forwarding true --network-security-group azure-csr-nsg -o none
az network nic create --name csr-int-nic -g $rgazure --subnet csr-internal --vnet hubvnet --ip-forwarding true --private-ip-address 10.0.1.4 --network-security-group azure-csr-nsg -o none

#Create Azure CSR VM
az vm create \
	--resource-group $rgazure \
	--location $locazure \
	--name azure-csr \
	--size Standard_D2S_v3 \
	--nics csr-ext-nic csr-int-nic  \
	--image cisco:cisco-csr-1000v:16_12_5-byol:latest \
	--admin-username azureuser \
	--admin-password "put your \ passsword" \
	-o none \
	--only-show-errors
```

Add UDR to CSR interfaces (0/0 to Internet) to  Azure and On-prem CSRs to avoid routing loop

```azurecli
#Add UDR to CSR interfaces  (0/0 to Internet) for On-orem CSR
az network route-table create --name onprem-static-rt --resource-group $rgonprem
az network route-table route create --name route-to-internet --resource-group $rgonprem --route-table-name onprem-static-rt --address-prefix "0.0.0.0/0" --next-hop-type Internet
az network vnet subnet update --name csr-external --vnet-name onprem-vnet --resource-group $rgonprem --route-table onprem-static-rt
az network vnet subnet update --name csr-internal --vnet-name onprem-vnet --resource-group $rgonprem --route-table onprem-static-rt

#Add UDR to CSR interfaces  (0/0 to Internet) for Azure CSR
az network route-table create --name azure-static-rt --resource-group $rgazure
az network route-table route create --name route-to-internet --resource-group $rgazure --route-table-name azure-static-rt --address-prefix "0.0.0.0/0" --next-hop-type Internet
az network vnet subnet update --name csr-external --vnet-name hubvnet --resource-group $rgazure --route-table azure-static-rt
az network vnet subnet update --name csr-internal --vnet-name hubvnet --resource-group $rgazure --route-table azure-static-rt
```

Get public IPs for Azure CSR and On-Prem CSR

```azurecli
az network public-ip show -g $rgazure -n azure-csr-pip --query "{address: ipAddress}"
az network public-ip show -g $rgonprem -n onprem-csr-pip --query "{address: ipAddress}"
```

Enable Serial Console
> Go to Portal -> Navigate to VM -> Help -> Serial Console. Configure boot diagnostics to enable Serial Console.

