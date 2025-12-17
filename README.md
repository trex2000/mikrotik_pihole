# mikrotik_pihole
Creates a Pihole instance on Mikrotik routers which do not have USB port but have plenty of RAM inside a tmpfs and redirects DNS requests to it.
Script is based on:

https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container

# Prerequisites:
# 1. Enable Container mode and follow the instructions the command gives you (read more about Device-mode). 
You will need to confirm the device-mode with a press of the reset button, or a cold reboot (if using Containers on x86):
/system/device-mode/update container=yes 

# 2. Create a new veth interface and assign an IP address in a range that is unique in your network:
The following configuration is equivalent to "bridge" networking mode in other Container engines such as Docker. It is possible to create a "host" equivalent configuration as well.
One veth interface can be used for many Containers. You can create multiple veth interfaces to create isolated networks for different Containers.
/interface/veth/add name=veth1 address=172.17.0.2/24 gateway=172.17.0.1

# 3. Create a new bridge that is going to be used for your Containers and assign the same IP address that was used for the veth interface's gateway:
/interface/bridge/add name=containers
/ip/address/add address=172.17.0.1/24 interface=containers

# 4. Add the veth interface to your newly created bridge:
/interface/bridge/port add bridge=containers interface=veth1

# 5. Create a NAT for outgoing traffic:
/ip/firewall/nat/add chain=srcnat action=masquerade src-address=172.17.0.0/24

# 6. Create a port forward to the container
/ip firewall nat
add action=dst-nat chain=dstnat dst-address=192.168.88.1 dst-port=80 protocol=tcp to-addresses=172.17.0.2 to-ports=80


# 7. Create environment variables for the Container:
/container/envs/add list=ENV_PIHOLE key=TZ value="Europe/Bucharest"
/container/envs/add list=ENV_PIHOLE key=FTLCONF_webserver_api_password value="YOUR_VERY_SECURE_PASSWORD"
/container/envs/add list=ENV_PIHOLE key=DNSMASQ_USER value="root"
/container/envs/add list=ENV_PIHOLE key=PIHOLE_DNS_ value="8.8.8.8;8.8.4.4;2001:4860:4860::8888;2001:4860:4860::8844"
#allow answer from all ip ranges
/container/envs/add list=ENV_PIHOLE key=DNSMASQ_LISTENING value="all"
/container/envs/add list=ENV_PIHOLE key=FTLCONF_dns_listeningMode value="all"

# 8 Create ramdisk for the container
/disk add type=tmpfs tmpfs-max-size=1G slot=ramdisk1

# 9. Create mounted volumes for the Container:
/container/mounts/add name=MOUNT_PIHOLE_PIHOLE src=ramdisk1/volumes/pihole/pihole dst=/etc/pihole
/container/mounts/add name=MOUNT_PIHOLE_DNSMASQD src=ramdisk1/volumes/pihole/dnsmasq.d dst=/etc/dnsmasq.d

# 10. Configure to use a specific Container repository, for example, to use Docker.io:
/container/config set registry-url="https://registry.hub.docker.com" tmpdir=ramdisk1/tmp

# 11. Import missing certificates
/tool fetch https://cacerts.digicert.com/DigiCertAssuredIDRootG2.crt.pem
/certificate import file-name=DigiCertAssuredIDRootG2.crt.pem passphrase=””
/tool/fetch url="https://curl.se/ca/cacert.pem"
/certificate import file-name=cacert.pem passphrase=””

:local containerName "pihole"
    :local cID [/container find name=$containerName]

    :if ($cID != "") do={
        :put "Stopping and removing container: $containerName"
        /container stop $cID
        
        # Wait for the container to stop before removing
        :while ([/container get $cID status] != "stopped") do={ :delay 1s }
        
        /container remove $cID
        :put "Container removed successfully."
    } else={
        :put "Container $containerName does not exist. Nothing to do."


# 12. Add a Containter  (Note: this is automated with the start_pihole script, see below)
/container/add remote-image=pihole/pihole interface=veth1 root-dir=ramdisk1/images/pihole mounts=MOUNT_PIHOLE_PIHOLE,MOUNT_PIHOLE_DNSMASQD envlist=ENV_PIHOLE name=pihole

# 13. Start the container (Note: this is automated with the start_pihole script, see below)
/container start pihole

# Information regarding deploying the script:

# Running create_ramdisk
The script shall be run with a scheduler with "Start time" set as startup

# Running start_pihole
This script shall be run only when the router has internet connection. One possibility is to run in on the On Up trigger of the PPOE interface.
Note: This script once successfully pulls the docker image and starts it will add the following rule to the Firewall's NAT table:

# Clearing the temporary rule
Run this script when the system starts up from the Scheduler, with "Start time" set as startup.
/ip firewall nat remove [find where comment="TEMP_DNS_REDIRECT_TO_PIHOLE"]

