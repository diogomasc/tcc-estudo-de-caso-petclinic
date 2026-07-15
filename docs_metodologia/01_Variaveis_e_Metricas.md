# 01 — Variáveis e Métricas

> **TCC:** Mitigação de Débito Técnico Estrutural — Spring PetClinic REST  
> **Seção:** 4 Metodologia — Variáveis e Métricas

---

## 1. Classificação das Variáveis

| Categoria | Variável | Instrumento | Unidade |
|---|---|---|---|
| **Independente** | CBO (Coupling Between Objects) | PMD `CouplingBetweenObjects` | tipos referenciados |
| **Independente** | LCOM / God Class | PMD `GodClass` (WMC/ATFD/TCC) | score composto |
| **Independente** | Complexidade Ciclomática (CC) | PMD `CyclomaticComplexity` | inteiro |
| **Independente** | Violações ArchUnit (camadas / ciclos) | `ValidacaoArquiteturalTest` | violações |
| **Independente** | Long Method (NCSS / LOC) | PMD `NcssCount` + XPath | statements / linhas |
| **Dependente** | Latência p95 / p99 por endpoint | K6 Trends customizadas (`{phase:test}`) | ms |
| **Dependente** | Taxa de erros | K6 Rates customizadas (`{phase:test}`) | proporção [0–1] |
| **Dependente** | Throughput Útil (escritas concluídas) | K6 Counters (`*_com_sucesso`) | contagem abs. |
| **Dependente** | Saturação (`dropped_iterations`) | K6 (executor constant-arrival-rate) | contagem abs. |
| **Dependente** | Profundidade de pilha (@Observed spans) | Prometheus `metodo_execucao_seconds_*` | spans/req |

---

## 2. Variáveis Independentes — Snippets de Configuração

### 2.1 CBO — `ruleset.xml`, linhas 108–112

```xml
<rule ref="category/java/design.xml/CouplingBetweenObjects">
    <properties>
        <property name="threshold" value="20" />
    </properties>
</rule>
```

Limiar: CBO ≤ 20 (Chidamber & Kemerer, 1994). Classes acima deste valor são candidatas a _Shotgun Surgery_.

### 2.2 God Class / LCOM — `ruleset.xml`, linha 42

```xml
<rule ref="category/java/design.xml/GodClass" />
```

Detecta via WMC ≥ 47, ATFD > 5 e TCC < 1/3 (Lanza & Marinescu, 2006).

### 2.3 Complexidade Ciclomática — `ruleset.xml`, linhas 97–102

```xml
<rule ref="category/java/design.xml/CyclomaticComplexity">
    <properties>
        <property name="classReportLevel"  value="80" />
        <property name="methodReportLevel" value="10" />
    </properties>
</rule>
```

Limiar: CC ≤ 10 por método (McCabe, 1976).

### 2.4 Feature Envy (Law of Demeter) — `ruleset.xml`, linha 85

```xml
<rule ref="category/java/design.xml/LawOfDemeter" />
```

### 2.5 Long Method — `ruleset.xml`, linhas 49–79

```xml
<!-- NCSS: statements executáveis (substitui ExcessiveMethodLength no PMD 7) -->
<rule ref="category/java/design.xml/NcssCount">
    <properties>
        <property name="methodReportLevel" value="40" />
        <property name="classReportLevel"  value="500" />
    </properties>
</rule>

<!-- XPath: linhas brutas por método -->
<rule name="ExcessiveMethodLength" language="java"
      class="net.sourceforge.pmd.lang.rule.xpath.XPathRule">
    <priority>3</priority>
    <properties>
        <property name="xpath">
            <value><![CDATA[//MethodDeclaration[@EndLine - @BeginLine > 50]]]></value>
        </property>
    </properties>
</rule>
```

### 2.6 Violações ArchUnit — `ValidacaoArquiteturalTest.java`, linhas 86–147

```java
// Controllers NÃO devem acessar Repositories
@ArchTest
static final ArchRule controllers_nao_acessam_repositories =
    noClasses()
        .that().resideInAPackage("..rest.controller..")
        .should().dependOnClassesThat().resideInAPackage("..repository..")
        .because("Controllers devem delegar para a camada Service");

// Ausência de ciclos entre pacotes de negócio
@ArchTest
static final ArchRule sem_ciclos_entre_pacotes =
    slices()
        .matching("org.springframework.samples.petclinic.(*)..")
        .should().beFreeOfCycles()
        .because("Ciclos violam a ISO 25010 – Modularidade");
```

### 2.7 Métricas Ca/Ce — `ValidacaoArquiteturalTest.java`, linhas 154–194

