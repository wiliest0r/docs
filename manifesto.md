# МАНІФЕСТ: Beacon Analytics — Архітектура Хмарної Інфраструктури (GCP, HCP Terraform & Wasm-Spin GKE)

> **Статус:** Офіційний архітектурний маніфест (Active / Living Document)  
> **Призначення:** Єдине джерело істини (Single Source of Truth, SSOT) щодо цільової архітектури, принципів безпеки, інфраструктурного управління, дорожньої карти та стандартів експлуатації платформи аналітики **Beacon Analytics**.

---

## 1. Стратегічні цілі (Goals & Objectives)

### 1.1. Декларативне управління інфраструктурою (No ClickOps & GitOps)
* **Повна автоматизація (IaC через HCP Terraform):** Усі хмарні ресурси GCP створюються та керуються виключно через код у репозиторії [`wiliest0r/terraform-central`](https://github.com/wiliest0r/terraform-central). Жодних ручних модифікацій через консоль GCP («Zero ClickOps»).
* **Ізоляція радіуса ураження (Blast Radius Reduction):** Відмова від монолітного state-файлу. Чітке розділення на ізольовані оточення (`dev`, `stage`, `prod`) з окремими проєктами в GCP та незалежними воркспейсами в HCP Terraform (`PlayTests`).
* **Абсолютна безпарольність (Keyless Security / Zero Static Secrets):** Повна відмова від експорту статичних сервісних ключів у форматі JSON (`Service Account Keys`). Вся автентифікація (як для HCP Terraform, так і для GitHub Actions) базується виключно на **Workload Identity Federation (OIDC)** із короткоживучими токенами доступу.
* **Мінімізація витоку чутливих даних (Decoupled Configuration):** Усі параметри цільової хмари (Project ID, Region, Zone, WIF Providers) передаються виключно через змінні оточення воркспейсів HCP Terraform та Repository Variables у GitHub Actions. Конфігураційні файли репозиторію не містять жорстко зашитих реальних імен проєктів чи облікових даних.

### 1.2. Організаційна ієрархія та фабрика проєктів GCP
* **Організація:** Ресурси розміщуються в межах виділеної організації **`<gcp>-org`** (ID: `455342405415`).
* **Ізольовані проєкти за середовищами:**
  * **Dev:** `playtests-beacon-dev` — пісочниця та середовище швидкої розробки з авто-розгортанням.
  * **Stage:** `playtests-beacon-stage` — середовище передрелізного тестування з незмінними (immutable) OCI-тегами.
  * **Prod:** `playtests-beacon-prod` — продуктивне середовище із суворими approval-гейтами та захистом від видалення.
* **Централізований білінг:** Усі проєкти автоматично підключаються до Cloud Billing Account із вимкненням дефолтних небезпечних мереж (`auto_create_subnetworks = false`).

### 1.3. Високопродуктивний хмарний WebAssembly (Wasm-Spin) у Kubernetes
* **Портативний запуск WASM (Fermyon Spin OCI):** Застосування платформи **Fermyon Spin** у складі керованого кластера **Google Kubernetes Engine (GKE)**. Пакування скомпільованого бінарника (`wasm32-wasip1` + `wasm-opt -Oz`) поверх легкого базового образу `ghcr.io/fermyon/spin:v2.7.0` забезпечує 100% сумісність зі стандартними GKE пулами нод (COS) без необхідності модифікації ядра вузлів чи складних custom containerd shims.
* **Наднизький час холодного старту (< 5 мс):** Миттєва ініціалізація WASM-модулів у пам'яті під навантаженням замість секундного прогріву важких контейнерів Linux.
* **Екстремальний FinOps для Dev:**
  * Використання **Zonal GKE Cluster** (`europe-west1-b`), вартість Control Plane якого ($74.40/міс) на 100% покривається щомісячним кредитом **GCP Free Tier**.
  * Робочі вузли пулу `beacon-node-pool-dev` використовують інстанси **`e2-small`** (0.5–2 vCPU, 2 GB RAM) у режимі **Spot VM** (знижка 60–80%).
  * Параметризований тип комп'юту: прапорець `var.gke_spot_nodes` у Terraform (`true` для Dev, `false` за замовчуванням для Prod).
  * Диски: 20 GB `pd-standard`.
  * Сумарна вартість усієї інфраструктури Dev-середовища: **~$4–$6 на місяць**.
* **Zero-Spike Dev Rolling Update:** Для збереження ультраекономного профілю Dev (1 Spot нода `e2-small`) стратегія оновлення у деплойменті налаштована як `max_surge = 0` та `max_unavailable = 1`. Це дозволяє Kubernetes вивільняти ресурси CPU перед запуском нового поду та запобігає небажаному автоскейлінгу пулу нод під час релізів.

---

## 2. Ключові принципи платформи (Guiding Principles)

| Принцип | Реалізація в Beacon Analytics |
|---|---|
| **Declarative GitOps** | Жодної мутації стану поза Git. Зміни в інфраструктурі розгортаються через гілки `develop`, `stage`, `main` репозиторію `terraform-central`. |
| **Dual-Track Delivery** | Чітке розділення контурів: 1) Інфраструктурний контур (HCP Terraform); 2) Контур застосунку (збірка Web-Tag + WASM Spin, OCI-пуш в Artifact Registry, Rolling Update подів). |
| **Zero-Drift App CD** | Використання `lifecycle.ignore_changes` для поля `image` у специфікації Kubernetes Deployment у Terraform, що усуває конфлікти дрифту між інфраструктурним кодом та безперервним деплоєм додатків. |
| **2-Dimensional Composite Tagging** | Стратегія тегування, що незалежно враховує версії клієнтського JS-трекера та бекенд-сервера (`:dev-b<beacon_sha>-t<tag_sha>` для Dev та `:v<beacon_ver>-tag.<tag_ver>` для релізів). |
| **Zero Static Secrets** | Жодного JSON-ключа сервісного акаунта в репозиторіях чи секретах GitHub. Лише Workload Identity Federation (WIF) за протоколом OpenID Connect (OIDC). |
| **Principle of Least Privilege** | Окремі цільові сервісні акаунти: `sa-terraform-executor` (IaC-виконавець), `sa-gha-deployer` (CI/CD релізи), `sa-beacon-runtime` (запуск подів та пуш у Pub/Sub), `sa-gke-nodes` (читання реєстру та логування). |
| **Extreme FinOps** | Автоматична оптимізація розмірів (Spot VMs, e2-small, Free Tier Zonal кластери, мінімальні розміри сховищ). |
| **Single Source of Truth** | Код репозиторіїв та документація в `docs` є єдиним актуальним джерелом знань про систему. |

