# WhatsNewAsia GCP Deployment

## Project Info
- **Project ID:** gda-p01
- **Region:** asia-southeast2
- **Backend URL:** https://whatsnewasia-backend-850916858221.asia-southeast2.run.app
- **Frontend URL:** https://whatsnewasia-frontend-850916858221.asia-southeast2.run.app

## View Logs

```bash
# Backend logs
gcloud run services logs read whatsnewasia-backend --region asia-southeast2

# Frontend logs
gcloud run services logs read whatsnewasia-frontend --region asia-southeast2
```

## Redeploy Backend

```bash
# Build and push image (run from project root)
gcloud builds submit ./backend/whatsnewasia_be_revision \
  --tag asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/backend \
  --region=asia-southeast2

# Deploy to Cloud Run
gcloud run deploy whatsnewasia-backend \
  --image asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/backend \
  --region asia-southeast2
```

## Redeploy Frontend

```bash
# Build and push image (run from project root)
gcloud builds submit ./frontend/whatsnewasia_fe_revision \
  --tag asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/frontend \
  --region=asia-southeast2

# Deploy to Cloud Run
gcloud run deploy whatsnewasia-frontend \
  --image asia-southeast2-docker.pkg.dev/gda-p01/whatsnewasia-repo/frontend \
  --region asia-southeast2
```

## Update Environment Variables

```bash
# Update backend env vars
gcloud run services update whatsnewasia-backend \
  --region asia-southeast2 \
  --update-env-vars "KEY=value"

# View current env vars
gcloud run services describe whatsnewasia-backend \
  --region asia-southeast2 \
  --format='yaml(spec.template.spec.containers[0].env)'
```

## Get Service URLs

```bash
gcloud run services describe whatsnewasia-backend --region asia-southeast2 --format='value(status.url)'
gcloud run services describe whatsnewasia-frontend --region asia-southeast2 --format='value(status.url)'
```
