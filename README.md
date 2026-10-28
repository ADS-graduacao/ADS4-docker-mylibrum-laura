# MyLibrum com Docker

Este repositório contém o frontend e o backend do MyLibrum, além da configuração
para subir a aplicação com PostgreSQL usando Docker Compose.

1. Acesse `mylibrum-docker` e copie `.env.example` para `.env`.
2. Preencha `POSTGRES_PASSWORD` e `JWT_SECRET` no `.env`.
3. Execute `docker compose up --build` nessa pasta.

O Compose constrói as imagens localmente usando os projetos em `BACK/mylibrum`
e `FRONT/Librum`.