# Installing DShield SIEM on a Raspberry Pi 5 - 8 GB RAM
Authored by linkedin.com/in/dylan-r-cyber | github.com/amelete11235/homelab

---
# Hardware and OS Installation
## Hardware Requirements

- You will need a separate DSHield honeypot, of course! 

***I tested this installation on a Raspberry Pi 5, with 8 GB of RAM.***
- I did not test this on other generations of Raspberry Pi.
- A Raspberry Pi with less than 8 GB of RAM will not have enough memory to run all of the Docker containers.
- Raspberry Pi's use an SD card as a boot drive. I would recommend a high-speed card with at least 64 gb.
- `github.com/bruneaug`'s guide states a minimum of a 300 GB partition assigned to `/var/lib/docker`
	- I tested this installation with an extra 1 TB external HDD connected via USB. For mid to long-term use, I recommend that you use your Micro SD for your OS installation, but not your log storage. Use an external hard drive.
- This installation was tested with an active cooler installed: https://www.raspberrypi.com/products/active-cooler/
- The Raspberry Pi was tested with a direct ethernet connection to the router.
---
## OS Requirements & Installation
###### Install the latest version of Raspberry Pi OS Lite ***(64-bit)*** using Raspberry Pi Imager:

1. Download the imager from https://www.raspberrypi.com/software/ 
![](a/Pasted%20image%2020240717154645.png)

