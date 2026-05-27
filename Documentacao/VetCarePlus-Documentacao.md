# Documentação de Projeto — VetCare+

**Sistema de Gestão para Clínica Veterinária**

Versão 1.0 — 27 de maio de 2026

Trabalho desenvolvido por **Luiz F. Moreira** como parte da disciplina de Projeto de Software.

---

## Histórico de Revisões

| Versão | Data       | Autor | Mudança                                                                 |
|--------|------------|-------|-------------------------------------------------------------------------|
| 0.1    | 08/05/2026 | Luiz  | Esboço inicial dos casos de uso e diagrama de classes.                  |
| 0.2    | 12/05/2026 | Luiz  | Atores externos definidos; enums adicionados às classes.                |
| 0.3    | 15/05/2026 | Luiz  | Primeiro diagrama de sequência (Agendar Consulta).                      |
| 0.4    | 19/05/2026 | Luiz  | Regras de Negócio (RN-01 a RN-15) e contratos de operação.              |
| 0.5    | 22/05/2026 | Luiz  | Diagramas de estado; sequências adicionais; refatoração para microsserviços. |
| 1.0    | 27/05/2026 | Luiz  | Revisão final; descrição expandida dos UCs; entrega.                    |

---

## Sumário

1. [Introdução](#1-introdução)
2. [Modelos de Usuário e Requisitos](#2-modelos-de-usuário-e-requisitos)
3. [Contratos de Operação](#3-contratos-de-operação)
4. [Modelos de Projeto](#4-modelos-de-projeto)
5. [Modelos de Dados](#5-modelos-de-dados)
6. [Escopo do MVP e Versões Futuras](#6-escopo-do-mvp-e-versões-futuras)

---

## 1. Introdução

O VetCare+ é um sistema de gestão para clínicas veterinárias. Cuida de agendamento, prontuário do pet, controle de vacinação com lembrete automático, pagamento por Pix ou cartão e relatório financeiro pro administrador.

Quem inspirou o projeto foi minha tia. Ela toca uma clínica de bairro há mais de dez anos e até hoje controla agenda em caderno e ficha de animal em papel. Quando o cliente liga pra remarcar, vira confusão. Quando a vacina antirrábica vence, ninguém avisa e o pet some por meses. No fim do mês ela soma recibo a mão pra saber quanto entrou. Vi isso de perto e quis modelar um sistema que resolvesse esse caos sem ser pesado demais pra uma clínica desse porte.

Aqui não tem código — só modelagem. O que entrego é a arquitetura, os casos de uso, as classes, as sequências, os estados e o modelo de dados.

Optei por desenhar em **microsserviços**. Pra uma clínica única seria exagero (um monolito Spring Boot resolveria), mas quis projetar pra cenário de rede — algo do tipo Petz ou Cobasi, em que cada contexto (agendamento, prontuário, pagamento) tem ciclo de mudança e equipe própria. Os limites entre serviços seguiram fronteiras de domínio, não de tabela.

---

## 2. Modelos de Usuário e Requisitos

### 2.1 Descrição dos Atores

O **Tutor** é o dono do pet. Cria conta com CPF, cadastra um ou mais animais e usa o app pra marcar consulta, ver o histórico (prontuário e vacina), pagar e receber aviso quando o reforço tá perto. Só enxerga os próprios pets.

A **Recepcionista** trabalha no balcão. Faz quase tudo que o tutor faz, só que em nome dele — comum no dia a dia da clínica, quando o cliente liga pedindo pra marcar ou chega no balcão sem conta. Também registra pagamento presencial e organiza a agenda do dia. Não entra no prontuário em detalhe; isso é território do vet.

O **Veterinário** precisa ter CRMV ativo. Atende a consulta e preenche o prontuário com anamnese, diagnóstico e observações. Também aplica vacina conforme protocolo da espécie, prescreve medicamento e pede exame. Só edita prontuário das consultas que está atendendo no momento — isso vem da RN-13 (histórico imutável).

O **Administrador** é o dono ou gerente. Cadastra e desliga vet, define tabela de preço, emite relatório financeiro e mexe nas permissões. É o único que reabre consulta finalizada ou estorna pagamento. Em clínica pequena costuma ser a mesma pessoa do balcão — e foi pensando nesse caso (que é o da minha tia) que separei o papel de "Administrador" do de "Recepcionista" mesmo sabendo que muitas vezes vai ser a mesma pessoa logada.

**Atores externos:**

- **Gateway de Pagamento** — serviço externo (Pix e cartão) que processa transações e devolve aprovação por webhook.
- **Serviço de E-mail (SMTP)** — gateway externo (ex.: SendGrid) que entrega lembretes e recibos.

### 2.2 Regras de Negócio

| ID    | Regra |
|-------|-------|
| RN-01 | Um veterinário não pode ter duas consultas no mesmo horário. Janela mínima entre consultas: 30 minutos. |
| RN-02 | Cancelamento com menos de 2 horas de antecedência gera taxa de 50% do valor da consulta. |
| RN-03 | Pet deve estar vinculado a exatamente 1 tutor responsável, com CPF cadastrado. |
| RN-04 | Vacina antirrábica tem reforço anual obrigatório. Sistema dispara lembrete 30 dias e 7 dias antes do vencimento. |
| RN-05 | Vacinas V8/V10 seguem protocolo: 1ª dose após 45 dias de vida, reforço a cada 21 dias até 4 meses, depois anual. |
| RN-06 | Prescrição de medicamento controlado exige CRMV ativo do veterinário no momento da prescrição. |
| RN-07 | Consulta só passa para REALIZADA depois do prontuário preenchido (anamnese e diagnóstico no mínimo). |
| RN-08 | Pagamento via Pix tem janela de 15 minutos para confirmação. Após isso, a consulta volta para AGUARDANDO_PAGAMENTO. |
| RN-09 | Tutor menor de 18 anos precisa de responsável legal cadastrado (CPF e grau de parentesco). |
| RN-10 | Veterinário só pode atender espécies da sua especialidade declarada, salvo override do administrador. |
| RN-11 | Relatório financeiro mensal consolida pagamentos com status CONFIRMADO entre o 1º e o último dia do mês. |
| RN-12 | E-mail de lembrete tenta envio até 3 vezes em caso de falha. Depois disso, é descartado e admin é notificado. |
| RN-13 | Histórico do prontuário é imutável. Correções entram como nova anotação com referência à anterior. |
| RN-14 | Senha mínima de 8 caracteres com hash BCrypt. Para perfis internos (recepcionista, vet, admin) expira em 90 dias. |
| RN-15 | Pet inativado (óbito ou transferência) não pode ser agendado, mas o histórico permanece consultável. |

### 2.3 Modelo de Casos de Uso

São 16 UCs no total. Os marcados como `<<include>>` (UC-01 e UC-16) entram em todos os fluxos que dependem deles — não dá pra agendar sem autenticar nem agendar sem disparar notificação.

**UC-01 — Autenticar.** Verifica e-mail e senha (BCrypt), aplica a RN-14 e emite JWT válido por 24h. Ator: usuário genérico.

**UC-02 — Gerenciar Perfil.** Qualquer usuário autenticado atualiza seus dados e troca senha.

**UC-03 — Cadastrar Pet.** Tutor cadastra pet vinculado ao próprio CPF (RN-03). Valida espécie, peso e data de nascimento.

**UC-04 — Agendar Consulta.** Tutor (ou recepcionista em nome dele) reserva horário com um vet. Inclui UC-01 e UC-16. Pré: pet ativo (RN-15), vet com agenda aberta, especialidade compatível (RN-10), sem conflito de horário (RN-01).

**UC-05 — Cancelar Consulta.** Tutor, recepcionista ou admin cancela uma consulta agendada. Se for menos de 2h antes, aplica taxa de 50% (RN-02). Se já estiver paga, dispara estorno via ms-pagamento.

**UC-06 — Reagendar Consulta.** Variação do cancelamento. Mantém o ID original e move pra nova data — assim preserva o histórico vinculado.

**UC-07 — Atender Consulta.** Vet inicia o atendimento e preenche o prontuário. Inclui UC-08; pode estender pra UC-09 (prescrição) e UC-10 (vacina). No fim, marca a consulta como REALIZADA (RN-07).

**UC-08 — Registrar Prontuário.** Anamnese, diagnóstico e observações. Imutável depois de gravado (RN-13).

**UC-09 — Prescrever Medicamento.** Vet adiciona prescrição com posologia e duração. Se for controlado, valida CRMV (RN-06).

**UC-10 — Aplicar Vacina.** Vet registra aplicação e lote. Sistema calcula a próxima dose pelo protocolo (RN-04, RN-05) e agenda lembrete 30 e 7 dias antes.

**UC-11 — Pagar Consulta.** Tutor ou recepcionista paga por Pix ou cartão. Integra com o Gateway externo. Pix expira em 15 min sem confirmação (RN-08).

**UC-12 — Visualizar Agenda do Dia.** Vet e recepcionista veem a lista do dia, com status (agendada, confirmada, em atendimento, realizada).

**UC-13 — Ver Prontuário do Pet.** Tutor vê o histórico do próprio pet. Vet vê de qualquer pet da clínica.

**UC-14 — Gerenciar Veterinários.** Admin cadastra, ativa ou desliga vet. Define especialidade e carga horária.

**UC-15 — Gerar Relatório Financeiro.** Admin gera relatório mensal de pagamentos confirmados (RN-11). Saída em PDF.

**UC-16 — Notificar.** Incluído por UC-04, UC-05 e UC-10. Encapsula envio de e-mail com retry e descarte após 3 falhas (RN-12).

### 2.4 Diagrama de Casos de Uso

> Fonte: [`Codigo/CasosDeUso/diagrama-de-caso-de-uso.puml`](../Codigo/CasosDeUso/diagrama-de-caso-de-uso.puml)
> Imagem: [`Modelagem/CasosDeUso/casos-de-uso.png`](../Modelagem/CasosDeUso/casos-de-uso.png)

O diagrama mostra a hierarquia de atores (Usuário abstrato → Tutor, Recepcionista, Veterinário, Administrador), os 16 UCs e as relações `<<include>>` e `<<extend>>`.

---

## 3. Contratos de Operação

### CT-01 — agendarConsulta

- **Operação:** `agendarConsulta(tutorId: Long, petId: Long, vetId: Long, dataHora: DateTime, motivo: String): ConsultaDTO`
- **UC:** UC-04
- **Pré-condições:** tutor autenticado; pet pertence ao tutor e está ATIVO; vet está ATIVO; dataHora futura; especialidade compatível com espécie (RN-10).
- **Pós-condições:** consulta persistida com status AGENDADA; evento `ConsultaAgendadaEvent` publicado no RabbitMQ; lembrete agendado pra 24h antes.

### CT-02 — cancelarConsulta

- **Operação:** `cancelarConsulta(consultaId: Long, motivo: String, solicitanteId: Long): ResultadoCancelamento`
- **UC:** UC-05
- **Pré-condições:** consulta existe e está em AGENDADA ou CONFIRMADA; solicitante é o tutor dono, recepcionista ou admin; dataHora futura.
- **Pós-condições:** status atualizado para CANCELADA; se antecedência < 2h, taxa de 50% aplicada (RN-02); se consulta paga, estorno disparado; tutor notificado.

### CT-03 — registrarProntuario

- **Operação:** `registrarProntuario(consultaId: Long, vetId: Long, anamnese: String, diagnostico: String, obs: String): ProntuarioDTO`
- **UC:** UC-08
- **Pré-condições:** consulta em EM_ATENDIMENTO; vetId é o atendente; anamnese e diagnóstico não vazios.
- **Pós-condições:** prontuário criado e imutável (RN-13); consulta passa pra REALIZADA; cobrança gerada.

### CT-04 — prescreverMedicamento

- **Operação:** `prescreverMedicamento(prontuarioId: Long, vetId: Long, medicamentoId: Long, posologia: String, duracaoDias: int): PrescricaoDTO`
- **UC:** UC-09
- **Pré-condições:** prontuário existe; vet com CRMV ativo (RN-06); medicamento cadastrado; duração > 0.
- **Pós-condições:** prescrição vinculada ao prontuário; se controlado, registro em log de auditoria.

### CT-05 — aplicarVacina

- **Operação:** `aplicarVacina(prontuarioId: Long, vetId: Long, vacinaTipo: TipoVacina, lote: String): AplicacaoVacinaDTO`
- **UC:** UC-10
- **Pré-condições:** prontuário existe; vacina compatível com espécie e idade do pet; lote informado.
- **Pós-condições:** aplicação registrada; `proximaDose` calculada por protocolo (RN-04, RN-05); lembrete de reforço agendado.

### CT-06 — pagarConsulta

- **Operação:** `pagarConsulta(consultaId: Long, metodo: MetodoPagamento, valor: BigDecimal): ResultadoPagamento`
- **UC:** UC-11
- **Pré-condições:** consulta REALIZADA; pagamento ainda não confirmado; valor confere com o valor da consulta.
- **Pós-condições:** cobrança gerada no gateway; se Pix expirar em 15 min (RN-08), status volta pra EXPIRADO; recibo enviado por e-mail em caso de aprovação.

### CT-07 — cadastrarPet

- **Operação:** `cadastrarPet(tutorId: Long, dados: PetData): PetDTO`
- **UC:** UC-03
- **Pré-condições:** tutor autenticado e maior de 18 (RN-09); CPF válido; espécie em lista controlada; data de nascimento não futura.
- **Pós-condições:** pet criado com status ATIVO.

### CT-08 — autenticar

- **Operação:** `autenticar(email: String, senha: String): TokenJWT`
- **UC:** UC-01
- **Pré-condições:** e-mail cadastrado; conta não BLOQUEADA; senha não expirada (RN-14).
- **Pós-condições:** hash BCrypt conferido; JWT emitido (exp 24h); login registrado em auditoria. Após 5 falhas seguidas, conta bloqueia por 15 min.

### CT-09 — gerarRelatorioFinanceiro

- **Operação:** `gerarRelatorioFinanceiro(adminId: Long, mesRef: YearMonth, filtros: Map): RelatorioMensalDTO`
- **UC:** UC-15
- **Pré-condições:** solicitante é Administrador; mês de referência ≤ mês atual.
- **Pós-condições:** dados agregados de pagamentos CONFIRMADO no período (RN-11); PDF gerado; total bruto, por método e por veterinário disponíveis.

### CT-10 — enviarLembrete

- **Operação:** `enviarLembrete(notificacaoId: Long): ResultadoEnvio`
- **UC:** UC-16
- **Pré-condições:** notificação com status PENDENTE; canal definido; destinatário com e-mail válido.
- **Pós-condições:** e-mail enviado via SMTP; status passa pra ENVIADA ou FALHA; após 3 tentativas (RN-12), notificação marcada como DESCARTADA e admin alertado.

---

## 4. Modelos de Projeto

### 4.1 Arquitetura

São seis microsserviços, cada um com seu PostgreSQL. A conversa entre eles passa por RabbitMQ quase sempre — quando algo importante acontece (consulta marcada, vacina aplicada, pagamento confirmado), o serviço publica o evento e quem se importa consome. REST síncrono ficou só onde a resposta faz parte da UX direta: marcar, pagar, atender.

| Serviço          | Responsabilidade |
|------------------|------------------|
| ms-usuarios      | Autenticação JWT, CRUD de perfis (tutor/recepcionista/vet/admin). |
| ms-agendamento   | Consultas, agenda dos vets, regras de conflito de horário. |
| ms-prontuario    | Anamnese, diagnóstico, vacinas, prescrições. Upload de exames pra S3. |
| ms-pagamento     | Integração com gateway Pix/cartão, webhooks, estorno. |
| ms-notificacao   | Worker que consome a fila e dispara e-mail. Scheduler de lembretes. |
| ms-relatorios    | Leitura cruzada de pagamento e agendamento para relatórios. |

Por dentro, mantive a divisão clássica de Spring Boot: controller só recebe HTTP e valida; service segura a regra de negócio (o "não pode marcar dois pets no mesmo horário pro mesmo vet" mora aí); repository é Spring Data JPA puro; DTO separa entidade de contrato de API. Nada inovador — é o que funciona e o que outro dev entende rápido.

Na borda fica um API Gateway que valida o JWT e roteia via ALB. Front-end React (SPA) e React Native (mobile) entram por CloudFront → API Gateway.

Os padrões que usei foram Repository, Service Layer e DTO no tradicional. Event-Driven via RabbitMQ pra desacoplar serviço de notificação e relatório. Strategy entrou no cálculo de próxima dose de vacina, porque o protocolo muda por tipo (antirrábica é anual, V8/V10 tem ciclo escalonado, etc.) — codificar isso em `if` ia ficar feio. E Factory na criação de notificação por canal, prevendo que um dia entra WhatsApp.

**Sobre microsserviços vs monolito.** Já mencionei na introdução, mas vale registrar aqui: pra clínica única isso é overengineering. Reconheço. A justificativa é que o desenho mira numa rede de clínicas — naquele cenário, ter o ms-pagamento isolado (com seu próprio time, seu próprio ciclo de deploy, sua certificação PCI à parte) faz sentido. Se fosse implementar de novo só pra minha tia, faria um monolito modular com os mesmos limites de domínio, e migrava se a hora chegasse.

**Banco por serviço, sem join entre eles.** Quando ms-relatorios precisa cruzar pagamento com agendamento, ou ele lê dos dois bancos em read-only ou consome eventos e monta sua própria view materializada. Aceitei a consistência eventual — relatório pode demorar uns segundos pra refletir um pagamento que acabou de cair, e tá tudo bem.

**JWT stateless** e **PostgreSQL** vieram quase como decisões default. Pagamento e agendamento querem ACID, então NoSQL ficou de fora.

### 4.2 Componentes e Implantação

> Fonte: [`Codigo/Componentes/diagrama-de-componentes-e-implantacao.puml`](../Codigo/Componentes/diagrama-de-componentes-e-implantacao.puml)

O diagrama separa o lado físico: Web e Mobile passam por CloudFront e API Gateway, batem no ALB e caem no cluster Docker/ECS onde rodam os seis microsserviços com seus bancos. RabbitMQ orquestra a troca assíncrona. Por fora ficam o gateway de pagamento, o SMTP e o S3 (anexos de exame).

### 4.3 Diagrama de Classes

> Fonte: [`Codigo/Classes/diagrama-de-classes.puml`](../Codigo/Classes/diagrama-de-classes.puml)

O modelo de domínio gira em torno da hierarquia `Usuario` (abstrata) → Tutor, Recepcionista, Veterinario, Administrador. A interface `Autenticavel` define o contrato pra qualquer entidade que precisa logar.

As entidades de negócio são `Pet`, `Consulta` (com o enum `StatusConsulta` cobrindo todo o ciclo de vida dela), `Prontuario`, `Vacina` e a classe associativa `AplicacaoVacina` (que fica entre Pet, Vacina e Prontuário carregando lote e próxima dose), `Prescricao`, `Medicamento`, `Pagamento` e `Notificacao`.

Os enums (`StatusConsulta`, `TipoConsulta`, `EspecieAnimal`, `Especialidade`, `TipoVacina`, `MetodoPagamento`, `StatusPagamento`, `CanalNotificacao`, `StatusNotificacao`, `StatusPet`, `StatusUsuario`) ficaram explícitos no diagrama porque ajudam a entender o domínio sem ter que abrir o código.

### 4.4 Diagramas de Sequência

Cobri os sete fluxos que importam. Cada um mostra a interação entre cliente, API Gateway, microsserviço, repositórios e — onde cabia — RabbitMQ e serviço externo. Os ramos `alt` e `opt` representam as RNs aplicáveis e os caminhos de exceção.

| UC    | Cenário                          | Arquivo |
|-------|----------------------------------|---------|
| UC-04 | Agendar Consulta                 | [`UC-04-agendar-consulta.puml`](../Codigo/Sequencia/UC-04-agendar-consulta.puml) |
| UC-05 | Cancelar Consulta                | [`UC-05-cancelar-consulta.puml`](../Codigo/Sequencia/UC-05-cancelar-consulta.puml) |
| UC-07 | Atender Consulta e Prontuário    | [`UC-07-atender-consulta.puml`](../Codigo/Sequencia/UC-07-atender-consulta.puml) |
| UC-10 | Aplicar Vacina com Reforço       | [`UC-10-aplicar-vacina.puml`](../Codigo/Sequencia/UC-10-aplicar-vacina.puml) |
| UC-11 | Pagar Consulta via Pix           | [`UC-11-pagar-consulta.puml`](../Codigo/Sequencia/UC-11-pagar-consulta.puml) |
| UC-15 | Gerar Relatório Financeiro       | [`UC-15-gerar-relatorio.puml`](../Codigo/Sequencia/UC-15-gerar-relatorio.puml) |
| UC-16 | Job de Envio de Lembrete         | [`UC-16-enviar-lembrete.puml`](../Codigo/Sequencia/UC-16-enviar-lembrete.puml) |

### 4.5 Diagramas de Comunicação

Fiz dois — mesma informação que as sequências, mas com o foco no quem-chama-quem em vez do quando:

- [Comunicação UC-04 — Agendar Consulta](../Codigo/Comunicacao/comunicacao-UC04-agendar-consulta.puml)
- [Comunicação UC-10 — Aplicar Vacina](../Codigo/Comunicacao/comunicacao-UC10-aplicar-vacina.puml)

### 4.6 Diagramas de Estado

Modelei estado pras quatro entidades cujo ciclo de vida tem peso suficiente:

- **Consulta** — do agendamento até o pagamento, passando por confirmação, atendimento e realização. Inclui cancelamento e falta. [`diagrama-de-estado-consulta.puml`](../Codigo/Estado/diagrama-de-estado-consulta.puml)
- **Vacinação** — ciclo: pendente → aplicada → vencendo → vencida → reforço aplicado. Com guards das RN-04 e RN-05. [`diagrama-de-estado-vacinacao.puml`](../Codigo/Estado/diagrama-de-estado-vacinacao.puml)
- **Pagamento** — pendente → processando → aprovado/recusado/expirado → confirmado/estornado. O timeout do Pix (RN-08) vive aqui. [`diagrama-de-estado-pagamento.puml`](../Codigo/Estado/diagrama-de-estado-pagamento.puml)
- **Pet** — ativo → inativo (transferido ou óbito). Histórico preservado mesmo depois de inativado (RN-15). [`diagrama-de-estado-pet.puml`](../Codigo/Estado/diagrama-de-estado-pet.puml)

---

## 5. Modelos de Dados

### 5.1 Tecnologia de Persistência

Cada microsserviço tem o próprio **PostgreSQL 16**, sem compartilhar tabela com vizinho. Acesso por Spring Data JPA + Hibernate.

A herança da hierarquia `Usuario` ficou em **table-per-subclass**: uma tabela base `usuarios` e as filhas `tutores`, `recepcionistas`, `veterinarios` e `administradores`, cada uma com `id` referenciando `usuarios.id`. Escolhi essa estratégia em vez de `SINGLE_TABLE` porque os perfis têm atributos bem diferentes entre si (CRMV só faz sentido pra vet, grau de parentesco só pra responsável de tutor menor) e uma tabela única ficaria cheia de coluna nula.

Enums são gravados como `varchar`, não inteiro. Custa um pouco mais de espaço, mas quando abro o psql pra debugar relatório consigo ler `CONFIRMADO` em vez de `3`.

### 5.2 Diagrama Entidade-Relacionamento

> Fonte: [`Codigo/ER/diagrama-er-vetcareplus.puml`](../Codigo/ER/diagrama-er-vetcareplus.puml)

São 14 tabelas distribuídas pelos bancos:

| Microsserviço    | Tabelas |
|------------------|---------|
| ms-usuarios      | `usuarios`, `tutores`, `recepcionistas`, `veterinarios`, `administradores` |
| ms-agendamento   | `consultas`, `pets` |
| ms-prontuario    | `prontuarios`, `vacinas`, `aplicacoes_vacina`, `medicamentos`, `prescricoes` |
| ms-pagamento     | `pagamentos` |
| ms-notificacao   | `notificacoes` |

### 5.3 Índices Importantes

| Tabela              | Índice                                | Justificativa |
|---------------------|----------------------------------------|---------------|
| usuarios            | `email` (unique)                       | Login |
| veterinarios        | `crmv` (unique)                        | Identificação profissional |
| consultas           | `(veterinario_id, data_hora)`          | Checagem de conflito (RN-01) |
| consultas           | `status`                               | Filtro de agenda do dia |
| pagamentos          | `(status, pago_em)`                    | Relatório financeiro (RN-11) |
| aplicacoes_vacina   | `proxima_dose`                         | Job de lembrete (RN-04) |
| notificacoes        | `(status, data_envio)`                 | Job de envio (RN-12) |

---

## 6. Escopo do MVP e Versões Futuras

### MVP (v1.0 — entrega)

A v1.0 entrega o ciclo completo do dia a dia da clínica. JWT com os quatro perfis. Cadastro de tutor e pet. Agendamento com cancelamento e reagendamento. Atendimento com prontuário, vacina e prescrição. Pagamento por Pix e cartão. Lembrete por e-mail (24h antes da consulta; 30 e 7 dias antes do vencimento da vacina). Relatório financeiro mensal em PDF.

### v1.1 — Próximo ciclo

Upload de exame complementar (PDF e imagem) pro S3, app mobile do tutor em React Native e notificação push. Coisas que ampliam, mas não mudam o domínio.

### v2.0 — Visão de longo prazo

WhatsApp como canal alternativo de notificação (a RN-12 abre pra isso). Telemedicina veterinária com videoconsulta integrada. Integração com farmácia parceira pra entrega de medicação. Programa de fidelidade.

### Fora de escopo (decisão consciente)

Algumas coisas deixei de fora de propósito.

**Estoque de medicamentos** não entrou porque clínica pequena terceiriza a venda — minha tia mesma encaminha pra petshop ao lado. Modelar entrada/saída de estoque ia ser mais sistema do que valor entregue.

**Internação e centro cirúrgico** ficou fora porque é outro domínio: anestesia, escala 24h, monitoramento contínuo. Merece módulo separado e talvez um ms próprio. Tentar enfiar no agendamento ia poluir tudo.

**Faturamento por convênio pet** (Petlove Saúde, Pet Vital) ficou de fora porque o mercado ainda é pequeno no perfil de cliente que tenho em mente. Quando ganhar tração, vira tema pra uma v3.

---

*Fim do documento.*
