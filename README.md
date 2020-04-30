# UPDATE - App Service vnet integration now works with private dsn zones 
Custom DNS not required. 

* All you need to do is add the following into web.config:
   * WEBSITE_DNS_SERVER with value 168.63.129.16
   * WEBSITE_VNET_ROUTE_ALL with value 1
* Documentation [link](https://docs.microsoft.com/en-us/azure/app-service/web-sites-integrate-with-vnet)
* [Example Deployment (Azure CLI)](https://github.com/fireblade95402/Private-Endpoint/blob/master/example.azcli "Example Deployment (Azure CLI)") 

I'll update the below to reflect this soon!
   
# Private Endpoint conenction to Azure SQL Server from an App Service

This shows a way of connecting to an Azure SQL database from an app Serivce using a Private Endpoint.
The reason I'm showing this is that I'd have challenges getting it working and realised their were a few services that were still needing to go GA before this would work as I'd expect.
What I notices is that when you create the endpoint and try and connect to it from an App Service it couldn't see the server. 
I then created a VM in the some VNET that the App Service was integrated with and was able to connect to the database testing it with ODBC (quick and dirty test). This proved the private endpoint was working as expected, but not with the App Service.
The reason for this is that as I right this post App Services that are VNET integrated don't work with Private DNS Zones, which underpins  private endpoint.
One of the quick wins to get this working is to deploy a custom DNS server in the app service VNET and create a forwarder to 168.63.129.16 ([Name resolution for Azure resources](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances)).

Below I'll be going through a high level walkthrough to create this solution in Azure and proving the private connection between App service and Azure SQL database.

Here's a rough architecture diagram showing a design  
![Architecture](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image13.JPG "Private endpoint Architecture") 

## Prerequisites

* Create 2x VNET's.
    * VNET to integrate the app service.
    * VNET to host the private endpoint for Azure SQL.  
    *(Can do the same with 1 VNET)*
    * If you create 2x VNET's. Please remember to peer them together. (VNET Peering)
* Create an app service with an application that connects to  Azure SQL DB ([Example to build this quickly](https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-dotnetcore-sqldb-app))


## Step 1 - Create Private Endpoint for the Azure SQL Server

This will create a private endpoint for the Azure SQL Server that was created above.  

* Go to the Azure SQL Server / "Private endpoint connections"  
![Azure SQL Server](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image1.JPG "Azure SQL Server - Private endpoint connections")  

* Click on "+ Private endpoint" to create a new endpoint  
    * Basics
        * Name for the new private endpoint connection  
        * Region to create it in.  
        ![Azure SQL Server](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image2.JPG "Azure SQL Server - Private endpoint connections - Basics")  

    * Resource
        * Select the resource type "Microsoft.Sql/servers.  
        * Select Sql server created above.  
        ![Azure SQL Server](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image3.JPG "Azure SQL Server - Private endpoint connections - Resource")  

    * Confirguration
        * Select a VNET created above  
        * Select subnet where the Private Endpoint will be created in.  
        * If for your first Private Endpoint. You'll need to select Yes for Integrate with Private DNS zone  
        ![Azure SQL Server](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image4.JPG "Azure SQL Server - Private endpoint connections - Configuration")  

    * You can added Tags is required.  
    * Click "Review + Create" to create the Private Endpoint.  
    * Once this has been completed you will be able to see that a new private endpoint and if selected a new Private DNS zone in your resource group:  
    ![Azure SQL Server](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image5.JPG "Azure SQL Server - Private endpoint and Private DNS zone")  
    (*Private endpoint name above incorrect in image*)  
    * Goto the Private DNS zone resource and link the VNET's created in the prerequisites (one might already to linked):  
    ![Private DNS Zone](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image14.JPG "Private DNS zone - VNET's")  

## Step 2 - Integrate App Service into VNET

This will integration the App Service into the one of the VNET's created above. To allow it to be connect to the Private Endpoint created above.

* Go to the Azure App Service  / Networking  
* Click to configure VNET Integration to integrate to a VNET created above.  
![App Serivce](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image6.JPG "Configure VNET ")  
* Configure the integration by selecting a VNET and subnet to integrate with.  
![App Service](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image7.JPG "Integrate with VNET")  



## Step 3 - Setup Custom DNS Server

A customer DNS server needs to be setup to complete the key configuration before this can be tested.

I'm keeping this high level. 

- Create a Windows 2016+ VM in the VNET where the App service resides.  
- Enable the DNS role on the Server.  
- Configure the forwarder to goto 168.63.129.16 ([Name resolution for Azure resources](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances)).  
- Change the app service VNET to use the custom DNS Service IS Address  
![App Service - VNET](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image9.JPG "Set custom DNS Server on app service VNET")  

## Notes

* The Azure SQL server URL for the private endpoint is found in the private endpoint resource.  
![Private Endpoint](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image8.JPG "Private Endpoint URL")  
* Remember to check that you're using the Azure SQL Server URL above in the app serivce connection string.  
![App Service](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image10.JPG "App Service Connection String")  
* You can also check transactions created in the Azure SQL Database by first turning on auditing and then checking transactions generated by the App Serivce. Which would come from the App Serivce and allocated by the VNET it's integrated with.  
![Azure SQL Database](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image11.JPG "Audit check of client IP")  
* You can also see that in the "Firewalls and virtual networks" section in the Azure SQL Server. That there's no public access, firewall rules for access or any integrations to VNETs. All access would be via the Private Endpoint.  
![Azure SQL Server](https://github.com/fireblade95402/Private-Endpoint/blob/master/img/image12.JPG "Firewalls and virtual networks")  

## Links

* [Name resolution for Azure resources](https://docs.microsoft.com/en-us/azure/virtual-network/virtual-networks-name-resolution-for-vms-and-role-instances)
* [Example to build this quickly](https://docs.microsoft.com/en-us/azure/app-service/containers/tutorial-dotnetcore-sqldb-app)
* [Azure Private Endpoint](https://docs.microsoft.com/en-us/azure/private-link/private-endpoint-overview)




