# Migration to Kubernetis using Anthos





## Implementation

Following implementation has the instance name as "vm-instance-01" which will be used for migration. Please update to specific instance name as needed.

**GKE**  
  - Stop the existing instance which needed to be migrated to Anthos.
   
  - Create GKE cluster using gcloud command:
	  - `gcloud container clusters create vm-instance-01 --project=$DEVSHELL_PROJECT_ID --zone=us-west1-a --machine-type n1-standard-4 --cluster-version=1.21 --release-channel=stable --image-type ubuntu --num-nodes 1 --logging=SYSTEM --monitoring=SYSTEM --subnetwork "projects/$DEVSHELL_PROJECT_ID/regions/us-west1/subnetworks/default"`  
  
**Installing Migrate for Anthos**  
  
- Creating a service account rkb-m4a-install.  
  - `gcloud iam service-accounts create rkb-m4a-install --project=$DEVSHELL_PROJECT_ID`  
  
- Assigning a role storage.admin to a service account rkb-m4a-install.  
	- `gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member="serviceAccount:rkb-m4a-install@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com" --role=roles/storage.admin`  
  
- Creating and downloading the service account rkb-m4a-install key:
	- `gcloud iam service-accounts keys create rkb-m4a-install.json --iam-account=rkb-m4a-install@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --project=$DEVSHELL_PROJECT_ID`  
  
- Connecting to the Kubernetes cluster:  
	- `gcloud container clusters get-credentials vm-instance-01 --zone us-west1-a --project $DEVSHELL_PROJECT_ID`  
  
- Setting up the Migrate for Anthos components in the GKE cluster vm-instance-01  
	- `migctl setup install --json-key=rkb-m4a-install.json`  
  
- Running the command below to validate the Migrate for Anthos installation:
	- `migctl doctor`
	-  After 2-3 minutes you will see the success status indicating that the Deployment step has been completed: *[✓] Deployment*  
  
**Steps for VM migration**  
  
- Creating a service account rkb-m4a-ce-src:  
	- `gcloud iam service-accounts create rkb-m4a-ce-src --project=$DEVSHELL_PROJECT_ID`

- Assigning the role compute.viewer to a service account rkb-m4a-ce-src:
	- `gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member="serviceAccount:rkb-m4a-ce-src@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com" --role="roles/compute.viewer"`  
  
- Assigning the role de compute.storageAdmin to a service account rkb-m4a-ce-src:
	- `gcloud projects add-iam-policy-binding $DEVSHELL_PROJECT_ID --member="serviceAccount:rkb-m4a-ce-src@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com" --role="roles/compute.storageAdmin"`  
  
- Creating and downloading the service account rkb-m4a-ce-src key:
	- `gcloud iam service-accounts keys create rkb-m4a-ce-src.json --iam-account=rkb-m4a-ce-src@$DEVSHELL_PROJECT_ID.iam.gserviceaccount.com --project=$DEVSHELL_PROJECT_ID`  
  
- Defining the 'CE' (Compute Engine) as a source:  
	- `migctl source create ce vm-instance-01-source --project $DEVSHELL_PROJECT_ID --json-key=rkb-m4a-ce-src.json`

**Creating/Downloading and Checking the Migration plan**  
  
**Source name: vm-instance-01-source**  
  
- Creating the migration plan:  
	- `migctl migration create my-migration --source vm-instance-01-source --vm-id vm-instance-01 --intent Image`

- Checking the migration status:
	- `migctl migration status my-migration`

-  (Optional) To update the migration plan
	- Downloading the migration plan:  
		- `migctl migration get my-migration`  
	  -	Using the Text Editor to open the file my-migration.yaml.  
	  -	 If you change anything, run the command below to update the migration plan:
		  -	`migctl migration update my-migration --file my-migration.yaml`
  
- Migrating the VM using a migration plan:  
	- `migctl migration generate-artifacts my-migration`

- Checking the migration status (It takes between 5-10 minutes).
	- `migctl migration status my-migration`  
  
**Steps to deploy the migrated workload**

- Downloading the artifacts:
	- `migctl migration get-artifacts my-migration`

- Using Text Editor, open the file deployment_spec.yaml and update
  
```apiVersion: v1  
kind: Service  
metadata:  
creationTimestamp: null  
name: vm-instance-01  
spec:  
clusterIP: None  
selector:  
app: vm-instance-01  
type: ClusterIP  
status:  
loadBalancer: {}  
  
---  
  
apiVersion: v1  
kind: Service  
metadata:  
name: talent-management-portal  
spec:  
selector:  
app: vm-instance-01  
ports:  
- protocol: TCP  
port: 80  
targetPort: 80  
type: LoadBalancer 
```

- Applying the changes and deploying the workload:  
	- `kubectl apply -f deployment_spec.yaml`  
  
- Checking the external IP external:
	- `kubectl get service talent-management-portal`
	  
- When the web server is ready, you will see the external IP address to access the Talent Management Portal. So, you must copy the IP and paste it in the browser to test it! If it's Ok, you will see this screen of you modern application running in the GKE.
