# 05 — Execução dos Testes e Relacionamentos

> **TCC:** Mitigação de Débito Técnico Estrutural — Spring PetClinic REST
> **Seção:** 4 Metodologia — Execução dos Testes

---

## 1. Infraestrutura de Execução

### 1.1 Topologia do Ambiente

```
Host (Linux — bare metal / VM)
├── Spring Boot (JVM — porta 9966)        ← API sob teste
│     └─ Tomcat embedded
│     └─ H2 in-memory (perfil: h2)
└── Docker Compose (infra/docker-compose.yml)
      ├── Prometheus (porta 9090)
      ├── Grafana    (porta 3000)
      └── K6         (--network host)
```

**Escolha do H2 in-memory:** isolamento total do estado do banco entre execuções (sem dados residuais de runs anteriores). A cada `run-benchmark.sh`, o banco é reiniciado junto com a JVM — garantindo reprodutibilidade.

### 1.2 Configuração da API (`api/docker-compose.yml`)

```yaml
services:
  petclinic-api:
    build:
      context: .
      dockerfile: Dockerfile
    ports:
      - "9966:9966"
    environment:
      SPRING_PROFILES_ACTIVE: "h2,spring-data-jpa"
      JAVA_TOOL_OPTIONS: >-
        -Xms512m -Xmx1g
        -XX:+UseG1GC
        -XX:MaxGCPauseMillis=200
```

**JVM flags:** G1GC com pausa máxima de 200 ms evita GC stalls que contaminariam as medições de latência.

### 1.3 Dockerfile — Build Determinístico

```dockerfile
FROM eclipse-temurin:17-jdk-alpine AS builder
WORKDIR /app
COPY .mvn/ .mvn/
COPY mvnw pom.xml ./
RUN ./mvnw dependency:go-offline -B
COPY src ./src
RUN ./mvnw package -DskipTests -B

FROM eclipse-temurin:17-jre-alpine
COPY --from=builder /app/target/*.jar app.jar
EXPOSE 9966
ENTRYPOINT ["java", "-jar", "app.jar"]
```

Multi-stage build: separa compilação (JDK) de execução (JRE), garantindo imagem final < 200 MB e build reprodutível.

---

## 2. Protocolo de Warm-up da JVM

A JVM HotSpot realiza compilação JIT incremental: os primeiros ~1000–5000 invocações de cada método são interpretados. Medir latência antes do JIT estar aquecido contamina os resultados com ruído de interpretação.

### 2.1 Fase de Warm-up no K6

```javascript
export const options = {
  scenarios: {
    // Fase 1: warm-up JVM (excluída dos thresholds e análise final)
    warmup: {
      executor: "ramping-vus",
      startVUs: 0,
      stages: [{ duration: "30s", target: 30 }],
      gracefulRampDown: "5s",
      tags: { phase: "warmup" }, // ← tag diferencia nos dados
    },

    // Fase 2: medições efetivas com carga constante
    steady_state: {
      executor: "constant-arrival-rate",
      startTime: "35s",  // Inicia rigorosamente após o warm-up
      rate: 26,          // Taxa nivelada pela branch mais lenta
      timeUnit: "1s",
      duration: "12m",
      preAllocatedVUs: 100,
      maxVUs: 300,
      tags: { phase: "test" }, // ← incluída nos thresholds
      gracefulStop: "10s",
    },
  },
};
```

**Duração total:** ~13 min por execução (35s warm-up + 12min steady_state + overhead).

### 2.2 Por que 30s de warm-up é suficiente?

- Com 30 VUs e endpoints simples (CRUD), os 30 segundos geram ~900 requisições.
- Métodos com CC baixo (como `saveOwner`) atingem o limiar de compilação JIT (~1000 invocações) dentro deste período.
- O `latencia_health` (`GET /actuator/health`) serve como proxy para monitorar quando a JVM estabiliza.

---

## 3. Protocolo de Múltiplas Execuções (N=5)

### 3.1 Script de Execução (`infra/scripts/run-benchmark.sh`)

```bash
# Uso:
bash infra/scripts/run-benchmark.sh baseline       # Fase 1
bash infra/scripts/run-benchmark.sh pos-refatoracao # Fase 2

# Fluxo interno por execução:
[1/5] docker compose down -v    # Limpa volumes Prometheus (estado zero)
[2/5] docker compose up -d      # Sobe Prometheus + Grafana
[3/5] ./mvnw spring-boot:run    # Inicia Spring Boot (aguarda /actuator/health)
[4/5] docker run k6 run \       # Executa K6
        --out csv=/results/k6-metrics-${LABEL}-${TIMESTAMP}.csv \
        /scripts/load-test.js
[5/5] docker compose down       # Derruba stack
```

