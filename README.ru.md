# CDN VPN: балансировщики нагрузки

Настройка через Ansible региональных TCP proxy балансировщиков Google Cloud с backend-ами типа Internet NEG. Балансировщик принимает TCP-соединения клиентов на публичный IP и перенаправляет их на заданные FQDN и порт backend-а.

## Схема сетевого взаимодействия

```mermaid
flowchart LR
  Client[Клиент] -->|DNS запрос| DNS[Cloud DNS A-запись]
  DNS -->|Возвращает ingress IP| FR[Региональное forwarding rule TCP]
  FR --> Proxy[Региональный target TCP proxy]
  Proxy --> BS[Региональный backend service]
  BS --> NEG[Региональный Internet NEG FQDN:port]
  NEG --> NAT[Cloud NAT подменяет исходящий IP на статический]
  NAT --> Backend[Backend 3x-ui]
  Health[Региональная TCP health check] -.->|Проверяет порт backend-а| Backend
```

Cloud DNS преобразует hostname из inventory в зарезервированный ingress IP балансировщика. TCP proxy передаёт соединение дальше, не завершая TLS. Google Cloud подключается к endpoint backend-а; Cloud NAT обеспечивает заданный статический исходящий IP. Health check отдельно проверяет доступность backend-а.

## Запуск Ansible

Запускайте playbook из корня репозитория в Linux. Установите [Google Cloud CLI](https://cloud.google.com/sdk/docs/install-sdk).

```bash
sudo apt update
sudo apt install ansible-core
```

Установите Google Cloud CLI из apt-репозитория Google:

```bash
sudo apt install apt-transport-https ca-certificates gnupg curl
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor --yes -o /usr/share/keyrings/cloud.google.gpg
echo "deb [signed-by=/usr/share/keyrings/cloud.google.gpg] https://packages.cloud.google.com/apt cloud-sdk main" | sudo tee /etc/apt/sources.list.d/google-cloud-sdk.list >/dev/null
sudo apt update
sudo apt install google-cloud-cli
```

После установки зависимостей задайте проект, войдите в Google Cloud CLI и запустите playbook:

```bash
gcloud auth login --no-launch-browser
# идем по предоставленной ссылке чтобы получить токен для аутентификации, авторизуем токеном gcloud консоль


export GCP_PROJECT_ID="vpn-cdn"
gcloud config set project "$GCP_PROJECT_ID"
ansible-playbook ansible/playbook.yml
```

Предварительно включите Compute Engine API и Cloud DNS API. У аутентифицированного Google-аккаунта должны быть роли `roles/compute.networkAdmin`, `roles/compute.loadBalancerAdmin`, `roles/dns.admin` и `roles/serviceusage.serviceUsageConsumer` в проекте.

Ansible выполняет каждую операцию `gcloud` отдельной задачей. Он проверяет ресурсы по имени, пропускает существующие без сравнения или изменения их полей и создаёт отсутствующие. Ресурсы, удалённые из inventory, автоматически не удаляются. GitHub Actions, ключ service account и постоянное хранилище состояния не используются.

Для ручной настройки используйте [инструкцию через `gcloud`](MANUAL_SETUP.ru.md).

## Настройка inventory

Файл inventory: [`config/inventory.yaml`](config/inventory.yaml). Замените демонстрационные значения на собственные. Добавьте запись в `proxy_subnet_cidrs` для каждого региона и запись в `balancers` для каждого балансировщика:

```yaml
dns_managed_zone: my-public-zone

proxy_subnet_cidrs:
  europe-west4: 10.0.0.0/24

router_asn: 64514
health_check_port: 445

balancers:
  edge1:
    region: europe-west4
    dns_name: edge1.example.net.
    backend_fqdn: backend1.example.org
    backend_port: 445
    frontend_port: 443
```

| Параметр | Назначение |
| --- | --- |
| `dns_managed_zone` | Имя существующей Cloud DNS managed zone в проекте. |
| `proxy_subnet_cidrs` | Map «регион → CIDR proxy-only subnet». Добавьте запись для каждого региона; CIDR не должен пересекаться с другими подсетями VPC. |
| Ключ в `balancers` | Уникальное базовое имя балансировщика. Скрипт формирует по нему имена NEG, backend service, health check, target proxy, forwarding rule и ingress IP. |
| `balancers.<name>.region` | Регион балансировщика. Балансировщики в одном регионе используют общие proxy-only subnet, router, NAT и egress IP этого региона. |
| `balancers.<name>.dns_name` | Уникальное полное DNS-имя A-записи балансировщика. Укажите hostname в выбранной зоне и завершающую точку. |
| `balancers.<name>.backend_fqdn` | Публичное DNS-имя backend-сервера, зарегистрированного в Internet NEG. |
| `balancers.<name>.backend_port` | Порт backend endpoint-а. |
| `balancers.<name>.frontend_port` | TCP-порт для клиентских подключений; по умолчанию `443`. |

`GCP_PROJECT_ID` читается из локальной переменной окружения. `router_asn` и `health_check_port` задаются в YAML inventory. Для запуска Ansible на управляющей машине требуется Python.

Чтобы добавить балансировщик, добавьте новый уникальный ключ в `balancers` и заполните его именованные поля. Для нового региона добавьте запись в `proxy_subnet_cidrs`. Лимит соединений backend-а зафиксирован в Ansible-задаче на 1000 и не настраивается через inventory.

Скрипт не сверяет поля уже существующих ресурсов. Изменения и удаление существующей инфраструктуры выполняйте вручную через `gcloud`; одного изменения inventory недостаточно.
