---
title: "Lab Results for 340B"
excerpt: "Due to a 340B program and government-mandated policies, our organization needs to report test results within 24 to 48 hours of receiving them. This project highlights the use of PowerShell to accomplish that!"
date: 2025-04-28
header:
  teaser: assets/images/lab-results-340b/lab-orders-code-teaser.jpg
classes: wide
tags:
  - powershell
  - sql
  - real work
  - ehr
  - healthcare
---

# Context
Due to a government policy we are required to report test results within 24 hours if they are positive or within 48 hours if they are negative. Here is a generalization of the workflow:
1. The staff member submits an order with the indicated date.
2. Patient comes in for a test on the date.
3. The results of the test are submitted to a patient's chart note.
4. Staff members need to review the results and fill out a long Excel file formatted exactly to the specifications.

The current workflow includes a daily, repetitive task that requires manual data entry into the government website. The goal is to quickly locate the test results and give them to staff members for review before submitting them to the government. 

## Terminology
- CPOE: Computerized Physician Order Entry. This part of the system allows the user to enter various types of orders (labs, tests, etc.)
- Order Date: Is the date the order was submitted into the CPOE system.
- Study Date: Is the date the order was scheduled for.

# How does it work?
Our current solutions use Microsoft "Task Scheduler" to run a custom PowerShell script that queries our OMS database. 

The PowerShell script has a small Hash table that uses `LONIC` to link to test names. Here is a generalization of the script steps:
1. Locate any CPOE orders that are scheduled for the current day.
2. Loop through each record and look at each line of the chart note for the specific test name. Get the result from the next line. 
3. Send the final file.

# Limitations
The chart note in the EHR system is a free-text field (`VARCHAR(MAX)`), which leaves room for user error. If the user enters the test name or the results incorrectly, the script will not return the correct results. For a transparent view of the script, here is the hash table that translates the `LONIC` code:

```powershell
$codeToTestName = @{
    "10000-1" = "Test Alpha"
    "20000-2" = "Test Beta"
    "30000-3" = "Test Gamma"
}
```

The test name written above must appear exactly the same in the chart note.

Additionally, the result options must be "positive" or "negative". Capitalization does not matter. If the script is unable to locate either of these, it returns the original value (the value it found on the next line):

```PowerShell
switch ($result.ToLower()) {
	"positive" { return "Reactive" }
	"negative" { return "Non-Reactive" }
	default    { return $result }
}
```

In order to combat user error, the organization has created a copy-and-paste template for these results.

# Technical Information
## SQL Script Information
The SQL script contains many hard-coded values. These values are typically not required by the state. Other values are identified through a CASE statement. 

I spent time testing the EHR system to make sure this logic is correct. An example is the logic concerning "race" and the `GroupCode`. This was a faster approach than identifying how these tables are linked in the backend. 
## `WHERE` condition
The query locates CPOE orders where:
- `StudyDate` equals the current date
- The `TerminologyDescription` (name of the test) starts with [REDACTED].
- The order is marked completed (`status = 3`).
- Excluded "test" patients.
## Chart Note Subquery
The chart note sub query finds only **one** chart note where:
- The `PatientID` on the order matches the `PatientID` on the Chart note.
- The chart note contains any one of the test names: [Screening Alpha], [Screening Beta], or [Screening Gamma].
- The `ChartNoteDate` (the date the chart note was created) equals the `StudyDate` (the date the order was scheduled for).

## PowerShell Script
The script contains the SQL script mentioned above. It loops through each record, then through each line of the script and the chart note, until it finds the name of the test (based on the `LONIC` code mentioned above).

The current version of the script emails the results in a CSV. This may change in the future to include a direct SFTP drop.