### 3.2 Isolamento Entre Runs

Cada execução garante:

- **Banco zerado:** H2 em memória é recriado pela JVM a cada restart
- **Prometheus zerado:** `docker compose down -v` remove o volume de dados TSDB
- **Porta liberada:** script mata processos em 9966 antes de subir
- **Timestamp único:** evita sobreescrita de arquivos entre runs

---

## 4. Configuração de Carga e Virtual Users (VUs)

| Fase | Parâmetro K6 | Valor | Justificativa |
| --- | --- | --- | --- |
| **warm-up** | executor | `ramping-vus` | Escalonamento de 0 a 30 VUs é suficiente para acionar a compilação JIT gradualmente sem sobrecarga térmica precoce. |
| **steady_state** | executor | `constant-arrival-rate` | Taxa fixa garante isonomia de carga entre branches (a latência não freará os disparos). |
| **steady_state** | rate | `26` | Limite empírico de requisições por segundo da branch degradada; previne acúmulo irreal (`dropped_iterations`) por estrangulamento da ferramenta. |
| **steady_state** | maxVUs | `300` | Margem de segurança dimensionada pela regra $MaxVUs \ge Rate \times p99$, capaz de absorver surtos anômalos de até ~11s de latência. |

O modelo adotado substituiu antigas variações por *stages* ("spike") porque, num modelo *open-loop*, o pico de estresse natural se manifesta através do acúmulo automático de instâncias (VUs ativas) caso o sistema demonstre estrangulamento perante os rigorosos 26 acessos paralelos por segundo.

---

## 5. Diagramas de Relacionamento — Endpoint × Cadeia de Chamadas

### 5.1 GET /api/owners — Cadeia Completa

```mermaid
flowchart TD
    K6["K6\nGET /api/owners\n(grupo: 'GET /owners')"] --> CTRL["OwnerRestController\n.listOwners()\n@Observed: Controller_Owner_ListAll"]

    CTRL -->|"findAllOwners()"| SVC["ClinicServiceImpl\n.findAllOwners()\n@Observed: Service_Owner_FindAll"]

    subgraph ORIGINAL["🟡 Baseline Original (fork original)"]
        SVC -.->|"findAll()\n[Delegação pura]"| OWN_REPO_0[("OwnerRepository")]
        OWN_REPO_0 -.->|"EAGER N+1 Implícito (pets)"| DB_0[("H2 In-Memory")]
    end

    subgraph DEGRADADO["❌ Baseline Degradada (intentional-code-smells)"]
        SVC -.->|"findAll() [1 query]"| OWN_REPO_1[("OwnerRepository\n(JPA)")]
        SVC -.->|"findByPetId() × N pets"| VISIT_REPO[("VisitRepository\n(N+1 queries)")]
        CTRL -.->|"Feature Envy: findByPetId() direto"| VISIT_REPO
    end

    subgraph REFATORADO["✅ Pós-Refatoração (pos-refactoring)"]
        SVC -->|"findAll() c/ JOIN FETCH\n[1 query otimizada]"| OWN_REPO_2[("OwnerRepository\n(JPA)")]
    end

    OWN_REPO_1 -.-> DB[("H2 In-Memory\nowners + pets")]
    VISIT_REPO -.-> DB
    OWN_REPO_2 --> DB

    K6 -->|"métricas"| M1["latencia_listar_owners\n(Trend)"]
    K6 -->|"checks"| M2["erro_listar_owners\n(Rate)"]

    style ORIGINAL fill:#fffacd,stroke:#f1c40f
    style DEGRADADO fill:#fff0f0,stroke:#ff6b6b
    style REFATORADO fill:#f0fff0,stroke:#51cf66
```

### 5.2 POST /api/owners — Cadeia Completa

```mermaid
flowchart TD
    K6["K6\nPOST /api/owners\n(grupo: 'POST /owners')"] --> CTRL["OwnerRestController\n.addOwner()\n@Observed: Controller_Owner_Add"]
    CTRL -->|"saveOwner()"| SVC["ClinicServiceImpl\n.saveOwner()\n@Observed: Service_Owner_Save"]
    SVC -->|"save() [1 INSERT]"| OWN_REPO[("OwnerRepository")]
    K6 -->|"métricas"| M["latencia_criar_owner / erro_criar_owner"]
```

### 5.3 POST /api/owners/{id}/pets — Cadeia Mais Profunda (4 spans)

