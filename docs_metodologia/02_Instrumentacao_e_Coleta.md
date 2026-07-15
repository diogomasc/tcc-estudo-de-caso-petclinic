# 02 — Instrumentação e Coleta de Dados

> **TCC:** Mitigação de Débito Técnico Estrutural — Spring PetClinic REST
> **Seção:** 4 Metodologia — Instrumentação e Coleta

---

## 1. Variáveis e Métricas do Experimento

Para garantir o rigor metodológico na avaliação do Débito Técnico Estrutural, o experimento foi desenhado com as seguintes variáveis:

### 1.1 Variáveis Independentes (Preditores)

São os indicadores que representam a degradação estrutural do código (inseridos na branch `intentional-code-smells` e removidos/otimizados na `refactoring-metrics-improvements`):

- **CBO (Coupling Between Object Classes):** Nível de acoplamento entre os componentes (ex: injeções de dependência excessivas).
- **LCOM (Lack of Cohesion in Methods):** Falta de coesão nos métodos (responsabilidades misturadas).
- **Violações PMD / ArchUnit:** Quebras de regras arquiteturais e más práticas estruturais (ex: métodos longos, quebra de camadas lógicas, consultas com N+1 implícito no Hibernate).

### 1.2 Variáveis Dependentes (Métricas de Observabilidade)

São as métricas de runtime avaliadas para verificar o impacto do débito técnico e a eficácia da refatoração:

- **Latência (p95 / p99):** Tempos de resposta no 95º e 99º percentis, medidos via K6 para refletir a experiência final do usuário.
- **Taxa de Erros (`rate`):** Proporção de requisições que falham (ex: status HTTP 5xx ou demoras que estouram timeouts).
- **Throughput Útil:** Medido pelos contadores de operações de escrita bem-sucedidas (ex: _owners criados com sucesso_), considerando a taxa constante de chegada.
- **Profundidade de Pilha (Trace Depth / Count):** Extraído via telemetria do Micrometer (@Observed) para evidenciar a sobrecarga da CPU, chamadas redundantes e complexidade de execução (ex: chamadas excessivas ao banco de dados dentro de loops).

---

## 2. Visão Geral da Pipeline

```
Spring Boot (JVM)
   └─ @Observed (Micrometer) ──────► Prometheus (scrape 5s) ──► Grafana (validação visual)
   └─ HTTP endpoints ──────────────► K6 (--network host)
                                         └─► --out csv  → k6-metrics-*.csv
                                         └─► handleSummary() → k6-summary-*.json
                                                  └─► Python / Pandas → análise estatística (Média e Desvio Padrão)
```

---

## 3. Instrumentação com @Observed (Micrometer)

> **Nota Metodológica:** Para garantir que a coleta de métricas de *runtime* fosse justa e comparável, toda a instrumentação via anotação `@Observed` (incluindo as dependências do pom.xml e classes de configuração) foi aplicada inicialmente na branch **baseline do projeto** (`refactoring-metrics`). Ou seja, a base de código original já foi tornada "observável" antes de sofrer qualquer injeção de anomalias (degradação), estabelecendo um parâmetro de controle íntegro e rastreável desde a estaca zero.

### 2.1 Dependências (`pom.xml`)

```xml
<!-- Actuator: expõe /actuator/prometheus -->
<dependency>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Micrometer Prometheus Registry -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>

<!-- AspectJ: habilita @Observed via AOP proxy -->
<dependency>
    <artifactId>spring-boot-starter-aspectj</artifactId>
</dependency>
```

### 2.2 Anotação @Observed — Camada Controller (6 pontos)

| contextualName                | Método instrumentado | Arquivo                        |
| ----------------------------- | -------------------- | ------------------------------ |
| `Controller_Owner_ListAll`    | `listOwners()`       | `OwnerRestController.java:98`  |
| `Controller_Owner_FindById`   | `getOwner()`         | `OwnerRestController.java:178` |
| `Controller_Owner_Add`        | `addOwner()`         | `OwnerRestController.java:189` |
| `Controller_Pet_AddToOwner`   | `addPetToOwner()`    | `OwnerRestController.java:230` |
| `Controller_Visit_AddToOwner` | `addVisitToOwner()`  | `OwnerRestController.java:306` |
| `Controller_Vet_ListAll`      | `listVets()`         | `VetRestController.java:59`    |

