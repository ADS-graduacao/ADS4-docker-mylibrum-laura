# MyLibrum — Ambiente Docker

Sistema de gerenciamento de biblioteca (Projeto Aplicado, UniSENAI).

| Camada | Tecnologia | Porta no host |
|---|---|---|
| Frontend | Angular + nginx | 4200 |
| Backend | Spring Boot 4 (Java 21) | 8080 |
| Banco de dados | PostgreSQL 16 | 5433 |

## Estrutura esperada de pastas

Os dois repositórios precisam estar **lado a lado**, porque o compose usa o caminho relativo `../../FRONT/...` para construir o frontend.

```
qualquer-pasta/
├── BACK/
│   └── mylibrum/     <- docker-compose.yml, Dockerfile e código Spring Boot
└── FRONT/
    └── librum/       <- Dockerfile, nginx.conf e código Angular
```

## Pré-requisitos

- Docker Desktop ou Docker Engine com Compose v2 (`docker compose version`)
- Git
- Portas **4200**, **8080** e **5433** livres

## 1. Construir e iniciar

```bash
git clone https://github.com/Librum-SGB/BACK.git
git clone https://github.com/Librum-SGB/FRONT.git

cd BACK/mylibrum
docker compose up -d --build
```

O primeiro build leva alguns minutos (Maven e npm baixam as dependências). Acompanhe com:

```bash
docker compose ps
docker compose logs -f backend
```

O ambiente está pronto quando os três serviços aparecem como `Up` (o postgres como `healthy`) e o log do backend mostra `Started MylibrumApplication`.

## 2. Testar

| O que testar | Como |
|---|---|
| Interface | Abrir http://localhost:4200 e cadastrar uma bibliotecária |
| API | Abrir http://localhost:8080/swagger-ui.html |
| Banco de dados | `docker compose exec postgres psql -U postgres -d librumDb -c "\dt"` |
| Comunicação entre containers | `docker compose exec backend nslookup postgres` (o nome do serviço resolve para o IP interno) |
| Backend sem root | `docker compose exec backend id` (deve mostrar `uid=100(spring)`, nunca `uid=0`) |
| Logs | `docker compose logs --tail 50 backend` |

## 3. Encerrar

```bash
docker compose stop        # pausa os containers
docker compose down        # remove containers e rede, MANTÉM os dados do banco
docker compose down -v     # remove tudo, incluindo o volume (APAGA o banco)
```

## Serviços e rede

- O Compose cria a rede `mylibrum_default`. Os serviços se encontram pelo **nome do serviço**.
- `backend` -> `postgres:5432` (`SPRING_DATASOURCE_URL=jdbc:postgresql://postgres:5432/librumDb`).
- `frontend` -> depende do `backend`. O navegador acessa a API em `http://localhost:8080` (URL definida no build do Angular).
- O `backend` só inicia depois que o healthcheck do `postgres` (`pg_isready`) passa.
- O volume `mylibrum-postgres-data` guarda os dados do banco entre reinicializações.

## Banco de dados

A estrutura é criada automaticamente pelo **Flyway** quando o backend inicia. Os scripts SQL de criação e inicialização ficam em:

```
BACK/mylibrum/src/main/resources/db/
```

Não é preciso importar nenhum script manualmente. O histórico das migrations fica na tabela `flyway_schema_history`.

## Variáveis de ambiente

Todas têm valor padrão de **desenvolvimento** no `docker-compose.yml`, então o ambiente sobe sem configurar nada. Para alterar, crie um arquivo `.env` ao lado do `docker-compose.yml`:

| Variável | Padrão | Função |
|---|---|---|
| `POSTGRES_DB` | `librumDb` | Nome do banco |
| `POSTGRES_USER` | `postgres` | Usuário do banco |
| `POSTGRES_PASSWORD` | `root` | Senha do banco (apenas desenvolvimento) |
| `POSTGRES_PORT` | `5433` | Porta do banco no host |
| `BACKEND_PORT` | `8080` | Porta do backend no host |
| `FRONTEND_PORT` | `4200` | Porta do frontend no host |
| `JWT_SECRET` | valor de desenvolvimento | Chave de assinatura dos tokens JWT |
| `JWT_EXPIRATION_MS` | `3600000` | Validade do token (1 hora) |

> Estes valores são **somente para uso local**. Em produção, defina senha e `JWT_SECRET` próprios e não os versione.
>
> Se mudar `BACKEND_PORT`, o frontend precisa ser reconstruído, porque a URL da API é fixada no build.

## Boas práticas aplicadas

- **Multi-stage build** no frontend e no backend (imagem final sem compiladores).
- Backend executa como **usuário não-root** (`spring`).
- `.dockerignore` em ambos os projetos.
- **Healthcheck** no banco e `depends_on` com `service_healthy`.
- Dados persistidos em **volume nomeado**.
- Política de reinício `unless-stopped`.

## Solução de problemas

| Sintoma | Causa | Solução |
|---|---|---|
| `port is already allocated` | Outra aplicação usa a porta | Parar o outro container ou mudar a porta no `.env` |
| `container name ... is already in use` | Containers antigos do projeto | `docker rm -f mylibrum-postgres mylibrum-backend mylibrum-frontend` |
| Backend não conecta no banco após mudar a senha | O volume guarda a senha antiga | `docker compose down -v` e subir de novo |
| Falha no build do frontend | Código incompleto ou desatualizado | `git pull` nos dois repositórios e rodar `docker compose up -d --build` |

## Imagens publicadas no Docker Hub (extra)

- https://hub.docker.com/r/laurasavibressansenai/mylibrum-backend
- https://hub.docker.com/r/laurasavibressansenai/mylibrum-frontend

```bash
docker pull laurasavibressansenai/mylibrum-backend:1.0.0
docker pull laurasavibressansenai/mylibrum-frontend:1.0.0
```
