---
title: "Referring Provider Dashboard"
excerpt: "A practice administrator approached me to help solve their issue of not being able to quickly see and identify referring provider data in the current practice manager system. The practice administrator asked if I could create dashboards to support quarterly auditing of these metrics. "
date: 2025-05-14
header:
  teaser: assets/images/referring-providers/org2-overview-obf.jpg
classes: wide
tags:
  - tableau
  - sql
  - real work
  - nextgen
  - healthcare

gallery1:
  - url: assets/images/referring-providers/by-users-obf.jpg
    image_path: assets/images/referring-providers/by-user-obf.jpg
    alt: "Org 1 by Staff Member"
    title: "Org 1 by Staff Member"
  - url: assets/images/referring-providers/by-state-obf.jpg
    image_path: assets/images/referring-providers/by-state-obf.jpg
    alt: "Org 1 by State"
    title: "Org 1 by State"
  - url: assets/images/referring-providers/by-provider-obf.jpg
    image_path: assets/images/referring-providers/by-provider-obf.jpg
    alt: "Org 1 by Provider"
    title: "Org 1 by Provider"

gallery2:
  - url: assets/images/referring-providers/org2-by-users-obf.jpg
    image_path: assets/images/referring-providers/org2-by-users-obf.jpg
    alt: "Org 2 by Staff Member"
    title: "Org 2 by Staff Member"
  - url: assets/images/referring-providers/org2-by-state-obf.jpg
    image_path: assets/images/referring-providers/org2-by-state-obf.jpg
    alt: "Org 2 by State"
    title: "Org 2 by State"
  - url: assets/images/referring-providers/org2-by-provider-obf.jpg
    image_path: assets/images/referring-providers/org2-by-provider-obf.jpg
    alt: "Org 2 by Provider"
    title: "Org 2 by Provider"

---

# Context
The practice administrator is tasked with identifying referring providers from NextGen. This is typically done manually or through a built-in report in the practice manager system. The goal is to create an easy and helpful dashboard in Tableau.

# How does it work?
The query looks at all appointment records:
- Where the date fell within the last 6 years. The query truncates the date to the year, meaning we will always have the full year’s worth of data. (For example, 6 years ago, as of typing this, is 2019-05-14. The query includes anything within 2019 forward.)
- Where the appointment event type is equal to "consult" or "cardiac risk assessment".
- Excludes any deleted, canceled, etc. appointments.
- Excludes any test patients.

# Dashboard
I created a single overview dashboard that gives the end user an overall understanding of the organization’s numbers. Metrics include the number of appointments by clinic location, the specific referring provider, and the rendering provider. 

These dashboards were created for two different organizations but were designed to look identical, with matching brand colors. 

Beyond the overview, additional dashboards were created to further understand the data. This included data related to the staff member who created the appointment, a view using location data, and a view focused on both the referring and rendering provider. All previous mentioned views were in table format.

The Tableau workbook includes 4 different dashboards:
- Referring Provider Dashboard showcasing: 
  - A line chart that shows the total number of referrals quarter to quarter
  - A pie chart of clinic locations
  - The number of appointments per referring providers
- A highlight table. This is helpful to see which provider referred in which year and month.
- Referring Provider by State. This dashboard combines a map view (by county) and the highlight table from before.
- Referrals created by users. This is meant to act as a way to audit users, and they date 



## Organization 1
{% include figure popup=true image_path="assets/images/referring-providers/overview-obf.jpg" alt="Org 1 Overview" caption="Org 1 Overview" %}
{% include gallery id="gallery1" caption="All dashboard for org 1." %}

## Organization 2
{% include figure popup=true image_path="assets/images/referring-providers/org2-overview-obf.jpg" alt="Org 2 Overview" caption="Org 2 Overview" %}
{% include gallery id="gallery2" caption="All dashboards for org 2." %}


