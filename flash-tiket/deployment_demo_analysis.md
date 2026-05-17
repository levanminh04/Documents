# Flash Ticket System - Phan tich deploy demo public URL

> Ngay cap nhat: 2026-05-17  
> Muc tieu: deploy de demo co URL public, chua toi uu production ve performance, HA hay security.  
> Pham vi: Full E2E demo gom frontend, gateway, cac Spring service, Keycloak, RabbitMQ, PostgreSQL/pgvector, MongoDB, Redis. Discovery-service co the bat neu co API key AI va ngan sach.

---

## 1. Ket luan nhanh

Phuong an thuc te nhat cho demo full E2E cua du an nay la **Docker tren mot cloud VM** truoc, sau do moi tinh ECS/Fargate, Cloud Run, Azure Container Apps hoac Kubernetes.

Ly do:

- Du an dang la microservice-like stack voi nhieu process: `configserver`, `eureka`, `apigateway`, `core-service`, `user-service`, `discovery-service`, frontend, Keycloak, RabbitMQ, PostgreSQL/pgvector, MongoDB, Redis.
- Repo hien tai chi co `docker-compose.yml` cho infra, **chua co Dockerfile cho cac Spring service/front-end**.
- Keycloak co custom RabbitMQ SPI JAR mount tu `keycloak-rabbitmq-spi/target/keycloak-rabbitmq-spi.jar`; day la case phu hop Docker hon buildpack/PaaS don gian.
- Config hien tai con nhieu `localhost`, IP EC2 cu va absolute path Windows, nen neu dua tung service len PaaS se phai sua config nhieu hon.
- Free tier/PaaS co the host 1-2 service rat de, nhung full E2E cua du an nay co qua nhieu process always-on, de bi vuot free tier hoac phai tach tung service rat phuc tap.

Khuyen nghi thuc thi theo 3 bac:

| Bac | Phuong an | Khi nao dung | Danh gia |
| --- | --- | --- | --- |
| Demo nhanh nhat | 1 VM + Docker Compose + Caddy/Nginx + HTTPS | Can public URL trong vai ngay, chap nhan single-node | **Khuyen nghi cho demo hien tai** |
| Cloud chuan hon | AWS ECS/Fargate + RDS + ElastiCache + Amazon MQ/CloudAMQP + MongoDB Atlas | Can giam van hanh VM, chuan bi production | Tot nhung ton chi phi va setup dai |
| Free/student toi da | Oracle Cloud VM hoac Heroku/Render/Railway/Koyeb ket hop managed free tiers | Muon tiet kiem chi phi, chap nhan gioi han free tier | Co the lam, nhung de gap gioi han RAM/service |

---

## 2. Ban do deploy hien tai cua du an

### 2.1 Thanh phan can deploy

| Nhom | Thanh phan | Cong local hien tai | Vai tro | Bat buoc cho demo? |
| --- | --- | ---: | --- | --- |
| Frontend | Vite React app | `5173` dev, static build khi deploy | UI cho buyer/organizer/admin | Co |
| Edge/API | `apigateway` | `8080` | Public API entrypoint, route den service noi bo | Co |
| Config | `configserver` | `8888` | Cap config tap trung cho services | Co neu giu kien truc hien tai |
| Discovery | `eureka` | `8761` | Service discovery cho `lb://CORE-SERVICE`, `lb://USER-SERVICE`, `lb://DISCOVERY-SERVICE` | Co neu giu gateway routing hien tai |
| Core domain | `core-service` | `8081` | Event, booking, payment, ticket, promotion | Co |
| User domain | `user-service` | `8082` | User, organizer, Keycloak sync, MongoDB | Co |
| AI/domain discovery | `discovery-service` | `8085` | Chat/RAG/discovery, can Gemini/OpenAI key | Tuy chon cho demo neu thieu API key |
| Auth | Keycloak 26 | `9090 -> 8080` container | Login, JWT issuer, roles | Co |
| Broker | RabbitMQ | `5672`, `15672` | Keycloak event sync, ticket/email async, bus AMQP | Co |
| SQL | PostgreSQL + pgvector | `5432` | Core schemas + discovery vector/schema | Co |
| NoSQL | MongoDB | `27018 -> 27017` | User/organizer data | Co |
| Cache/lock | Redis | `6379` | Rate limit, distributed lock, payment/booking guards | Co |
| Kafka/ZooKeeper | Kafka, ZooKeeper | `9092`, `2181` | Dang comment trong config, khong thay code dung Kafka | **Khong can cho demo** |
| Admin tools | pgAdmin, RabbitMQ management UI | `5050`, `15672` | Quan tri/dev debug | Khong nen public rong rai |

### 2.2 Luong request public nen co

```text
Browser
  -> Frontend public URL
  -> API Gateway public URL (/api/**)
  -> Internal services qua Eureka/load balancer
  -> PostgreSQL / MongoDB / Redis / RabbitMQ noi bo

Browser
  -> Keycloak public URL (/realms/flash-ticket/**)
  -> JWT
  -> API Gateway validate issuer/JWKS
```

Voi demo, chi nen public:

- Frontend: `https://app.<domain>`
- Gateway: `https://api.<domain>`
- Keycloak: `https://auth.<domain>`
- Tuy chon admin tam thoi co basic auth/IP allowlist: `https://rabbit.<domain>`, `https://pgadmin.<domain>`

Khong nen public truc tiep:

- `core-service`, `user-service`, `discovery-service`
- PostgreSQL, MongoDB, Redis, AMQP RabbitMQ
- Eureka, Config Server

---

## 3. Deploy blockers hien tai trong repo

Day la cac diem can xu ly truoc khi demo public URL. Khong phai tat ca deu la production blocker, nhung neu bo qua thi kha nang cao deploy se fail hoac login/API bi loi.

