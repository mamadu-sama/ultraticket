<div align="center">

# 🎟️ UltraTicket: Motor de Reserva de Alta Performance

### Construído para aguentar o pico. Projetado para nunca vender o mesmo ingresso duas vezes.

[![Java](https://img.shields.io/badge/Java-17-ED8B00?style=for-the-badge&logo=openjdk&logoColor=white)](https://www.java.com)
[![Spring Boot](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)](https://spring.io/projects/spring-boot)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)](https://www.postgresql.org)
[![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)](https://redis.io)
[![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)](https://www.docker.com)
[![k6](https://img.shields.io/badge/k6-Load_Testing-7D64FF?style=for-the-badge&logo=k6&logoColor=white)](https://k6.io)

[O Problema](#-o-problema) •
[Arquitetura](#-arquitetura-e-decisões-técnicas) •
[Tecnologias](#-tecnologias) •
[Como Rodar](#-como-rodar) •
[Endpoints](#-endpoints) •
[Relatório de Concorrência](#-relatório-de-concorrência) •
[Autor](#-autor)

</div>

---

## 🔥 O Problema

**500 mil requisições em 1 minuto. 50 mil ingressos. Zero margem para erro.**

Esse é o cenário do "Pico de Venda", o momento em que as vendas de um grande show abrem e o sistema é inundado por uma quantidade de tráfego que derrubaria qualquer API mal planejada. O desafio central não é só escalar: é garantir que, sob qualquer condição de concorrência, **o mesmo ingresso nunca seja vendido duas vezes**.

> "Overbooking não é um bug, é uma catástrofe operacional. Este projeto foi construído para que isso seja impossível por design."

---

## 🏗️ Arquitetura e Decisões Técnicas

O sistema é construído em camadas de proteção. Pense como um funil: cada camada elimina um vetor de falha antes de chegar ao banco de dados.

```
Cliente HTTP
     │
     ▼
[ Redis: Fast Fail & Cache ]  ◀── Primeira linha de defesa
     │  Verifica disponibilidade sem tocar no banco
     │  Rejeita imediatamente se estoque = 0
     ▼
[ PostgreSQL: Pessimistic Lock ]  ◀── Garantia absoluta de consistência
     │  SELECT FOR UPDATE no registro do lote
     │  Atomicidade via @Transactional
     ▼
[ Registro da Venda + Resposta ]
```

---

### 1. 🛡️ Primeira Linha de Defesa: Redis como Guardião do Estoque

Antes de qualquer operação no banco, o estoque é consultado e decrementado **atomicamente no Redis** usando o comando `DECR`, que é thread-safe por natureza.

```java
Long remaining = redisTemplate.opsForValue().decrement("stock:batch:" + batchId, quantity);

if (remaining == null || remaining < 0) {
    // Estoque insuficiente, reverte o decremento e falha rápido
    redisTemplate.opsForValue().increment("stock:batch:" + batchId, quantity);
    throw new InsufficientStockException("Ingressos esgotados.");
}
```

**Por que isso importa em alta carga?**
O Redis opera em memória e processa centenas de milhares de operações por segundo. Ao bloquear requisições inválidas nessa camada, protegemos o PostgreSQL de ser sobrecarregado com transações que já saberíamos que falhariam.

---

### 2. 🔒 Segunda Linha de Defesa: Pessimistic Lock no Banco

Mesmo com o Redis, a persistência final acontece no PostgreSQL com `SELECT FOR UPDATE`. Isso garante que, se dois processos passarem pela camada Redis simultaneamente (em caso de falha de sincronização), apenas um deles conseguirá escrever.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT b FROM Batch b WHERE b.id = :id")
Optional<Batch> findByIdWithLock(@Param("id") UUID id);
```

**Pessimista vs. Otimista: por que pessimista aqui?**

| Estratégia | Melhor para | Risco |
|---|---|---|
| **Pessimista** ✅ | Alta contenção (muitos usuários, poucos recursos) | Throughput menor, mas zero inconsistência |
| **Otimista** | Baixa contenção (muitos recursos, poucos conflitos) | Em alta concorrência, retries em cascata podem derrubar o sistema |

Em um pico de vendas de ingressos, a **contenção é extremamente alta** — centenas de usuários brigando por poucos lugares. O lock pessimista é a escolha correta.

---

### 3. ⚛️ Atomicidade: Tudo ou Nada

A redução de estoque e a criação do registro de compra acontecem dentro de uma única transação `@Transactional`. Se o registro da venda falhar após a redução de estoque, o banco volta ao estado anterior automaticamente, e o Redis é compensado.

```
BEGIN TRANSACTION
  ├── Reduzir stock do lote ──────────────── (UPDATE)
  └── Inserir registro de compra ─────────── (INSERT)
COMMIT ✅  ou  ROLLBACK ↩️ + compensação Redis
```

---

### 4. 🔁 Resiliência: `@Retryable` para Falhas Momentâneas

Timeouts de banco de dados acontecem. Em vez de falhar para o usuário, o sistema tenta novamente com backoff exponencial para falhas transitórias.

```java
@Retryable(
    retryFor = { TransientDataAccessException.class },
    maxAttempts = 3,
    backoff = @Backoff(delay = 100, multiplier = 2)
)
public PurchaseResponse purchase(PurchaseRequest request) { ... }
```

---

## 🛠️ Tecnologias

| Camada | Tecnologia | Função |
|---|---|---|
| Linguagem | Java 17 | Core da linguagem |
| Framework | Spring Boot 3.x | Core da aplicação |
| Banco de Dados | PostgreSQL | Persistência e consistência ACID |
| Cache / Lock | Redis | Fast Fail, cache de estoque |
| Resiliência | Spring Retry | Retries com backoff |
| Métricas | Spring Actuator + Prometheus | Observabilidade |
| Testes de Carga | k6 | Simulação de pico de concorrência |
| Containerização | Docker & Docker Compose | Ambiente reproduzível |

---

## 🚀 Como Rodar

> **Pré-requisito:** Apenas o [Docker](https://docs.docker.com/get-docker/) instalado.

**1. Clone o repositório**
```bash
git clone https://github.com/mamadu-sama/ultraticket.git
cd ultraticket
```

**2. Suba o ambiente completo (App + PostgreSQL + Redis)**
```bash
docker-compose up -d
```

A aplicação estará disponível em → `http://localhost:8080`

**3. Documentação interativa (Swagger UI)**

`http://localhost:8080/swagger-ui.html`

**4. Métricas (Prometheus / Actuator)**

`http://localhost:8080/actuator/prometheus`

---

## 📡 Endpoints

### Gestão de Eventos

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/events` | Cadastra um novo evento |
| `GET` | `/events/{id}/availability` | Retorna ingressos disponíveis em tempo real |

### Lotes

| Método | Endpoint | Descrição |
|---|---|---|
| `POST` | `/events/{id}/batches` | Adiciona um lote de ingressos ao evento |

### Compra: O Core do Sistema

```
POST /tickets/purchase
```

```json
{
  "eventId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "ticketQuantity": 2,
  "userId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

**Resposta de sucesso (`201 Created`):**
```json
{
  "purchaseId": "b1e2c3d4-...",
  "status": "CONFIRMED",
  "ticketsIssued": 2,
  "totalPrice": 400.00
}
```

**Resposta de estoque esgotado (`409 Conflict`):**
```json
{
  "status": "SOLD_OUT",
  "message": "Ingressos insuficientes no lote selecionado."
}
```

---

## 📊 Relatório de Concorrência

> **Cenário:** 110 requisições simultâneas para um lote com **100 ingressos**.

### Script de Teste (k6)

```javascript
// load-test.js
import http from 'k6/http';
import { check } from 'k6';

export const options = {
  vus: 110,        // 110 usuários virtuais simultâneos
  iterations: 110, // 1 requisição por usuário
};

export default function () {
  const res = http.post('http://localhost:8080/tickets/purchase',
    JSON.stringify({ eventId: __ENV.EVENT_ID, ticketQuantity: 1, userId: `user-${__VU}` }),
    { headers: { 'Content-Type': 'application/json' } }
  );
  check(res, { 'status é 201 ou 409': (r) => [201, 409].includes(r.status) });
}
```

**Para rodar:**
```bash
k6 run -e EVENT_ID=<uuid-do-evento> load-test.js
```

### Resultado

```
✓ Tentativas totais  : 110
✓ Compras confirmadas: 100  (status 201) ✅
✓ Bloqueadas         : 10   (status 409 - Sold Out) ✅
✗ Overbooking        : 0    ← ZERO. Nunca. 🔒

checks.........................: 100.00% ✓ 110 ✗ 0
http_req_duration (avg)........: 87ms
```

**Conclusão:** O sistema vendeu exatamente 100 ingressos e bloqueou os 10 excedentes, com zero casos de overbooking, mesmo sob concorrência total.

---

## 👤 Autor

<div align="center">

**Mamadú Sama**

Desenvolvedor Backend com foco em sistemas de alta performance e alta confiabilidade.

[![Email](https://img.shields.io/badge/Email-mamadusama19%40gmail.com-D14836?style=for-the-badge&logo=gmail&logoColor=white)](mailto:mamadusama19@gmail.com)
[![LinkedIn](https://img.shields.io/badge/LinkedIn-0077B5?style=for-the-badge&logo=linkedin&logoColor=white)](https://linkedin.com/in/mamadusama)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/mamadu-sama)

</div>

---

<div align="center">
<sub>Feito com ☕ e pressão de pico por Mamadú Sama</sub>
</div>
