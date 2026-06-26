# Ving Application - Google Cloud Marketplace Deployment Guide

Welcome to the deployment guide for the **Ving Application**. This document outlines the prerequisites and step-by-step instructions for deploying Ving via the Google Cloud Marketplace into your Google Kubernetes Engine (GKE) cluster.

## 1. Prerequisites

Before deploying the Ving Application from the Google Cloud Marketplace, ensure that you have the following Google Cloud services provisioned and accessible from your GKE cluster:

- **Google Kubernetes Engine (GKE)**: A running cluster with **Workload Identity** enabled.
- **Cloud SQL for PostgreSQL**: A running PostgreSQL instance.
- **Cloud Spanner**: A Spanner instance and database.
- **Memorystore for Redis**: A running Redis instance.

---

## 2. Pre-Deployment Preparation

Because Ving relies on native Google Cloud data services, you must prepare these services before initiating the Marketplace deployment.

### A. Prepare PostgreSQL for Temporal

Ving includes Temporal, which requires two specific databases to be created in your Cloud SQL PostgreSQL instance beforehand. Connect to your PostgreSQL instance and execute the following commands:

```sql
CREATE DATABASE temporal;
CREATE DATABASE temporal_visibility;
```

### B. Configure Cloud Spanner Authentication (Workload Identity)

The Ving backend authenticates to Cloud Spanner using Kubernetes Workload Identity. You must create a Google Service Account (GSA) and bind it to the Kubernetes Service Account (KSA) that will be deployed by the Helm chart (`ving-backend-sa`).

**Step 1: Create a Google Service Account (GSA) and grant Spanner access**

```bash
# Export your GCP Project ID
export PROJECT_ID="your-gcp-project-id"

# Create the service account
gcloud iam service-accounts create ving-backend-sa

# Grant the Database User role for Spanner
gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:ving-backend-sa@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/spanner.databaseUser"
```

**Step 2: Bind the GSA to the Kubernetes Service Account (KSA)**

```bash
# Allow the KSA to impersonate the GSA
gcloud iam service-accounts add-iam-policy-binding ving-backend-sa@$PROJECT_ID.iam.gserviceaccount.com \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[ving/ving-backend-sa]"

# Create the deployment namespace (ensure this matches your marketplace configuration)
kubectl create namespace ving

# Annotate the KSA (The Ving deployment chart will create the KSA, but you can pre-annotate it or apply this after the KSA is generated)
kubectl annotate serviceaccount ving-backend-sa \
    --namespace ving \
    iam.gke.io/gcp-service-account=ving-backend-sa@$PROJECT_ID.iam.gserviceaccount.com
```

> [!NOTE]
> The Ving Helm chart automatically creates the `ving-backend-sa` KSA. The application code using the GCP SDK will now automatically authenticate to Spanner using this Workload Identity setup.

---

## 3. Deploying via Google Cloud Marketplace

Once the prerequisites are prepared, you can deploy Ving directly from the Google Cloud Console.

1. Navigate to the **Ving Application** page in the Google Cloud Marketplace.
2. Click **Deploy**.
3. Select the target **GKE Cluster** where you have enabled Workload Identity.
4. Fill in the **Configuration Parameters**:
   - **Namespace**: Specify the namespace (e.g., `ving`). Must match the namespace used in your Workload Identity binding.
   - **Application Name**: Name your deployment instance.
   - **Minio Credentials**: Provide a unique Username and Password (this will be injected into `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`).
   - **Database Credentials**: Input your specific connection strings, hosts, usernames, and passwords for:
     - Cloud SQL (Postgres Host, User, Password, DB)
     - Cloud Spanner (Project ID, Instance ID, Database ID)
     - Memorystore (Redis URL)
5. Review the settings and click **Deploy**.

---

## 4. Post-Deployment & Verification

After the deployment succeeds, the Google Cloud Console will display your Application resource.

1. **Verify Pod Health**: 
   Ensure all pods (Frontend, Backend, Minio, Temporal) are in a `Running` state.
   ```bash
   kubectl get pods -n ving
   ```

2. **Access the Application**: 
   The Frontend and Backend are exposed via `LoadBalancer` services. Retrieve their external IP addresses using:
   ```bash
   kubectl get services -n ving
   ```
   Navigate to the `EXTERNAL-IP` of the frontend service in your browser to access the Ving application.
