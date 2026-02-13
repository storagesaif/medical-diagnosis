# Medical Diagnosis —  Deployment Guide (IBM Fusion - Ceph)

This guide walks you through deploying the **Medical Diagnosis Validated Pattern** on a fresh OpenShift cluster. 

We assume you are using **local Ceph storage** (OpenShift Data Foundation) instead of AWS. By the end of this, you will have a fully functional AI/ML pipeline for chest X-ray analysis, complete with a live Grafana dashboard.

---

## 🛠 Prerequisites

Before you start, make sure your workstation and cluster are ready:
1. **oc CLI**: You must be logged in to your OpenShift cluster as a **cluster-admin**.
2. **git**: To manage the code.
3. **GitHub Account**: You need to fork this repo so ArgoCD can track your changes.
4. **OpenShift Data Foundation (ODF)**: Must be installed on the cluster with an active **Object Gateway (RGW)**.

---

## 🚀 Phase 1: Preparation (Workstation)

### Step 1: Fork and Clone
1. Go to the [original repository](https://github.com/storagesaif/medical-diagnosis) and click **Fork** at the top right.
2. Clone **YOUR** fork to your computer:
   ```bash
   git clone https://github.com/<your-github-username>/medical-diagnosis.git
   cd medical-diagnosis
   ```

### Step 2: Identify Your Cluster Domain
You need to know your cluster's base URL. Run this:
```bash
oc get ingress.config cluster -o jsonpath='{.spec.domain}' && echo
# Example output: apps.mycluster.example.com
```
*   **Cluster Name**: `mycluster`
*   **Domain**: `example.com`

---

## 📦 Phase 2: Storage Setup (Ceph)

You need S3-compatible credentials to create the buckets where X-rays are stored.

### Step 3: Create a Ceph User
Run this command to create a user in your local storage:
```bash
cat <<EOF | oc apply -f -
apiVersion: ceph.rook.io/v1
kind: CephObjectStoreUser
metadata:
  name: xraylab-user
  namespace: openshift-storage
spec:
  store: ocs-storagecluster-cephobjectstore
  displayName: "XRay Lab User"
EOF
```

### Step 4: Extract Credentials
Wait 10 seconds, then run this to get your **Access Key** and **Secret Key**:
```bash
# Get the Access Key
oc get secret rook-ceph-object-user-ocs-storagecluster-cephobjectstore-xraylab-user \
  -n openshift-storage -o jsonpath='{.data.AccessKey}' | base64 -d && echo

# Get the Secret Key
oc get secret rook-ceph-object-user-ocs-storagecluster-cephobjectstore-xraylab-user \
  -n openshift-storage -o jsonpath='{.data.SecretKey}' | base64 -d && echo
```
> [!IMPORTANT]
> Copy these keys somewhere safe! You will need them in the next step.

---

## 🔐 Phase 3: Configuration (GitOps)

### Step 5: Create your Secrets File
Create a file on your computer in your **Home Directory** (NOT inside the git folder). This keeps your keys safe from being pushed to GitHub.

Save this as `~/values-secret-medical-diagnosis.yaml`:
```yaml
version: "2.0"
secrets:
  - name: xraylab
    fields:
      - name: database-user
        value: xraylab
      - name: database-password
        value: xraylab2024!Secure  # You can keep this or change it
      - name: database-host
        value: xraylabdb
      - name: database-db
        value: xraylabdb
  - name: s3-secret-bck
    fields:
      - name: AWS_ACCESS_KEY_ID
        value: <YOUR_ACCESS_KEY_FROM_STEP_4>
      - name: AWS_SECRET_ACCESS_KEY
        value: <YOUR_SECRET_KEY_FROM_STEP_4>
  - name: grafana
    fields:
      - name: GF_SECURITY_ADMIN_USER
        value: admin
      - name: GF_SECURITY_ADMIN_PASSWORD
        value: grafana2024!Secure  # This is your dashboard password
```

### Step 6: Update Global Values
Edit the `values-global.yaml` file inside your cloned `medical-diagnosis` folder:
```yaml
global:
  datacenter:
    clustername: <your-cluster-name> # From Step 2
    domain: <your-domain>           # From Step 2
```

### Step 7: Push to GitHub
ArgoCD needs to see your changes on GitHub to deploy them.
```bash
git add values-global.yaml
git commit -m "Update cluster details"
git push origin main
```

---

## 🏗 Phase 4: Deployment

### Step 8: Run the Installer
In your terminal, inside the `medical-diagnosis` folder, run:
```bash
./pattern.sh make install
```
*This command will take 10–15 minutes. It installs the operators, Kafka, the database, and the AI components.*

### Step 9: Seed the X-Ray Images (CRITICAL)
Your cluster starts with empty buckets. You MUST copy the demo images from the public AWS dataset to your local Ceph buckets.

Run this command (it uses a temporary Python pod to do the work):
```bash
# Set your keys as variables first
ACCESS_KEY="<YOUR_ACCESS_KEY>"
SECRET_KEY="<YOUR_SECRET_KEY>"

oc run seed-xrays --rm -it --restart=Never --image=python:3.9-slim -n xraylab-1 -- bash -c "pip install boto3 -q && python3 -c \"
import boto3
from botocore import UNSIGNED
from botocore.config import Config
print('Starting image transfer...')
src = boto3.client('s3', config=Config(signature_version=UNSIGNED))
dst = boto3.client('s3', 
    endpoint_url='http://rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc.cluster.local', 
    aws_access_key_id='$ACCESS_KEY', 
    aws_secret_access_key='$SECRET_KEY')

for folder in ['PNEUMONIA/', 'NORMAL/']:
    count = 0
    for obj in src.list_objects_v2(Bucket='validated-patterns-md-xray', Prefix=folder).get('Contents', []):
        data = src.get_object(Bucket='validated-patterns-md-xray', Key=obj['Key'])['Body'].read()
        dst.put_object(Bucket='xray-source', Key=obj['Key'], Body=data)
        count += 1
    print(f'Uploaded {count} images from {folder}')
print('Data Seeding Complete!')
\""
```

---

## 📊 Phase 5: Verification (Grafana)

### Step 10: The SSL Workaround
Because we use self-signed certificates, your browser will block images in the dashboard unless you "Accept the Risk" for the storage endpoint.

1. Find your S3 route:
   ```bash
   oc get route s3-rgw -n openshift-storage
   ```
2. Open that URL in a new browser tab.
3. You will see a security warning. Click **Advanced** -> **Proceed**.
4. You should see a blank XML page. **This is good.** 

### Step 11: Access the Dashboard
1. Go to your OpenShift Console.
2. Click the **App Launcher** (top right 3x3 dots) -> **Grafana**.
3. Log in with: 
   - **User**: `admin`
   - **Password**: `grafana2024!Secure`
4. Go to **Dashboards** -> **xraylab-1** -> **XRay Lab**.

### Step 12: Scale the Pipeline
If you don't see numbers increasing, make sure the image generator is alive:
```bash
oc scale deployment/image-generator --replicas=1 -n xraylab-1
```

---

## 🆘 Troubleshooting
- **Dashboard is empty?** Check the **Time Range** (top right) in Grafana. Set it to "Last 30 minutes".
- **Images not loading in panels?** Repeat Step 10 (SSL Workaround) and refresh Grafana.
- **Pods not starting?** Check ArgoCD logs: `oc get applications -n openshift-gitops`.

---
*Maintained by the Validated Patterns Team.*
