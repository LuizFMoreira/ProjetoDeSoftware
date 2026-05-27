# VetCare+

> Sistema fictício para gestão de uma clínica veterinária: agendamento de consultas, prontuário eletrônico do pet e controle de vacinação.

Trabalho 2 da disciplina de Projeto de Software. Projeto individual, focado em documentação e diagramação (não tem código implementado).

---

## Status do Projeto

![Versão](https://img.shields.io/badge/versão-1.0.0-blue)
![Status](https://img.shields.io/badge/status-documentação-yellow)
![Licença](https://img.shields.io/badge/licença-MIT-green)
![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen)
![React](https://img.shields.io/badge/React-19-blue)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue)

---

## Índice

- [Sobre o Projeto](#sobre-o-projeto)
- [Funcionalidades](#funcionalidades)
- [Tecnologias](#tecnologias)
- [Arquitetura](#arquitetura)
- [Diagramas](#diagramas)
- [Instalação e Execução](#instalação-e-execução)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Estrutura de Pastas](#estrutura-de-pastas)
- [Testes](#testes)
- [Deploy](#deploy)
- [Referências](#referências)
- [Autor](#autor)
- [Licença](#licença)

---

## Sobre o Projeto

O VetCare+ nasceu de uma situação bem real: minha tia tem uma clínica veterinária pequena e até hoje controla agenda em caderno. Quando o cliente liga pra remarcar, é uma confusão. Pensei em projetar um sistema simples que resolvesse os três problemas que ela mais reclama:

1. Perder o histórico do animal quando a ficha de papel some.
2. Esquecer de avisar o tutor que a vacina do pet está vencendo.
3. Não saber quanto entrou no caixa no fim do mês sem ficar somando recibo.

A ideia aqui não é entregar um software pronto, e sim mostrar como eu estruturaria essa aplicação: quem usa, quais regras, como as classes se relacionam e como os componentes se conversam.

O contexto é acadêmico, mas tentei pensar no problema como se fosse mesmo entrar em produção numa clínica de bairro.

---

## Funcionalidades

- Login com perfis diferentes para tutor, recepcionista, veterinário e administrador.
- Cadastro de pets vinculados ao tutor (espécie, raça, peso, data de nascimento).
- Agendamento de consultas com checagem de conflito na agenda do veterinário.
- Prontuário eletrônico — anamnese, diagnóstico, vacinas aplicadas e prescrições.
- Lembretes automáticos por e-mail (consulta marcada, vacina pra vencer).
- Pagamento da consulta via Pix ou cartão.
- Relatório financeiro mensal para o administrador.

---

## Tecnologias

A stack é parecida com a que vimos em sala. Escolhi porque já tinha familiaridade com Spring e queria treinar React no front.

**Front-end**
- React 19 + Vite
- TypeScript
- Tailwind CSS para estilização
- Zustand para estado global (mais leve que Redux)
- Axios para chamadas HTTP

**Back-end**
- Java 17
- Spring Boot 3.3 (Web, Data JPA, Security)
- Hibernate como ORM
- JWT para autenticação
- Maven

**Banco**
- PostgreSQL 16

**Outros**
- Docker + Docker Compose pra subir tudo junto
- GitHub Actions pra CI (build e testes em cada PR)

---

## Arquitetura

Modelei o VetCare+ como uma aplicação de **microsserviços**. Cada serviço tem seu próprio banco, expõe REST e se comunica de forma assíncrona via RabbitMQ pra eventos de domínio (consulta agendada, vacina aplicada, pagamento confirmado).

São seis serviços:

- **ms-usuarios** — autenticação JWT, CRUD de tutor/recepcionista/vet/admin.
- **ms-agendamento** — consultas, agenda dos vets, regras de conflito.
- **ms-prontuario** — anamnese, diagnóstico, vacinas, prescrições. Upload de exames pra S3.
- **ms-pagamento** — integração com gateway Pix/cartão, webhooks, estorno.
- **ms-notificacao** — worker que consome a fila e dispara e-mail via SMTP. Tem também um scheduler pros lembretes agendados (vacina vencendo, consulta de amanhã).
- **ms-relatorios** — leitura cruzada de pagamento e agendamento pros relatórios financeiros.

Na frente fica um API Gateway que faz autenticação na borda e roteamento pros serviços. Um ALB distribui carga horizontalmente quando algum serviço precisa escalar.

Por dentro de cada serviço eu mantive a estrutura em camadas (Controller → Service → Repository → DTO) que o Spring já incentiva. JWT é stateless — o filtro valida o token no API Gateway antes de chegar nos serviços.

**Por que microsserviços numa clínica?** Honestamente, pra uma clínica pequena isso é over-engineering — um monolito modular resolveria. Escolhi essa arquitetura aqui pra mostrar o desenho de um sistema que **escalaria** pra uma rede de clínicas (Petz, Cobasi e afins têm centenas de unidades). Os limites estão definidos por contexto de negócio: agendamento, prontuário clínico e pagamento são domínios diferentes, com times diferentes e ciclos de mudança diferentes — daí faz sentido separar.

### Diagramas

Os fontes em PlantUML ficam em [Codigo/](Codigo/), organizados por tipo. As imagens renderizadas (PNG) ficam em [Modelagem/](Modelagem/). A documentação completa em PDF está em [Documentacao/](Documentacao/).

**Casos de Uso e Classes**
- [Casos de Uso](Codigo/CasosDeUso/diagrama-de-caso-de-uso.puml) — 16 UCs cobrindo os 4 perfis + atores externos.
- [Classes](Codigo/Classes/diagrama-de-classes.puml) — entidades de domínio, enums, interface `Autenticavel`.

**Sequência** (1 por caso de uso principal)
- [UC-04 Agendar Consulta](Codigo/Sequencia/UC-04-agendar-consulta.puml)
- [UC-05 Cancelar Consulta](Codigo/Sequencia/UC-05-cancelar-consulta.puml)
- [UC-07 Atender Consulta](Codigo/Sequencia/UC-07-atender-consulta.puml)
- [UC-10 Aplicar Vacina](Codigo/Sequencia/UC-10-aplicar-vacina.puml)
- [UC-11 Pagar Consulta (Pix)](Codigo/Sequencia/UC-11-pagar-consulta.puml)
- [UC-15 Gerar Relatório Financeiro](Codigo/Sequencia/UC-15-gerar-relatorio.puml)
- [UC-16 Enviar Lembrete (job)](Codigo/Sequencia/UC-16-enviar-lembrete.puml)

**Estados** (ciclo de vida das entidades principais)
- [Consulta](Codigo/Estado/diagrama-de-estado-consulta.puml)
- [Vacinação](Codigo/Estado/diagrama-de-estado-vacinacao.puml)
- [Pagamento](Codigo/Estado/diagrama-de-estado-pagamento.puml)
- [Pet](Codigo/Estado/diagrama-de-estado-pet.puml)

**Comunicação, Componentes e Dados**
- [Componentes e Implantação](Codigo/Componentes/diagrama-de-componentes-e-implantacao.puml)
- [Comunicação — UC-04](Codigo/Comunicacao/comunicacao-UC04-agendar-consulta.puml)
- [Comunicação — UC-10](Codigo/Comunicacao/comunicacao-UC10-aplicar-vacina.puml)
- [Modelo Entidade-Relacionamento](Codigo/ER/diagrama-er-vetcareplus.puml)

Pra abrir, dá pra usar a extensão PlantUML do VS Code (`Alt+D` renderiza ao lado) ou colar no <https://www.plantuml.com/plantuml/uml/>.

---

## Instalação e Execução

Como o projeto é só documentação, não há código pra rodar. Mas deixei o passo a passo que seria usado caso a aplicação fosse implementada, seguindo o que combinamos em aula.

### Pré-requisitos

- Java JDK 17+
- Node.js 18+ (com npm)
- Docker e Docker Compose
- PostgreSQL 16 (ou subir via Docker)

### Rodando localmente

Clone o repositório:

```bash
git clone https://github.com/LuizFMoreira/vetcare-plus.git
cd vetcare-plus
```

**Banco com Docker:**

```bash
docker run --name vetcare-db \
  -e POSTGRES_USER=postgres \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=vetcare_db \
  -p 5432:5432 -d postgres:16
```

**Back-end:**

```bash
cd backend
./mvnw spring-boot:run
```

Sobe em <http://localhost:8080>.

**Front-end (em outro terminal):**

```bash
cd frontend
npm install
npm run dev
```

Sobe em <http://localhost:5173>.

### Tudo via docker-compose

Se preferir subir tudo de uma vez:

```bash
docker-compose up --build
```

---

## Variáveis de Ambiente

**Back-end** (`backend/.env` ou `application.properties`):

| Variável | Exemplo |
|---|---|
| `SERVER_PORT` | `8080` |
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/vetcare_db` |
| `SPRING_DATASOURCE_USERNAME` | `postgres` |
| `SPRING_DATASOURCE_PASSWORD` | `postgres` |
| `JWT_SECRET` | uma chave longa em base64 |
| `JWT_EXPIRATION_MS` | `86400000` (1 dia) |
| `MAIL_USERNAME` | `notificacoes@vetcareplus.com` |
| `MAIL_PASSWORD` | senha do serviço de e-mail |

**Front-end** (`frontend/.env.local`):

| Variável | Exemplo |
|---|---|
| `VITE_API_URL` | `http://localhost:8080/api` |
| `VITE_PAGAMENTO_PUBLIC_KEY` | chave pública do gateway |

---

## Estrutura de Pastas

```
.
├── README.md                       # este arquivo
├── template_README.md              # template original do prof (referência)
├── docker-compose.yml              # orquestração dos serviços
│
├── docs/
│   ├── README.md                   # índice da documentação
│   └── diagrams/                   # diagramas PlantUML
│       ├── 01-casos-de-uso.puml
│       ├── 02-classes.puml
│       ├── 03-sequencia-agendamento-consulta.puml
│       └── 04-componentes-arquitetura.puml
│
├── frontend/                       # SPA React + Vite
│   ├── src/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── services/
│   │   ├── hooks/
│   │   └── styles/
│   └── package.json
│
└── backend/                        # API Spring Boot
    ├── src/main/java/com/vetcare/app/
    │   ├── controller/
    │   ├── service/
    │   ├── repository/
    │   ├── model/                  # entidades JPA
    │   ├── dto/
    │   ├── config/
    │   ├── exception/
    │   └── security/
    ├── src/main/resources/
    │   ├── application.yml
    │   └── db/migration/           # Flyway
    └── pom.xml
```

A pasta `frontend/` e `backend/` estão listadas aqui porque é como o projeto ficaria se fosse implementado — neste trabalho elas não existem ainda.

---

## Testes

Plano de testes para quando o código existir:

- **Back-end:** JUnit 5 + Mockito pros unitários, Testcontainers pros testes de integração com Postgres real.
- **Front-end:** Vitest + Testing Library nos componentes.
- **E2E:** Cypress cobrindo o fluxo principal (login → agendar consulta → pagar).

Comandos:

```bash
# backend
cd backend && ./mvnw test

# frontend
cd frontend && npm test

# e2e
cd frontend && npm run cypress
```

---

## Deploy

A ideia seria deployar:

- **Front** na Vercel (build do Vite gera os arquivos estáticos).
- **Back** no Railway ou num droplet da DigitalOcean rodando o JAR.
- **Banco** no Supabase ou num Postgres gerenciado.

Build de produção:

```bash
# front
cd frontend && npm run build   # gera /dist

# back
cd backend && ./mvnw clean package   # gera o .jar em /target
java -jar target/vetcare-0.0.1-SNAPSHOT.jar
```

---

## Referências

- [Documentação do Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [React](https://react.dev/)
- [Vite](https://vitejs.dev/)
- [PlantUML](https://plantuml.com/)
- [Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)
- Slides e materiais da disciplina de Projeto de Software (Prof. Dr. João Paulo Aramuni).
- Template de README utilizado como base: [joaopauloaramuni/laboratorio-de-desenvolvimento-de-software](https://github.com/joaopauloaramuni/laboratorio-de-desenvolvimento-de-software/blob/main/TEMPLATES/template_README.md).

---

## Autor

| Nome | GitHub | E-mail |
|---|---|---|
| Luiz F. Moreira | [@LuizFMoreira](https://github.com/LuizFMoreira) | groowfi.ads@gmail.com |

Trabalho individual.

---

## Licença

Distribuído sob a licença MIT. Veja o arquivo `LICENSE` para mais detalhes.
