# Medical Diagnosis Validated Pattern — On-Prem Ceph Deployment

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)

## Overview

This repository is a comprehensive guide to deploying the **Medical Diagnosis Validated Pattern** on an **on-premise OpenShift cluster using local Ceph object storage** (OpenShift Data Foundation).

The pattern automates a complete AI/ML pipeline for chest X-ray analysis:
1. **Source**: Raw X-ray images are uploaded to a Ceph S3 bucket.
2. **Processing**: An "image-generator" triggers a Kafka event notification.
3. **Inference**: A KNative Serving (Serverless) function runs a pre-trained ML model to detect pneumonia.
4. **Results**: Processed images (with bounding boxes) and metadata are stored back in S3 and a PostgreSQL database.
5. **Visualization**: A Grafana dashboard displays real-time metrics and the processed X-ray results.

---

## 🚀 Quick Start for First-Time OpenShift Users

### 1. Prerequisites (Setup your Workstation)
Before starting, ensure you have the following installed:
- **oc CLI**: Log into your cluster using `oc login`.
- **git**: To clone this repository.
- **GitHub Account**: To fork this repository for your own deployment.
- **OpenShift Cluster (4.14+)**: You should have `cluster-admin` privileges.
- **ODF/Ceph**: OpenShift Data Foundation must be installed and a Ceph Object Store (RGW) must be active.

### 2. Fork and Clone
1. Fork this repository on GitHub to your account.
2. Clone your fork locally:
   ```bash
   git clone https://github.com/<your-username>/medical-diagnosis.git
   cd medical-diagnosis
   ```

### 3. Extract Ceph Credentials
The deployment needs to talk to your local S3 storage. Generate/Retrieve a Ceph user:
```bash
# Create the user if it doesn't exist
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

# Get the credentials
ACCESS_KEY=$(oc get secret rook-ceph-object-user-ocs-storagecluster-cephobjectstore-xraylab-user -n openshift-storage -o jsonpath='{.data.AccessKey}' | base64 -d)
SECRET_KEY=$(oc get secret rook-ceph-object-user-ocs-storagecluster-cephobjectstore-xraylab-user -n openshift-storage -o jsonpath='{.data.SecretKey}' | base64 -d)
echo "Access Key: $ACCESS_KEY"
echo "Secret Key: $SECRET_KEY"
```

### 4. Configure Your Dashboard Secrets
Create a secrets file in your **home directory** (DO NOT commit this to git):
```bash
cat > ~/values-secret-medical-diagnosis.yaml <<EOF
version: "2.0"
secrets:
  - name: xraylab
    fields:
      - name: database-user
        value: xraylab
      - name: database-password
        value: xraylab2024!Secure
      - name: database-host
        value: xraylabdb
      - name: database-db
        value: xraylabdb
  - name: s3-secret-bck
    fields:
      - name: AWS_ACCESS_KEY_ID
        value: $ACCESS_KEY
      - name: AWS_SECRET_ACCESS_KEY
        value: $SECRET_KEY
  - name: grafana
    fields:
      - name: GF_SECURITY_ADMIN_USER
        value: admin
      - name: GF_SECURITY_ADMIN_PASSWORD
        value: grafana2024!Secure
EOF
```

### 5. Update Global Configuration
Edit `values-global.yaml` in this repo. Replace the placeholders with your cluster's domain:
```yaml
global:
  datacenter:
    clustername: <your-cluster-name>
    domain: <apps.example.com>
```
*Tip: Run `oc get ingress.config cluster -o jsonpath='{.spec.domain}'` to find your base domain.*

### 6. Push Changes and Deploy
```bash
git add values-global.yaml
git commit -m "update cluster domain"
git push origin main

# Start the deployment
./pattern.sh make install
```

### 7. Seed the Data (CRITICAL)
The pattern is empty until you provide X-ray images. Run this command to copy the 624-image training set from the public AWS bucket to your local Ceph storage:
```bash
oc run seed-xrays --rm -it --restart=Never --image=python:3.9-slim -n xraylab-1 -- bash -c "pip install boto3 -q && python3 -c \"
import boto3
from botocore import UNSIGNED
from botocore.config import Config
src = boto3.client('s3', config=Config(signature_version=UNSIGNED))
dst = boto3.client('s3', endpoint_url='http://rook-ceph-rgw-ocs-storagecluster-cephobjectstore.openshift-storage.svc.cluster.local', aws_access_key_id='$ACCESS_KEY', aws_secret_access_key='$SECRET_KEY')
for folder in ['PNEUMONIA/', 'NORMAL/']:
    for obj in src.list_objects_v2(Bucket='validated-patterns-md-xray', Prefix=folder).get('Contents', []):
        data = src.get_object(Bucket='validated-patterns-md-xray', Key=obj['Key'])['Body'].read()
        dst.put_object(Bucket='xray-source', Key=obj['Key'], Body=data)
print('Data Seeding Complete!')
\""
```

---

## 📊 Verification & Access

### 🏥 The Grafana Dashboard
Access the dashboard via the OpenShift App Launcher or the following route:
- **URL**: `https://xraylab-grafana-route-xraylab-1.<your-domain>`
- **Username**: `admin`
- **Password**: `grafana2024!Secure`

> [!IMPORTANT]
> **SSL Certificate Workaround**: Because we use self-signed certificates for S3 storage, you **MUST** open the S3 RGW Route in your browser first and accept the certificate. Otherwise, Grafana will not be able to display the images.
> Find the RGW URL with: `oc get route s3-rgw -n openshift-storage`

### 🔍 Checking the Pipeline
Monitor the logs of the processing engine:
```bash
oc logs deployment/risk-assessment -n xraylab-1
```
Check the processed bucket for results:
```bash
oc run list-results --rm -it --image=python:3.9-slim -n xraylab-1 -- bash -c "pip install boto3 -q && python3 -c \"
import boto3
s3 = boto3.client('s3', endpoint_url='...', aws_access_key_id='...', aws_secret_access_key='...')
objs = s3.list_objects_v2(Bucket='xray-source-processed')['Contents']
print(f'Detected {len(objs)} processed images in S3.')
\""
```

---

## 🔧 Troubleshooting

| Issue | Solution |
|-------|----------|
| **"CreateContainerConfigError" on Grafana** | Ensure the `grafana-creds` secret exists. Re-run `make load-secrets`. |
| **Empty Dashboard Panels** | 1. Scaled up `image-generator`? 2. Accepted the S3 RGW SSL certificate in your browser? 3. Seeded the images? |
| **ArgoCD OutOfSync** | Use the ArgoCD UI or `oc patch` to force a sync on the `xraylab-grafana-dashboards` app. |

---

## 📖 Additional Documentation
- [Official Pattern Site](https://validatedpatterns.io/patterns/medical-diagnosis/)
- [Demo Script & Use Case](https://validatedpatterns.io/patterns/medical-diagnosis/demo-script/)
- [Cluster Sizing Guide](https://validatedpatterns.io/patterns/medical-diagnosis/cluster-sizing/)

---
*Created by the Red Hat Edge & AI Team.*
