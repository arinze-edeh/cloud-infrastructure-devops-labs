# Tomcat Application Deployment on App Server 2



## Objective
- Install Apache Tomcat on App Server 2.

- Configure Tomcat to run on port 3000.

- Deploy Java ROOT.war application.

- Ensure application loads on base URL.

## Environment
- DATA CENTER: `Stratos DC.`

- SERVER: `App Server 2 (stapp02).`

- IP ADDRESS: `172.16.238.11.`

- APPLICATION: `Java-based ROOT.war.`

- PORT: `3000.`

## Tools Used
- Linux (CentOS Stream 9).

- Apache Tomcat 9.0.87.

- SCP (Secure Copy Protocol).

- Systemd (Systemctl).

## Deployment Steps

## Step 1: SSH to App Server
Accessed stapp02 via SSH from the Jump Host using user steve.

Authenticated with the password Am3ric@.

Screenshot:`ssh-login`
<img width="1034" height="585" alt="image" src="https://github.com/user-attachments/assets/212d3909-4d0d-4e62-ae7d-76fdefa9f604" />

## Step 2: Install Tomcat
Ran `sudo yum install tomcat -y to install` the application server and its dependencies.

Verified the installation of tomcat-1:9.0.87-7.el9.noarch.

Screenshots:`tomcat-installed`
<img width="1038" height="835" alt="image" src="https://github.com/user-attachments/assets/1d0248ec-574f-4320-ab13-b89a811bb069" />
<img width="1039" height="857" alt="image" src="https://github.com/user-attachments/assets/28a9e849-5e80-4478-8462-6d75c89550f5" />
<img width="1032" height="856" alt="image" src="https://github.com/user-attachments/assets/891e23f0-10bd-4467-8c8d-54567c3dc7a0" />
<img width="1031" height="853" alt="image" src="https://github.com/user-attachments/assets/e7579392-139f-4abf-ab30-0a662d5a6698" />
<img width="1033" height="855" alt="image" src="https://github.com/user-attachments/assets/d8b4fd52-03d3-4c66-bbfe-a6b9845a0815" />
<img width="1032" height="857" alt="image" src="https://github.com/user-attachments/assets/1b88b996-16a1-4a9a-ba88-a79dbed95cb8" />

## Step 3: Configure Tomcat Port
Modified the configuration file located at /etc/tomcat/server.xml.

Changed the default HTTP connector port from `8080` to `3000`.

Screenshot:`port-configuration`
<img width="1036" height="861" alt="image" src="https://github.com/user-attachments/assets/58446ba4-3dbf-4713-981c-9df468747060" />
<img width="1014" height="855" alt="image" src="https://github.com/user-attachments/assets/34597781-359e-4062-a2dc-05f7df19cbc3" />
<img width="1042" height="865" alt="image" src="https://github.com/user-attachments/assets/9dc42913-2ea2-4692-b829-25fae9cb191a" />

## Step 4: Start and Enable Tomcat
Used `systemctl` to initialize and manage the Tomcat service.

Ensured the service was configured to persist across system reboots.

Screenshot:`tomcat-running`

## Step 5: Transfer Application File
Successfully transferred `ROOT.war` from the `Jump Host` to `stapp02:/tmp/` using `SCP`.

Confirmed the 100% transfer of the 4529-byte file.

Screenshot:`war-transfer`

## Step 6: Deploy Application
Cleaned the deployment directory by removing default ROOT files.

Moved the artifact using `sudo mv /tmp/ROOT.war /var/lib/tomcat/webapps/`.

Updated file ownership with `sudo chown tomcat:tomcat /var/lib/tomcat/webapps/ROOT.war`.

Screenshot:`war-deployed`
<img width="1032" height="657" alt="image" src="https://github.com/user-attachments/assets/f082a3a4-15c5-403b-bc43-a85530c24ade" />

## Step 7: Restart Tomcat
Executed `sudo systemctl restart tomcat` to apply the new configuration and deploy the `.war` file.

screenshot:`tomcat-restarted`

## Step 8: Verify Application
Performed a final check using `curl http://stapp02:3000`.

Confirmed the application returned the message: "Welcome to xFusionCorp Industries!".

Screenshot:`app-working`
<img width="1033" height="770" alt="image" src="https://github.com/user-attachments/assets/fa86c5e2-6824-4776-b579-7061d3c9de1d" />

## Outcome

- Tomcat successfully installed on `CentOS Stream 9`.

- Service successfully reconfigured to listen on `port 3000`.

- ROOT application successfully deployed and accessible via the base URL.

- Deployment completed successfully for the Nautilus project.

## Tags
`devops` `linux-administration` `tomcat` `deployment` `centos` `automation`








<img width="1029" height="868" alt="image" src="https://github.com/user-attachments/assets/0a295bc6-3d05-4cc4-a5ba-7a8fc1eb7c8f" />
<img width="1029" height="866" alt="image" src="https://github.com/user-attachments/assets/24dc75a2-370a-461f-af35-09bd715fb157" />
<img width="1028" height="873" alt="image" src="https://github.com/user-attachments/assets/3ff96d01-7b89-4655-9ba1-977b02d09f0c" />
<img width="1029" height="860" alt="image" src="https://github.com/user-attachments/assets/d595a582-df93-4da9-b516-550841b9af91" />
<img width="1035" height="442" alt="image" src="https://github.com/user-attachments/assets/a680425e-d506-4333-8a4c-62f5d8528afb" />


