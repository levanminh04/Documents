# Tài Liệu Quản Lý Cấu Hình Spring Boot: External Configuration & YAML vs Properties

## Tổng Quan

Tài liệu này giải thích chi tiết về thực tiễn đặt cấu hình bên ngoài `src/main/resources`, so sánh YAML vs Properties, và cung cấp hướng dẫn triển khai trong môi trường doanh nghiệp.

**Mục tiêu**: Giúp lập trình viên Java và DevOps hiểu rõ nguyên lý, áp dụng đúng pattern, và tránh các anti-patterns phổ biến.

---

## 1. Vấn Đề Cốt Lõi: Tại Sao "Build Lại JAR" Là Vấn Đề?

### 1.1 Hiểu Về JAR File và Quá Trình Build

**JAR file là gì?**
- JAR (Java Archive) là file nén chứa tất cả code Java đã được compile (.class files)
- Khi build Spring Boot project, Maven/Gradle tạo ra 1 file JAR chứa toàn bộ ứng dụng
- File JAR này có thể chạy độc lập: `java -jar myapp.jar`

**Quá trình build thông thường:**
```bash
# Bước 1: Compile code Java (.java → .class)
mvn compile

# Bước 2: Package thành JAR (bao gồm cả config trong src/main/resources)
mvn package
→ Tạo ra: target/myapp.jar (chứa code + config)

# Bước 3: Deploy JAR lên server
scp myapp.jar user@server:/app/
```

### 1.2 Vấn Đề Khi Config Nằm Trong JAR

**Tình huống thực tế:**

Bạn có ứng dụng cần deploy lên 3 server khác nhau:

```yaml
# src/main/resources/application.yml (nằm TRONG JAR)
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/devdb  # ← Chỉ phù hợp với DEV
    username: dev_user
    password: dev_password

server:
  port: 8080
```

**Vấn đề phát sinh:**

**🔴 Khi deploy lên STAGING:**
- Config trong JAR vẫn trỏ tới `localhost:5432/devdb`
- STAGING server cần trỏ tới `staging-db:5432/stagingdb`
- **Buộc phải sửa code → Build lại JAR → Deploy**

**🔴 Khi deploy lên PRODUCTION:**
- Config trong JAR vẫn trỏ tới database dev
- PRODUCTION cần trỏ tới `prod-cluster:5432/proddb`
- **Lại phải sửa code → Build lại JAR → Deploy**

### 1.3 Chi Phí Của Việc "Build Lại JAR"

**⏰ Thời gian:**
```bash
# Mỗi lần thay đổi config:
git commit -m "Update config for staging"  # 2 phút
mvn clean package                          # 5-10 phút (tùy project)
scp JAR lên server                         # 2-5 phút
Restart service                            # 1-2 phút
Test                                       # 5-10 phút
---
TỔNG: 15-30 phút cho MỖI lần thay đổi config
```

**🚨 Rủi ro:**
1. **Code drift**: 3 JAR files khác nhau cho 3 môi trường
2. **Test không đồng nhất**: Code chạy trên DEV khác với code chạy trên PROD
3. **Rollback khó khăn**: Phải rebuild JAR cũ
4. **Hotfix config**: Không thể sửa config nhanh khi có sự cố

### 1.4 So Sánh: Config Trong JAR vs External Config

**❌ Config trong JAR (Traditional Way):**

```
📦 JAR cho DEV     → Chứa config DEV
📦 JAR cho STAGING → Chứa config STAGING  
📦 JAR cho PROD    → Chứa config PROD

Kết quả: 3 JAR files khác nhau!
```

```bash
# Flow deploy truyền thống
1. Sửa application.yml cho STAGING
2. git commit + push
3. mvn clean package  ← BUILD LẠI JAR
4. Deploy JAR mới
5. Test

# Nếu có bug config → Lặp lại từ bước 1
```

**✅ External Config (Modern Way):**

