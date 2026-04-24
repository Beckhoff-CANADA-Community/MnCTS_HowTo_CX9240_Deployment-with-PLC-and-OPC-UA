# Beckhoff MnCTS Setup Guide <br> CX9240 Linux ePC w/TwinCAT RT & OPC UA Server

A MnCTS guide (pronounced "Mints") stands for Minimum number of Clicks to Success.
It is designed to be a refreshing, user-interaction-optimised guide that takes you from ground zero to an operational state as smoothly and quickly as possible.

MnTCS guides will always lean towards getting equipment up and running first, over the most complete implementation.

Always refer to the appropriate relevant manuals for further details and complete instructions.

<br>

## Purpose: 
-  Setup a CX9240 with a new base image
-  install TwinCAT 3 RT for Linux (PLC control)
-  install OPC UA functionality and configure the firewall to allow operational access

### Test Fixturing Disclosure
This guide was developed using the following test Fixturing
<details>
  <Summary>Click to expand</Summary>
  
>-  TwinCAT IDE V3.1.4026.22
>-  TwinCAT RT for Linux V3.1.4026.23
>-  CX9240 ePC Image: V13 Build 306707 | Date: 10.04.2026
>-  CX9240-0215
>-  Hw: 1.1 | Date : 05/03/2025

</details>


### Requirements
- CX9240 ePC
- Internet Access
- MicroSD Card Reader

<br>

## Step 1: Download the latest image

Navigate to www.beckhoff.com/CX9240

  Under **Software and Tools**; download the Beckhoff Linux RT (CX9240) image file
  >[!TIP]
  >You can try the following download link directly \
  >[Download Beckhoff TwinCAT Runtime / Linux package](https://www.beckhoff.com/en-ca/support/download-finder/search-result/?download_group=829549649)
<br>
  
## Step 2: Deploy the Image
Download the latest version of [Rufus](https://rufus.ie/en/) \
The portable or standard versions will work\

 Unzip the Beckhoff-RT-Linux_cx9240-arm64 file
 
 Open Rufus, and with the microSD card inserted into a card reader
 1) Verify the destination
 2) Select the image you downloaded / unzipped
 3) Start the image process

 <img width="50%" alt="image" src="https://github.com/user-attachments/assets/bfb19808-1a9e-42ba-b71c-df72c7fff9d9" />

>[!NOTE]
 >You will be prompted that all data will be deleted, and additionally you may be warned about multiple partitions existing - this is normal for a Linux OS target.\
 Say **OK** to both warnings.
 <br>

 Once the imaging process is complete, Remove the microSD card, insert it into the CX9240 and power up the device.

<br>
 
 ## Step 3: Initial network address discovery
 Connect the CX9240 to a network with internet access.
 
 To help find the IP address, we can use the arp table but we first need to have hit the connection atleast once.

From an **elevated PowerShell**, run the following:
```shell
# ping all ports once in address span - replace the IP address with the first 3 octets of your network
1..254 | ForEach-Object { 
    Start-Process -WindowStyle Hidden ping -ArgumentList "-n 1 -w 100 192.168.1.$_" 
}
```
>[!IMPORTANT]
>The command above can take up to 1 minute to complete - be patient!
>Your PC may experience some lag while the commands complete, this is expected behaviour.
<br>

Now that we have hit all IP addresses in the pool with one ping, any that responded will be registed on the local arp table.
We can now filter the results with the following command:
```shell
Get-NetNeighbor -AddressFamily IPv4 | Where-Object LinkLayerAddress -like '00-01-05*' 
```

This should return a filtered list of results where the MAC address matches the Beckhoff pool. Remember the MAC address is also printed on the Beckhoff Product sticker on the side of the controller. Port X001 will be incremented by hex + 1 if you are using this port, compared to X000 which is listed directly on the sticker.

<img width="1350" height="439" alt="image" src="https://github.com/user-attachments/assets/60479a3d-ebe2-4dbf-839f-ea0b7f4e376e" />

<br><br>

## Step 4: SSH Connection

From **MobaXterm** (downloaded [here](https://mobaxterm.mobatek.net/download-home-edition.html)).
1)  Start a new Session
2)  Click SSH
3)  Enter the IP address you found from the previous step
4)  Check "Specify username"
5)  Enter "Administrator"
6)  Click Ok
<img width="848" height="527" alt="image" src="https://github.com/user-attachments/assets/cef264da-8afb-4bc6-ade3-02d4ef5a17af" />

