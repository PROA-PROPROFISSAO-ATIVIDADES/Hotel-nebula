# Parte 1 — Origem do Sistema: Modelagem

## 1. Núcleos de informação identificados

1. **Pessoas:** hóspedes e funcionários.
2. **Estrutura física e catálogo:** quartos e serviços oferecidos.
3. **Fluxo de negócio:** reserva (intenção) → hospedagem (estadia real) → pagamento.
4. **Opinião:** feedbacks dos clientes.

## 2. Padrões de acesso (as consultas que guiaram a modelagem)

Em banco não relacional o documento é desenhado a partir de **como a informação será lida**. Antes de definir as coleções, listei as perguntas que o hotel precisa responder:

| # | Consulta | Coleção principal | O que ela exige do modelo |
|---|---|---|---|
| 1 | Quartos por característica (vista para o mar, banheira, luxo) | `quartos` | `caracteristicas` como **array** + índice multikey; `categoria` indexada |
| 2 | Hóspedes que reservaram mais de uma vez | `reservas` | `hospedeId` na reserva (referência) para agrupar |
| 3 | Faturamento por período (mês, ano) | `pagamentos` | valor como número inteiro (centavos), `dataPagamento` como `Date`, `status` para excluir estornos |
| 4 | Ocupação por mês e dia da semana | `reservas` | `dataCheckin`/`dataCheckout` como `Date`, `noites` já calculado |
| 5 | Reservas de um hóspede (mais recentes primeiro) | `reservas` | índice `{ hospedeId, dataCheckin }` |
| 6 | Criar reserva sem conflito de datas no quarto | `reservas` + `quartos` | índice `{ quartoId, dataCheckin }` e operação transacional |
| 7 | Últimas avaliações de um quarto | `feedbacks` | `quartoId` no feedback + índice `{ quartoId, data }` |

## 3. Coleções

| Coleção | Papel | Principais campos |
|---|---|---|
| `hospedes` | Cadastro do cliente | nome, email, documento{}, contato, preferencias[], fidelidade{} |
| `quartos` | Inventário | numero, categoria, andar, capacidade, precoDiariaCentavos, caracteristicas[] |
| `funcionarios` | Profissionais | nome, cargo, turno, habilidades[] |
| `servicos` | Catálogo de serviços extras | nome, categoria, precoCentavos, unidade |
| `reservas` | Intenção de hospedagem | hospedeId + snapshot, quartoId + snapshot, datas, valores, status, historicoStatus[] |
| `hospedagens` | Estadia efetiva (check-in/out) | reservaId, checkinReal, checkoutReal, servicosConsumidos[] |
| `pagamentos` | Movimento financeiro | reservaId, hospedagemId, valorCentavos, metodo, status, detalhes{} |
| `feedbacks` | Avaliações | hospedagemId, hospedeId, quartoId, nota, comentario, categorias{} |

## 4. Proposta de modelagem

Modelo **híbrido**, guiado por três regras:

1. **Embutir** o que é pequeno, limitado e sempre lido junto com o pai (serviços de uma estadia, histórico de status, documento do hóspede).
2. **Referenciar** (guardar o `_id`) o que tem vida própria ou cresce sem limite (hóspede, quarto, pagamentos, avaliações).
3. **Copiar um snapshot** do que não pode mudar retroativamente (preço da diária, nome do serviço, nome do hóspede no momento da reserva).

```
hospedes 1───N reservas 1───0..1 hospedagens 1───N pagamentos
                 │                    │ └─ servicosConsumidos[] (embutido, snapshot do catálogo)
quartos 1────────┘                    └──0..1 feedbacks
funcionarios ── referenciados em hospedagens (check-in) e feedbacks (resposta)
```

**Convenções:** `_id` como `ObjectId`; datas como `Date` (UTC); dinheiro em **centavos** (inteiro), nunca `double`, para não acumular erro de arredondamento; nomes de campos em camelCase, sem acento.

## 5. Respostas às questões importantes

