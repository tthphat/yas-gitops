# Chart Values Reference

## Cấu trúc Chart

```
k8s/charts/
├── backend/              # Chart cha cho Java backend (Spring Boot) services
├── ui/                   # Chart cha cho frontend (NextJS/React) services
├── swagger-ui/           # Chart riêng cho Swagger UI
├── yas-configuration/    # Chart chứa ConfigMap/Secret dùng chung
│
├── backoffice-bff/       # Spring Boot gateway (dùng backend chart)
├── storefront-bff/       # Spring Boot gateway (dùng backend chart)
├── customer/             # Backend service (dùng backend chart)
├── cart/                 # Backend service (dùng backend chart)
├── inventory/            # Backend service (dùng backend chart)
├── media/                # Backend service (dùng backend chart)
├── order/                # Backend service (dùng backend chart)
├── product/              # Backend service (dùng backend chart)
├── search/               # Backend service (dùng backend chart)
├── tax/                  # Backend service (dùng backend chart)
├── sampledata/           # Backend service (dùng backend chart)
├── backoffice-ui/        # Frontend (dùng ui chart)
└── storefront-ui/        # Frontend (dùng ui chart)
```

---

## backend Chart — `k8s/charts/backend/values.yaml`

Dùng cho tất cả service Java Spring Boot (cart, customer, order, product, ...).

| Field | Type | Default | Mô tả |
|-------|------|---------|-------|
| `replicaCount` | int | `1` | Số pod replica |
| `image.repository` | string | `""` | Docker image repo |
| `image.tag` | string | `""` | Image tag (mặc định là `.Chart.AppVersion`) |
| `image.pullPolicy` | string | `IfNotPresent` | Pull policy |
| `imagePullSecrets` | list | `[dh-registry-secret]` | Secret để pull private image |
| `nameOverride` | string | `""` | Override tên resource |
| `fullnameOverride` | string | `""` | Override tên đầy đủ |
| `serviceAccount.create` | bool | `true` | Tạo service account |
| `httpPort` | int | `80` | Container port cho HTTP |
| `metricPort` | int | `8090` | Container port cho actuator metrics |
| `databaseConnectionUrl` | string | `jdbc:postgresql://postgresql.postgres:5432` | JDBC connection string |
| `databaseName` | string | `postgres` | Tên database |
| `logbackXmlPath` | string | `/opt/yas/config/logback.xml` | Path tới logback config |
| `deployment.annotations` | map | `{reloader.stakater.com/search: "true"}` | Annotations cho Deployment |
| `extraEnvs` | list | `[]` | Extra environment variables (VD: `SPRING_PROFILES_ACTIVE`, `SPRING_APPLICATION_NAME`) |
| `extraEnvFroms` | list | `[]` | Extra env from Secret/ConfigMap |
| `extraVolumes` | list | `[]` | Extra volumes mount vào pod |
| `extraVolumeMounts` | list | `[]` | Extra volume mounts |
| `extraApplicationConfigPaths` | list | `[]` | Extra Spring config paths |
| `replaceDefaultApplicationConfig` | bool | `false` | Nếu `true`, dùng `SPRING_CONFIG_LOCATION` thay vì `SPRING_CONFIG_ADDITIONAL_LOCATION` |
| `lifecycle.preStop` | hook | `sleep 10` | Graceful shutdown delay |
| `terminationGracePeriodSeconds` | int | `45` | Thời gian chờ pod terminate |
| `livenessProbe` | probe | `initialDelaySeconds: 90, periodSeconds: 10, failureThreshold: 12` | Liveness check (actuator/health/liveness port 8090) |
| `readinessProbe` | probe | `initialDelaySeconds: 90, periodSeconds: 10, failureThreshold: 12` | Readiness check (actuator/health/readiness port 8090) |
| `ingress.enabled` | bool | `false` | Bật/tắt ingress |
| `ingress.host` | string | `chart-example.local` | Hostname cho ingress |
| `ingress.path` | string | `/` | Path cho ingress |
| `ingress.pathType` | string | `ImplementationSpecific` | Path type |
| `resources` | map | `{}` | Resource requests/limits |
| `autoscaling.enabled` | bool | `false` | Bật autoscaling (HPA) |

