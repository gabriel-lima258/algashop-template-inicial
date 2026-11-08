# algashop-template-inicial

Esqueleto para começar um microsserviço novo do AlgaShop.

Não é um serviço — é o ponto de partida dos outros, com as decisões repetitivas já tomadas e nenhum domínio dentro.

---

## Por que existe

Todo microsserviço novo começa com as mesmas duas horas de trabalho que ninguém quer repetir: escolher versões, configurar Flyway, decidir como os perfis do Spring se combinam, escrever de novo o handler de erro que devolve `ProblemDetail`, separar as suítes de teste no Gradle.

Nada disso é interessante na segunda vez — e ser feito ligeiramente diferente a cada vez é pior ainda, porque a inconsistência entre serviços custa mais que qualquer uma das escolhas isoladas.

Este repositório congela essas decisões. Clonar daqui significa que o serviço novo já nasce parecido com os outros três.

---

## O que já vem decidido

### Plataforma

| | |
|---|---|
| **Java** | 25 |
| **Spring Boot** | 4.0.1 |
| **Spring Cloud** | 2025.0.1 |
| **Gradle** | 9.2.1 (wrapper) |
| **Banco** | PostgreSQL + Flyway |

### Web e integração

`spring-boot-starter-webmvc` para servir, `spring-boot-starter-restclient` para chamar, `spring-boot-starter-validation` para validar a borda.

### Persistência

JPA com **`ddl-auto: none`** e Flyway como fonte da verdade do schema. Não é preferência de estilo: `ddl-auto` deduz o schema das anotações, e o que ele deduz não é versionado, não é revisável e não é reproduzível. Com Flyway, o schema é código — com histórico, revisão e o mesmo resultado em qualquer ambiente.

### Perfis compostos

```yaml
spring:
  profiles:
    active: development
    group:
      development: [base, development-env]
      docker:      [base, development-env, docker-env]
      production:  [base, production-env]
```

A ideia é separar **o que é igual em qualquer ambiente** (`base`: nome da aplicação, `ddl-auto`, locations do Flyway) do **que muda por ambiente** (`*-env`: URL do banco, credenciais, se a carga de dados liga). Trocar o serviço de ambiente passa a ser trocar um arquivo, não caçar propriedades espalhadas.

Repare que `docker` inclui `development-env` **e** `docker-env`, nessa ordem: herda tudo do desenvolvimento e sobrescreve só o que difere dentro da rede Docker.

### Tratamento de erro pronto

`ApiExceptionHandler` já mapeia o vocabulário HTTP inteiro para **`ProblemDetail`** (RFC 7807):

| Situação | Status | `type` |
|---|---|---|
| Campos inválidos | `400` | `/errors/invalid-fields` — com o mapa `fields` no corpo |
| Recurso não existe | `404` | `/errors/not-found` |
| Regra de negócio violada | `422` | `/errors/unprocessable-entity` |
| Integração respondeu errado | `502` | `/errors/bad-gateway` |
| Integração não respondeu | `504` | `/errors/gateway-timeout` |
| Qualquer outra coisa | `500` | `/errors/internal` |

As duas últimas linhas são o que essa lista tem de menos óbvio. **Falha de um serviço do qual você depende não é erro seu** — devolver `500` quando o problema foi o gateway apaga a informação de quem realmente quebrou, e é a diferença entre um alerta acionável e uma madrugada perdida.

As exceções de domínio (`DomainException`, `DomainEntityNotFoundException`) já vêm criadas, e não importam nada de `org.springframework.http` — quem traduz negócio para HTTP é o handler, na borda.

### Três suítes de teste

```bash
./gradlew test              # unitários — exclui classes *IT
./gradlew integrationTest   # só as *IT
./gradlew contractTest      # Spring Cloud Contract
./gradlew check             # as três
```

A separação existe para **falha barata aparecer antes da cara**. Um teste unitário quebrado não deveria custar os 40 segundos de subir um container para ser descoberto.

`integrationTest` não é um source set próprio: é uma task que reusa `src/test` e filtra por sufixo de nome. Menos configuração, mesmo efeito.

Já vêm no classpath: Testcontainers (PostgreSQL), AssertJ, RestAssured (fixado em 6.0.0), e o Mockito configurado como **Java agent** — exigência de JDKs recentes, que quebra sem aviso claro quando falta.