| Blocker | Dang o dau | Tac dong khi deploy | Cach xu ly cho demo |
| --- | --- | --- | --- |
| Chua co Dockerfile cho app services | Root chi co `docker-compose.yml`, khong co `Dockerfile` app | Khong build/run duoc backend bang container mot cach lap lai | Tao Dockerfile cho tung Spring service va frontend, hoac 1 Dockerfile param hoa bang `SERVICE_DIR` |
| Java version lech | `apigateway`, `core-service`, `user-service`, `configserver` dung Java 24; `discovery-service` Java 21; `eureka` Java 17; local JDK la 21 | Build bang JDK 21 co the fail voi module khai bao Java 24 | Demo nhanh: build Java 24 services bang image JDK 24. Tot hon: ha ve Java 21 LTS neu code khong dung feature Java 24 |
| Config Server native path la Windows absolute path | `configserver/src/main/resources/application.yml` tro `file:/D:/Project/...` | Container/Linux khong co path nay, Config Server khong doc duoc config | Doi sang `classpath:/config/` hoac env `CONFIG_SEARCH_LOCATIONS`; demo container nen dung `classpath:/config/` |
| Services import Config Server bang localhost | `*/src/main/resources/application.yml` co `optional:configserver:http://localhost:8888` | Trong Docker, `localhost` la container cua chinh service, khong phai Config Server | Doi thanh `${CONFIG_SERVER_URL:http://localhost:8888}` va set `CONFIG_SERVER_URL=http://configserver:8888` |
| `core-service.yml` hardcode Eureka localhost | `configserver/.../core-service.yml` | Core service trong Docker khong dang ky duoc Eureka neu localhost | Doi thanh `${EUREKA_SERVER_URL:http://localhost:8761/eureka/}` |
| Gateway CORS chi allow localhost | `apigateway/.../SecurityConfig.java` | FE public URL se bi CORS block khi goi API | Externalize allowed origins, them `https://app.<domain>` |
| FE Keycloak URL rewrite | `frontend/src/main.tsx` rewrite request bat dau bang `VITE_KEYCLOAK_URL` thanh relative path | Neu FE static host khac domain voi Keycloak, request Keycloak co the bi rewrite sai | Hoac bo rewrite trong prod, hoac dung same-origin reverse proxy cho `/realms/**` ve Keycloak |
| Keycloak redirect/web origins con local | Keycloak realm/client config | Login redirect khong ve duoc FE public URL | Set Valid Redirect URIs: `https://app.<domain>/*`; Web Origins: `https://app.<domain>` |
| DB migration manual | Flyway dang comment, SQL nam o `database/postgresql` | DB moi khong co schema neu khong import thu cong | Demo: chay SQL manual theo thu tu. Sau demo: bat Flyway va xu ly duplicate version `V6` |
| Co 2 file migration `V6__...` | `database/postgresql/V6__init_discovery_schema.sql`, `V6__event_layout_tables.sql` | Neu bat Flyway se fail duplicate version | Rename mot file thanh `V7__...` truoc khi dung Flyway |
| Secret/default dev con yeu | RabbitMQ guest/guest, Keycloak admin/admin, Redis no password | Demo public co rui ro bi truy cap neu expose | Khong expose port infra ra internet; set password that trong `.env` |

---

## 4. Docker co nen dung khong?

Co. Voi du an nay, Docker la lua chon thuc dung nhat.

### 4.1 Neu khong dung Docker

Ban se phai:

- Cai dung Java 24/21/17 tren server.
- Build va chay tung JAR bang systemd.
- Cai Keycloak va copy SPI JAR dung thu muc provider.
- Cai PostgreSQL co pgvector, Redis, RabbitMQ, MongoDB.
- Quan ly env var rieng cho tung process.
- Dam bao startup order: infra -> configserver -> eureka -> services -> gateway.

Cach nay van lam duoc, nhung kho lap lai, kho debug, va rat de sai path/config.

### 4.2 Dung Docker Compose tren 1 VM

Phu hop demo nhat:

- Mot file compose quan ly toan bo process.
- Networking noi bo bang service name: `postgres`, `mongo`, `rabbitmq`, `redis`, `keycloak`, `configserver`, `eureka`.
- Reverse proxy Caddy/Nginx dung chung 80/443.
- De backup, tear down, rebuild, chuyen VM.

Nhuoc diem:

- Single point of failure.
- Scale tung service khong dep.
- Secret management thua managed cloud.
- Can RAM kha nhieu; VM 1GB/2GB gan nhu khong du cho full E2E.

### 4.3 Dung Docker tren PaaS

Heroku, Render, Railway, Koyeb, Fly.io deu ho tro container o cac muc do khac nhau. Van de khong nam o Docker, ma nam o so luong service:

- Moi Spring service la mot web/private service rieng.
- Keycloak la mot service rieng.
- RabbitMQ/Redis/Postgres/Mongo can managed add-on hoac service rieng.
- Eureka/Config Server nen la private services.
- Tong so instance always-on de vuot free/cheap tier.

PaaS phu hop hon neu ban rut gon kien truc: bo Eureka/Config Server, thay bang URL/env truc tiep, hoac gop backend cho demo.

---

## 5. Phan tich cac nen tang deploy

### 5.1 AWS

#### Phuong an A: EC2 + Docker Compose

Day la huong AWS nhanh nhat cho demo.

Kien truc:

```text
Route 53 / DNS
  -> EC2 public IP
  -> Caddy/Nginx 80/443
      app.<domain>  -> frontend static
      api.<domain>  -> apigateway:8080
      auth.<domain> -> keycloak:8080
      rabbit.<domain> / pgadmin.<domain> -> optional admin, protect bang basic auth/IP allowlist

Docker network private:
  postgres, mongo, redis, rabbitmq, keycloak,
  configserver, eureka, core-service, user-service, discovery-service, apigateway
```

May goi y:

- Toi thieu de full E2E: 2 vCPU, 4GB RAM, discovery-service co the bat nhung se chat.
- Thoai mai hon: 4 vCPU, 8GB RAM.
- Free tier EC2 micro khong phu hop full stack nay vi Keycloak + 6 Spring processes + databases qua nang.

Uu diem:

- It can sua kien truc nhat.
- De public URL.
- De dung dung compose hien co va mo rong.
- Chi phi ro rang neu dung 1 VM.

Nhuoc diem:

- Het credits/free tier la mat phi lien tuc.
- Tu quan ly backup, security group, patch OS, disk.
- Neu server restart, tat ca cung down.

Khuyen nghi cho demo AWS:

1. Dung EC2 Ubuntu 22.04/24.04.
2. Gan Elastic IP neu can URL on dinh.
3. Security group chi mo `80`, `443`, `22` theo IP cua ban.
4. Khong mo `5432`, `27017`, `6379`, `5672`, `8761`, `8888`, `8081`, `8082`, `8085` ra internet.
5. Dat Caddy hoac Nginx lam reverse proxy HTTPS.

#### Phuong an B: ECS/Fargate + managed services

Day la huong cloud-standard hon.

Thanh phan:

- ECS/Fargate service:
  - `apigateway`
  - `configserver`
  - `eureka`
  - `core-service`
  - `user-service`
  - `discovery-service`
  - `keycloak`
- RDS PostgreSQL voi pgvector neu region/engine ho tro extension.
- ElastiCache Redis.
- Amazon MQ for RabbitMQ hoac CloudAMQP.
- MongoDB Atlas thay DocumentDB, vi app dung MongoDB semantics va Atlas co free M0.
- ALB public cho gateway/keycloak/frontend.

Uu diem:

- Gan voi production hon.
- Co health check, restart, logs, IAM, secrets.
- De tach public/private networking.

