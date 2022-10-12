# Azure Hybrid Networking Routing Lab Series

## Lab 3 - Deploying PaloAlto VM Series Firewall in Hub VNET and BGP peering PAN with Cisco CSR and Azure Route Server in Azure

### Introduction

This lab deploys PaloAlto VM Series Firewall (PAN) in Hub VNET and is BGP peered with Cisco CSR and Azure Route Server in Azure Hub. Goal of this lab is to force 0/0 traffic originating from spoke VNETs via PAN.

North-South traffic (to internet) via PAN

East-West traffic (to on-prem and other VNETs) bypasses PAN

> *This lab is for testing/learning purposes only and should not be considered production configurations*

### Networking Architecture

![lab-3-architecture](assets/lab-3-architecture.png)

### Expected Traffic Flow Post lab 3 deployment

![lab-3-traffic-flow](assets/lab-3-traffic-flow.png)
