# Zabbix 7.0 LTS + PostgreSQL 16 (Docker Compose)

Stack para monitoramento com Zabbix Server, Zabbix Web (Nginx + PHP) e PostgreSQL 16, isolados em redes Docker separadas (banco de dados sem acesso externo).

## Serviços

| Serviço        | Imagem                                            | Porta exposta |
|----------------|----------------------------------------------------|---------------|
| zabbix-db      | postgres:16-alpine                                  | -             |
| zabbix-server  | zabbix/zabbix-server-pgsql:alpine-7.0-latest        | 10051         |
| zabbix-web     | zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest     | 8080          |

## Pré-requisitos

- Docker instalado
- Docker Compose v2 (`docker compose version`)

## Passo a passo

### 1. Clonar o repositório

```bash
git clone https://github.com/yurythx/zabbix.git
cd zabbix
```

### 2. Criar o arquivo `.env`

O arquivo `.env` guarda a senha do banco e **não é versionado** no Git (está no `.gitignore`). Copie o exemplo e edite:

```bash
cp .env.example .env
```

Abra o `.env` e troque a senha padrão por uma senha forte:

```env
POSTGRES_DB=zabbix
POSTGRES_USER=zabbix
POSTGRES_PASSWORD=SuaSenhaForteAqui!
```

> Se você pular esta etapa, o `docker-compose.yml` usa uma senha padrão fraca definida como fallback — não recomendado para produção.

### 3. Subir os containers

```bash
docker compose up -d
```

Isso cria as networks (`backend-net`, `frontend-net`), os volumes (`pgdata`, `zabbix-alertscripts`, `zabbix-externalscripts`) e inicia os 3 containers.

### 4. Aguardar a inicialização

Na primeira subida, o `zabbix-server` demora cerca de 1 minuto criando o schema no PostgreSQL. Acompanhe o log:

```bash
docker logs -f zabbix-server
```

Quando aparecer uma mensagem como `... started successfully`, está pronto.

### 5. Acessar a interface web

Abra [http://localhost:8080](http://localhost:8080)

- **Usuário:** `Admin`
- **Senha:** `zabbix`

> Troque a senha padrão do usuário `Admin` assim que fizer o primeiro login.

## Comandos úteis

```bash
# Ver status dos containers
docker compose ps

# Ver logs de um serviço
docker logs -f zabbix-server
docker logs -f zabbix-web
docker logs -f zabbix-db

# Parar os containers (mantém os dados)
docker compose down

# Parar e apagar volumes (apaga TODOS os dados do banco)
docker compose down -v

# Reiniciar apenas um serviço
docker compose restart zabbix-server
```

## Estrutura de rede

- `backend-net` (interna, sem saída para a internet): comunicação entre `zabbix-db` e `zabbix-server`.
- `frontend-net`: comunicação entre `zabbix-server` e `zabbix-web`, e acesso externo via porta 8080.

## Observações

- Os dados do PostgreSQL persistem no volume `pgdata` mesmo após `docker compose down`.
- Scripts de alerta/externos podem ser colocados nos volumes `zabbix-alertscripts` e `zabbix-externalscripts`.
- O fuso horário do container web está fixado em `America/Sao_Paulo` (variável `PHP_TZ`).
