# Zabbix 7.0 LTS + PostgreSQL 16 + Grafana (Docker Compose)

Stack para monitoramento com Zabbix Server, Zabbix Web (Nginx + PHP), PostgreSQL 16 e Grafana (dashboards), isolados em redes Docker separadas (banco de dados sem acesso externo).

## Serviços

| Serviço         | Imagem                                            | Porta exposta |
|-----------------|----------------------------------------------------|---------------|
| zabbix-db       | postgres:16-alpine                                  | -             |
| zabbix-server   | zabbix/zabbix-server-pgsql:alpine-7.0-latest        | 10051         |
| zabbix-web      | zabbix/zabbix-web-nginx-pgsql:alpine-7.0-latest     | 8080          |
| zabbix-grafana  | grafana/grafana-oss:latest                          | 3000          |

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

Se as portas padrão (`8080` e `10051`) já estiverem em uso no servidor (comum em ambientes com outros serviços rodando), ajuste no `.env`:

```env
ZABBIX_WEB_PORT=8081
ZABBIX_SERVER_PORT=10052
GRAFANA_PORT=3001
```

Para checar o que já está usando uma porta antes de subir:

```bash
sudo ss -tulpn | grep 8080
```

### 3. Subir os containers

```bash
docker compose up -d
```

Isso cria as networks (`backend-net`, `frontend-net`), os volumes (`pgdata`, `zabbix-alertscripts`, `zabbix-externalscripts`, `grafana-data`) e inicia os 4 containers.

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

### 6. Acessar o Grafana e conectar ao Zabbix

Abra [http://localhost:3000](http://localhost:3000)

- **Usuário:** valor de `GRAFANA_ADMIN_USER` (padrão `admin`)
- **Senha:** valor de `GRAFANA_ADMIN_PASSWORD` definido no `.env`

O plugin **Zabbix** (`alexanderzobnin-zabbix-app`) já vem instalado automaticamente na subida do container. Para configurar a fonte de dados:

1. No Grafana, vá em **Connections → Data sources → Add data source**.
2. Escolha **Zabbix**.
3. Em **HTTP → URL**, use `http://zabbix-web:8080/api_jsonrpc.php` (nome do container na rede interna, não `localhost`).
4. Em **Zabbix API details**, informe o usuário/senha do Zabbix (`Admin` / a senha que você definiu no primeiro login).
5. Clique em **Save & test**.

Depois é só importar ou criar dashboards usando os hosts/itens monitorados pelo Zabbix.

## Comandos úteis

```bash
# Ver status dos containers
docker compose ps

# Ver logs de um serviço
docker logs -f zabbix-server
docker logs -f zabbix-web
docker logs -f zabbix-db
docker logs -f zabbix-grafana

# Parar os containers (mantém os dados)
docker compose down

# Parar e apagar volumes (apaga TODOS os dados do banco)
docker compose down -v

# Reiniciar apenas um serviço
docker compose restart zabbix-server
```

## SSO/SAML (opcional)

O `zabbix-web` já vem preparado para SSO/SAML via variável de ambiente. Para habilitar:

1. Coloque o certificado do seu IdP em `./certs/idp.crt` (a pasta `certs/` já existe no repositório, mas os certificados reais **não são versionados** — veja o `.gitignore`).
2. No `.env`, defina:

   ```env
   ZBX_SSO_SETTINGS='{"baseurl": "http://SEU_IP_OU_DOMINIO:PORTA"}'
   ```

3. Suba/reinicie: `docker compose up -d`.

Quem não usa SSO simplesmente deixa `ZBX_SSO_SETTINGS` de fora do `.env` — o valor padrão é vazio e não afeta o funcionamento normal do Zabbix.

## Customizações específicas de um servidor (avançado)

Para qualquer outra configuração local que não deva ir para o repositório compartilhado e que não seja coberta por uma variável de ambiente, crie um `docker-compose.override.yml` (já listado no `.gitignore`) na mesma pasta — o Docker Compose mescla esse arquivo automaticamente com o `docker-compose.yml` principal ao rodar `docker compose up -d`, sem gerar conflito em `git pull`.

## Estrutura de rede

- `backend-net` (interna, sem saída para a internet): comunicação entre `zabbix-db` e `zabbix-server`.
- `frontend-net`: comunicação entre `zabbix-server`, `zabbix-web` e `grafana`, e acesso externo via portas 8080/3000.

## Observações

- Os dados do PostgreSQL persistem no volume `pgdata` mesmo após `docker compose down`.
- Os dashboards e configurações do Grafana persistem no volume `grafana-data`.
- Scripts de alerta/externos podem ser colocados nos volumes `zabbix-alertscripts` e `zabbix-externalscripts`.
- O fuso horário do container web está fixado em `America/Sao_Paulo` (variável `PHP_TZ`).
