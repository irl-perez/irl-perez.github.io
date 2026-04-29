---
title: "CSQ Call Volume Dashboard"
excerpt: "The size of this project was massive. At first, what started as a verification check on the dashboard to ensure call numbers were accurate eventually turned into a complete check of our database ingestion system."
date: 2025-09-15
header:
  #overlay_image: assets/images/csq-call-volume/csq-agent-call-volume-obf.jpg
  #caption: "Tableau CSQ Call Volume Dashboard"
  teaser: assets/images/csq-call-volume/csq-agent-call-volume-obf.jpg
classes: wide
tags:
  - tableau
  - dashboard
  - sql
  - real work

gallery1:
  - url: assets/images/csq-call-volume/csq-tableau-numbers-obf.jpg
    image_path: assets/images/csq-call-volume/csq-tableau-numbers-obf.jpg
    alt: "Tableau Dashboard Example. One CSQ."
    title: "Tableau Dashboard Example."
  - url: assets/images/csq-call-volume/csq-cisco-numbers-obf.jpg
    image_path: assets/images/csq-call-volume/csq-cisco-numbers-obf.jpg
    alt: "Finesse Report Example. One CSQ."
    title: "Finesse Report Example."
  
gallery2:
  - url: assets/images/csq-call-volume/csq-queue-relationship.jpg
    image_path: assets/images/csq-call-volume/csq-queue-relationship.jpg
    alt: "Tableau relationships"
    title: "Tableau relationships"
  - url: assets/images/csq-call-volume/csq-queue-call-volume-obf.jpg
    image_path: assets/images/csq-call-volume/csq-queue-call-volume-obf.jpg
    alt: "CSQ Call Volume"
    title: "CSQ Call Volume"
  - url: assets/images/csq-call-volume/csq-agent-call-volume-obf.jpg
    image_path: assets/images/csq-call-volume/csq-agent-call-volume-obf.jpg
    alt: "Agent Call Volume"
    title: "Agent Call Volume"
  - url: assets/images/csq-call-volume/csq-queue-call-wait-time-obf.jpg
    image_path: assets/images/csq-call-volume/csq-queue-call-wait-time-obf.jpg
    alt: "Queue Call Wait Time"
    title: "Queue Call Wait Time"
  - url: assets/images/csq-call-volume/csq-agent-call-by-callers-obf.jpg
    image_path: assets/images/csq-call-volume/csq-agent-call-by-callers-obf.jpg
    alt: "Top Callers and Reason"
    title: "Top Callers and Reason"
  - url: assets/images/csq-call-volume/csq-queue-call-to-clinics-obf.jpg
    image_path: assets/images/csq-call-volume/csq-queue-call-to-clinics-obf.jpg
    alt: "Calls to Clinics"
    title: "Calls to Clinics"
  - url: assets/images/csq-call-volume/csq-agent-call-by-ringbacks-obf.jpg
    image_path: assets/images/csq-call-volume/csq-agent-call-by-ringbacks-obf.jpg
    alt: "Unanswered Calls"
    title: "Unanswered Calls"


---
{% include figure popup=true image_path="assets/images/csq-call-volume/csq-agent-call-volume-obf.jpg" alt="Agent Call Volume Dashboard Example." %}
# Context
REDACTED voiced her concern that the Tableau reports do not match (in metrics and numbers) those of Cisco Unified Intelligence Center.

# How does it work?
The Finesse reports use an Informix Database. REDACTED and REDACTED are unable to access this database directly. Instead we decided to host a copy of the necessary components on our instance of `REDACTED`, under the `REDACTED` database. 

There are multiple scheduled jobs that run reoccurring that collect data based on yesterday's time frame. In other words, our data is always a day behind. This was to prevent inaccurate numbers by collecting data up to the instant.

We then use Tableau to connect to our database and display the data.

# Dashboards
The original Tableau report has multiple dashboards. My goal was to recreate them accurately and document them for future staff members.

{% include gallery id="gallery2" caption="All dashboards created." %}

