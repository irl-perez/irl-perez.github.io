---
title: "EHR System Error Dashboard"
excerpt: "The IT manager would like a dashboard that provides insight into EHR generated error logs. The concern is that many staff members may not alert the IT team. This dashboard aims to track those errors and address them before they become a bigger issue."
date: 2025-08-08
header:
  teaser: assets/images/ehr-system-error/error-dashboard-overview-obf.jpg
classes: wide
tags:
  - tableau
  - sql
  - real work
  - ehr
  - healthcare
gallery:
  - url: assets/images/ehr-system-error/error-dashboard-group_and_user-obf.jpg
    image_path: assets/images/ehr-system-error/error-dashboard-group_and_user-obf.jpg
    alt: "EHR Errors by Group and Users."
    title: "EHR Errors by Group and Users."
  - url: assets/images/ehr-system-error/error-dashboard-group_and_user-obf.jpg
    image_path: assets/images/ehr-system-error/error-dashboard-group_and_user-obf.jpg
    alt: "EHR Errors by Group and Users."
    title: "EHR Errors by Group and Users."
---

# Context
The IT manager would like a dashboard that provides insight into EHR generated error logs. The concern is that many staff members may not alert the IT team. This dashboard aims to track those errors and address them before they become a bigger issue.

## Terminology
Within the error logs, there is a field called `Message` that contains multiple lines of information. That field is parsed to group each data record into a specific error. This field is known as `ErrorSource`.

Within Tableau, `ErrorSource` is further aggregated into a field called `Error Group`. This is meant to remove errors related to "variables" in their messages. For example, "ID not found: 123456". In this example, we'd remove the "123456" to aggregate our data.

# How does it work?
We query the EHR reporting database for error logs and returns all records from 3 months ago to the current date. 

Each message can vary. To allow for data aggregation, error messages were truncated to the second line, preserving the line that provides insight into the error. This is known as the `ErrorSource`. `ErrorSource` is further truncated to remove variables and length, allowing for more aggregation in Tableau.

# Dashboards
The initial dashboard shows an error in the user’s location and system group. Additionally, featuring errors over time to showcase specific days when errors occurred. A second dashboard shows errors by users. The idea is to identify users with high error rates so our EHR team can investigate further. Finally, for a more detailed view of error groups, I included an error table dashboard that shows how each error message is classified. 

{% include figure popup=true image_path="assets/images/ehr-system-error/error-dashboard-overview-obf.jpg" alt="EHR Error Dashboard Overview" caption="EHR Error Dashboard Overview"%}

{% include gallery id="gallery" caption="Additional EHR Error Dashboards." %}

# Technical Information
{: .notice--info}
**SQL Query Information:**
It's worth noting that since the original error message is truncated, a user might need to go back and look at the database. To assist with this, there is a field in the query called `sqlQuery` that includes all fields and searches just for the current record. This query can be accessed through Tableau by downloading the "full data".
## SQL Script Information
This report is similar to [PENDING UPDATE] takes that query and slightly modifies it. For details, see [PENDING UPDATE].

## Tableau Information
### Resource Group Field
The CTE has a large `CASE` statement that checks for common error messages, for example, "Dicom Carotid". If these words are mentioned, the entire record is considered to be coming from that `ErrorSource`.

After the common errors, there is a `CHARINDEX` that parses through the longer messages to find the error.

