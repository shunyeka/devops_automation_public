# Here we do the installation process of Deep Security Agent into EC2 using SSM Document and State Manager

1. Create an AWS SSM Document
    - Here need to create AWS SSM Document
    - Example 1:  
    ---
        schemaVersion: "2.2"
        description: "aws:runShellScript"
        parameters:
          commands:
            type: "String"
            description: "This document installs DeepSecurity agent and makes sure that it is up and running."
            default: "DeepSecurity Automation"
        mainSteps:
        - action: "aws:runShellScript"
          name: "runShellScript"
          inputs:
            timeoutSeconds: "600"
            runCommand:
            - "#!/bin/bash"
            - "CURLOUT=$(eval curl https://raw.githubusercontent.com/shunyeka/devops_automation_public/main/dsa_linux.sh -o /tmp/PlatformDetection --silent --tlsv1.2;)"
            - "source /tmp/PlatformDetection"
            - "platform_detect"
            - "echo Downloading agent package..."
            - "curl -H 'Agent-Version-Control: on' https://app.deepsecurity.trendmicro.com:443/software/agent/${runningPlatform}${majorVersion}/${archType}/agent.rpm?tenantID=87198 -o /tmp/agent.rpm --silent --tlsv1.2"
            - "echo Installing agent package..."
            - "sudo rpm -ihv /tmp/agent.rpm"
            - "echo Install the agent package successfully"
            - "sleep 15"
            - "sudo /opt/ds_agent/dsa_control -r"
            - "sudo /opt/ds_agent/dsa_control -a <dsm://agents.domain_name>/ 'tenantID:B3F-XXXXXXXXX' 'token:367-XXXXXX' 'policyid:X'"
            
            
    - Example 2:  
    ---
        schemaVersion: '2.2'
        description: cross-platform 
        parameters:
          commands:
            type: "String"
            description: "(Required) The commands to run or the path to an existing script on the instance."
            default: "echo Hello World"
        mainSteps:
        - action: aws:runPowerShellScript
          name: PatchWindows
          precondition:
            StringEquals:
            - platformType
            - Windows
          inputs:
            runCommand:
            - "$managerUrl=\"https://app.deepsecurity.trendmicro.com:443/\""
            - "$env:LogPath = \"$env:appdata\\Trend Micro\\Deep Security Agent\\installer\""
            - "New-Item -path $env:LogPath -type directory 2> err"
            - "Start-Transcript -path \"$env:LogPath\\dsa_deploy.log\" -append"
            - "echo \"$(Get-Date -format T) - DSA download started\""
            - "$sourceUrl=-join($managerUrl, \"software/agent/Windows/x86_64/agent.msi\")"
            - "echo \"$(Get-Date -format T) - Download Deep Security Agent Package\" $sourceUrl"
            - "$WebClient = New-Object System.Net.WebClient"
            - "# Add agent version control info"
            - "$WebClient.Headers.Add(\"Agent-Version-Control\", \"on\")"
            - "$WebClient.QueryString.Add(\"tenantID\", \"56938\")"
            - "$WebClient.QueryString.Add(\"windowsVersion\", (Get-CimInstance Win32_OperatingSystem).Version)"
            - "$WebClient.QueryString.Add(\"windowsProductType\", (Get-CimInstance Win32_OperatingSystem).ProductType)"
            - "$WebClient.DownloadFile($sourceUrl,  \"$env:temp\\agent.msi\")"
            - "echo \"$(Get-Date -format T) - Downloaded File Size:\" (Get-Item \"$env:temp\\agent.msi\").length"
            - "echo \"$(Get-Date -format T) - DSA install started\""
            - "echo \"$(Get-Date -format T) - Installer Exit Code:\" (Start-Process -FilePath msiexec -ArgumentList \"/i $env:temp\\agent.msi /qn ADDLOCAL=ALL /l*v `\"$env:LogPath\\dsa_install.log`\"\" -Wait -PassThru).ExitCode"
            - "echo \"$(Get-Date -format T) - DSA activation started\""
            - "Start-Sleep -s 50"
            - "& echo '...waiting for ready the connection'"
            - "& $Env:ProgramFiles\"\\Trend Micro\\Deep Security Agent\\dsa_control\" -r"
            - "& $Env:ProgramFiles'\\Trend Micro\\Deep Security Agent\\dsa_control' -a <dsm://agents.domain_name>/ \"tenantID:0FA5-XXXXXX\" \"token:4FD7E8A2-XXXXXXXX\" \"policyid:X\""
            - "Stop-Transcript"
            - "echo \"$(Get-Date -format T) - DSA Deployment Finished\""
        - action: aws:runShellScript
          name: PatchLinux
          precondition:
            StringEquals:
            - platformType
            - Linux
          inputs:
            runCommand:
            - "#!/bin/bash"
            - "CURLOUT=$(eval curl https://raw.githubusercontent.com/shunyeka/devops_automation_public/main/dsa_linux.sh -o /tmp/PlatformDetection --silent --tlsv1.2;)"
            - "source /tmp/PlatformDetection"
            - "platform_detect"
            - "echo Downloading agent package..."
            - "echo RunningPlatform: ${runningPlatform}"
            - "echo MajorVersion: ${majorVersion}"
            - "echo ArchType: ${archType}"
            - "curl -H 'Agent-Version-Control: on' https://app.deepsecurity.trendmicro.com:443/software/agent/${runningPlatform}${majorVersion}/${archType}/agent.rpm?tenantID=87198 -o /tmp/agent.rpm --silent --tlsv1.2"
            - "echo Installing agent package..."
            - "sudo rpm -ihv /tmp/agent.rpm"
            - "echo Install the agent package successfully"
            - "sleep 15"
            - "sudo /opt/ds_agent/dsa_control -r"
            - "sudo /opt/ds_agent/dsa_control -a <dsm://agents.domain_name>/ 'tenantID:B3F-XXXXXXXXX' 'token:367-XXXXXX' 'policyid:X'"
        
                
                