```
📦 JAR duy nhất    → Không chứa config môi trường cụ thể
📄 config-dev.yml
📄 config-staging.yml  
📄 config-prod.yml

Kết quả: 1 JAR + 3 config files riêng biệt
```

```bash
# Flow deploy hiện đại
1. Build JAR 1 lần duy nhất
2. Copy JAR + config tương ứng lên server
3. Chạy: java -jar myapp.jar --spring.profiles.active=prod

# Nếu cần sửa config → Chỉ sửa file config, restart app (không cần rebuild)
```

### 1.5 Ví Dụ Thực Tế Từ Dự Án Của Bạn

**Hiện tại dự án bạn đã làm đúng:**

```
api-introduce/
├── target/
│   └── api-introduce.jar          ← JAR không chứa config production
├── config/                        ← External config
│   ├── application.yml            ← Config cho production
│   └── ...
```

**Điều này có nghĩa:**
1. ✅ JAR file chỉ build 1 lần
2. ✅ Thay đổi config không cần rebuild
3. ✅ Deploy nhanh bằng cách thay đổi file config
4. ✅ Có thể hotfix config mà không động tới code

---

## 2. Hiểu Đơn Giản Về External Configuration

### 2.1 Câu Hỏi Thường Gặp

**Q: Tại sao phải đặt config ra ngoài thư mục `src/main/resources`?**

Hãy tưởng tượng bạn có 1 ứng dụng cần chạy ở 3 môi trường:

```
📱 DEV (máy local)    → database: localhost:5432, timeout: 30s
🧪 STAGING (server)   → database: staging-db:5432, timeout: 10s  
🏭 PRODUCTION (server) → database: prod-cluster:5432, timeout: 5s
```

**❌ Cách cũ (config trong `src/main/resources`):**
- Muốn deploy lên STAGING → Phải sửa `application.yml` → Build lại JAR → Deploy
- Muốn deploy lên PROD → Phải sửa `application.yml` → Build lại JAR → Deploy
- Kết quả: 3 JAR files khác nhau, rủi ro cao, mất thời gian

**✅ Cách mới (external config):**
- Build 1 lần duy nhất → JAR file giống hệt nhau
- Deploy DEV: Copy JAR + config-dev.yml
- Deploy STAGING: Copy JAR + config-staging.yml  
- Deploy PROD: Copy JAR + config-prod.yml

**Q: Spring Boot có tự động tìm thư mục `config/` không?**

**Có!** Spring Boot **TỰ ĐỘNG** tìm config theo thứ tự ưu tiên:

```bash
1. ./config/application.yml        ← Thư mục config ở root project (PRIORITY CAO NHẤT)
2. ./application.yml               ← File ở root project  
3. classpath:/config/application.yml ← Trong JAR (src/main/resources/config/)
4. classpath:/application.yml     ← Trong JAR (src/main/resources/) (PRIORITY THẤP NHẤT)
```

**Q: Khi nào cần `spring.config.location`?**

**Không cần** nếu bạn đặt config ở 4 vị trí trên!

**Chỉ cần** khi muốn đặt config ở chỗ khác:

```bash
# Ví dụ: Đặt config ở /etc/myapp/
java -jar myapp.jar --spring.config.location=file:/etc/myapp/

# Hoặc đặt trong Docker container ở /app/config/
java -jar myapp.jar --spring.config.additional-location=file:/app/config/
```

**Q: `spring.config.location` đặt ở đâu?**

Có 3 cách:

```bash
# Cách 1: Command line (khuyến nghị cho production)
java -jar myapp.jar --spring.config.location=file:config/

# Cách 2: Environment variable
export SPRING_CONFIG_LOCATION=file:config/
java -jar myapp.jar

# Cách 3: Trong application.properties (như dự án của bạn)
# src/main/resources/application.properties
spring.config.location=file:config/
```

### 2.2 Demo Với Dự Án Thực Tế

