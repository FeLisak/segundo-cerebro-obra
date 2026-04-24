# Segundo Cérebro de Obra — Knowledge Graph

Este documento descreve a arquitetura do Knowledge Graph do Segundo Cérebro de Obra: como os dados do RDO Web e comunicações contratuais são ingeridos, classificados e transformados em inteligência acionável para gestão contratual e montagem de pleitos.

---

## Diagrama

```mermaid
flowchart TD

    subgraph FONTES["FONTES DE DADOS"]
        direction LR
        F1["RDO Web — atividades · paralisacoes · efetivo · fatos relevantes"]
        F2["Comunicacoes — cartas · notificacoes · atas"]
        F3["Contrato — clausulas · cronograma · medicoes"]
    end

    subgraph LOOP["CLASSIFICACAO + GRAFO — rodam juntos"]
        direction TB

        P1["Ingestao — CSV/Excel exportado do RDO Web · documentos textuais"]
        P2["Dedup — Redis hash SHA-256 · TTL por obra"]

        subgraph CLASSIF["motor de classificacao"]
            direction LR
            LLM{"LLM — este registro afeta alguma entidade da obra?"}
            CRIAR{"da pra criar ou atualizar alguma entidade?"}

            subgraph KG["Knowledge Graph — Neo4j · Graphiti"]
                direction LR

                subgraph CTX["contexto da obra"]
                    direction TB
                    E1["Obra — nome · cliente · fiscalizadora · fase · data_fim_contratual"]
                    E2["Atividade — tipo · data_inicio · data_fim · efetivo · status"]
                    E3["Comunicacao — tipo · data · destinatario · trecho_relevante · caminho_doc"]
                    E5["Motivo — categoria · responsabilidade · recorrencia · horas_acumuladas"]
                    E2 -->|PERTENCE_A| E1
                    E3 -->|ASSOCIADA_A| E2
                end

                E4["PARALISACAO — categoria · horas · responsabilidade · data · status"]

                E6["Periodo — tipo · data_inicio · data_fim · horas_total · count"]

                E1 -->|TEM_PARALISACAO — horas · horas_acumuladas| E4
                E2 -->|IMPACTADA_POR| E4
                E3 -->|REFERENCIA — trecho_relevante| E4
                E5 -->|CLASSIFICA| E4
                E4 -->|OCORREU_EM — valid_from · valid_until| E6
            end

            LLM -->|sim — cria no Paralisacao vinculado| KG
            LLM -->|nao afeta entidade existente| CRIAR
            CRIAR -->|sim — nova atividade · motivo · comunicacao| KG
            CRIAR -->|nao ha nada aproveitavel| DESCARTE[["descartado"]]
            LLM <-->|consulta e atualiza| KG
        end

        P1 -->|hash do conteudo| P2
        P2 -->|novo registro| LLM
    end

    subgraph OUTPUT["OUTPUT"]
        direction LR
        O1["Levantamento — paralisacoes por motivo · frequencia · horas por periodo"]
        O2["Narrativa Argumentativa — rascunho de pleito auditavel com referencias"]
        O3["Alertas de Padrao — paralisacao recorrente detectada · acao preventiva"]
    end

    FONTES --> P1
    E4 -->|alimenta| O1
    E3 -->|cruza com| O2
    O1 --> O2
    E4 -->|dispara| O3

    classDef fonte    fill:#1e2330,stroke:#3a4560,color:#8892a4
    classDef pipeline fill:#0a2a1a,stroke:#00c896,color:#00c896
    classDef entity   fill:#0e1428,stroke:#4a9eff,color:#b8d8ff
    classDef risco    fill:#3d0a0a,stroke:#ff5f6d,color:#ff5f6d
    classDef output   fill:#1c1200,stroke:#f5a623,color:#f5a623
    classDef descarte fill:#111,stroke:#333,color:#555
    classDef decisao  fill:#1a1400,stroke:#f5a623,color:#f5a623

    class F1,F2,F3 fonte
    class P1,P2 pipeline
    class E1,E2,E3,E5,E6 entity
    class E4 risco
    class O1,O2,O3 output
    class DESCARTE descarte
    class CRIAR decisao
```

---

## Por que um Knowledge Graph?

Obras de infraestrutura produzem dois tipos de dado que raramente se conversam: o dado **quantitativo** — horas de paralisação, efetivo alocado, atividades executadas, registrado no RDO Web — e o dado **qualitativo** — a carta enviada ao cliente naquela semana, a notificação sobre atraso de fornecimento, a ata onde o cliente assumiu responsabilidade por uma interferência.