**Exemplo de uso em `OwnerRestController.java`:**

```java
@PreAuthorize("hasRole(@roles.OWNER_ADMIN)")
@Override
@Observed(name = "metodo.execucao", contextualName = "Controller_Owner_ListAll")
public ResponseEntity<List<OwnerDto>> listOwners(String lastName) {
    // ...
}
```

### 2.3 Anotação @Observed — Camada Service (7 pontos)

| contextualName             | Método instrumentado | Arquivo                      |
| -------------------------- | -------------------- | ---------------------------- |
| `Service_Owner_FindAll`    | `findAllOwners()`    | `ClinicServiceImpl.java:144` |
| `Service_Owner_FindById`   | `findOwnerById()`    | `ClinicServiceImpl.java:311` |
| `Service_Owner_Save`       | `saveOwner()`        | `ClinicServiceImpl.java:439` |
| `Service_Vet_FindAll`      | `findAllVets()`      | `ClinicServiceImpl.java:117` |
| `Service_PetType_FindById` | `findPetTypeById()`  | `ClinicServiceImpl.java:245` |
| `Service_Pet_Save`         | `savePet()`          | `ClinicServiceImpl.java:332` |
| `Service_Visit_Save`       | `saveVisit()`        | `ClinicServiceImpl.java:417` |

**Exemplo em `ClinicServiceImpl.java`:**

```java
@Override
@Transactional(readOnly = true)
@Observed(name = "metodo.execucao", contextualName = "Service_Owner_FindAll")
public Collection<Owner> findAllOwners() throws DataAccessException {
    // ...
}
```

### 2.4 Métricas Prometheus Geradas

Cada `@Observed` gera automaticamente:

| Métrica Prometheus                                                | Tipo      | Descrição                 |
| ----------------------------------------------------------------- | --------- | ------------------------- |
| `metodo_execucao_seconds_count{observation_contextualName="..."}` | Counter   | Total de invocações       |
| `metodo_execucao_seconds_sum{observation_contextualName="..."}`   | Gauge     | Soma dos tempos (s)       |
| `metodo_execucao_seconds_bucket{...}`                             | Histogram | Distribuição de latências |

**PromQL para p95 de um método:**

```promql
histogram_quantile(0.95,
  rate(metodo_execucao_seconds_bucket{
    observation_contextualName="Service_Owner_FindAll"
  }[2m])
)
```

---

## 4. Raspagem do Prometheus

**Configuração em `infra/prometheus.yml`:**

```yaml
- job_name: "spring-petclinic-rest"
  metrics_path: "/petclinic/actuator/prometheus"
  scrape_interval: 5s
  static_configs:
    - targets: ["host.docker.internal:9966"]
      labels:
        app: "spring-petclinic-rest"
        fase: "baseline" # Altera-se para "pos-refatoracao" na Fase 2
```

- **Intervalo:** 5 segundos (resolução adequada para testes de 13 min)
- **Endpoint:** `/petclinic/actuator/prometheus` (porta 9966)
- **Label `fase`:** segmenta os dados no Grafana entre as duas fases experimentais

---

## 5. Geração de Carga com K6

### 5.1 Executor: `constant-arrival-rate` — Open-Loop Model

**Por que migrar de `ramping-vus` para `constant-arrival-rate`?**

O executor `ramping-vus` opera em **closed-loop**: o throughput global varia
diretamente com a latência da aplicação. Se a aplicação ficar lenta, os VUs
ficam ocupados por mais tempo e o número efetivo de requisições por segundo cai —
quebrando a premissa de "carga de trabalho constante" necessária para comparar
duas branches com comportamentos de latência distintos.

O `constant-arrival-rate` opera em **open-loop**: o k6 dispara iterações em
taxa fixa (`rate`) independentemente da latência do servidor. Quando a aplicação
fica lenta, o k6 aloca novos VUs para honrar a taxa — VUs em uso cresce, mas
`rate` permanece constante. Isso elimina o confound: se um endpoint aparecer
pior em uma branch, é degradação real — não artefato do closed-loop.

**Referência:** k6 Docs — _Constant Arrival Rate executor_ e _Dropped Iterations_
(grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate).

