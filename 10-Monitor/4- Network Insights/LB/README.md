
Task 1: Setup VM with IIS Servers

```ps
$RGName = "langocanh"

New-AzResourceGroupDeployment -ResourceGroupName $RGName -TemplateFile azuredeploy.json -TemplateParameterFile azuredeploy.parameters.json
```
Task 2: Setup LoadBalancing and test Connections and gain Logs informations


- LOGS:
 ( https://learn.microsoft.com/en-us/azure/azure-monitor/reference/supported-logs/microsoft-network-loadbalancers-logs )

(Microsoft.Network/loadBalancers) - Latest SNAT Port Exhaustion per LB Frontend




Task 3: Use a Functional Dependency View && View detailed metrics 

Task 4: View resource health