---

## 3. Топологія репозиторіїв та GitOps-модель

### 3.1. Структура репозиторіїв проєкту

```mermaid
graph TD
    subgraph AppTrack["Контур застосунку (Application Track)"]
        TagRepo["wiliest0r/web-tag<br/>(JS клієнтський трекер < 5KB)"]
        BeaconRepo["wiliest0r/beacon<br/>(Rust Spin WASM бекенд)"]
        DeployRepo["wiliest0r/deploy<br/>(K8s WASM маніфести та скрипти)"]
    end

    subgraph PlatformTrack["Контур платформи та CI/CD (Platform Track)"]
        TFRepo["wiliest0r/terraform-central<br/>(HCP Terraform IaC + WIF)"]
        GHARepo["wiliest0r/gh-actions-central<br/>(Перевикористовувані workflow)"]
        DocsRepo["wiliest0r/docs<br/>(Архітектура, діаграми та Маніфест)"]
    end

    TagRepo -->|Скрипт t.min.js| BeaconRepo
    BeaconRepo -->|OCI WASM образ| DeployRepo
    TFRepo -->|GKE кластер, VPC, IAM| DeployRepo
    GHARepo -->|DevSecOps гейти| TagRepo
    GHARepo -->|DevSecOps гейти| BeaconRepo
    GHARepo -->|CI/CD пайплайни| TFRepo
```

