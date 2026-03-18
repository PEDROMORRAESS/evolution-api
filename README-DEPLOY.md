# Evolution API v2.1.1 - Guia de Deploy

## Arquitetura

```
Internet
   │
   ▼
Traefik v3.4.0 (já existente)
   │
   ├──► api2.crescentiagroup.com.br ──► evolution_api:8080
   └──► manager2.crescentiagroup.com.br ──► evolution_manager:80
         │
    evolution_network (overlay isolado)
         ├── evolution_postgres:5432
         └── evolution_redis:6379
```

**Isolamento total:** A stack usa rede própria `evolution_network` e não interfere nos serviços existentes (n8n, Chatwoot, etc).

---

## Pré-requisitos

- Docker Swarm inicializado (`docker swarm init`)
- Traefik rodando com rede `traefik-public`
- DNS configurado:
  - `api2.crescentiagroup.com.br` → IP da VPS
  - `manager2.crescentiagroup.com.br` → IP da VPS

---

## Configurar GitHub Actions (CI/CD automático)

### Secrets necessários no GitHub

Vá em **Settings → Secrets and variables → Actions** e adicione:

| Secret | Descrição | Exemplo |
|--------|-----------|---------|
| `SSH_PRIVATE_KEY` | Chave SSH privada para acessar a VPS | `-----BEGIN OPENSSH PRIVATE KEY-----...` |
| `VPS_HOST` | IP ou hostname da VPS | `123.456.789.0` |
| `VPS_USER` | Usuário SSH | `root` |
| `SERVER_URL` | URL da Evolution API | `https://api2.crescentiagroup.com.br` |
| `AUTHENTICATION_API_KEY` | Chave de autenticação (32+ chars) | `abc123...` |
| `POSTGRES_DB` | Nome do banco | `evolution_db` |
| `POSTGRES_USER` | Usuário do banco | `evolution_user` |
| `POSTGRES_PASSWORD` | Senha do banco (16+ chars) | `SenhaSegura123!` |

### Gerar chaves seguras

```bash
# API Key (32 bytes hex = 64 caracteres)
openssl rand -hex 32

# Postgres Password (16 bytes base64)
openssl rand -base64 16
```

### Gerar par de chaves SSH (se não tiver)

```bash
# Na sua máquina local
ssh-keygen -t ed25519 -C "github-actions-evolution" -f ~/.ssh/evolution_deploy

# Copiar chave pública para a VPS
ssh-copy-id -i ~/.ssh/evolution_deploy.pub root@SEU_IP_VPS

# O conteúdo de ~/.ssh/evolution_deploy vai no secret SSH_PRIVATE_KEY
cat ~/.ssh/evolution_deploy
```

---

## Deploy

### Automático (recomendado)

Basta fazer push para `main`:

```bash
git add .
git commit -m "deploy: evolution api v2.1.1"
git push origin main
```

O GitHub Actions vai:
1. Conectar na VPS via SSH
2. Criar o `.env` com as variáveis dos Secrets
3. Copiar o `evolution-stack.yml`
4. Criar a rede `evolution_network` se não existir
5. Fazer `docker stack deploy`
6. Verificar o status dos serviços

### Manual (na VPS)

```bash
cd /root/evolution

# Criar .env com suas variáveis
cp /caminho/do/repo/.env.example .env
nano .env  # Preencher os valores

# Criar rede (se não existir)
docker network create --driver overlay --attachable evolution_network

# Deploy
set -a && source .env && set +a
docker stack deploy --compose-file evolution-stack.yml --with-registry-auth evolution
```

---

## Verificação pós-deploy

### Listar serviços

```bash
docker stack services evolution
```

Saída esperada (todos com `1/1`):
```
ID      NAME                        MODE    REPLICAS  IMAGE
xxxx    evolution_evolution_api     replicated  1/1   atendai/evolution-api:v2.1.1
xxxx    evolution_evolution_manager replicated  1/1   atendai/evolution-manager:latest
xxxx    evolution_evolution_postgres replicated  1/1  postgres:15-alpine
xxxx    evolution_evolution_redis   replicated  1/1   redis:7-alpine
```

### Ver tasks e erros

```bash
docker stack ps evolution
docker stack ps evolution --no-trunc  # Ver erros completos
```

### Testar API

```bash
# Health check (substitua pela sua API KEY)
curl -s https://api2.crescentiagroup.com.br/ | jq .

# Listar instâncias
curl -s -H "apikey: SUA_API_KEY" \
  https://api2.crescentiagroup.com.br/instance/fetchInstances | jq .
```

---

## Logs

```bash
# Evolution API
docker service logs evolution_evolution_api -f --tail 100

# Postgres
docker service logs evolution_evolution_postgres -f --tail 50

# Redis
docker service logs evolution_evolution_redis -f --tail 50

# Manager
docker service logs evolution_evolution_manager -f --tail 50
```

---

## Conectar WhatsApp (QR Code)

### 1. Criar instância

```bash
curl -X POST https://api2.crescentiagroup.com.br/instance/create \
  -H "apikey: SUA_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "instanceName": "minha-instancia",
    "integration": "WHATSAPP-BAILEYS"
  }'
```

### 2. Obter QR Code (base64)

```bash
curl -s https://api2.crescentiagroup.com.br/instance/connect/minha-instancia \
  -H "apikey: SUA_API_KEY" | jq .
```

### 3. Via Manager Web

Acesse `https://manager2.crescentiagroup.com.br` e use a interface gráfica.

---

## Troubleshooting

### Serviço não sobe (0/1 replicas)

```bash
# Ver erro detalhado
docker stack ps evolution --no-trunc | grep -i shutdown

# Ver logs do container
docker service logs evolution_evolution_api --tail 50
```

**Causa comum:** Postgres ainda inicializando. Aguarde 60s e verifique novamente.

### Erro de rede Traefik

Verifique se a rede `traefik-public` existe:
```bash
docker network ls | grep traefik
```

Se o nome for diferente, edite `evolution-stack.yml` e atualize:
```yaml
networks:
  traefik-public:
    external: true
    name: NOME_REAL_DA_REDE_TRAEFIK
```

### Erro de certificado SSL

Verifique se o DNS está propagado:
```bash
nslookup api2.crescentiagroup.com.br
```

### Remover e redeployar

```bash
# Remover stack (mantém volumes)
docker stack rm evolution

# Aguardar remoção completa
sleep 15

# Redeployar
set -a && source .env && set +a
docker stack deploy --compose-file evolution-stack.yml evolution
```

### Remover dados (CUIDADO - irreversível)

```bash
docker stack rm evolution
docker volume rm evolution_postgres_data evolution_redis_data evolution_instances_data
```

---

## URLs de Acesso

| Serviço | URL |
|---------|-----|
| Evolution API | https://api2.crescentiagroup.com.br |
| Evolution Manager | https://manager2.crescentiagroup.com.br |
| Swagger/Docs | https://api2.crescentiagroup.com.br/docs |

---

## Versão

- **Evolution API:** `atendai/evolution-api:v2.1.1` (estável - não alterar)
- **Postgres:** `postgres:15-alpine`
- **Redis:** `redis:7-alpine`
- **Manager:** `atendai/evolution-manager:latest`