O valor não está em nenhum dos dois isoladamente. Está na conexão entre eles. Um Knowledge Graph representa essas conexões nativamente: ele sabe que a paralisação do dia X foi pelo motivo W, que esse motivo é de responsabilidade do cliente, que há uma carta de notificação associada com data anterior à paralisação, e que o impacto acumulado no mês justifica um pleito. Essa é a história da obra — e hoje ela existe apenas na memória dos gestores.

---

## Fontes de dados

O grafo é alimentado por três categorias de fontes, em ordem de prioridade de ingestão:

**RDO Web** — extrato de atividades, registro de paralisações, controle de efetivo e fatos relevantes, exportados em CSV/Excel. É a fonte mais estruturada e o ponto de partida. A automação via API do RDO Web é avaliada em paralelo para eliminar o processo de exportação manual em versões futuras.

**Comunicações contratuais** — cartas ao cliente, notificações formais e atas de reunião. São fontes textuais não estruturadas que, cruzadas com os dados do RDO Web, constroem o registro qualificado da obra: quem disse o quê, quando, sobre qual paralisação.

**Contrato** — cláusulas contratuais, cronograma de entregas e medições. Contextualiza a responsabilidade de cada motivo de paralisação (Contratada / Cliente / Externo) e define os marcos temporais relevantes para o cálculo de impacto.

---

## Pipeline de classificação

O pipeline tem uma premissa central: **nenhum dado é armazenado antes de ser classificado como relevante para a obra**. Isso evita acúmulo de registros sem valor e mantém o grafo limpo e auditável.

### Ingestão e deduplicação

Todo registro ingerido passa por deduplicação via Redis. Um hash SHA-256 é calculado sobre o conteúdo e verificado contra os hashes já processados dentro da janela de TTL configurada por obra. Se o registro já foi visto, é descartado imediatamente. Se é novo, segue para classificação.

### Classificação com contexto do grafo

O LLM e o Knowledge Graph rodam juntos, não em sequência. O modelo não classifica registros de forma genérica — ele classifica em relação ao contexto específico da obra, consultando o grafo em tempo real para saber quais atividades estão em andamento, quais motivos de paralisação já foram mapeados e quais comunicações existem no período.

O fluxo de decisão tem duas etapas:

**Etapa 1 — O registro afeta alguma entidade existente no grafo?** Se sim, o modelo extrai motivo, horas, responsabilidade e período, e cria um nó `:Paralisacao` já vinculado às entidades afetadas. Nenhum documento intermediário é salvo.

**Etapa 2 — Se não afeta diretamente, dá para criar ou atualizar alguma entidade?** Uma nova atividade, um novo motivo de paralisação, uma comunicação ainda não mapeada — tudo isso enriquece o grafo mesmo sem gerar uma paralisação imediata. Só após essa segunda verificação, se não houver nada aproveitável, o registro é descartado.

---

## Entidades do Knowledge Graph

### `:Paralisacao`

O nó central de todo o grafo. Todo o sistema existe para identificar, classificar e quantificar paralisações e seu impacto contratual. Um nó `:Paralisacao` só é criado quando o LLM confirma relevância para alguma entidade da obra.

| Campo | Descrição |
|---|---|
| `categoria` | `falta_material`, `climatico`, `aprovacao_cliente`, `interferencia_terceiros`, `decisao_cliente`, `outro` |
| `horas` | Total de horas perdidas no evento |
| `responsabilidade` | `Contratada`, `Cliente` ou `Externo` |
| `data_inicio` | Início da paralisação |
| `data_fim` | Fim da paralisação |
| `status` | `aberto`, `documentado`, `pleiteado` |
| `created_at` | Timestamp de criação no grafo |

### `:Obra`

Entidade raiz do grafo. Todas as demais entidades pertencem a uma obra.

| Campo | Descrição |
|---|---|
| `nome` | Identificador da obra |
| `cliente` | Contratante |
| `fiscalizadora` | Órgão fiscalizador quando aplicável |
| `data_inicio_contratual` | Data de início prevista em contrato |
| `data_fim_contratual` | Data de entrega prevista |
| `fase_atual` | Fase contratual corrente |
| `status` | `em_andamento`, `paralisada`, `concluida` |

### `:Atividade`

Cada frente de trabalho em execução na obra. Paralisações são sempre vinculadas a uma ou mais atividades impactadas.

