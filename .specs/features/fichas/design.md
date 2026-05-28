# Fichas — Design

**Spec**: `.specs/features/fichas/spec.md`
**Context**: `.specs/features/fichas/context.md`
**Status**: Draft

---

## Visão Geral da Arquitetura

Dois domínios novos independentes (`Anamnese` e `LashMapping`) seguindo o mesmo padrão hexagonal do projeto. O ponto especial é o **fluxo público da anamnese**: a rota `/ficha/:token` vive fora do `authGuard`, com controller próprio sem JWT e página Angular sem sidebar/bottom-nav.

```
AUTENTICADO                          PÚBLICO
─────────────────────────────        ───────────────────────
/fichas/anamnese                     /ficha/:token
  AnamneseListComponent ──────────── PublicAnamneseComponent
  AnamneseFormComponent    token       (sem auth, sem layout)
         │                   │               │
  AnamneseController    GenerateLinkUC   PublicAnamneseController
         │                               (permitAll no Security)
  SaveAnamneseUseCase
  GetOrCreateAnamneseUseCase

/fichas/mapping
  MappingListComponent
  MappingHistoryComponent
  MappingFormComponent
    └── EyeCanvasComponent (HTML5 Canvas)
         │
  LashMappingController
  CreateMappingUseCase / UpdateMappingUseCase
```

---

## Migration — V7

```sql
-- Anamnese: uma por cliente (UNIQUE em client_id)
CREATE TABLE anamneses (
    id                      UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id               UUID NOT NULL UNIQUE REFERENCES clients(id),
    guardian_name           VARCHAR(255),
    address                 VARCHAR(255),
    neighborhood            VARCHAR(100),
    city                    VARCHAR(100),
    state                   VARCHAR(2),
    birth_date              DATE,
    phone                   VARCHAR(20),
    cpf                     VARCHAR(14),
    rg                      VARCHAR(20),
    -- saúde (toggles)
    had_lash_extensions     BOOLEAN NOT NULL DEFAULT false,
    wears_mascara           BOOLEAN NOT NULL DEFAULT false,
    has_allergies           BOOLEAN NOT NULL DEFAULT false,
    has_thyroid_issues      BOOLEAN NOT NULL DEFAULT false,
    sleep_side              VARCHAR(20)  NOT NULL DEFAULT 'AMBOS',
    had_eye_procedure       BOOLEAN NOT NULL DEFAULT false,
    is_pregnant_or_nursing  BOOLEAN NOT NULL DEFAULT false,
    had_oncological_treatment BOOLEAN NOT NULL DEFAULT false,
    has_skin_disease        BOOLEAN NOT NULL DEFAULT false,
    has_health_treatment    BOOLEAN NOT NULL DEFAULT false,
    uses_medication         BOOLEAN NOT NULL DEFAULT false,
    -- termo
    term_accepted           BOOLEAN NOT NULL DEFAULT false,
    term_accepted_at        TIMESTAMP,
    created_at              TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at              TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Tokens para preenchimento público pela cliente
CREATE TABLE anamnese_tokens (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id  UUID NOT NULL REFERENCES clients(id),
    token      VARCHAR(64) NOT NULL UNIQUE,
    expires_at TIMESTAMP NOT NULL,
    used       BOOLEAN NOT NULL DEFAULT false,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);

-- Mapping: múltiplos por cliente (histórico)
CREATE TABLE lash_mappings (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    client_id      UUID NOT NULL REFERENCES clients(id),
    mapping_date   DATE NOT NULL,
    mapping_type   VARCHAR(100),
    curvature      VARCHAR(10),
    humidity       VARCHAR(50),
    temperature    VARCHAR(50),
    thickness      VARCHAR(10),
    thread_brand   VARCHAR(100),
    thread_format  VARCHAR(50),
    adhesive       VARCHAR(100),
    lengths_used   VARCHAR(255),
    observations   TEXT,
    canvas_data    TEXT,     -- base64 PNG do canvas
    photo_before   TEXT,     -- base64 JPEG comprimido (max ~800px, q0.7)
    photo_after    TEXT,
    created_at     TIMESTAMP NOT NULL DEFAULT NOW(),
    updated_at     TIMESTAMP NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_anamnese_tokens_token     ON anamnese_tokens(token);
CREATE INDEX idx_lash_mappings_client_id   ON lash_mappings(client_id);
CREATE INDEX idx_lash_mappings_date        ON lash_mappings(mapping_date DESC);
```

---

## Backend — Componentes