```java
ComponentDependencyMetrics metrics =
    ArchitectureMetrics.componentDependencyMetrics(components);

int ca = metrics.getAfferentCoupling(pkgName);  // Fan-In
int ce = metrics.getEfferentCoupling(pkgName);  // Fan-Out
double instability = metrics.getInstability(pkgName); // Ce/(Ca+Ce)
```

### 2.8 Maven — versões (`pom.xml`, linhas 37, 47, 155–157, 500–501)

```xml
<properties>
    <maven.pmd-plugin.version>3.26.0</maven.pmd-plugin.version>
    <archunit.version>1.3.0</archunit.version>
</properties>

<dependency>
    <groupId>com.tngtech.archunit</groupId>
    <artifactId>archunit-junit5</artifactId>
    <version>${archunit.version}</version>
    <scope>test</scope>
</dependency>
```

---

## 3. Variáveis Dependentes — Configuração K6

### 3.1 Latência p95/p99 (`load-test.js`)

```javascript
thresholds: {
    "latencia_listar_owners{phase:test}":   ["p(95)<4000", "p(99)<5000"],
    "latencia_criar_owner{phase:test}":     ["p(95)<3000", "p(99)<4000"],
    "latencia_consultar_owner{phase:test}": ["p(95)<3000", "p(99)<4000"],
    "latencia_criar_pet{phase:test}":       ["p(95)<3000", "p(99)<4000"],
    "latencia_criar_visit{phase:test}":     ["p(95)<3000", "p(99)<4000"],
    "latencia_listar_vets{phase:test}":     ["p(95)<2000", "p(99)<3000"],
}
```

### 3.2 Taxa de Erros (`load-test.js`)

```javascript
thresholds: {
    "taxa_erro":                        ["rate<0.10"], // Global
    "erro_listar_owners{phase:test}":   ["rate<0.10"],
    "erro_criar_owner{phase:test}":     ["rate<0.10"],
    "erro_consultar_owner{phase:test}": ["rate<0.10"],
    "erro_criar_pet{phase:test}":       ["rate<0.10"],
    "erro_criar_visit{phase:test}":     ["rate<0.10"],
    "erro_listar_vets{phase:test}":     ["rate<0.10"],
}
```

### 3.3 Profundidade de pilha — Cadeia de spans por endpoint

A estrutura de "cadeia de spans" instrumentada com `@Observed` se manteve constante ao longo das três branches do experimento. O destaque metodológico ocorre no endpoint `POST /api/owners/{id}/pets`, que sustenta uma profundidade de 4 spans em todas as versões. Contudo, a refatoração (`pos-refactoring`) **esvaziou a complexidade interna** e a ineficiência contidas nesses spans, eliminando anomalias estruturais (como repetições desnecessárias) que ocorriam sob o mesmo "teto" de profundidade na versão degradada.

| Endpoint | Spans (@Observed) | Profundidade |
|---|---|---|
| `GET /api/owners` | `Controller_Owner_ListAll` → `Service_Owner_FindAll` | 2 |
| `POST /api/owners` | `Controller_Owner_Add` → `Service_Owner_Save` | 2 |
| `GET /api/owners/{id}` | `Controller_Owner_FindById` → `Service_Owner_FindById` | 2 |
| `POST /api/owners/{id}/pets` | `Controller_Pet_AddToOwner` → `Service_Owner_FindById` → `Service_PetType_FindById` → `Service_Pet_Save` | **4** |
| `POST .../visits` | `Controller_Visit_AddToOwner` → `Service_Visit_Save` | 2 |
| `GET /api/vets` | `Controller_Vet_ListAll` → `Service_Vet_FindAll` | 2 |

---

## 4. Mapeamento ISO 25010

| Característica | Sub-característica | Métricas |
|---|---|---|
| Manutenibilidade | Analisabilidade | CC, NCSS, LOC |
| Manutenibilidade | Modificabilidade | CBO, Ca/Ce, ArchUnit |
| Manutenibilidade | Modularidade | Ciclos, LCOM |
| Eficiência de Desempenho | Comportamento temporal | Latência p95/p99, throughput |
| Confiabilidade | Tolerância a falhas | Taxa de erros |

---

## Referências

- Chidamber & Kemerer (1994). _A Metrics Suite for OO Design_. IEEE TSE 20(6).
- McCabe (1976). _A Complexity Measure_. IEEE TSE 2(4).
- Lanza & Marinescu (2006). _Object-Oriented Metrics in Practice_. Springer.
- Fowler (2018). _Refactoring_, 2ª ed. Addison-Wesley.