| Campo | Descrição |
|---|---|
| `tipo` | Concretagem, instalações, estrutura, mobilização... |
| `data_inicio` | Início previsto ou realizado |
| `data_fim` | Término previsto ou realizado |
| `efetivo_previsto` | Número de trabalhadores previstos |
| `efetivo_realizado` | Número de trabalhadores efetivamente alocados |
| `status` | `em_andamento`, `paralisada`, `concluida` |

### `:Comunicacao`

Toda comunicação formal da obra: cartas ao cliente, notificações e atas de reunião. É a entidade que conecta o dado quantitativo do RDO Web ao registro qualitativo contratual.

| Campo | Descrição |
|---|---|
| `tipo` | `carta`, `notificacao`, `ata` |
| `data` | Data de emissão |
| `destinatario` | Cliente, fiscalizadora ou terceiro |
| `trecho_relevante` | Trecho extraído pelo LLM que justifica o vínculo com a paralisação |
| `caminho_doc` | Caminho para o arquivo físico do documento (PDF, digitalização) |

### `:Motivo`

Taxonomia de motivos de paralisação. Permite detectar recorrência e calcular a participação percentual de cada categoria ao longo do tempo.

| Campo | Descrição |
|---|---|
| `categoria` | Categoria principal do motivo |
| `descricao` | Descrição detalhada |
| `responsabilidade` | `Contratada`, `Cliente` ou `Externo` |
| `recorrencia` | Contador de ocorrências na obra |
| `horas_acumuladas` | Total de horas perdidas por este motivo na obra |

### `:Periodo`

Agrupa paralisações por recorte temporal, permitindo quantificação por semana, mês ou fase contratual.

| Campo | Descrição |
|---|---|
| `tipo` | `semana`, `mes`, `fase_contratual` |
| `data_inicio` | Início do período |
| `data_fim` | Fim do período |
| `horas_total` | Total de horas de paralisação no período |
| `count` | Número de eventos de paralisação no período |

---

## Relações entre entidades

| Relação | De → Para | Propriedades |
|---|---|---|
| `TEM_PARALISACAO` | Obra → Paralisacao | `horas`, `horas_acumuladas` |
| `IMPACTADA_POR` | Atividade → Paralisacao | — |
| `REFERENCIA` | Comunicacao → Paralisacao | `trecho_relevante` |
| `CLASSIFICA` | Motivo → Paralisacao | — |
| `OCORREU_EM` | Paralisacao → Periodo | `valid_from`, `valid_until` |
| `PERTENCE_A` | Atividade → Obra | — |
| `ASSOCIADA_A` | Comunicacao → Atividade | `data`, `tipo` |

---

## Como o impacto é medido

O impacto de uma paralisação é medido em **horas absolutas**, não em percentual. Essa decisão tem três razões práticas:

A primeira é que as horas já existem no RDO Web — são o dado mais estruturado e confiável disponível desde o início. A segunda é que horas são diretamente auditáveis: o argumento de pleito aponta para o registro específico do RDO Web que originou aquele número, sem margem para contestação. A terceira é que o percentual pode ser calculado em cima das horas quando necessário — horas perdidas divididas pelas horas previstas no período — mas depende do efetivo previsto estar corretamente preenchido, o que será validado ao longo do projeto.

Na prática, o grafo armazena dois campos de horas que servem a propósitos distintos:

**`horas`** na aresta `TEM_PARALISACAO` — as horas de um evento específico de paralisação. É o dado atômico, rastreável até o registro individual do RDO Web.

**`horas_acumuladas`** no nó `:Motivo` — a soma de todas as horas perdidas por aquele motivo ao longo da obra. É o dado agregado que alimenta o argumento de pleito: "foram X horas perdidas por atraso de aprovação do cliente no período de Y a Z."

O `:Periodo` também armazena `horas_total`, que é a soma de todas as paralisações naquele recorte temporal, independente do motivo — útil para comparações entre fases da obra.

---

## Output

**Levantamento de Paralisações** — classificação automática por motivo, frequência e horas perdidas, segmentada por semana, mês ou fase contratual. Base auditável para pleitos com rastreabilidade direta ao RDO Web.

**Narrativa Argumentativa** — o LLM cruza os dados quantitativos das paralisações com as comunicações formais associadas e gera um rascunho de argumentação de pleito: cronologia dos fatos, responsabilidades, horas acumuladas por motivo e referências documentais. O que levava semanas de levantamento manual passa a levar horas.

**Alertas de Padrão** — quando um motivo de paralisação acumula recorrência acima de um threshold configurável, o sistema emite um alerta para que a equipe tome ação preventiva antes que o impacto se torne irreversível no cronograma.
