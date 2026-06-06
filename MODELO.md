# Remuneração / Expenses-Acc — Modelo e Cálculos

Documentação do app de acompanhamento da remuneração (single-file `index.html`, dados em `localStorage` + export/import JSON). Serve de mapa para futuros ajustes: cada conceito remete para a constante ou função onde se mexe.

-----

## 1. Estrutura de uma revisão

Cada entrada do array de dados (`compensation-data.json`) representa **uma revisão salarial** numa data:

```json
{
  "date": "2025-12",            // AAAA-MM (mês da revisão)
  "level": "M",                  // código do nível (ver LEVELS)
  "banding": "Talent Priority",  // texto livre, opcional
  "baseMonthly": 3165,           // salário base mensal (€)

  "fixed": {                     // valores ANUAIS (€)
    "lunch": 1847,               // Lunch Allowance      → Allowances
    "car": 7800,                 // Car Allowance        → Allowances
    "comms": 720,                // Plafond Telemóvel    → Benefícios Fixos
    "health": 598.92,            // Seguro de saúde      → Benefícios Fixos (Seguros)
    "healthFamily": 520.1        // Seguro saúde família → Benefícios Fixos (Seguros)
  },

  "variable": {                  // prémios desta revisão (€); chaves de VAR_FIELDS
    "localVariable": 3390,
    "indivPerf": 6780,
    "globalAnnual": 3830
  },

  "benefits": [                  // Benefícios Flex (convertidos dos prémios)
    { "type": "leasing", "amount": 4000 }
  ],
  "expenseBenefits": [           // conversões do Expenses Allowance
    { "type": "galp", "amount": 1800 }
  ],

  "plafond": 3000,               // Expenses Allowance (valor anual, €)
  "stockPlanPct": 10             // % alocada ao Plano de Ações (PAAT)
}
```

> **Migração automática (`normalize`)**: o antigo campo `fixed.paat` é descartado (o plano passou a ser calculado); `stockPlanPct` assume **10%** quando não existe; `benefits` e `expenseBenefits` são garantidos como arrays. Os dados antigos carregam sem alterações manuais.

-----

## 2. Categorias de remuneração

|Categoria                 |O que inclui                                             |Conta no total? |
|--------------------------|---------------------------------------------------------|----------------|
|**Salário base**          |`baseMonthly × 14`                                       |Sim             |
|**Allowances**            |Lunch Allowance + Car Allowance                          |Sim             |
|**Benefícios Fixos**      |Plafond Telemóvel + Seguros + **ganho** do Plano de Ações|Sim             |
|**Variável**              |Soma dos prémios da revisão                              |Sim             |
|**Expenses Allowance**    |`plafond` (~3.000 €)                                     |Sim             |
|**Benefícios Flex**       |Convertidos dos prémios (leasing, cheques-creche…)       |Não — realocação|
|**Conversões de despesas**|Do Expenses Allowance (Galp Frota, Faturas Restauração…) |Não — realocação|

**Porque é que Flex e conversões não somam:** são apenas uma forma diferente de receber dinheiro que **já está contado** (os prémios e o Expenses Allowance). Somá-los seria contar duas vezes. Aparecem no detalhe a título informativo — e entram no cálculo do Plano de Ações (ver abaixo).

`14 meses` = 12 + férias + Natal → constante `MONTHS_PER_YEAR`.

-----

## 3. Plano de Ações (PAAT) — a parte com regras

Descontas **10%** de tudo o que recebes **em conta** e compras ações Accenture a um preço com **15% de desconto**. O único ganho real é esse desconto.

```
Base   = ordenado + prémios + Car Allowance + Expenses Allowance
         − Benefícios Flex − conversões de despesas
Contribuição = Base × 10%        (stockPlanPct)
Ganho        = Contribuição × 15% (STOCK_DISCOUNT)   ← é só ISTO que entra no total
```

A lógica de “em conta”: tudo o que conviertes em qualquer benefício deixa de ser recebido em conta, por isso **sai da base**. Exemplo: dos 7.800 € de Car Allowance, alocas 4.000 € ao leasing → o Car entra inteiro mas o leasing sai, sobrando 3.800 € líquidos a contar.

|Passo             |Função                             |
|------------------|-----------------------------------|
|Base de cálculo   |`stockPlanBase`                    |
|Contribuição (10%)|`stockPlanContribution`            |
|Ganho (15%)       |`stockPlanBenefit` ← entra no total|


> **Nota:** o **Lunch Allowance não entra** na base do plano (cartão refeição, não é recebido em conta). Se algum dia mudar, é em `stockPlanBase`.