Nhuoc diem:

- Setup lau hon nhieu.
- Fargate + ALB + managed DB/cache/broker de ton tien hon 1 VM.
- Eureka trong ECS khong con qua can thiet neu dung Cloud Map/Service Connect, nhung neu giu code hien tai thi van phai chay.

Danh gia: tot cho giai doan sau demo, chua phai duong ngan nhat.

#### AWS Free Tier/credits

AWS Free Tier da thay doi cho account moi sau 2025-07-15: account moi co the chon Free account plan/Paid account plan, co credits ban dau va thoi han Free plan toi da 6 thang hoac den khi het credits, tuy account/plan. Account cu truoc moc nay van lien quan toi free tier 12 thang cho mot so dich vu. Can kiem tra Billing Console cua chinh tai khoan, khong nen gia dinh EC2/RDS con free.

Nguon: [AWS Free Tier](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier.html), [AWS Free Tier eligibility](https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-eligibility.html).

### 5.2 Google Cloud

#### Cloud Run

Cloud Run rat tot cho container stateless:

- Gateway, core-service, user-service, discovery-service co the chay Cloud Run.
- Co scale-to-zero de tiet kiem voi traffic demo.
- Pricing tinh theo usage, co free tier request/CPU/RAM hang thang.

Nhung du an nay co cac thanh phan stateful/long-running:

- Keycloak can DB/persistent config va URL issuer on dinh.
- RabbitMQ can broker stateful.
- Redis can cache/lock stateful.
- PostgreSQL/pgvector, MongoDB can managed DB.
- Eureka/Config Server la long-running internal services.

Neu len Cloud Run, nen doi kien truc:

- Bo Eureka cho demo, route gateway toi URL Cloud Run truc tiep, hoac dung service discovery noi bo cua cloud.
- Bo Config Server, inject env var truc tiep hoac dung Secret Manager.
- Keycloak chay tren VM/GKE/Cloud Run with external DB, nhung can cau hinh proxy/hostname ky.

Danh gia:

- Tot neu ban san sang refactor config/deploy model.
- Khong nhanh bang VM Compose cho demo full E2E.

