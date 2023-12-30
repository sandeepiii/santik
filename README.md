# santik
All my scripts 
SQL DBA Monitoring scripts and when ever i create any scripts will be keeping it here
#########################################################
#
# Disk space monitoring and reporting script
# Reference from https://www.red-gate.com/simple-talk/sysadmin/powershell/disk-space-monitoring-and-early-warning-with-powershell/
# This script will give alerts for disk space if threshold is breached or above the threshold
#########################################################
 
$users = "sra@gmail.com" # List of users to email your report to (separate by comma)
$fromemail = "no-reply@pnd.io"
$server = "email-smtp.us-.com" #enter your own SMTP server DNS name / IP address here
$smtpPort = '587'
$list = "F:\Files\list.txt"  #This accepts the argument you add to your scheduled task for the list of servers. i.e. list.txt
$computers = Get-Content $list #grab the names of the servers/computers to check from the list.txt file.
$cred = 'put your credentials'
# Set free disk space threshold below in percent (default at 10%)
[decimal]$thresholdspace = 50

$secpasswd = ConvertTo-SecureString "PASSWORD" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("CREDENTIALS", $secpasswd)
 
#assemble together all of the free disk space data from the list of servers and only include it if the percentage free is below the threshold we set above.
$tableFragment= Get-WMIObject  -ComputerName $computers Win32_LogicalDisk `
| select __SERVER, DriveType, VolumeName, Name, @{n='Size (Gb)' ;e={"{0:n2}" -f ($_.size/1gb)}},@{n='FreeSpace (Gb)';e={"{0:n2}" -f ($_.freespace/1gb)}}, @{n='PercentFree';e={"{0:n2}" -f ($_.freespace/$_.size*100)}} `
| Where-Object {$_.DriveType -eq 3 -and [decimal]$_.PercentFree -lt [decimal]$thresholdspace} `
| ConvertTo-HTML -fragment 
 
# assemble the HTML for our body of the email report.
$HTMLmessage = @"
<font color=""black"" face=""Arial, Verdana"" size=""3"">
<u><b>Disk Space Storage Report</b></u>
<br>This report was generated because the drive(s) listed below have less than $thresholdspace % free space. Drives above this threshold will not be listed.
<br>
<style type=""text/css"">body{font: .8em ""Lucida Grande"", Tahoma, Arial, Helvetica, sans-serif;}
ol{margin:0;padding: 0 1.5em;}
table{color:#FFF;background:#C00;border-collapse:collapse;width:647px;border:5px solid #900;}
thead{}
thead th{padding:1em 1em .5em;border-bottom:1px dotted #FFF;font-size:120%;text-align:left;}
thead tr{}
td{padding:.5em 1em;}
tfoot{}
tfoot td{padding-bottom:1.5em;}
tfoot tr{}
#middle{background-color:#900;}
</style>
<body BGCOLOR=""white"">
$tableFragment
</body>
"@ 
 
# Set up a regex search and match to look for any <td> tags in our body. These would only be present if the script above found disks below the threshold of free space.
# We use this regex matching method to determine whether or not we should send the email and report.
$regexsubject = $HTMLmessage
$regex = [regex] '(?im)<td>'
 
# if there was any row at all, send the email
if ($regex.IsMatch($regexsubject)) {
                        send-mailmessage -from $fromemail -to $users -subject "Disk Space Monitoring Report" -BodyAsHTML -body $HTMLmessage -priority High -smtpServer $server -UseSsl -Credential $cred 
}
 
# End of Script


# Another script for disk alert monitoring
$computers = $env:COMPUTERNAME

function Get-DiskDetails
{
    [CmdletBinding()]
    param (
        [string]$Computer = $env:COMPUTERNAME 
    )
    $cimSessionOptions = New-CimSessionOption -Protocol Default
    $query = "SELECT DeviceID, VolumeName, Size, FreeSpace FROM Win32_LogicalDisk WHERE DriveType = 3"
    $cimsession = New-CimSession -Name $Computer -ComputerName $Computer -SessionOption $cimSessionOptions
    Get-CimInstance -Query $query -CimSession $cimsession 
}

$newHtmlFragment = [System.Collections.ArrayList]::new()
foreach ($computer in $computers)
{
    $disks = Get-DiskDetails -Computer $computer 
    $diskinfo = @()
    foreach ($disk in $disks) {
        [int]$percentUsage = ((($disk.Size - $disk.FreeSpace)/1gb -as [int]) / ($disk.Size/1gb -as [int])) * 100  #(50/100).tostring("P")
        $bars = "<div style='background-color: DodgerBlue font-weight:bold; height: 28px; width: $percentUsage%'><span>$percentUsage%</span></div>" 
        $diskInfo += [PSCustomObject]@{
            Volume = $disk.DeviceID
            VolumeName = $disk.VolumeName
            TotalSize_GB = $disk.Size / 1gb -as [int]
            UsedSpace_GB = ($disk.Size - $disk.FreeSpace)/1gb -as [int]
            FreeSpace_GB = [System.Math]::Round($disk.FreeSpace/1gb)
            Usage = "usage {0}"  -f $bars #, $percentUsage%
        }
    }
    $htmlFragment = $diskInfo | ConvertTo-Html -Fragment
    $newHtmlFragment += $htmlFragment[0]
    $newHtmlFragment += "<tr><th class='ServerName' colspan='4'>$($computer.ToUpper())</th></tr>"
    $newHtmlFragment += $htmlFragment[2].Replace('<th>',"<th class='tableheader'>")

    $diskData =  $htmlFragment[3..($htmlFragment.count -2)]
    for ($i = 0; $i -lt $diskData.Count; $i++) {
        if ($($i % 2) -eq 0)
        {
            $newHtmlFragment += $diskData[$i].Replace('<td>',"<td class='td0'>")
        }
        else 
        {
            $newHtmlFragment += $diskData[$i].Replace('<td>',"<td class='td1'>")
        }
    }
    $newHtmlFragment += $htmlFragment[-1]
}
$newHtmlFragment = $newHtmlFragment.Replace("<td class='td0'>usage ", "<td class='usage'>")
$newHtmlFragment = $newHtmlFragment.Replace("<td class='td1'>usage ", "<td class='usage'>")
$newHtmlFragment = $newHtmlFragment.Replace('&lt;', '<')
$newHtmlFragment = $newHtmlFragment.Replace('&gt;', '>')
$newHtmlFragment = $newHtmlFragment.Replace('&#39', "'")

$html = @"
<html lang='en'>
    <head>
        <meta charset='UTF-8'>
        <meta http-equiv='X-UA-Compatible' content='IE=edge'>
        <meta name='viewport' content='width=device-width, initial-scale=1.0'>
        <title>Disk Usage Report</title>
        <style>
            body {
                font-family: Calibri, sans-serif, 'Gill Sans', 'Gill Sans MT', 'Trebuchet MS';
                background-color: whitesmoke;
            }
            .mainhead {
                margin: auto;
                width: 100%;
                text-align: center;
                font-size: xx-large;
                font-weight: bolder;
            }
            table {
                margin: 10px auto;
                width: 90%;
            }
            .ServerName {
                font-size: x-large;
                margin: 10px;
                text-align: left;
                padding: 10 0;
                color: DodgerBlue;
            }
            .tableheader {
                background-color: black;
                color: white;
                padding: 10px;
                text-align: left;
                /* font-size: large; */
                border-bottom-style: solid;
                border-bottom-color: darkgray;
            }
            td {
                background-color: white;
                border-bottom: 1px;
                border-bottom-style: solid;
                border-bottom-color: #404040;
            }

            .usage {
                background-color: SkyBlue ;
                width: 90%;
                text-align: center;
                color:  black;
            }

            span {
                color: black;
            }

            .td1 {
                background-color: #F0F0F0;
            }
        </style>
    </head>
    <body>
        <div class='mainhead'>
            <img style='vertical-align: middle;' src='data:image/jpeg;base64,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' alt='Disk Usage' width='50' height='50'/>
            Disk Utilization Report
        </div>
        <br>
        <div><i><b>Generated on: </b> $((Get-Date).DateTime)</i></div>
        $newHtmlFragment
    </body>
</html>
"@

$html > test.html

$from = 'no-reply@pdf.io'
$to = 's@gmail.com'
$subject = 'Weekly disk utilization report'
$body = $html
$smtpServer = 'email-smtp.com'
$smtpPort = '587'



$secpasswd = ConvertTo-SecureString "BNZk1" -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ("username", $secpasswd)

 
if ($sizemb -gt 200)
 

Send-MailMessage -From $from -to $to -Subject $Subject -Body $body -Body AsHtml -SmtpServer $smtpServer -Port $smtpPort -UseSsl -Credential $cred 


--------------------------------------------------------------------------------------------------------------------------------------------------------------
----------------------------------------------------------------------------------------------------------------------------------------------------------------
I have created a solutioning for Always on E-Mail Alerts when ever there is failover on primary db server:

1.Job is created to check the Primary server This job is related to Failover Alert Mail , This job checks which server is primary .Schedule is every 2min 
-----------------------------------------------------------------------------------------------------------------------------------------
if exists(select is_local, role_desc from sys.dm_hadr_availability_replica_states where role = 1 and role_desc = 'PRIMARY') begin
print 'This server [' + upper(@@servername) + '] is the primary.' 
EXEC msdb.dbo.sp_update_job @job_name='Failover Alert Mail',@enabled = 1; 
end
else
EXEC msdb.dbo.sp_update_job @job_name='Failover Alert Mail',@enabled = 0;
--------------------------------------------------------------------------------------------------------------------------------------------
2.Job Primary Failover Mail Alert Schedule Every 2 Min

This is our DBA Primary Failover Alert 
--------------------------------------------------------------------------------------------------------------------------------------------
if exists(select is_local, role_desc from sys.dm_hadr_availability_replica_states where role = 1 and role_desc = 'PRIMARY') begin
print 'This server [' + upper(@@servername) + '] is the primary.' end
else
BEGIN

EXEC msdb.dbo.sp_send_dbmail
  @recipients=N's.r@gmail.com;d.r@gmail.com',
  @body='Failover happened on Serverxxxx-x', 
  @subject ='Failover  happened on primary Serverxxxx-x',
  @profile_name ='sqldba'
  

END

---------------------------------------------------------------------------------------------------------------------------------------------------
