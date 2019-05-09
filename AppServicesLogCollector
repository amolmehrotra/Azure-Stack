$collectionScript = { 
    $node = $env:computername
    $nodename = $using:server

    Write-Host -ForegroundColor White "******************************************************************************************************"
    Write-Host -ForegroundColor White "                             Starting log collection on" $node "(" $nodename ")"
    Write-Host -ForegroundColor White "******************************************************************************************************"
    
    $DataCollectionDir = "c:\temp\CSS_AppServiceLogs"   
    $sharename = "CSS"

    #Logs to collect
    $CollectGuestLogsPath = "C:\WindowsAzure\GuestAgent*\"
    $CollectGuestLogsPath2 = "C:\WindowsAzure\Packages\"
    $httplogdirectory = "C:\DWASFiles\Log\"
    $ftpLogDirectory = "C:\inetpub\logs\LogFiles\"
    $WebsitesInstalldir = "C:\WebsitesInstall\"
    $windowsEventLogdir = "C:\Windows\System32\winevt\Logs"
    $webPILogdir = "C:\Program Files\IIS\Microsoft Web farm framework\roles\resources\antareslogs"
    $packagesdir = "C:\Packages"

    #Remove CSS_AppServiceLogs directory if already exists
    if (test-path -Path $DataCollectionDir -PathType Any){
        Remove-Item -Path $DataCollectionDir -Recurse -Force -Confirm:$false | Out-Null
    }

    #Create CSS_AppServiceLogs directory 
    New-Item -path $DataCollectionDir -ItemType Directory -Force | out-null
    
    #Create CSS share
    #Get-SmbShare -name $sharename | Remove-SmbShare -Confirm:$false -Force -ErrorAction SilentlyContinue | Out-Null
    New-SmbShare -Name $sharename -Path $DataCollectionDir -FullAccess "$env:UserDomain\$env:UserName" | Out-Null   

    #Starting CollectGuestLogs
    Write-host "Collect Guest Logs started on" $node -ForegroundColor Green
    if (test-path -Path $CollectGuestLogsPath -PathType Any){
       start-process "$CollectGuestLogsPath\CollectGuestLogs.exe" -Verb runAs -WorkingDirectory $CollectGuestLogsPath -wait ;
        Move-Item $CollectGuestLogsPath\*.zip -Destination $DataCollectionDir\ -Force 
        Move-Item $CollectGuestLogsPath\*.zip.json -Destination $DataCollectionDir\ -Force
        Dir $DataCollectionDir\*.zip.json | rename-item -newname {  $_.name  -replace ".zip.json",".json"  }  
    }
    elseif(test-path -Path $CollectGuestLogsPath2 -PathType Any){
        start-process "$CollectGuestLogsPath2\CollectGuestLogs.exe" -Verb runAs -WorkingDirectory $CollectGuestLogsPath2 -wait ;
        Move-Item $CollectGuestLogsPath2\*.zip -Destination $DataCollectionDir\ -Force 
        Move-Item $CollectGuestLogsPath2\*.zip.json -Destination $DataCollectionDir\ -Force
        Dir $DataCollectionDir\*.zip.json | rename-item -newname {  $_.name  -replace ".zip.json",".json"  }
    }
    
    Write-host "Collect Guest Logs completed on" $node -ForegroundColor Green

    if ($node -notlike "CN*"){
    #Collecting IIS logs
    Write-host "Collect IIS logs started on" $node -ForegroundColor Yellow
    Copy-Item $httplogdirectory\ -Recurse -Destination $DataCollectionDir\HTTPLogs -Force 
    Write-host "Collect IIS logs completed on" $node -ForegroundColor Yellow

    #Collecting WFF logs
    Write-host "Collect WFF logs started on" $node -ForegroundColor Green
    Copy-Item $webPILogdir\ -Recurse -Destination $DataCollectionDir\ 
    Write-host "Collect WFF logs completed on" $node -ForegroundColor Green  
    }

    #Collect FTP logs on Publisher servers
    if ($node -like "FTP*"){
    #Collecting FTP logs
    Write-host "Collect FTP logs started on" $node -ForegroundColor Yellow
    Copy-Item $ftpLogDirectory\ -Recurse -Destination $DataCollectionDir\FTPLogs -Force 
    Write-host "Collect FTP logs completed on" $node -ForegroundColor Yellow
    }


    #Collecting Event logs
    Write-host "Collect Event logs started on" $node -ForegroundColor Yellow
    Copy-Item $windowsEventLogdir\Microsoft-Windows-WebSites%4Administrative.evtx -Destination $DataCollectionDir\
    Copy-Item $windowsEventLogdir\Microsoft-Windows-WebSites%4Operational.evtx -Destination $DataCollectionDir\
    Copy-Item $windowsEventLogdir\Microsoft-Windows-WebSites%4Verbose.evtx -Destination $DataCollectionDir\ 
    Write-host "Collect Event logs completed on" $node -ForegroundColor Yellow

    #Collecting WebsitesInstall logs
    Write-host "Collect WebsitesInstall logs started on" $node -ForegroundColor Green
    Copy-Item $WebsitesInstalldir\ -Recurse -Destination $DataCollectionDir\ 
    Write-host "Collect WebsitesInstall logs completed on" $node -ForegroundColor Green     
    
    #Collecting C:\Packages
    Write-host "Collect C:\Packages started on" $node -ForegroundColor Yellow
    Copy-Item $packagesdir\ -Recurse -Destination $DataCollectionDir\ 
    Write-host "Collect C:\Packages completed on" $node -ForegroundColor Yellow       

    write-host "Compressing Files" -ForegroundColor Green    
    Compress-Archive -Path $DataCollectionDir\*  -DestinationPath $DataCollectionDir\$nodename.$env:computername.zip -Force 
    Get-ChildItem -Path $DataCollectionDir\ -Recurse | Where-Object {$_.Name -notlike "*$nodename.*"} | Remove-Item -Recurse -Force -Confirm:$false   
}

