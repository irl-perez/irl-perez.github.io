---
title: "Resource Groups Next Available Slots"
excerpt: "Leadership wanted their phone staff to have quicker access to the next available appointment data. This means, instead of digging into the practice manager application, they can reference the dashboard to find an appointment slot more quickly."
date: 2025-03-11
header:
  teaser: assets/images/resource-groups-next-available-slots/slots-by-resource-obf.jpg
classes: wide
tags:
  - tableau
  - sql
  - real work
  - nextgen
  - healthcare
---

# Context
Leadership member at the virual care center wanted to transform information sent through a daily email into a dashboard that could be seen daily by the phone answering team. The daily email contains data regarding appointment availability. 

It would list a modality group, such as SPECT or Treadmill, and the available dates for scheduling appointments. The task is done manually by nurses at the end of the day. They would go through each schedule individually and note which dates had openings.

An update in March 2025 included [OTHER ORGANIZATION] data and newly built Tableau dashboards.

## Terminology
Resource Group: In this context, resource group signifies the name of the group of the resources:
- Lab
- CT
- Echo
- PET
- SPECT
- RPM
- Stress Echo
- Treadmill
- Vascular

# How does it work?
The report looks at NextGen appointment data as of yesterday. The dashboard does not update in real-time. The report collects data only when "resource type" equals "place" in NextGen.

Appointment data is filtered to exclude inactive appointments (e.g., "canceled"). 

The resource names are grouped by their title. There is logic in place that looks at the beginning of the title and categorizes it appropriately. Here are some of the examples:
- If the resource name begins with "Nuclear PET," then it's under "PET."
- If the resource name begins with "Stress Echo," then it's under "Stress Echo."

# Dashboards
Because there are two organizations, I developed these data views accordingly. Two distinct dashboards were created: Organization 1 requested a separation between its two main clinics, while Organization 2 requested the data divided by region.

Cells highlighted in green indicate a low number of slots, meaning the day is mostly booked, ideal for a clinic. In contrast, red-highlighted slots show high availability, so scheduling staff should focus on those days.

## Organization 1
{% include figure popup=true image_path="assets/images/resource-groups-next-available-slots/org1-overview-obf.jpg" alt="Org 1: Resource Groups Next Available Slots" caption="Org 1: Resource Groups Next Available Slots." %}

## Organization 2
{% include figure popup=true image_path="assets/images/resource-groups-next-available-slots/org2-overview-obf.jpg" alt="Org 2: Resource Groups Next Available Slots" caption="Org 2: Resource Groups Next Available Slots." %}

## Combined Organizations and Granular Resource View
While I believe these dashboards were great for presentation, I went a step further and created a third dashboard that combined both organizations and provided multiple filters that let me drill down to see which specific “resources” were contributing to which groups. This view is especially helpful for leadership users who navigate Tableau.
{% include figure popup=true image_path="assets/images/resource-groups-next-available-slots/slots-by-resource-obf.jpg" alt="Combined orgs." caption="Combined orgs." %}


# Technical Information
## SQL Script Information
This report is similar to [PENDING UPDATE] takes that query and slightly modifies it. For details, see [PENDING UPDATE].

## Tableau Information
### Resource Group Field
There is a single calculated field called `Resource Group`. This uses an if statement to group. Here's the logic:
```python
IF STARTSWITH([resource_name],'Lab') = TRUE THEN 'Lab'
    ELSEIF STARTSWITH([resource_name],'CT') THEN 'CT'
    ELSEIF STARTSWITH([resource_name],'Echo') THEN 'Echo'
    ELSEIF STARTSWITH([resource_name],'Nuclear PET') THEN 'PET'
    ELSEIF STARTSWITH([resource_name],'Nuclear SPECT') THEN 'SPECT'
    ELSEIF STARTSWITH([resource_name],'RPM') THEN 'RPM'
    ELSEIF STARTSWITH([resource_name],'Stress Echo') THEN 'Stress Echo'
    ELSEIF STARTSWITH([resource_name],'Treadmill') THEN 'Treadmill'
    ELSEIF STARTSWITH([resource_name],'Vascular') THEN 'Vascular'
    ELSE "Other"
END
```

### Organization 2 Region Group
With the update implemented in March 2025, organization 2 was added to the report. In order to visualize organization 2 reports we broke up the clinic locations into a field know as `organization 2 Regions`, here is the logic:
```sql
CASE [mst_location_name]
WHEN IN (
    'Location A1',
    'Location A2',
    'Location A3'
) THEN 'Region A'

WHEN IN (
    'Location B1',
    'Location B2',
    'Location B3',
    'Location B4',
    'Location B5',
    'Location B6',
    'Location B7',
    'Location B8'
) THEN 'Region B'

WHEN IN (
    'Location C1',
    'Location C2',
    'Location C3',
    'Location C4',
    'Location C5',
    'Location C6',
    'Location C7'
) THEN 'Region C'

WHEN IN (
    'Location D1'
) THEN 'Region D'
END
```
Since this includes all regions in a single view, the coloring marks would appear skewed, showing only the highest values in red and the rest in green. 
### Ranking for Coloring
The logic behind how we color numeric values has now changed. Previously, we would look at all numeric values from 1 to n. The scale would then determine where the minimum, median, and max where and color according. This logic is flawed due to the wide range of numbers. In other words, if we have open slots in the 20s and 30s, but another resource group has a value of 200, the 20 and 30 values would most likely appear green.

To combat this, I've used Tableau's "rank calculated table" function. I'd recommend [reading online](https://help.tableau.com/current/pro/desktop/en-us/functions_functions_tablecalculation.htm) for more information. 
