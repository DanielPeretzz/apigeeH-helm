# מדריך התקנת Apigee Hybrid 1.16 - סיכום מסודר ומדויק

## הכנת הסביבה והורדת Helm Charts

### 1. התקנת כלים בסיסיים
```bash
# התקנת Helm (פעם אחת בלבד)
winget install helm.helm
export PATH=$PATH:/c/helm/windows-amd64

# התחברות לחשבון Google Cloud
gcloud auth login
```

### 2. יצירת תיקיות והורדת Helm Charts
```bash
# יצירת תיקיית עבודה
mkdir -p apigee-hybrid/helm-charts
cd apigee-hybrid/helm-charts
export APIGEE_HELM_CHARTS_HOME=$PWD

# הגדרת משתני סביבה
export CHART_REPO=oci://us-docker.pkg.dev/apigee-release/apigee-hybrid-helm-charts
export CHART_VERSION=1.16.0
export PROJECT_ID=bnhp-apigeeh-poc

# הורדת כל ה-Helm Charts (גרסה 1.16.0)
helm pull $CHART_REPO/apigee-operator --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-datastore --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-env --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-ingress-manager --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-org --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-redis --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-telemetry --version $CHART_VERSION --untar
helm pull $CHART_REPO/apigee-virtualhost --version $CHART_VERSION --untar
```

## הגדרת Kubernetes Cluster

### 3. חיבור ל-EKS Cluster
```bash
# הגדרת credentials AWS
export AWS_ACCESS_KEY_ID="YOUR_ACCESS_KEY"
export AWS_SECRET_ACCESS_KEY="YOUR_SECRET_KEY"
export AWS_SESSION_TOKEN="YOUR_SESSION_TOKEN"

# התחברות ל-EKS cluster
aws eks update-kubeconfig --region eu-central-1 --name apigee-hybrid

# יצירת namespace עבור Apigee
kubectl create namespace apigee
```

## יצירת Service Accounts ו-Certificates

### 4. יצירת Service Account של Google Cloud
```bash
# יצירת service account (עבור environment: non-prod)
$APIGEE_HELM_CHARTS_HOME/apigee-operator/etc/tools/create-service-account \
  --env non-prod \
  --dir $APIGEE_HELM_CHARTS_HOME/apigee-datastore

# העתקת ה-service account JSON לכל התיקיות הרלוונטיות
cp $APIGEE_HELM_CHARTS_HOME/apigee-datastore/${PROJECT_ID}-apigee-non-prod.json \
   $APIGEE_HELM_CHARTS_HOME/apigee-operator/

cp $APIGEE_HELM_CHARTS_HOME/apigee-datastore/${PROJECT_ID}-apigee-non-prod.json \
   $APIGEE_HELM_CHARTS_HOME/apigee-telemetry/

cp $APIGEE_HELM_CHARTS_HOME/apigee-datastore/${PROJECT_ID}-apigee-non-prod.json \
   $APIGEE_HELM_CHARTS_HOME/apigee-org/

cp $APIGEE_HELM_CHARTS_HOME/apigee-datastore/${PROJECT_ID}-apigee-non-prod.json \
   $APIGEE_HELM_CHARTS_HOME/apigee-env/
```

### 5. יצירת TLS Certificates
```bash
# יצירת תיקיית certificates
mkdir $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs/

# יצירת self-signed certificate (ל-10 שנים)
openssl req -nodes -new -x509 \
  -keyout $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs/keystore_test-poc.key \
  -out $APIGEE_HELM_CHARTS_HOME/apigee-virtualhost/certs/keystore_test-poc.pem \
  -subj '//CN=blabla.com' \
  -days 3650
```

### 6. הגדרת Control Plane Access
```bash
# הגדרת access token
export TOKEN=$(gcloud auth print-access-token)

# הגדרת Synchronizer identities
curl -X PATCH -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type:application/json" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/controlPlaneAccess?update_mask=synchronizer_identities" \
  -d "{\"synchronizer_identities\": [\"serviceAccount:apigee-non-prod@${PROJECT_ID}.iam.gserviceaccount.com\"]}"

# הגדרת Analytics Publisher identities
curl -X PATCH -H "Authorization: Bearer $(gcloud auth print-access-token)" \
  -H "Content-Type:application/json" \
  "https://apigee.googleapis.com/v1/organizations/${PROJECT_ID}/controlPlaneAccess?update_mask=analytics_publisher_identities" \
  -d "{\"analytics_publisher_identities\": [\"serviceAccount:apigee-non-prod@${PROJECT_ID}.iam.gserviceaccount.com\"]}"
```

## התקנת Cert-Manager ו-CRDs

### 7. התקנת Cert-Manager
```bash
# התקנת cert-manager גרסה 1.19.2 (נתמכת ע"י Apigee 1.16)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.19.2/cert-manager.yaml

# בדיקת הסטטוס
kubectl get all -n cert-manager -o wide
```

### 8. התקנת Apigee CRDs
```bash
kubectl apply -k apigee-operator/etc/crds/default/ \
  --server-side \
  --force-conflicts \
  --validate=false

# אימות ההתקנה
kubectl get crds | grep apigee
```

