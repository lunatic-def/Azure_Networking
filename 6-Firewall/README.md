

**1 - Create virtual network and subnets:**
- VNET: 
    + name: vnet
    + region: east asia
    + cidr: 10.0.0.0/16
- Subnet: 
    + name: Workload-SN - range: 10.0.2.0/24
    + name: Bastion-SN - range: 10.0.3.0/24
    + name: AzureFirewallSubnet - range: 10.0.1.128/26 - Subnet purpose: Firewall Management (forced tunneling)
    + name: AzureFirewallManagementSubnet - range: 10.0.1.192/26 - Subnet purpose: Firewall Management (forced tunneling)

```ps
az network vnet create --name Test-FW-VN --resource-group langocanh --location eastasia --address-prefix 10.0.0.0/16 --subnet-name AzureFirewallSubnet --subnet-prefix 10.0.1.128/26

az network vnet subnet create --name AzureFirewallManagementSubnet --resource-group langocanh --vnet-name Test-FW-VN --address-prefix 10.0.1.192/26 

az network vnet subnet create --name Workload-SN --resource-group langocanh --vnet-name Test-FW-VN --address-prefix 10.0.2.0/24

az network vnet subnet create --name Jump-SN --resource-group langocanh --vnet-name Test-FW-VN --address-prefix 10.0.3.0/24
```



**2 - Create VM in Workload-SN and Bastion host**
- Workload: 
    + Name: Workload - network: 10.0.2.0/24
```ps
# NIC configure with DNS server:
az network nic create -g langocanh --name workload-nic -l eastasia --vnet-name Test-FW-VN --subnet Workload-SN --dns-server 209.244.0.3 209.244.0.4

az vm create --resource-group langocanh --name workload-vm --location eastasia --image win2016datacenter --nics workload-nic --admin-username azureuser
```

- Bastion host: 
    + Name: Bastion - network: 10.0.3.0/24
    + Public IP
```ps
az vm create -g langocanh -n jump-vm --location eastasia --vnet-name Test-FW-VN --subnet Jump-SN --admin-username azureuser --image win2016datacenter

az vm open-port --port 3389 -g langocanh -n jump-vm
```

**3 - Deploy Azure firewall**
- Deploy firewall and create networking for firewall

```ps
# create firewall
az network firewall create --name myfirewall -g langocanh -l eastasia

# create static public ip for firewall
az network public-ip create --name firewall-pubip -g langocanh -l eastasia --allocation-method static --sku standard

az network firewall ip-config create -n firewall-config --firewall-name myfirewall --public-ip-address firewall-pubip -g langocanh --vnet-name Test-FW-VN

```

**4 - Create a default route**
- Create a route table, with BGP route propagation disabled
```ps
az network route-table create --name firewall-route -g langocanh -l eastasia --disable-bgp-route-propagation true

# Create the route - route to firewall
az network route-table route create -g langocanh -n route-to-firewall --route-table-name firewall-route --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address 10.0.1.132 (Firewall IP)

# Add route table to the Workload-subnet
az network vnet subnet update -n Workload-SN -g langocanh --vnet-name Test-FW-VN --address-prefixes 10.0.2.0/24 --route-table firewall-route
```

**5 - Configure application rule**
- Application rule to allow Workload-subnet to outbound access to website only "www.google.com"

```ps
 az network firewall application-rule create --collection-name app-rule1 --firewall-name myfirewall --name Allow-Google --protocols Http=80 Https=443 -g langocanh --target-fqdns www.google.com --source-addresses 10.0.2.0/24 --priority 200 --action Allow
```

**6 - Configure network rule**
- Network rule to allow Workload-subnet to DNS server with UDP protocol

```ps
az network firewall network-rule create --collection-name net-rule1 --destination-addresses 209.244.0.3 209.244.0.4 --destination-ports 53 --firewall-name myfirewall --name allowDNS --protocols UDP -g langocanh --priority 200 --source-address 10.0.2.0/24 --action Allow
```

**7 - Configure DNAT rule**
 - Source * to Firewall Public IP translate to WorkLoad-IP on port 3389

```ps
az network firewall nat-rule create -n RDB_DNAT --firewall-name myfirewall -g langocanh --collection-name dnat-rule1 --protocols TCP --source-addresses '*' --dest-addr 52.229.200.132 --destination-ports 3389 --translated-address 10.0.2.4 --translated-port 3389 --action Dnat

```

**7 - Test firewall connections**

Connect RDP to Workload-VM and test :))
- Enable diagnostic settings and queries for logs 
```ps
 mstsc /v:Srv-Work
```

```ps
nslookup www.google.com
nslookup www.microsoft.com
```

```ps
Invoke-WebRequest -Uri https://www.google.com
Invoke-WebRequest -Uri https://www.google.com

Invoke-WebRequest -Uri https://www.microsoft.com
Invoke-WebRequest -Uri https://www.microsoft.com
```

1. Remote RDP into Workload-VM through Firewall public IP  **Azure Firewall logs data**
pic ..

**LOGS**


2. NSlookup for DNS 

**LOGS**

3. Website access

**LOGS**




Logs reference: 
Link : Azure Diagnostics 
Azure Networking
https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-network-azurefirewalls-logs

Firewall
https://learn.microsoft.com/en-us/azure/azure-monitor/reference/queries/azurediagnostics#queries-for-microsoftnetwork