### 5.2 Configuração de Scenarios (`load-test.js`)

```javascript
export const options = {
  scenarios: {
    // Warm-up: ramping-vus é CORRETO aqui — VUs crescentes aquecem:
    //   a) JVM: JIT compilation dos métodos críticos
    //   b) Cache Spring: warm-up do ConcurrentMapCacheManager (petTypes, vets)
    warmup: {
      executor: "ramping-vus",
      startVUs: 0,
      stages: [{ duration: "30s", target: 30 }],
      gracefulRampDown: "5s",
      tags: { phase: "warmup" }, // EXCLUÍDO dos thresholds
    },

    // Teste principal: open-loop com taxa fixa
    steady_state: {
      executor: "constant-arrival-rate",
      startTime: "35s", // 30s warmup + 5s graceful ramp-down
      rate: 26, // 26 it/s — referência da branch mais lenta
      timeUnit: "1s",
      duration: "12m",
      preAllocatedVUs: 100, // ≈ rate × p99_boa (26 × 4s)
      maxVUs: 300, // ≥ rate × p99_pior (26 × 10s = 260)
      tags: { phase: "test" }, // INCLUÍDO nos thresholds
      gracefulStop: "10s",
    },
  },
};
```

### 5.3 Definições e Calibração dos Parâmetros (Metodologia)

As configurações do K6 foram delineadas para refletir isoladamente o impacto das anomalias no sistema, neutralizando gargalos da própria ferramenta de testes. As definições conceituais e empíricas utilizadas para o experimento são:

> **Virtual Users (VUs):** Instâncias de execução concorrentes. Cada VU atua de forma autônoma, simulando um usuário independente que executa o script em laço (loop).

> **Executor `ramping-vus` (Closed-Loop) em `stages` e `duration`:** Modelo no qual o volume de carga obedece à quantidade estrita de usuários simultâneos, escalonados em estágios de duração determinada (de 0 a 30 VUs em 30s).
> _Justificativa de uso:_ Foi estritamente restrito à **fase de warm-up (aquecimento)** do experimento. Essa alocação gradual é imperativa para acionar a Compilação _Just-In-Time_ (JIT) da JVM — que compila códigos acessados frequentemente para código de máquina — e para instanciar (popular) a infraestrutura de Cache-Aside (ex: lista de veterinários) antes da coleta oficial de dados.

> **Executor `constant-arrival-rate` (Open-Loop):** Modelo no qual as requisições são despachadas a uma taxa inflexível, independentemente do tempo de resposta (latência) da aplicação.
> _Justificativa de uso:_ Aplicado no cenário principal (`steady_state`). Em pesquisas comparativas de performance, é necessário garantir que a branch deteriorada receba matematicamente o mesmo estresse que a branch refatorada. Se houvesse dependência de resposta (closed-loop), a branch lenta "frearia" o envio do K6, invalidando a premissa fundamental de carga constante.

> **Rate e TimeUnit (`rate: 26`, `timeUnit: '1s'`):** Frequência de requisições desejada para o cenário. O teste injeta exatamente 26 fluxos completos por segundo.
> _Justificativa de uso:_ O valor fixado de 26 foi balizado empiricamente pela capacidade limite da branch degradada (`refactoring-metrics-intentional-code-smells`). Submeter uma frequência maior induziria ao estrangulamento precoce do k6 (_dropped iterations_), deturpando as observações de concorrência e banco de dados que devem ditar o gargalo, e não a saturação abstrata do gerador de carga.

> **MaxVUs (`maxVUs: 300`):** Teto absoluto de Instâncias (VUs) que o K6 está autorizado a alocar dinamicamente em picos de estrangulamento da API.
> _Justificativa de uso:_ Dimensionado através da heurística $MaxVUs \ge Rate \times Latência_{P99}$. Empregando o pior percentil de lentidão diagnosticado na fase preliminar (aproximadamente 10 segundos): $26 \times 10 = 260$. Um teto de 300 provê resiliência suficiente para absorver flutuações extremas de latência sem perdas forçadas no disparo.

