# 📁 Demo Didática — Arquitetura S3 + Lambda + RDS

Repositório de apoio para uma **aula introdutória de Arquitetura de Soluções**.
Demonstra, na prática, como três serviços AWS se combinam para resolver um
problema real de armazenamento e catalogação de arquivos.

> ⚠️ **Material didático.** Este projeto foi desenvolvido exclusivamente para
> **demonstrar conceitos básicos de Arquitetura de Soluções** em sala de aula.
> Não é um sistema pronto para produção: não há autenticação robusta, não é
> multi-tenant e a topologia foi simplificada de propósito para destacar o
> papel de cada serviço AWS.

---

## 🎓 Objetivo da aula

Ao final da leitura (e do deploy opcional), o aluno deve ser capaz de:

1. Explicar o papel de cada serviço: **S3** (armazenamento de objetos),
   **Lambda** (compute serverless) e **RDS** (persistência relacional).
2. Justificar **por que separar** arquivo (S3) de metadado (RDS) em vez de
   guardar tudo no banco ou tudo em disco.
3. Reconhecer **trade-offs de simplicidade vs. robustez** — o que foi
   deixado de fora de propósito e por quê.
4. Identificar **quais peças seriam adicionadas** para levar este desenho a
   um cenário de produção (VPC, autenticação, CDN, filas, cache).

---

## 🧭 Contexto do problema (hipotético, usado como pano de fundo)

**João** é fotógrafo freelancer e acumula milhares de arquivos pesados
(RAW, PDFs, contratos) espalhados entre HD externo, e-mails e pen drives.
Ele quer **guardar os arquivos barato**, **catalogar por cliente/data/tag** e
**ter uma API simples** para automatizar upload/download — sem depender de
serviços prontos nem gerenciar servidor.

O cenário é apenas o gatilho narrativo da aula. O foco está nos **conceitos
arquiteturais**, não na solução em si.

---

## 🎯 Por que S3 + Lambda + RDS resolvem

| Necessidade | Como a arquitetura atende |
|---|---|
| Guardar arquivos pesados barato | **S3** — centavos por GB, sem servidor dedicado |
| Metadados pesquisáveis | **RDS** — SQL puro, queries por cliente/data/tag |
| Lógica sem gerenciar servidor | **Lambda** — só roda quando chamada, custo ~zero em uso pessoal |
| API simples | **Lambda + Function URL** |
| Baixo custo total | RDS `db.t3.micro` + S3 standard + Lambda no free tier |

---

## 🏗️ Arquitetura

```
┌──────────┐   HTTP    ┌────────────┐   SQL   ┌──────────┐
│ Cliente  │ ────────▶ │  Lambda    │ ──────▶ │   RDS    │
│ (curl /  │           │ (CRUD API) │         │ (dados)  │
│  script) │           │            │         └──────────┘
└──────────┘           │            │   Put/Get ┌──────────┐
                       │            │ ────────▶ │    S3    │
                       └────────────┘           │(arquivos)│
                                                └──────────┘
```

**Conceito central:** separar **o que é dado estruturado** (vai pro RDS)
do **que é binário/arquivo** (vai pro S3). Essa divisão é a ideia-chave da
aula — o resto é consequência.

**Fluxo básico:**

1. Cliente chama a Lambda (via Function URL) com uma requisição HTTP
2. Lambda processa a lógica de negócio
3. Metadados → **RDS** (PostgreSQL)
4. Arquivo em si → **S3**
5. Resposta volta ao cliente

---

## 🚀 Endpoints planejados

| Método | Rota | Descrição |
|---|---|---|
| `POST` | `/files` | Faz upload de arquivo (S3) + salva metadados (RDS) |
| `GET` | `/files` | Lista arquivos, com filtros por tag/cliente/data |
| `GET` | `/files/{id}` | Retorna URL pré-assinada para download |
| `DELETE` | `/files/{id}` | Remove arquivo do S3 e metadados do RDS |

---

## 🧱 Stack