**Uma reserva guarda dados do hóspede embutidos ou referenciados?**
Os dois. Guarda `hospedeId` (referência, fonte da verdade, usada em `$lookup` e para agrupar) **e** um pequeno snapshot (`hospede: {nome, email}`). A referência evita duplicar o cadastro completo; o snapshot evita um join para listar reservas e preserva o nome como estava na data da reserva. O mesmo vale para o quarto (`quarto: {numero, categoria}`).

**Avaliações ficam dentro do quarto ou em coleção própria?**
Coleção própria (`feedbacks`). Um quarto acumula avaliações indefinidamente: embutir faria o documento crescer sem teto (limite de 16 MB e reescrita cada vez mais cara). Em coleção própria é simples agregar notas por quarto, por período ou por hóspede.

**Serviços consumidos ficam dentro da hospedagem?**
Sim, como array embutido (`servicosConsumidos`). Só existem no contexto de uma estadia, são lidos junto com ela e o volume por estadia é pequeno e limitado. Cada item guarda `valorUnitarioCentavos` e `nome` copiados do catálogo: mudar o preço no catálogo não altera o histórico.

**Como representar histórico sem perder flexibilidade?**
- `reservas.historicoStatus[]`: array de `{status, em, por}`, uma entrada a cada mudança (criada → confirmada → concluída/cancelada).
- Snapshots de preço e nome nos documentos de fato (`precoDiariaCentavos`, `valorUnitarioCentavos`).
- Documentos não são apagados: o ciclo de vida é controlado por `status`.
- Campos opcionais (`detalhes` em pagamentos, `preferencias` em hóspedes) variam por registro sem exigir migração.

## 6. Justificativas resumidas

| Decisão | Motivo |
|---|---|
| `reservas` e `hospedagens` separadas | Nem toda reserva vira hospedagem (cancelamento, no-show) e a estadia real tem dados próprios (horários reais, serviços) |
| `pagamentos` separado | Uma reserva pode ter vários pagamentos (sinal, saldo, estorno) e o financeiro consulta isso isoladamente |
| `servicos` como catálogo | Preço e descrição centralizados; os itens consumidos guardam cópia |
| `caracteristicas[]` em quartos | Consulta por característica vira `find` simples com índice multikey |
| Dinheiro em centavos | Soma exata nas agregações de faturamento |
| `ObjectId` nos `_id` | Padrão do MongoDB; comparação e índice eficientes. Os IDs dos exemplos são previsíveis só para facilitar a conferência |
| Índices | Cada índice atende uma consulta da seção 2 (tabela em `02-ARQUITETURA.md`) |

## 7. Esquema inicial das coleções

```
hospedes     { _id, nome, email, telefone, documento{tipo,numero}, nacionalidade,
               endereco{cidade,estado,pais}?, preferencias[]?, fidelidade{nivel,pontos}?, criadoEm }
quartos      { _id, numero, andar, categoria, capacidade, precoDiariaCentavos, caracteristicas[], ativo }
funcionarios { _id, nome, cargo, turno, habilidades[]?, admissao, ativo }
servicos     { _id, nome, categoria, precoCentavos, unidade }
reservas     { _id, hospedeId, hospede{nome,email}, quartoId, quarto{numero,categoria},
               dataCheckin, dataCheckout, noites, numHospedes, precoDiariaCentavos, valorDiariasCentavos,
               status, canal, observacoes?, criadaEm, historicoStatus[{status, em, por, porId?}] }
hospedagens  { _id, reservaId, hospedeId, quartoId, checkinReal, checkoutReal, funcionarioCheckinId,
               servicosConsumidos[{servicoId, nome, quantidade, valorUnitarioCentavos, data}],
               totalServicosCentavos, status }
pagamentos   { _id, reservaId, hospedagemId?, hospedeId, tipo, valorCentavos, metodo, status,
               dataPagamento, detalhes{} }
feedbacks    { _id, hospedagemId, hospedeId, quartoId, nota, categorias{}?, comentario, data,
               resposta{funcionarioId, texto, data}? }
```
`?` = campo opcional (varia entre documentos). Os exemplos abaixo estão em Extended JSON (`$oid` = ObjectId, `$date` = Date), o mesmo formato dos arquivos de [`dados/`](dados/).

