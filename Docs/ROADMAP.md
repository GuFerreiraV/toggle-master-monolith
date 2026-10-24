# ROADMAP — Tech Challenge Fase 1 (ToggleMaster)

Guia de apoio para a apresentação e para o deploy na AWS. Ordem sugerida do vídeo (até 15 min):

1. Aplicação rodando localmente (Docker)
2. Análise: monolito e 12-Factor App
3. Diagrama de arquitetura
4. Provisionamento e deploy na AWS
5. Validação: app pública, conectada ao RDS, Security Groups e credenciais

---

## 1. Aplicação local

**O que é:** API REST de feature flags em Python (Flask), servida pelo Gunicorn, com PostgreSQL.

```bash
docker compose up --build                            # sobe app + db
curl http://localhost:5000/health                    # {"status": "ok"}
docker compose exec db psql -U user -d togglemaster  # \dt, SELECT * FROM flags;
```

**Pontos para falar:**
- `entrypoint.sh`: espera o banco ficar disponível (`pg_isready`), cria a tabela (`flask init-db`) e inicia o Gunicorn.
- Gunicorn: servidor WSGI de produção que roda o Flask (`app:app`) na porta 5000.
- A raiz `/` retorna 404 porque é uma API. As rotas são `/health` e `/flags`.

| Método | Rota | Função |
|---|---|---|
| GET | `/health` | Status da API |
| GET / POST | `/flags` | Lista / cria flags |
| GET / PUT | `/flags/<nome>` | Consulta / atualiza uma flag |

---

## 2. Análise da aplicação

### Por que é um monolito
- Toda a funcionalidade está em **uma única base de código** (`app.py`).
- Roda em **um único processo** (Gunicorn) e é entregue como **uma única unidade de deploy**.
- Qualquer alteração exige redeploy da aplicação inteira, e para escalar é preciso replicar a aplicação completa.
- Reforço: um único banco PostgreSQL, acessado diretamente pelas rotas, sem separação de camadas.
- Ressalva: hoje existe um único domínio (flags). O que caracteriza o monolito é a plataforma inteira (MVP) estar nessa unidade, e as próximas funcionalidades entrariam nela também.

| Vantagens (para um MVP) | Desvantagens |
|---|---|
| Deploy simples | Qualquer mudança exige redeploy completo |
| Fácil de rodar e testar localmente | Uma falha pode derrubar toda a aplicação |
| Sem latência de rede entre componentes | Escala apenas como um todo |
| Entrega rápida para validar a ideia | O acoplamento tende a crescer com o tempo |

### 12-Factor App

**Atendidos**

| Fator | Evidência |
|---|---|
| I. Base de código | Um repositório git, implantado em vários ambientes (local e EC2) |
| II. Dependências | `requirements.txt` com versões fixas + isolamento (container / venv na EC2) |
| III. Configurações | `os.getenv()` no `app.py`. Local: valores do `docker-compose.yaml`. AWS: `export` na EC2 |
| IV. Serviços de apoio | O banco é um recurso anexado: trocar o Postgres local pelo RDS exige só mudar o `DB_HOST` |
| VI. Processos | Stateless: cada requisição abre sua conexão, e o estado fica no banco |
| VII. Vínculo de porta | O Gunicorn expõe a app sozinho em `0.0.0.0:5000` |

**Precisam de ajuste**

| Fator | Problema |
|---|---|
| V. Build, release, run | Sem versionamento de releases. Na EC2 o código vem de `git clone` direto, sem artefato buildado |
| VIII. Concorrência | Gunicorn com 1 worker e sem separação por tipo de processo |
| IX. Descartabilidade | Inicia rápido, mas o loop de espera do banco não tem timeout |
| X. Paridade dev/prod | Dev roda em Docker; prod roda direto na EC2 (venv), sem Docker |
| XI. Logs | `print` + logs do Gunicorn; erros das rotas vão para a resposta HTTP, não para o log |
| XII. Processos de admin | `init-db` roda automaticamente a cada start (local), em vez de ser pontual |

---

## 3. Diagrama de arquitetura

Arquivo: `Docs/diagram-toggle-master.drawio`

### Componentes

