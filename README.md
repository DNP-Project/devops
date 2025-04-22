# DevOps Architecture Overview

Эта документация подробно описывает архитектуру инфраструктуры и DevOps-процессы для проекта **dnp-project** (Phonebook JSON-RPC сервис + React фронтенд). Основное внимание уделено настройке CI/CD, управлению Docker-контейнерами, сетевыми схемами, мониторингу и логированию.

---

## 1. Общая схема инфраструктуры

```text
Интернет ↔ Traefik (LB + TLS) ↔ Docker-сети (prod: infra_prod, stage: infra_stage)
                   │                ├── Prod: frontend, backend
                   │                └── Stage: frontend, backend
                   └── Infra: registry, prometheus, grafana, loki, promtail, cadvisor, node_exporter
```  

- **Traefik** в роли внешнего балансировщика и маршрутизатора HTTPS  
- **Prod и Stage** изолированы разными Docker-сетями: чтобы избежать конфликтов роутеров и сервисов
- **Infra**-сервисы в двух сетях сразу (`infra_prod`, `infra_stage`) для мониторинга обоих окружений

---

## 2. Docker-сети и разделение окружений

- `infra_prod`: внутренняя сеть для продакшн-контейнеров (frontend, backend)  
- `infra_stage`: внутренняя сеть для стейдж-контейнеров  
- Traefik и мониторинговый стек сидят в обеих сетях одновременно  

### Зачем разделять сети

- Избежать **конфликтов DNS-имен** контейнеров (одинаковые сервисы в прод и стейдж)
- Упрощает **маршрутизацию** и **ограничение доступа**
- Гибкость: можно запускать оба окружения на одном сервере без проблем

---

## 3. Traefik (v3.0) конфигурация

- Включена опция `--providers.docker.exposedbydefault=false`  
- `--providers.docker.network` указывают на обе сети  
- HTTP->HTTPS редирект и ACME с Let’s Encrypt  
- Маршрутизация по лейблам Docker сервисов  

### Основные entrypoints

- `web` (80) → редирект в `websecure`  
- `websecure` (443) → TLS

### Примеры лейблов для сервиса backend

```yaml
labels:
  - "traefik.enable=true"
  - "traefik.http.services.backend.loadbalancer.server.port=8000"
  - "traefik.http.routers.backend.rule=Host(`dnp-project.ru`) && (PathPrefix(`/rpc`) || PathPrefix(`/docs`) || PathPrefix(`/openapi.json`))"
  - "traefik.http.routers.backend.entrypoints=websecure"
  - "traefik.http.routers.backend.tls=true"
  - "traefik.http.routers.backend.tls.certresolver=letsencrypt"
  - "traefik.http.routers.backend.priority=100"
  - "traefik.http.routers.backend.service=backend"
  - "traefik.http.middlewares.strip-docs.stripprefix.prefixes=/docs,/redoc"
  - "traefik.http.routers.backend.middlewares=strip-docs"
```

---

## 4. CI/CD и Docker Registry

1. **GitHub Actions** или GitLab CI  
2. При пуше в `dev`/`main`:
   - Запуск линтеров, тестов, coverage  
   - Сборка Docker образа  
   - Тэгирование (`dev` → `registry.dnp-project.ru/backend:dev`; `main` → `...:latest`)  
   - Push в **личный Docker Registry** (Traefik проксирует `registry.dnp-project.ru`)  
3. **SSH Action**: автоматически `docker compose pull` + `up -d` на сервере

---

## 5. База данных

- **PostgreSQL** запускается отдельным контейнером  
- Volume для постоянного хранения данных  
- Подключается к backend через Docker-сеть

---

## 6. Мониторинг и логирование

- **Grafana & Prometheus**: метрики приложений и хостов  
- **cAdvisor & node_exporter**: метрики контейнеров и железа  
- **Loki + Promtail**: агрегирует логи контейнеров  
- Дашборды Grafana доступны по `grafana.dnp-project.ru`

---

## 7. Порядок деплоя

1. **Создать сети**:  
```bash
docker network create infra_prod
docker network create infra_stage
```
2. **Запустить Infra**:  
```bash
cd infra && docker compose up -d --build
```
3. **Запустить Prod**:  
```bash
cd prod && docker compose up -d --build
```
4. **Запустить Stage**:  
```bash
cd stage && docker compose up -d --build
```

---

## 8. Best Practices и советы

- **Идемпотентность**: `docker compose down && up -d --build` всегда чистит старые лейблы  
- **Секреты**: используйте Docker Secrets или Vault для паролей DB и ACME_EMAIL  
- **Мониторинг**: настраивайте алерты в Prometheus Alertmanager (например, падение контейнера)  
- **Бэкапы**: регулярный экспорт PostgreSQL через cron + хранение на S3-compatible хранилище

---

*Конец документации.*