2. After installing the imager in your preferred OS and inserting your Micro SD card, run it and select your device type, **Raspberry Pi OS Lite (64-bit)**, and the correct SD card to format for your boot drive **(don't overwrite something else by mistake!)**
![](a/Pasted%20image%2020240717162048.png)
![](a/Pasted%20image%2020240717155943.png)
![](a/Pasted%20image%2020240717161656.png)
![](a/Pasted%20image%2020240717162237.png)

3. Click *NEXT* and then *Edit Settings*
![](a/Pasted%20image%2020240717162343.png)
![](a/Pasted%20image%2020240717162924.png)

4. Set your own information in the fields configured from the below screenshots. 
![](a/Pasted%20image%2020240717162853.png)

5. Make sure that you enable SSH in the *SERVICES* tab using password authentication. You will be able to authenticate to the SSH server using the username and password that you set previously, in the *GENERAL* tab. When finished, click *SAVE*
![](a/Pasted%20image%2020240717163703.png)

6. Apply OS customization settings.
![](a/Pasted%20image%2020240717164148.png) 

7. You will be notified that the drive is going to be overwritten. **If you chose the wrong drive and you continue, you will lose data.

8. When finished, remove the SD card from your computer, insert it into your Raspberry Pi, and make sure that the Pi is powered and plugged in to your router.
![](a/Pasted%20image%2020240717164352.png)

9. Look in your router's admin console (usually by going to http://192.168.0.1 in your web browser) for the new device and it's IP address. Use this IP address and the username/password you configured in step 4 to ssh to the Pi:
``` bash
ssh dshieldsiem@192.168.0.2
```

10. In the Pi, run the following commandshieldsiem:
``` bash
sudo apt update && sudo apt upgrade -y
```

11. If you are getting locale errors during upgrades:
``` bash
sudo dpkg-reconfigure locales                          # Select en_US.UTF-8 UTF-8
sudo locale-gen en_US.UTF-8
```

---
# Creating your 300+ GB partition assigned to `/var/lib/docker`
*Commands modified/copied from: [bruneaug/Build a Docker Partition](https://github.com/bruneaug/DShield-SIEM/blob/main/AddOn/Build_a_Docker_Partition.md)

1. Switch to root
``` bash
dshieldsiem@dshieldsiem:~ $ sudo su -
```

2. Find your extra storage device. For me, it's the one with `931.5G` named `sda`
``` bash 
root@dshieldsiem:~# lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
sda           8:0    0 931.5G  0 disk 
└─sda1        8:1    0 931.5G  0 part 
```

3. You will need to delete any existing partitions. **Don't delete a partition that you don't want to lose!** Create a new partition using all of the default values. Use your full disk, unless you have special plans.
``` bash
root@dshieldsiem:~# cfdisk /dev/sda
```

![](a/Pasted%20image%2020240717175830.png)

![](a/Pasted%20image%2020240717180444.png)

![](a/Pasted%20image%2020240717180532.png)

![](a/Pasted%20image%2020240717180718.png)

4. Install `lvm2`. Initialize the physical volume for use with LVM, then create a new volume group and a logical volume.
``` bash
root@dshieldsiem:~# pvcreate /dev/sda1
  Physical volume "/dev/sda1" successfully created.

root@dshieldsiem:~# vgcreate vg01 /dev/sda1
  Volume group "vg01" successfully created

root@dshieldsiem:~# vgdisplay vg01 
  --- Volume group ---
  VG Name               vg01
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable

root@dshieldsiem:~# lvcreate -n lv01 --size 931G vg01 
  Logical volume "lv01" created.

root@dshieldsiem:~# lvdisplay /dev/vg01/lv01 
  --- Logical volume ---
  LV Path                /dev/vg01/lv01
  LV Name                lv01
  VG Name                vg01
```

5. Create an xfs
``` bash
root@dshieldsiem:~# sudo apt install xfsprogs 

root@dshieldsiem:~# sudo mkfs.xfs /dev/vg01/lv01 

root@dshieldsiem:~# echo "/dev/vg01/lv01 /var/lib/docker xfs defaults,noatime,nosuid 0 0" >> /etc/fstab 

root@dshieldsiem:~# mkdir -p /var/lib/docker

root@dshieldsiem:~# mount /dev/vg01/lv01 /var/lib/docker/

root@dshieldsiem:~# df -h
/dev/mapper/vg01-lv01  931G  6.6G  925G   1% /var/lib/docker
```

---
# Enable `cgroup` - *Important!*
#### This step solves an error during the docker portion: `Your kernel does not support memory limit capabilities or the cgroup is not mounted. Limitation discarded.` *If you do not follow this step for an 8 GB RAM Raspberry Pi, expect critically degraded performance from the SIEM's dashboard.* 

1. Append this code to the first line in `/boot/firmware/cmdline.txt` to enable cgroup for memory. This will allow docker to manage the different containers' memory without crashing.
``` bash
root@dshieldsiem:~# echo -n "cgroup_enable=memory cgroup_memory=1" >> /boot/firmware/cmdline.txt
```
2. Reboot the Pi, and ssh back in when it's finished.
---
# Add swap - *Also Important!*
#### Don't do this on your boot drive (the Micro SD), do it on your HDD. You run the risk of damaging your physical drive with swap, so do your own research.
``` bash
# Turn off existing swap:
sudo dphys-swapfile swapoff

# Make a swap file and edit the dphys-swapfile configs:
sudo su -
cd /var/lib/docker
fallocate -l 4G swapfile
chmod 600 swapfile
mkswap swapfile
vi /etc/dphys-swapfile

# Uncomment the following lines and add these new values:
CONF_SWAPFILE=/var/lib/docker/swapfile
CONF_SWAPSIZE=4096
CONF_SWAPFACTOR=1
CONF_MAXSWAP=4096

# Initialize the swapfile
dphys-swapfile setup
dphys-swapfile swapon

# Check your new swap file and reboot:
swapon --show
free -h
reboot
```

---
# Install Docker for Debian (Not Raspbian!) 
*Commands modified/copied from: [bruneaug/DShield SIEM](https://github.com/bruneaug/DShield-SIEM/blob/main/README.md)

``` bash
sudo apt-get install ca-certificates curl gnupg network-manager txt2html

sudo install -m 0755 -d /etc/apt/keyrings

curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update && sudo apt upgrade

sudo reboot 

sudo apt-get install -y jq docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin pip

sudo systemctl enable docker
```

---
# Install DShield ELK
*Commands modified/copied from: [bruneaug/DShield SIEM](https://github.com/bruneaug/DShield-SIEM/blob/main/README.md)

1. Clone the SIEM's repository
``` bash
git clone https://github.com/bruneaug/DShield-SIEM.git

chmod 754 ~/DShield-SIEM/scripts/cowrie-setup.sh

mkdir scripts

mv DShield-SIEM/AddOnScripts/parsing_tty.sh scripts

mv DShield-SIEM/AddOnScripts/rename_arkime_pcap.sh scripts

chmod 754 scripts/*.sh
```

2. Modify the `.env` file to configure the stack to your environment:
``` bash
cd ~/DShield-SIEM

vi .env
```
Important variables to configure, and passwords to change:
- `ELASTIC_PASSWORD=student`
- `KIBANA_PASSWORD=changeme`
- `HOSTNAME="dshieldsiem"`
- `IPADDRESS="192.168.0.2"` (the IP of your Pi)

3. (Optional) Change the default nameservers in logstash configs. My entry for all: `nameserver => [ "9.9.9.9", "1.1.1.1" ]`
``` bash
cd logstash/pipeline

vim logstash-200-filter-cowrie.conf                          

vi logstash-201-filter-iptables.conf

vi logstash-202-filter-cowrie-webhoneypot.conf
```

4.  Use docker compose to build the ELK server's apps. There may still be some "`memory limit capabilities`" errors, but that's fine.
``` bash
cd ~/DShield-SIEM

sudo docker compose up -d
```
![](a/Pasted%20image%2020240717211708.png)
- You may need to unplug your Pi and plug it back in if it freezes after this.

5. Enable `docker.service` and check to see if it's running.
``` bash
sudo systemctl enable docker.service

sudo systemctl start docker.service

sudo systemctl status docker.service
```

6. Use `sudo docker container ls` to check that the services are running. It is normal as of this writing for `dshield-elk-setup-1` and `cowrie` containers to show up as constantly restarting.
![](a/Pasted%20image%2020240717214339.png)

7. Check to make sure the ELK services are listening on ports 9200, 8220, 5601, and 5044.
``` bash
sudo netstat -tulnp | grep '9200\|8220\|5601\|5044'
tcp        0      0 0.0.0.0:9200            0.0.0.0:*               LISTEN      1111/gdocker-proxy   
tcp        0      0 0.0.0.0:5044            0.0.0.0:*               LISTEN      1111/gdocker-proxy   
tcp        0      0 0.0.0.0:8220            0.0.0.0:*               LISTEN      1111/gdocker-proxy   
tcp        0      0 0.0.0.0:5601            0.0.0.0:*               LISTEN      1111/gdocker-proxy   
tcp6       0      0 :::9200                 :::*                    LISTEN      1111/gdocker-proxy   
tcp6       0      0 :::5044                 :::*                    LISTEN      1111/gdocker-proxy   
tcp6       0      0 :::8220                 :::*                    LISTEN      1111/gdocker-proxy   
tcp6       0      0 :::5601                 :::*                    LISTEN      1111/gdocker-proxy   
```

---
# Login to your Kibana server on port 5601 and configure the dashboard

1. In your web browser go to `https://<YOUR PI's IP>:5601` and bypass warnings about the certificate.

2. Login using the credentials that you configured in [Install DShield ELK Step 2](Installing%20DShield%20SIEM%20on%20a%20Raspberry%20Pi%205%20-%208%20GB%20RAM.md#Install%20DShield%20ELK)
![](a/Pasted%20image%2020240717215630.png)

3. Go to *Management > Stack Monitoring > **Or, set up with self monitoring**
![](a/Pasted%20image%2020240717221045.png)

---
# Configure elastic-agent
1. Go to *Management > Fleet > Settings > Outputs > Actions column > Pencil Icon*
 ![](a/Pasted%20image%2020240718161820.png)
![](a/Pasted%20image%2020240718162025.png)


2. In another window, SSH into your Pi, and get the SHA256 fingerprint of the dshield ELK cert:
``` bash 
ssh dshieldsiem@192.168.0.2

sudo cp /var/lib/docker/volumes/dshield-elk_certs/_data/ca/ca.crt /tmp

sudo openssl x509 -fingerprint -sha256 -noout -in /tmp/ca.crt | awk -F"=" {' print $2 '} | sed s/://g
```
The output should look like this: `0D9A25F4C147EB3A496253525DF6F039CF3C19776E64A1F77CEFCCD08B76BC61`


3. In the SSH session, print out a formatted version of the cert. Notice that each line in the output has been indented by 4 spaces. This will be pasted into the advanced YAML configuration along with some YAML syntax.
``` bash
sudo cat /tmp/ca.crt | sed -r 's/(.*)/    \1/g'
```
Output example:
```
    -----BEGIN CERTIFICATE-----
    MIIDSjCCAjKgAwIBAgIVAJ9e3PH7L0ay/zr1yX1j9Uy26A7SMA0GCSqGSIb3DQEB
    CwUAMDQxMjAwBgNVBAMTKUVsYXN0aWMgQ2VydGlmaWNhdGUgVG9vbCBBdXRvZ2Vu
    ZXJhdGVkIENBMB4XDTI0MDEwMzAyMjIzNloXDTI3MDEwMjAyMjIzNlowNDEyMDAG
    A1UEAxMpRWxhc3RpYyBDZXJ0aWZpY2F0ZSBUb29sIEF1dG9nZW5lcmF0ZWQgQ0Ew
    ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDSvwTbdP58APLOYUOCPl66
    V+Hy5HUYiCAe/302o7fSOUIbfLVqTLuFzA24EAWhCfQlM+nBE1ut7N+MJDTnePB2
    CmVTlUoMW9p69lbBgneDk3ytdgADbdu9k6bAI0okXRgGwJtANufjC4tXtD02fV+N
    MIe6WPwnDWkEeShNztFXUAsNAAy+V94MxJN+i4N4dQqda+tu+Gi7AyOVtl15vIns
    dSl+lTwqYXoSyS5eYOyYV1b3U8EI8ZMJe3gqnbHK18OB6DjcdOZMfuDOeZcVltkm
    iJODWcF48nmc/FkTX/RTmLrZltS+12vNrJytoVoVd+s82ezSTdmoykIGvPqWXOj1
    AgMBAAGjUzBRMB0GA1UdDgQWBBRi/jb1EKE8d1a7V96l6p7qZY3+GDAfBgNVHSME
    GDAWgBRi/jb1EKE8d1a7V96l6p7qZY3+GDAPBgNVHRMBAf8EBTADAQH/MA0GCSqG
    SIb3DQEBCwUAA4IBAQCLDTtyNQKaIm7FGSUVemL5kPL0viHbpaqtRyQBeY1wuZ0I
    ZHxCyjbUzWXYFv9YrZg/4YczKKPO/vIjw3REcayZeVD2WDuvABbPLKr15rgN9JP8
    ppzU5mX4Urb/8faRVcLNyVQvraVQ9kvwhT0B1pjL0go4ZDZ7LXJTMgtZbOWbfqFq
    BPN2S4HYs7o4T7ixYVAwr/QiUpX9I8MEr5/dE/cH46V7ov2h8luHdg0qZrE7jgyq
    neMAyt9RzDgZQLjD5vNHY6GzwnteRzPfEHrfb0AfSaG9oltdKL40yhZ4CP+teRp7
    oiwXvNQUtLjMoSKSA7gGujws2SY5Rd4YibQDQCYQ
    -----END CERTIFICATE-----
```


4. Add YAML syntax to the cert:
``` YAML
ssl:
  certificate_authorities:
  - |
    -----BEGIN CERTIFICATE-----
    MIIDSjCCAjKgAwIBAgIVAJ9e3PH7L0ay/zr1yX1j9Uy26A7SMA0GCSqGSIb3DQEB
    CwUAMDQxMjAwBgNVBAMTKUVsYXN0aWMgQ2VydGlmaWNhdGUgVG9vbCBBdXRvZ2Vu
    ZXJhdGVkIENBMB4XDTI0MDEwMzAyMjIzNloXDTI3MDEwMjAyMjIzNlowNDEyMDAG
    A1UEAxMpRWxhc3RpYyBDZXJ0aWZpY2F0ZSBUb29sIEF1dG9nZW5lcmF0ZWQgQ0Ew
    ggEiMA0GCSqGSIb3DQEBAQUAA4IBDwAwggEKAoIBAQDSvwTbdP58APLOYUOCPl66
    V+Hy5HUYiCAe/302o7fSOUIbfLVqTLuFzA24EAWhCfQlM+nBE1ut7N+MJDTnePB2
    CmVTlUoMW9p69lbBgneDk3ytdgADbdu9k6bAI0okXRgGwJtANufjC4tXtD02fV+N
    MIe6WPwnDWkEeShNztFXUAsNAAy+V94MxJN+i4N4dQqda+tu+Gi7AyOVtl15vIns
    dSl+lTwqYXoSyS5eYOyYV1b3U8EI8ZMJe3gqnbHK18OB6DjcdOZMfuDOeZcVltkm
    iJODWcF48nmc/FkTX/RTmLrZltS+12vNrJytoVoVd+s82ezSTdmoykIGvPqWXOj1
    AgMBAAGjUzBRMB0GA1UdDgQWBBRi/jb1EKE8d1a7V96l6p7qZY3+GDAfBgNVHSME
    GDAWgBRi/jb1EKE8d1a7V96l6p7qZY3+GDAPBgNVHRMBAf8EBTADAQH/MA0GCSqG
    SIb3DQEBCwUAA4IBAQCLDTtyNQKaIm7FGSUVemL5kPL0viHbpaqtRyQBeY1wuZ0I
    ZHxCyjbUzWXYFv9YrZg/4YczKKPO/vIjw3REcayZeVD2WDuvABbPLKr15rgN9JP8
    ppzU5mX4Urb/8faRVcLNyVQvraVQ9kvwhT0B1pjL0go4ZDZ7LXJTMgtZbOWbfqFq
    BPN2S4HYs7o4T7ixYVAwr/QiUpX9I8MEr5/dE/cH46V7ov2h8luHdg0qZrE7jgyq
    neMAyt9RzDgZQLjD5vNHY6GzwnteRzPfEHrfb0AfSaG9oltdKL40yhZ4CP+teRp7
    oiwXvNQUtLjMoSKSA7gGujws2SY5Rd4YibQDQCYQ
    -----END CERTIFICATE-----
```


5. Make the following config changes, then click *Save and apply settings,* and then *Save and deploy changes*
	- Change the *Hosts* field to `http://es01:9200`
	- Paste the SHA-256 fingerprint you generated in the terminal into the *Elasticsearch CA trusted fingerprint (optional)* field
	- Paste your version of the above cert + YAML syntax into the *Edit output* form that you opened in the elastic web console
![](a/Pasted%20image%2020240718170719.png)


6. In the same *Management > Fleet > Settings* window, generate a fleet server policy and add a fleet server.
![](a/Pasted%20image%2020240718172233.png)

7. Copy the code in the RPM tab into a text editor. After copying you must add an `s` to `http` as shown in the image below. Don't copy the top two lines.
![](a/Pasted%20image%2020240718173207.png)
Copy these next lines and add them to the end of the `elastic-agent enroll` command (before the systemctl commands). Don't forget to add a backslash after `--fleet-server-port=8220`
``` bash
  --url=https://fleet-server:8220\
  --certificate-authorities=/certs/ca/ca.crt \  
  --fleet-server-es-ca=/certs/es01/es01.crt \
  --insecure
```


8.  Execute a shell into your fleet-server Docker container using `sudo docker exec -ti fleet-server bash` and then run the following commands to check your agent, there may be errors:
``` bash
elastic-agent status
```


9. Paste your carefully edited commands from the text editor into the shell. This step can be very "touchy". If you make little mistakes, you can crash your Pi, but if everything is followed correctly, there won't be any problems. 
![](a/Pasted%20image%2020240718204642.png)
![](a/Pasted%20image%2020240718204837.png)


10. Check your `elastic-agent` again, and it should be healthy.
![](a/Pasted%20image%2020240718205134.png)


11. Exit the `fleet-server`

12. Back in the web console, go to the *Agents* tab and refresh your browser page if there's no agents present. You will see the `fleet-server`. Then click on the *Agent Policies* tab.
![](a/Pasted%20image%2020240718210503.png)

---
# Integrate Threat Intelligence
#### *Note: These installations can take a long time. You can save yourself some pain if you properly add swap, as shown earlier. You may prefer to customize those settings yourself.*
## Run these commands if you continue having problems, or see https://github.com/bruneaug/DShield-SIEM/blob/main/Troubleshooting/docker_useful_commands..md
``` bash
sudo docker compose stop
sudo docker compose start
```
## Requirements
You will need a free account and an API key for the following integrations:
- AlienVault OTX


1. From *Agent Policies* click on *Fleet Server Policy > Add Integration
![](a/Pasted%20image%2020240718210824.png)
![](a/Pasted%20image%2020240718210759.png)

2. Select the following integrations

- **AbuseCH:** Make sure you select *Where to add this integration? > Existing Hosts > Fleet Server Policy*. 
![](a/Pasted%20image%2020240718215931.png)
![](a/Pasted%20image%2020240718211959.png)
![](a/Pasted%20image%2020240718220200.png)
![](a/Pasted%20image%2020240718220222.png)

- **Threat Intelligence Utilities:** Make sure you select *Where to add this integration? > Existing Hosts > Fleet Server Policy*.
![](a/Pasted%20image%2020240718215340.png)
![](a/Pasted%20image%2020240718215445.png)
![](a/Pasted%20image%2020240718215855.png)


- **AlienVault OTX:** 
	- Make sure you select *Where to add this integration? > Existing Hosts > Fleet Server Policy*.
	- This requires you to create an account at otx.alienvault.com
	- Retrieve your API key in your account: *Click the gear in the upper right > Settings > Copy the OTX Key*
![](a/Pasted%20image%2020240723201444.png)
![](a/Pasted%20image%2020240723201511.png)
![](a/Pasted%20image%2020240723201835.png)
![](a/Pasted%20image%2020240723202023.png)
![](a/Pasted%20image%2020240723202101.png)


- **ElasticSearch**
![](a/Pasted%20image%2020240723215033.png)
![](a/Pasted%20image%2020240723215102.png)
![](a/Pasted%20image%2020240723230325.png)


* **Kibana**
![](a/Pasted%20image%2020240723230734.png)
![](a/Pasted%20image%2020240723230746.png)
![](a/Pasted%20image%2020240723230814.png)


- **Docker**
![](a/Pasted%20image%2020240723230904.png)
![](a/Pasted%20image%2020240723230916.png)
![](a/Pasted%20image%2020240723231009.png)

## Keep your integrations up to date. Click each one and enable *Keep integration policies up to date automatically*
#### (Most won't have this setting, but check anyways.)
![](a/Pasted%20image%2020240723231344.png)
![](a/Pasted%20image%2020240723231423.png)
![](a/Pasted%20image%2020240723231526.png)

---
# Add `, cowrie*` to the Elasticsearch Indices:
![](a/Pasted%20image%2020240723232826.png)
#### Click Advanced settings on the lefthand menu (appears after clicking *Management*) and then scroll down to *Elasticsearch Indices*.  Append `, cowrie*` then click out of the text box before clicking *Save changes*.
![](a/Pasted%20image%2020240723233224.png)

---
# From this point forward, follow Guy Bruneau's original tutorial to finish your SIEM installation and install the agent on your honeypot.
### Go to the *Configuration Security -> Rules* step at https://github.com/bruneaug/DShield-SIEM/tree/main
---
# I'd like to include a special *"Thank you!"* to the ISC's Guy Bruneau and  Jesse La Grew for their mentorship during my time in the ISC Internship.