Dự án của bạn hiện tại:

```
api-introduce/
├── src/main/resources/
│   └── application.properties     ← spring.config.location=file:config/
├── config/                        ← External config (PRIORITY CAO)
│   ├── application.yml
│   ├── beans.xml
│   ├── log4j2.xml
│   └── schedule-conf.xml
└── target/
    └── api-introduce.jar          ← JAR không chứa config nhạy cảm
```

**Luồng hoạt động:**
1. Spring Boot đọc `src/main/resources/application.properties`
2. Thấy `spring.config.location=file:config/`
3. Chuyển sang đọc config từ thư mục `config/`
4. Config trong `config/` sẽ **override** config trong JAR

### 2.3 Ví Dụ Thực Tế: 1 JAR - 3 Môi Trường

**File JAR duy nhất:**
```bash
api-introduce.jar  # Chứa code, không chứa config production
```

**Config riêng cho từng môi trường:**

```yaml
# config/application-dev.yml
app:
  database:
    url: jdbc:h2:mem:devdb
    show-sql: true
  external-api:
    timeout: 30000
    
# config/application-prod.yml  
app:
  database:
    url: jdbc:postgresql://prod-cluster:5432/myapp
    show-sql: false
  external-api:
    timeout: 5000
```

**Deploy:**
```bash
# Development
java -jar api-introduce.jar --spring.profiles.active=dev

# Production  
java -jar api-introduce.jar --spring.profiles.active=prod
```

## 3. Nền Tảng Lý Thuyết

### 3.1 Cơ Chế Nạp Cấu Hình Spring Boot

Spring Boot tuân theo thứ tự ưu tiên (order of precedence) khi nạp cấu hình:

1. **Command line arguments** (`--server.port=8080`)
2. **Environment variables** (`SERVER_PORT=8080`)
3. **External configuration files** (`file:./config/application.yml`)
4. **JAR-internal configuration** (`classpath:application.yml`)
5. **Default properties**

```java
// Ví dụ: Cấu hình được nạp theo thứ tự ưu tiên
@Value("${server.port:8080}")
private int serverPort; // Giá trị cuối cùng được quyết định bởi source có priority cao nhất
```

### 2.2 Externalized Configuration Locations

Spring Boot tìm kiếm cấu hình theo thứ tự:

```bash
# Thứ tự tìm kiếm (high → low priority)
./config/           # Thư mục config ở working directory
./                  # Working directory
classpath:/config/  # config package trong JAR
classpath:/         # root của classpath (src/main/resources)
```

### 2.3 Nguyên Lý 12-Factor App

**Factor III - Config**: "Store config in the environment"

- **Vấn đề**: Code và config được bundle cùng nhau
- **Nguyên lý**: Config phải tách khỏi code, có thể thay đổi giữa các deployment mà không rebuild
- **Giải pháp**: Externalized configuration

---

## 3. Vì Sao Đặt Config Bên Ngoài `src/main/resources`

### 3.1 Phân Tách Code/Config Theo Môi Trường

**Vấn đề**: Cùng một artifact cần chạy trên nhiều môi trường khác nhau

```yaml
# Cấu hình DEV (config/application-dev.yml)
app:
  database:
    url: jdbc:postgresql://dev-db:5432/myapp
    username: dev_user
  external-api:
    base-url: https://api-dev.example.com
    timeout: 30000

---
# Cấu hình PROD (config/application-prod.yml)
app:
  database:
    url: jdbc:postgresql://prod-cluster:5432/myapp
    username: prod_user
  external-api:
    base-url: https://api.example.com
    timeout: 10000
```

**Lợi ích**:
- **Single artifact**: Một JAR file chạy được trên tất cả môi trường
- **No rebuild**: Thay đổi config không cần rebuild/redeploy application
- **Environment parity**: Đảm bảo code giống nhau, chỉ khác config

### 3.2 Bảo Mật và Quản Lý Secrets