## 8. Exemplos de documentos JSON

### hospedes
```json
{
  "_id": {
    "$oid": "a10000000000000000000001"
  },
  "nome": "Mariana Costa",
  "email": "mariana.costa@email.com",
  "telefone": "+55 11 90000-0001",
  "documento": {
    "tipo": "CPF",
    "numero": "000.000.001-00"
  },
  "nacionalidade": "BR",
  "endereco": {
    "cidade": "São Paulo",
    "estado": "SP",
    "pais": "Brasil"
  },
  "preferencias": [
    "andar alto",
    "sem glúten"
  ],
  "fidelidade": {
    "nivel": "ouro",
    "pontos": 1800
  },
  "criadoEm": {
    "$date": "2025-11-02T18:00:00.000Z"
  }
}
```

### quartos
```json
{
  "_id": {
    "$oid": "a20000000000000000000201"
  },
  "numero": 201,
  "andar": 2,
  "categoria": "luxo",
  "capacidade": 3,
  "precoDiariaCentavos": 65000,
  "caracteristicas": [
    "vista para o mar",
    "banheira",
    "varanda",
    "minibar",
    "wifi",
    "ar-condicionado"
  ],
  "ativo": true
}
```

### reservas (hóspede e quarto referenciados + snapshot, histórico embutido)
```json
{
  "_id": {
    "$oid": "a50000000000000000000001"
  },
  "hospedeId": {
    "$oid": "a10000000000000000000001"
  },
  "hospede": {
    "nome": "Mariana Costa",
    "email": "mariana.costa@email.com"
  },
  "quartoId": {
    "$oid": "a20000000000000000000201"
  },
  "quarto": {
    "numero": 201,
    "categoria": "luxo"
  },
  "dataCheckin": {
    "$date": "2026-03-10T18:00:00.000Z"
  },
  "dataCheckout": {
    "$date": "2026-03-13T15:00:00.000Z"
  },
  "noites": 3,
  "numHospedes": 2,
  "precoDiariaCentavos": 65000,
  "valorDiariasCentavos": 195000,
  "status": "concluida",
  "canal": "site",
  "criadaEm": {
    "$date": "2026-02-20T14:00:00.000Z"
  },
  "historicoStatus": [
    {
      "status": "criada",
      "em": {
        "$date": "2026-02-20T14:00:00.000Z"
      },
      "por": "sistema"
    },
    {
      "status": "confirmada",
      "em": {
        "$date": "2026-02-20T14:05:00.000Z"
      },
      "por": "sistema"
    },
    {
      "status": "concluida",
      "em": {
        "$date": "2026-03-13T15:00:00.000Z"
      },
      "por": "funcionario",
      "porId": {
        "$oid": "a30000000000000000000001"
      }
    }
  ],
  "observacoes": "Aniversário de casamento"
}
```

### hospedagens (serviços embutidos com snapshot de preço)
```json
{
  "_id": {
    "$oid": "a60000000000000000000001"
  },
  "reservaId": {
    "$oid": "a50000000000000000000001"
  },
  "hospedeId": {
    "$oid": "a10000000000000000000001"
  },
  "quartoId": {
    "$oid": "a20000000000000000000201"
  },
  "checkinReal": {
    "$date": "2026-03-10T18:30:00.000Z"
  },
  "checkoutReal": {
    "$date": "2026-03-13T14:50:00.000Z"
  },
  "funcionarioCheckinId": {
    "$oid": "a30000000000000000000001"
  },
  "servicosConsumidos": [
    {
      "servicoId": {
        "$oid": "a40000000000000000000002"
      },
      "nome": "Spa",
      "quantidade": 1,
      "valorUnitarioCentavos": 22000,
      "data": {
        "$date": "2026-03-11T20:00:00.000Z"
      }
    },
    {
      "servicoId": {
        "$oid": "a40000000000000000000003"
      },
      "nome": "Lavanderia",
      "quantidade": 1,
      "valorUnitarioCentavos": 6000,
      "data": {
        "$date": "2026-03-12T20:00:00.000Z"
      }
    }
  ],
  "totalServicosCentavos": 28000,
  "status": "encerrada"
}
```