2. Create State Manager
    - Inside ` AWS System Manager `, Click the` State Manager ` tab then
    - Click the ` Create Association ` button placed on the top-right corner
    - On this Create Association page need to configure some fields
    
      Provide association details
      - name(optional): install_deep_security_agent_association.
      
      Document
      - Here in the search field select Owner: Owned by me, then select the particular document
      - Document version: `Default at runtime`
      
      Targets
      - Target selection: `Choose instances with tags`
      - Provide relevant tag.
      
      Keep the rest of the fields as a default 
      
    - Click the ` Create Association ` button.  
      
3. Create launch template
    - In `AWS EC2` Service, click the ` Launch templates ` tab
    - Click the ` create launch template ` button
    - In This create launch template page need to configure some fields
    
      Launch template name and description
      - Launch template name(required): `install_deep_security_agent_template`
      - Template version description: `v1.0.0`
      - Select this if you intend to use this template with EC2 Auto Scaling
      
      Launch template contents
      - Amazon machine image (AMI): `Amazon Linux 2 AMI`
      - Instance type: `t2.micro`
      - Key pair (login): `install_deep_security_agent_keypair`
      
      Advance Details
      - IAM instance profile: `add proper IAM role which has AWS SSM access so that all the SSM related work done properly `

      Keep the rest of the fields as a default 
      
      - Click the ` create launch template ` button.
      - Then click the `Scalingbutton, then click the ` Create Auto Scalling Group ` Option.  
     
4. Create an Auto Scaling group
    - In this page have some steps to create Auto Scaling 
    
     Step1: Choose launch template or configuration
     - Auto Scaling group name: `install_deep_security_agent_autoscallinggroup`
     - Keep rest of the fields as a default
     - Click `Next`  
     
     Step2: Configure settings
     - Instance purchase options: `keep default`
     - Network: Subnets `ap-south-1a` 
     - Click `Next`
     
     Step3: Configure advanced options
     - Keep all as default and click ` Next `
     
     Step4: Group size(optional)
     - Desired capacity: `1`
     - Minimum capacity: `1`
     - Maximum capacity: '2'
     - Keep rest of the thing as a default
     - click `Next`
     
     Step5: Add notifications
     - Keep all as default and click ` Next `
    
     Step6: Add tags
     - Keep all as default and click ` Next `
     
     Step7: Review
     - Click `Create Auto Scaling group`, if all of the configurations are fine the auto-scaling group will be created
     
    - Go to the `Instances` section and check, if it new instance is created or not.
    - After few minutes verify the State Manager, is it deep security is installed or not   
    

5. Verify from Trend Micro Deep Security
    - From here Inside ` Computers ` Menu Check instance id is available or not.

The deep Security agent installation process is done, We appreciate your feedback on any enhancement that needs to be done.

Thank you
