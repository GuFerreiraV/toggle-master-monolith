# Anotações — Toggle Master

## Docker

```bash
docker compose up --build    # builda a imagem e sobe app + db (logs no terminal)
docker compose up -d         # sobe em segundo plano
docker compose logs -f app   # acompanha os logs da aplicação
docker compose ps            # lista os containers rodando
docker compose down          # para e remove os containers (dados do banco são mantidos no volume)
docker compose down -v       # idem, mas apaga também o volume do banco
```

- API: `http://localhost:5000` (Gunicorn)
- Banco: só acessível dentro da rede do Docker (porta 5432 não exposta)

## PostgreSQL (psql)

`psql` = cliente de linha de comando do PostgreSQL. Já vem na imagem `postgres:13`.

Com os containers rodando:

```bash
docker compose exec db psql -U user -d togglemaster
```

Dentro do psql (prompt `togglemaster=#`):

```sql
\dt                   -- lista as tabelas
\d flags              -- estrutura da tabela flags
SELECT * FROM flags;  -- SQL sempre termina com ;
\q                    -- sair
```

> Comandos do psql usam contrabarra (`\dt`), não barra (`/dt`).

## Endpoints

A raiz `/` não tem rota → retorna 404 (normal, é uma API).

| Método | Rota            | O que faz                    | Body (JSON)                                   |
|--------|-----------------|------------------------------|-----------------------------------------------|
| GET    | `/health`       | Verifica se a API está no ar | —                                             |
| GET    | `/flags`        | Lista todas as flags         | —                                             |
| POST   | `/flags`        | Cria uma flag                | `{"name": "new-feature", "is_enabled": true}` |
| GET    | `/flags/<nome>` | Mostra o status de uma flag  | —                                             |
| PUT    | `/flags/<nome>` | Atualiza uma flag            | `{"is_enabled": false}`                       |

Exemplos com `curl`:

```bash
curl http://localhost:5000/health

curl -X POST -H "Content-Type: application/json" \
  -d '{"name": "new-feature", "is_enabled": true}' \
  http://localhost:5000/flags

curl http://localhost:5000/flags/new-feature

curl -X PUT -H "Content-Type: application/json" \
  -d '{"is_enabled": false}' \
  http://localhost:5000/flags/new-feature
```