### Domínio

**Modelos:**
- `Anamnese` — todos os campos da tabela + `clientName` (join read-only)
- `AnamneseToken` — `id, clientId, token, expiresAt, used`
- `LashMapping` — todos os campos da tabela + `clientName` (join read-only)

**Exceções:**
- `AnamneseNotFoundException`
- `AnamneseTokenNotFoundException`
- `AnamneseTokenExpiredException`
- `AnamneseTokenAlreadyUsedException`
- `LashMappingNotFoundException`

**Ports/In:**

| Interface | Comando / Query |
|---|---|
| `GetOrCreateAnamneseUseCase` | `execute(clientId: UUID) → AnamneseResult` |
| `SaveAnamneseUseCase` | `execute(SaveAnamneseCommand) → AnamneseResult` |
| `ListAnamneseSummaryUseCase` | `execute(search, pageable) → Page<AnamneseSummaryResult>` |
| `GenerateAnamneseLinkUseCase` | `execute(clientId: UUID) → String (URL com token)` |
| `GetAnamneseByTokenUseCase` | `execute(token: String) → AnamneseTokenResult` — **sem auth** |
| `SubmitAnamneseByTokenUseCase` | `execute(SubmitAnamneseCommand) → void` — **sem auth** |
| `CreateLashMappingUseCase` | `execute(CreateMappingCommand) → MappingResult` |
| `UpdateLashMappingUseCase` | `execute(UpdateMappingCommand) → MappingResult` |
| `GetLashMappingUseCase` | `execute(id: UUID) → MappingResult` |
| `ListMappingsByClientUseCase` | `execute(clientId, pageable) → Page<MappingResult>` |
| `ListMappingSummaryUseCase` | `execute(search, pageable) → Page<MappingSummaryResult>` |

**Ports/Out:**
- `AnamneseRepository` — `save, findByClientId, findAll(search, pageable), existsByClientId`
- `AnamneseTokenRepository` — `save, findByToken, deleteExpired`
- `LashMappingRepository` — `save, findById, findByClientId(pageable), findAll(search, pageable), delete`

### Infraestrutura

**Entities:** `AnamneseEntity`, `AnamneseTokenEntity`, `LashMappingEntity`
- `LashMappingEntity` e `AnamneseEntity`: `@ManyToOne(fetch=LAZY)` para `ClientEntity` com `insertable=false, updatable=false` apenas para resolução do `clientName`

**JPA Repositories:**
- `AnamneseJpaRepository` — `findByClientId`, `findAllWithClient(search, pageable)` com `LEFT JOIN FETCH`
- `AnamneseTokenJpaRepository` — `findByTokenAndUsedFalseAndExpiresAtAfter`
- `LashMappingJpaRepository` — `findByClientIdOrderByMappingDateDesc`, `findAllWithClient(search, pageable)`

### Controllers

**`AnamneseController`** — `/api/anamnese` (autenticado)
```
GET  /api/anamnese?search=&page=&size=   → Page<AnamneseSummaryResponse>
GET  /api/anamnese/{clientId}            → AnamneseResponse (ou 204 se não existe)
PUT  /api/anamnese/{clientId}            → AnamneseResponse (cria ou atualiza)
POST /api/anamnese/{clientId}/link       → { url: String } — gera token
```

**`PublicAnamneseController`** — `/api/public/anamnese` (sem JWT — `permitAll`)
```
GET  /api/public/anamnese/{token}        → AnamnesePublicResponse (dados cliente + ficha atual)
POST /api/public/anamnese/{token}        → 200 OK — submete preenchimento
```

**`LashMappingController`** — `/api/mappings` (autenticado)
```
GET  /api/mappings?search=&page=&size=          → Page<MappingSummaryResponse>
GET  /api/mappings/client/{clientId}?page=&size= → Page<MappingResponse>
GET  /api/mappings/{id}                         → MappingResponse
POST /api/mappings/client/{clientId}            → MappingResponse (201)
PUT  /api/mappings/{id}                         → MappingResponse
DELETE /api/mappings/{id}                       → 204
```

**SecurityConfig** — adicionar:
```java
.requestMatchers("/api/public/**").permitAll()
```

---

## Frontend — Componentes

### Estrutura de rotas