```mermaid
flowchart TD
    K6["K6\nPOST /api/owners/{id}/pets\n(grupo: 'POST /owners/{ownerId}/pets')"] --> CTRL["OwnerRestController\n.addPetToOwner()\n@Observed: Controller_Pet_AddToOwner"]

    CTRL -->|"findOwnerById(id)"| SVC_OWN["ClinicServiceImpl\n.findOwnerById()\n@Observed: Service_Owner_FindById"]
    SVC_OWN -->|"findById()"| OWN_REPO[("OwnerRepository")]

    CTRL -->|"savePet(pet)"| SVC_PET["ClinicServiceImpl\n.savePet()\n@Observed: Service_Pet_Save"]
    SVC_PET -->|"findPetTypeById()"| SVC_PT["ClinicServiceImpl\n.findPetTypeById()\n@Observed: Service_PetType_FindById"]
    
    subgraph ORIGINAL["🟡 Baseline Original (fork original)"]
        SVC_PT -.->|"findById()"| PT_REPO_0[("PetTypeRepository")]
    end

    subgraph DEGRADADO["❌ Baseline Degradada (intentional-code-smells)"]
        SVC_PT -.->|"findById()"| PT_REPO_1[("PetTypeRepository")]
        SVC_PET -.->|"findAll() redundante"| PT_REPO_1
        SVC_PET -.->|"findOwnerById() extra"| SVC_OWN2["Service_Owner_FindById\n(2ª chamada)"]
    end

    subgraph REFATORADO["✅ Pós-Refatoração (pos-refactoring)"]
        SVC_PT -->|"findById() c/ @Cacheable"| PT_REPO_2[("PetTypeRepository\n(Cache Hit - 0 DB queries)")]
    end

    SVC_PET -->|"save() [1 INSERT]"| PET_REPO[("PetRepository")]
    K6 -->|"métricas"| M["latencia_criar_pet / erro_criar_pet"]

    style ORIGINAL fill:#fffacd,stroke:#f1c40f
    style DEGRADADO fill:#fff0f0,stroke:#ff6b6b
    style REFATORADO fill:#f0fff0,stroke:#51cf66
```

### 5.4 GET /api/vets — Relação N:M EAGER

```mermaid
flowchart TD
    K6["K6\nGET /api/vets\n(grupo: 'GET /vets')"] --> CTRL["VetRestController\n.listVets()\n@Observed: Controller_Vet_ListAll"]
    CTRL -->|"findAllVets()"| SVC["ClinicServiceImpl\n.findAllVets()\n@Observed: Service_Vet_FindAll"]
    
    subgraph ORIGINAL["🟡 Baseline Original (fork original)"]
        SVC -.->|"findAll()\n[EAGER N:M]"| VET_REPO_0[("VetRepository")]
        VET_REPO_0 -.->|"JOIN automático JPA"| DB_0[("H2 In-Memory")]
    end

    subgraph DEGRADADO["❌ Baseline Degradada (intentional-code-smells)"]
        SVC -.->|"findAll()\n[EAGER N:M]"| VET_REPO_1[("VetRepository")]
        VET_REPO_1 -.->|"JOIN"| DB[("H2 In-Memory\n(vets + specialties)")]
    end

    subgraph REFATORADO["✅ Pós-Refatoração (pos-refactoring)"]
        SVC -->|"@Cacheable('vets')\n[Cache Hit]"| CACHE[("Spring Cache\n(0 DB queries após warmup)")]
    end

    K6 -->|"métricas"| M["latencia_listar_vets / erro_listar_vets"]

    style ORIGINAL fill:#fffacd,stroke:#f1c40f
    style DEGRADADO fill:#fff0f0,stroke:#ff6b6b
    style REFATORADO fill:#f0fff0,stroke:#51cf66
```

### 5.5 Fluxo Completo K6 — Iteração Baseline Original

```mermaid
sequenceDiagram
    participant K6
    participant API as Spring Boot
    participant DB as H2

    rect rgb(255, 250, 205)
    Note over K6,DB: 🟡 Iteração Típica (fork original)
    
    K6->>API: GET /actuator/health
    API-->>K6: 200 OK

    K6->>API: GET /api/owners
    API->>DB: SELECT owners
    API->>DB: SELECT pets by owner_id (EAGER implícito N+1)
    API-->>K6: 200 OK | latencia_listar_owners

    K6->>API: POST /api/owners
    API->>DB: INSERT owner
    API-->>K6: 201 Created | latencia_criar_owner

    K6->>API: GET /api/owners/{id}
    API->>DB: SELECT owner by id
    API-->>K6: 200 OK | latencia_consultar_owner

    K6->>API: POST /api/owners/{id}/pets
    API->>DB: SELECT owner by id
    API->>DB: SELECT pet_type by id
    API->>DB: INSERT pet
    API-->>K6: 201 Created | latencia_criar_pet

    K6->>API: POST .../pets/{pid}/visits
    API->>DB: INSERT visit
    API-->>K6: 201 Created | latencia_criar_visit

    K6->>API: GET /api/vets
    API->>DB: SELECT vets LEFT OUTER JOIN vet_specialties
    API-->>K6: 200 OK | latencia_listar_vets
    end
```