| Componente | Configuração | Por quê |
|---|---|---|
| Região | us-east-2 (Ohio) | Escolha de região; todas as AZs pertencem a ela |
| VPC | `10.0.0.0/16` | Rede isolada do projeto |
| Internet Gateway | Anexado à VPC | Única porta de entrada/saída da VPC para a internet |
| Public subnet | `10.0.1.0/24` (us-east-2a) | Onde fica a EC2; tem rota para o IGW |
| Private subnet A | `10.0.2.0/24` (us-east-2a) | Onde fica o RDS; sem rota para a internet |
| Private subnet B | `10.0.3.0/24` (us-east-2b) | Vazia. Existe porque o DB Subnet Group exige 2 AZs |
| DB Subnet Group | Private A + Private B | Pré-requisito da AWS para criar o RDS |
| EC2 | IP público, Gunicorn na 5000 | Roda a API |
| RDS PostgreSQL | Single-AZ, acesso público: **Não** | Banco de dados gerenciado |

### Route tables

| Tabela | Rotas | Associada a |
|---|---|---|
| Public | `10.0.0.0/16 → local`, `0.0.0.0/0 → IGW` | Public subnet |
| Private | `10.0.0.0/16 → local` (sem internet) | Private subnets A e B |

**Pergunta provável: "o que torna uma subnet pública?"** A rota `0.0.0.0/0 → IGW` na route table dela. Não é o nome da subnet.

### Security Groups

**SG-EC2 (entrada)**

| Porta | Origem | Motivo |
|---|---|---|
| TCP 80 | 0.0.0.0/0 | HTTP (requisito do desafio) |
| TCP 443 | 0.0.0.0/0 | HTTPS (requisito do desafio) |
| TCP 5000 | 0.0.0.0/0 | Porta onde o Gunicorn escuta: é por ela que a API é acessada |
| TCP 22 | MEU_IP/32 | SSH apenas do IP do administrador |

**SG-RDS (entrada)**

| Porta | Origem | Motivo |
|---|---|---|
| TCP 5432 | SG-EC2 | Só a EC2 acessa o banco |

**Pontos para falar:**
- **Origem = Security Group, não IP:** a regra continua valendo se o IP da EC2 mudar ou se novas instâncias forem adicionadas ao SG-EC2.
- **Defesa em camadas no RDS:** subnet privada (sem rota para a internet) + acesso público desativado + SG restrito à EC2.
- **SSH restrito:** um IP diferente de `MEU_IP` recebe *timeout*, porque o SG descarta o pacote sem responder.
- **80/443 x 5000:** 80 e 443 atendem o requisito do desafio, mas a API responde na 5000. Em produção, um proxy reverso (Nginx) na 80/443 permitiria fechar a 5000.
- **SGs são stateful:** a resposta de uma conexão permitida volta automaticamente, sem regra de saída específica.

### Fluxo de uma requisição
```
Usuário → Internet → IGW → Route Table Public → SG-EC2 (5000) → EC2/Gunicorn → Flask
       → SG-RDS (5432, origem SG-EC2) → RDS PostgreSQL → resposta faz o caminho inverso
```

### Decisões tomadas (para o relatório)
- **Segunda subnet privada:** não está no enunciado, mas sem ela o console não deixa criar o RDS.
- **Sem NAT Gateway:** o RDS é gerenciado e não precisa de saída para a internet. Também evita custo por hora.
- **Single-AZ:** suficiente para um MVP e mais barato. Multi-AZ seria o próximo passo para alta disponibilidade.
- **Porta 5000 liberada:** necessária para validar a API, conforme o README.

---

## 4. Provisionamento na AWS (ordem)

1. **VPC** `10.0.0.0/16`
2. **Subnets:** pública `10.0.1.0/24` (2a), privada A `10.0.2.0/24` (2a), privada B `10.0.3.0/24` (2b)
3. **Internet Gateway** → criar e anexar à VPC
4. **Route tables:** pública com `0.0.0.0/0 → IGW`, associada à subnet pública; privada só com `local`, associada às privadas
5. **Subnet pública:** ativar *auto-assign public IPv4*
6. **Security Groups:** criar o SG-EC2 primeiro, depois o SG-RDS (5432 ← SG-EC2)
7. **DB Subnet Group** com as privadas A e B
8. **RDS PostgreSQL:** Single-AZ, *Public access = No*, SG-RDS, **definir "Initial database name"** (ex.: `togglemaster`)
9. **EC2** na subnet pública, com SG-EC2 e key pair (.pem)

> Atalho: o assistente **"VPC and more"** cria VPC, subnets em 2 AZs, IGW e route tables de uma vez.

---

## 5. Deploy na EC2