```
/fichas                            → FichasComponent (shell com sub-nav)
  /fichas/anamnese                 → AnamneseListComponent
  /fichas/anamnese/:clientId       → AnamneseFormComponent
  /fichas/mapping                  → MappingListComponent
  /fichas/mapping/:clientId        → MappingHistoryComponent
  /fichas/mapping/:clientId/nova   → MappingFormComponent (novo)
  /fichas/mapping/:clientId/:id    → MappingFormComponent (editar)

/ficha/:token                      → PublicAnamneseComponent
  (fora do LayoutComponent, fora do authGuard)
```

### Feature `features/fichas`

```
features/fichas/
├── fichas.routes.ts
├── fichas.component.ts/.html/.css       ← shell com tabs Anamnese | Mapping
├── models/
│   └── fichas.model.ts
├── services/
│   ├── anamnese.service.ts
│   └── mapping.service.ts
├── store/
│   ├── fichas.actions.ts
│   ├── fichas.reducer.ts
│   ├── fichas.selectors.ts
│   └── fichas.effects.ts
├── components/
│   ├── anamnese-list/               ← lista clientes com status da ficha
│   ├── anamnese-form/               ← formulário completo (profissional)
│   ├── mapping-list/                ← lista clientes com qtd de fichas
│   ├── mapping-history/             ← histórico de fichas de um cliente
│   ├── mapping-form/                ← formulário + canvas + fotos
│   └── eye-canvas/                  ← componente standalone do canvas
└── public/
    └── public-anamnese/             ← fora do layout autenticado
```

### `EyeCanvasComponent` — detalhe

Componente standalone, usa HTML5 Canvas API nativa (sem biblioteca externa).

**Inputs/Outputs:**
```typescript
@Input() initialData: string | null   // base64 PNG para pré-carregar
@Output() canvasChange = new EventEmitter<string>()  // base64 atualizado

// Controles internos:
selectedColor: string   // hex
selectedSize: number    // 2 | 5 | 10
history: ImageData[]    // para undo
redoStack: ImageData[]  // para redo
```

**SVG do olho:** inline no template como `<svg>` de fundo (contorno ovóide estilizado). Canvas fica sobre o SVG via `position: absolute`.

**Paleta de cores predefinida:** 6 cores fixas + 1 color picker nativo:
`['#000000', '#C8A2A2', '#D4AF37', '#e53935', '#1565c0', '#2e7d32', 'custom']`

**Persistência:** `canvas.toDataURL('image/png')` → base64 string → campo `canvasData` da ficha.

### `MappingFormComponent` — campos

Grid 2 colunas no desktop, 1 coluna no mobile:

| Campo | Tipo | Opções |
|---|---|---|
| Tipo de Mapping | `mat-select` | Natural Clássico, Volume, Mega Volume, Híbrido, Russa, Wet Look |
| Curvatura | `mat-select` | B, C, D, L, M, J, CC |
| Umidade | `input number` | — |
| Temperatura | `input number` | — |
| Espessura | `mat-select` | 0.03, 0.05, 0.07, 0.10, 0.12, 0.15, 0.18, 0.20, 0.25 |
| Marca do fio | `input text` | — |
| Formato do fio | `mat-select` | Natural, Gatinho, Boneca, Modelo, Fox Eye |
| Adesivo / Cola | `input text` | — |
| Comprimentos usados | `input text` | ex: "8, 10, 12mm" |
| Observações | `textarea` | — |

### `AnamneseFormComponent` — estrutura

Seções com `mat-card` ou divisores:
1. **Dados do cliente** — campos pré-preenchidos do cliente (readonly para nome/telefone do cadastro, editáveis os demais)
2. **Saúde do cliente** — lista de `mat-slide-toggle` + 1 `mat-radio-group` (lado de dormir)
3. **Termo de autorização** — texto estático + `mat-checkbox` → ao marcar, registra timestamp e exibe badge "Assinado ✓"

### Integração com `ClientDetailComponent`

Adicionar dois botões de ação na tela de detalhe do cliente:
- `[routerLink]="['/fichas/anamnese', client.id]"` → "Ficha de Anamnese"
- `[routerLink]="['/fichas/mapping', client.id]"` → "Fichas de Mapping"

### Navegação — sidebar e bottom nav

**Sidebar** — adicionar após Estoque:
```typescript
{ label: 'Fichas', icon: 'assignment', route: '/fichas' }
```

**Bottom nav** — agora serão 7 itens. Solução: reorganizar para 5 + "Mais":
- 5 fixos: Dashboard, Agenda, Clientes, Fichas, Financeiro
- "Mais" (menu_open) → abre um `MatBottomSheet` com: Serviços, Estoque
- Simplifica e dá espaço para Fichas (usado diariamente)