| Репозиторій | Призначення | Посилання |
| :--- | :--- | :--- |
| **`terraform-central`** | Модулі Terraform та конфігурація середовищ GCP через HCP Terraform | [wiliest0r/terraform-central](https://github.com/wiliest0r/terraform-central) |
| **`beacon`** | Rust WASM/WASI HTTP-сервер для прийому аналітики та роздачі трекера | [wiliest0r/beacon](https://github.com/wiliest0r/beacon) |
| **`web-tag`** | Ультралегкий (< 5 KB) клієнтський скрипт збору метрик | [wiliest0r/web-tag](https://github.com/wiliest0r/web-tag) |
| **`deploy`** | K8s маніфести (Deployment, Service LoadBalancer, RuntimeClass `wasm-spin`) | [wiliest0r/deploy](https://github.com/wiliest0r/deploy) |
| **`gh-actions-central`** | Централізовані шаблони пайплайнів, лінтери, DevSecOps-сканери | [wiliest0r/gh-actions-central](https://github.com/wiliest0r/gh-actions-central) |
| **`docs`** | Архітектурна документація, діаграми, інструкції та цей Маніфест | [wiliest0r/docs](https://github.com/wiliest0r/docs) |

---

### 3.2. Модель воркспейсів HCP Terraform (VCS GitOps)

Інфраструктура репозиторію `terraform-central` організована навколо єдиного дерева коду, де конфігурація для кожного оточення прив'язана до відповідної гілки з автоматичним плануванням та застосуванням:

```mermaid
flowchart LR
    subgraph GitBranches["Гілки Git (terraform-central)"]
        DevBranch["develop"]
        StageBranch["stage"]
        MainBranch["main"]
    end

    subgraph HCPWorkspaces["Воркспейси HCP Terraform (Орг: PlayTests)"]
        WS_Dev["terraform-central-dev"]
        WS_Stage["terraform-central-stage"]
        WS_Prod["terraform-central-prod"]
    end

    subgraph GCPSites["Проєкти GCP (Орг: <gcp>-org)"]
        Proj_Dev["playtests-beacon-dev"]
        Proj_Stage["playtests-beacon-stage"]
        Proj_Prod["playtests-beacon-prod"]
    end

    DevBranch -->|Автоматичний Apply| WS_Dev --> Proj_Dev
    StageBranch -->|Manual Review Gate| WS_Stage --> Proj_Stage
    MainBranch -->|Manual Review Gate| WS_Prod --> Proj_Prod
```

* **`terraform-central-dev`**:
  * **Гілка:** `develop`
  * **VCS Інтеграція:** автоматичний запуск спекулятивного плану на кожен PR та **автоматичний `Apply`** після злиття у гілку `develop` (без очікування ручного підтвердження).
  * **Параметри:** `var.gke_spot_nodes = true`, мінімальні квоти та потужності.
  * **Цільовий проєкт:** `playtests-beacon-dev`.
* **`terraform-central-stage`**:
  * **Гілка:** `stage`
  * **Поведінка:** Speculative Plan на PR, обов'язковий Review Gate (`wiliest0r`) перед застосуванням.
  * **Цільовий проєкт:** `playtests-beacon-stage`.
* **`terraform-central-prod`**:
  * **Гілка:** `main`
  * **Поведінка:** Суворий захист релізу, аудит безпеки, ручне підтвердження перед `apply`. `var.gke_spot_nodes = false` (використання стабільних інстансів).
  * **Цільовий проєкт:** `playtests-beacon-prod`.

---

## 4. Контур доставки додатку (WASM Package CI/CD & GKE Rollout)

### 4.1. Складові та архітектура WASM-пакета
WASM-пакет є єдиною атомарною одиницею доставки, що об'єднує:
1. **`t.min.js`**: мініфікований клієнтський JS-трекер (< 5 KB), що збирається з репозиторію `web-tag` та інжектується у бандл сервера. Віддається ендпоінтом `GET /tag.js` із попередньою динамічною генерацією сесійних конфігурацій `window.__OP_CONFIG__={...}` прямо з пам'яті WASM-рантайму.
2. **`beacon_server-opt.wasm`**: скомпільований Rust-модуль (`wasm32-wasip1`), оптимізований за допомогою `wasm-opt -Oz`, що валідує HMAC-підписи, розбирає телеметрію та формує структурований потік подій на ендпоінт `POST /v1/sync`.
3. **`spin.toml`**: декларативний дескриптор компонентів Fermyon Spin, що маршрутизує вхідні HTTP-запити до відповідних функцій WASM.
4. **OCI Image Ref**: образ на базі `ghcr.io/fermyon/spin:v2.7.0`, збережений у Google Artifact Registry:
   `europe-west1-docker.pkg.dev/playtests-beacon-dev/beacon-repo-dev/beacon-server:<TAG>`

---

### 4.2. Стратегія композитного тегування (Стратегія 1)
Для забезпечення незалежного життєвого циклу клієнтського трекера та бекенд-сервера впроваджено 2-вимірне композитне тегування:

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Dev комміти (гілка develop):                             │
│    :dev-b<beacon_sha>-t<tag_sha>  та  :latest-dev           │
│                                                             │
│ 2. Офіційні релізи (Git Tags):                              │
│    :v<beacon_ver>-tag.<tag_ver>                             │
└─────────────────────────────────────────────────────────────┘
```

* **Незалежний реліз клієнтського тегу:** При випуску нової версії в `wiliest0r/web-tag`, репозиторій відправляє подію `repository_dispatch: web_tag_released` у `wiliest0r/beacon`.
* **Перезбірка без коммітів у сервер:** Пайплайн у `beacon` підтягує новий тег трекера, перепаковує WASM-пакет із новим складеним тегом та оновлює поди в Dev.

---

### 4.3. Автоматизований пайплайн доставки (GitHub Actions + WIF)

```mermaid
sequenceDiagram
    autonumber
    actor Dev as Розробник
    participant Git as GitHub (beacon / web-tag)
    participant GHA as GitHub Actions (OIDC)
    participant WIF as GCP Workload Identity (tfc-pool)
    participant GAR as Google Artifact Registry
    participant GKE as GKE Cluster (beacon-gke-dev)

    Dev->>Git: PR Merge -> develop (або web_tag_released)
    Git->>GHA: Запуск .github/workflows/release.yml
    GHA->>GHA: 1. Збірка web-tag (npm run build -> t.min.js)
    GHA->>GHA: 2. Інжекція t.min.js у static/
    GHA->>GHA: 3. cargo build --release --target wasm32-wasip1
    GHA->>GHA: 4. wasm-opt -Oz оптимізація бінарника
    GHA->>GHA: 5. Розрахунок композитного тегу (:dev-b<sha>-t<sha>)
    GHA->>WIF: 6. OIDC Keyless Exchange (sa-gha-deployer)
    WIF-->>GHA: Короткоживучий GCP Access Token
    GHA->>GAR: 7. Docker OCI Push (:tag та :latest-dev)
    GHA->>GKE: 8. kubectl set image deployment/beacon-server ...
    GKE->>GKE: 9. Zero-Spike Rolling Update (maxSurge=0, maxUnavailable=1)
    GHA->>GKE: 10. kubectl rollout status --timeout=180s
    GHA->>GKE: 11. Live Smoke Tests (/healthz та /tag.js)
```

---

## 5. Безпека та критерії успіху (Security & Verification)

1. **Безпека облікових записів (Zero Credential Leakage):**
   * Жодного статичного ключа `credentials.json` у репозиторіях чи секретах GitHub.
   * WIF-провайдери обмежені суворим зіставленням суб'єкта: `assertion.repository_owner == 'wiliest0r'`.
   * Права `sa-gha-deployer` строго обмежені: `roles/artifactregistry.writer` та `roles/container.developer`.
2. **Нульовий простій (Zero-Downtime Rolling Update):**
   * Оновлення версії коду здійснюється стандартним механізмом Kubernetes RollingUpdate з перевіркою готовності подів (`readinessProbe` / `healthz`).
3. **Економічна ефективність (FinOps Compliance):**
   * Застосування преривних нод (Spot VM) типу `e2-small`.
   * Фактичні витрати на Dev-середовище вкладаються в рамки **$4–$6 на місяць**, а Control Plane покривається GCP Free Tier.
4. **Ідемпотентність та стійкість GitOps:**
   * Повний подвійний контур протестовано: інфраструктурні зміни в `terraform-central` не конфліктують із релізами додатків завдяки блоку `ignore_changes` для образу контейнера.

---

## 6. Поточний статус та дорожня карта (Roadmap)

### 6.1. Статус реалізації етапів

* [x] **Етап 1: Інфраструктурний фундамент та GitOps автоматизація (Milestone 1 — COMPLETED ✅)**
  * Організація GCP, створення проєкту `playtests-beacon-dev`.
  * Workload Identity Federation (WIF) OIDC для HCP Terraform та GitHub Actions.
  * Zonal GKE кластер `beacon-gke-dev` (`europe-west1-b`) на Spot VMs (`e2-small`).
  * Google Artifact Registry `beacon-repo-dev` для OCI WASM-образів.
  * VCS-інтеграція HCP Terraform: авто-план на PR, авто-Apply при мерджі в `develop`.
  * Dockerfile на базі Fermyon Spin (`ghcr.io/fermyon/spin:v2.7.0`).
  * Композитне тегування (Стратегія 1) та крос-репозиторний тригер `web-tag` ➔ `beacon`.
  * Автоматичний Rollout поду `beacon-server` у GKE зі стратегією `max_surge = 0`.
  * Smoke-тестування ендпоінтів `/healthz`, `/tag.js` та валідація HMAC на `/v1/sync`.

* [x] **Етап 2: Multi-Environment (Stage & Prod) & Promotion Governance (Milestone 2 — COMPLETED ✅)**
  * Створення та конфігурація проєкту `playtests-beacon-stage`.
  * Налаштування воркспейсу `terraform-central-stage` в HCP Terraform із підключенням до гілки `stage`.
  * Реалізація пайплайну просування релізів: `develop` ➔ PR у `stage` із Speculative Plan та Review Gate.

* [x] **Етап 3: Pipeline аналітики та стрімінгу подій (Milestone 3 — COMPLETED ✅)**
  * Агностична архітектура зберігання та обробки телеметрії (Lakehouse Ready).
  * Реалізація буферизації подій через sidecar-контейнер **Vector** (`timberio/vector:0.43.0-alpine`).
  * Розділення потоків даних: чистий NDJSON потік подій у `stdout` (shared volume), ізольований від операційного логування.
  * Шина повідомлень **Google Cloud Pub/Sub** (`beacon-events-<env>`) із гарантованою доставкою.
  * **Dead-Letter Queue (DLQ)** топік та підписка (`beacon-events-dlq-<env>`) для ізоляції збійних або пошкоджених повідомлень (після 5 невдалих спроб доставки).
  * Хмарне озеро даних (Lakehouse) у **Google Cloud Storage** з автоматичною партиційною структурою за часом (`events/year=YYYY/month=MM/day=DD/hour=HH/`).
  * Готовність до безшовної аналітичної обробки через **BigQuery External Tables**, **Apache Spark** або **Databricks**.

* [x] **Етап 4: Observability & Alerting (Milestone 4 — COMPLETED ✅)**
  * Політики сповіщень **Google Cloud Monitoring Alert Policies**:
    * Моніторинг черги недоставлених повідомлень DLQ (тригер при появі хоча б одного повідомлення в черзі довше 60 секунд).
    * Моніторинг затримки доставки в Lakehouse (тригер при віці найстарішого непідтвердженого повідомлення > 300 с).
    * Моніторинг стабільності подів GKE (тригер при рестартах контейнерів > 0 за 5 хвилин).
  * Канал сповіщень про інциденти: Email Notification Channel (`alerts@playtests.io`).
  * Спеціалізований Cloud Monitoring Dashboard: **Beacon Telemetry & Lakehouse Dashboard** (DLQ backlog, Lakehouse lag, темп інжекції подій, рестарти подів).
  * Метрики на основі логів (Log-based Metrics): `beacon-ingested-events` та `beacon-quarantined-events`.
  * Експортер Prometheus метрик у Vector на порті `9090` із нативним скрапінгом через **Google Cloud Managed Service for Prometheus (GMP)**.
  * Структуроване JSON-логування операцій сервера в `stderr` із кореляцією спанів **Cloud Trace APM** (підтримка заголовків W3C `traceparent` та `x-cloud-trace-context`).

---

## 7. Архітектура Pipeline стрімінгу та Lakehouse (Stage 3)

```mermaid
flowchart LR
    subgraph GKEPod["GKE Pod: beacon-server"]
        WASM["beacon-server<br/>(Spin WASM)"]
        SharedVol[("Shared Volume<br/>/var/log/beacon/events.log")]
        Vector["vector<br/>(Sidecar)"]
        DiskBuffer[("Vector Buffer<br/>/var/lib/vector")]
    end

    subgraph StreamingBus["Cloud Streaming Bus"]
        PubSub["Google Cloud Pub/Sub<br/>topic: beacon-events"]
        DLQ["Dead-Letter Queue<br/>topic: beacon-events-dlq"]
        LakehouseSub["Pub/Sub GCS Push Subscription"]
    end

    subgraph LakehouseStorage["Analytical Lakehouse & Compute"]
        GCS["Google Cloud Storage Bucket<br/>events/year=YYYY/month=MM/day=DD/hour=HH/"]
        BigQuery["BigQuery External Table<br/>(SQL Engine)"]
        Spark["Apache Spark / Databricks<br/>(Batch & ML Processing)"]
    end

    WASM -->|NDJSON write| SharedVol
    SharedVol -->|File tailing| Vector
    Vector -->|Disk-buffered delivery| DiskBuffer
    DiskBuffer -->|GCP Pub/Sub Sink| PubSub
    PubSub -->|Max 5 retries fail| DLQ
    PubSub -->|Cloud Storage Subscription| LakehouseSub
    LakehouseSub -->|Parquet/JSON write| GCS
    GCS -.-> BigQuery
    GCS -.-> Spark
```

### 7.1. Принципи агностичності та надійності
* **Чітке розділення потоків (Stream Segregation):** Бекенд WASM спрямовує сирі валідовані події телеметрії виключно у `stdout` у форматі NDJSON. Vector відслідковує файл подій на спільному монтованому томі `emptyDir`.
* **Zero Event Loss (On-Disk Buffering):** У разі тимчасової мережевої недоступності або проблем із Pub/Sub API, Vector автоматично буферизує події на локальний диск (`/var/lib/vector`, до 256 MB).
* **Dead-Letter Queue (DLQ):** Повідомлення, які не вдалося обробити після 5 спроб доставки, автоматично скидаються у виділений топік `beacon-events-dlq` без блокування основного стріму.
* **Lakehouse Storage Format:** Дані зберігаються у хмарному бакеті GCS у партиціонованому вигляді за шкалою часу (рік/місяць/день/година). Ця структура є відкритою та сумісною для читання будь-якими розподіленими рушіями обробки даних (BigQuery, Spark, Trino, Databricks).

---

## 8. Observability, Cloud Monitoring, APM & Alerting (Stage 4)

```mermaid
flowchart TD
    subgraph Client["Web Client"]
        Browser["Browser / Mobile App"]
    end

    subgraph BeaconPod["GKE Pod: beacon-server"]
        SpinEngine["WASM Server (Spin)"]
        VectorEngine["Vector Sidecar"]
    end

    subgraph ObservabilityStack["Google Cloud Observability Suite"]
        CloudLogging["Cloud Logging<br/>(Structured JSON stderr)"]
        CloudTrace["Cloud Trace APM<br/>(Distributed Spans)"]
        CloudMonitoring["Cloud Monitoring<br/>(Dashboards & Alert Policies)"]
        GMP["Managed Service for Prometheus<br/>(GMP Scraper)"]
    end

    subgraph AlertChannels["Incident Response"]
        Email["Email Alerts<br/>(alerts@playtests.io)"]
    end

    Browser -->|HTTP + traceparent / x-cloud-trace-context| SpinEngine
    SpinEngine -->|Structured Logs with traceId/spanId| CloudLogging
    CloudLogging -->|Log Correlation| CloudTrace
    SpinEngine -->|Log Metric Filter| CloudMonitoring
    VectorEngine -->|Prometheus Metrics :9090/metrics| GMP
    GMP --> CloudMonitoring
    CloudMonitoring -->|DLQ backlog > 0 / Lag > 300s / Restarts > 0| Email
```

### 8.1. Структуроване логування та кореляція APM/Trace
* **Розділення каналів:** Operational logs надсилаються у `stderr` у форматі JSON із метаданими:
  * `severity`: рівень важливості логу (`INFO`, `WARNING`, `ERROR`).
  * `component`: назва модуля (`beacon-server`).
  * `action`: виконана дія (`serve_tag`, `event_ingested`, `bad_request`, `event_quarantined`).
  * `logging.googleapis.com/trace`: ідентифікатор трейсу GCP, витягнутий із W3C `traceparent` або `x-cloud-trace-context`.
  * `logging.googleapis.com/spanId`: ідентифікатор конкретного спану для повної кореляції у Cloud Trace.

### 8.2. Метрики та наскрізний Prometheus скрапінг
* **Vector Internal Prometheus Exporter:** Vector експортує внутрішні операційні метрики продуктивності пайплайну на порті `9090`:
  * `vector_component_received_events_total`: темп прийому подій.
  * `vector_component_sent_events_total`: темп відправки подій у Pub/Sub.
  * `vector_buffer_sent_bytes_total`: обсяг переданих байтів.
  * `vector_http_client_response_rtt_seconds`: латентність мережевих запитів.
* **GMP Auto-Scraping:** Поди мають стандартні K8s анотації `prometheus.io/scrape = "true"`, `prometheus.io/port = "9090"`, `prometheus.io/path = "/metrics"`, що дозволяє агенту GMP автоматично збирати метрики без додаткових сторонніх агентів.

### 8.3. Політики сповіщень та Dashboard
* **DLQ Backlog Alert Policy:** Негайне сповіщення команди інженерів, якщо кількість непідтверджених повідомлень у DLQ-підписці перевищує 0 протягом 60 секунд.
* **Lakehouse Lag Alert Policy:** Сповіщення при накопиченні затримки (вік найстарішого непідтвердженого повідомлення > 300 секунд), що запобігає затримкам в аналітичних вітринах.
* **Pod Restarts Alert Policy:** Сповіщення у разі будь-якого аварійного перезапуску контейнерів у неймспейсі `beacon` за останні 5 хвилин.
