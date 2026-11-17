# Parte 3 — Radar de Comando

Consultas principais do banco `hotel_nebula`. Execute no `mongosh` depois de
importar os arquivos JSON da pasta `dados/`.

## Convenções

- Valores financeiros são armazenados em centavos e divididos por 100 nos
  resultados.
- Datas são `Date` em UTC. Os agrupamentos usam o fuso de São Paulo.
- As reservas possuem snapshots de hóspede e quarto, evitando consultas extras.
- O `$unwind` aparece somente na consulta de serviços, pois é necessário para
  separar os itens do array `servicosConsumidos`.

## 1. Quartos por característica

### 1.1 Vista para o mar

```js
db.quartos.find(
  { caracteristicas: "vista para o mar" },
  { _id: 0, numero: 1, categoria: 1, precoDiariaCentavos: 1 }
)
```

Resultado:

```js
[
  { numero: 201, categoria: "luxo", precoDiariaCentavos: 65000 },
  { numero: 202, categoria: "luxo", precoDiariaCentavos: 58000 },
  { numero: 301, categoria: "suite presidencial", precoDiariaCentavos: 120000 }
]
```

### 1.2 Vista para o mar e banheira

```js
db.quartos.find(
  { caracteristicas: { $all: ["vista para o mar", "banheira"] } },
  { _id: 0, numero: 1, categoria: 1 }
)
```

Resultado:

```js
[
  { numero: 201, categoria: "luxo" },
  { numero: 301, categoria: "suite presidencial" }
]
```

### 1.3 Quartos de luxo

```js
db.quartos.find(
  { categoria: { $in: ["luxo", "suite presidencial"] } },
  { _id: 0, numero: 1, categoria: 1, precoDiariaCentavos: 1 }
).sort({ precoDiariaCentavos: 1 })
```

## 2. Hóspedes recorrentes

### 2.1 Mais de uma reserva

```js
db.reservas.aggregate([
  { $group: {
      _id: "$hospedeId",
      nome: { $first: "$hospede.nome" },
      totalReservas: { $sum: 1 }
  } },
  { $match: { totalReservas: { $gt: 1 } } },
  { $project: { _id: 0, nome: 1, totalReservas: 1 } },
  { $sort: { totalReservas: -1, nome: 1 } }
])
```

Resultado:

```js
[
  { nome: "Ana Beatriz Lima", totalReservas: 2 },
  { nome: "Carlos Ferreira", totalReservas: 2 },
  { nome: "John Smith", totalReservas: 2 },
  { nome: "Mariana Costa", totalReservas: 2 }
]
```

### 2.2 Mais de uma reserva sem cancelamentos

```js
db.reservas.aggregate([
  { $match: { status: { $ne: "cancelada" } } },
  { $group: {
      _id: "$hospedeId",
      nome: { $first: "$hospede.nome" },
      reservas: { $sum: 1 },
      noites: { $sum: "$noites" },
      gastoCentavos: { $sum: "$valorDiariasCentavos" }
  } },
  { $match: { reservas: { $gt: 1 } } },
  { $project: {
      _id: 0,
      nome: 1,
      reservas: 1,
      noites: 1,
      gasto: { $divide: ["$gastoCentavos", 100] }
  } },
  { $sort: { gasto: -1 } }
])
```

Resultado:

```js
[
  { nome: "Mariana Costa", reservas: 2, noites: 7, gasto: 6750 },
  { nome: "Ana Beatriz Lima", reservas: 2, noites: 5, gasto: 2300 },
  { nome: "Carlos Ferreira", reservas: 2, noites: 5, gasto: 1820 }
]
```

## 3. Faturamento

### 3.1 Faturamento em abril de 2026

```js
db.pagamentos.aggregate([
  { $match: {
      status: "pago",
      dataPagamento: {
        $gte: ISODate("2026-04-01T00:00:00-03:00"),
        $lt: ISODate("2026-05-01T00:00:00-03:00")
      }
  } },
  { $group: {
      _id: null,
      totalCentavos: { $sum: "$valorCentavos" },
      pagamentos: { $sum: 1 }
  } },
  { $project: {
      _id: 0,
      periodo: "2026-04",
      faturamento: { $divide: ["$totalCentavos", 100] },
      pagamentos: 1
  } }
])
```