- **Runtime:** Node.js 20 (ou Python 3.12)
- **Banco:** Amazon RDS PostgreSQL (`db.t3.micro`)
- **Armazenamento:** Amazon S3 (bucket privado)
- **Compute:** AWS Lambda + Function URL
- **IaC:** AWS CDK em TypeScript (ou Serverless Framework)

---

## 📦 Estrutura do repositório

```
.
├── src/
│   ├── handlers/
│   │   ├── upload.ts       # POST /files
│   │   ├── list.ts         # GET  /files
│   │   ├── download.ts     # GET  /files/{id}
│   │   └── delete.ts       # DELETE /files/{id}
│   ├── db/
│   │   └── client.ts       # pool de conexão RDS
│   └── s3/
│       └── client.ts       # wrapper do SDK S3
├── infra/
│   └── stack.ts            # AWS CDK
├── schema.sql              # DDL inicial
├── package.json
└── README.md
```

---

## 🗄️ Schema inicial (RDS)

```sql
CREATE TABLE files (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  filename     TEXT NOT NULL,
  s3_key       TEXT NOT NULL,
  content_type TEXT,
  size_bytes   BIGINT,
  client       TEXT,
  tags         TEXT[],
  uploaded_at  TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_files_client ON files(client);
CREATE INDEX idx_files_tags   ON files USING GIN(tags);
```

---

## ⚙️ Setup local

```bash
# 1. Clone
git clone https://github.com/seu-usuario/demo-s3-lambda-rds.git
cd demo-s3-lambda-rds

# 2. Instale dependências
npm install

# 3. Configure credenciais AWS
export AWS_PROFILE=default
export AWS_REGION=us-east-1

# 4. Deploy da infraestrutura
npm run deploy

# 5. Teste rápido
curl -X POST $FUNCTION_URL/files \
  -F "file=@foto.jpg" \
  -F "client=Cliente X" \
  -F "tags=foto,2024,casamento"
```

---

## 💰 Custo estimado (uso pessoal, baixo tráfego)

| Serviço | Estimativa mensal |
|---|---|
| S3 (10 GB + poucas requisições) | ~$0,25 |
| Lambda (1.000 invocações/mês) | ~$0,00 (free tier) |
| RDS `db.t3.micro` (24/7) | ~$15,00 |
| **Total** | **~$15,00/mês** |

> 💡 Para reduzir ainda mais: parar a instância RDS quando não estiver em uso,
> ou migrar metadados para **DynamoDB** (free tier generoso).

---

## 🚧 Fora de escopo (intencionalmente)

- Autenticação multiusuário / OAuth
- Versionamento de arquivos
- CDN / CloudFront na frente do S3
- Filas (SQS) ou cache (ElastiCache)
- Réplicas de leitura do RDS
- VPC dedicada com subnets privadas

Se algum dia precisar, essas peças podem ser adicionadas **sem reescrever a base**.

---

## 🧠 Conceitos demonstrados

| Conceito | Onde aparece no projeto |
|---|---|
| Separação de responsabilidades | S3 guarda arquivo, RDS guarda metadado |
| Computação serverless | Lambda como única camada de lógica |
| Persistência relacional | RDS com schema simples e indexado |
| Acoplamento mínimo | Três serviços, sem fila, sem cache, sem orquestração |
| Trade-off explícito | O que foi deixado fora **de propósito** |

---

## 🚀 Próximos passos (sugestão de exercício em sala)

Peça aos alunos para propor, em grupo, **como cada uma dessas peças seria
adicionada** e **por quê**:

- [ ] Autenticação (API Key → Cognito → IAM)
- [ ] API Gateway na frente da Lambda
- [ ] CloudFront na frente do S3
- [ ] SQS entre Lambda e processamento pesado
- [ ] VPC + subnets privadas para RDS e Lambda
- [ ] DynamoDB no lugar do RDS (e o trade-off disso)

Cada adição deve vir acompanhada de **custo, complexidade e motivo** — esse
é o exercício real da disciplina.

---

## 📄 Licença

MIT — material livre para uso, adaptação e ensino. É didático: replique,
modifique e discuta em aula.