### Backend Template: Deployment environment variables tự động

Biến môi trường tự động inject bởi template:

| Env | Giá trị |
|-----|---------|
| `SPRING_DATASOURCE_URL` | `{databaseConnectionUrl}/{databaseName}` |
| `SPRING_CONFIG_LOCATION` hoặc `SPRING_CONFIG_ADDITIONAL_LOCATION` | `/opt/yas/config/application.yaml,{extraApplicationConfigPaths}` |
| `LOGGING_CONFIG` | `/opt/yas/config/logback.xml` |
| Secrets từ `extraEnvFroms` + `yas-postgresql-credentials-secret` | |

---

## ui Chart — `k8s/charts/ui/values.yaml`

Dùng cho frontend services (backoffice-ui, storefront-ui).

| Field | Type | Default | Mô tả |
|-------|------|---------|-------|
| `replicaCount` | int | `1` | Số pod replica |
| `image.repository` | string | `""` | Docker image repo |
| `image.tag` | string | `""` | Image tag |
| `imagePullSecrets` | list | `[dh-registry-secret]` | Secret pull image |
| `httpPort` | int | `3000` | Container port (NextJS mặc định 3000) |
| `deployment.annotations` | map | `{reloader.stakater.com/search: "true"}` | Annotations |
| `extraEnvs` | list | `[]` | Extra env vars |
| `extraVolumes` | list | `[]` | Extra volumes |
| `extraVolumeMounts` | list | `[]` | Extra volume mounts |
| `ingress` | map | `enabled: false` | Ingress config (tương tự backend) |
| `service.type` | string | `ClusterIP` | Service type |
| `service.port` | int | `3000` | Service port |
| `autoscaling` | map | `enabled: false` | HPA config |

> **Lưu ý:** UI chart không có liveness/readiness probe mặc định, probe được hardcode trong template (httpGet `/` port http, không initialDelay).

---

## swagger-ui Chart — `k8s/charts/swagger-ui/values.yaml`

| Field | Type | Default | Mô tả |
|-------|------|---------|-------|
| `image.repository` | string | `swaggerapi/swagger-ui` | Image |
| `image.tag` | string | `v4.16.0` | Version |
| `baseUrl` | string | `/swagger-ui` | Base URL path |
| `apiBaseUrl` | string | `http://api.yas.local.com` | API backend URL |
| `ingress.enabled` | bool | `true` | Bật ingress |
| `ingress.host` | string | `api.yas.local.com` | Hostname |
| `ingress.hosts[].host` | string | `api.yas.local.com` | Hostname (dạng list) |
| `ingress.hosts[].paths` | list | `[{path: /swagger-ui, pathType: ImplementationSpecific}]` | Path rules |

---

## yas-configuration Chart — `k8s/charts/yas-configuration/values.yaml`

Chart này **không phải service** mà là chart tạo shared ConfigMap và Secret cho tất cả service.

### Credentials

| Field | Mô tả |
|-------|-------|
| `credentials.postgresql` | Username/password PostgreSQL |
| `credentials.elasticsearch` | Username/password Elasticsearch |
| `credentials.keycloak.*ClientSecret` | OAuth2 client secrets cho backoffice-bff, storefront-bff, customer |
| `credentials.redis.password` | Redis password |
| `credentials.openai.apiKey` | OpenAI API key |

### applicationConfig

Cấu hình Spring Boot chung cho toàn bộ service (server port 80, management port 8090, tracing OTLP, datasource, kafka, OAuth2 resource server, v.v.)

### gatewayRoutesConfig

Cấu hình Spring Cloud Gateway routes cho BFF (backoffice-bff, storefront-bff). Định nghĩa route mapping từ `/api/{service}` tới service nội bộ.

### Media/Customer/Search/etc. Application Configs

