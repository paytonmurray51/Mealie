# Mealie on Google Cloud Run - Deployment Guide

Complete guide for deploying [Mealie](https://mealie.io/) recipe manager on Google Cloud Run with PostgreSQL.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Deployment Steps](#deployment-steps)
- [Post-Deployment](#post-deployment)
- [Updating](#updating)
- [Troubleshooting](#troubleshooting)
- [Cost Estimate](#cost-estimate)

## Prerequisites

- Google Cloud Project with billing enabled
- `gcloud` CLI installed and authenticated
- Docker installed locally (or use Cloud Build)

## Configuration

Set your project-specific variables:
```bash
export PROJECT_ID="your-project-id"
export REGION="us-central1"
export DB_INSTANCE="mealie-db"
export SERVICE_NAME="mealie"
export BASE_URL="https://mealie.yourdomain.com"
export DEFAULT_EMAIL="your-email@example.com"
export TZ="America/Chicago"
```

## Deployment Steps

### 1. Enable Required APIs
```bash
gcloud config set project $PROJECT_ID
gcloud services enable run.googleapis.com
gcloud services enable sqladmin.googleapis.com
gcloud services enable secretmanager.googleapis.com
gcloud services enable artifactregistry.googleapis.com
gcloud services enable cloudbuild.googleapis.com
```

### 2. Create Cloud SQL PostgreSQL Database
```bash
# Create PostgreSQL instance
gcloud sql instances create $DB_INSTANCE \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=$REGION \
  --root-password=$(openssl rand -base64 32)

# Enable public IP
gcloud sql instances patch $DB_INSTANCE \
  --assign-ip \
  --authorized-networks=0.0.0.0/0

# Create database
gcloud sql databases create mealie --instance=$DB_INSTANCE

# Generate and store password
MEALIE_DB_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)
echo "Database password: $MEALIE_DB_PASSWORD"

# Create database user
gcloud sql users create mealie \
  --instance=$DB_INSTANCE \
  --password=$MEALIE_DB_PASSWORD
```

### 3. Store Password in Secret Manager
```bash
# Create secret
echo -n "$MEALIE_DB_PASSWORD" | gcloud secrets create mealie-db-password --data-file=-

# Grant Cloud Run access to secret
PROJECT_NUMBER=$(gcloud projects describe $PROJECT_ID --format="value(projectNumber)")
gcloud secrets add-iam-policy-binding mealie-db-password \
  --member="serviceAccount:${PROJECT_NUMBER}-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

### 4. Mirror Docker Image to Artifact Registry
```bash
# Create Artifact Registry repository
gcloud artifacts repositories create mealie \
  --repository-format=docker \
  --location=$REGION \
  --description="Mealie recipe manager images"

# Use Cloud Build to mirror the image
gcloud builds submit --no-source --config=- <<EOF
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['pull', 'ghcr.io/mealie-recipes/mealie:v3.5.0']
- name: 'gcr.io/cloud-builders/docker'
  args: ['tag', 'ghcr.io/mealie-recipes/mealie:v3.5.0', '$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:v3.5.0']
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', '$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:v3.5.0']
images:
- '$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:v3.5.0'
EOF
```

### 5. Deploy to Cloud Run
```bash
# Get database IP
DB_HOST=$(gcloud sql instances describe $DB_INSTANCE --format="value(ipAddresses[0].ipAddress)")

# Deploy Mealie
gcloud run deploy $SERVICE_NAME \
  --image=$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:v3.5.0 \
  --region=$REGION \
  --memory=1Gi \
  --cpu=1 \
  --port=9000 \
  --timeout=600 \
  --min-instances=0 \
  --max-instances=1 \
  --set-env-vars="DB_ENGINE=postgres,POSTGRES_USER=mealie,POSTGRES_SERVER=${DB_HOST},POSTGRES_PORT=5432,POSTGRES_DB=mealie,ALLOW_SIGNUP=false,TZ=$TZ,BASE_URL=$BASE_URL,DEFAULT_EMAIL=$DEFAULT_EMAIL,API_PORT=9000,MAX_WORKERS=1,WEB_CONCURRENCY=1" \
  --set-secrets="POSTGRES_PASSWORD=mealie-db-password:latest" \
  --allow-unauthenticated \
  --no-cpu-throttling \
  --cpu-boost

# Get service URL
gcloud run services describe $SERVICE_NAME --region=$REGION --format="value(status.url)"
```

### 6. Grant Database Permissions (for backup restore)
```bash
# Set postgres password
POSTGRES_PASSWORD=$(openssl rand -base64 32)
echo "Postgres password: $POSTGRES_PASSWORD"
gcloud sql users set-password postgres \
  --instance=$DB_INSTANCE \
  --password="$POSTGRES_PASSWORD"

# Connect to database
gcloud sql connect $DB_INSTANCE --user=postgres
```

In the PostgreSQL prompt, run:
```sql
ALTER USER mealie WITH REPLICATION;
ALTER DATABASE mealie OWNER TO mealie;
GRANT cloudsqlsuperuser TO mealie;
\q
```

## Post-Deployment

### Default Credentials

- **Email**: `changeme@example.com`
- **Password**: `MyPassword`

⚠️ **Important**: Change these immediately after first login!

### Set Up Custom Domain (Optional)
```bash
gcloud run domain-mappings create \
  --service=$SERVICE_NAME \
  --domain=$BASE_URL \
  --region=$REGION
```

Follow the DNS instructions provided to add CNAME records to your domain.

### Set Up Billing Alerts
```bash
# Create a budget alert
gcloud billing budgets create \
  --billing-account=$(gcloud billing projects describe $PROJECT_ID --format="value(billingAccountName)") \
  --display-name="Mealie Monthly Budget" \
  --budget-amount=20 \
  --threshold-rule=percent=50 \
  --threshold-rule=percent=90 \
  --threshold-rule=percent=100
```

## Updating

To update Mealie to a new version:
```bash
# Set new version
NEW_VERSION="v3.6.0"  # Update as needed

# Mirror new image
gcloud builds submit --no-source --config=- <<EOF
steps:
- name: 'gcr.io/cloud-builders/docker'
  args: ['pull', "ghcr.io/mealie-recipes/mealie:$NEW_VERSION"]
- name: 'gcr.io/cloud-builders/docker'
  args: ['tag', "ghcr.io/mealie-recipes/mealie:$NEW_VERSION", "$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:$NEW_VERSION"]
- name: 'gcr.io/cloud-builders/docker'
  args: ['push', "$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:$NEW_VERSION"]
images:
- "$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:$NEW_VERSION"
EOF

# Update deployment
gcloud run deploy $SERVICE_NAME \
  --image=$REGION-docker.pkg.dev/$PROJECT_ID/mealie/mealie:$NEW_VERSION \
  --region=$REGION
```

## Troubleshooting

### View Logs
```bash
gcloud logging read "resource.type=cloud_run_revision AND resource.labels.service_name=$SERVICE_NAME" \
  --limit=50 \
  --format="table(timestamp,textPayload)"
```

### Check Cloud SQL Status
```bash
gcloud sql instances describe $DB_INSTANCE
```

### Test Database Connection
```bash
gcloud sql connect $DB_INSTANCE --user=mealie --database=mealie
```

### Common Issues

**Issue**: Container fails to start with "empty host" error
- **Solution**: Ensure `POSTGRES_SERVER` is set to the database IP address, not a Unix socket path

**Issue**: "relation does not exist" errors
- **Solution**: Database migrations didn't run. Check logs and ensure database connection is established

**Issue**: Backup restore fails with permission denied
- **Solution**: Run the database permission grants from step 6

**Issue**: Unable to pull image from ghcr.io
- **Solution**: Cloud Run requires images in GCR or Artifact Registry. Use the Cloud Build mirror step

## Cost Estimate

| Service | Configuration | Monthly Cost |
|---------|--------------|--------------|
| Cloud SQL | db-f1-micro (0.6GB RAM) | ~$10 |
| Cloud Run | Scale-to-zero, light usage | ~$1-3 |
| Storage | Backups and data | ~$0.50 |
| **Total** | | **~$12-15/month** |

### Cost Optimization Tips

1. **Scale to zero**: Set `--min-instances=0` to avoid charges when not in use
2. **Use db-f1-micro**: Sufficient for personal/family use (1-20 users)
3. **Enable automated backups to Cloud Storage**: More cost-effective than Cloud SQL backups
4. **Set billing alerts**: Avoid unexpected charges

## Security Considerations

- Store all sensitive passwords in Secret Manager
- Restrict Cloud SQL `authorized-networks` to Cloud Run's IP ranges after testing
- Enable Cloud SQL SSL connections for production
- Regularly update to the latest Mealie version
- Use strong, unique passwords for all accounts
- Consider using OIDC authentication for additional security

## Architecture
```
┌─────────────┐
│   User      │
└──────┬──────┘
       │
       ▼
┌─────────────────────┐
│  Cloud Run Service  │
│   (Mealie v3.5.0)   │
│  - Auto-scaling     │
│  - Scale-to-zero    │
└──────┬──────────────┘
       │
       ▼
┌─────────────────────┐
│   Cloud SQL         │
│   (PostgreSQL 15)   │
│   - db-f1-micro     │
│   - Auto backups    │
└─────────────────────┘
```

## Additional Resources

- [Mealie Documentation](https://docs.mealie.io/)
- [Cloud Run Documentation](https://cloud.google.com/run/docs)
- [Cloud SQL Documentation](https://cloud.google.com/sql/docs)
- [Artifact Registry Documentation](https://cloud.google.com/artifact-registry/docs)

## License

This deployment guide is provided as-is. Mealie is licensed under the AGPL-3.0 License.

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

---

**Note**: Remember to store all passwords securely. Database passwords are displayed during setup but cannot be retrieved later (except from Secret Manager).
