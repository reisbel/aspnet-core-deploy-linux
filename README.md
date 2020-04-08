# aspnet-core-deploy-linux

Publishing an ASP.NET Core website to a Linux VM instance

## Intro

.NET Core is free, open-source, cross-platform and runs everywhere. Here are the step I followed to deploy an ASP.NET Core website to a Linux VM Instance.

## Steps

### Create the Linux VM Instance

For this example, I will create a Google Cloud VM Instance. If you want to know more about Google Cloud Platform see the following [link](https://cloud.google.com/ai-platform/deep-learning-vm/docs/quickstart-cli), or you could use any other preferred cloud provider.

Set variables

```bash
INSTANCE_NAME=aspnet-core-debian9
ZONE=us-central1-c
INSTANCE_TYPE=n1-standard-1
```

Create VM Instance

```bash
gcloud compute instances create $INSTANCE_NAME \
        --zone=$ZONE \
        --image-family=debian-9 \
        --image-project=debian-cloud \
        --maintenance-policy=TERMINATE \
        --machine-type=$INSTANCE_TYPE \
        --boot-disk-size=10GB \
        --tags=http-server
```

Connect to the VM Instance via ssh. In these steps, I'm connecting to the VM using the gcloud CLI, for other environments an ssh client will be needed.

```bash
gcloud compute ssh $INSTANCE_NAME --zone $ZONE
```

### Install the .NET Core runtime on the server

* Visit the [Download .NET Core page](https://dotnet.microsoft.com/download/dotnet-core).
* Select the latest non-preview .NET Core version.
* Download the latest non-preview runtime in the table under Run apps - Runtime.
* Select the Linux Package manager instructions link and follow the instructions.

For this example, I followed the instructions provided for [Debian 9](https://docs.microsoft.com/en-us/dotnet/core/install/linux-package-manager-debian9)

Register the Microsoft key, register the product repository and install required dependencies.

```bash
wget -O- https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.asc.gpg
sudo mv microsoft.asc.gpg /etc/apt/trusted.gpg.d/
wget https://packages.microsoft.com/config/debian/9/prod.list
sudo mv prod.list /etc/apt/sources.list.d/microsoft-prod.list
sudo chown root:root /etc/apt/trusted.gpg.d/microsoft.asc.gpg
sudo chown root:root /etc/apt/sources.list.d/microsoft-prod.list
```

Install the ASP.NET Core runtime

```bash
sudo apt-get update
sudo apt-get install apt-transport-https
sudo apt-get update
sudo apt-get install aspnetcore-runtime-3.1
```

### Make an ASP.NET Core website

Create a basic ASP.NET Core website, that's fine for this demo.

```bash
dotnet new web
```

Now I can restore and run my web app. It will startup on localhost:5000.

```bash
dotnet restore
dotnet run
```

```output
info: Microsoft.Hosting.Lifetime[0]
      Now listening on: http://localhost:5000
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: /Users/reisbel/rep/github-workspace/aspnet-core-deploy-linux
```

Publish the application into a directory that later can be moved to the server:

```bash
dotnet publish --configuration Release
```

### Configure Kestrel

Every application runs its instance of a web server called Kestrel, which by default will listen on port 5000. To run our website we must create a systemd service.

```bash
sudo nano /etc/systemd/system/aspnet-core-deploy-linux
```

```config
[Unit]
Description=ASP.NET Core Website
After=network.target

[Service]
WorkingDirectory=/var/www/app
ExecStart=/usr/bin/dotnet /var/www/app/aspnet-core-deploy-linux.dll
Restart=always
RestartSec=10
SyslogIdentifier=aspnet-core-deploy-linux
User=www-data
Environment=ASPNETCORE_ENVIRONMENT=Production

[Install]
WantedBy=multi-user.target
```

Start the service and enable the startup of the service at boot

```bash
sudo systemctl start aspnet-core-deploy-linux
sudo systemctl enable aspnet-core-deploy-linux
```

If everything works as expected you should now be able to open the site on port 5000.

```bash
curl http://localhost:5000
```

### Install and configure Apache

Since Kestrel is not a fully-featured web server, using a reverse proxy server might be a good choice, such as Internet Information Services (IIS), Nginx, or Apache. A reverse proxy server receives HTTP requests from the network and forwards them to Kestrel. For this example I've selected Apache.

Install Apache

```bash
sudo apt install apache2
```

Enable required modules

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod headers
```

Create the Apache site configuration file under the path

```bash
sudo nano /etc/apache2/sites-available/aspnet-core-deploy-linux.conf
```

```config
<VirtualHost *:80>
    ProxyPreserveHost On
    ProxyPass / http://localhost:5000/
    ProxyPassReverse / http://localhost:5000/
</VirtualHost>
````

Disable the default apache website

```bash
sudo a2dissite default
```

And finally, enable the new apache site configuration.

```bash
sudo a2ensite aspnet-core-deploy-linux
```

You should be able to query the external address and see the website

```bash
export publiIP=$(gcloud compute instances describe $INSTANCE_NAME --zone=$ZONE --format="value(networkInterfaces[0].accessConfigs.natIP)")
curl $publiIP
```

Output

```bash
Hello World!
```