-----

## 4. Total Compensation

```
Total = Salário base anual          (annualBase)
      + Allowances                  (allowanceTotal: lunch + car)
      + Benefícios Fixos            (fixedBenefTotal: comms + seguros + ganho 15%)
      + Variável                    (varTotal)
      + Expenses Allowance          (plafond)
```

Função: `totalComp` (= `basePlusOther` + `plafond`).
No resumo anual, base/allowances/benefícios fixos/Expenses usam a **última revisão do ano**; o variável é a **soma do ano**.

-----

## 5. Onde mexer — receitas rápidas

|Quero…                                               |Mexer em                                                                                            |
|-----------------------------------------------------|----------------------------------------------------------------------------------------------------|
|Mudar a % do plano para uma revisão                  |campo `stockPlanPct` dessa revisão (form: “% alocada”)                                              |
|Mudar a % por defeito de novas revisões              |`STOCK_PLAN_DEFAULT_PCT`                                                                            |
|Mudar o desconto das ações                           |`STOCK_DISCOUNT` (0.15)                                                                             |
|Tirar/pôr o Car ou o Expenses na base do plano       |função `stockPlanBase`                                                                              |
|Acrescentar um tipo de prémio                        |array `VAR_FIELDS`                                                                                  |
|Acrescentar um tipo de Benefício Flex                |array `FLEX_TYPES`                                                                                  |
|Acrescentar um tipo de conversão de despesas         |array `EXPENSE_TYPES`                                                                               |
|Acrescentar/renomear um nível                        |array `LEVELS`                                                                                      |
|Mudar nº de meses do salário                         |`MONTHS_PER_YEAR`                                                                                   |
|Alterar o que entra em Allowances vs Benefícios Fixos|`ALLOWANCE_FIELDS` / `BENEFIT_FIELDS`                                                               |
|Afinar o nível nos gráficos (cor/tamanho/posição)    |`levelAxisPlugin` (cor `--acc`, `10px`, offset `bar.y+7`); `AXIS_PAD` mantém eixo e plugin alinhados|

Os labels visíveis estão nos próprios arrays (ex.: `['car','Car Allowance']`), por isso renomear é trocar a segunda posição.

-----

## 6. Mapa de funções

|Função                    |Devolve                                    |
|--------------------------|-------------------------------------------|
|`annualBase(r)`           |salário base × 14                          |
|`allowanceTotal(r)`       |lunch + car                                |
|`insurancesTotal(r)`      |health + healthFamily (Seguros)            |
|`varTotal(r)`             |soma dos prémios                           |
|`flexTotal(r)`            |soma dos Benefícios Flex                   |
|`expenseBenefitsTotal(r)` |soma das conversões de despesas            |
|`stockPlanBase(r)`        |base do plano (em conta − benefícios)      |
|`stockPlanContribution(r)`|base × %                                   |
|`stockPlanBenefit(r)`     |contribuição × 15% (**ganho**)             |
|`fixedBenefManual(r)`     |comms + seguros                            |
|`fixedBenefTotal(r)`      |comms + seguros + ganho do plano           |
|`basePlusOther(r)`        |base + allowances + benef. fixos + variável|
|`totalComp(r)`            |basePlusOther + Expenses Allowance         |

As variantes com sufixo `F` (`allowanceTotalF`, `varTotalF`, `byYearF`…) são as mesmas contas mas a **respeitar os filtros** dos gráficos (itens excluídos via chips na vista Evolução).

-----

## 7. Vistas e persistência

- **Vistas:** Resumo · Evolução (gráficos + filtros) · Revisões · Nova · Backup.
- **Nível nos gráficos:** na Evolução, o código do nível (`LEVELS`) aparece por baixo da data/ano no eixo da esquerda. A vista **mensal** usa o nível de cada revisão; a **anual** usa o da última revisão do ano (mesma regra do resto do agregado anual). Desenhado pelo `levelAxisPlugin`, irmão do `barTotalsPlugin` (que põe o total à direita).
- **Dados:** vivem só no dispositivo (`localStorage`, chave `salario_accenture_v1`). O HTML não contém dados.
- **Backup:** exportar/importar JSON na vista Backup. Convém exportar com regularidade.
- **`normalize()`** corre ao carregar e ao guardar — é o sítio certo para futuras migrações de formato.
- **Catálogo de tipos** (não apagar valores antigos para não perder histórico): `VAR_FIELDS`, `FLEX_TYPES`, `EXPENSE_TYPES`.