> **PreAllocatedVUs (`preAllocatedVUs: 100`):** Alocação de memória prévia no Sistema Operacional para abrigar VUs inativos antes da ignição do teste, suprimindo o atraso de alocá-los "sob demanda".
> _Justificativa de uso:_ A inicialização de novas instâncias pela ferramenta de teste insere overhead. Baseado na equação supracitada utilizando a latência da branch refatorada (≈4s), tem-se: $26 \times 4s = 104$ VUs necessários em regime ideal. Pré-alocar 100 instâncias mitiga contaminações nos dados geradas puramente por custos operacionais da engine do k6 no início da coleta.

> **StartTime e GracefulRampDown:** Parâmetros de sequenciamento temporal. `startTime: '35s'` adia o início da coleta oficial, permitindo os 30s de _warm-up_ acrescidos de 5s estipulados para encerramento pacífico (`gracefulRampDown`), impedindo sobreposição e contaminação de transações entre as duas fases.

### 5.4 Thresholds com Tag de Fase

Os thresholds aplicam apenas ao cenário `steady_state` (tag `{phase:test}`),
excluindo matematicamente os dados do warm-up:

```javascript
thresholds: {
  // Threshold global
  "http_req_duration": ["p(95)<5000"],
  "taxa_erro":         ["rate<0.10"],

  // Por endpoint — filtrado pela tag phase:test
  "latencia_listar_owners{phase:test}":   ["p(95)<4000", "p(99)<5000"],
  "latencia_criar_owner{phase:test}":     ["p(95)<3000", "p(99)<4000"],
  "latencia_consultar_owner{phase:test}": ["p(95)<3000", "p(99)<4000"],
  "latencia_criar_pet{phase:test}":       ["p(95)<3000", "p(99)<4000"],
  "latencia_criar_visit{phase:test}":     ["p(95)<3000", "p(99)<4000"],
  "latencia_listar_vets{phase:test}":     ["p(95)<2000", "p(99)<3000"],
}
```

### 4.5 Interpretação dos Resultados com Carga Constante

Com `constant-arrival-rate`, a comparação entre branches é direta:

| Situação                   | Branch lenta (degradada)      | Branch rápida (pos-refactoring) |
| -------------------------- | ----------------------------- | ------------------------------- |
| Iterações/s (arrival rate) | **26** (fixo)                 | **26** (fixo)                   |
| VUs em uso                 | alto (acumula)                | baixo                           |
| `dropped_iterations`       | possível                      | 0                               |
| p95/p99 `GET /owners`      | alto (N+1)                    | baixo (JOIN FETCH)              |
| p95/p99 `POST /pets`       | médio (4 queries)             | baixo (cache+INSERT)            |
| p95/p99 `GET /vets`        | médio (EAGER N:M)             | baixo (cache hit)               |
| p95/p99 `POST /owners`     | **comparável entre branches** | **comparável**                  |

> `POST /owners` deve apresentar latência similar entre branches — confirma
> ausência do closed-loop confound. Se ainda aparecer pior numa branch, é
> regressão real, não artefato.

### 5.5 Endpoints Exercitados

Para exercitar as diferentes frentes do sistema e capturar as métricas dos `Controllers` e `Services`, o script K6 simula o comportamento real de uso e encadeia operações de leitura e escrita.

```javascript
// Grupo 1: GET /owners (Listagem - Pode sofrer com N+1)
group("GET /owners", () => {
  const res = http.get(`${BASE_URL}/owners`, HEADERS);
  latenciaListarOwners.add(res.timings.duration);
  erroListarOwners.add(res.status !== 200);
});

// Grupo 2: POST /owners (Escrita - Insere carga no banco)
group("POST /owners", () => {
  const payload = JSON.stringify({ firstName: faker.person.firstName(), ... });
  const res = http.post(`${BASE_URL}/owners`, payload, HEADERS);
  // Se criado com sucesso (201), captura o ID gerado para alimentar os próximos steps
});
// (A lógica se repete de forma encadeada para /owners/{id}, /pets, /visits, /vets e /actuator/health)
```

### 5.6 Exportação de Dados — CSV Granular + JSON Summary Estruturado

```bash
# Comando executado pelo run-benchmark.sh
docker run --rm --network host \
  --user "$(id -u):$(id -g)" \
  -e HOME=/tmp \
  -v "$INFRA_DIR/k6:/scripts:ro" \
  -v "$LOG_DIR:/results" \
  grafana/k6:latest run \
    --out "csv=/results/k6-metrics-${LABEL}-${TIMESTAMP}.csv" \
    /scripts/load-test.js
```