**Vấn đề**: Secrets không được commit vào source code

```yaml
# ❌ KHÔNG BAO GIỜ làm thế này trong src/main/resources
app:
  database:
    password: prod_secret_password_123
  jwt:
    secret: super_secret_jwt_key
```

**Giải pháp**: External configuration với secret management

```bash
# Environment variables
export DB_PASSWORD=prod_secret_password_123
export JWT_SECRET=super_secret_jwt_key

# Hoặc mount secrets từ Vault/K8s Secrets
```

```yaml
# config/application.yml
app:
  database:
    password: ${DB_PASSWORD}
  jwt:
    secret: ${JWT_SECRET}
```

### 3.3 Triển Khai Container & Kubernetes

**Docker Strategy**:

```dockerfile
FROM openjdk:17-jre-slim

# Application JAR (không chứa config nhạy cảm)
COPY target/myapp.jar /app/myapp.jar

# Config được mount từ bên ngoài
VOLUME ["/app/config"]

CMD ["java", "-jar", "/app/myapp.jar", "--spring.config.additional-location=file:/app/config/"]
```

**Kubernetes ConfigMap & Secret**:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  application.yml: |
    app:
      feature:
        cache-enabled: true
        batch-size: 1000

---
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
data:
  database-password: cGFzc3dvcmQ=  # base64 encoded

---
apiVersion: apps/v1
kind: Deployment
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
        - name: secrets-volume
          mountPath: /app/secrets
        env:
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: app-secrets
              key: database-password
      volumes:
      - name: config-volume
        configMap:
          name: app-config
      - name: secrets-volume
        secret:
          secretName: app-secrets
```

### 3.4 Hot Reload và Runtime Configuration

**Spring Boot Actuator**: Cho phép refresh config mà không restart

```yaml
# config/application.yml
management:
  endpoints:
    web:
      exposure:
        include: refresh,health,info
  endpoint:
    refresh:
      enabled: true
```

```bash
# Trigger config refresh
curl -X POST http://localhost:8080/actuator/refresh
```

### 3.5 Audit và Version Control

**Tracking Configuration Changes**:

```bash
# Config được version riêng, độc lập với code
git log --oneline config/
a1b2c3d Update database timeout for prod
e4f5g6h Add new feature flag for user service
h7i8j9k Update API endpoints for staging
```

**Configuration Promotion Pipeline**:

```yaml
# .github/workflows/config-promotion.yml
name: Config Promotion
on:
  push:
    paths: ['config/**']
jobs:
  promote-config:
    runs-on: ubuntu-latest
    steps:
    - name: Validate config syntax
      run: yamllint config/
    - name: Deploy to staging
      run: kubectl apply -f k8s/staging/
    - name: Run smoke tests
      run: ./scripts/smoke-test.sh staging
    - name: Deploy to production (manual approval)
      if: github.ref == 'refs/heads/main'
      run: kubectl apply -f k8s/production/
```

### 3.6 Nhược Điểm và Đánh Đổi

**Configuration Drift**:
- **Vấn đề**: Config khác nhau giữa các môi trường mà không được track
- **Giải pháp**: Configuration management tools, automated testing

**Phức Tạp Hoá Quy Trình**:
- **Vấn đề**: Thêm bước config management vào deployment pipeline
- **Giải pháp**: Automation, infrastructure as code

**Dependency Management**:
- **Vấn đề**: Application phụ thuộc vào external config để chạy
- **Giải pháp**: Fallback values, health checks, fail-fast validation

---

## 4. Mức Độ Phổ Biến Trong Công Nghiệp

### 4.1 Cloud-Native Applications

**Thực tiễn tiêu chuẩn**: 90%+ ứng dụng cloud-native sử dụng external configuration

**Common Patterns**:
- **Environment Variables**: Microservices, serverless
- **Mounted Files**: Docker containers, Kubernetes
- **Config Server**: Spring Cloud Config, Consul
- **Secret Management**: HashiCorp Vault, AWS Secrets Manager

### 4.2 Enterprise Applications

**Typical Architecture**:

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Config Repo   │────│   Config Server  │────│   Application   │
│   (Git/Vault)   │    │  (Spring Cloud)  │    │   (Spring Boot) │
└─────────────────┘    └──────────────────┘    └─────────────────┘
```