### Empacotamento

`Dockerfile` sobre `eclipse-temurin:25-jre` e a task `dockerBuild`, que faz build multi-arquitetura (arm64 + amd64) e push.

> A tag da imagem base precisa acompanhar o toolchain do `build.gradle`. Um jar compilado para Java 25 não roda numa JRE 21 — o container sobe e morre com `UnsupportedClassVersionError`.

---

## Como usar

```bash
git clone https://github.com/gabriel-lima258/algashop-template-inicial.git meu-servico
cd meu-servico
rm -rf .git && git init
```

Depois, renomeie o que carrega o nome `template`:

| Onde | O quê |
|---|---|
| `settings.gradle` | `rootProject.name` |
| `build.gradle` | `group`, `description`, `bootJar.archiveFileName` |
| `build.gradle` | a tag da imagem na task `dockerBuild` |
| `src/main/java/.../template/` | o pacote — `com.algaworks.algashop.<servico>` |
| `Dockerfile` | `ENV JAR_NAME` |
| `application-base.yml` | `spring.application.name` |
| `application-development-env.yml` | nome do banco na URL do datasource e do Flyway |

E crie o banco antes de subir — o template espera que ele exista, porque `ddl-auto` está desligado e o Flyway só cuida do schema, não da criação.

A primeira migration vai em `src/main/resources/db/migration/V001__*.sql`. O diretório ainda não existe: o Flyway está configurado apontando para ele, mas o template não traz schema nenhum, porque schema é a única coisa que não dá para adivinhar.

---

## Estrutura

```
src/main/java/com/algaworks/algashop/template/
├── TemplateApplication.java
├── domain/model/
│   ├── DomainException.java
│   └── DomainEntityNotFoundException.java
└── presentation/
    ├── ApiExceptionHandler.java
    ├── BadGatewayException.java
    ├── GatewayTimeoutException.java
    ├── UnprocessableEntityException.java
    └── PageModel.java
```

`PageModel` merece nota: é o DTO de página com `number`, `size`, `totalPages`, `totalElements` e `content`, com um factory `of(Page<T>)`. Ele existe para o `Page` do Spring Data **não** virar contrato de API — a serialização padrão dele carrega campos internos e muda entre versões, e o dia em que muda, quebra todo cliente.

---

## Os serviços feitos a partir daqui

| Serviço | O que ele acrescenta |
|---|---|
| [`algashop-ordering`](https://github.com/gabriel-lima258/algashop-ordering) | DDD tático e arquitetura hexagonal completa |
| [`algashop-billing`](https://github.com/gabriel-lima258/algashop-billing) | DDD pragmático e integração com gateway de pagamento |
| [`product-catalog`](https://github.com/gabriel-lima258/algashop-product-catalog) | MongoDB no lugar do PostgreSQL |
| [`algashop-billing-scheduler`](https://github.com/gabriel-lima258/algashop-billing-scheduler) | job efêmero, sem servidor HTTP |

Os quatro divergem do template em algum ponto — comparar cada um com este esqueleto é uma forma rápida de ver **o que naquele serviço foi decisão**, e o que veio de fábrica.

---

## Documentação

Caderno de estudos do projeto: [`algashop-docs`](https://github.com/gabriel-lima258/algashop-docs).

- [Ambiente local](https://github.com/gabriel-lima258/algashop-docs/blob/main/04-infraestrutura/01-ambiente-local.md) — portas, bancos e submódulos
- [Flyway](https://github.com/gabriel-lima258/algashop-docs/blob/main/02-persistencia/01-flyway.md) — por que `ddl-auto` não serve para produção
- [Tratamento de erros](https://github.com/gabriel-lima258/algashop-docs/blob/main/03-testes-integracao/02-tratamento-erros-api.md) — `ProblemDetail` e a hierarquia de exceções
- [Contract tests](https://github.com/gabriel-lima258/algashop-docs/blob/main/03-testes-integracao/01-stubs-contract-tests.md) — as três suítes e por que são separadas
- [Docker](https://github.com/gabriel-lima258/algashop-docs/blob/main/04-infraestrutura/02-docker.md) — build multi-arquitetura
