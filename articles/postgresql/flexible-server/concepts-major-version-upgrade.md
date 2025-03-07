---
title: Major Version Upgrade  - Azure Database for PostgreSQL - Flexible Server 
description: Learn about the concepts of in-place major version upgrade with Azure Database for PostgreSQL - Flexible Server
author: kabharati
ms.author: kabharati
ms.reviewer: rajsell
ms.date: 12/30/2023
ms.service: postgresql
ms.subservice: flexible-server
ms.custom: references_regions
ms.topic: conceptual
---

# Major Version Upgrade for PostgreSQL Flexible Server 

[!INCLUDE [applies-to-postgresql-Flexible-server](../includes/applies-to-postgresql-Flexible-server.md)]

Azure Database for PostgreSQL Flexible Server supports PostgreSQL versions 11, 12, 13, 14, 15 and 16. Postgres community releases a new major version containing new features about once a year. Additionally, major version receives periodic bug fixes in the form of minor releases. Minor version upgrades include changes that are backward-compatible with existing applications. Azure Database for PostgreSQL Flexible Server periodically updates the minor versions during customer’s maintenance window. Major version upgrades are more complicated than minor version upgrades, as they can include internal changes and new features that may not be backward-compatible with existing applications. 

## Overview 

Azure Database for PostgreSQL Flexible Server Postgres has now introduced in-place major version upgrade feature that performs an in-place upgrade of the server with just a click. In-place major version upgrade simplifies the upgrade process minimizing the disruption to users and applications accessing the server. In-place upgrades are a simpler way to upgrade the major version of the instance, as they retain the server name and other settings of the current server after the upgrade, and don't require data migration or changes to the application connection strings. In-place upgrades are faster and involve shorter downtime than data migration. 


## Process

Here are some of the important considerations with in-place major version upgrade. 

- During in-place major version upgrade process,  Flexible Server runs a pre-check procedure to identify any potential issues that might cause the upgrade to fail. If the pre-check finds any incompatibilities, it creates a log event showing that the upgrade pre-check failed, along with an error message. 

- If the pre-check is successful, then Flexible Server stops the service and takes an implicit backup just before starting the upgrade. This backup can be used to restore the database instance to its previous version if there's an upgrade error. 