Nguon: [Cloud Run pricing](https://cloud.google.com/run/pricing).

### 5.3 Azure

Lua chon:

- Azure Container Apps cho backend containers.
- Azure App Service for Containers cho tung app.
- Azure Database for PostgreSQL.
- Azure Cache for Redis.
- MongoDB Atlas tren Azure hoac Cosmos DB Mongo API.
- RabbitMQ tu host container/VM hoac ben thu ba.

Uu diem:

- Container Apps co revision, scale, managed ingress.
- De ket hop managed Postgres/Redis.

Nhuoc diem:

- RabbitMQ va Keycloak van la phan phuc tap.
- Neu moi hoc cloud, Azure networking + Container Apps + managed identity khong ngan hon AWS EC2.
- Free account/credits thay doi theo chuong trinh, can kiem tra tai khoan.

Danh gia: phu hop neu ban muon hoc Azure, khong phai duong nhanh nhat.

Nguon: [Azure free services](https://azure.microsoft.com/en-us/pricing/free-services/).

### 5.4 Oracle Cloud Infrastructure (OCI)

OCI Always Free la ung vien rat dang can nhac cho demo VM-based:

- Co Always Free compute Ampere A1, tuy region/capacity.
- Co Always Free DB options khac, nhung voi du an nay van nen tu host PostgreSQL/Redis/RabbitMQ/Mongo tren VM de don gian.

Uu diem:

- Neu lay duoc Ampere A1 4 OCPU/24GB RAM, day la free-tier rat tot cho Docker Compose full stack.
- Phu hop de chay mot VM demo dai ngay.

Nhuoc diem:

- Hay gap loi het capacity o region.
- ARM64 co the anh huong image/dependency; da so image official ho tro ARM64, nhung can test Keycloak SPI va Java build.
- UX/Networking OCI co the la hon AWS/GCP neu chua quen.

Danh gia: tot nhat ve chi phi neu tao duoc instance, nhung co rui ro availability.

Nguon: [OCI Always Free resources](https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm).

### 5.5 Heroku

Heroku rat de deploy app don le, nhung phuc tap voi full microservice stack nay.

Co the lam:

- Moi Spring service la mot Heroku app/dyno rieng.
- Dung Heroku Postgres cho PostgreSQL.
- Dung Heroku Key-Value Store cho Redis.
- RabbitMQ phai dung third-party add-on hoac CloudAMQP, nhung Heroku student credits khong ap dung cho third-party add-ons theo Heroku Help.
- MongoDB dung MongoDB Atlas.
- Keycloak chay custom container tren Heroku, nhung filesystem ephemeral va port `$PORT` can Dockerfile/cau hinh dung.

Diem can luu y:

- Heroku Docker deploy yeu cau web process listen `$PORT`; `EXPOSE` khong duoc ton trong nhu local Docker.
- Dyno filesystem ephemeral, khong phu hop luu Keycloak data local; Keycloak nen dung external Postgres.
- Heroku khong co Docker Compose production theo nghia chay nhieu container chung network nhu local.
- Network linking dynos khong giong Docker Compose; service-to-service phai di qua public/internal URL tuy setup.

Ve GitHub Student:

- Trang Heroku Student offer hien ghi credits $13/thang trong 24 thang, tong $312.
- Heroku Help article hien ghi chuong trinh 12 thang, tong $156.
- Hai nguon chinh thuc dang khong dong nhat, nen truoc khi quyet dinh phai xem billing/offer trong dashboard cua tai khoan ban.

Danh gia:

- Tot neu deploy 1 monolith hoac 1-2 service.
- Voi full E2E Flash Ticket, Heroku khong don gian hon VM Compose; no chi chuyen do phuc tap sang nhieu app/dyno/add-on.

Nguon: [Heroku Student offer](https://www.heroku.com/github-students/), [Heroku Help Student program](https://help.heroku.com/Z3RHNRHD/how-does-the-heroku-for-github-students-program-work), [Heroku Container Registry](https://devcenter.heroku.com/articles/container-registry-and-runtime).

### 5.6 Render

Render ho tro:

- Static Site cho frontend.
- Web Services, Private Services, Background Workers.
- Docker support.
- Render Postgres, Render Key Value.

Uu diem:

- DX de hon cloud raw.
- Private Services hop voi Config Server/Eureka/internal Spring services.
- Co free web service tier 512MB/0.1 CPU theo pricing page, nhung cac service Java/Keycloak thuong can hon.

Nhuoc diem:

- Full stack can nhieu services: gateway, core, user, discovery, configserver, eureka, keycloak, RabbitMQ/MongoDB.
- Free instance co the sleep/yeu, khong hop Keycloak va Spring Boot nang.
- MongoDB/RabbitMQ can ben ngoai hoac tu chay container rieng.

Danh gia: kha tot cho demo neu chap nhan tra tien nho, nhung khong phai free full E2E.

Nguon: [Render pricing](https://render.com/pricing), [Render Docker docs](https://render.com/docs/docker).

### 5.7 Railway

Railway manh o deploy tu GitHub/Dockerfile va provision service nhanh.

Uu diem:

- Rat nhanh de spin up nhieu service.
- UI env var va logs tot.
- Phu hop prototype.

Nhuoc diem:

- Full stack many services se tieu credits nhanh.
- Can Dockerfile/railway config cho tung service.
- Neu service sleep/scale, Keycloak/Eureka/Config Server co the gay cold-start/registry issue.

Danh gia: tot de demo neu chap nhan chi phi usage-based va muon setup nhanh hon AWS.

Nguon: [Railway pricing docs](https://docs.railway.com/pricing), [Railway Dockerfiles](https://docs.railway.com/builds/dockerfiles).

### 5.8 Fly.io

Fly.io phu hop container gan user region, co private networking tot.

Uu diem:

- Deploy Docker app kha linh hoat.
- Private networking giua apps tot.
- Co volumes, Postgres options, Machines.

Nhuoc diem:

- Tai lieu cost management noi khong con free account/free tier chung cho Fly.io; chu yeu la pay-as-you-go/free allowances cu hoac gioi han cu the.
- Nhieu service always-on se tich phi.
- Stateful services nhu RabbitMQ/Keycloak/Postgres can thiet ke ky.

Danh gia: tot cho engineer da quen infra, khong phai lua chon re/de nhat cho demo dau tien.

Nguon: [Fly.io pricing](https://fly.io/docs/about/pricing/), [Fly.io cost management](https://fly.io/docs/about/cost-management/).

### 5.9 Koyeb

Koyeb co free web service nho:

- 1 free web service 512MB RAM, 0.1 vCPU, 2GB SSD o Frankfurt hoac Washington, D.C.
- 1 free PostgreSQL gioi han 5 gio active time va 1GB storage.

Uu diem:

- Co Docker, GitHub deploy.
- Co free service that su cho app nho.

Nhuoc diem:

- 512MB/0.1 vCPU khong du cho full Spring Boot + Keycloak stack.
- Full E2E can nhieu service, nhanh chong vuot free.
- RabbitMQ/MongoDB/Redis can ben ngoai.

Danh gia: tot cho frontend/API nho, khong nen dung lam default full E2E.

Nguon: [Koyeb pricing FAQ](https://www.koyeb.com/docs/faqs/pricing).

---

## 6. Free tier va managed service nen can nhac

### 6.1 Frontend static hosting

| Nen tang | Free tier/diem manh | Phu hop voi Flash Ticket? | Ghi chu |
| --- | --- | --- | --- |
| Cloudflare Pages | Free, unlimited static requests/bandwidth theo pricing page | Rat phu hop | Tot neu FE goi `https://api.<domain>` va Keycloak `https://auth.<domain>` |
| Vercel Hobby | Free forever cho personal/hobby, CDN va CI/CD tot | Phu hop | Chu y gioi han commercial/team va usage |
| Netlify Free | Co included usage credits hang thang | Phu hop | De deploy Vite static |
| Same VM + Caddy/Nginx | Khong them service ben ngoai | Phu hop nhat neu muon same-origin/proxy Keycloak | Can build FE va serve static tu reverse proxy |

Nguon: [Cloudflare Pages](https://pages.cloudflare.com/), [Vercel pricing](https://vercel.com/pricing), [Netlify pricing](https://www.netlify.com/pricing/).

### 6.2 PostgreSQL/pgvector

| Lua chon | Uu diem | Nhuoc diem | Phu hop |
| --- | --- | --- | --- |
| Self-host `pgvector/pgvector:pg14` tren VM | Khop compose hien tai, pgvector san | Tu backup/patch | **Tot nhat cho demo VM** |
| Neon Free | Free no time limit, Postgres serverless | Can verify pgvector/extension va connection wake-up | Tot cho demo neu muon managed Postgres |
| Supabase Free | De dung, co Postgres | Gioi han project/size, can check extension | Tot cho demo |
| AWS RDS | Cloud-standard | Het credits de ton chi phi, pgvector/extension can check version | Tot cho production path |
| Heroku Postgres Mini | Don gian voi Heroku credits | Credits co gioi han, khong phai full free | Neu chon Heroku |

Nguon: [Neon pricing](https://neon.com/pricing), [Supabase pricing](https://supabase.com/pricing).

### 6.3 MongoDB

| Lua chon | Uu diem | Nhuoc diem | Phu hop |
| --- | --- | --- | --- |
| MongoDB Atlas M0 | Free forever 512MB shared | Gioi han dung luong/perf | **Rat phu hop user-service demo** |
| Self-host MongoDB tren VM | Don gian voi compose | Tu backup/security | Tot neu muon all-in-one |
| DocumentDB/Cosmos Mongo API | Managed cloud | Compatibility/cost can test | Chua can cho demo |

Nguon: [MongoDB pricing](https://www.mongodb.com/pricing).

### 6.4 Redis

| Lua chon | Uu diem | Nhuoc diem | Phu hop |
| --- | --- | --- | --- |
| Self-host Redis tren VM | Khop compose, latency noi bo | Tu van hanh | **Tot nhat cho VM demo** |
| Upstash Redis | Co free/low-cost serverless Redis | Redisson compatibility can test vi Upstash serverless/REST va TCP tuy goi | Tot neu deploy PaaS |
| Render Key Value/Heroku KV/ElastiCache | Managed | Cost | Tot cho cloud-standard |

Nguon: [Upstash Redis pricing](https://upstash.com/pricing/redis).

### 6.5 RabbitMQ

| Lua chon | Uu diem | Nhuoc diem | Phu hop |
| --- | --- | --- | --- |
| Self-host RabbitMQ tren VM | Khop compose, co management UI | Tu backup/security | **Tot nhat cho demo VM** |
| CloudAMQP Little Lemur | Free dev tier, 1M messages/month, 20 connections | Shared broker, gioi han queue/message | Tot cho PaaS demo |
| Amazon MQ for RabbitMQ | Managed AWS | Co the dat voi demo, ton phi | Cloud-standard sau demo |

Nguon: [CloudAMQP plans](https://www.cloudamqp.com/plans.html).

### 6.6 Keycloak

Keycloak la thanh phan kho tim free managed nhat. Voi du an nay con co custom RabbitMQ SPI, nen **self-host Keycloak bang Docker** la hop ly nhat cho demo.

Can chu y:

- Dung public hostname dung: `KC_HOSTNAME=auth.<domain>` hoac cau hinh tuong duong theo Keycloak 26.
- Neu dung reverse proxy HTTPS, can set proxy headers/hostname dung de issuer URI khong bi lech.
- Persistent data khong nen nam trong ephemeral dyno; VM volume hoac external DB tot hon.
- SPI JAR phai build truoc va mount/copy vao `/opt/keycloak/providers/`.

---

## 7. Khuyen nghi kien truc demo mac dinh

### 7.1 Option khuyen nghi: 1 cloud VM + Docker Compose + Caddy

Ten goi: **Demo Single-VM Cloud Compose**

Muc tieu:

- Co URL public cho nguoi khac truy cap.
- It sua code nhat.
- Chi public frontend/gateway/keycloak.
- Chua can HA, autoscale, blue/green deploy.

Tai nguyen goi y:

| Muc | Gia tri |
| --- | --- |
| CPU/RAM toi thieu | 2 vCPU / 4GB RAM, tat discovery neu can |
| CPU/RAM khuyen nghi | 4 vCPU / 8GB RAM |
| Disk | 40-80GB SSD |
| OS | Ubuntu 22.04 LTS hoac 24.04 LTS |
| Reverse proxy | Caddy de tu dong HTTPS, hoac Nginx + Certbot |
| Domain | 3 subdomain: `app`, `api`, `auth` |

DNS:

```text
app.example.com  A  <VM_PUBLIC_IP>
api.example.com  A  <VM_PUBLIC_IP>
auth.example.com A  <VM_PUBLIC_IP>
```

Reverse proxy sketch:

```text
app.example.com {
  root * /srv/flash-ticket/frontend-dist
  try_files {path} /index.html
  file_server
}

api.example.com {
  reverse_proxy apigateway:8080
}

auth.example.com {
  reverse_proxy keycloak:8080
}
```

Neu muon giu logic FE rewrite Keycloak ve relative path, co the dung same-origin proxy thay vi subdomain rieng:

```text
app.example.com/realms/* -> keycloak:8080
app.example.com/resources/* -> keycloak:8080
app.example.com/api/* -> apigateway:8080
```

Nhung cach nay can can than vi API Gateway cung co `/api/**`; subdomain rieng ro rang hon. Neu dung subdomain rieng, nen bo/disable rewrite trong `frontend/src/main.tsx` cho production.

### 7.2 Service startup order

```text
postgres + mongo + redis + rabbitmq
  -> keycloak
  -> configserver
  -> eureka
  -> core-service + user-service + discovery-service
  -> apigateway
  -> frontend static
```

Trong Compose nen co `depends_on` va healthcheck, nhung khong nen tin `depends_on` la du. Spring services can retry connection hoac restart policy.

### 7.3 Public vs internal ports

Expose ra internet:

- 80/443 reverse proxy.
- 22 SSH, gioi han theo IP cua ban neu co the.

Khong expose ra internet:

- 5432 PostgreSQL
- 27017 MongoDB
- 6379 Redis
- 5672 RabbitMQ AMQP
- 15672 RabbitMQ UI, tru khi bao ve
- 8761 Eureka
- 8888 Config Server
- 8081/8082/8085 downstream services

---

## 8. Cac thay doi nen lam truoc khi deploy

### 8.1 Bat buoc cho demo

1. **Tao Dockerfile cho app services**
   - Spring Boot services: multi-stage Maven build -> runtime JRE.
   - Frontend: `npm ci && npm run build` -> serve bang Nginx/Caddy/static.
   - Keycloak SPI: build JAR truoc, sau do mount/copy vao Keycloak provider.

2. **Externalize Config Server URL**

   Thay:

   ```yaml
   spring:
     config:
       import: optional:configserver:http://localhost:8888
   ```

   Bang:

   ```yaml
   spring:
     config:
       import: optional:configserver:${CONFIG_SERVER_URL:http://localhost:8888}
   ```

   Trong Docker:

   ```env
   CONFIG_SERVER_URL=http://configserver:8888
   ```

3. **Sua Config Server native search location**

   Doi Windows path:

   ```yaml
   search-locations: file:/D:/Project/flash-ticket-system/configserver/src/main/resources/config/
   ```

   Thanh mot trong hai cach:

   ```yaml
   search-locations: classpath:/config/
   ```

   hoac:

   ```yaml
   search-locations: ${CONFIG_SEARCH_LOCATIONS:classpath:/config/}
   ```

4. **Externalize Eureka URL cho core-service**

   Hien `core-service.yml` dang hardcode:

   ```yaml
   defaultZone: http://localhost:8761/eureka/
   ```

   Nen doi thanh:

   ```yaml
   defaultZone: ${EUREKA_SERVER_URL:http://localhost:8761/eureka/}
   ```

   Trong Docker:

   ```env
   EUREKA_SERVER_URL=http://eureka:8761/eureka/
   ```

5. **Sua CORS cho Gateway**

   Hien chi allow:

   ```text
   http://localhost:5173
   http://localhost:3000
   ```

   Can them:

   ```text
   https://app.<domain>
   ```

   Tot hon la dung env:

   ```env
   CORS_ALLOWED_ORIGINS=https://app.example.com,http://localhost:5173
   ```

6. **Cau hinh Keycloak public URL**

   Backend:

   ```env
   KEYCLOAK_SERVER_URL=https://auth.example.com
   KEYCLOAK_ISSUER_URI=https://auth.example.com/realms/flash-ticket
   KEYCLOAK_JWK_SET_URI=https://auth.example.com/realms/flash-ticket/protocol/openid-connect/certs
   ```

   Frontend build:

   ```env
   VITE_KEYCLOAK_URL=https://auth.example.com
   VITE_KEYCLOAK_REALM=flash-ticket
   VITE_KEYCLOAK_CLIENT_ID=flash-ticket-frontend
   VITE_API_GATEWAY_URL=https://api.example.com
   ```

7. **Chuan hoa Java build**

   Lua chon A - nhanh, it doi code:
   - Services Java 24 build bang JDK 24 Docker image.
   - Discovery build bang JDK 21.
   - Eureka build bang JDK 17.

   Lua chon B - khuyen nghi lau dai:
   - Ha cac service Java 24 ve Java 21 LTS neu code khong dung feature Java 24.
   - Mot runtime Java 21 de deploy de hon.

8. **Chay DB SQL manual**

   Vi Flyway dang comment, demo can import SQL thu cong:

   ```text
   database/postgresql/V2__complete_schema.sql
   database/postgresql/V3__add_denormalized_fields.sql
   database/postgresql/V4__seed_data.sql
   database/postgresql/V5__add_user_id_to_tickets.sql
   database/postgresql/V6__init_discovery_schema.sql
   database/postgresql/V6__event_layout_tables.sql
   ```

   Can kiem tra thu tu voi schema hien tai. Vi co 2 file `V6`, neu sau nay bat Flyway thi phai rename version.

### 8.2 Nen lam neu co them thoi gian

- Them healthcheck cho Compose:
  - `/actuator/health` cho Spring services.
  - `pg_isready` cho Postgres.
  - `rabbitmq-diagnostics ping` cho RabbitMQ.
  - Redis `PING`.
- Doi log level demo tu DEBUG qua INFO cho cac service nang.
- Restrict Actuator endpoints, it nhat khong expose `/actuator/env`.
- Dung Postgres external DB cho Keycloak thay vi `start-dev` va local H2/file data.
- Bat Flyway that su de DB moi tu khoi tao duoc.

---

## 9. Env var can co cho demo

### 9.1 Public URLs

| Bien | Gia tri vi du | Dung boi |
| --- | --- | --- |
| `PUBLIC_FRONTEND_URL` | `https://app.example.com` | VNPay return URL, Keycloak redirect docs |
| `PUBLIC_GATEWAY_URL` | `https://api.example.com` | FE build, callbacks |
| `PUBLIC_KEYCLOAK_URL` | `https://auth.example.com` | FE/backend auth |
| `VITE_API_GATEWAY_URL` | `https://api.example.com` | Frontend |
| `VITE_KEYCLOAK_URL` | `https://auth.example.com` | Frontend |
| `VITE_KEYCLOAK_REALM` | `flash-ticket` | Frontend |
| `VITE_KEYCLOAK_CLIENT_ID` | `flash-ticket-frontend` | Frontend |

### 9.2 Spring infrastructure

| Bien | Gia tri Docker demo |
| --- | --- |
| `CONFIG_SERVER_URL` | `http://configserver:8888` |
| `EUREKA_SERVER_URL` | `http://eureka:8761/eureka/` |
| `KEYCLOAK_SERVER_URL` | `https://auth.example.com` |
| `KEYCLOAK_ISSUER_URI` | `https://auth.example.com/realms/flash-ticket` |
| `KEYCLOAK_JWK_SET_URI` | `https://auth.example.com/realms/flash-ticket/protocol/openid-connect/certs` |

### 9.3 Data stores

| Bien | Gia tri Docker demo |
| --- | --- |
| `POSTGRES_URL` | `jdbc:postgresql://postgres:5432/postgres` hoac DB rieng |
| `POSTGRES_USER` | `postgres` hoac user rieng |
| `POSTGRES_PASSWORD` | secret manh |
| `MONGO_URI` | `mongodb://mongo:27017/ticketbox_users` |
| `MONGO_DATABASE` | `ticketbox_users` |
| `REDIS_HOST` | `redis` |
| `REDIS_PORT` | `6379` |
| `REDIS_PASSWORD` | secret hoac rong neu Redis chi internal |
| `RABBITMQ_HOST` | `rabbitmq` |
| `RABBITMQ_PORT` | `5672` |
| `RABBITMQ_USERNAME` | user rieng |
| `RABBITMQ_PASSWORD` | secret manh |

### 9.4 Keycloak

| Bien | Ghi chu |
| --- | --- |
| `KEYCLOAK_ADMIN` | Admin bootstrap |
| `KEYCLOAK_ADMIN_PASSWORD` | Khong dung `admin` tren public VM |
| `KEYCLOAK_REALM` | `flash-ticket` |
| `KEYCLOAK_CLIENT_ID` | `user-service` cho admin API |
| `KEYCLOAK_CLIENT_SECRET` | Secret cua service account client |
| `KC_SPI_EVENTS_LISTENER_RABBITMQ_HOST` | `rabbitmq` |
| `KC_SPI_EVENTS_LISTENER_RABBITMQ_PORT` | `5672` |
| `KC_SPI_EVENTS_LISTENER_RABBITMQ_USERNAME` | RabbitMQ user |
| `KC_SPI_EVENTS_LISTENER_RABBITMQ_PASSWORD` | RabbitMQ password |
| `KC_SPI_EVENTS_LISTENER_RABBITMQ_EXCHANGE` | `keycloak.events` |

### 9.5 External integrations

| Bien | Ghi chu |
| --- | --- |
| `CLOUDINARY_CLOUD_NAME` | Upload anh/QR/avatar |
| `CLOUDINARY_API_KEY` | Secret |
| `CLOUDINARY_API_SECRET` | Secret |
| `MAIL_USERNAME` | Gmail SMTP account |
| `MAIL_PASSWORD` | Gmail App Password |
| `QR_SECRET_KEY` | Nen random >= 32 ky tu |
| `VNPAY_TMN_CODE` | Sandbox merchant |
| `VNPAY_HASH_SECRET` | Sandbox hash secret |
| `VNPAY_PAYMENT_URL` | `https://sandbox.vnpayment.vn/paymentv2/vpcpay.html` cho demo |
| `VNPAY_RETURN_URL` | `https://app.example.com/payment/result` |
| `VNPAY_API_URL` | VNPay sandbox query API neu dung |
| `GEMINI_API_KEY` | Bat discovery-service |
| `GEMINI_MODEL` | Vi du `gemini-2.0-flash` |
| `OPENAI_API_KEY` | Optional |
| `OPENAI_MODEL` | Optional |

---

## 10. Runbook deploy demo single VM

### Buoc 0 - Chot domain va scope

Chon domain/subdomain:

```text
app.example.com
api.example.com
auth.example.com
```

Quyet dinh discovery-service:

- Bat neu co `GEMINI_API_KEY` va can demo AI chat.
- Tat neu chi can booking/payment/user flow.

### Buoc 1 - Tao VM

Tren AWS/OCI/GCP/Azure:

1. Tao Ubuntu VM.
2. Gan public IP on dinh neu co.
3. Mo inbound:
   - `80/tcp`
   - `443/tcp`
   - `22/tcp` chi tu IP cua ban neu co the
4. Cai:
   - Docker Engine
   - Docker Compose plugin
   - Git
   - Caddy hoac Nginx

### Buoc 2 - Clone repo va tao secret file

Tren server:

```bash
git clone <repo-url> flash-ticket-system
cd flash-ticket-system
```

Tao `.env.demo` cho Compose. Khong commit file nay.

Toi thieu can co:

```env
POSTGRES_USER=postgres
POSTGRES_PASSWORD=<strong-password>

RABBITMQ_DEFAULT_USER=flash
RABBITMQ_DEFAULT_PASS=<strong-password>
RABBITMQ_USERNAME=flash
RABBITMQ_PASSWORD=<strong-password>

KEYCLOAK_ADMIN=admin
KEYCLOAK_ADMIN_PASSWORD=<strong-password>

CONFIG_SERVER_URL=http://configserver:8888
EUREKA_SERVER_URL=http://eureka:8761/eureka/

KEYCLOAK_SERVER_URL=https://auth.example.com
KEYCLOAK_ISSUER_URI=https://auth.example.com/realms/flash-ticket
KEYCLOAK_JWK_SET_URI=https://auth.example.com/realms/flash-ticket/protocol/openid-connect/certs

POSTGRES_URL=jdbc:postgresql://postgres:5432/postgres
MONGO_URI=mongodb://mongo:27017/ticketbox_users
MONGO_DATABASE=ticketbox_users
REDIS_HOST=redis
REDIS_PORT=6379
REDIS_PASSWORD=
RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672

VNPAY_RETURN_URL=https://app.example.com/payment/result
QR_SECRET_KEY=<random-32-plus-chars>
```

### Buoc 3 - Build Keycloak RabbitMQ SPI

```bash
cd keycloak-rabbitmq-spi
./mvnw clean package -DskipTests
cd ..
```

Kiem tra file:

```text
keycloak-rabbitmq-spi/target/keycloak-rabbitmq-spi.jar
```

Neu deploy bang container build trong CI, nen tao custom Keycloak image copy JAR vao `/opt/keycloak/providers/` thay vi mount file host.

### Buoc 4 - Build Docker images cho app services

Can them Dockerfile truoc. Huong khuyen nghi:

- `configserver/Dockerfile`
- `eureka/Dockerfile`
- `apigateway/Dockerfile`
- `core-service/Dockerfile`
- `user-service/Dockerfile`
- `discovery-service/Dockerfile`
- `frontend/Dockerfile` hoac build static ra host

Voi Java 24 services, neu chua ha ve Java 21, Dockerfile build stage can JDK 24. Voi demo, khong nen build tren server bang local Java 21 neu POM van `java.version=24`.

### Buoc 5 - Start infra

Chay Postgres, Mongo, Redis, RabbitMQ, Keycloak truoc.

```bash
docker compose --env-file .env.demo up -d postgres mongo redis rabbitmq keycloak
```

Kiem tra:

```bash
docker compose ps
docker compose logs -f keycloak
```

### Buoc 6 - Khoi tao database

Chay SQL Postgres manual theo thu tu. Vi file migration hien nam ngoai classpath service:

```bash
docker exec -i postgres_flash_ticket psql -U "$POSTGRES_USER" -d postgres < database/postgresql/V2__complete_schema.sql
docker exec -i postgres_flash_ticket psql -U "$POSTGRES_USER" -d postgres < database/postgresql/V3__add_denormalized_fields.sql
docker exec -i postgres_flash_ticket psql -U "$POSTGRES_USER" -d postgres < database/postgresql/V4__seed_data.sql
docker exec -i postgres_flash_ticket psql -U "$POSTGRES_USER" -d postgres < database/postgresql/V5__add_user_id_to_tickets.sql
docker exec -i postgres_flash_ticket psql -U "$POSTGRES_USER" -d postgres < database/postgresql/V6__init_discovery_schema.sql
docker exec -i postgres_flash_ticket psql -U "$POSTGRES_USER" -d postgres < database/postgresql/V6__event_layout_tables.sql
```

Neu dung shell non-interactive tren server, truyen user/password dung theo compose cua ban. Sau demo, nen chuyen sang Flyway de khong can thao tac manual.

MongoDB:

```bash
docker exec -i mongodb_flash_ticket mongosh ticketbox_users < database/mongodb/user_service_schema.js
```

Neu script chi la schema/documentation va khong tao index, can bo sung lenh tao collection/index tuong ung.

### Buoc 7 - Start Config Server va Eureka

```bash
docker compose --env-file .env.demo up -d configserver eureka
```

Kiem tra:

```bash
curl http://localhost:8888/core-service/default
curl http://localhost:8761
```

Tren VM, `localhost` chi dung trong SSH. Khong public 8888/8761.

### Buoc 8 - Start backend services

```bash
docker compose --env-file .env.demo up -d core-service user-service discovery-service apigateway
```

Kiem tra:

```bash
curl http://localhost:8080/actuator/health
curl http://localhost:8081/actuator/health
curl http://localhost:8082/actuator/health
curl http://localhost:8085/actuator/health
```

Vao Eureka dashboard noi bo qua SSH tunnel neu can:

```bash
ssh -L 8761:localhost:8761 user@server
```

### Buoc 9 - Cau hinh Keycloak realm/client

Trong Keycloak admin:

1. Tao/import realm `flash-ticket`.
2. Tao client `flash-ticket-frontend`:
   - Client type: public.
   - Valid Redirect URIs: `https://app.example.com/*`
   - Web Origins: `https://app.example.com`
   - PKCE S256 enabled neu can.
3. Tao client/service account cho `user-service`:
   - Luu `KEYCLOAK_CLIENT_SECRET`.
   - Gan role/permission quan ly user/role theo logic `KeycloakAdminService`.
4. Tao roles realm:
   - `BUYER`
   - `ORGANIZER`
   - `ADMIN`
5. Bat event listener RabbitMQ SPI neu extension yeu cau cau hinh trong Keycloak.

Kiem tra issuer:

```bash
curl https://auth.example.com/realms/flash-ticket/.well-known/openid-configuration
```

`issuer` phai chinh xac la:

```text
https://auth.example.com/realms/flash-ticket
```

Neu issuer tra ve `http://...` hoac internal hostname, backend JWT validation se loi.

### Buoc 10 - Build va deploy frontend

Build env:

```env
VITE_API_GATEWAY_URL=https://api.example.com
VITE_KEYCLOAK_URL=https://auth.example.com
VITE_KEYCLOAK_REALM=flash-ticket
VITE_KEYCLOAK_CLIENT_ID=flash-ticket-frontend
```

Build:

```bash
cd frontend
npm ci
npm run build
```

Deploy:

- Neu same VM: copy `frontend/dist` vao `/srv/flash-ticket/frontend-dist`.
- Neu Cloudflare Pages/Vercel/Netlify: set env build tuong ung va deploy static.

Luu y: neu giu code rewrite Keycloak trong `frontend/src/main.tsx`, test login ky. Neu FE host `https://app.example.com` va Keycloak `https://auth.example.com`, rewrite co the bien request auth thanh relative path tren app domain. Khi do can:

- bo rewrite trong production, hoac
- proxy `/realms/**` tren app domain ve Keycloak, hoac
- set `VITE_KEYCLOAK_URL` thanh relative path co chu dich.

### Buoc 11 - Smoke test public

Chay theo thu tu:

1. `https://app.example.com` load duoc UI.
2. `https://api.example.com/actuator/health` tra `UP` neu public health duoc permit.
3. `https://auth.example.com/realms/flash-ticket/.well-known/openid-configuration` co issuer dung.
4. FE load danh sach event public.
5. Dang ky/dang nhap Keycloak.
6. Sau login, user-service tao/sync user vao MongoDB.
7. Buyer dat ve va tao order.
8. Thanh toan VNPay sandbox redirect ve `https://app.example.com/payment/result`.
9. Ticket/order hien trong "My Orders"/"My Tickets".
10. Organizer flow co the tao/sua event neu role dung.
11. Discovery/chat hoat dong neu co `GEMINI_API_KEY`.

---

## 11. Acceptance checklist cho demo

| Checklist | Pass khi |
| --- | --- |
| Public FE | Nguoi khac truy cap duoc `https://app.<domain>` khong can VPN |
| Public Gateway | FE goi API qua `https://api.<domain>` khong bi CORS |
| Public Keycloak | Login/register redirect dung ve FE |
| JWT validation | Gateway/core/user/discovery accept token Keycloak public issuer |
| Event public | Trang home/search/detail load event |
| User sync | Dang ky/login tao user trong MongoDB |
| Booking | Tao order PENDING thanh cong |
| Payment sandbox | VNPay sandbox redirect ve FE result page |
| Ticket/email | Ticket issuance va email worker khong crash; neu email fail co log ro |
| RabbitMQ | Queue khong accumulate bat thuong |
| Redis lock | Booking/payment khong loi Redis connection |
| Discovery | Chat/discovery tra response neu bat AI service |
| No infra exposure | DB/Redis/RabbitMQ AMQP/Eureka/Config Server khong public internet |

---

## 12. Neu muon deploy theo PaaS thay vi VM

### 12.1 Phuong an hybrid PaaS hop ly

Neu khong muon quan ly DB/broker tren VM:

- Frontend: Cloudflare Pages.
- Backend containers: Render/Railway/Koyeb/Fly/Cloud Run.
- PostgreSQL: Neon/Supabase/Render Postgres/RDS.
- MongoDB: MongoDB Atlas M0.
- Redis: Upstash/Render Key Value/ElastiCache.
- RabbitMQ: CloudAMQP Little Lemur cho dev/demo.
- Keycloak: van nen self-host bang container co external Postgres.

De lam duoc, nen refactor deployment config:

- Bo hard dependency vao Eureka cho demo, route bang URL env truc tiep.
- Bo Config Server cho demo, inject env var truc tiep.
- Giam service count: chi deploy gateway/core/user/discovery, khong deploy eureka/configserver.

### 12.2 Khi nao PaaS dang gia hon VM?

Nen dung PaaS neu:

- Ban uu tien UI deploy/logs/restart de dung.
- San sang tra phi cho nhieu service.
- Chap nhan sua config de loai bo `localhost`, Eureka/Config Server complexity.
- Khong can toan bo infra nam cung mot private Docker network.

Khong nen dung PaaS neu:

- Muc tieu la "full E2E demo nhanh nhat".
- Muon tan dung compose hien co.
- Muon it sua code/config.
- Muon chay Keycloak custom SPI + RabbitMQ noi bo don gian.

---

## 13. Lo trinh sau demo

### Phase Demo

- 1 VM + Docker Compose.
- Manual DB migration.
- Caddy HTTPS.
- Backup thu cong volume Postgres/Mongo.
- Single instance moi service.

### Phase Staging

- Them Dockerfile chuan.
- Them `docker-compose.demo.yml` rieng, khong tron voi local infra.
- Bat Flyway cho Postgres.
- Config profiles `dev/demo/prod`.
- Healthcheck va restart policy day du.
- Logs tap trung toi CloudWatch/Loki/Render logs tuy nen tang.
- Dung external managed MongoDB Atlas va CloudAMQP neu can giam state tren VM.

### Phase Production-ish

- ECS/Fargate hoac Kubernetes.
- RDS PostgreSQL backup/PITR.
- Managed Redis.
- Managed RabbitMQ.
- Keycloak external DB, HA neu can.
- Secrets Manager/Parameter Store.
- ALB/WAF/rate limiting.
- Observability: metrics, logs, tracing.
- CI/CD build image va deploy tu GitHub Actions.

---

## 14. Quyet dinh de xuat

Voi muc tieu hien tai la demo public URL:

1. **Dung Docker.**
2. **Dung mot cloud VM truoc.**
3. **Khong dung Kafka/ZooKeeper cho demo** vi code/config hien tai chi comment Kafka.
4. **Chay discovery-service neu co AI key va VM du RAM**, con khong thi tat de giam rui ro.
5. **Frontend nen deploy bang Cloudflare Pages hoac serve cung VM.**
   - Neu muon it CORS/Keycloak issue: serve cung VM va proxy ro rang.
   - Neu muon FE nhanh/dep: Cloudflare Pages + CORS/Keycloak config dung.
6. **Sau demo moi chuyen sang ECS/Fargate/Cloud Run/Container Apps**, vi luc do co thoi gian refactor Config Server/Eureka/secrets.

---

## 15. Nguon tham khao

- Heroku Student offer: https://www.heroku.com/github-students/
- Heroku Help student credits: https://help.heroku.com/Z3RHNRHD/how-does-the-heroku-for-github-students-program-work
- Heroku Docker deploys: https://devcenter.heroku.com/articles/container-registry-and-runtime
- Render pricing: https://render.com/pricing
- Render Docker docs: https://render.com/docs/docker
- Railway pricing: https://docs.railway.com/pricing
- Railway Dockerfiles: https://docs.railway.com/builds/dockerfiles
- Fly.io pricing: https://fly.io/docs/about/pricing/
- Fly.io cost management: https://fly.io/docs/about/cost-management/
- Koyeb pricing FAQ: https://www.koyeb.com/docs/faqs/pricing
- Google Cloud Run pricing: https://cloud.google.com/run/pricing
- Azure free services: https://azure.microsoft.com/en-us/pricing/free-services/
- AWS Free Tier: https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier.html
- AWS Free Tier eligibility: https://docs.aws.amazon.com/awsaccountbilling/latest/aboutv2/free-tier-eligibility.html
- Oracle Always Free: https://docs.oracle.com/en-us/iaas/Content/FreeTier/freetier_topic-Always_Free_Resources.htm
- MongoDB Atlas pricing: https://www.mongodb.com/pricing
- Neon pricing: https://neon.com/pricing
- Supabase pricing: https://supabase.com/pricing
- Upstash Redis pricing: https://upstash.com/pricing/redis
- CloudAMQP pricing: https://www.cloudamqp.com/plans.html
- Cloudflare Pages: https://pages.cloudflare.com/
- Vercel pricing: https://vercel.com/pricing
- Netlify pricing: https://www.netlify.com/pricing/

