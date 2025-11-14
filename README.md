# MVP — Giftcards / Top-ups API (NestJS + Prisma + OpenAPI)
 Versão: 1.0  
Última atualização: 2025-11-14  

> contendo o pacote inicial para um MVP de venda legal de gift cards / top-ups (modelo seguro, estilo Codashop / Rei dos Coins).

---

## Estrutura do repositório (sugestão)

```
mvp-giftcards/
├─ backend/
│  ├─ prisma/
│  │  └─ schema.prisma
│  ├─ src/
│  │  ├─ app.module.ts
│  │  ├─ main.ts
│  │  ├─ auth/
│  │  ├─ products/
│  │  ├─ orders/
│  │  └─ payments/
│  ├─ Dockerfile
│  └─ docker-compose.yml
├─ frontend/
│  ├─ package.json
│  └─ pages/
│     ├─ index.tsx
│     └─ checkout.tsx
├─ openapi.yaml
└─ README.md
```

---

## 1) `openapi.yaml` (arquivo principal de documentação)

> Arquivo OpenAPI pronto para visualizar no Swagger UI / Stoplight. Inclui endpoints básicos: `/products`, `/checkout`, `/orders`, `/webhooks`, `/auth`.

```yaml
openapi: 3.0.3
info:
  title: Giftcards TopUp API - MVP
  version: 1.0.0
  description: API para venda de giftcards e top-ups (modelo legal)
servers:
  - url: https://api.seudominio.com
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  schemas:
    User:
      type: object
      properties:
        id:
          type: string
        email:
          type: string
        role:
          type: string
    Product:
      type: object
      properties:
        id:
          type: string
        sku:
          type: string
        name:
          type: string
        description:
          type: string
        price_cents:
          type: integer
        currency:
          type: string
        stock:
          type: integer
    Order:
      type: object
      properties:
        id:
          type: string
        userId:
          type: string
        total_cents:
          type: integer
        status:
          type: string
paths:
  /products:
    get:
      summary: Lista produtos (giftcards/vouchers)
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: array
                items:
                  $ref: '#/components/schemas/Product'
  /checkout:
    post:
      summary: Cria um pedido e inicia checkout
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                productId:
                  type: string
                quantity:
                  type: integer
                paymentMethod:
                  type: string
      responses:
        '201':
          description: Pedido criado
  /webhooks/payment:
    post:
      summary: Webhook de confirmação de pagamento (Stripe / Mercado Pago)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
      responses:
        '200':
          description: Recebido
  /auth/login:
    post:
      summary: Login (gera JWT)
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              properties:
                email:
                  type: string
                password:
                  type: string
      responses:
        '200':
          description: Token JWT
security:
  - BearerAuth: []
```

---

## 2) `prisma/schema.prisma`

> Modelo simples com Users, Products, Orders, OrderItems, Transactions e Vouchers.

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  password  String
  role      String   @default("user")
  orders    Order[]
  createdAt DateTime @default(now())
}

model Product {
  id          String  @id @default(uuid())
  sku         String  @unique
  name        String
  description String?
  priceCents  Int
  currency    String  @default("BRL")
  stock       Int     @default(0)
  createdAt   DateTime @default(now())
}

model Order {
  id         String      @id @default(uuid())
  user       User        @relation(fields: [userId], references: [id])
  userId     String
  totalCents Int
  status     String      @default("pending")
  items      OrderItem[]
  tx         Transaction?
  createdAt  DateTime    @default(now())
}

model OrderItem {
  id        String  @id @default(uuid())
  order     Order   @relation(fields: [orderId], references: [id])
  orderId   String
  product   Product @relation(fields: [productId], references: [id])
  productId String
  quantity  Int
  priceCents Int
}

model Transaction {
  id         String @id @default(uuid())
  order      Order  @relation(fields: [orderId], references: [id])
  orderId    String @unique
  provider   String
  providerTx String
  amountCents Int
  status     String
  createdAt  DateTime @default(now())
}

