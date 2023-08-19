# Helm Chart for SFTP Server with Google Cloud Integration
This repository provides a Helm chart for deploying an SFTP server that synchronizes with Google Cloud Storage. It leverages the power of gcloud and gsutil to seamlessly fetch files from a specified bucket. Once synchronized, these files are made available via SFTP.

## Table of Contents
- [Configuration](#configuration)
    - [SFTP Users Secret](#sftp-users-secret)
    - [Workload-Identity](#workload-identity)
    - [GCS Service Account Secret](#google-cloud-service-account-secret)
- [Connection Details](#connection-details)

## Features
- SFTP Server: Provides a standard SFTP interface for accessing files.
- Google Cloud Storage Synchronization: Periodically fetches and updates files from a specified Google Cloud Storage bucket using gsutil.
- Security: Uses Kubernetes' native features to ensure the security and safety of your data.

## Prerequisites
1. A Kubernetes cluster.
2. Helm v3 or newer installed.
3. Google Cloud Service Account with permissions to access the desired GCS bucket.

## Configuration
To correctly deploy this Helm chart, you need to adjust the values.yaml and provide specific Kubernetes secrets.

### SFTP Users Secret
Define a username and password in a combined format for your SFTP access:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sftp-secret
type: Opaque
stringData:
  combinedcreds: username:password
```
**The combinedcreds value in the format username:password will be used as the SFTP credentials. When connecting to the SFTP server, you will use this username and password to authenticate on port 22.**

Then, in your `values.yaml`:
```yaml
environment:
  SFTP_USERS:
    secretKeyRef:
      name: sftp-secret
      key: combinedcreds
```
Additional configuration regarding your bucket/CRONJOB schedule should be adjusted
```yaml
sftp:
  schedule: 0 0 2 * * # CRONJOB Schedule - Once every month on its second day
  bucketName: carwiz_jato # Files will be downloaded from this bucket
```
### Workload Identity
If you're using Google Kubernetes Engine (GKE), Workload Identity allows your Kubernetes Service Account (KSA) to act as a Google Service Account (GSA). This avoids the need to manage secrets manually.

- Set `workloadIdentity` to `enabled` if you want to use Workload Identity.

  ```yaml
  workloadIdentity: enabled
  ```
- Configure the serviceAccount section:
    ```yaml
    serviceAccount:
    create: true
    annotations: 
        iam.gke.io/gcp-service-account: <GSA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com
    name: ""
    ```
**Setting up Workload Identity:** 
1. Make sure Workload Identity is enabled on your GKE cluster.
2. Create the Google Service Account if it does not exist.
3. Bind the KSA to the GSA:
```yaml
gcloud iam service-accounts add-iam-policy-binding \
  <GSA_NAME>@<PROJECT_ID>.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:<PROJECT_ID>.svc.id.goog[<NAMESPACE>/<KSA_NAME>]"
```
**Remember to replace placeholders (<..>) with actual values.**

4. Deploy your application. The Kubernetes Service Account used by your application will now act as the specified Google Service Account.
### Google Cloud Service Account Secret
**Optional - Incase Workload-Identity was configured this step can be skipped**

Create a secret containing the Google Cloud Service Account in JSON format with GCS permissions:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sftp-bucket-credentials
type: Opaque
stringData:
  credentials.json: |
    {
      "type": "service_account",
      "project_id": "your-project-id",
      ...
    }
```
Reference it in your `values.yaml`:
```yaml
volumes:
  gcsCredentials: sftp-bucket-credentials
```
### Installation
Deploy the Helm chart:
```
helm upgrade --install sftp-server . -f values.yaml --namespace sftp --create-namespace
```
After a few moments, your SFTP server should be up and running. It will periodically synchronize with the specified Google Cloud Storage bucket.
## Connection Details
To connect to the SFTP server, use the credentials specified in the [`combinedcreds`](#sftp-users-secret) secret. Here's a breakdown:

- **Username/Password: Extracted directly from the combinedcreds value, which is formatted as username:password.**
- **Port: 22**
- **Host/Address: The external IP of the SFTP service.**

Upon successful deployment of the Helm chart, you can obtain the external IP address of the service, and then use your preferred SFTP client to connect using the credentials and the IP address.


# Feedback and Contributions
Feedback is welcomed, issues, and pull requests! If you have any suggestions or find any bugs, please open an issue on my GitHub repository.
