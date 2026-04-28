---
title: "Smart Form Dashboard"
excerpt: "The goal of this project was to gain insight and audit “smart forms”. Smart forms are template documents that staff can use when communicating with hospitals, clinics, and patients."
date: 2024-11-26
header:
  teaser: assets/images/smart-form-dashboard/main-obfuscated.jpg
gallery:
  - url: assets/images/smart-form-dashboard/main-obfuscated.jpg
    image_path: assets/images/smart-form-dashboard/main-obfuscated.jpg
    alt: "Smartform dashboard."
classes: wide
---

# Context
Leadership lacks the ability to audit which user created specific smart forms within our system. Additionally, leadership lacks a report to gain valuable insights into smart form metrics. This report will not only allow users to download data but also gain a quick overview through the dashboard.

# How does it work?
The report will run on the OMS reporting server (meaning the data will be one day behind production). The data query looks at the following parameters:
- Only active smart forms are included.
- Where the smart form created date is between yesterday and 60 days before yesterday (inclusive).

{% include figure popup=true image_path="assets/images/smart-form-dashboard/main-obfuscated.jpg" alt="Smartform Dashboard" caption="Smartform Dashboard." %}

# Technical Information
## SQL Script Information
This select handles joining our `SmartFormData` with any relevant information.

When investigated, the location had two possible fields:
- `Facility` which is a list of "locations" that the user has access to within OMS. This field is found in the `Users` table. I do not recommend using this field because it takes the first option selected in OMS as it appears.
- `Default Facility`: This is the preferred field. I spoke to \[REDACTED\], and she described this field as the "location they belong to." When looking at this field, there are only three options:
	- A valid facility
	- 0: 0 meant the field was unassigned in OMS. The OMS team can update this field.
	- NULL: The null option reflects the fact that we don't have the "user" associated with the smart form; in other words, we don't know who created it. I noticed a couple of records posted on or after 10/22/2024 that still reported not having the user. If the problem persists, it may be worth reporting to the OMS development team for further investigation.

I decided to construct a user's "full name" based on the `FirstName` and `LastName` fields in the database. This felt more consistent than using the `FullName` field, which was not the same as those fields. In many cases, the `FullName` field contained a user's title.

## SQL Code
```sql
-- =============================================
-- Author:      Oscar Perez
-- Create date: 2024-11-21
-- Description: Collects data related to Smart for data. The idea was to see a raw count of users.
--
-- Change History:
--		2024-11-21: Document created.
--		2024-11-22:  Do not use Users.FacilityID this ID is the first Facility selected from the OMS general settings menu. 
-- =============================================
DROP TABLE IF EXISTS #tempSFD;

WITH SmartFormData AS (
  SELECT
    PatientID
    ,TemplateID
    ,CreatedDate
    ,AppUser AS rawAppUser
    ,SUBSTRING(AppUser, CHARINDEX('-', AppUser) + 2, LEN(AppUser) - CHARINDEX('-', AppUser)-2) AS 'OMSUserName'
  FROM [REDACTED].[dbo].[SmartForms] sf
  WHERE IsActive = 1
    AND sf.CreatedDate >= DATEADD(DAY,-61, CAST(GETDATE() AS DATE)) -- wokring with whole dates instead of timestamps.
    AND sf.CreatedDate < CAST(GETDATE() AS DATE)

)
SELECT 
  --sfd.PatientID
  demP.MedicalRecordNumber
  ,sfd.TemplateID
  ,temp.Name as templateName
  ,sfd.CreatedDate
  ,CAST(sfd.CreatedDate AS date) createdDateDateOnly
  ,Users.UserID
  --,sfd.rawAppUser
  ,sfd.OMSUserName
  ,defFac.FacilityID
  ,CASE
    WHEN defFac.FacilityID = 0 THEN 'Unassigned Location'
    WHEN defFac.FacilityID IS NULL THEN 'Unknown Location'
    WHEN defFac.FacilityID > 0 THEN demFac.FacilityCode
    ELSE 'ERROR'
  END AS 'Facility Name'
  ,CASE
    WHEN sfd.OMSUserName IS NOT NULL THEN CONCAT(Users.LastName,', ', Users.FirstName) -- manually creating the user's full name because the full name field in OMS is a mess
    ELSE NULL
  END AS lastFirstName
INTO #tempSFD
FROM SmartFormData sfd 
  LEFT OUTER JOIN [REDACTED].[dbo].[Users] ON Users.UserName COLLATE DATABASE_DEFAULT = sfd.OMSUserName
  LEFT OUTER JOIN [REDACTED].[dbo].[Demographics.Patient] demP ON demP.PatientID = sfd.PatientID
  LEFT OUTER JOIN [REDACTED].[dbo].tblTemplate temp ON temp.Id = sfd.TemplateID
  LEFT OUTER JOIN [REDACTED].[dbo].[UsersDefaultFacility] defFac ON defFac.UserID = Users.UserID -- 1. checking this table first for facilitiy
  LEFT OUTER JOIN [REDACTED].[dbo].[Demographics.Facility] demFac ON demFac.FacilityID = defFac.FacilityID -- ^2. getting the facility name from this join

ORDER BY createdDate DESC
```