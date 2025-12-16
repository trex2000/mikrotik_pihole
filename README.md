# mikrotik_pihole
Creates a Pihole instance on Mikrotik routers which do not have USB port but have plenty of RAM inside a tmpfs and redirects DNS requests to it.
Script is based on:

https://help.mikrotik.com/docs/spaces/ROS/pages/84901929/Container

# Running create_ramdisk
The script shall be run with a scheduler with "Start time" set as startup

# Running start_pihole
This script shall be run only when the router has internet connection. One possibility is to run in on the On Up trigger of the PPOE interface.
Note: This script once successfully pulls the docker image and starts it will add the following rule to the Firewall's NAT table:

# Clearing the temporary rule
Run this script when the system starts up from the Scheduler, with "Start time" set as startup.
/ip firewall nat remove [find where comment="TEMP_DNS_REDIRECT_TO_PIHOLE"]