**`handleSummary()` — Summary JSON Estruturado:**

O `JSON.stringify(data)` original exportava apenas `thresholds.ok: boolean` —
sem os valores numéricos reais. A versão consolidada extrai diretamente:
`data.metrics["latencia_listar_owners"].values["p(95)"]` → número em ms.

```javascript
export function handleSummary(data) {
  // Extrai p95, p99, avg, min, max, med, p90 de um Trend
  function extractTrend(metricName) {
    const v = data.metrics[metricName]?.values;
    if (!v) return null;
    return {
      count: v.count,
      avg: parseFloat(v.avg?.toFixed(2)),
      p90: parseFloat(v["p(90)"]?.toFixed(2)),
      p95: parseFloat(v["p(95)"]?.toFixed(2)), // ← valor numérico direto
      p99: parseFloat(v["p(99)"]?.toFixed(2)), // ← valor numérico direto
      max: parseFloat(v.max?.toFixed(2)),
    };
  }

  const summaryJson = {
    meta: {
      gerado_em: new Date().toISOString(),
      executor: "constant-arrival-rate",
      rate_por_segundo: 26,
    },
    // p95 e p99 por endpoint — diretamente consumíveis pelo Python/Pandas
    resumo_por_endpoint: {
      "GET /owners": { latencia_ms: extractTrend("latencia_listar_owners") },
      "POST /owners": { latencia_ms: extractTrend("latencia_criar_owner") },
      // ...
    },
    global: {
      dropped_iterations: data.metrics["dropped_iterations"]?.values ?? {
        count: 0,
      },
    },
    raw: data, // objeto completo para auditoria
  };

  return {
    stdout: textSummary(data, { indent: " ", enableColors: true }),
    "/results/summary.json": JSON.stringify(summaryJson, null, 2),
  };
}
```

**Estrutura do `summary.json` gerado:**

```json
{
  "meta": {
    "gerado_em": "2026-07-13T02:00:00.000Z",
    "executor": "constant-arrival-rate",
    "rate_por_segundo": 26
  },
  "resumo_por_endpoint": {
    "GET /owners": {
      "latencia_ms": {
        "count": 18720,
        "avg": 312.45,
        "p90": 890.12,
        "p95": 1240.33,
        "p99": 3450.87,
        "max": 8901.23
      },
      "erro": { "rate": 0.0001 }
    },
    "POST /pets": {
      "latencia_ms": { "p95": 180.12, "p99": 420.55 },
      "erro": { "rate": 0.0 }
    }
  },
  "global": {
    "dropped_iterations": { "count": 0 },
    "counters": {
      "owners_criados_com_sucesso": 18720
    }
  },
  "threshold_status": {
    "GET /owners": { "p(95)<4000": true, "p(99)<5000": true }
  },
  "raw": { ... }
}
```

---

## 6. Análise Estatística (Python / Pandas)

A extração bruta do K6 gera milhares de pontos de dados. O tratamento com Python e a biblioteca Pandas é fundamental para agrupar esses resultados de forma estatisticamente rigorosa, removendo o ruído e extraindo a **Média** e o **Desvio Padrão (σ)** das latências (p95 e p99).

### 5.1 Leitura do Summary JSON Estruturado

```python
import json
import pandas as pd

# Leitura direta dos valores numéricos de p95/p99 por endpoint
with open("results/summary.json") as f:
    summary = json.load(f)

# Extrair p95 e p99 diretamente (sem parsing de thresholds)
rows = []
for endpoint, dados in summary["resumo_por_endpoint"].items():
    lat = dados.get("latencia_ms", {})
    if lat:
        rows.append({
            "endpoint": endpoint,
            "p95_ms":   lat.get("p95"),
            "p99_ms":   lat.get("p99"),
            "avg_ms":   lat.get("avg"),
            "count":    lat.get("count"),
        })

df = pd.DataFrame(rows)
print(df.to_string())
```

### 5.2 Leitura do CSV Granular (Séries Temporais)

