# microservices-gke

Two small Flask services (users and orders) packaged as Docker images and deployed to Google Kubernetes Engine behind one GCE Ingress, with CPU-based autoscaling and a Cloud Build pipeline. It is a worked example of the GKE deployment path for someone learning it: image build, manifests, ingress routing, HPA, CI.

## What it does not do

- No database. Both services return hard-coded data.
- No service-to-service calls. The two services do not talk to each other.
- No tests, no authentication, no TLS on the ingress.
- No health probes in the manifests.

## Quickstart

Run one service locally (verified with Flask 2.3.2 on Python 3):

```bash
cd user-service
pip install -r requirements.txt
python app.py
curl localhost:8080/users     # {"users":["alice","bob","carol"]}
```

`order-service` is the same, with `GET /orders` returning two sample orders.

Deploying to GKE (not verified in this cleanup: it needs a billed GCP project and Docker, which were not available):

```bash
export PROJECT_ID=my-gke-microservices
gcloud auth configure-docker
docker build -t gcr.io/$PROJECT_ID/user-service:v1 ./user-service
docker push gcr.io/$PROJECT_ID/user-service:v1
docker build -t gcr.io/$PROJECT_ID/order-service:v1 ./order-service
docker push gcr.io/$PROJECT_ID/order-service:v1

gcloud container clusters create microservices-cluster --zone us-central1-a --num-nodes 2 --machine-type n1-standard-1
gcloud container clusters get-credentials microservices-cluster --zone us-central1-a

kubectl apply -f k8s/
kubectl get ingress microservices-ingress -w     # wait for an ADDRESS
curl http://<ADDRESS>/users
```

The Deployment manifests hard-code the image names `gcr.io/my-gke-microservices/...:v1`. If your project ID differs, edit the two `image:` lines first. If `kubectl get hpa` shows `<unknown>` for CPU, the cluster has no metrics server.

## How it works

```
client -> GCE Ingress -+- /users  -> user-service  (ClusterIP :80 -> pod :8080)
                       +- /orders -> order-service (ClusterIP :80 -> pod :8080)
```

- `user-service/` and `order-service/` each hold a single-file Flask app, a `requirements.txt` pinning Flask 2.3.2, and a Dockerfile based on `python:3.11-slim`.
- `k8s/` has a Deployment (2 replicas, 50m CPU request) and a ClusterIP Service per app, one Ingress that routes by path prefix, and two HorizontalPodAutoscalers that scale each Deployment between 2 and 5 replicas at 50% average CPU.
- `cloudbuild.yaml` builds and pushes both images tagged `$SHORT_SHA`, fetches cluster credentials, rewrites the `:v1` tag in the Deployment manifests with `sed`, and runs `kubectl apply`. The cluster name and zone are substitutions (`_CLUSTER_NAME`, `_ZONE`).

## Status

Built in 2025 as a cloud computing project. Archived: no further changes planned.

## Known limits

- `$SHORT_SHA` is only filled in by Cloud Build when a build comes from a trigger, so a manual `gcloud builds submit` would need that substitution passed in. Not verified.
- The `sed` step in `cloudbuild.yaml` only matches image names containing the build's `$PROJECT_ID`. Because the manifests hard-code `my-gke-microservices`, it does nothing in any other project.
- Container Registry (`gcr.io`) has been superseded by Artifact Registry; the image paths would need updating.

## License

MIT, see [LICENSE](LICENSE).