model Voucher {
  id        String @id @default(uuid())
  code      String @unique
  product   Product @relation(fields: [productId], references: [id])
  productId String
  claimed   Boolean @default(false)
  createdAt DateTime @default(now())
}
```

---

## 3) Backend — NestJS (esqueleto / arquivos principais)

**package.json (backend)** — dependências principais

```json
{
  "name": "mvp-giftcards-backend",
  "version": "1.0.0",
  "scripts": {
    "start:dev": "nest start --watch",
    "build": "nest build",
    "start": "node dist/main.js"
  },
  "dependencies": {
    "@nestjs/common": "^10.0.0",
    "@nestjs/core": "^10.0.0",
    "@nestjs/jwt": "^10.0.0",
    "@nestjs/passport": "^10.0.0",
    "class-transformer": "^0.5.1",
    "class-validator": "^0.14.0",
    "passport": "^0.6.0",
    "passport-jwt": "^4.0.0",
    "@prisma/client": "^5.0.0",
    "prisma": "^5.0.0",
    "stripe": "^12.0.0"
  }
}
```

### `src/main.ts` (bootstrap)

```ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import * as helmet from 'helmet';
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  app.use(helmet());
  app.enableCors({ origin: true });
  app.useGlobalPipes(new ValidationPipe({ whitelist: true }));
  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

### `src/app.module.ts` — modules importados (exemplo)

```ts
import { Module } from '@nestjs/common';
import { ProductsModule } from './products/products.module';
import { OrdersModule } from './orders/orders.module';
import { AuthModule } from './auth/auth.module';
import { PaymentsModule } from './payments/payments.module';

@Module({
  imports: [ProductsModule, OrdersModule, AuthModule, PaymentsModule],
})
export class AppModule {}
```

### `src/products/products.controller.ts` (exemplo rápido)

```ts
import { Controller, Get } from '@nestjs/common';
import { ProductsService } from './products.service';

@Controller('products')
export class ProductsController {
  constructor(private readonly productsService: ProductsService) {}

  @Get()
  async findAll() {
    return this.productsService.findAll();
  }
}
```

### `src/products/products.service.ts` (exemplo)

```ts
import { Injectable } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient();

@Injectable()
export class ProductsService {
  async findAll() {
    return prisma.product.findMany({ where: { stock: { gt: 0 } } });
  }
}
```

### `src/payments/payments.service.ts` — stub Stripe

```ts
import Stripe from 'stripe';
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY || '', { apiVersion: '2022-11-15' });

export class PaymentsService {
  async createCheckoutSession(lineItems: any[], metadata = {}) {
    const session = await stripe.checkout.sessions.create({
      payment_method_types: ['card'],
      mode: 'payment',
      line_items: lineItems,
      success_url: process.env.SUCCESS_URL,
      cancel_url: process.env.CANCEL_URL,
      metadata,
    });
    return session;
  }
}
```

---

## 4) `docker-compose.yml` (exemplo)

```yaml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
      POSTGRES_DB: giftcards
    ports:
      - '5432:5432'
    volumes:
      - db-data:/var/lib/postgresql/data

  backend:
    build: ./backend
    environment:
      DATABASE_URL: postgres://postgres:postgres@db:5432/giftcards
      STRIPE_SECRET_KEY: sk_test_xxx
      JWT_SECRET: changeme
    ports:
      - '3000:3000'
    depends_on:
      - db

volumes:
  db-data:
```

---

## 5) Front-end (Next.js) — arquivos essenciais

### `frontend/pages/index.tsx` — catálogo simples

```tsx
import useSWR from 'swr';

const fetcher = (url: string) => fetch(url).then(r => r.json());

export default function Home() {
  const { data: products } = useSWR('/api/products', fetcher);

  if (!products) return <div>Carregando...</div>;

  return (
    <div>
      <h1>Giftcards</h1>
      <ul>
        {products.map((p: any) => (
          <li key={p.id}>
            {p.name} — R$ {(p.priceCents/100).toFixed(2)}
            <button onClick={() => window.location.href = `/checkout?product=${p.id}`}>Comprar</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### `frontend/pages/checkout.tsx` — inicia checkout

```tsx
import { useRouter } from 'next/router';
import { useEffect, useState } from 'react';

export default function Checkout(){
  const router = useRouter();
  const { product } = router.query;
  const [loading, setLoading] = useState(false);

  async function startCheckout(){
    setLoading(true);
    const res = await fetch('/api/checkout', { method: 'POST', headers: {'Content-Type':'application/json'}, body: JSON.stringify({ productId: product, quantity: 1 }) });
    const data = await res.json();
    window.location.href = data.checkoutUrl; // redireciona para Stripe/Mercado Pago
  }

  useEffect(()=>{ if (product) startCheckout(); }, [product]);

  return <div>{loading ? 'Redirecionando...' : 'Iniciando...'}</div>
}
```

---

## 6) `README.md` (resumo rápido de setup)

```
# MVP Giftcards

