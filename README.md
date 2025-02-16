# vWAN-to-vWAN-Connection-Options
This artcile explains the various conection options available to connect two different Virtual WAN environments. Often customers who are not using high bandwith links like Express-Route because of cost want other options to connect different Virtual Wan Environments! They also might have SDWan links that they already use to connect to On-Prem and want to continue with that method! 

# Option 1: Native Azure S2S IPSec Tunnels
In this option customers can simply provision Azure Virutal Network Gateways on each vhub in each vWAN environment to connect two different vWAN environments. Its important to note that today you cannot change the vWAN virtual Network Gateway ASN, which is 65515.This is hardcoded inside the vhub gateway config. At at later date, there should be an option to change the Virutal Network Gateway ASN. For this reason, you cannot do BGP over IPSEC due to BGP loop prevention and seeing its own ASN in the path. The remote vhub will receive routes from the source vhub with 65515 in the AS-PATH and BGP will drop that due to loop prevention. Therefore if you want to connect two different vWAN environments, the tunnels need to be IKEv1 or IKEv2, but no BGP option. The other thing to note is max throughput you can get per tunnel is 2.4Gbps, and this is an SDN limitation. This can be increased by "N" number of tunnels. 

> [!NOTE]
As mentioend above on vWAN gateways we cannot change the default 65515 ASN today:

![image](https://github.com/user-attachments/assets/872b5cf8-f07d-41a8-aecd-39b042bdd02c)


![image](https://github.com/user-attachments/assets/59630f3e-18e6-48b5-a14b-8ae821952151)

# Option 2: ExpressRoute Using MSEE Hair-Pinning
This option is also using Azure native solutions. If there is already an Express-Route circuit provisioned, the customer can simply hook the vhubs in each vWAN environment to the same express-route circuit. Just like regular hub+spoke, when vnets are connected to the same circuit the egress point is the MSEE at the pop location. So, traffic between each vhub in different vWANs will simply hairpin down the the Azure MSEEs and each pop location before ingressing to the remote vhub. Simmilar to option 1, this is easy to setup but is not ideal due to added latency due to hairpinning, and the taxing of the remote gateway on the ingress flow path. 
> [!NOTE]
If the circuit is a direct port circuit, fast-path can be used on the vhub gateways in order to bypass the gateway for inbound flows.

![image](https://github.com/user-attachments/assets/17258a2f-de52-4103-8b8c-022e27a06cbb)

# Option 3: SDWAN Tunnels with BGP+IPSEC Inside the vHubs
This option is good for customers who are already using SDWAN devices today to connect to on-prem WAN links or cannot afford Express-Route, or don't need the bandwidth. [Users can deploy approved SDWAN devices](https://learn.microsoft.com/en-us/azure/virtual-wan/about-nva-hub#partners) inside the vhubs in each respective vWAN environment to build SDWAN tunnels with BGP+IPSEC to connect each vWAN environment. In order to make the routing work though, inside the SDWAN operating system, its necessary to do "AS-Replace or AS-Path Exclude" BGP commands for the vhub 65520 ASN and Route Server instances 65515. So the command would be for example "as-path exclude 65520 65515" or simmmilar depending on the SDWAN vendor. You would then need to apply that route-map to each NVA BGP peer. That way, the remote vWAN vhub won't drop the route, because it won't see its own ASN in the path on the receving route. This is the same behavior is option 1, except here we have the ability to do BGP manipulation on third party devices unlike the Azure Virutal Network Gateway.The SDWANs can use different ASNs and do eBGP, or they could also be the same ASN and have an iBGP session. 

![image](https://github.com/user-attachments/assets/0367b69f-becb-4096-9602-a9686cd78b92)

# Option 3a: SDWAN Tunnels with BGP+IPSEC with SDWAN Devices BGP Peered In Spokes
This would be the simmilar to option 3, except we have the SDWAN device in a spoke vnet that is vnet peered to each vhub. We then would [BGP peer the SDWAN device to the Route Server instances inside the vhub](https://learn.microsoft.com/en-us/azure/virtual-wan/scenario-bgp-peering-hub). This is a good for scenarios where customers have SDWAN devices that cannot be deployed inside vhubs, but still support BGP. Like above, we need to apply route-maps to each NVA and do the same as-exclude or as-replace on both 65520 and 65515 so the receiving end does not drop the routes becuse it would see its own ASN.

![image](https://github.com/user-attachments/assets/c6ab9df5-68db-4c20-99fd-d72d9f0bd1cb)

> [!NOTE]
> Although this covers vWAN to vWAN connection options, the remote side could be a regular vnet in hub+spoke topology as well. Going back to option 1, we could add BGP because the remote gateway and vnet are not inside the vhub, and would have the ability to change the gateway ASN to something other then 65515. For Option 2 however, in order to allow the MSEE hairpin with non-vwan vnets, we need to allow the hair-pinning on both sides, **the vhub + the remote virutal network gateway**! On new deployments MSEE hairpinning is **blocked by default, it must manually allowed**!

# Allow MSEE Hairpin between vWAN and non vWAN vnets (Option 2 -Modified)
 vHub side:
 <br>
 ![image](https://github.com/user-attachments/assets/654d1707-bac5-4b1a-8122-caaceb48404e)
 <br>
 <br>
 Express-Route Gatway -Regular Vnet:
 <br>
 ![image](https://github.com/user-attachments/assets/13b7725c-244d-427a-b968-f8b0820ad21e)











