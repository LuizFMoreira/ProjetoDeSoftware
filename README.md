# VetCare+

> Sistema fictício para gestão de uma clínica veterinária: agendamento de consultas, prontuário eletrônico do pet, controle de vacinação, pagamento online e relatório financeiro.

Trabalho 2 da disciplina de **Projeto de Software** — projeto individual, focado em documentação e arquitetura. Não há código implementado: a entrega são os modelos UML, o modelo de dados e o documento que amarra tudo.

---

## Status do Projeto

![Versão](https://img.shields.io/badge/versão-1.0.0-blue)
![Status](https://img.shields.io/badge/status-entregue-brightgreen)
![Licença](https://img.shields.io/badge/licença-MIT-green)
![PlantUML](https://img.shields.io/badge/UML-PlantUML-orange)
![Java](https://img.shields.io/badge/Java-17-orange)
![Spring Boot](https://img.shields.io/badge/Spring%20Boot-3.3-brightgreen)
![React](https://img.shields.io/badge/React-19-blue)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-blue)
![RabbitMQ](https://img.shields.io/badge/RabbitMQ-3.13-red)

---

## 📄 Entrega Principal

> **[📥 VetCarePlus-Documentacao.pdf](Documentacao/VetCarePlus-Documentacao.pdf)** — documento completo (27 páginas) seguindo o template oficial do professor, com capa, sumário, histórico de revisões, 16 casos de uso, 10 contratos de operação, 21 diagramas UML e modelo de dados.

---

## Índice

- [Sobre o Projeto](#sobre-o-projeto)
- [Funcionalidades](#funcionalidades)
- [Atores](#atores)
- [Tecnologias](#tecnologias)
- [Arquitetura](#arquitetura)
- [Diagramas](#diagramas)
- [Regras de Negócio](#regras-de-negócio)
- [Estrutura do Repositório](#estrutura-do-repositório)
- [Como Renderizar os Diagramas](#como-renderizar-os-diagramas)
- [Instalação e Execução (hipotética)](#instalação-e-execução-hipotética)
- [Variáveis de Ambiente](#variáveis-de-ambiente)
- [Testes (plano)](#testes-plano)
- [Deploy (plano)](#deploy-plano)
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

O contexto é acadêmico, mas tentei pensar no problema como se fosse mesmo entrar em produção numa clínica de bairro — ou, no caso da arquitetura escolhida, numa rede de clínicas tipo Petz/Cobasi.

---

## Funcionalidades

- **Autenticação JWT** com quatro perfis: tutor, recepcionista, veterinário e administrador.
- **Cadastro de pets** vinculados ao tutor (espécie, raça, peso, data de nascimento).
- **Agendamento de consultas** com checagem de conflito na agenda do veterinário (RN-01) e validação de especialidade (RN-10).
- **Cancelamento e reagendamento** com cálculo automático de taxa de cancelamento se menos de 2h de antecedência (RN-02).
- **Prontuário eletrônico imutável** (RN-13) — anamnese, diagnóstico, vacinas e prescrições.
- **Controle de vacinação** com cálculo automático de próxima dose seguindo o protocolo da espécie e idade do pet (RN-04, RN-05).
- **Pagamento online** via Pix ou cartão, com integração de gateway externo e janela de 15 min para confirmação do Pix (RN-08).
- **Lembretes automáticos por e-mail** — 24h antes da consulta, 30 e 7 dias antes do vencimento de vacina, com retry de até 3 tentativas (RN-12).
- **Relatório financeiro mensal** em PDF, agregando pagamentos confirmados por método e por veterinário (RN-11).

---

## Atores

| Ator | Descrição |
|---|---|
| **Tutor** | Dono do pet. Cadastra animais, marca consulta, paga e recebe lembretes. Só enxerga os próprios pets. |
| **Recepcionista** | Trabalha no balcão. Faz quase tudo que o tutor faz, mas em nome dele. Organiza a agenda do dia. |
| **Veterinário** | Precisa ter CRMV ativo. Atende consulta, preenche prontuário, prescreve medicamento e aplica vacina. |
| **Administrador** | Cadastra vets, define tabela de preços, emite relatórios financeiros, estorna pagamento. |
| **Gateway de Pagamento** *(externo)* | Processa Pix e cartão, devolve aprovação por webhook. |
| **Serviço de E-mail** *(externo)* | Entrega lembretes e recibos via SMTP. |

---

## Tecnologias

A stack foi escolhida pensando em ferramentas que eu já tinha familiaridade ou queria treinar.

**Front-end**
- React 19 + Vite
- TypeScript
- Tailwind CSS
- Zustand (estado global, mais leve que Redux)
- Axios

**Back-end (microsserviços em Java)**
- Java 17
- Spring Boot 3.3 (Web, Data JPA, Security)
- Hibernate como ORM
- JWT stateless para autenticação
- Maven

**Mensageria e dados**
- PostgreSQL 16 (um banco por microsserviço)
- RabbitMQ para eventos de domínio

**Infraestrutura**
- Docker + Docker Compose
- API Gateway + ALB (na borda)
- CloudFront (CDN para o front)
- S3 (anexos de exames)
- GitHub Actions (CI hipotético)

---

## Arquitetura

Modelei o VetCare+ como uma aplicação de **microsserviços**. Cada serviço tem seu próprio banco, expõe REST e se comunica de forma assíncrona via RabbitMQ pra eventos de domínio (consulta agendada, vacina aplicada, pagamento confirmado).

São seis serviços:

| Serviço | Responsabilidade |
|---|---|
| **ms-usuarios** | Autenticação JWT, CRUD de tutor/recepcionista/vet/admin. |
| **ms-agendamento** | Consultas, agenda dos vets, regras de conflito (RN-01). |
| **ms-prontuario** | Anamnese, diagnóstico, vacinas, prescrições. Upload de exames pra S3. |
| **ms-pagamento** | Integração com gateway Pix/cartão, webhooks, estorno. |
| **ms-notificacao** | Worker que consome a fila e dispara e-mail via SMTP. Scheduler de lembretes. |
| **ms-relatorios** | Leitura cruzada de pagamento e agendamento pros relatórios financeiros. |

Na frente fica um **API Gateway** que faz autenticação na borda e roteamento pros serviços. Um **ALB** distribui carga horizontalmente quando algum serviço precisa escalar.

Por dentro de cada serviço mantive a estrutura em camadas (Controller → Service → Repository → DTO) que o Spring já incentiva. JWT é stateless — o filtro valida o token no API Gateway antes de chegar nos serviços.

**Por que microsserviços numa clínica?** Honestamente, pra uma clínica pequena isso é over-engineering — um monolito modular resolveria. Escolhi essa arquitetura aqui pra mostrar o desenho de um sistema que **escalaria** pra uma rede de clínicas (Petz, Cobasi e afins têm centenas de unidades). Os limites estão definidos por contexto de negócio: agendamento, prontuário clínico e pagamento são domínios diferentes, com times diferentes e ciclos de mudança diferentes — daí faz sentido separar.

> Detalhamento completo da arquitetura, padrões adotados (Repository, Service Layer, DTO, Event-Driven, Strategy, Factory) e justificativas estão na seção 3.1 do [PDF da documentação](Documentacao/VetCarePlus-Documentacao.pdf).

---

## Diagramas

**Total: 21 diagramas PlantUML** organizados por tipo. Fontes em [`Codigo/`](Codigo/), imagens renderizadas em [`Modelagem/`](Modelagem/).

| Tipo | Quantidade | Pasta |
|---|---|---|
| Casos de Uso | 1 (com 16 UCs) | [`Codigo/CasosDeUso/`](Codigo/CasosDeUso/) |
| Classes | 1 (com 6 packages) | [`Codigo/Classes/`](Codigo/Classes/) |
| Sequência do Sistema (DSS / black-box) | 3 | [`Codigo/SequenciaSistema/`](Codigo/SequenciaSistema/) |
| Sequência detalhada (white-box) | 7 | [`Codigo/Sequencia/`](Codigo/Sequencia/) |
| Comunicação | 3 | [`Codigo/Comunicacao/`](Codigo/Comunicacao/) |
| Estados | 4 | [`Codigo/Estado/`](Codigo/Estado/) |
| Componentes e Implantação | 1 | [`Codigo/Componentes/`](Codigo/Componentes/) |
| Entidade-Relacionamento (DER) | 1 | [`Codigo/ER/`](Codigo/ER/) |

### Casos de Uso e Classes

- [Casos de Uso](Codigo/CasosDeUso/diagrama-de-caso-de-uso.puml) — 16 UCs cobrindo os 4 perfis + atores externos.
- [Classes](Codigo/Classes/diagrama-de-classes.puml) — entidades de domínio, enums, interface `Autenticavel`, classe associativa `AplicacaoVacina`.

### Sequência do Sistema (DSS — visão black-box, seção 2.3 do template)

- [DSS UC-04 Agendar Consulta](Codigo/SequenciaSistema/DSS-UC-04-agendar-consulta.puml)
- [DSS UC-10 Aplicar Vacina](Codigo/SequenciaSistema/DSS-UC-10-aplicar-vacina.puml)
- [DSS UC-11 Pagar Consulta](Codigo/SequenciaSistema/DSS-UC-11-pagar-consulta.puml)

### Sequência detalhada (white-box, seção 3.4 do template)

- [UC-04 Agendar Consulta](Codigo/Sequencia/UC-04-agendar-consulta.puml)
- [UC-05 Cancelar Consulta](Codigo/Sequencia/UC-05-cancelar-consulta.puml)
- [UC-07 Atender Consulta](Codigo/Sequencia/UC-07-atender-consulta.puml)
- [UC-10 Aplicar Vacina](Codigo/Sequencia/UC-10-aplicar-vacina.puml)
- [UC-11 Pagar Consulta (Pix)](Codigo/Sequencia/UC-11-pagar-consulta.puml)
- [UC-15 Gerar Relatório Financeiro](Codigo/Sequencia/UC-15-gerar-relatorio.puml)
- [UC-16 Enviar Lembrete (job)](Codigo/Sequencia/UC-16-enviar-lembrete.puml)

### Estados (ciclo de vida das entidades principais)

- [Consulta](Codigo/Estado/diagrama-de-estado-consulta.puml) — agendada → confirmada → em atendimento → realizada → paga.
- [Vacinação](Codigo/Estado/diagrama-de-estado-vacinacao.puml) — pendente → aplicada → vencendo → vencida → reforço.
- [Pagamento](Codigo/Estado/diagrama-de-estado-pagamento.puml) — pendente → processando → aprovado/recusado/expirado.
- [Pet](Codigo/Estado/diagrama-de-estado-pet.puml) — ativo → inativo (transferido ou óbito).

### Comunicação, Componentes e Dados

- [Componentes e Implantação](Codigo/Componentes/diagrama-de-componentes-e-implantacao.puml) — clientes, gateway, cluster de microsserviços, fila e serviços externos.
- [Comunicação UC-04 — Agendar Consulta](Codigo/Comunicacao/comunicacao-UC04-agendar-consulta.puml)
- [Comunicação UC-10 — Aplicar Vacina](Codigo/Comunicacao/comunicacao-UC10-aplicar-vacina.puml)
- [Comunicação UC-11 — Pagar Consulta](Codigo/Comunicacao/comunicacao-UC11-pagar-consulta.puml)
- [Modelo Entidade-Relacionamento](Codigo/ER/diagrama-er-vetcareplus.puml) — 14 tabelas agrupadas por microsserviço.

---

## Regras de Negócio

Catálogo completo das 15 regras de negócio está no [Apêndice A do PDF](Documentacao/VetCarePlus-Documentacao.pdf). Resumo das mais relevantes:

- **RN-01** — Vet não pode ter duas consultas no mesmo horário (janela de 30 min).
- **RN-02** — Cancelamento com < 2h gera taxa de 50%.
- **RN-04** — Vacina antirrábica tem reforço anual, com lembrete 30 e 7 dias antes.
- **RN-06** — Prescrição de medicamento controlado exige CRMV ativo.
- **RN-08** — Pix tem 15 minutos pra confirmar, depois expira.
- **RN-11** — Relatório financeiro mensal consolida pagamentos CONFIRMADO do período.
- **RN-13** — Prontuário é imutável; correções entram como nova anotação.

---

## Estrutura do Repositório

```
ProjetoDeSoftware/
├── README.md                           # este arquivo
├── template_README.md                  # template do prof (referência)
├── .gitignore
│
├── Documentacao/
│   └── VetCarePlus-Documentacao.pdf    # 📄 documento final (27 páginas)
│
├── Codigo/                             # fontes PlantUML (.puml)
│   ├── CasosDeUso/
│   ├── Classes/
│   ├── Comunicacao/
│   ├── Componentes/
│   ├── ER/
│   ├── Estado/
│   ├── Sequencia/                      # white-box (detalhadas)
│   └── SequenciaSistema/               # black-box (DSS)
│
├── Modelagem/                          # imagens renderizadas (.png)
│   ├── CasosDeUso/
│   ├── Classes/
│   ├── Comunicacao/
│   ├── Componentes/
│   ├── ER/
│   ├── Estado/
│   ├── Sequencia/
│   └── SequenciaSistema/
│
├── Assets/                             # logos e imagens auxiliares
└── enunciado/                          # PDF do template oficial do professor
```

---

## Como Renderizar os Diagramas

Os arquivos `.puml` em `Codigo/` podem ser renderizados de três formas:

**1. Extensão PlantUML do VS Code** (mais prático)
- Instale a extensão `jebbs.plantuml`
- Abra qualquer `.puml` e pressione `Alt+D` — o preview aparece ao lado

**2. Servidor online**
- Cole o conteúdo do `.puml` em <https://www.plantuml.com/plantuml/uml/>

**3. Linha de comando** (precisa do `plantuml.jar`)
```bash
java -jar plantuml.jar -tpng Codigo/CasosDeUso/diagrama-de-caso-de-uso.puml
```

As versões já renderizadas (PNG) ficam em [`Modelagem/`](Modelagem/) — mesma hierarquia de pastas.

---

## Instalação e Execução (hipotética)

> O projeto é só documentação — não há código implementado. O passo a passo abaixo é o que seria usado **se** a aplicação fosse construída, seguindo a arquitetura proposta.

### Pré-requisitos

- Java JDK 17+
- Node.js 18+ (com npm)
- Docker e Docker Compose
- PostgreSQL 16 (ou via Docker)
- RabbitMQ 3.13 (ou via Docker)

### Subindo a infraestrutura

```bash
# Banco de cada microsserviço (porta diferente por serviço)
docker run --name vetcare-usuarios-db   -e POSTGRES_PASSWORD=postgres -p 5432:5432 -d postgres:16
docker run --name vetcare-agendamento-db -e POSTGRES_PASSWORD=postgres -p 5433:5432 -d postgres:16
# ... idem para os outros serviços

# RabbitMQ
docker run --name vetcare-rabbit -p 5672:5672 -p 15672:15672 -d rabbitmq:3.13-management
```

### Rodando um microsserviço

```bash
cd ms-agendamento
./mvnw spring-boot:run
```

### Front-end

```bash
cd frontend
npm install
npm run dev
```

### Tudo via docker-compose

```bash
docker-compose up --build
```

---

## Variáveis de Ambiente

**Microsserviço Spring Boot** (cada `ms-*/.env` ou `application.yml`):

| Variável | Exemplo |
|---|---|
| `SERVER_PORT` | `8080` (varia por serviço) |
| `SPRING_DATASOURCE_URL` | `jdbc:postgresql://localhost:5432/agendamento_db` |
| `SPRING_DATASOURCE_USERNAME` | `postgres` |
| `SPRING_DATASOURCE_PASSWORD` | `postgres` |
| `SPRING_RABBITMQ_HOST` | `localhost` |
| `JWT_SECRET` | chave longa em base64 |
| `JWT_EXPIRATION_MS` | `86400000` (24h) |
| `MAIL_USERNAME` | `notificacoes@vetcareplus.com` |
| `MAIL_PASSWORD` | senha do gateway SMTP |

**Front-end** (`frontend/.env.local`):

| Variável | Exemplo |
|---|---|
| `VITE_API_URL` | `https://api.vetcareplus.com` |
| `VITE_PAGAMENTO_PUBLIC_KEY` | chave pública do gateway de pagamento |

---

## Testes (plano)

Quando o código existir:

- **Back-end:** JUnit 5 + Mockito (unitários), Testcontainers (integração com Postgres e RabbitMQ reais).
- **Front-end:** Vitest + Testing Library.
- **E2E:** Cypress cobrindo o fluxo principal (login → agendar → atender → pagar).
- **Contrato entre microsserviços:** Spring Cloud Contract ou Pact, validando os eventos publicados no RabbitMQ.

---

## Deploy (plano)

- **Front** na Vercel ou CloudFront (estáticos do Vite).
- **Microsserviços** em AWS ECS (Docker) ou Railway.
- **Bancos** em Amazon RDS (PostgreSQL gerenciado) — um por serviço.
- **Fila** em Amazon MQ (RabbitMQ gerenciado) ou auto-hospedado com HA.
- **Anexos** em S3 com URLs pré-assinadas.

CI/CD via **GitHub Actions**: build do JAR e da imagem Docker em cada PR, deploy automático na main.

---

## Referências

- [Documentação do Spring Boot](https://docs.spring.io/spring-boot/docs/current/reference/html/)
- [React](https://react.dev/) e [Vite](https://vitejs.dev/)
- [PlantUML](https://plantuml.com/)
- [RabbitMQ Patterns](https://www.rabbitmq.com/getstarted.html)
- [Microservices Patterns — Chris Richardson](https://microservices.io/patterns/index.html)
- [Conventional Commits](https://www.conventionalcommits.org/pt-br/v1.0.0/)
- Slides e materiais da disciplina de Projeto de Software (Prof. Dr. João Paulo Aramuni).
- Template oficial: [joaopauloaramuni/laboratorio-de-desenvolvimento-de-software](https://github.com/joaopauloaramuni/laboratorio-de-desenvolvimento-de-software/blob/main/TEMPLATES/template_README.md).

---

## Autor

| Nome | GitHub | E-mail |
|---|---|---|
| Luiz F. Moreira | [@LuizFMoreira](https://github.com/LuizFMoreira) | groowfi.ads@gmail.com |

Trabalho **individual**.

---

## Licença

Distribuído sob a licença MIT. Veja o arquivo `LICENSE` para mais detalhes.
