# Sistema de Benefícios Eventuais — Backend (Java + Spring Boot)

> **Status:** Em desenvolvimento — documentação inicial. Algumas partes poderão ser editadas posteriormente.

---

## Visão geral

Este repositório contém o backend para um sistema de **benefícios eventuais** (cesta básica, kit natalidade, gás de cozinha, auxílio funeral). O backend será consumido por um frontend e foi pensado para ser **simples, testável e escalável** usando Java + Spring Boot, seguindo princípios de **Clean Code**, **Clean Architecture** e padrões de projeto (Design Patterns).

### Objetivos principais

* CRUD de usuários/beneficiários.
* Histórico e relatório socioeconômico com data de última atualização e registro da assistente social/psicólogo responsável pela escuta.
* Regras de negócio: verificação se já se passaram **3 meses** desde a última cesta básica (com diferenciação entre cesta mensal e trimestral).
* Documentação via **Swagger** e boas práticas de código.

---

## Tecnologias

* Java 17+ (recomendado)
* Spring Boot (versão estável atual)
* Spring Data JPA
* Hibernate
* MySQL / PostgreSQL (ex.: profiles para switch)
* Spring Security + JWT (opcional, recomendado)
* Lombok (opcional para reduzir boilerplate)
* Swagger (OpenAPI) — documentação automática
* Flyway ou Liquibase — migrações de banco
* JUnit + Mockito — testes unitários
* Docker / Docker Compose (opcional)

---

## Arquitetura e organização do código

Seguir a ideia de Clean Architecture (camadas):

```
src/main/java
 └─ com.example.beneficios
    ├─ adapter
    │  ├─ in (controllers, dtos de entrada)
    │  └─ out (repositories, mappers para entidades)
    ├─ application (casos de uso / services)
    ├─ domain (entidades, value objects, exceptions)
    └─ config (security, swagger, database, beans)
```

* **domain** — regras de negócio e modelos puros.
* **application** — casos de uso (ex.: `CreateBeneficiary`, `UpdateReport`, `CheckEligibilityForMonthlyBasket`).
* **adapter.in** — Controllers REST, DTOs, validações.
* **adapter.out** — Repositórios JPA, implementações de persistência.

Favor manter classes pequenas, métodos com responsabilidades únicas e nomes descritivos.

---

## Modelos principais (Domain)

### Beneficiary (Usuário / Beneficiário)

* `id` (Long)
* `fullName` (String)
* `cpf` (String) — único
* `nis` (String) — número do NIS
* `phone` (String)
* `address` (String)
* `reports` (List<Report>)
* `createdAt` (Instant)
* `updatedAt` (Instant)

### Report (Relatório / Escuta)

* `id` (Long)
* `beneficiaryId` (Long)
* `type` (ENUM: CESTA_MENSAL, CESTA_TRIMESTRAL, KIT_NATALIDADE, GAS_COZINHA, AUXILIO_FUNERAL)
* `reason` (String) — relato do motivo
* `socialWorker` (String) — nome da assistente social/psicólogo
* `providedAt` (LocalDate) — data de quando o benefício foi entregue
* `lastUpdatedAt` (Instant) — data da última atualização do relatório

> Observação: separar `CESTA_MENSAL` e `CESTA_TRIMESTRAL` para regras de elegibilidade distintas.

---

## Regras de negócio (importantes)

1. **Elegibilidade de cesta básica**: ao tentar registrar uma nova cesta básica para um beneficiário, o sistema deve verificar quando foi a última entrega do tipo correspondente (mensal/trimestral).

   * Para `CESTA_MENSAL`: permitir nova entrega apenas se *>= 1 mês* desde `providedAt` anterior, mas o requisito solicitado é verificar **3 meses** — confirmar regra com o produto. (Atualmente configurado para **3 meses** como regra escrita.)
   * Para `CESTA_TRIMESTRAL`: permitir nova entrega apenas se *>= 3 meses* desde `providedAt` anterior.
2. Ao criar ou atualizar um `Report`, atualizar o campo `lastUpdatedAt` com `Instant.now()`.
3. `cpf` e `nis` devem ser validados quanto ao formato e unicidade.
4. Todas as operações que alteram estado devem ficar em transações atômicas.

---

## Endpoints (API REST) — Proposta inicial

> Base path: `/api/v1`

### Beneficiários

* `GET /beneficiaries` — listar (paginação + filtros por nome, cpf)
* `GET /beneficiaries/{id}` — buscar por id
* `POST /beneficiaries` — criar beneficiário
* `PUT /beneficiaries/{id}` — atualizar beneficiário
* `DELETE /beneficiaries/{id}` — remover (ou soft delete — recomendado)

### Relatórios / Benefícios

* `GET /beneficiaries/{id}/reports` — listar relatórios do beneficiário
* `POST /beneficiaries/{id}/reports` — criar relatório / registrar entrega de benefício
* `PUT /beneficiaries/{id}/reports/{reportId}` — atualizar relatório (atualiza `lastUpdatedAt` automaticamente)
* `DELETE /beneficiaries/{id}/reports/{reportId}` — remover relatório

