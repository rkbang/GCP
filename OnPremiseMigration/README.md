# Migration to GCP
Migration of server from on premise instance to GCP instance.


## Implementation

- Create a GCP project with name `anthos-test` and make sure billing is enabled.

- Run `gcloud config set project anthos-test`
	- Now the variable $DEVSHELL_PROJECT_ID has the value anthos-test project ID.

- Run `gcloud config set compute/zone us-west1-a` to set the zone.  
  
- Adding firewall rules to allow traffic on http and ssh ports.
	- `gcloud compute firewall-rules create allow-ssh --network default --allow tcp:22 --source-ranges 0.0.0.0/0`
	- `gcloud compute firewall-rules create allow-http --network default --allow tcp:80 --source-ranges 0.0.0.0/0`

- Create the VM to be used for hosting server in GCP.
	- `gcloud compute instances create vm-instance-01 --project=$DEVSHELL_PROJECT_ID --zone=us-west1-a --machine-type=n1-standard-1 --subnet=default --scopes="cloud-platform" --tags=http-server,https-server --image=ubuntu-minimal-1604-xenial-v20210119a --image-project=ubuntu-os-cloud --boot-disk-size=10GB --boot-disk-type=pd-standard --boot-disk-device-name=vm-instance-01`

- Generating key pair and setting up the permissions.
	- `ssh-keygen -t rsa -f ~/.ssh/app-key -C [USERNAME]`
	- `chmod 400 ~/.ssh/app-key`
	- Checking the changes permission:  `ls -l | grep app-key`  
  
- Importing the key pair generated to Google Cloud:
	- `gcloud compute config-ssh --ssh-key-file=~/.ssh/app-key`

- Connect to the newly created VM  via SSH and running the following commands to install apache and unzip
	- `sudo apt-get update && sudo apt-get install apache2 unzip -y`  

- Move the existing index.html to replace it with new file.
	- `cd /var/www/html`  
	- `sudo mv index.html index.html.bkp`

- Download your server files and unzip them
	- `sudo curl -O https://storage.googleapis.com/bootcamp-gcp-en/hands-on-compute-website-files-en.zip`  
	- `sudo unzip hands-on-compute-website-files-en.zip`  
	- `sudo chmod 644 *`

- Copy the External IP of the Compute Engine created and access it via browser to validate if the move to GCP  was completed successfully.

- Stop VM instances: vm-instance-01.