$cleanupScript = { 
    Write-Host -ForegroundColor Yellow "Starting log cleanup on" $env:computername
    $DataCollectionDir = "c:\temp\CSS_AppServiceLogs"    
    $sharename = "CSS"       

    #Remove CSS share
    Get-SmbShare -name $sharename | Remove-SmbShare -Confirm:$false -Force -ErrorAction SilentlyContinue | Out-Null

    #Remove CSS_AppServiceLogs directory if already exists
    if (test-path -Path $DataCollectionDir -PathType Any){
        Remove-Item -Path $DataCollectionDir -Recurse -Force -Confirm:$false | Out-Null
    }       

}

$logDirectory = "c:\temp\CSS"
if (test-path -Path $logDirectory -PathType Any){
        Remove-Item -Path $logDirectory -Recurse -Force -Confirm:$false | Out-Null
    } 
New-Item -path $logDirectory -ItemType Directory -Force | out-null

$workerCred = Get-Credential -Message "Enter credentials for Worker Admin"

Get-AppServiceServer | select Name, Status, Role, CpuPercentage, MemoryPercentage, ServerState, PlatformVersion | ft > $logDirectory"\Get-AppServiceServer.txt"
Get-AppServiceServer | ConvertTo-Json | Out-File $logDirectory"\AppServiceServer.json"
Get-AppServiceEvent | ConvertTo-Json | Out-File $logDirectory"\AppServiceEvent.json"

$roleServers = Get-AppServiceServer | Select -ExpandProperty Name

foreach ($server in $roleServers) {

    $serverRole = Get-AppServiceServer | Where-Object {$_.Name -eq $server} | Select -ExpandProperty Role

    if($serverRole -eq "WebWorker")
    {
	    $workerSession = New-PSSession -Credential $workerCred -ComputerName $server
	    Invoke-Command -Session $workerSession -ScriptBlock $collectionScript 
	    Copy-Item -FromSession $workerSession -Path c:\temp\CSS_AppServiceLogs\*.zip -Destination $logDirectory -Verbose
	    Invoke-Command -Session $workerSession -ScriptBlock $cleanupScript
	    Remove-PSSession $workerSession
    }
    else
    {
        Invoke-Command -ComputerName $server -ScriptBlock $collectionScript
        Copy-Item \\$server\CSS\* -Destination $logDirectory -Verbose    
        Invoke-Command -ComputerName $server -ScriptBlock $cleanupScript
    }
}
