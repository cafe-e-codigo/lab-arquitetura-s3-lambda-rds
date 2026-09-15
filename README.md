# 📁 Demo Didática — S3 + Lambda + RDS

Demo mínima usada em **aula introdutória de Arquitetura de Soluções**.
Mostra, na prática, como S3, Lambda e RDS se combinam para resolver um
problema simples de armazenamento e catalogação de arquivos.

> ⚠️ **Material didático.** Não é produção: sem autenticação, sem VPC,
> sem multi-tenant. A topologia foi simplificada de propósito.

---

## 🏗️ Arquitetura

```
┌──────────┐   HTTP    ┌────────────┐   SQL   ┌──────────┐
│ Cliente  │ ────────▶ │  Lambda    │ ──────▶ │   RDS    │
│          │           │ (CRUD API) │         │ (dados)  │
└──────────┘           │            │   Put/Get ┌──────────┐
                       │            │ ────────▶ │    S3    │
                       └────────────┘           │(arquivos)│
                                                └──────────┘
```

**Ideia central:** separar **dado estruturado** (RDS) de **arquivo binário** (S3).
Essa divisão é o conceito da aula — o resto é consequência.

---

## 🧠 Conceitos demonstrados

- Separação de responsabilidades: metadado ≠ arquivo
- Serverless: Lambda como única camada de lógica
- Persistência relacional: RDS com schema simples e indexado
- Trade-off explícito: o que ficou **fora** e por quê

---

## 🚀 Como rodar

```bash
git clone https://github.com/seu-usuario/lab-arquitetura-s3-lambda-rds.git
cd lab-arquitetura-s3-lambda-rds
npm install
npm run deploy
```

Endpoint de teste:

```bash
curl -X POST $FUNCTION_URL/files \
  -F "file=@foto.jpg" \
  -F "client=Cliente X" \
  -F "tags=foto,2024"
```

Schema do banco em [`schema.sql`](./schema.sql).

---

## 🧩 Exercício em sala

Adicione **uma peça** por vez e justifique **custo, complexidade e motivo**:

- API Gateway na frente da Lambda
- CloudFront na frente do S3
- VPC + subnets privadas
- SQS para processamento assíncrono
- DynamoDB no lugar do RDS

---

## 📄 Licença

MIT — material livre para uso, adaptação e ensino.
