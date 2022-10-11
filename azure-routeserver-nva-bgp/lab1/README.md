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

Create On-prem Simulated environment VNET and Subnets

```azurecli
az network vnet create --name onprem-vnet --resource-group $rgonprem --address-prefix 10.100.0.0/16 -o none

az network vnet subnet create -g $rgonprem --vnet-name onprem-vnet -n csr-internal --address-prefix 10.100.1.0/24 -o none
az network vnet subnet create -g $rgonprem --vnet-name onprem-vnet -n csr-external --address-prefix 10.100.0.0/24 -o none
az network vnet subnet create -g $rgonprem --vnet-name onprem-vnet -n test-vm-subnet --address-prefix 10.100.10.0/24 -o none

```

Create on-prem On-prem CSR with public and private NICs, NSGs(allow ipsec UDP Ports and 10/8 traffic inbound)

`Change you password before creating VM`

```azurecli
az network nsg create -g $rgonprem -l $loconprem -n onprem-csr-nsg -o none

az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name csr-ipsec1 --access Allow --protocol Udp --direction Inbound --priority 100 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range 500 -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name csr-ipsec2 --access Allow --protocol Udp --direction Inbound --priority 110 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range 4500 -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name allow-10slash --access Allow --protocol "*" --direction Inbound --priority 130 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name allow-192slash --access Allow --protocol "*" --direction Inbound --priority 140 --source-address-prefix 192.168.0.0/16 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none
az network nsg rule create -g $rgonprem --nsg-name onprem-csr-nsg --name allow-Outbound --access Allow --protocol "*" --direction Outbound --priority 120 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none

az network public-ip create -g $rgonprem --name onprem-csr-pip --idle-timeout 30 --allocation-method Static --sku standard -o none
az network nic create -g $rgonprem --name csr-ext-nic --subnet csr-external --vnet onprem-vnet --public-ip-address onprem-csr-pip --private-ip-address 10.100.0.4 --ip-forwarding true --network-security-group onprem-csr-nsg -o none
az network nic create -g $rgonprem --name csr-int-nic --subnet csr-internal --vnet onprem-vnet --ip-forwarding true --private-ip-address 10.100.1.4 --network-security-group onprem-csr-nsg -o none

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
az network nsg create -g $rgonprem -l $loconprem --name onprem-vm-nsg -o none

az network nsg rule create -g $rgonprem --nsg-name onprem-vm-nsg --name allow-10slash --access Allow --protocol "*" --direction Inbound --priority 130 --source-address-prefix 10.0.0.0/8 --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*" -o none

az network public-ip create --name onprem-testvm-pip -g $rgonprem -l $loconprem --allocation-method Dynamic -o none

az network nic create -g $rgonprem -l $loconprem -n testvm-nic --subnet test-vm-subnet --private-ip-address 10.100.10.10 --vnet-name onprem-vnet --public-ip-address onprem-testvm-pip --network-security-group onprem-vm-nsg  -o none

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
