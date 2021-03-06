---
title: 'Azure reference architecture: Web application with high availability | Microsoft Docs'
description: Recommended architecture for web application with high availability, running in Microsoft Azure.
services: app-service,app-service\web,sql-database
documentationcenter: na
author: MikeWasson
manager: roshar
editor: ''
tags: ''

ms.assetid: 7b7e3cd7-ecc6-4ca4-a4e9-468b0fa8ad9e
ms.service: guidance
ms.devlang: na
ms.topic: article
ms.tgt_pltfrm: na
ms.workload: na
ms.date: 11/23/2016
ms.author: mwasson

---
# Running a web application with high availability in Azure
[!INCLUDE [pnp-RA-branding](../../includes/guidance-pnp-header-include.md)]

This article shows a recommended architecture for running a web application in multiple regions, to achieve high availability. The architecture builds on [Improving scalability in a web application][guidance-web-apps-scalability].

A multi-region architecture can provide higher availability than deploying to a single region. If a regional outage affects the primary region, you can use [Traffic Manager][traffic-manager] to fail over to the secondary region. This architecture can also help if an individual subsystem of the application fails.

There are several general approaches to achieving high availability across regions: 

* Active/passive with hot standby. Traffic goes to one region, while the other waits on hot standby. Hot standby means the VMs in the secondary region are allocated and running at all times.
* Active/passive with cold standby. Traffic goes to one region, while the other waits on cold standby. Cold standby means the VMs in the secondary region are not allocated until needed for failover. This approach costs less to run, but will generally take longer to come online during a failure.
* Active/active. Both regions are active, and requests are load balanced between them. If one region becomes unavailable, it is taken out of rotation. 

This reference architecture focuses on active/passive with hot standby, using Traffic Manager for failover. 


## Architecture diagram
![Reference architecture: Web application with high availability](media/blueprints/paas-web-app-multi-region.png) 

This architecture builds on the one shown in [Improving scalability in a web application][guidance-web-apps-scalability]. The main differences are:

