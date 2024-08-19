**How to Host a React Application with ASP.NET Core Embedded Sample**

1. **Move the Publish Folder to the Linux Machine:**
   Use the following command to securely copy the publish folder to your Linux machine:
   ```bash
   scp "location to your publish folder" username@ipaddress:/home/username
   scp "C:\Users\user\Downloads\publish1.zip" syncfusion@20.3.131.204:/home/syncfusion
   ```

2. **Create a New Directory for the App:**
   We will host the app from the `/var/www/` directory on Ubuntu. Create a new folder called "nginxdotnet" using the following command:
   ```bash
   sudo mkdir /var/www/nginxdotnet
   ```

3. **Copy the Published Files:**
   Navigate to the app's "publish" folder and run the following command to copy the embedded sample's published files into the new directory:
   ```bash
   sudo cp -r . /var/www/nginxdotnet
   ```

4. **Set Permissions:**
   After copying the files, set the permissions to ensure all users have read, write, and execute access:
   ```bash
   sudo chmod -R 777 /var/www/nginxdotnet
   ```

5. **Create a Service File:**
   Create a new service file to manage the application:
   ```bash
   sudo nano /etc/systemd/system/embed-nginxdotnet.service
   ```

6. **Add the Following Configuration to the Service File:**
   ```ini
   [Unit]
   Description=Nginx .NET App

   [Service]
   WorkingDirectory=/var/www/nginxdotnet
   ExecStart=/var/www/bold-services/dotnet/dotnet /var/www/nginxdotnet/BoldBI.Embed.Sample.dll --urls=http://localhost:5000
   Restart=always
   # Restart the service after 10 seconds if the .NET service crashes:
   RestartSec=10
   KillSignal=SIGINT
   SyslogIdentifier=nginxdotnet
   User=www-data
   Environment=ASPNETCORE_ENVIRONMENT=Production

   [Install]
   WantedBy=multi-user.target
   ```

7. **Enable and Start the Service:**
   Now, enable the service and then start it using the following commands:
   ```bash
   sudo systemctl enable embed-nginxdotnet.service
   sudo systemctl start embed-nginxdotnet.service
   sudo systemctl status embed-nginxdotnet.service
   ```

8. **Configure Nginx for the .NET App:**
   Open the Nginx site configuration file:
   ```bash
   sudo nano /etc/nginx/sites-available/nginxdotnet
   ```

   Add the following configuration:
   ```nginx
   server {
       listen 80;
       server_name hubsync.boldbi.com;
       return 301 https://hubsync.boldbi.com$request_uri;
   }

   server {
       server_name hubsync.boldbi.com;
       listen 443 ssl;
       ssl_certificate /etc/nginx/sites-available/Certificate.pem;
       ssl_certificate_key /etc/nginx/sites-available/Certificate.key;

       location / {
           proxy_pass         http://localhost:5000/;
           proxy_http_version 1.1;
           proxy_set_header   Upgrade $http_upgrade;
           proxy_set_header   Connection keep-alive;
           proxy_set_header   Host $host;
           proxy_cache_bypass $http_upgrade;
           proxy_set_header   X-Forwarded-For $proxy_add_x_forwarded_for;
           proxy_set_header   X-Forwarded-Proto $scheme;
       }
   }
   ```
9. **Move the SSL Certificate**
   After writing the Nginx configuration file, move the certificate file and the PEM file to /etc/nginx/sites-available. Use the following command to transfer the files:
   
   ```bash
   scp "C:\Users\PriyaSunilKumar\Downloads\Certificate.key" syncfusion@20.3.131.204:/home/syncfusion
   ```
   
   Then, use the following command to move the files:
   
   ```bash
   mv Certificate.key Certificate.pem /etc/nginx/sites-available/
   ```
10. **Create the Symlink:**
   Create a symbolic link to enable the site:
   ```bash
   sudo ln -s /etc/nginx/sites-available/nginxdotnet /etc/nginx/sites-enabled/nginxdotnet
   ```

   Validate the syntax of the Nginx configuration with the following command:
   ```bash
   sudo nginx -t
   ```

   You should receive the following messages:
   ```
   nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
   nginx: configuration file /etc/nginx/nginx.conf test is successful
   ```

   Finally, reload Nginx to apply the changes:
   ```bash
   sudo nginx -s reload
   ```