1. Copie os arquivos para o repositório.
2. Copie `.env.example` e configure as variáveis (DATABASE_URL, STRIPE_SECRET_KEY, JWT_SECRET, SUCCESS_URL, CANCEL_URL).
3. `docker-compose up --build` (vai subir Postgres + backend).
4. `npx prisma migrate dev --name init` para criar tabelas.
5. Rodar frontend com `npm run dev` na pasta frontend (Next.js).
```

---

## 7) Checklist de segurança e operações (essencial) — DETALHADO

Abaixo um checklist acionável e detalhado — cada item tem **o que fazer**, **por que é importante**, e **exemplo/tip**.

### 7.1 Infra / Rede

* **Forçar TLS (HTTPS)**

  * O que fazer: usar certificados válidos (Let's Encrypt / Cloudflare) e redirecionar HTTP → HTTPS.
  * Por que: proteção contra interceptação de dados (incluindo tokens e cartões).
  * Tip: ative HSTS (HTTP Strict Transport Security) com período mínimo de 30 dias.

* **Firewall / WAF (Web Application Firewall)**

  * O que fazer: usar Cloudflare WAF ou AWS WAF para bloquear tráfego malicioso.
  * Por que: impede ataques comuns (SQLi, XSS, bots bruteforce).

* **Rate limiting**

  * O que fazer: limitar requisições por IP/endpoints críticos (ex.: 100 req/min por IP; endpoints sensíveis menor).
  * Por que: reduz ataques de força bruta e abuso de checkout.
  * Tip: usar soluções no nível do API Gateway (NGINX, Cloudflare, Kong).

## 7.2) Checklist Operacional

### Antes do deploy:
- [ ] SSL configurado  
- [ ] `.env` revisado  
- [ ] migrations aplicadas  
- [ ] webhooks testados  
- [ ] antifraude habilitado  

### Cada compra:
- [ ] pagamento aprovado  
- [ ] webhook validado  
- [ ] idempotência verificada  
- [ ] prova de entrega registrada  

### Diário:
- [ ] monitorar falhas  
- [ ] revisar transações suspeitas  
- [ ] verificar incidentes  

### Semanal:
- [ ] revisar chargebacks  
- [ ] analisar logs de segurança  

### Mensal:
- [ ] testar backup  
- [ ] testar rollback  
- [ ] revisar políticas  

### 7.3 Autenticação & Autorização

* **JWT com exp curto + refresh tokens**

  * O que: tokens de acesso com 15–60 min, refresh tokens longos revogáveis.
  * Por que: menor janela de comprometimento.

* **Hash de senha (bcrypt/argon2)**

  * O que: salvar apenas hash das senhas (bcrypt cost >= 12 ou argon2 padrão).
  * Por que: mitiga vazamento de senhas.

* **MFA (opcional)**

  * O que: oferecer 2FA por app (TOTP) para contas com valores altos.
  * Por que: reduz fraude em contas grandes.

### 7.4 Pagamentos & Webhooks

* **Verificação de assinatura dos webhooks**

  * O que: sempre validar assinatura (Stripe signature header / Mercado Pago’s signature) antes de processar.
  * Por que: evita aceitar eventos forjados.

* **Idempotência**

  * O que: usar `idempotency-key` em pagamentos e proteger contra duplicidade de criação de pedidos.
  * Por que: evitar cobranças duplicadas em retries.

* **Antifraude**

  * O que: integrar checks (MaxMind, Sift, manual rules de velocity, cartões BIN checks).
  * Por que: reduzir chargebacks e perdas.

### 7.5 Dados & Privacidade

* **Criptografia em trânsito e em repouso**

  * O que: TLS + criptografia de backups sensíveis.

* **Segurança de secrets**

  * O que: usar vault (HashiCorp Vault, AWS Secrets Manager, o equivalente) — não commitar `.env` com segredos.

* **Logs sensíveis**

  * O que: redigir logs para nunca armazenar PANs (números de cartão), CVV, ou senhas em claro.

### 7.6 Operações & Backup

* **Backups automáticos (DB)**

  * O que: snapshots diários + retentativa de 30 dias; testar restauração trimestralmente.

* **Métricas e monitoring**

  * O que: usar Prometheus + Grafana ou serviço gerenciado (Datadog) para métricas: erros 5xx, latência, taxa de checkout, chargebacks.

* **Alerting**

  * O que: configurar alertas críticos (ex.: aumento de 5xx > 3x base, spike de chargebacks) via PagerDuty/Slack.

### 7.7 Segurança de entrega (vouchers/giftcards)

* **Prova de entrega**

  * O que: gerar evidências (voucher code, timestamp, txId do provedor, IP do consumidor, email/telefone) e armazenar com o pedido.
  * Por que: facilita disputa em chargebacks.

* **Limites por pedido/conta**

  * O que: limite diário e por transação para novos usuários (ex.: R$200/dia nos primeiros 7 dias).
  * Por que: reduzir risco de fraude de cartão roubado.

### 7.8 Rotina de incident response

* **Playbook rápido**

  * O que: ter um runbook com passos (isolar serviço, trocar chaves, bloquear contas, comunicar time legal/financeiro).
  * Por que: ações rápidas reduzem dano.

---

## 8) Segurança contra chargebacks e fraudes — processo operacional

Passo a passo para operacionalizar: detector → verificador → mitigador.

1. **Detector (automático)**

   * Regra: transação com mismatch geográfico (IP != billing country), cartão BIN de alto risco, velocity (várias tentativas em 1h) → marca como "revisão".
2. **Verificador (semi-automático)**

   * Operador analisa: logs, histórico do usuário, e pede comprovação adicional (documento, 2FA, selfie) para continuar.
3. **Mitigador**

   * Se suspeita forte: cancelar entrega, reembolsar e abrir disputa com provedor (se necessário). Caso entregue, coletar prova e subir para disputa.

Dicas:

* Mantém um campo `disputeEvidence` ligado à transaction com upload de arquivos.
* Use pré-autorizações em cartões (quando suportado) e confirme antes da entrega para pedidos grandes.

---

## 9) Deploy, CI/CD e infra recomendada

### 9.1 Infra mínima recomendada (MVP)

* **Backend**: 1 app container (Node/NestJS) com auto-scale manual
* **DB**: managed Postgres (Railway, Supabase, Render DB)
* **Storage**: S3-compatible para assets e comprovantes
* **Proxy / CDN**: Cloudflare ou Vercel Edge

### 9.2 Pipeline CI/CD (exemplo GitHub Actions)

* **steps**:

  1. `lint` + `test`
  2. `prisma migrate deploy`
  3. build image + push to registry
  4. deploy to provider (Railway/Render) via API

### 9.3 Blue/Green ou Canary (opcional)

* Ao liberar endpoints críticos (webhooks), use deploy canary ou versão paralela para evitar quebra total.

---

## 10) Observabilidade e testes

* **Testes unitários**: controllers, services e validações (Jest).
* **Testes e2e**: fluxo completo de checkout simulando webhooks.
* **Chaos testing (básico)**: simular falha de webhook e checar idempotência.
* **Logs estruturados**: JSON logs com requestId para correlacionar eventos.

---

## 11) Compliance legal e fiscal — checklist prático

* **Registro da empresa**: escolher regime (MEI/EIRELI/Simples/Outro) conforme faturamento.
* **Emissão de nota fiscal**: automatizar emissão ao cliente (se aplicável); integrar com sistema de emissão (e.g., NFe no Brasil).
* **Política de privacidade e termos**: escrever termos claros cobrindo reembolsos, responsabilidade e limitações.
* **Verificação de partners**: manter contrato escrito com fornecedores de gift cards (pra não revender sem autorização).
* **Impostos**: retenções e impostos de serviços — consulte contador antes do lançamento.

---

## 12) Operação comercial e growth (curto prazo)

* **Modelo de precificação**: margem sobre custo do voucher; descontos por volume; frete digital grátis.
* **Promoções**: cupons por email, descontos temporários por jogos populares.
* **Suporte**: chat + tickets; modelos de resposta prontos para problemas comuns (atraso, reembolso, voucher inválido).
* **Parcerias**: streamers e micro-influencers no nicho para tráfego inicial.

---

## 13) Arquivos auxiliares úteis (exemplo de `.env.example`)

```
DATABASE_URL=postgres://postgres:postgres@db:5432/giftcards
PORT=3000
JWT_SECRET=troque_para_valor_forte
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
SUCCESS_URL=https://seudominio.com/checkout/success
CANCEL_URL=https://seudominio.com/checkout/cancel
S3_URL=
S3_KEY=
S3_SECRET=
```

---

## 14) Próximos passos que eu posso fazer AGORA

* Gerar arquivos separados (cada arquivo pronto para colar): controllers completos, DTOs, serviços, seeds do Prisma, frontend pages com estilo.
* Gerar integração completa de webhooks com verificação de assinatura para Stripe e Mercado Pago.
* Gerar GitHub Actions YAML para CI/CD.


---

## 15) Conclusão
Este documento define o padrão mínimo de segurança necessário para operar um marketplace de moedas/vouchers com segurança, estabilidade e conformidade.