**Benefits in Enterprise**:
- **Compliance**: Config changes require approval workflow
- **Audit**: All changes tracked and logged
- **Rollback**: Quick rollback without code deployment

### 4.3 Ngoại Lệ Hợp Lý

**Khi nào giữ config trong `src/main/resources`**:

1. **Desktop Applications**: Config không đổi giữa deployments
2. **PoC/Learning Projects**: Đơn giản hoá setup
3. **Offline Distribution**: Artifact tự chứa đủ mọi thứ
4. **Library/Framework**: Default configuration

```java
// Ví dụ: Library cung cấp default config
@ConfigurationProperties(prefix = "mylib")
public class MyLibraryConfig {
    private boolean enabled = true;  // Default value
    private int timeout = 30000;     // Default value
    // getters/setters
}
```

---

## 5. YAML vs Properties: So Sánh Chi Tiết

### 5.1 Cấu Trúc và Khả Năng Biểu Diễn

**YAML: Hierarchical Structure**

```yaml
# application.yml - Cấu trúc phân cấp tự nhiên
app:
  database:
    primary:
      url: jdbc:postgresql://primary-db:5432/myapp
      username: app_user
      pool:
        min-size: 5
        max-size: 20
    replica:
      url: jdbc:postgresql://replica-db:5432/myapp
      username: readonly_user
  security:
    jwt:
      secret: ${JWT_SECRET}
      expiration: 3600
    cors:
      allowed-origins:
        - https://frontend.example.com
        - https://admin.example.com
      allowed-methods:
        - GET
        - POST
        - PUT
        - DELETE
```

**Properties: Flat Structure**

```properties
# application.properties - Cấu trúc phẳng
app.database.primary.url=jdbc:postgresql://primary-db:5432/myapp
app.database.primary.username=app_user
app.database.primary.pool.min-size=5
app.database.primary.pool.max-size=20
app.database.replica.url=jdbc:postgresql://replica-db:5432/myapp
app.database.replica.username=readonly_user
app.security.jwt.secret=${JWT_SECRET}
app.security.jwt.expiration=3600
app.security.cors.allowed-origins[0]=https://frontend.example.com
app.security.cors.allowed-origins[1]=https://admin.example.com
app.security.cors.allowed-methods[0]=GET
app.security.cors.allowed-methods[1]=POST
app.security.cors.allowed-methods[2]=PUT
app.security.cors.allowed-methods[3]=DELETE
```

### 5.2 Multi-Document và Profiles

**YAML: Multi-Document Support**

```yaml
# application.yml - Multiple profiles trong 1 file
spring:
  profiles:
    active: dev

---
spring:
  config:
    activate:
      on-profile: dev
app:
  database:
    url: jdbc:h2:mem:devdb
    show-sql: true
  logging:
    level: DEBUG

---
spring:
  config:
    activate:
      on-profile: prod
app:
  database:
    url: jdbc:postgresql://prod-db:5432/myapp
    show-sql: false
  logging:
    level: INFO
```

**Properties: Separate Files Required**

```properties
# application.properties
spring.profiles.active=dev

# application-dev.properties
app.database.url=jdbc:h2:mem:devdb
app.database.show-sql=true
app.logging.level=DEBUG

# application-prod.properties
app.database.url=jdbc:postgresql://prod-db:5432/myapp
app.database.show-sql=false
app.logging.level=INFO
```

### 5.3 YAML Advanced Features

**Anchors và Aliases để tái sử dụng**:

