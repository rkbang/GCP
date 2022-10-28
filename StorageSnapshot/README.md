# VM and DB Migration 

Hosts an application involving database on GCP. Then migrate respective working application and database to a different region using GCP infrastructure. 

## Implementation

Create VM instances to host app and database
- `gcloud compute instances create usa-app01 --project=aws-gcp-364823 --zone=us-west1-b --machine-type=e2-micro --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=652569283014-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --create-disk=auto-delete=yes,boot=yes,device-name=usa-app01,image=projects/debian-cloud/global/images/debian-10-buster-v20220920,mode=rw,size=10,type=projects/aws-gcp-364823/zones/us-west1-b/diskTypes/pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any`

- `gcloud compute instances create usa-db01 --project=aws-gcp-364823 --zone=us-west1-b --machine-type=e2-micro --network-interface=network-tier=PREMIUM,subnet=default --maintenance-policy=MIGRATE --provisioning-model=STANDARD --service-account=652569283014-compute@developer.gserviceaccount.com --scopes=https://www.googleapis.com/auth/cloud-platform --create-disk=auto-delete=yes,boot=yes,device-name=usa-app01,image=projects/debian-cloud/global/images/debian-10-buster-v20220920,mode=rw,size=10,type=projects/aws-gcp-364823/zones/us-west1-b/diskTypes/pd-balanced --no-shielded-secure-boot --shielded-vtpm --shielded-integrity-monitoring --reservation-affinity=any`

Download MySQL Server
SSH to database vm instance(usa-db01) and run following commands
- `sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 467B942D3A79BD29`
- Update: `sudo apt update`
- Install wget: `sudo apt-get -y install wget`
- Download MySql: `wget http://repo.mysql.com/mysql-apt-config_0.8.13-1_all.deb`
- Install MySql: `sudo dpkg -i mysql-apt-config_0.8.13-1_all.deb`
    - Select OK using MySql version 8

Install MySQL Server
- Update: `sudo apt update`
- Install: `sudo apt install mysql-server -y`
    - Insert and repeat the password of your choice
    - Click OK
    - Select default authentication plugin

Start MySQL service
- `sudo systemctl restart mysql.service`
- `sudo mysql_secure_installation`
    - Enter the chosen password
    - Answer N for all questions

Download your sql file
- `wget https://github.com/rkbang/GCP/blob/51b90651ef76d8bb525e3413dbd2e03517e815fc/StorageSnapshot/SQL/clinic.sql`

Connect to database
- `mysql -u root -p`
    - use the chosen password

Create the tables
- `source clinic.sql`

Creating an user and changing the privileges
-  `CREATE USER app@'%' IDENTIFIED BY 'welcome1';`
-  `GRANT ALL PRIVILEGES ON clinic.* TO app@'%';`
- `FLUSH PRIVILEGES;`
- `exit`

**Setup the app server on VM to use the created db**

Updating the OS and packages
- `sudo apt-get update`
- `sudo apt-get install -y npm`
- `sudo apt-get install -y zip`
- `sudo apt-get install -y wget`

Creating and accessing the folder to download application files (Node.js) 

- `wget https://github.com/rkbang/GCP/blob/51b90651ef76d8bb525e3413dbd2e03517e815fc/StorageSnapshot/app_server.zip`
- `unzip app_server.zip`
- `cd app_server`

From the folder app_server, run the following command to edit the file index.js:

`nano src/index.js`

Once inside of the file index.js, in the Middlewares section, replace:

```
     host: to the Private IP address of the vm usa-db01.
     user: to app,
     password: to welcome1
     database: to clinic
```

Save it.

Install: `npm install`

Start the Node.js application: `node src/index.js`

Creating a firewall rule to allow the TCP access on 3000 port of app server. From Firewall services, click on Create Firewall Rule:
- Name: allow-app-port-3000
- Targets: All instances in the network
- Source IP ranges: 0.0.0.0/0
- Select TCP and the port 3000
- Click on Create

Copy the public IP of the instance usa-app01 and paste in the browser! You shall now be able to add patients information which will then be updated in database.

After the successful access to the application, we can stop the VMs usa-db01 and usa-app01.

## Migrating the VMs using the storage snapshot

Checking the disks "in use" by VMs usa-app01 and usa-db01 and then Creating snapshot from disks of the VMs usa-app01 and usa-db01

- `gcloud compute disks snapshot usa-db01 --snapshot-names usa-db01-snapshot --zone us-west1-b`
- `gcloud compute disks snapshot usa-app01 --snapshot-names usa-app01-snapshot --zone us-west1-b`

Copy the snapshots to different region, using
australia-southeast1 (Sydney) region for the following implementation.

- `gcloud compute disks create aus-db01 --source-snapshot usa-db01-snapshot --zone australia-southeast1-a`
- `gcloud compute disks create aus-app01 --source-snapshot usa-app01-snapshot --zone australia-southeast1-a`

Spawn VM instances in new(Sydney) region using the snapshot copies

- `gcloud compute instances create aus-app01 --machine-type e2-micro --zone australia-southeast1-a --disk name=aus-app01,boot=yes,mode=rw`
- `gcloud compute instances create aus-db01 --machine-type e2-micro --zone australia-southeast1-a --disk name=aus-db01,boot=yes,mode=rw`

SSH to aus-app01 instance to update the IP address of db VM instance(aus-db01)

- `cd app_server`
- `vim src/index.js`

```host: to the private IP private of the VM aus-db01.```

Start the application 
- `node src/index.js`

Access the application `http://<external-ip-from-aus-app01:3000`