### 5.6 Fluxo Completo K6 — Iteração na Baseline Degradada

```mermaid
sequenceDiagram
    participant K6
    participant API as Spring Boot
    participant DB as H2

    rect rgb(255, 240, 240)
    Note over K6,DB: ❌ Iteração Típica (intentional-code-smells)
    
    K6->>API: GET /actuator/health
    API-->>K6: 200 OK

    K6->>API: GET /api/owners
    API->>DB: SELECT owners
    loop N+1 Queries
        API->>DB: SELECT visits by petId
    end
    API-->>K6: 200 OK | latencia_listar_owners

    K6->>API: POST /api/owners
    API->>DB: INSERT owner
    API-->>K6: 201 Created | latencia_criar_owner

    K6->>API: GET /api/owners/{id}
    API->>DB: SELECT owner by id
    API-->>K6: 200 OK | latencia_consultar_owner

    K6->>API: POST /api/owners/{id}/pets
    API->>DB: SELECT * FROM pet_types (findAll)
    API->>DB: SELECT owner by id (redundante)
    API->>DB: INSERT pet
    API-->>K6: 201 Created | latencia_criar_pet

    K6->>API: POST .../pets/{pid}/visits
    API->>DB: INSERT visit
    API-->>K6: 201 Created | latencia_criar_visit

    K6->>API: GET /api/vets
    API->>DB: SELECT vets LEFT JOIN specialties (EAGER)
    API-->>K6: 200 OK | latencia_listar_vets
    end
```

### 5.7 Fluxo Completo K6 — Iteração Pós-Refatoração

```mermaid
sequenceDiagram
    participant K6
    participant API as Spring Boot
    participant Cache as Spring Cache
    participant DB as H2

    rect rgb(240, 255, 240)
    Note over K6,DB: ✅ Iteração Típica após Warm-up (pos-refactoring)
    
    K6->>API: GET /actuator/health
    API-->>K6: 200 OK

    K6->>API: GET /api/owners
    API->>DB: SELECT owners JOIN FETCH pets/visits (1 query)
    API-->>K6: 200 OK | latencia_listar_owners

    K6->>API: POST /api/owners
    API->>DB: INSERT owner
    API-->>K6: 201 Created | latencia_criar_owner

    K6->>API: GET /api/owners/{id}
    API->>DB: SELECT owner by id
    API-->>K6: 200 OK | latencia_consultar_owner

    K6->>API: POST /api/owners/{id}/pets
    API->>Cache: Hit: petType
    API->>DB: INSERT pet
    API-->>K6: 201 Created | latencia_criar_pet

    K6->>API: POST .../pets/{pid}/visits
    API->>DB: INSERT visit
    API-->>K6: 201 Created | latencia_criar_visit

    K6->>API: GET /api/vets
    API->>Cache: Hit: vets (0 queries DB)
    API-->>K6: 200 OK | latencia_listar_vets
    end
```

---

## 6. Onde as Anomalias se Localizam na Cadeia

```mermaid
flowchart LR
    subgraph ENDPOINT["GET /api/owners — Evolução da Cadeia"]
        direction TB
        
        subgraph ORIGINAL["🟡 Baseline Original (fork original)"]
            direction TB
            C0["Controller\nlistOwners()"] -->|"Delega"| S0["Service\nfindAllOwners()"]
            S0 -->|"JPA Repository"| R0["OwnerRepository\nfindAll()"]
            R0 -->|"EAGER implícito"| A0["⚠️ Potencial N+1\n(Pets carregados via queries extras)"]
        end

        subgraph DEGRADADA["❌ Baseline Degradada (intentional-code-smells)"]
            direction TB
            C1["Controller\nlistOwners()"] -->|"Feature Envy +\nVisitRepository direto"| A1["❌ Anomalia 2:\nViolação de Camada"]
            C1 --> S1["Service\nfindAllOwners()"]
            S1 -->|"N+1 Loop\nfindByPetId × N"| A2["❌ Anomalia 1:\nN+1 Queries"]
            S1 -->|"Sort em memória"| A3["⚠️ Anomalia:\nCC > 15"]
            S1 -->|"ownerHitCount\nnão thread-safe"| A4["⚠️ Anomalia:\nEstado mutável"]
        end

        subgraph REFATORADA["✅ Pós-Refatoração (pos-refactoring)"]
            direction TB
            C2["Controller\nlistOwners()"] -->|"Delega puramente"| S2["Service\nfindAllOwners()"]
            S2 -->|"Delega puramente"| R2["OwnerRepository\n(JOIN FETCH)"]
        end
    end
    
    style ORIGINAL fill:#fffacd,stroke:#f1c40f
    style DEGRADADA fill:#fff0f0,stroke:#ff6b6b
    style REFATORADA fill:#f0fff0,stroke:#51cf66
```



