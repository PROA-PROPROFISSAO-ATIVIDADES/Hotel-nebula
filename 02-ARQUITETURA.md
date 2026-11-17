# Parte 2 — Arquitetura adotada

**Banco:** MongoDB (Atlas ou local), database `hotel_nebula`, com 8 coleções. Os documentos estão exportados em `dados/*.json` (Extended JSON: `$oid` = ObjectId, `$date` = Date) e são importados pelo Compass ou pelo `mongoimport`.

## Como a base foi montada
1. **Coleções e documentos:** um arquivo JSON por coleção em `dados/`. Importar na ordem de dependência: `hospedes`, `quartos`, `funcionarios`, `servicos` → `reservas` → `hospedagens` → `pagamentos` → `feedbacks`.
2. **Consistência entre registros:** os valores derivados batem com a origem. `noites` = dias entre check-in e check-out; `valorDiariasCentavos` = `noites` × `precoDiariaCentavos`; `totalServicosCentavos` = soma de `quantidade` × `valorUnitarioCentavos`; `valorCentavos` do pagamento total = diárias + serviços. Todas as referências (`hospedeId`, `quartoId`, `reservaId`...) apontam para documentos que existem.
3. **Índices:** criados depois da importação, cada um ligado a uma consulta (seção "Índices").
4. **Validação mínima (opcional):** `$jsonSchema` em `hospedes`, `quartos`, `reservas` e `pagamentos`, exigindo só os campos indispensáveis e os tipos essenciais (`objectId`, `date`, `number`). O resto fica livre.

## Tipos escolhidos
| Dado | Tipo | Por quê |
|---|---|---|
| `_id` e referências | `ObjectId` | Padrão do Mongo. Consultar com string não encontra nada: é preciso converter para `ObjectId` |
| Datas | `Date` (UTC) | Ordena e filtra por intervalo corretamente; texto como `"19/05/1982"` não ordena nem filtra por período |
| Dinheiro | inteiro em **centavos** (`*Centavos`) | Soma exata; `double` acumula erro de arredondamento |
| Nomes de campo | camelCase sem acento | Um padrão só, sem `preco` vs `preço` |

## Uso intencional de arrays e subdocumentos
| Estrutura | Onde | Por quê |
|---|---|---|
| Array de strings | `quartos.caracteristicas`, `hospedes.preferencias`, `funcionarios.habilidades` | Busca com `$in` / `$all` e índice multikey |
| Array de subdocumentos | `reservas.historicoStatus`, `hospedagens.servicosConsumidos` | Histórico e itens de consumo vivem e morrem com o documento-pai |
| Subdocumento | `hospedes.documento`, `reservas.hospede` e `reservas.quarto` (snapshots), `feedbacks.categorias` | Agrupa campos que sempre andam juntos |

## Snapshots (o que é copiado e por quê)
- `reservas.precoDiariaCentavos` e `valorDiariasCentavos`: o preço combinado não muda se o quarto mudar de tarifa.
- `reservas.hospede` e `reservas.quarto`: listar reservas sem buscar em outras coleções.
- `hospedagens.servicosConsumidos[].nome` e `valorUnitarioCentavos`: o histórico não muda se o catálogo mudar.

## Dados variáveis sem quebrar a estrutura
- `hospedes`: `endereco`, `preferencias` e `fidelidade` existem só em alguns documentos; o documento pode ser CPF ou passaporte.
- `pagamentos.detalhes`: cartão (bandeira/final/parcelas), PIX (txid), boleto (código/vencimento), dinheiro (vazio), estorno (acrescenta `reembolsadoEm`).
- `feedbacks`: `categorias` e `resposta` são opcionais.
- `hospedagens.servicosConsumidos` pode ser `[]`.

## Índices e as consultas que eles atendem
Regra dos índices compostos: **igualdade → ordenação → intervalo**. Um índice `{ a, b }` também atende consultas só por `a`, mas não só por `b`.

| Índice | Consulta |
|---|---|
| `hospedes { email } unique` | Cadastro sem e-mail duplicado; login |
| `quartos { caracteristicas }` (multikey) | 1.1, 1.2, 1.4: quartos por característica |
| `quartos { categoria }` | 1.3: quartos por categoria |
| `reservas { hospedeId, dataCheckin: -1 }` | Reservas de um hóspede, mais recentes primeiro; apoio à 2.1/2.2 |
| `reservas { quartoId, dataCheckin }` | Busca de conflito de datas ao criar uma reserva (igualdade no quarto + intervalo na data) |
| `reservas { dataCheckin }` | 4.1/4.2: ocupação em um período |
| `hospedagens { reservaId } unique` | Integridade: no máximo uma hospedagem por reserva |
| `pagamentos { status, dataPagamento }` | 3.1 a 3.4: `status = "pago"` (igualdade) + intervalo de datas |
| `feedbacks { hospedagemId } unique` | Integridade: no máximo um feedback por hospedagem |
| `feedbacks { quartoId, data: -1 }` | 5.2: últimas avaliações de um quarto |

Comandos para criar (no `mongosh`, com `use hotel_nebula`; no Compass, aba **Indexes → Create Index**):

```js
db.hospedes.createIndex({ email: 1 }, { unique: true })
db.quartos.createIndex({ caracteristicas: 1 })
db.quartos.createIndex({ categoria: 1 })
db.reservas.createIndex({ hospedeId: 1, dataCheckin: -1 })
db.reservas.createIndex({ quartoId: 1, dataCheckin: 1 })
db.reservas.createIndex({ dataCheckin: 1 })
db.hospedagens.createIndex({ reservaId: 1 }, { unique: true })
db.pagamentos.createIndex({ status: 1, dataPagamento: 1 })
db.feedbacks.createIndex({ hospedagemId: 1 }, { unique: true })
db.feedbacks.createIndex({ quartoId: 1, data: -1 })
```

Não há índice em `hospedagens.servicosConsumidos` nem em `status` de reservas: nenhuma consulta de gestão filtra seletivamente por eles, e cada índice a mais custa tempo de escrita e espaço.

## Concorrência: evitar overbooking
Reservar um quarto é "ler se está livre e depois gravar", e dois hóspedes podem fazer isso ao mesmo tempo. Uma operação em **um** documento é atômica no MongoDB, mas a checagem de conflito envolve vários documentos. A solução proposta é uma **transação** em que toda tentativa grava no documento do quarto (`$inc` em `versaoReservas`). Duas transações concorrentes no mesmo quarto escrevem no mesmo documento, o MongoDB aborta uma delas e o `withTransaction` a repete; na repetição a checagem já enxerga a reserva vencedora e recusa a nova. Exige replica set (o Atlas já é). Check-in às 15h no mesmo dia do checkout (12h) é permitido.

## Escalabilidade
- Nenhum array cresce sem limite: avaliações e pagamentos ficam em coleções próprias.
- `reservas` e `pagamentos` podem ser particionados (sharding) por data ou por `hospedeId` se o volume crescer.
- Listagens grandes devem usar paginação por cursor (`_id` ou data maior que o último item) em vez de `skip`.

## Casos de teste incluídos nos dados
- Reserva **cancelada** com pagamento **reembolsado** (não conta no faturamento nem na ocupação).
- Reserva **futura** com apenas **sinal** pago (sem hospedagem).
- Hospedagens **sem serviços**.
- Hóspedes **recorrentes** (Mariana, Carlos, Ana Beatriz) e um que cancelou (John Smith).
- Check-in de **15h (horário de São Paulo)** guardado como 18:00Z: os agrupamentos usam o fuso `America/Sao_Paulo`.
