# Ручная настройка через gcloud

Эта инструкция описывает ручное создание общей сети и региональных TCP proxy балансировщиков. Примеры команд приведены для PowerShell: каждый параметр находится на отдельной строке. Для Bash замените обратные апострофы продолжения строк на обратные слэши, а присваивания PowerShell-переменных адаптируйте под Bash.

```powershell
$PROJECT_ID = "vpn-cdn"
$REGION = "europe-west4"
$DNS_ZONE = "cdn2954732"
```

## Первичная настройка проекта

При необходимости [создайте Google Cloud project](https://console.cloud.google.com/projectcreate). Чтобы выбрать существующий проект в консоли, откройте [Project selector](https://console.cloud.google.com/projectselector2/home/dashboard). Включите [Compute Engine API](https://console.cloud.google.com/apis/library/compute.googleapis.com?project=PROJECT_ID) и [Cloud DNS API](https://console.cloud.google.com/apis/library/dns.googleapis.com?project=PROJECT_ID), заменив `PROJECT_ID` на ID своего проекта. Перед созданием тарифицируемых ресурсов подключите [billing account](https://console.cloud.google.com/billing/linkedaccount?project=PROJECT_ID).

```powershell
gcloud projects create "$PROJECT_ID"
gcloud config set project "$PROJECT_ID"
gcloud services enable compute.googleapis.com dns.googleapis.com
```

Установите [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk), затем войдите в аккаунт и выберите проект:

```powershell
gcloud init
gcloud config set project "$PROJECT_ID"
```

В консоли эти ресурсы находятся в разделе [VPC networks](https://console.cloud.google.com/networking/networks/list?project=PROJECT_ID). Создайте общую VPC и proxy-only subnet:

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

Зарезервированный [исходящий IP](https://console.cloud.google.com/networking/addresses/list?project=PROJECT_ID) появится в разделе Addresses. Cloud Router находится в [Cloud Routers](https://console.cloud.google.com/net-services/cloudrouter/list?project=PROJECT_ID), NAT gateway — в [Cloud NAT](https://console.cloud.google.com/net-services/nat/list?project=PROJECT_ID). Создайте их командами:

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

Существующая публичная [Cloud DNS зона `cdn2954732`](https://console.cloud.google.com/net-services/dns/zones/cdn2954732/details?project=vpn-cdn) должна находиться в этом проекте. Для другой зоны откройте [Cloud DNS Zones](https://console.cloud.google.com/net-services/dns/zones?project=PROJECT_ID) и измените `$DNS_ZONE` в начале инструкции. Проверьте имя зоны командой:

```powershell
gcloud dns managed-zones describe "$DNS_ZONE" `
  --project="$PROJECT_ID"
```

У регистратора домена укажите NS-серверы, показанные для этой managed zone. Не создавайте вторую зону, если домен уже делегирован на эту.

## Добавление балансировщика

Задайте параметры нового балансировщика: уникальное имя, публичный IP backend-а и уникальное DNS-имя внутри делегированной зоны. В конце полного DNS-имени нужна точка.

```powershell
$BALANCER_NAME = "balancer1"
$BACKEND_IP = "212.118.36.11"
$BACKEND_PORT = "445"
$FRONTEND_PORT = "443"
$DNS_NAME = "balancer1.cdn2954732.de5.net."
```

Региональные Internet NEG отображаются в разделе [Network Endpoint Groups](https://console.cloud.google.com/compute/networkendpointgroups/list?project=PROJECT_ID). Создайте NEG для backend-а:

```powershell
gcloud beta compute network-endpoint-groups create "${BALANCER_NAME}-neg" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --network-endpoint-type="INTERNET_IP_PORT" `
  --network="lb-network"
```

Зарезервируйте входящий IP балансировщика:

```powershell
gcloud compute addresses create "${BALANCER_NAME}-ip" `
  --project="$PROJECT_ID" `
  --description="IP for ingress balancer traffic" `
  --network-tier="STANDARD" `
  --region="$REGION"
```

Создайте backend service, target TCP proxy и forwarding rule. Конфигурация балансировщиков доступна в разделе [Load balancing](https://console.cloud.google.com/net-services/loadbalancing/list?project=PROJECT_ID):

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

Создайте TCP health check и привяжите его к backend service:

```powershell
gcloud compute health-checks create tcp "${BALANCER_NAME}-health-check" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --port="$BACKEND_PORT" `
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

Добавьте IP и порт backend-а в NEG:

```powershell
gcloud compute network-endpoint-groups update "${BALANCER_NAME}-neg" `
  --project="$PROJECT_ID" `
  --region="$REGION" `
  --add-endpoint="ip=${BACKEND_IP},port=${BACKEND_PORT}"
```

Создайте DNS A-запись, указывающую на зарезервированный ingress IP. Наборы записей и их значения видны на странице [DNS zone details](https://console.cloud.google.com/net-services/dns/zones/cdn2954732/details?project=vpn-cdn):

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

Если A RRset с таким DNS-именем уже существует, используйте `gcloud dns record-sets update` с теми же параметрами вместо `create`. Эта команда обновляет весь набор A-записей для hostname.

Проверьте DNS-запись и IP forwarding rule:

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
