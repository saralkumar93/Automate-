STEP 1: Prepare Your Environment

1.1. Ensure PowerShell is available:

Windows comes with PowerShell pre-installed.

You can also install PowerShell 7 from https://github.com/PowerShell/PowerShell.


1.2. Create a folder to store the script and reports:

C:\RubrikAutomation\


---

STEP 2: Collect Required Information

You will need:

Item	Description

Rubrik IP or FQDN	e.g., https://rubrik01.company.local
Username & Password	Rubrik account with report access
Report ID	From Rubrik UI or API
SMTP Server (optional)	If you want to email the report


How to get the Report ID:

1. Log in to the Rubrik web interface.


2. Go to Reports.


3. Click the VM report you want.


4. Copy the last part of the URL — that’s your report ID.




---

STEP 3: Write the PowerShell Script

3.1. Open Notepad or VS Code and paste the following:

# --- Configuration ---
$rubrikIP = "https://rubrik01.company.local"
$username = "your-username"
$password = "your-password"
$reportID = "abcd-1234-5678-vmreportid"
$date = Get-Date -Format "yyyy-MM-dd"
$outputPath = "C:\RubrikAutomation\rubrik_vm_report_$date.pdf"

# Email Settings
$sendEmail = $true
$smtpServer = "smtp.company.com"
$fromEmail = "reportbot@company.com"
$toEmail = "admin@company.com"

# --- Trust all SSL certificates (for self-signed certs) ---
add-type @"
using System.Net;
using System.Security.Cryptography.X509Certificates;
public class TrustAllCertsPolicy : ICertificatePolicy {
    public bool CheckValidationResult(ServicePoint srvPoint, X509Certificate certificate, WebRequest request, int certificateProblem) {
        return true;
    }
}
"@
[System.Net.ServicePointManager]::CertificatePolicy = New-Object TrustAllCertsPolicy

# --- Auth Header ---
$pair = "$username:$password"
$bytes = [System.Text.Encoding]::ASCII.GetBytes($pair)
$base64 = [Convert]::ToBase64String($bytes)
$headers = @{
    Authorization = "Basic $base64"
}

# --- Download the Report ---
$reportUrl = "$rubrikIP/report/$reportID/download"

try {
    Invoke-RestMethod -Uri $reportUrl -Headers $headers -Method Get -OutFile $outputPath
    Write-Output "Report saved to $outputPath"
} catch {
    Write-Error "Failed to download report: $_"
    exit
}

# --- Send Email (Optional) ---
if ($sendEmail) {
    $msg = New-Object System.Net.Mail.MailMessage
    $msg.From = $fromEmail
    $msg.To.Add($toEmail)
    $msg.Subject = "Rubrik Daily VM Report - $date"
    $msg.Body = "Please find the attached Rubrik VM report."
    $msg.Attachments.Add($outputPath)

    $smtp = New-Object Net.Mail.SmtpClient($smtpServer)
    try {
        $smtp.Send($msg)
        Write-Output "Email sent to $toEmail"
    } catch {
        Write-Error "Failed to send email: $_"
    }
}

3.2. Save as:

C:\RubrikAutomation\Get-RubrikVMReport.ps1


---

STEP 4: Test the Script

1. Open PowerShell as Administrator.


2. Run:



Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
cd C:\RubrikAutomation
.\Get-RubrikVMReport.ps1

You should:

See a PDF file created in C:\RubrikAutomation\.

Receive the email (if enabled).



---

STEP 5: Automate with Task Scheduler

5.1. Open Task Scheduler

Press Win + R, type taskschd.msc, press Enter.


5.2. Create a Task

Click Create Task

General Tab: Name it Rubrik VM Report Automation

Triggers Tab: Add a new trigger to run Daily at 7:00 AM

Actions Tab:

Action: Start a program

Program/script: powershell.exe

Add arguments:

-ExecutionPolicy Bypass -File "C:\RubrikAutomation\Get-RubrikVMReport.ps1"


Conditions Tab: (Optional) Uncheck “Start only if on AC power”

Click OK



---

Done!

You now have:

A working script that downloads the VM report daily.

A scheduled task that runs it automatically.

Optional email delivery.