>[!TIP]
>If you have used this IP address for SSH in the past, you may be prompted with the following since you have a previously saved thumb print (digital ID) for the previous PC. <BR>
> <img width="50%"  alt="image" src="https://github.com/user-attachments/assets/617ecfea-f525-4153-9fa5-7d7fcf1712e2" />

## Step 5: Package Manager Authentication
Next we need to add the credentials for your myBeckhoff Account in order to connect to the package manager.
Enter the following into the SSH terminal, you will be prompted for your username and password. 
Afterwards the script will update the packages from the server.

Copy the code below, then **Right Click** inside the MobaXterm window to paste the command into the SSH session.

```bash
sudo bash -c '
read -p "Enter your myBeckhoff email: " email
read -s -p "Enter your myBeckhoff password: " pass
echo
cat > /etc/apt/auth.conf.d/bhf.conf << EOF
machine deb.beckhoff.com
login $email
password $pass

machine deb-mirror.beckhoff.com
login $email
password $pass
EOF
chown root:root /etc/apt/auth.conf.d/bhf.conf && chmod 600 /etc/apt/auth.conf.d/bhf.conf &&
echo "Credentials file created. Running apt update..." &&
apt update
'
```



If successful you should see an output similar to the following:
<img width="70%"  alt="image" src="https://github.com/user-attachments/assets/902da398-9a6b-4aa4-9105-5539b62224d9" />

## Step 6: Install TwinCAT RT Linux
```bash
sudo apt install tc31-xar-um -y
```

End result should look similar to the image below. You should also now note that that "TC" Light on the front face of the CX9240 (previously off) is now Blue indicating that the system service is operating and in *Config Mode*. This light is located directly below the PWR Indicator light.

<img width="2168" height="1286" alt="image" src="https://github.com/user-attachments/assets/9b358730-dae4-4abc-b6ed-2a748a9abaac" />

## Step 7: OPC UA Install
To Install OPC UA we need to install the package from the package server. The second part of this command punches through the firewall with the correct default port of 4840 used by OPC UA so that we don't get caught up later.

```bash
# Install OPC UA package - auto yes to all prompts
sudo apt install tf6100-opc-ua-server -y

# Punches through firewall for 4840 permanently & Restart Firewall application
sudo bash -c 'cat > /etc/nftables.conf.d/50-opcua.conf << EOF
table inet filter {
    chain input {
        # accept TwinCAT OPC UA Server (TF6100)
        tcp dport 4840 accept
    }
}
EOF'
# Reloads the firewall application to read in the new rules
sudo systemctl reload nftables

# Checks for open port
sudo nft list ruleset | grep -E "4840|dport"
```

## Step 8: Initial Connection with TwinCAT IDE
Now that we have the TwinCAT RT installed and operational, we also have a working ADS router that we can connect to.

Start by opening the TwinCAT IDE, creating or opening a new TwinCAT project file (XML format).

Add a new route, and broadcast search for the target. 

>[!WARNING]
>Ensure you add the route as secure, TwinCAT RT for Linux does not support non-secure ADS connections by default

Default user: Administrator
Default Pass: 1

## Step 9: Licensing for PLC + OPC UA
Inside the TwinCAT IDE

Add a PLC project

Under "Properties for the PLC project, ensure that TMC file is transfered to the remote target. The OPC UA server uses this file as the lookup table for ADS symbols.

<img width="50%" alt="image" src="https://github.com/user-attachments/assets/67c6b299-f01a-4129-80f1-11a9ea42d9d4" />

<br>
<br>

Declare a PLC variable and expose it for access on OPC UA
```bash
	{attribute 'OPC.UA.DA' := '1' }
	iTemp: INT;
```

Add some PLC code 

```bash
iTemp := iTemp + 1;
```

Under the System -> License\
Ensure that TF6100 is listed under licenses, if not manually add it to the system by clicking the check box on the Manage tab.

<img width="50%" alt="image" src="https://github.com/user-attachments/assets/050e0769-765a-4d7f-be9a-f540be7c2ca6" />

<br>
<br>

Activate the project.

TwinCAT should go into **Run Mode**

If you are prompted to enable the boot project, set the check box to true. The PLC project needs to be in a running state for the OPC UA Server to read the Tag values over ADS. It's a common oversight to have the Runtime in RUN mode, but forget to set the PLC project to Autoboot.

Now that the PLC is running, and the TMC file containing the OPC UA exposure pragma is on the controller, it's time to finish the setup of the OPC UA server.

## Step 10: Finalize OPC UA Server Setup
Follow the standard OPC UA setup from this point on.