Resultado: `{ periodo: "2026-04", faturamento: 8200, pagamentos: 3 }`.

### 3.2 Faturamento anual

```js
db.pagamentos.aggregate([
  { $match: {
      status: "pago",
      dataPagamento: {
        $gte: ISODate("2026-01-01T00:00:00-03:00"),
        $lt: ISODate("2027-01-01T00:00:00-03:00")
      }
  } },
  { $group: { _id: null, totalCentavos: { $sum: "$valorCentavos" } } },
  { $project: {
      _id: 0,
      ano: 2026,
      faturamento: { $divide: ["$totalCentavos", 100] }
  } }
])
```

Resultado: `{ ano: 2026, faturamento: 21210 }`.

### 3.3 Faturamento por método

```js
db.pagamentos.aggregate([
  { $match: { status: "pago" } },
  { $group: {
      _id: "$metodo",
      totalCentavos: { $sum: "$valorCentavos" },
      pagamentos: { $sum: 1 }
  } },
  { $project: {
      _id: 0,
      metodo: "$_id",
      faturamento: { $divide: ["$totalCentavos", 100] },
      pagamentos: 1
  } },
  { $sort: { faturamento: -1 } }
])
```

Resultado:

```js
[
  { metodo: "cartao_credito", faturamento: 14430, pagamentos: 4 },
  { metodo: "pix", faturamento: 4130, pagamentos: 3 },
  { metodo: "boleto", faturamento: 2090, pagamentos: 1 },
  { metodo: "dinheiro", faturamento: 560, pagamentos: 1 }
]
```

### 3.4 Receita de serviços

```js
db.hospedagens.aggregate([
  { $unwind: "$servicosConsumidos" },
  { $group: {
      _id: "$servicosConsumidos.nome",
      quantidade: { $sum: "$servicosConsumidos.quantidade" },
      totalCentavos: { $sum: {
        $multiply: [
          "$servicosConsumidos.quantidade",
          "$servicosConsumidos.valorUnitarioCentavos"
        ]
      } }
  } },
  { $project: {
      _id: 0,
      servico: "$_id",
      quantidade: 1,
      receita: { $divide: ["$totalCentavos", 100] }
  } },
  { $sort: { receita: -1 } }
])
```

## 4. Reservas e períodos movimentados

### 4.1 Reservas por mês

```js
db.reservas.aggregate([
  { $match: {
      status: { $ne: "cancelada" },
      dataCheckin: {
        $gte: ISODate("2026-01-01T00:00:00-03:00"),
        $lt: ISODate("2027-01-01T00:00:00-03:00")
      }
  } },
  { $group: {
      _id: { $month: {
        date: "$dataCheckin",
        timezone: "America/Sao_Paulo"
      } },
      reservas: { $sum: 1 },
      noitesReservadas: { $sum: "$noites" }
  } },
  { $project: {
      _id: 0,
      mes: "$_id",
      reservas: 1,
      noitesReservadas: 1
  } },
  { $sort: { reservas: -1, mes: 1 } }
])
```

Resultado:

```js
[
  { mes: 4, reservas: 3, noitesReservadas: 9 },
  { mes: 3, reservas: 2, noitesReservadas: 5 },
  { mes: 7, reservas: 2, noitesReservadas: 9 },
  { mes: 5, reservas: 1, noitesReservadas: 3 },
  { mes: 12, reservas: 1, noitesReservadas: 5 }
]
```

### 4.2 Reservas por dia da semana

```js
db.reservas.aggregate([
  { $match: { status: { $ne: "cancelada" } } },
  { $group: {
      _id: { $dayOfWeek: {
        date: "$dataCheckin",
        timezone: "America/Sao_Paulo"
      } },
      reservas: { $sum: 1 }
  } },
  { $project: {
      _id: 0,
      dia: { $arrayElemAt: [
        ["domingo", "segunda", "terça", "quarta", "quinta", "sexta", "sábado"],
        { $subtract: ["$_id", 1] }
      ] },
      reservas: 1
  } },
  { $sort: { reservas: -1 } }
])
```

Resultado: domingo (3), segunda (2), sábado (2), terça (1) e sexta (1).

