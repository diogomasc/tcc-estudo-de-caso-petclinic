### 5.3.2. Diagrama de Fluxo — Gargalo N+1 em `GET /api/owners`

Os diagramas de sequência abaixo contrastam o caminho de execução de `GET /api/owners` nas duas
branches relevantes. Note que a **profundidade instrumentada (`@Observed`) permanece em 2 spans**
em ambos os casos (Controller → Service, Tabela 5.4) — a diferença crítica não está no número de
camadas atravessadas, mas no que acontece **dentro** da camada Service/Repository, invisível ao
Micrometer e só identificável pela análise estática (Feature Envy, Tabela 5.3) combinada ao
tracing de queries.

```mermaid
%%{init: { 'theme': 'default', 'themeVariables': { 'textColor': '#000000', 'noteTextColor': '#000000', 'actorTextColor': '#000000', 'loopTextColor': '#000000', 'messageTextColor': '#000000' } } }%%
sequenceDiagram
    autonumber
    actor K6
    participant Controller as Controller_Owner_ListAll
    participant Service as Service_Owner_FindAll
    participant Repo as OwnerRepository
    participant DB as H2

    rect rgb(255, 230, 230)
    Note over K6,DB: ❌ Baseline Degradada — Feature Envy + violação de camada (Tabela 5.3)

    K6->>+Controller: GET /api/owners
    Note right of Controller: Profundidade @Observed: 2 spans (constante)

    Controller->>+Service: findAllOwners()
    Service->>+Repo: findAll()

    Repo->>+DB: SELECT * FROM owners
    DB-->>-Repo: N owners
    Repo-->>-Service: retorna N owners

    loop N+1 Queries (1 por owner iterado)
        Service->>+Repo: findVisitsByPetId(petId)
        Repo->>+DB: SELECT * FROM visits WHERE pet_id = ?
        DB-->>-Repo: visits
        Repo-->>-Service: dados iterados
    end

    Note over Service,Repo: Montagem dos dados em memória completa

    Service-->>-Controller: List<OwnerDTO>
    Controller-->>-K6: 200 OK | p95 ≈ 60.000 ms (Tabela 3a)
    end
```

```mermaid
%%{init: { 'theme': 'default', 'themeVariables': { 'textColor': '#000000', 'noteTextColor': '#000000', 'actorTextColor': '#000000', 'loopTextColor': '#000000', 'messageTextColor': '#000000' } } }%%
sequenceDiagram
    autonumber
    actor K6
    participant Controller as Controller_Owner_ListAll
    participant Service as Service_Owner_FindAll
    participant Repo as OwnerRepository
    participant DB as H2

    rect rgb(230, 255, 230)
    Note over K6,DB: ✅ Pós-Refatoração — persistência delegada ao ORM (Fowler, 2019)

    K6->>+Controller: GET /api/owners
    Note right of Controller: Profundidade @Observed: 2 spans (constante)

    Controller->>+Service: findAllOwners()
    Service->>+Repo: findAllWithPetsAndVisits()

    Repo->>+DB: SELECT owners JOIN FETCH pets, visits (1 query)
    DB-->>-Repo: owners + pets + visits (lote único)

    Repo-->>-Service: List<Owner>
    Service-->>-Controller: List<OwnerDTO>
    Controller-->>-K6: 200 OK | Δ p95 = −73,5% (Tabela 3a)
    end
```

### 5.3.3. Diagrama de Fluxo — Profundidade Constante em `POST /api/owners/{id}/pets`

Este par de diagramas ilustra o achado contra-intuitivo discutido no texto: a cadeia de **4
spans** (a maior profundidade mapeada na Tabela 5.4) permanece estruturalmente idêntica entre
Baseline Degradada e Pós-Refatoração — mesmo com a Complexidade Ciclomática da classe
responsável caindo de CC = 26 para CC = 2 após a extração da God Class. A refatoração
"esvaziou" a complexidade interna de cada span, sem eliminar ou colapsar nenhuma camada do
caminho de execução.

```mermaid
%%{init: { 'theme': 'default', 'themeVariables': { 'textColor': '#000000', 'noteTextColor': '#000000', 'actorTextColor': '#000000', 'loopTextColor': '#000000', 'messageTextColor': '#000000' } } }%%
sequenceDiagram
    autonumber
    actor K6
    participant Controller as Controller_Pet_AddToOwner
    participant SvcOwner as Service_Owner_FindById
    participant SvcType as Service_PetType_FindById
    participant SvcPet as Service_Pet_Save
    participant DB as H2

    rect rgb(255, 230, 230)
    Note over K6,DB: ❌ Baseline Degradada — CC = 26 (God Class, Tabela 5.3)

    K6->>+Controller: POST /api/owners/{id}/pets
    Note over Controller,SvcPet: Profundidade @Observed: 4 spans

    Controller->>+SvcOwner: findById(ownerId)
    SvcOwner->>+DB: SELECT owner by id
    DB-->>-SvcOwner: owner
    SvcOwner-->>-Controller: owner

    Controller->>+SvcType: findById(petTypeId)
    Note right of SvcType: God Class: validações e<br/>regras de negócio dispersas<br/>dentro do mesmo span
    SvcType->>+DB: SELECT pet_type by id
    DB-->>-SvcType: petType
    SvcType-->>-Controller: petType

    Controller->>+SvcPet: save(pet)
    SvcPet->>+DB: INSERT pet
    DB-->>-SvcPet: pet salvo
    SvcPet-->>-Controller: PetDTO

    Controller-->>-K6: 201 Created
    end
```

```mermaid
%%{init: { 'theme': 'default', 'themeVariables': { 'textColor': '#000000', 'noteTextColor': '#000000', 'actorTextColor': '#000000', 'loopTextColor': '#000000', 'messageTextColor': '#000000' } } }%%
sequenceDiagram
    autonumber
    actor K6
    participant Controller as Controller_Pet_AddToOwner
    participant SvcOwner as Service_Owner_FindById
    participant SvcType as Service_PetType_FindById
    participant SvcPet as Service_Pet_Save
    participant DB as H2

    rect rgb(230, 255, 230)
    Note over K6,DB: ✅ Pós-Refatoração — CC = 2 (Extract Class / SRP, Fowler 2019)

    K6->>+Controller: POST /api/owners/{id}/pets
    Note over Controller,SvcPet: Profundidade @Observed: 4 spans (inalterada)

    Controller->>+SvcOwner: findById(ownerId)
    SvcOwner->>+DB: SELECT owner by id
    DB-->>-SvcOwner: owner
    SvcOwner-->>-Controller: owner

    Controller->>+SvcType: findById(petTypeId)
    Note right of SvcType: Responsabilidade única:<br/>apenas consulta, sem<br/>regra de negócio embutida
    SvcType->>+DB: SELECT pet_type by id
    DB-->>-SvcType: petType
    SvcType-->>-Controller: petType

    Controller->>+SvcPet: save(pet)
    SvcPet->>+DB: INSERT pet
    DB-->>-SvcPet: pet salvo
    SvcPet-->>-Controller: PetDTO

    Controller-->>-K6: 201 Created
    end
```