## הגדרת Storage Class

### 9. יצירת Storage Class עבור AWS EBS
```bash
# יצירת קובץ sc.yaml והחלתו
kubectl apply -f sc.yaml

# הגדרה כ-default storage class
kubectl patch storageclass apigee-sc \
  -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# אימות
kubectl get sc
```

**תוכן קובץ sc.yaml:**
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: apigee-sc
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

## התקנת Apigee Components בסדר הנכון

### 10. התקנת Apigee Operator
```bash
helm upgrade operator apigee-operator/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

kubectl get pod -n apigee
```

### 11. התקנת Datastore (Cassandra)
```bash
helm upgrade datastore apigee-datastore/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

kubectl get pod -n apigee
```

### 12. התקנת Telemetry
```bash
helm upgrade telemetry apigee-telemetry/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml
```

### 13. התקנת Redis
```bash
helm upgrade redis apigee-redis/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml
```

### 14. התקנת Ingress Manager
```bash
helm upgrade ingress-manager apigee-ingress-manager/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml
```

### 15. התקנת Organization
```bash
helm upgrade ${PROJECT_ID} apigee-org/ \
  --install \
  --namespace apigee \
  --atomic \
  -f overrides.yaml

# במקרה של בעיות, ניתן להשתמש ב-force
helm upgrade ${PROJECT_ID} apigee-org/ \
  --install \
  --namespace apigee \
  --force \
  -f overrides.yaml
```

### 16. התקנת Environment
```bash
helm upgrade test-poc apigee-env/ \
  --install \
  --namespace apigee \
  --atomic \
  --set env=test-poc \
  -f overrides.yaml
```

### 17. התקנת Virtual Host
```bash
helm upgrade test-poc-group apigee-virtualhost/ \
  --install \
  --namespace apigee \
  --set envgroup=test-poc-group \
  -f overrides.yaml
```

## חשיפת Load Balancer

### 18. יצירת Load Balancer Service
```bash
# יצירת קובץ apigee-lb.yaml והחלתו
kubectl apply -f apigee-lb.yaml

# בדיקת הסטטוס
kubectl get svc -n apigee -l app=apigee-ingressgateway
kubectl get svc -n apigee apigee-lb
```

## פקודות אימות

```bash
# בדיקת כל הרכיבים
kubectl get all -n apigee

# בדיקת Apigee Organization
kubectl get apigeeorganization -n apigee

# בדיקת Apigee Runtime
kubectl -n apigee get ar
kubectl -n apigee get arc

# בדיקת ingress
kubectl get ingress -n apigee
```

---


### הערות חשובות:
- **סדר ההתקנה קריטי**: Operator → Datastore → Telemetry → Redis → Ingress Manager → Org → Env → VirtualHost
- **Storage Class**: חובה להגדיר לפני התקנת Datastore
- **Cert-Manager**: חייב להיות מותקן לפני Operator (גרסה 1.18 או 1.19)
- **Service Accounts**: חייבים להיות בכל התיקיות הרלוונטיות
- **Control Plane Access**: חייב להיות מוגדר לפני התקנת Organization

### דרישות מקדימות:
- Helm v3.14.2 או גבוה יותר
- kubectl בגרסה תואמת ל-Kubernetes platform שלך
- cert-manager בגרסה נתמכת (1.18 או 1.19)
- Kubernetes cluster עם תצורה מינימלית לפי הדרישות של Apigee

### תכונות חדשות ב-Apigee Hybrid 1.16:
- Seccomp Profiles עבור runtime components
- apigee-guardrails Google IAM service account
- UDCA component הוסר - analytics דרך Google Cloud Pub/Sub
- תמיכה ב-cert-manager 1.18 ו-1.19

---

## מקורות

התיעוד הרשמי של Google Cloud:
- [Installing and managing Apigee hybrid with Helm charts](https://docs.cloud.google.com/apigee/docs/hybrid/preview/helm-install)
- [Upgrading Apigee hybrid to version 1.16](https://docs.cloud.google.com/apigee/docs/hybrid/v1.16/upgrade)
- [Apigee hybrid release notes](https://docs.cloud.google.com/apigee/docs/hybrid/release-notes)
- [Apigee Hybrid now uses Helm charts for configuration](https://cloud.google.com/blog/products/api-management/apigee-hybrid-now-uses-helm-charts-for-configuration)

---

## סיכום התהליך (TL;DR)

1. התקן Helm ו-gcloud CLI
2. הורד את כל ה-Helm charts בגרסה 1.16.0
3. התחבר ל-EKS cluster וצור namespace
4. צור service accounts והעתק אותם לכל התיקיות
5. צור TLS certificates
6. הגדר Control Plane Access
7. התקן cert-manager (v1.19.2)
8. התקן Apigee CRDs
9. צור Storage Class והגדר אותו כ-default
10. התקן את הרכיבים בסדר: Operator → Datastore → Telemetry → Redis → Ingress Manager → Org → Env → VirtualHost
11. חשוף את ה-Load Balancer
12. אמת את ההתקנה

**זמן התקנה משוער**: 2-4 שעות (תלוי בגודל ה-cluster ומהירות הרשת)