### pagamentos (o campo `detalhes` varia conforme o método)
```json
{
  "_id": {
    "$oid": "a70000000000000000000001"
  },
  "reservaId": {
    "$oid": "a50000000000000000000001"
  },
  "hospedagemId": {
    "$oid": "a60000000000000000000001"
  },
  "hospedeId": {
    "$oid": "a10000000000000000000001"
  },
  "tipo": "total",
  "valorCentavos": 223000,
  "metodo": "cartao_credito",
  "status": "pago",
  "dataPagamento": {
    "$date": "2026-03-13T15:00:00.000Z"
  },
  "detalhes": {
    "bandeira": "visa",
    "final": "1111",
    "parcelas": 2
  }
}
```

```json
{
  "_id": {
    "$oid": "a70000000000000000000002"
  },
  "reservaId": {
    "$oid": "a50000000000000000000002"
  },
  "hospedagemId": {
    "$oid": "a60000000000000000000002"
  },
  "hospedeId": {
    "$oid": "a10000000000000000000002"
  },
  "tipo": "total",
  "valorCentavos": 74000,
  "metodo": "pix",
  "status": "pago",
  "dataPagamento": {
    "$date": "2026-03-24T15:00:00.000Z"
  },
  "detalhes": {
    "txid": "NEBULA0002ABC"
  }
}
```

Estorno da reserva cancelada: tem `status: "reembolsado"`, por isso não entra no faturamento, e o `detalhes` ganha o campo extra `reembolsadoEm` (já o sinal pago de uma reserva futura entra no mês em que foi recebido):
```json
{
  "_id": {
    "$oid": "a70000000000000000000010"
  },
  "reservaId": {
    "$oid": "a50000000000000000000010"
  },
  "hospedeId": {
    "$oid": "a10000000000000000000004"
  },
  "tipo": "sinal",
  "valorCentavos": 116000,
  "metodo": "cartao_credito",
  "status": "reembolsado",
  "dataPagamento": {
    "$date": "2026-04-05T15:00:00.000Z"
  },
  "detalhes": {
    "bandeira": "amex",
    "final": "3333",
    "parcelas": 1,
    "reembolsadoEm": {
      "$date": "2026-04-10T16:00:00.000Z"
    }
  }
}
```

### feedbacks (avaliação em coleção própria)
```json
{
  "_id": {
    "$oid": "a80000000000000000000001"
  },
  "hospedagemId": {
    "$oid": "a60000000000000000000001"
  },
  "hospedeId": {
    "$oid": "a10000000000000000000001"
  },
  "quartoId": {
    "$oid": "a20000000000000000000201"
  },
  "nota": 5,
  "categorias": {
    "limpeza": 5,
    "atendimento": 5,
    "conforto": 5
  },
  "comentario": "Vista incrível e atendimento impecável.",
  "data": {
    "$date": "2026-03-14T15:00:00.000Z"
  },
  "resposta": {
    "funcionarioId": {
      "$oid": "a30000000000000000000003"
    },
    "texto": "Obrigado, Mariana! Esperamos você de volta.",
    "data": {
      "$date": "2026-03-15T12:00:00.000Z"
    }
  }
}
```

### funcionarios e servicos
```json
{
  "_id": {
    "$oid": "a30000000000000000000001"
  },
  "nome": "Paulo Mendes",
  "cargo": "recepcionista",
  "turno": "manhã",
  "habilidades": [
    "inglês",
    "espanhol"
  ],
  "admissao": {
    "$date": "2023-05-10T18:00:00.000Z"
  },
  "ativo": true
}
```

```json
{
  "_id": {
    "$oid": "a40000000000000000000002"
  },
  "nome": "Spa",
  "categoria": "bem-estar",
  "precoCentavos": 22000,
  "unidade": "sessão"
}
```