```bash
ssh -i chave.pem ec2-user@<ip-publico>        # Ubuntu: ubuntu@<ip-publico>

sudo yum install -y git python3 python3-pip   # Amazon Linux
git clone <url-do-repositorio> && cd toggle-master-monolith
python3 -m venv venv && source venv/bin/activate
pip install -r requirements.txt

export DB_HOST='<endpoint-do-rds>'
export DB_NAME='togglemaster'
export DB_USER='<usuario>'
export DB_PASSWORD='<senha>'

flask --app app init-db                       # cria a tabela flags (o README não menciona!)
gunicorn --bind 0.0.0.0:5000 app:app
```

### Armadilhas conhecidas
- **Tabela não criada:** o README pula o `init-db` (localmente o `entrypoint.sh` fazia isso). Sem ele, as rotas `/flags` retornam erro 500 (`relation "flags" does not exist`).
- **Initial database name vazio:** o RDS cria só o banco `postgres`, e `DB_NAME=togglemaster` falha.
- **Versão do Python:** `psycopg2-binary==2.9.5` não tem pacote pronto para Python 3.12+. Use Amazon Linux 2023 (3.9) ou Ubuntu 22.04 (3.10); evite Ubuntu 24.04.
- **Timeout no SSH:** confira se o IP atual ainda é o `MEU_IP` da regra (IP residencial muda).
- **Timeout na porta 5000:** confira a regra da 5000 no SG-EC2 e se o Gunicorn está com `0.0.0.0`.
- **App cai ao fechar o SSH:** o Gunicorn roda em foreground (aceitável no desafio; em produção, usar `systemd`).

---

## 6. Validação (roteiro do vídeo)

```bash
curl http://<ip-publico>:5000/health

curl -X POST -H "Content-Type: application/json" \
  -d '{"name": "new-feature", "is_enabled": true}' \
  http://<ip-publico>:5000/flags

curl http://<ip-publico>:5000/flags
```

- [ ] `/health` responde pelo IP público → **app acessível publicamente**
- [ ] `POST` + `GET /flags` funcionando → **app conectada ao RDS**
- [ ] Mostrar o SG-EC2 (80, 443, 5000, 22 ← MEU_IP) e o SG-RDS (5432 ← SG-EC2)
- [ ] Mostrar o RDS com *Public access: No* e na subnet privada
- [ ] Credenciais: só em variáveis de ambiente (`export`), nunca no código nem no git

---

## 7. Estimativa de custos (AWS Pricing Calculator)

Ferramenta: https://calculator.aws → **Create estimate** → região **US East (Ohio)**.

| Serviço na calculadora | Configuração |
|---|---|
| Amazon EC2 | Linux, `t3.micro`, On-Demand, 730 h/mês, EBS gp3 8 GB |
| Amazon RDS for PostgreSQL | `db.t3.micro`, Single-AZ, On-Demand, 730 h/mês, gp3 20 GB, sem backup extra |
| Amazon VPC | 1 IPv4 público (In-use public IPv4 address), 730 h/mês |
| Data Transfer | Saída para a internet ~1 GB/mês (fica dentro do gratuito de 100 GB) |

Custos que **não** entram: VPC, subnets, route tables, Internet Gateway, Security Groups e DB Subnet Group são gratuitos. NAT Gateway não foi usado.

**Valores da estimativa** (calculadora, 19/10/2026):

| Item | US$/mês |
|---|---|
| Amazon EC2 (t3.micro, 100%) | 7,59 |
| Amazon RDS for PostgreSQL (db.t3.micro Single-AZ, 730 h + gp3 20 GB) | 15,44 |
| Amazon VPC (1 IPv4 público) | 3,65 |
| AWS Data Transfer (1 GB entrada / 1 GB saída) | 0,09 |
| **Total mensal** | **26,77** |
| **Total 12 meses** | **321,24** |

- Preços On-Demand, sem considerar Free Tier/créditos. Em conta elegível ao Free Tier, o custo efetivo pode ser zero.
- Link da estimativa: https://calculator.aws/#/estimate?id=2bba7de997249160f940e62f9b9cef3da75bdb5e
- PDF exportado: `Docs/estimativa-custos-aws.pdf`

---

## 8. Pendências da entrega

- [x] Estimativa de custo mensal na **AWS Pricing Calculator** (seção 7)
- [ ] Print da estimativa no relatório
- [ ] Link do diagrama (diagrams.net)
- [ ] Vídeo de até 15 min
- [ ] Relatório (.pdf/.txt): participantes, links, desafios encontrados e decisões tomadas (seção 3)
