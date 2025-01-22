1. VNet and subnet

2. Create application security groups
- WebServer ASG: 'myAsgWebServers' and 'myAsgMgmtServers'
```ps
az network asg create -g $rg -n myAsgWebServers -l southeastasia
az network asg create -g $rg -n myAsgMgmtServers -l southeastasia
```
*Pics:*


3. Create network security groups 
- Network SG: 'myNsg'
```ps
az network nsg create -g $rg -n myNsg -l southeastasia
```
- Associate subnet 'myVNet' and subnet 'default' to NSG: 
```ps
az network vnet subnet update -g $rg -n default  --vnet-name myVNet --network-security-group myNsg 
```
*pics*

- Create INBOUND NSG to all traffic to **web servers** and **RDP to the servers**
    + myAsgWebServers   = inbound rule - tcp - 80,443
    + myAsgMgmtServers  = inbount rule - tcp - 3389

```ps
az network nsg rule create -g $rg --nsg-name myNsg -n nsgwithasgweb --source-address-prefixes Internet --destination-port-ranges 80 443 --destination-asgs myAsgWebServers --access allow --protocol tcp --description "allow internet access to web on port 80,443" --priority 100
```

```ps
az network nsg rule create -g $rg --nsg-name myNsg -n nsgwithasgMgmt --source-address-prefixes Internet --destination-port-ranges 3389 --destination-asgs myAsgMgmtServers --access allow --protocol tcp --description "allow internet access to Mgmt Server with RDP" --priority 110
```

*pics*

4. Create VM for testing 
*pics*

- Associate each VM network interface to its ASG
```
# Get the Network Interface Card (NIC) ID of the VM
$nicId= $(az vm show -n myVMMgmt -g langocanh --query networkProfile.networkInterfaces[0].id -o tsv)

# Associate the ASG with the NIC's IP configuration
$ipConfigName= $(az network nic show --ids $nicId --query ipConfigurations[0].name -o tsv)
az network nic ip-config update --resource-group langocanh --nic-name myvmmgmt814_z1 --name $ipConfigName --application-security-groups myAsgMgmtServers

az network nic ip-config update --resource-group langocanh --nic-name myvmweb170_z1 --name ipconfig1 --application-security-groups myAsgWebServers
```

**TESTING**
- Access RDP on MgmtServer:

- Access Web Server:
Run Powershell script to install Webserver:
```ps
 Install-WindowsFeature -name Web-Server -IncludeManagementTools
```

- CANNOT RDP into WebServer:


# Log Analytics workspace: 
**Network security events**


# NSG flow log: 
**IPv4 NSG Flow Log Search**