* **Primary and secondary regions**. This architecture uses two regions to achieve higher availability. The application is deployed to each region. During normal operations, network traffic is routed to the primary region. If the primary region becomes unavailable, traffic is routed to the secondary region. For details about failover, see [Managing failover](#managing-failover-and-failback).
* **Azure Traffic Manager**. [Traffic Manager][traffic-manager] routes incoming requests to the primary region. If the application running that region becomes unavailable, Traffic Manager fails over to the secondary region.
* **Geo-replication** of SQL Database and DocumentDB. 

## Recommendations

Your requirements might differ from the architecture described here. Use the recommendations in this section as a starting point.

### Regional pairing
Each Azure region is paired with another region within the same geography. In general, choose regions from the same regional pair (for example, East US 2 and Central US). Benefits of doing so include:

* If there is a broad outage, recovery of at least one region out of every pair is prioritized.
* Planned Azure system updates are rolled out to paired regions sequentially to minimize possible downtime.
* In most cases, regional pairs reside within the same geography to meet data residency requirements.

However, make sure that both regions support all of the Azure services needed for your application. See [Services by region][services-by-region]. For more information about regional pairs, see [Business continuity and disaster recovery (BCDR): Azure Paired Regions][regional-pairs].

### Resource groups
Consider placing the primary region, secondary region, and Traffic Manager into separate [resource groups][resource groups]. This lets you manage the resources deployed to each region as a single collection.

### Traffic Manager configuration 

**Routing**. Traffic Manager supports several [routing algorithms][tm-routing]. For the scenario described in this article, use *priority* routing (formerly called *failover* routing). With this setting, Traffic Manager sends all requests to the primary region unless the endpoint for that region becomes unreachable. At that point, it automatically fails over to the secondary region. See [Configure Failover routing method][tm-configure-failover].

**Health probe**. Traffic Manager uses an HTTP (or HTTPS) probe to monitor the availability of each endpoint. The probe gives Traffic Manager a pass/fail test for failing over to the secondary region. It works by sending a request to a specified URL path. If it gets a non-200 response within a timeout period, the probe fails. After four failed requests, Traffic Manager marks the endpoint as degraded and fails over to the other endpoint. For details, see [Traffic Manager endpoint monitoring and failover][tm-monitoring].

As a best practice, create a health probe endpoint that reports the overall health of the application and use this endpoint for the health probe. The endpoint should check critical dependencies such as the App Service apps, storage queue, and SQL Database. Otherwise, the probe might report a healthy endpoint when critical parts of the application are actually failing.

On the other hand, don't use the health probe to check lower priority services. For example, if an email service goes down the application can switch to a second provider or just send emails later. This is not a high enough priority to cause the application to fail over. For more information, see [Health Endpoint Monitoring Pattern][health-endpoint-monitoring-pattern].
 
### SQL Database
Use [Active Geo-Replication][sql-replication] to create a readable secondary replica in a different region. You can have up to four readable secondary replicas. Fail over to a secondary database if your primary database fails or needs to be taken offline. Active Geo-Replication can be configured for any database in any elastic database pool.

### DocumentDB
DocumentDB supports geo-replication across regions. One region is designated as writable and the others are read-only replicas.

If there is a regional outage, you can fail over by selecting another region to be the write region. The DocumentDB client SDK automatically sends write requests to the current write region, so you don't need to update the client configuration after a failover. For more information, see [Distribute data globally with DocumentDB][docdb-geo].

> [!NOTE]
> All of the replicas belong to the same resource group.
>
>

### Storage
For Azure Storage, use [read-access geo-redundant storage][ra-grs] (RA-GRS). With RA-GRS storage, the data is replicated to a secondary region. You have read-only access to the data in the secondary region through a separate endpoint. If there is a regional outage or disaster, the Azure Storage team might decide to perform a geo-failover to the secondary region. There is no customer action required for this failover.

For Queue storage, create a backup queue in the secondary region. During failover, the app can use the backup queue until the primary region becomes available again. That way, the application can still process new requests.

## Availability considerations


### Traffic Manager

Traffic Manager automatically fails over if the primary region becomes unavailable. When Traffic Manager fails over, there is a period of time when clients cannot reach the application. The duration is affected by the following factors:

* The health probe must detect that the primary data center has become unreachable.
* Domain name service (DNS) servers must update the cached DNS records for the IP address, which depends on the DNS time-to-live (TTL). The default TTL is 300 seconds (5 minutes), but you can configure this value when you create the Traffic Manager profile.

For details, see [About Traffic Manager Monitoring][tm-monitoring].

Traffic Manager is a possible failure point in the system. If the service fails, clients cannot access your application during the downtime. Review the [Traffic Manager service level agreement (SLA)][tm-sla] and determine whether using Traffic Manager alone meets your business requirements for high availability. If not, consider adding another traffic management solution as a failback. If the Azure Traffic Manager service fails, change your canonical name (CNAME) records in DNS to point to the other traffic management service. This step must be performed manually, and your application will be unavailable until the DNS changes are propagated.

### SQL Database
The recovery point objective (RPO) and estimated recovery time (ERT) for SQL Database are documented in [Overview of business continuity with Azure SQL Database][sql-rpo]. 

### Storage
RA-GRS storage provides durable storage, but it's important to understand what can happen during an outage:

* If a storage outage occurs, there will be a period of time when you don't have write-access to the data. You can still read from the secondary endpoint during the outage.
* If a regional outage or disaster affects the primary location and the data there cannot be recovered, the Azure Storage team may decide to perform a geo-failover to the secondary region.
* Data replication to the secondary region is performed asynchronously. Therefore, if a geo-failover is performed, some data loss is possible if the data can't be recovered from the primary region.
* Transient failures, such as a network outage, will not trigger a storage failover. Design your application to be resilient to transient failures. Possible mitigations:
  
  * Read from the secondary region.
  * Temporarily switch to another storage account for new write operations (for example, to queue messages).
  * Copy data from the secondary region to another storage account.
  * Provide reduced functionality until the system fails back.

For more information, see [What to do if an Azure Storage outage occurs][storage-outage].

## Manageability Considerations

### Traffic Manager

If Traffic Manager fails over, we recommend performing a manual failback rather than implementing an automatic failback. Otherwise, you can create a situation where the application flips back and forth between regions. Verify that all application subsystems are healthy before failing back.

Note that Traffic Manager automatically fails back by default. To prevent this, manually lower the priority of the primary region after a failover event. For example, suppose the primary region is priority 1 and the secondary is priority 2. After a failover, set the primary region to priority 3, to prevent automatic failback. When you are ready to switch back, update the priority to 1.

The following commands update the priority.

**PowerShell**

```bat
$endpoint = Get-AzureRmTrafficManagerEndpoint -Name <endpoint> -ProfileName <profile> -ResourceGroupName <resource-group> -Type AzureEndpoints
$endpoint.Priority = 3
Set-AzureRmTrafficManagerEndpoint -TrafficManagerEndpoint $endpoint
```

For more information, see [Azure Traffic Manager Cmdlets][tm-ps].

**Azure command line interface (CLI)**

```bat
azure network traffic-manager endpoint set --name <endpoint> --profile-name <profile> --resource-group <resource-group> --type AzureEndpoints --priority 3
```    

### SQL Database
If the primary database fails, perform a manual failover to the secondary database. See [Restore an Azure SQL Database or failover to a secondary][sql-failover]. The secondary database remains read-only until you fail over.


<!-- links -->

[azure-sql-db]: https://azure.microsoft.com/en-us/documentation/services/sql-database/
[docdb-geo]: ../documentdb/documentdb-distribute-data-globally.md
[guidance-web-apps-scalability]: guidance-web-apps-scalability.md
[health-endpoint-monitoring-pattern]: https://msdn.microsoft.com/library/dn589789.aspx
[ra-grs]: ../storage/storage-redundancy.md#read-access-geo-redundant-storage
[regional-pairs]: ../best-practices-availability-paired-regions.md
[resource groups]: ../azure-resource-manager/resource-group-overview.md#resource-groups
[services-by-region]: https://azure.microsoft.com/en-us/regions/#services
[sql-failover]: ../sql-database/sql-database-disaster-recovery.md
[sql-replication]: ../sql-database/sql-database-geo-replication-overview.md
[sql-rpo]: ../sql-database/sql-database-business-continuity.md#sql-database-features-that-you-can-use-to-provide-business-continuity
[storage-outage]: ../storage/storage-disaster-recovery-guidance.md
[tm-configure-failover]: ../traffic-manager/traffic-manager-configure-failover-routing-method.md
[tm-monitoring]: ../traffic-manager/traffic-manager-monitoring.md
[tm-ps]: https://msdn.microsoft.com/en-us/library/mt125941.aspx
[tm-routing]: ../traffic-manager/traffic-manager-routing-methods.md
[tm-sla]: https://azure.microsoft.com/en-us/support/legal/sla/traffic-manager/v1_0/
[traffic-manager]: https://azure.microsoft.com/en-us/services/traffic-manager/