### Rota pública `/ficha/:token`

Em `app.routes.ts`, adicionar **fora** do bloco `canActivate: [authGuard]`:
```typescript
{
  path: 'ficha/:token',
  loadChildren: () => import('./features/fichas/public/public-anamnese.routes')
    .then(m => m.publicAnamneseRoutes)
}
```
A `PublicAnamneseComponent` renderiza um layout próprio (logo + formulário limpo + botão salvar), sem sidebar nem bottom nav.

### NgRx Store

Estado único `fichas`:
```typescript
interface FichasState {
  // Anamnese
  anamneseSummaries: AnamneseSummary[];
  totalAnamneses: number;
  currentAnamnese: Anamnese | null;
  generatedLink: string | null;
  // Mapping
  mappingSummaries: MappingSummary[];
  totalMappings: number;
  clientMappings: LashMapping[];      // histórico do cliente selecionado
  currentMapping: LashMapping | null;
  // Filtros
  search: string;
  page: number;
  // UI
  isLoading: boolean;
  error: string | null;
}
```

---

## Modelos TypeScript

```typescript
// Anamnese
export interface Anamnese {
  id: string;
  clientId: string;
  clientName: string;
  guardianName: string | null;
  address: string | null;
  neighborhood: string | null;
  city: string | null;
  state: string | null;
  birthDate: string | null;
  phone: string | null;
  cpf: string | null;
  rg: string | null;
  // saúde
  hadLashExtensions: boolean;
  wearsMascara: boolean;
  hasAllergies: boolean;
  hasThyroidIssues: boolean;
  sleepSide: 'DIREITO' | 'ESQUERDO' | 'AMBOS';
  hadEyeProcedure: boolean;
  isPregnantOrNursing: boolean;
  hadOncologicalTreatment: boolean;
  hasSkinDisease: boolean;
  hasHealthTreatment: boolean;
  usesMedication: boolean;
  // termo
  termAccepted: boolean;
  termAcceptedAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface AnamneseSummary {
  clientId: string;
  clientName: string;
  hasAnamnese: boolean;
  updatedAt: string | null;
}

// Mapping
export interface LashMapping {
  id: string;
  clientId: string;
  clientName: string;
  mappingDate: string;
  mappingType: string | null;
  curvature: string | null;
  humidity: string | null;
  temperature: string | null;
  thickness: string | null;
  threadBrand: string | null;
  threadFormat: string | null;
  adhesive: string | null;
  lengthsUsed: string | null;
  observations: string | null;
  canvasData: string | null;    // base64 PNG
  photoBefore: string | null;   // base64 JPEG
  photoAfter: string | null;    // base64 JPEG
  createdAt: string;
  updatedAt: string;
}

export interface MappingSummary {
  clientId: string;
  clientName: string;
  mappingCount: number;
  lastMappingDate: string | null;
}
```

---

## Decisões Técnicas

| Decisão | Escolha | Motivo |
|---|---|---|
| Canvas lib | HTML5 Canvas nativo | Sem dependência extra, API suficiente para freehand |
| Persistência do canvas | `toDataURL('image/png')` base64 | Simples, sem endpoint separado; tamanho aceitável para canvas 400×200px |
| Compressão de fotos | Canvas resample no frontend antes de enviar | Limita fotos a ~100KB; evita payload gigante no JSON |
| Token de anamnese pública | UUID + validade 7 dias, one-time use | Seguro e simples; após uso o link expira |
| Anamnese: upsert por clientId | `PUT /api/anamnese/{clientId}` sempre | Evita lógica POST/PUT no frontend; backend faz insert or update |
| Bottom nav com 7 destinos | 5 fixos + "Mais" com MatBottomSheet | Melhor UX que 7 ícones espremidos |

---

## Tratamento de Erros

| Cenário | Handling | Usuário vê |
|---|---|---|
| Token inválido ou não encontrado | `AnamneseTokenNotFoundException` → 404 | Página pública: "Link inválido ou expirado" |
| Token expirado | `AnamneseTokenExpiredException` → 410 | Página pública: "Este link expirou. Peça um novo à profissional." |
| Token já usado | `AnamneseTokenAlreadyUsedException` → 409 | Página pública: "Esta ficha já foi preenchida." |
| Mapping não encontrado | `LashMappingNotFoundException` → 404 | Toast de erro |
| Foto muito grande | Validação no frontend (max 5MB antes de comprimir) | Aviso inline antes de enviar |