---

## 7. Síntese Comparativa do Impacto Estrutural

A comparação entre os cenários deve ser realizada considerando o mesmo endpoint, antes e após a introdução das anomalias arquiteturais, mantendo constantes o ambiente de execução, a carga de trabalho e os demais fatores experimentais. Dessa forma, busca-se isolar o impacto da degradação estrutural sobre as métricas de runtime.

O objetivo não é demonstrar que determinados endpoints são naturalmente mais lentos por acessarem banco de dados ou por executarem operações mais complexas, mas verificar se a degradação estrutural aumenta o custo de execução **desses mesmos fluxos** em comparação ao cenário original. A análise também deve considerar a magnitude da variação entre diferentes perfis de execução: caso endpoints com maior profundidade de chamadas, maior número de dependências ou maior interação com recursos externos apresentem degradação significativamente superior aos fluxos mais simples, isso constitui evidência de que o efeito depende do contexto de execução.

O mapeamento das anomalias introduzidas por módulo permite correlacionar características arquiteturais — como aumento do acoplamento e redução de coesão — com alterações nas métricas dinâmicas coletadas via Micrometer Observation. Embora essa abordagem não estabeleça causalidade absoluta, ela fornece evidências experimentais consistentes sobre a relação entre degradação estrutural e comportamento do sistema em tempo de execução. Mesmo que parte dos endpoints apresente pouca ou nenhuma alteração nas métricas de desempenho, esse resultado continua sendo cientificamente relevante, pois indica que os efeitos da degradação não são uniformes e dependem das características do fluxo executado.

### 7.1 Diferença de Profundidade no Fluxo (Exemplo: GET /owners)

O diagrama abaixo ilustra como a degradação altera a profundidade e a complexidade do fluxo de execução para uma mesma requisição K6, afetando o esforço computacional (CPU/Memória) e o número de interações com o banco de dados.

```mermaid
sequenceDiagram
    participant K6 as K6 (Carga Constante)
    participant API as Spring Boot (Controller & Service)
    participant DB as H2 Database / Cache

    rect rgb(255, 250, 205)
    Note over K6,DB: 🟡 Baseline Original (Profundidade Média)
    K6->>API: GET /api/owners
    API->>DB: SELECT owners (JPA findAll)
    API->>DB: SELECT pets (EAGER implícito)
    API-->>K6: 200 OK (Custo: N+1 queries geradas pelo ORM)
    end

    rect rgb(255, 240, 240)
    Note over K6,DB: ❌ Baseline Degradada (Profundidade Alta - Anomalias)
    K6->>API: GET /api/owners
    API->>DB: SELECT owners
    loop Para cada pet (Feature Envy + Quebra de Camada)
        API->>DB: SELECT visits by petId (Acesso direto a repositório vizinho)
    end
    Note right of API: Alto acoplamento, perda de coesão<br>e manipulação de estado complexa em memória
    API-->>K6: 200 OK (Custo: N+1 explícito + Custo de CPU em loops/sorts)
    end

    rect rgb(240, 255, 240)
    Note over K6,DB: ✅ Pós-Refatoração (Profundidade Baixa - Otimizado)
    K6->>API: GET /api/owners
    API->>DB: SELECT owners JOIN FETCH pets/visits
    API-->>K6: 200 OK (Custo: 1 Query otimizada, Delegação pura)
    end
```

---

## Referências

- Grafana Labs. _K6 Scenarios Documentation_. https://grafana.com/docs/k6/latest/using-k6/scenarios/
- Beyer, B. et al. (2016). _Site Reliability Engineering_. O'Reilly.
- Mann, H. B.; Whitney, D. R. (1947). "On a Test of Whether One of Two Random Variables Is Stochastically Larger Than the Other". _The Annals of Mathematical Statistics_, 18(1), 50–60.
- Ford, N.; Parsons, R.; Kua, P. (2022). _Building Evolutionary Architectures_, 2ª ed. O'Reilly.