```python
# CSV gerado por --out csv — 1 linha por data point (timestamp + valor)
df_raw = pd.read_csv("results/k6-metrics-baseline-20260713.csv",
                     parse_dates=["timestamp"])

# Filtrar apenas dados da fase "test" (excluir warmup)
df_test = df_raw[df_raw["tags"].str.contains("phase:test", na=False)]

# Calcular p95 e p99 por endpoint
per_endpoint = (df_test[df_test["metric_name"].str.startswith("latencia_")]
                .groupby("metric_name")["metric_value"]
                .quantile([0.95, 0.99])
                .unstack()
                .rename(columns={0.95: "p95_ms", 0.99: "p99_ms"}))

print(per_endpoint)
```

### 6.3 Baterias de Execução: Protocolo de 5 Rodadas

Para mitigar a variabilidade inerente ao hardware e oscilações momentâneas de rede e de SO (mesmo rodando com `--network host`), cada branch (baseline degradada e pos-refatoracao) é testada em uma bateria de **5 rodadas independentes (runs)**.

Cada execução contém a fase de **warm-up inicial de 30 segundos com 30 VUs progressivos (ramping-vus)**. Essa fase é metodologicamente essencial para aquecer a JVM (JIT Compilation) e popular os caches internos do Spring. Sem o warm-up (ou considerando seus dados nos relatórios finais), o overhead temporário de compilação de bytecode em tempo real distorceria severamente os percentis p95/p99 do regime estável. Os dados do warm-up são desconsiderados via tag `phase:test`.

Para consolidar as 5 rodadas em Python:

```python
# Coleta dos valores consolidados de p95 por execução (5 runs)
per_run = pd.DataFrame([
    {"run": i, "p95": runs[i]["resumo_por_endpoint"]["GET /owners"]["latencia_ms"]["p95"]}
    for i in range(5)
])
```

**Retirando a Média e o Desvio Padrão Consolidado:**

**Critério de estabilidade:**

- **σ < 10% da média**: alta reprodutibilidade → valor médio é confiável
- **σ > 20% da média**: variabilidade excessiva → investigar ambiente
- **`dropped_iterations > 0`**: aplicação saturada na taxa de 26 it/s — registrar como
  indicativo de limitação da branch, não erro de medição

**Cálculo do valor único para o TCC:**

```python
p95_mean = per_run['p95'].mean()
p95_std  = per_run['p95'].std(ddof=1)
print(f"p95 reportado: {p95_mean:.0f} ± {p95_std:.0f} ms")
```

---

## 7. Dropped Iterations — Indicador de Saturação

Com `constant-arrival-rate`, `dropped_iterations > 0` significa que o k6 não
conseguiu honrar a taxa de 26 it/s por falta de VUs disponíveis (todos
os 300 VUs estavam ocupados). Isso indica saturação da aplicação, não
configuração errada do teste.

| `dropped_iterations`  | Interpretação                                                        |
| --------------------- | -------------------------------------------------------------------- |
| 0                     | Aplicação saudável — honrou 26 it/s durante todo o teste             |
| > 0 (início)          | `preAllocatedVUs` insuficiente — aumentar                            |
| > 0 (progressivo)     | Aplicação degrada com o tempo (crescimento de dados, memory leak)    |
| > 0 (constante, alto) | Aplicação não consegue sustentar 26 it/s — documentar como limitação |

O valor de `dropped_iterations` é exportado em `summary.json > global > dropped_iterations`.

---

## Referências

- Grafana Labs. _K6 Documentation: constant-arrival-rate executor_.
  grafana.com/docs/k6/latest/using-k6/scenarios/executors/constant-arrival-rate
- Grafana Labs. _K6 Documentation: Dropped Iterations_.
  grafana.com/docs/k6/latest/using-k6/scenarios/concepts/dropped-iterations
- Grafana Labs. _K6 Documentation: handleSummary()_.
  grafana.com/docs/k6/latest/results-output/end-of-test/custom-summary
- Grafana Labs. _K6 Documentation: Arrival Rate VU Allocation_.
  grafana.com/docs/k6/latest/using-k6/scenarios/concepts/arrival-rate-vu-allocation
- Micrometer. _@Observed annotation_. micrometer.io/docs/observation
- Prometheus. _Scrape configuration_. prometheus.io/docs/prometheus/latest/configuration/
