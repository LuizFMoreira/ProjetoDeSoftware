# Documentação - VetCare+

Esta pasta guarda os diagramas do projeto, feitos em PlantUML.

## Diagramas

| Arquivo | O que mostra |
|---|---|
| [01-casos-de-uso.puml](diagrams/01-casos-de-uso.puml) | Atores e principais funcionalidades do sistema |
| [02-classes.puml](diagrams/02-classes.puml) | Entidades do domínio, atributos e relacionamentos |
| [03-sequencia-agendamento-consulta.puml](diagrams/03-sequencia-agendamento-consulta.puml) | Fluxo do tutor agendando uma consulta |
| [04-componentes-arquitetura.puml](diagrams/04-componentes-arquitetura.puml) | Visão de componentes / deploy da aplicação |

## Como renderizar

Tem duas formas que uso:

1. **Extensão do VS Code** "PlantUML" (jebbs.plantuml). Abre o `.puml` e aperta `Alt+D` que ele renderiza num painel ao lado.
2. **Linha de comando** (precisa do Java e do `plantuml.jar`):

```bash
java -jar plantuml.jar diagrams/01-casos-de-uso.puml
```

Também dá pra usar o site oficial: <https://www.plantuml.com/plantuml/uml/> — cola o conteúdo do arquivo e ele renderiza.