### SQL Query
```sql
DROP TABLE IF EXISTS #TempErrorData;

WITH source_a AS (
    SELECT
        'ORG_A' AS Organization,
        err.ID AS ErrorID,
        loc.LocationCode,
        err.UserKey,
        CASE 
            WHEN usr.DisplayName IS NULL OR usr.DisplayName = '' THEN 'Unknown User'
            ELSE usr.DisplayName
        END AS UserName,
        CASE
            WHEN role.RoleID IS NOT NULL THEN 'Yes'
            ELSE 'No'
        END AS IsSpecialRole,
        err.SystemName,
        err.EventTime,
        err.EventMessage,

        CASE 
            WHEN err.EventMessage LIKE 'Resource limit exceeded%' 
                THEN 'Resource constraint issue...'

            WHEN CHARINDEX('Module A', err.EventMessage) > 0 
                THEN 'Module A related...'

            WHEN CHARINDEX('Module B', err.EventMessage) > 0 
                THEN 'Module B related...'

            WHEN CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) > 0 
                THEN TRIM(SUBSTRING(
                    err.EventMessage,
                    CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) + LEN('-----') + 2,
                    CHARINDEX(CHAR(13), err.EventMessage,
                        CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) + LEN('-----') + 2
                    ) - (
                        CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) + LEN('-----') + 2
                    )
                ))

            WHEN err.EventMessage LIKE 'at SomeNamespace.ProcessAction%' 
                THEN 'Application process failure...'

            ELSE err.EventMessage
        END AS ErrorCategory,

        err.AppName,

        SUBSTRING(
            err.AppName,
            CHARINDEX('v', err.AppName),
            CHARINDEX(' -', err.AppName) - CHARINDEX('v', err.AppName)
        ) AS AppVersion,

        err.NotificationFlag,

        CONCAT('SELECT * FROM ErrorLog WHERE ID = ', err.ID) AS DebugQuery

    FROM DataWarehouseA.dbo.ErrorLog err
    LEFT JOIN CoreDB.dbo.Users usr 
        ON usr.UserKey = err.UserKey AND usr.IsActive = 1
    LEFT JOIN CoreDB.dbo.UserRoles role 
        ON role.UserKey = err.UserKey
    LEFT JOIN CoreDB.dbo.Locations loc 
        ON loc.LocationID = usr.LocationID

    WHERE err.EventTime >= DATEADD(MONTH, -3, GETDATE())
      AND err.EventMessage NOT LIKE '%operation completed successfully%'
),

source_b AS (
    SELECT
        'ORG_B' AS Organization,
        err.ID AS ErrorID,
        loc.LocationCode,
        err.UserKey,
        CASE 
            WHEN usr.DisplayName IS NULL OR usr.DisplayName = '' THEN 'Unknown User'
            ELSE usr.DisplayName
        END AS UserName,
        CASE
            WHEN role.RoleID IS NOT NULL THEN 'Yes'
            ELSE 'No'
        END AS IsSpecialRole,
        err.SystemName,
        err.EventTime,
        err.EventMessage,

        CASE 
            WHEN err.EventMessage LIKE 'Resource limit exceeded%' 
                THEN 'Resource constraint issue...'

            WHEN CHARINDEX('Module A', err.EventMessage) > 0 
                THEN 'Module A related...'

            WHEN CHARINDEX('Module B', err.EventMessage) > 0 
                THEN 'Module B related...'

            WHEN CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) > 0 
                THEN TRIM(SUBSTRING(
                    err.EventMessage,
                    CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) + LEN('-----') + 2,
                    CHARINDEX(CHAR(13), err.EventMessage,
                        CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) + LEN('-----') + 2
                    ) - (
                        CHARINDEX('-----', err.EventMessage, CHARINDEX('-----', err.EventMessage) + 1) + LEN('-----') + 2
                    )
                ))

            WHEN err.EventMessage LIKE 'at SomeNamespace.ProcessAction%' 
                THEN 'Application process failure...'

            ELSE err.EventMessage
        END AS ErrorCategory,

        err.AppName,

        SUBSTRING(
            err.AppName,
            CHARINDEX('v', err.AppName),
            CHARINDEX(' -', err.AppName) - CHARINDEX('v', err.AppName)
        ) AS AppVersion,

        err.NotificationFlag,

        CONCAT('SELECT * FROM ErrorLog WHERE ID = ', err.ID) AS DebugQuery

    FROM LinkedServerB.ReportingDB.dbo.ErrorLog err
    LEFT JOIN LinkedServerB.CoreDB.dbo.Users usr 
        ON usr.UserKey = err.UserKey AND usr.IsActive = 1
    LEFT JOIN LinkedServerB.CoreDB.dbo.UserRoles role 
        ON role.UserKey = err.UserKey
    LEFT JOIN LinkedServerB.CoreDB.dbo.Locations loc 
        ON loc.LocationID = usr.LocationID

    WHERE err.EventTime >= DATEADD(MONTH, -6, GETDATE())
      AND err.EventMessage NOT LIKE '%operation completed successfully%'
)

SELECT *
INTO #TempErrorData
FROM source_a

UNION ALL

SELECT *
FROM source_b;
```