```yaml
# Định nghĩa anchors
defaults: &defaults
  timeout: 30000
  retry-attempts: 3
  pool:
    min-size: 5
    max-size: 20

app:
  primary-service:
    <<: *defaults
    url: https://primary.example.com
  
  backup-service:
    <<: *defaults
    url: https://backup.example.com
    timeout: 60000  # Override default timeout
```

**Lists và Maps phức tạp**:

```yaml
app:
  services:
    - name: user-service
      url: https://user.example.com
      health-check: /health
      circuit-breaker:
        failure-threshold: 5
        timeout: 10000
    - name: payment-service
      url: https://payment.example.com
      health-check: /actuator/health
      circuit-breaker:
        failure-threshold: 3
        timeout: 5000
```

### 5.4 Rủi Ro và Cạm Bẫy

**YAML Pitfalls**:

```yaml
# ❌ Lỗi thụt lề (indentation)
app:
  database:
    url: jdbc:postgresql://localhost:5432/myapp
  username: myuser  # Sai thụt lề - nên thuộc database

# ❌ Lỗi quoting
app:
  password: "yes"     # String "yes"
  enabled: yes        # Boolean true
  version: "1.10"     # String "1.10"
  timeout: 1.10       # Number 1.1

# ❌ Type coercion không mong muốn
app:
  norwegian-postal-code: 1234   # Number, không phải String "1234"
  swedish-postal-code: "1234"   # String "1234"
```

**Properties Pitfalls**:

```properties
# ❌ Khó quản lý lists phức tạp
app.users[0].name=John
app.users[0].roles[0]=ADMIN
app.users[0].roles[1]=USER
app.users[1].name=Jane
app.users[1].roles[0]=USER
# Dễ bị lỗi index, khó maintain

# ❌ Trùng lặp key prefix
app.service.user.timeout=30000
app.service.user.retry=3
app.service.payment.timeout=30000
app.service.payment.retry=3
```

### 5.5 Performance và Tooling

**Parsing Performance**:
- **YAML**: Chậm hơn do phải parse structure
- **Properties**: Nhanh hơn do format đơn giản

**IDE Support**:
- **YAML**: Tốt hơn cho autocomplete, validation
- **Properties**: Cơ bản nhưng ổn định

**Grep/Search**:
```bash
# Properties: Dễ grep
grep "database.url" application.properties

# YAML: Cần hiểu structure
grep -A5 -B5 "url:" application.yml
```

### 5.6 Bảng So Sánh và Quy Tắc Quyết Định

| Tiêu Chí | YAML | Properties | Khuyến Nghị |
|----------|------|------------|-------------|
| **Cấu trúc phân cấp** | ✅ Excellent | ❌ Poor | YAML cho config phức tạp |
| **Lists/Arrays** | ✅ Native | ❌ Verbose | YAML cho collections |
| **Multi-document** | ✅ Yes | ❌ No | YAML cho multiple profiles |
| **Parsing speed** | ❌ Slower | ✅ Faster | Properties cho performance-critical |
| **Error prone** | ⚠️ Indentation | ✅ Simple | Properties cho beginners |
| **Tooling** | ✅ Good | ✅ Good | Cả hai đều OK |
| **Readability** | ✅ Better | ⚠️ OK | YAML cho human-readable |
| **Grep/Search** | ⚠️ Complex | ✅ Easy | Properties cho ops/debugging |

**Quy Tắc Quyết Định**:

```yaml
# ✅ Chọn YAML khi:
- Cấu hình có cấu trúc phân cấp sâu
- Nhiều lists/arrays
- Cần multiple profiles trong 1 file
- Team có kinh nghiệm với YAML
- Ưu tiên readability

# ✅ Chọn Properties khi:
- Cấu hình đơn giản, ít phân cấp
- Performance parsing quan trọng
- Team mới với Spring Boot
- Cần grep/search thường xuyên
- Legacy system compatibility
```

---