Cấu hình riêng cho từng service:
- `mediaApplicationConfig` — context path `/media`, public URL
- `customerApplicationConfig` — Keycloak auth config
- `searchApplicationConfig` — Elasticsearch URL
- `paymentPaypalApplicationConfig` — Public URL callback
- `recommendationApplicationConfig` — OpenAI Azure config
- `sampledataApplicationConfig` — Product/media datasource URLs

### reloader

Cấu hình cho Stakater Reloader (tự động restart pod khi ConfigMap/Secret thay đổi).

---

## Service-specific values.yaml (dev/staging)

Mỗi service có 2 file values:
- `values.yaml` — dùng cho **dev** (deploy vào namespace `dev`)
- `values.staging.yaml` — dùng cho **staging** (deploy vào namespace `staging`)

### Cấu trúc chung (backoffice-bff, storefront-bff làm VD)

```yaml
backend:              # namespace tương ứng chart dependency (backend hoặc ui)
  image:
    repository: thaithienphu/{service-name}
    tag: latest
  nameOverride: {service-name}
  fullnameOverride: {service-name}
  replaceDefaultApplicationConfig: true   # nếu service có custom config
  deployment:
    annotations:
      configmap.reloader.stakater.com/reload: "..."  # list ConfigMap để reload
  ingress:
    enabled: true
    host: {service}.yas.local.com          # dev
    # host: {service}.staging.yas.local.com  # staging
    path: /
  extraEnvs:
    - name: SPRING_PROFILES_ACTIVE
      value: k8s                           # activate k8s profile
    - name: SPRING_APPLICATION_NAME
      value: {service-name}
    - name: UI_HOST                        # chỉ cho BFF services
      value: http://{ui-service}.yas.svc.cluster.local:3000
  extraEnvFroms:
    - secretRef:
        name: yas-keycloak-credentials-secret
    - secretRef:
        name: yas-redis-credentials-secret
  extraVolumes:
    - name: {config-volume}
      configMap:
        name: {configmap-name}
  extraVolumeMounts:
    - name: {config-volume}
      mountPath: /opt/yas/{path}
  extraApplicationConfigPaths:
    - /opt/yas/{path}/{config-file}.yaml
```

### Các field quan trọng

| Field | Bắt buộc | Mô tả |
|-------|----------|-------|
| `backend.image.repository` | ✅ | Docker image trên Docker Hub |
| `backend.image.tag` | ✅ | Image tag (`latest`) |
| `backend.nameOverride` | ✅ | Tên service |
| `backend.ingress.enabled` | ✅ | Tạo ingress |
| `backend.ingress.host` | ✅ | Hostname (unique giữa dev/staging) |
| `backend.extraEnvs` | ✅ | `SPRING_PROFILES_ACTIVE=k8s` + `SPRING_APPLICATION_NAME` |
| `backend.extraEnvFroms` | Khi cần | Secret reference cho credentials |
| `backend.extraVolumes` | Khi có extra config | Mount ConfigMap |
| `backend.extraApplicationConfigPaths` | Khi có extra config | Path tới file config phụ |
| `backend.replaceDefaultApplicationConfig` | Khi service cần config riêng | `true` nếu override toàn bộ main config |

### CORS Proxy (Legacy)

Một số UI chart có:

```yaml
legacyAlias:
  enabled: true
  name: storefront-nextjs
```

Tạo Service alias cho tương thích ngược.

---

## Ingress Host Convention

| Môi trường | Host pattern | Ví dụ |
|-----------|-------------|-------|
| Dev | `*.yas.local.com` | `backoffice.yas.local.com`, `api.yas.local.com`, `storefront.yas.local.com` |
| Staging | `*.staging.yas.local.com` | `backoffice.staging.yas.local.com`, `api.staging.yas.local.com` |

---

## SPRING_PROFILES_ACTIVE=k8s

Bắt buộc cho tất cả backend service. Kích hoạt profile `k8s` để Spring Boot load config từ `application-k8s.yaml` (shared qua `yas-configuration` ConfigMap). Nếu thiếu, app fallback về `default` profile và không load đúng datasource/kafka/keycloak config.