```powershell
# Mapping of internal identifiers to generalized screening names
$identifierMap = @{
    "10000-1" = "Screening Alpha"
    "20000-2" = "Screening Beta"
    "30000-3" = "Screening Gamma"
}

# Database connection placeholders
$serverName = 'prod-server'
$dbName = 'EnterpriseDB'

$query = @"
SELECT
    loc.LocationName       AS SiteName,
    p.RecordNumber         AS RecordID,
    p.LastName             AS LastName,
    p.FirstName            AS FirstName,
    c.AddressLine1         AS Address,
    c.City                 AS City,
    st.StateCode           AS State,
    LEFT(c.PostalCode,5)  AS ZipCode,

    CASE
        WHEN demo.Gender = 0 THEN 'F'
        WHEN demo.Gender = 1 THEN 'M'
        ELSE 'U'
    END AS Gender,

    CONVERT(VARCHAR(8), CAST(ord.ModifiedDate AS DATE), 112) AS ResultDate,

    CASE
        WHEN term.TermID = 111111 THEN '10000-1'
        WHEN term.TermID = 222222 THEN '20000-2'
        WHEN term.TermID = 333333 THEN '30000-3'
    END AS IdentifierCode,

    CASE
        WHEN term.TermID = 111111 THEN 'Screening Alpha'
        WHEN term.TermID = 222222 THEN 'Screening Beta'
        WHEN term.TermID = 333333 THEN 'Screening Gamma'
    END AS TestName,

    notes.NoteText AS RawResult

FROM [EnterpriseDB].[dbo].[Orders] ord
LEFT JOIN [EnterpriseDB].[dbo].[Terminology] term
    ON term.TermID = ord.TermID
LEFT JOIN [EnterpriseDB].[dbo].[Patients] p
    ON p.PatientID = ord.PatientID
LEFT JOIN [EnterpriseDB].[dbo].[Contacts] c
    ON c.ContactID = p.ContactID
LEFT JOIN [EnterpriseDB].[dbo].[States] st
    ON st.StateID = c.StateID
LEFT JOIN [EnterpriseDB].[dbo].[Facilities] loc
    ON loc.FacilityID = ord.FacilityID
LEFT JOIN [EnterpriseDB].[dbo].[ClinicalNotes] notes
    ON notes.PatientID = ord.PatientID

WHERE
    CAST(ord.ModifiedDate AS DATE) = CAST(GETDATE() AS DATE)
    AND term.Description LIKE '%screening%'
    AND ord.Status = 3
ORDER BY ord.ModifiedDate ASC
"@

# Execute query
$results = Invoke-Sqlcmd `
    -ServerInstance $serverName `
    -Database $dbName `
    -Query $query `
    -TrustServerCertificate

function Get-CleanedResult {
    param (
        [string]$noteText,
        [string]$testName
    )

    $cleanText = $noteText -replace '<[^>]+>', ''
    $lines = $cleanText -split "`r?`n"

    for ($i = 0; $i -lt $lines.Count; $i++) {
        $line = $lines[$i].Trim()

        if ($line.ToLower().EndsWith("$($testName.ToLower()):")) {

            if ($i + 1 -lt $lines.Count) {
                $result = $lines[$i + 1].Trim()

                switch ($result.ToLower()) {
                    "positive" { return "Flagged" }
                    "negative" { return "Clear" }
                    default    { return $result }
                }
            }
        }
    }

    return "Result unavailable"
}

foreach ($row in $results) {

    $identifier = $row.IdentifierCode
    $note = $row.RawResult

    if ($identifierMap.ContainsKey($identifier)) {

        $testName = $identifierMap[$identifier]

        $cleanResult = Get-CleanedResult `
            -noteText $note `
            -testName $testName

        $row.RawResult = $cleanResult
    }
    else {
        $row.RawResult = "Unknown identifier"
    }
}

# Export results
$tempFile = "$env:TEMP\results_export.csv"

$results | Export-Csv `
    $tempFile `
    -NoTypeInformation

# Email placeholders
$smtpServer = "smtp.internal.local"
$from = "automation@company.local"
$to = @("team@company.local")

$subject = "Daily Screening Results"

$body = @'
Attached is the automated screening report.

Generated by internal automation workflow.
'@

Send-MailMessage `
    -From $from `
    -To $to `
    -Subject $subject `
    -SmtpServer $smtpServer `
    -Body $body `
    -Attachments $tempFile
```