- Flexible Server uses  [**pg_upgrade**](https://www.postgresql.org/docs/current/pgupgrade.html) utility to perform in-place major version upgrades and provides the flexibility to skip versions and upgrade directly to higher versions. 

-	During an in-place major version upgrade of a High Availability (HA) enabled server, the service disables HA, performs the upgrade on the primary server, and then re-enables HA after the upgrade is complete. 

-	Most extensions are automatically upgraded to higher versions during an in-place major version upgrade, with some exceptions. Refer **limitations** section for more details. 

-	In-place major version upgrade process for Flexible Server automatically deploys the latest supported minor version. 

-	The process of performing an in-place major version upgrade is an offline operation that results in a brief period of downtime. Typically, the downtime is under 15 minutes, although the duration may vary depending on the number of system tables involved.

-	Long-running transactions or high workload before the upgrade might increase the time taken to shut down the database and increase upgrade time. 

-	If an in-place major version upgrade fails, the service restores the server to its previous state using a backup taken as part of second step described in this list.

-	Once the in-place major version upgrade is successful, there are no automated ways to revert to the earlier version. However, you can perform a Point-In-Time Recovery (PITR) to a time prior to the upgrade to restore the previous version of the database instance.

## Limitations  

If in-place major version upgrade pre-check operations fail, then the upgrade aborts with a detailed error message for all the below limitations.

- In-place major version upgrade currently doesn't support read replicas, so if you have a read replica enabled server, you need to delete the replica before performing the upgrade on the primary server. After the upgrade, you can recreate the replica. 

- In-place major version upgrade doesn't support certain extensions and there are some limitations to upgrading certain extensions. The extensions **Timescaledb**, **pgaudit**, **dblink**, **orafce** and **postgres_fdw** are unsupported for all PostgreSQL versions. 

-	When upgrading servers with PostGIS extension installed, set the `search_path` server parameter to explicitly include the schemas of the PostGIS extension, extensions that depend on PostGIS, and extensions that serve as dependencies for the below extensions.
  **e.g postgis,postgis_raster,postgis_sfcgal,postgis_tiger_geocoder,postgis_topology,address_standardizer,address_standardizer_data_us,fuzzystrmatch (required for postgis_tiger_geocoder).**

-	Servers configured with logical replication slots aren't supported. 

## How to perform in-place major version upgrade: 

It's recommended to perform a dry run of the in-place major version upgrade in a non-production environment before upgrading the production server. It allows you to identify any application incompatibilities and validate that the upgrade completes successfully before upgrading the production environment. You can perform a Point-In-Time Recovery (PITR) of your production server, and restore it in the non-production environment to test the upgrade there. Addressing these issues before the production upgrade minimizes downtime and ensures a smooth upgrade process. 

**Steps**

1. You can perform in-place major version upgrade using Azure portal or CLI (command-line interface).  Click the **Upgrade** button in Overview blade.

   :::image type="content" source="media/concepts-major-version-upgrade/upgrade-tab.png" alt-text="Diagram of Upgrade tab to perform in-place major version upgrade.":::

2. You'll see an option to select the major version of your choice, you have an option to skip versions to directly upgrade to higher versions. Choose the version and click **Upgrade**. 

   :::image type="content" source="media/concepts-major-version-upgrade/set-postgresql-version.png" alt-text="Diagram of PostgreSQL version to Upgrade.":::

3. During upgrade, users have to wait for the process to complete. You can resume accessing the server once the server is back online. 

   :::image type="content" source="media/concepts-major-version-upgrade/deployment-progress.png" alt-text="Diagram of deployment progress for major version upgrade.":::

4. Once the upgrade is successful, you can expand the **Deployment details** tab and click **Operation details**  to see more information about upgrade process like duration, provisioning state etc. 

   :::image type="content" source="media/concepts-major-version-upgrade/deployment-success.png" alt-text="Diagram of successful deployment of for major version upgrade.":::

5. You can click on the **Go to resource** tab to validate your upgrade. Notice that server name remained unchanged and PostgreSQL version upgraded to desired higher version with the latest minor version. 

   :::image type="content" source="media/concepts-major-version-upgrade/upgrade-verification.png" alt-text="Diagram of Upgraded version to Flexible Server after major version upgrade.":::


## Post upgrade

Run the **ANALYZE** operation to refresh the `pg_statistic` table. You should do this for every database on your Flexible Server. Optimizer statistics aren't transferred during a major version upgrade, so you need to regenerate all statistics to avoid performance issues. Run the command without any parameters to generate statistics for all regular tables in the current database, as follows:


```
VACUUM ANALYZE VERBOSE;
```
> [!NOTE]   
>
> The VERBOSE flag is optional, but using it shows you the progress. 

> [!NOTE]  
> If you have pg_qs enabled and collecting data on an instance of PostgreSQL running a major version <= 14, and perform an [in-place major version upgrade](./concepts-major-version-upgrade.md) to any version >= 15, know that the values returned in the query_type column of query_store.qs_view for any newly created time windows can be considered correct. However, for all the time windows which were created when the version of the engine was <= 14, where it reports `merge` it corresponds to `utility`, and when it reports `nothing` it corresponds to `utility`. The reason for that inconsistency has to do with the way [MERGE statement was implemented in PostgreSQL](https://github.com/postgres/postgres/commit/7103ebb7aae8ab8076b7e85f335ceb8fe799097c), which, instead of appending a new item to the existing ones in the CmdType enum, interleaved an item for MERGE between DELETE and UTILITY.
 
## Next steps

- Learn about [business continuity](./concepts-business-continuity.md).
- Learn about [zone-redundant high availability](./concepts-high-availability.md).
- Learn about [backup and recovery](./concepts-backup-restore.md).

