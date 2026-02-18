# Alice — Consolidação de Transações por Janela de Tempo

Solução para o desafio técnico de consolidação de transações financeiras, implementada em **TLPP** (linguagem nativa do Protheus/TOTVS).

---

## Sobre o Desafio

A API recebe uma lista de transações financeiras desordenadas e as consolida por usuário, agrupando transações que ocorram com diferença de **até 5 minutos** entre si em um único grupo.

---

## Como Testar

Compile o fonte `DesafioAlice.tlpp` no ambiente Protheus e acesse o endpoint abaixo via **Postman** ou **Insomnia**:

### Endpoint

```
POST {{RestUrl}}/api/alice/consolidar
Content-Type: application/json
```

### Exemplo de Requisição

```json
[
  { "id": "t01", "userId": 1, "amount": 100,  "timestamp": "2024-01-10T10:00:00Z" },
  { "id": "t02", "userId": 1, "amount": 50,   "timestamp": "2024-01-10T10:03:00Z" },
  { "id": "t03", "userId": 1, "amount": -20,  "timestamp": "2024-01-10T10:04:30Z" },
  { "id": "t04", "userId": 1, "amount": 80,   "timestamp": "2024-01-10T10:15:00Z" },
  { "id": "t05", "userId": 2, "amount": 200,  "timestamp": "2024-01-10T09:50:00Z" },
  { "id": "t06", "userId": 2, "amount": -50,  "timestamp": "2024-01-10T09:54:00Z" },
  { "id": "t07", "userId": 2, "amount": 30,   "timestamp": "2024-01-10T10:10:00Z" }
]
```

### Saída Esperada

```json
[
  { "userId": 1, "inicio": "2024-01-10T10:00:00Z", "fim": "2024-01-10T10:04:30Z", "total": 130 },
  { "userId": 1, "inicio": "2024-01-10T10:15:00Z", "fim": "2024-01-10T10:15:00Z", "total": 80  },
  { "userId": 2, "inicio": "2024-01-10T09:50:00Z", "fim": "2024-01-10T09:54:00Z", "total": 150 },
  { "userId": 2, "inicio": "2024-01-10T10:10:00Z", "fim": "2024-01-10T10:10:00Z", "total": 30  }
]
```

> **Dica:** Você pode importar a requisição diretamente no **Postman** ou **Insomnia** usando a URL acima com o body em JSON.

---

## Abordagem e Lógica da Solução

A solução foi dividida em três etapas principais:

### 1. Ordenação — `OrdenarTransacoes`
Como a lista de entrada não vem ordenada, o primeiro passo é ordenar as transações por `userId` (crescente) e, dentro do mesmo usuário, por `timestamp` (crescente).

Para garantir performance com grandes volumes de dados, foi utilizado o **`ASort` nativo do TLPP**, que internamente usa **QuickSort — O(n log n)** — em vez de um Bubble Sort manual que seria O(n²).

A comparação de timestamps é feita diretamente como **string**, o que é válido e eficiente para o formato ISO-8601, que é lexicograficamente ordenável.

### 2. Agrupamento — `ProcessarGrupos`
Com as transações ordenadas, o algoritmo percorre a lista uma única vez (**O(n)**) mantendo o estado do grupo atual:

- Se a transação pertence ao **mesmo usuário** e a diferença de tempo para a última transação do grupo é **≤ 300 segundos (5 min)** → acumula no grupo atual.
- Caso contrário → **fecha o grupo atual** e inicia um novo.

A diferença de tempo é calculada pela função `TsDiffSeg`, que converte timestamps ISO-8601 em segundos totais para comparação numérica precisa.

### 3. Serialização — resposta JSON
Os grupos consolidados são serializados manualmente em JSON string e devolvidos via `oRest:setResponse`.

---

## 📁 Funções Principais

| Função | Responsabilidade |
|---|---|
| `ConsolidarTransacoes` | Ponto de entrada da API. Orquestra leitura, ordenação, agrupamento e resposta. |
| `OrdenarTransacoes` | Ordena as transações por `userId` e `timestamp` usando `ASort` nativo (QuickSort). |
| `ProcessarGrupos` | Percorre a lista ordenada e forma os grupos por janela de 5 minutos. |
| `AddGrupo` | Cria o objeto JSON de um grupo consolidado e adiciona ao array de resultado. |
| `TsDiffSeg` | Calcula a diferença em segundos entre dois timestamps ISO-8601. |
| `TsParaSeg` | Converte um timestamp ISO-8601 em total de segundos para comparação numérica. |

---

## ⚠️ Casos de Borda Tratados

- `amount` pode ser **negativo** — somado normalmente ao total do grupo.
- Usuário com **transação única** — gera um grupo com `inicio == fim`.
- **Múltiplos usuários** com timestamps intercalados — a ordenação prévia garante que cada usuário seja processado de forma contínua e correta.
- Timestamps em **dias e meses diferentes** — `TsParaSeg` considera ano, mês e dia no cálculo.

---

## 🗂️ Estrutura do Código

```
ConsolidarTransacoes.prw
│
├── ConsolidarTransacoes()   ← @Post — entrada da API
├── OrdenarTransacoes()      ← Ordenação com ASort (QuickSort)
├── ProcessarGrupos()        ← Lógica de janela deslizante
├── AddGrupo()               ← Monta e armazena cada grupo
├── TsDiffSeg()              ← Diferença entre dois timestamps
└── TsParaSeg()              ← Converte ISO-8601 para segundos
```
