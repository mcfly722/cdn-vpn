# Manual setup with gcloud

This guide creates the shared network and regional TCP proxy balancers manually. Examples use PowerShell and place one option on each line. For Bash, replace continuation backticks with backslashes and adapt the PowerShell variable assignments.

```powershell
$PROJECT_ID = "vpn-cdn"
$REGION = "europe-west4"
$DNS_ZONE = "cdn2954732"
```

## One-time project setup

Create a Google Cloud project if needed, select it in gcloud, and enable the required APIs. Link a billing account to the project in the Google Cloud console before creating billable resources.

```powershell
gcloud projects create "$PROJECT_ID"
gcloud config set project "$PROJECT_ID"
gcloud services enable compute.googleapis.com dns.googleapis.com
```

Install the [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk), then authenticate and select the project:

```powershell
gcloud init
gcloud config set project "$PROJECT_ID"
```

Create the shared VPC network and proxy-only subnet:

```powershell
gcloud compute networks create "lb-network" `
  --project="$PROJECT_ID" `
  --subnet-mode="custom" `
  --bgp-routing-mode="regional" `
  --bgp-best-path-selection-mode="legacy"

gcloud compute networks subnets create "lb-subnet" `
  --project="$PROJECT_ID" `
  --network="lb-network" `
  --region="$REGION" `
  --purpose="REGIONAL_MANAGED_PROXY" `
  --role="ACTIVE" `
  --range="10.0.0.0/24"
```

Reserve the shared egress IP and create the Cloud Router and NAT gateway:

```powershell
gcloud compute addresses create "lb-router-ip" `
  --project="$PROJECT_ID" `
  --description="IP for egress router traffic" `
  --network-tier="STANDARD" `
  --region="$REGION"

gcloud compute routers create "lb-router" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --network="lb-network" `
  --asn=64514

gcloud compute routers nats create "lb-nat" `
  --project="$PROJECT_ID" `
  --router="lb-router" `
  --region="$REGION" `
  --endpoint-types="ENDPOINT_TYPE_MANAGED_PROXY_LB" `
  --nat-custom-subnet-ip-ranges="lb-subnet" `
  --nat-external-ip-pool="lb-router-ip"
```

The existing public Cloud DNS zone is `cdn2954732`. Confirm it is in this project and note its DNS name:

```powershell
gcloud dns managed-zones describe "$DNS_ZONE" `
  --project="$PROJECT_ID"
```

At the domain registrar, delegate the domain to the NS servers shown for this managed zone. Do not create a second zone if this one is already delegated.

## Add a balancer

Set values for the new balancer. Use a unique name, a public backend FQDN, and a unique hostname inside the delegated DNS zone. The DNS hostname includes the required trailing dot.

```powershell
$BALANCER_NAME = "balancer1"
$BACKEND_FQDN = "v1025101.hosted-by-vdsina.ru"
$BACKEND_PORT = "445"
$HEALTH_CHECK_PORT = "445"
$FRONTEND_PORT = "443"
$DNS_NAME = "balancer1.cdn2954732.de5.net."
```

Create a regional Internet NEG for this backend:

```powershell
gcloud beta compute network-endpoint-groups create "${BALANCER_NAME}-neg" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --network-endpoint-type="INTERNET_FQDN_PORT" `
  --network="lb-network"
```

Reserve the balancer's client-facing IP:

```powershell
gcloud compute addresses create "${BALANCER_NAME}-ip" `
  --project="$PROJECT_ID" `
  --description="IP for ingress balancer traffic" `
  --network-tier="STANDARD" `
  --region="$REGION"
```

Create the backend service, target TCP proxy, and forwarding rule:

```powershell
gcloud compute backend-services create "${BALANCER_NAME}-lb" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --load-balancing-scheme="EXTERNAL_MANAGED" `
  --protocol="TCP" `
  --timeout="30s" `
  --connection-draining-timeout="300s" `
  --session-affinity="NONE" `
  --locality-lb-policy="ROUND_ROBIN"

gcloud compute target-tcp-proxies create "${BALANCER_NAME}-lb-target-proxy" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --backend-service="${BALANCER_NAME}-lb" `
  --proxy-header="NONE"

gcloud compute forwarding-rules create "${BALANCER_NAME}-lb-forwarding-rule" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --load-balancing-scheme="EXTERNAL_MANAGED" `
  --network="lb-network" `
  --network-tier="STANDARD" `
  --ip-protocol="TCP" `
  --ports="$FRONTEND_PORT" `
  --address="${BALANCER_NAME}-ip" `
  --target-tcp-proxy="${BALANCER_NAME}-lb-target-proxy" `
  --target-tcp-proxy-region="$REGION"
```

Create a TCP health check and attach it to the backend service:

```powershell
gcloud compute health-checks create tcp "${BALANCER_NAME}-health-check" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --port="$HEALTH_CHECK_PORT" `
  --timeout=5s `
  --check-interval=5s `
  --healthy-threshold=2 `
  --unhealthy-threshold=3

gcloud compute backend-services update "${BALANCER_NAME}-lb" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --health-checks="${BALANCER_NAME}-health-check" `
  --health-checks-region="$REGION"
```

Register the backend FQDN and port in the NEG:

```powershell
gcloud compute network-endpoint-groups update "${BALANCER_NAME}-neg" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --add-endpoint="fqdn=${BACKEND_FQDN},port=${BACKEND_PORT}"
```

Create the Cloud DNS A record using the reserved ingress IP:

```powershell
$BALANCER_IP = gcloud compute addresses describe "${BALANCER_NAME}-ip" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --format="get(address)"

gcloud dns record-sets create "$DNS_NAME" `
  --project="$PROJECT_ID" `
  --zone="$DNS_ZONE" `
  --type="A" `
  --ttl=300 `
  --rrdatas="$BALANCER_IP"
```

If that DNS name already has an A record set, use `gcloud dns record-sets update` with the same arguments instead of `create`. This updates the entire A record set for that hostname.

Verify the DNS record and forwarding-rule IP:

```powershell
gcloud dns record-sets describe "$DNS_NAME" `
  --project="$PROJECT_ID" `
  --zone="$DNS_ZONE" `
  --type="A"

gcloud compute forwarding-rules describe "${BALANCER_NAME}-lb-forwarding-rule" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --format="get(IPAddress)"
```