## CSQ Call Volume Dashboard
Biggest concept to understand is that all calls here are from the perspective of the CSQ. These are calls before they reach agents.
### Call Volume by Day View
Regarding the metrics: total number of calls, abandoned calls, and handled calls we quickly realize that the sum of abandon and handled calls don't equal total number of calls.

This is because we exclude several disposition reasons:
- Null
- Dequeued from CSQ 
- Handled by another CSQ
- Handled by script

This is the same method that CUIC reflect its numbers.

Within Tableau I handled it using a LOD expression and then combined this field with a dual axis. 
```sql
{FIXED 
    [contactQueueDetail_startDateOnlyCST], [CSQ Group], [csqname]: 
COUNT(([CSQ Data Set]))}
```

Keep in mind this is the only visualization where we manipulate the total. The other visualizations that work along side *Call Volume by Day* are not manipulated:
- Call Volume by Clinic
- Call Volume by Hour

## Agent Call Volume
### Call Volume by Day (Handled Calls)
In order to look at call volumes per agent we need to filter for calls that were considered *Handled* (`Queue Disposition Decoded = Handled`).
## Top Callers and Reason
This dashboard logically remains nearly identical.
### Unanswered Clinic Calls
New dashboard. That takes advantage of Tableau's "data blending". Our primary source is missed calls and is "left joined" with secondary data source. This causes some records to read as NULL and there wasn't a way I could find to correct this behavior. 
## Calls to Clinics
Nothing to comment here. 
## Queue Wait Times
Nothing to comment here. 

# Technical Information
## Database Discrepancy 
When I approached this project, I was made aware that my company had access to the Informix database via a "linked server". This database contains the call records we were interested in.

In our SQL Server database, there was a job that looked for new records and collected them every 5 minutes into a local instance. Through investigation, I realized that the numbers were inaccurate and that collecting records every five minutes may not be enough time for the records to settle.

{% include figure popup=true image_path="assets/images/csq-call-volume/database-discrepancy-obf.jpg" alt="Database Discrepancy" %}

Instead, I suggested we run a daily extract rather than every five minutes.

```sql
-- example of one job: 
USE CiscoCallData

DECLARE 
	@currentDate VARCHAR(30)
	,@startTimeGMT VARCHAR(30)
	,@stopTimeGMT VARCHAR(30)
	,@TSQL varchar(8000);

SELECT
	@currentDate = CAST(GETDATE() AS DATE)
	,@startTimeGMT = CONVERT(NVARCHAR,CAST(DATEADD(day,-1,@currentDate) AT TIME ZONE 'Central Standard Time' AT TIME ZONE 'UTC' AS DATETIME),121)
	,@stopTimeGMT = CONVERT(NVARCHAR,CAST(CAST(@currentDate AS DATETIME) AT TIME ZONE 'Central Standard Time' AT TIME ZONE 'UTC' AS DATETIME),121)

SELECT 
	@currentDate AS currentDate
	,@startTimeGMT AS startTimeGMT
	,@stopTimeGMT AS stopTimeGMT

SET @TSQL = 'INSERT INTO 
				ContactCallDetail
			SELECT		
				-- field selection
			FROM OpenQuery(UCCX1,''SELECT
										-- field selection from IBM DB
									FROM contactcalldetail
									WHERE startDateTime >=  ''''' + @startTimeGMT + '''''
										AND startDateTime < ''''' + @stopTimeGMT + '''''
									ORDER BY startDateTime ASC
							''); '
EXEC(@TSQL)
```
After deleting and reinserting records, this proved successful!

{% include figure popup=true image_path="assets/images/csq-call-volume/database-compare-new-and-old-obf.jpg" alt="Comparing old and new records" %}

After ensuring the database was reflecting accurate information, it took more creative thinking to mimic Cisco's Finesse Reporting numbers. Through trial and error and data validation, this was also successful.

{% include gallery id="gallery1" caption="Comparing internal Tableau dashboard against Finesse's reporting tool." %}