# Technical Information
## SQL Script Information
The one piece worth highlighting in this query is the section that truncates a full year of data. The rest of the query is below. 

```sql
WHERE YEAR(appt.appt_date) >= YEAR(DATEADD(year,-6,GETDATE()))
```

## SQL Code
```sql
SELECT
	CAST(appt.[appt_date] AS DATE) appt_date
	,appt.practice_id
	,p.practice_name 
	,loc_m.location_name
	,e.event
	,CAST(pat.med_rec_nbr AS INT) AS MRN
	,appt.[description] AS patient_full_name
	,appt.[last_name] AS patient_last_name
	,appt.[first_name] AS patient_first_name
	,renpm.national_provider_id AS rendering_provider_npi
	,renpm.description AS rendering_provider_full_name
	,renpm.last_name AS rendering_provider_last_name
	,renpm.first_name AS rendering_provider_first_name	
	
	,refpm.national_provider_id AS referring_provider_npi
	,refpm.description AS referring_provider_full_name
	,refpm.last_name AS referring_provider_last_name
	,refpm.first_name AS referring_provider_first_name
	,refpm.address_line_1
	,refpm.address_line_2
	,refpm.city
	,refpm.state
	,refpm.zip
	,zcm.county
	,um.last_name + ', ' + um.first_name AS created_by_user_full_name
	,um.last_name AS created_by_user_last_name
	,um.first_name AS created_by_user_first_name

  FROM [ProdDB].[dbo].[appt_fact] AS appt
  LEFT OUTER JOIN [ProdDB].[dbo].[appt_detail] AS appt_m ON appt_m.appt_id = appt.appt_id -- used for appt deletion check
  LEFT OUTER JOIN [ProdDB].[dbo].[location_dim] AS loc_m ON loc_m.location_id = appt.location_id -- location
  LEFT OUTER JOIN [ProdDB].[dbo].[patient_dim] AS pat ON pat.person_id = appt.person_id -- patient info (mrn)
  LEFT OUTER JOIN [ProdDB].[dbo].[user_dim] AS um ON um.[user_id] = appt.created_by
  LEFT OUTER JOIN [ProdDB].[dbo].[event_dim] AS e ON e.event_id = appt.event_id -- event name filtering
  LEFT OUTER JOIN [ProdDB].[dbo].[provider_dim] refpm ON refpm.provider_id = appt.refer_provider_id -- referring provider
  LEFT OUTER JOIN [ProdDB].[dbo].[provider_dim] renpm ON renpm.provider_id = appt.rendering_provider_id -- rendering provider
  LEFT OUTER JOIN [ProdDB].[dbo].[zip_dim] AS zm ON zm.zip = refpm.zip
  LEFT OUTER JOIN [ProdDB].[dbo].[zip_county_dim] zcm ON zcm.zip = refpm.zip
  LEFT OUTER JOIN [ProdDB].[dbo].[practice_dim] AS p ON p.practice_id = appt.practice_id

  WHERE YEAR(appt.appt_date) >= YEAR(DATEADD(year, -3, GETDATE()))
	--AND appt.practice_id = 004
	AND (e.event = 'Consult' 
		OR e.event = 'Cardiac Risk Assessment'
		OR e.event = 'Vein Consult'
		)
	AND appt.delete_ind = 'N'
	AND appt_m.delete_ind = 'N'
	AND (
		(
			appt.appt_kept_ind = 'N' -- this means "checked in"
			AND (
				appt.appt_status IS NULL
				OR appt.appt_status <> 'P'
				)
			AND appt.cancel_ind = 'N'
			AND appt.appt_date >= GETDATE()
			)
		OR (
			appt.appt_kept_ind = 'Y'
			AND (
				appt.appt_status IS NULL
				OR appt.appt_status <> 'P'
				)
			)
		OR (
			appt.appt_status = 'P'
			AND appt.cancel_ind = 'N'
			)
		)
	--ORDER BY appt_date DESC, location_name, event
```