### Verificações

* `GET /beneficiaries/{id}/eligibility?type=CESTA_MENSAL` — retorna se o beneficiário é elegível para o tipo de benefício agora (considera regras de tempo)

---

## Exemplo de payloads

**Criar beneficiário** (`POST /beneficiaries`)

```json
{
  "fullName": "Maria Silva",
  "cpf": "12345678901",
  "nis": "00012345678",
  "phone": "(11) 91234-5678",
  "address": "Rua A, 123"
}
```

**Criar relatório** (`POST /beneficiaries/{id}/reports`)

```json
{
  "type": "CESTA_TRIMESTRAL",
  "reason": "Família em vulnerabilidade após demissão do provedor",
  "socialWorker": "Joana Souza",
  "providedAt": "2025-10-01"
}
```

Resposta esperada: `201 Created` com o objeto `Report` criado e campo `lastUpdatedAt` preenchido.

---

## Banco de dados — Exemplo de schema (DDL simplificado)

```sql
CREATE TABLE beneficiary (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  full_name VARCHAR(200) NOT NULL,
  cpf VARCHAR(11) NOT NULL UNIQUE,
  nis VARCHAR(20),
  phone VARCHAR(20),
  address VARCHAR(300),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  deleted BOOLEAN DEFAULT FALSE
);

CREATE TABLE report (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  beneficiary_id BIGINT NOT NULL,
  type VARCHAR(50) NOT NULL,
  reason TEXT,
  social_worker VARCHAR(200),
  provided_at DATE,
  last_updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  FOREIGN KEY (beneficiary_id) REFERENCES beneficiary(id)
);
```

---

## Validações e DTOs

* Validar `cpf` (tamanho + máscara se houver) e `nis`.
* Usar DTOs para entrada/saída (`BeneficiaryRequest`, `BeneficiaryResponse`, `ReportRequest`, `ReportResponse`).
* Usar `@Valid` + Bean Validation (`@NotNull`, `@Size`, `@Pattern`).

---

## Exceções e tratamentos de erro

* Padrão: resposta JSON com `{timestamp, status, error, message, path}`.
* Erros específicos: `BeneficiaryNotFoundException`, `InvalidCpfException`, `NotEligibleForBenefitException`.

---

## Swagger / OpenAPI

* Ativar configuração para `/swagger-ui.html` ou `/swagger-ui/index.html` dependendo da versão.
* Documentar controllers com `@Operation`, `@ApiResponses` para gerar uma especificação consistente.

---

## Segurança

* Recomendado: Spring Security + JWT para autenticar usuários do sistema (assistentes sociais, administradores).
* Roles/Authorities: `ROLE_ADMIN`, `ROLE_SOCIAL_WORKER`.
* Proteja endpoints que alteram dados (POST/PUT/DELETE).

---

## Testes

* Cobertura mínima: serviços de domínio (casos de uso) e validadores.
* Testes de integração com banco em memória (H2) para endpoints.

---

## Como rodar (modo rápido)

Pré-requisitos: Java 17+, Maven ou Gradle, banco MySQL/Postgres (ou usar Docker)

1. Configurar `application.yml` (profiles `dev`, `prod`).
2. Rodar migrations (Flyway/Liquibase).
3. `./mvnw spring-boot:run` ou `./gradlew bootRun`.

> Alternativa: usar `docker-compose` com serviço da app e banco.

---

## Exemplo de `docker-compose` (simplificado)

```yaml
version: '3.8'
services:
  db:
    image: mysql:8
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: beneficios
    ports:
      - "3306:3306"
  app:
    image: beneficios-app:latest
    depends_on:
      - db
    ports:
      - "8080:8080"
```

---

## Padrões de projeto recomendados

* **Repository Pattern** via Spring Data JPA
* **Factory** para criação de objetos complexos (se necessário)
* **Strategy** para regras de elegibilidade por tipo de benefício
* **Builder** para construção de DTOs/entidades onde necessário
* **Adapter/Facade** para integrar com serviços externos (ex.: verificação de CPF/NIS)

---

## Checklist / TODOs (em desenvolvimento)

* [ ] Implementar autenticação JWT
* [ ] Definir regra final para `CESTA_MENSAL` vs `CESTA_TRIMESTRAL` (1 mês x 3 meses)
* [ ] Implementar migrations (Flyway)
* [ ] Testes de integração e cobertura mínima
* [ ] Documentar deployment (Kubernetes / Docker)
* [ ] Implementar soft delete e auditoria básica

---

## Contribuição

1. Fork o repositório
2. Crie uma branch: `feature/nome-da-feature`
3. Faça commits claros e pequenos
4. Abra um PR com descrição do que foi feito

---

## Contato

Para dúvidas e alinhamentos do produto, inclua comentários no PR ou contate a equipe responsável.

---

**Observação final:** esta documentação é uma base inicial. Como o sistema está em andamento, detalhes (regras exatas de negócio, campos do modelo, endpoints) poderão ser ajustados. Recomendo manter essa README sempre sincronizada com as tarefas no board (ex.: GitHub Projects, Jira).
