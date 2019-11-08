+++ 
draft = false
date = 2019-11-08T03:15:31Z
title = "Coding in the Cloud - VSCode on OCI Free Tier"
+++

Oracle Cloud recently started offering a [free tier](https://www.oracle.com/cloud/free/) that gives you two micro VMs each with 1GB Ram and 1/8 OCPU (Looks like this results in 2 cores at ~2Ghz). I thought this would be a fun chance to checkout [Code Server](https://github.com/cdr/code-server) on my new iPad Pro for some mobile development. Code Server allows you to run Visual Studio Code through a web browser thus allowing you to run it from anywhere.

## Getting The VM
After Oracle Announced a free tier, people swarmed to take advantage and thus capacity was quickly taken over. This paused my original intent down this path but recently I tried again and was able to snap a few VMs. If you get an error saying "No Host Capacity", you'll need to just try again another day.

The first trick is finding where you can get your free instances. Find this info by:

1. Login to OCI Console
2. Make sure you have the home region selected. (Top right)
3. Click on the profile icon (Very top right corner) and select Tenancy:YourTenancy option
4. Expand Compute and look for the line labeled "VM.Standard.E2.1.Micro"
5. Note what Availability Domain (AD) is listed

Create your new VM as follows:

1. Log in to the OCI Console
2. Verify Region
3. Menu (Top Left) -> Compute -> Instances
4. Click Create Instance
5. Fill out the form:
   - Name
   - Availability Domain (Pulled from the service limits above)
   - Shape: "VM.Standard.E2.1.Micro" (Note the Always Free label)
   - Assign a public IP
   - Add SSH Key
6. Wait for it to provision
7. Look at details page and note the Public IP
7. Log in with something like ```ssh -i YOUR_SSH_KEY opc@PUBLIC_IP_OF_VM```

## Install Code Server
Now that you have a VM and can log in, you can pull and configure Code Server. First head over to [Code Server Releases](https://github.com/cdr/code-server/releases) to find the latest and greatest. For my case I used 2.1665-vsc1.39.2.

1. Basic install with the following
{{< highlight bash >}}
VERSION=2.1665-vsc1.39.2
wget https://github.com/cdr/code-server/releases/download/${VERSION}/code-server${VERISON}-linux-x86_64.tar.gz
tar -xvf code-server${VERSION}-linux-x86_64.tar.gz
ln -s code-server${VERSION}-linux-x86_64 code-server
{{< / highlight >}}
2. Configure Linus Firewall
{{< highlight bash >}}
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload
{{< / highlight >}}
3. Create Service File as /usr/lib/systemd/system/code-server.service
{{< highlight bash >}}
[Unit]
Description=Code Server IDE
After=network.target

[Service]
Type=simple
User=opc
EnvironmentFile=$HOME/.bash_profile
WorkingDirectory=$HOME/code
Restart=on-failure
RestartSec=10
ExecStart=$HOME/code-server/code-server --cert $HOME/code-server/server.crt --cert-key $HOME/code-server/server.key $HOME/code

[Install]
WantedBy=multi-user.target
{{< / highlight >}}
4. Update your .bash_profile to specify a password for Code Server
{{< highlight bash >}}
PASSWORD=$PASSWORD
{{< / highlight >}}
4. Update your .bash_profile to fix SSL Issues downloading extentions
{{< highlight bash >}}
NODE_TLS_REJECT_UNAUTHORIZED=0
NODE_EXTRA_CA_CERTS=/etc/pki/tls/certs/ca-bundle.crt
export NODE_EXTRA_CA_CERTS
export NODE_TLS_REJECT_UNAUTHORIZED
{{< / highlight >}}
5. Install Service
{{< highlight bash >}}
sudo systemctl start code-server
sudo systemctl enable code-server
{{< / highlight >}}

## Configure OCI Virtual Cloud Network (VCN)
In order to access the newly running code server, the port needs to be expoed in OCI's VCN.

1. Log in to the OCI Console
2. Verify Region
3. Menu (Top Left) -> Compute -> Instances
4. Under Instance Information click the VirtualCloudNetwork-YYYYMMDD-HHMI link for the auto generated VCN
5. Click the Public Subnet link
6. Click the Default Security List
7. Add Ingress Rule
   - Source CIDR: 0.0.0.0/0
   - Destination Port Range: 8080
8. Save it

## Log in and Enjoy!
If everything went smoothly you should now have Code Server running on a few OCI instance and available at http://YOUR_PUBLIC_IP:8080. It isn't perfect and it isn't a full blown Eclipse or Intelij but for things like writing some code and maintaining a blog it works quite well so far. 


