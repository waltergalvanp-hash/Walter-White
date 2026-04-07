# BESS-CRM — Especificación Técnica para Claude Code

## Descripción del Proyecto

Construir una **Single-Page Application (SPA)** de CRM ligero para el seguimiento comercial y técnico de proyectos de sistemas de almacenamiento de energía en baterías (BESS). La base de clientes máxima es de ~50 empresas. No requiere backend ni base de datos remota. Todo el estado persiste en `localStorage` con exportación a Excel al cierre trimestral.

---

## Stack Tecnológico (sin negociación)

| Capa | Tecnología | Versión |
|---|---|---|
| Framework | React | 18.x |
| Build tool | Vite | 5.x |
| Estilos | Tailwind CSS | 3.x |
| Componentes UI | shadcn/ui | latest |
| Iconos | lucide-react | latest |
| Persistencia | localStorage + JSON serializado | — |
| Exportación Excel | xlsx (SheetJS) | latest |
| Calendario Teams | Microsoft Graph REST API v1.0 | — |
| Calendario Notion | Notion API v1 | — |
| Routing | React Router DOM | 6.x |
| Estado global | Zustand | 4.x |
| Formularios | React Hook Form + Zod | latest |
| Fechas | date-fns | 3.x |
| Notificaciones | sonner (toast) | latest |
| Gráficos dashboard | Recharts | 2.x |

**No usar**: Redux, Axios, moment.js, jQuery, Bootstrap, MUI, Ant Design.

---

## Estructura de Archivos

```
bess-crm/
├── CLAUDE.md                      ← este archivo
├── index.html
├── vite.config.ts
├── tailwind.config.ts
├── tsconfig.json
├── package.json
├── src/
│   ├── main.tsx
│   ├── App.tsx                    ← router raíz
│   ├── types/
│   │   ├── client.ts              ← tipos TypeScript para entidades
│   │   ├── opportunity.ts
│   │   ├── interaction.ts
│   │   ├── visit.ts
│   │   ├── quotation.ts
│   │   └── project.ts
│   ├── store/
│   │   ├── useClientStore.ts      ← Zustand store: clientes
│   │   ├── useOpportunityStore.ts ← Zustand store: oportunidades
│   │   ├── useUIStore.ts          ← Zustand store: estado UI global
│   │   └── persistence.ts        ← lógica de localStorage serialize/deserialize
│   ├── lib/
│   │   ├── excel.ts               ← exportación SheetJS + cierre trimestral
│   │   ├── teams.ts               ← integración Microsoft Graph API
│   │   ├── notion.ts              ← integración Notion Calendar API
│   │   ├── alerts.ts              ← motor de alertas y recordatorios
│   │   └── utils.ts               ← helpers generales
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Sidebar.tsx
│   │   │   ├── TopBar.tsx
│   │   │   └── AlertBanner.tsx
│   │   ├── clients/
│   │   │   ├── ClientList.tsx
│   │   │   ├── ClientCard.tsx
│   │   │   ├── ClientForm.tsx
│   │   │   └── ClientDetail.tsx
│   │   ├── opportunities/
│   │   │   ├── OpportunityKanban.tsx
│   │   │   ├── OpportunityCard.tsx
│   │   │   ├── OpportunityForm.tsx
│   │   │   └── OpportunityDetail.tsx
│   │   ├── interactions/
│   │   │   ├── InteractionTimeline.tsx
│   │   │   └── InteractionForm.tsx
│   │   ├── visits/
│   │   │   ├── VisitList.tsx
│   │   │   └── VisitForm.tsx
│   │   ├── quotations/
│   │   │   ├── QuotationList.tsx
│   │   │   └── QuotationForm.tsx
│   │   ├── dashboard/
│   │   │   ├── Dashboard.tsx
│   │   │   ├── FunnelChart.tsx
│   │   │   ├── AlertsPanel.tsx
│   │   │   └── KpiCards.tsx
│   │   ├── calendar/
│   │   │   ├── CalendarView.tsx
│   │   │   └── CalendarIntegrationSettings.tsx
│   │   ├── settings/
│   │   │   ├── Settings.tsx
│   │   │   ├── QuarterlyClose.tsx
│   │   │   └── UserRoles.tsx
│   │   └── shared/
│   │       ├── StatusBadge.tsx
│   │       ├── BessAppBadge.tsx
│   │       ├── ConfirmDialog.tsx
│   │       ├── EmptyState.tsx
│   │       └── SearchBar.tsx
│   └── pages/
│       ├── DashboardPage.tsx
│       ├── ClientsPage.tsx
│       ├── ClientDetailPage.tsx
│       ├── OpportunitiesPage.tsx
│       ├── OpportunityDetailPage.tsx
│       ├── CalendarPage.tsx
│       └── SettingsPage.tsx
```

---

## Modelos de Datos (TypeScript — exactos)

### `Client`
```typescript
// src/types/client.ts
export type BessApplication =
  | 'peak_shaving'
  | 'primary_frequency_regulation'
  | 'ramp_support_blackout';

export type ClientStatus = 'prospect' | 'active' | 'inactive' | 'lost';

export interface Client {
  id: string;                         // nanoid()
  createdAt: string;                  // ISO 8601
  updatedAt: string;

  // Datos corporativos
  companyName: string;                // requerido
  ruc: string;                        // 11 dígitos, validar con Zod
  sector: string;                     // minería | industria | generación | transmisión | comercio | otro
  subSector?: string;
  website?: string;
  address?: string;
  region: string;                     // departamento del Perú
  city?: string;

  // Datos de contacto principal
  primaryContact: {
    name: string;
    role: string;
    email: string;
    phone: string;
    linkedIn?: string;
  };

  // Contactos adicionales (0..n)
  additionalContacts: Array<{
    id: string;
    name: string;
    role: string;
    email: string;
    phone?: string;
  }>;

  // Aplicaciones BESS asociadas (1..3)
  bessApplications: BessApplication[];  // mínimo 1, máximo 3

  // Datos técnicos preliminares (por aplicación)
  technicalProfile: {
    estimatedPowerKw?: number;
    estimatedEnergyKwh?: number;
    gridConnectionVoltageKv?: number;
    hasExistingGeneration?: boolean;
    existingGenerationMw?: number;
    notes?: string;
  };

  status: ClientStatus;

  // Clasificación de riesgo / potencial
  potentialValue?: number;            // USD estimado del proyecto
  priority: 'high' | 'medium' | 'low';

  assignedTo: string;                 // userId del responsable
  tags: string[];
  internalNotes?: string;
}
```

### `Opportunity`
```typescript
// src/types/opportunity.ts
export type OpportunityStage =
  | 'prospect_registered'       // 01 - Registro de prospecto
  | 'initial_contact'           // 02 - Contacto inicial
  | 'initial_meeting'           // 03 - Reunión comercial inicial
  | 'site_visit_1'              // 04 - Visita presencial
  | 'preliminary_evaluation'    // 05 - Evaluación preliminar
  | 'quotation_requested'       // 06 - Solicitud de cotización
  | 'technical_visit_2'         // 07 - Segunda visita técnica
  | 'deliverables_preparation'  // 08 - Preparación de entregables
  | 'proposal_presented'        // 09 - Presentación de propuesta
  | 'bid_negotiation'           // 10 - Seguimiento licitación / negociación
  | 'awarded'                   // 11a - Adjudicada
  | 'not_awarded'               // 11b - No adjudicada
  | 'in_execution'              // 12 - En ejecución
  | 'aftersales';               // 13 - Postventa

export type OpportunityResult = 'open' | 'awarded' | 'lost' | 'cancelled';

export interface Opportunity {
  id: string;
  clientId: string;
  createdAt: string;
  updatedAt: string;

  name: string;                       // ej: "RPF San Gabán II - 2025"
  bessApplications: BessApplication[];// aplicaciones específicas de esta oportunidad

  stage: OpportunityStage;
  result: OpportunityResult;

  // Datos económicos
  estimatedValueUsd?: number;
  currency: 'USD' | 'PEN';
  probability: number;                // 0-100 (%)

  // Fechas clave
  expectedClosingDate?: string;       // ISO
  bidSubmissionDeadline?: string;
  contractStartDate?: string;
  contractEndDate?: string;

  // Competencia
  competitors: string[];
  competitiveAdvantages?: string;

  // Requerimientos técnicos básicos
  requiredPowerKw?: number;
  requiredEnergyKwh?: number;
  requiredAvailabilityPct?: number;
  regulatoryFramework?: string;       // ej: "RPF Perú - OSINERGMIN OS-0538-2022"

  // Entregables asociados
  deliverables: Array<{
    id: string;
    name: string;
    type: 'proposal' | 'technical_report' | 'quotation' | 'contract' | 'study' | 'other';
    status: 'pending' | 'in_progress' | 'review' | 'submitted' | 'approved';
    dueDate?: string;
    fileUrl?: string;
    notes?: string;
  }>;

  assignedTo: string;
  nextAction?: string;
  nextActionDate?: string;

  lostReason?: string;
  internalNotes?: string;
}
```

### `Interaction`
```typescript
// src/types/interaction.ts
export type InteractionType =
  | 'email'
  | 'phone_call'
  | 'video_call'
  | 'in_person_meeting'
  | 'site_visit'
  | 'proposal_delivery'
  | 'bid_response'
  | 'internal_note'
  | 'whatsapp';

export interface Interaction {
  id: string;
  clientId: string;
  opportunityId?: string;             // opcional: vincula a oportunidad específica
  createdAt: string;
  date: string;                       // fecha real de la interacción

  type: InteractionType;
  subject: string;
  summary: string;                    // máx 500 chars
  participants: string[];             // nombres o emails
  outcome?: string;
  nextSteps?: string;
  nextStepDate?: string;

  attachments: Array<{
    name: string;
    url?: string;
    type: string;
  }>;

  createdBy: string;                  // userId
}
```

### `Visit`
```typescript
// src/types/visit.ts
export type VisitType = 'commercial_visit_1' | 'technical_visit_2' | 'followup_visit';
export type VisitStatus = 'scheduled' | 'completed' | 'cancelled' | 'rescheduled';

export interface Visit {
  id: string;
  clientId: string;
  opportunityId?: string;
  createdAt: string;

  type: VisitType;
  status: VisitStatus;

  scheduledDate: string;              // ISO datetime
  completedDate?: string;
  durationMinutes?: number;

  location: string;
  participants: string[];

  // Solo visita técnica (visita 2)
  technicalDataCollected?: {
    loadProfileAvailable: boolean;
    gridConnectionDataAvailable: boolean;
    existingEquipmentInventory: boolean;
    sitePhotos: boolean;
    spatialConstraints?: string;
    additionalFindings?: string;
  };

  reportSummary?: string;
  nextSteps?: string;

  // Recordatorio calendario
  reminderDays: number;               // días de anticipación para alerta
  calendarEventId?: string;           // ID del evento en Teams/Notion
  calendarProvider?: 'teams' | 'notion';

  createdBy: string;
}
```

### `Quotation`
```typescript
// src/types/quotation.ts
export type QuotationStatus =
  | 'draft' | 'internal_review' | 'sent' | 'clarification_requested'
  | 'revised' | 'accepted' | 'rejected' | 'expired';

export interface QuotationLineItem {
  id: string;
  description: string;
  bessApplication: BessApplication;
  powerKw: number;
  energyKwh: number;
  unitPriceUsd: number;
  quantity: number;
  totalUsd: number;
  notes?: string;
}

export interface Quotation {
  id: string;
  clientId: string;
  opportunityId: string;
  createdAt: string;
  updatedAt: string;

  code: string;                       // ej: "COT-2025-Q1-001"
  version: number;                    // empieza en 1
  parentQuotationId?: string;         // para revisiones

  status: QuotationStatus;

  lineItems: QuotationLineItem[];
  subtotalUsd: number;
  discountPct?: number;
  totalUsd: number;
  currency: 'USD' | 'PEN';
  exchangeRate?: number;

  validityDays: number;               // por defecto 30
  expirationDate: string;

  paymentTerms?: string;
  warrantyYears?: number;
  deliveryWeeks?: number;

  technicalSpecs?: string;
  exclusions?: string;
  assumptions?: string;

  sentDate?: string;
  acceptedDate?: string;
  rejectedDate?: string;
  rejectionReason?: string;

  internalNotes?: string;
  createdBy: string;
}
```

---

## Estados del Embudo Comercial (Kanban Stages)

Implementar como columnas en `OpportunityKanban.tsx`. Orden estricto:

| ID | Etapa | Color hex | Ícono lucide |
|---|---|---|---|
| `prospect_registered` | Prospecto | `#64748b` | `UserPlus` |
| `initial_contact` | Contacto inicial | `#0ea5e9` | `Phone` |
| `initial_meeting` | Reunión comercial | `#6366f1` | `Users` |
| `site_visit_1` | Visita 1 presencial | `#8b5cf6` | `MapPin` |
| `preliminary_evaluation` | Evaluación preliminar | `#f59e0b` | `Search` |
| `quotation_requested` | Cotización solicitada | `#f97316` | `FileText` |
| `technical_visit_2` | Visita 2 técnica | `#ec4899` | `Wrench` |
| `deliverables_preparation` | Preparación entregables | `#14b8a6` | `FolderOpen` |
| `proposal_presented` | Propuesta presentada | `#22c55e` | `Send` |
| `bid_negotiation` | Licitación / negociación | `#eab308` | `Scale` |
| `awarded` | Adjudicada ✓ | `#16a34a` | `Trophy` |
| `not_awarded` | No adjudicada ✗ | `#dc2626` | `XCircle` |
| `in_execution` | En ejecución | `#0284c7` | `Zap` |
| `aftersales` | Postventa | `#7c3aed` | `HeartHandshake` |

---

## Módulos del Sistema

### Módulo 1 — Dashboard
**Ruta**: `/`

KPI Cards a mostrar:
- Total clientes activos
- Oportunidades abiertas
- Valor pipeline total (USD)
- Tasa de adjudicación (%) del trimestre
- Alertas críticas pendientes (número)

Gráficos con Recharts:
- `FunnelChart`: Distribución de oportunidades por etapa (BarChart horizontal)
- `BessAppChart`: PieChart de oportunidades por aplicación BESS
- `MonthlyPipeline`: LineChart de valor de oportunidades cerradas por mes

AlertsPanel: lista scrolleable de alertas urgentes (vencen en ≤3 días).

### Módulo 2 — Clientes
**Ruta**: `/clients`

`ClientList`: tabla con filtros por status, sector, región, aplicación BESS, responsable. Columnas: empresa, RUC, sector, aplicaciones BESS (badges), estado, # oportunidades, valor estimado, responsable, última interacción.

`ClientDetail` (`/clients/:id`): vista completa con tabs:
- **Resumen**: datos corporativos + contactos
- **Oportunidades**: lista filtrada de oportunidades de este cliente
- **Timeline**: `InteractionTimeline` — historial cronológico descendente de todas las interacciones
- **Visitas**: lista de visitas programadas y realizadas
- **Cotizaciones**: historial de cotizaciones

`ClientForm`: drawer lateral (Sheet de shadcn/ui) para crear/editar. Validar RUC (11 dígitos numéricos) con Zod.

**Regla multi-aplicación BESS**: un cliente puede tener entre 1 y 3 `bessApplications`. Renderizar como badges con colores:
- `peak_shaving` → badge azul: "Peak Shaving"
- `primary_frequency_regulation` → badge verde: "Reg. Frecuencia"
- `ramp_support_blackout` → badge naranja: "Rampas / Respaldo"

### Módulo 3 — Oportunidades
**Ruta**: `/opportunities`

Vista por defecto: **Kanban** (`OpportunityKanban`). Toggle para cambiar a tabla (`OpportunityTable`).

Kanban: columnas scrolleables horizontalmente, cards arrastrables (usar `@dnd-kit/core`). Al arrastrar una card entre columnas, actualizar `stage` del opportunity en el store.

Filtros: por cliente, aplicación BESS, responsable, fecha estimada de cierre, resultado.

`OpportunityDetail` (`/opportunities/:id`): vista con tabs:
- **Resumen**: datos comerciales y técnicos básicos
- **Entregables**: lista de deliverables con estado y fecha límite
- **Interacciones**: timeline filtrado a esta oportunidad
- **Cotizaciones**: cotizaciones vinculadas
- **Visitas**: visitas vinculadas

`OpportunityForm`: drawer para crear/editar. El campo `bessApplications` hereda del cliente pero puede sobrescribirse.

### Módulo 4 — Interacciones / Timeline
No tiene página propia. Se accede desde `ClientDetail` y `OpportunityDetail`.

`InteractionForm`: modal para registrar interacción. Campos: tipo, fecha, asunto, resumen, participantes (multi-input), resultado, próximos pasos, fecha próximos pasos.

`InteractionTimeline`: lista de cards ordenadas por fecha descendente. Cada card muestra: ícono por tipo, fecha, asunto, resumen truncado (expandible), autor.

### Módulo 5 — Visitas
**Ruta**: `/visits` (vista de calendario + lista)

`CalendarView`: calendario mensual mostrando visitas y próximas acciones. Usar `react-big-calendar` o implementar grid simple con date-fns.

`VisitForm`: modal para programar visita. Al guardar, ejecutar `scheduleCalendarReminder()` según provider configurado.

### Módulo 6 — Cotizaciones
**Ruta**: `/quotations`

Tabla de todas las cotizaciones con filtros. Código autogenerado: `COT-{AÑO}-Q{TRIMESTRE}-{SECUENCIA}`.

`QuotationForm`: pantalla completa (no drawer) para crear/editar cotización. Incluye tabla de line items editable en línea, cálculo automático de subtotal y total. Al cambiar de versión, crear nuevo documento con `parentQuotationId` apuntando al anterior.

### Módulo 7 — Calendario
**Ruta**: `/calendar`

Vista calendario mensual con todos los eventos: visitas, reuniones (de interacciones con fecha), vencimientos de cotizaciones, fechas límite de licitaciones.

`CalendarIntegrationSettings`: configurar provider (Teams o Notion) con campos de API key / OAuth.

### Módulo 8 — Configuración
**Ruta**: `/settings`

Tabs:
- **General**: nombre de la empresa, logo (upload base64 a localStorage), moneda por defecto, zona horaria
- **Usuarios**: lista de usuarios ficticios (no auth real) con nombre, email, rol
- **Alertas**: configurar días de anticipación por tipo de evento
- **Cierre Trimestral**: botón para ejecutar `quarterlyClose()`
- **Integraciones**: tokens para Teams (Microsoft Graph) y Notion API

---

## Roles de Usuario (sin autenticación real)

Implementar como selector de "perfil activo" en TopBar. El sistema recordará el último perfil en localStorage.

| Rol | Permisos |
|---|---|
| `admin` | CRUD completo en todos los módulos + cierre trimestral + configuración |
| `commercial_manager` | CRUD en clientes, oportunidades, interacciones, visitas, cotizaciones. No: configuración de sistema |
| `commercial_analyst` | Crear/editar interacciones y visitas. Ver todo. No: eliminar, cotizaciones, configuración |
| `viewer` | Solo lectura |

Implementar `useRoleGuard(permission)` hook. Renderizar botones de acción condicionalmente.

---

## Sistema de Alertas y Recordatorios

### `src/lib/alerts.ts`

Ejecutar `checkAlerts()` al montar la app y cada 4 horas. Retorna un array de `Alert[]`.

```typescript
interface Alert {
  id: string;
  type: 'visit_due' | 'quotation_expiring' | 'bid_deadline' | 'followup_overdue' | 'deliverable_due' | 'next_action_due';
  severity: 'critical' | 'warning' | 'info'; // ≤1 día: critical, 2-3: warning, 4-7: info
  title: string;
  description: string;
  daysRemaining: number;
  entityType: 'opportunity' | 'visit' | 'quotation' | 'interaction';
  entityId: string;
  clientId: string;
  linkTo: string;                     // ruta de navegación
}
```

Criterios de generación:

| Trigger | Critical | Warning | Info |
|---|---|---|---|
| Visita programada | 0-1 días | 2-3 días | 4-7 días |
| Vencimiento de cotización | 0-3 días | 4-7 días | 8-14 días |
| Fecha límite de licitación | 0-3 días | 4-7 días | 8-14 días |
| Fecha de próxima acción vencida | Vencida | — | — |
| Revisión de entregable | 0-1 días | 2-3 días | 4-5 días |
| Seguimiento comercial sin interacción en N días | >30 días | 21-30 días | 14-20 días |

Mostrar en:
1. `AlertBanner` en TopBar (badge con número de críticas)
2. `AlertsPanel` en Dashboard
3. Toasts sonner al abrir app si hay críticas

---

## Integración Calendarios

### Microsoft Teams / Outlook (`src/lib/teams.ts`)

```typescript
// Requiere token OAuth del usuario (almacenar en localStorage['teams_token'])
// Endpoint: https://graph.microsoft.com/v1.0/me/events

async function createTeamsEvent(params: {
  subject: string;
  start: string;         // ISO datetime
  end: string;
  location?: string;
  body?: string;
  reminderMinutesBeforeStart: number;
}): Promise<string>      // retorna eventId
```

Implementar también `deleteTeamsEvent(eventId)` y `updateTeamsEvent(eventId, params)`.

Flujo OAuth: abrir ventana popup a `https://login.microsoftonline.com/common/oauth2/v2.0/authorize` con scopes `Calendars.ReadWrite`. Capturar token en callback.

### Notion Calendar (`src/lib/notion.ts`)

```typescript
// Requiere Notion integration token (almacenar en localStorage['notion_token'])
// y database_id del calendario de Notion

async function createNotionCalendarEntry(params: {
  title: string;
  date: string;          // ISO date
  type: string;
  description?: string;
  clientName: string;
  daysBeforeReminder: number;
}): Promise<string>      // retorna page_id de Notion
```

Usar endpoint `https://api.notion.com/v1/pages` con `Authorization: Bearer {token}`.

---

## Exportación Excel y Cierre Trimestral

### `src/lib/excel.ts`

Función `exportCurrentData()`: genera un archivo `.xlsx` con las siguientes hojas:

| Hoja | Contenido |
|---|---|
| `Clientes` | Todos los campos de `Client[]` aplanados |
| `Oportunidades` | `Opportunity[]` con columna de nombre de cliente |
| `Interacciones` | `Interaction[]` ordenadas por fecha desc |
| `Visitas` | `Visit[]` |
| `Cotizaciones` | `Quotation[]` con líneas agregadas |
| `Dashboard` | KPIs del periodo: conteos, valores, tasas |

Función `quarterlyClose()`:
1. Llamar `exportCurrentData()` con nombre: `BESS-CRM_{AÑO}Q{TRIMESTRE}.xlsx`
2. Disparar descarga automática al browser
3. En localStorage:
   - Archivar oportunidades con `result !== 'open'` en clave `crm_archive_{year}Q{quarter}`
   - Mantener activos: clientes activos + oportunidades abiertas + últimas 5 interacciones por cliente
   - Resetear KPIs del trimestre
4. Mostrar modal de confirmación antes de ejecutar

Nombre de archivo para nuevo periodo: `BESS-CRM_{AÑO}Q{SIGUIENTE_TRIMESTRE}.xlsx`

---

## Persistencia en localStorage

### Claves de almacenamiento

```
crm_clients           → Client[]
crm_opportunities     → Opportunity[]
crm_interactions      → Interaction[]
crm_visits            → Visit[]
crm_quotations        → Quotation[]
crm_users             → User[]
crm_settings          → AppSettings
crm_alerts_cache      → Alert[]  (con timestamp, TTL 4h)
crm_archive_{year}Q{n}→ QuarterlyArchive
teams_token           → string
notion_token          → string
notion_database_id    → string
active_user_id        → string
```

### `src/store/persistence.ts`

```typescript
// Middleware Zustand para auto-save
const persist = <T>(config: StateCreator<T>, key: string) => ...

// Al inicializar: deserializar de localStorage
// Cada mutación de estado: serializar a localStorage
// Manejar try/catch por cuota excedida de localStorage (5MB típico)
```

---

## Diseño de Interfaz

### Paleta de colores

```css
/* Usar variables CSS en index.css */
--color-primary:        #0f172a;   /* azul marino oscuro (sidebar) */
--color-primary-light:  #1e293b;
--color-accent:         #0ea5e9;   /* azul eléctrico (acciones) */
--color-success:        #16a34a;
--color-warning:        #f59e0b;
--color-danger:         #dc2626;
--color-bess-peak:      #3b82f6;   /* Peak Shaving */
--color-bess-rpf:       #22c55e;   /* Regulación Frecuencia */
--color-bess-ramp:      #f97316;   /* Rampas / Respaldo */
```

### Layout global

```
┌─────────────────────────────────────────────┐
│  TopBar: logo | breadcrumb | alerts | user  │ ← 60px height
├──────────┬──────────────────────────────────┤
│          │                                  │
│ Sidebar  │   Main Content Area              │
│ 240px    │   (padding 24px, max-w full)     │
│ fixed    │                                  │
│          │                                  │
└──────────┴──────────────────────────────────┘
```

Sidebar items (con íconos lucide-react):
- Dashboard (`LayoutDashboard`)
- Clientes (`Building2`)
- Oportunidades (`TrendingUp`)
- Visitas (`CalendarCheck`)
- Cotizaciones (`FileSpreadsheet`)
- Calendario (`Calendar`)
- Configuración (`Settings`) — al fondo

### Principios UX

- Máximo 3 niveles de jerarquía visual en cualquier pantalla
- Usar `Sheet` (drawer lateral) para formularios de creación rápida, pantalla completa solo para cotizaciones
- Usar `Dialog` para confirmaciones destructivas (eliminar, cierre trimestral)
- Todas las tablas deben tener: búsqueda global, filtros por columna relevante, paginación (20 items/página), ordenación por columna
- Estado vacío (`EmptyState`) siempre con CTA para crear primer registro
- Loading states con `Skeleton` de shadcn/ui

---

## Orden de Implementación (fases para Claude Code)

### Fase 1 — Scaffolding y datos
1. Inicializar proyecto Vite + React + TypeScript + Tailwind
2. Instalar todas las dependencias del stack
3. Crear todos los archivos de tipos en `src/types/`
4. Implementar stores Zustand con persistencia en localStorage
5. Crear `src/lib/utils.ts` con helpers (nanoid wrapper, formatters de fecha/moneda)

### Fase 2 — Layout y navegación
6. Implementar `Sidebar.tsx` con navegación React Router
7. Implementar `TopBar.tsx` con selector de usuario activo
8. Implementar `AlertBanner.tsx`
9. Implementar `App.tsx` con rutas completas

### Fase 3 — Módulo Clientes
10. Implementar `ClientList.tsx` con tabla + filtros + búsqueda
11. Implementar `ClientForm.tsx` con validación Zod (RUC 11 dígitos, email, teléfono)
12. Implementar `ClientDetail.tsx` con tabs
13. Sembrar 5 clientes de ejemplo al inicializar (si localStorage vacío)

### Fase 4 — Módulo Oportunidades
14. Implementar `OpportunityKanban.tsx` con @dnd-kit
15. Implementar `OpportunityForm.tsx`
16. Implementar `OpportunityDetail.tsx` con tabs
17. Sembrar 8 oportunidades de ejemplo distribuidas en etapas

### Fase 5 — Interacciones y Visitas
18. Implementar `InteractionTimeline.tsx` y `InteractionForm.tsx`
19. Implementar `VisitList.tsx` y `VisitForm.tsx`
20. Sembrar interacciones y visitas de ejemplo

### Fase 6 — Cotizaciones
21. Implementar `QuotationList.tsx`
22. Implementar `QuotationForm.tsx` con tabla de line items editable
23. Autogenerar códigos COT-{AÑO}-Q{N}-{SEQ}

### Fase 7 — Dashboard y Alertas
24. Implementar motor de alertas `src/lib/alerts.ts`
25. Implementar `Dashboard.tsx` con KpiCards, FunnelChart, BessAppChart, MonthlyPipeline, AlertsPanel

### Fase 8 — Exportación y Configuración
26. Implementar `src/lib/excel.ts` con SheetJS
27. Implementar `QuarterlyClose.tsx` con modal de confirmación
28. Implementar `Settings.tsx` completo

### Fase 9 — Integraciones de Calendario
29. Implementar `src/lib/teams.ts` con OAuth flow
30. Implementar `src/lib/notion.ts`
31. Implementar `CalendarView.tsx` y `CalendarIntegrationSettings.tsx`

### Fase 10 — Pulido final
32. Implementar guards de roles en todos los módulos
33. Verificar responsividad (min-width: 1024px, layout colapsable en 768px)
34. Agregar toasts de confirmación en todas las operaciones CRUD
35. Revisar estados de carga y vacíos en todos los componentes
36. Optimizar bundle con `vite build`

---

## Datos de Ejemplo (Seed Data)

Al detectar `crm_clients` vacío en localStorage, llamar `seedDemoData()` que carga:

- 5 clientes peruanos con sectores: minería (Cusco), generación (Puno), industria (Lima), transmisión (Arequipa), minería (Cajamarca)
- Aplicaciones BESS variadas: algunos con 1, algunos con 2, uno con 3
- 8 oportunidades distribuidas en distintas etapas del funnel
- 15 interacciones con fechas en los últimos 90 días
- 3 cotizaciones en distintos estados
- 4 visitas (2 pasadas, 2 futuras próximas)
- 2 usuarios: "Carlos Mendoza" (admin) y "Ana Torres" (commercial_manager)

Los datos deben reflejar el contexto peruano: nombres de empresas reales del sector energético peruano (Kallpa Generación, SN Power, Orazul Energy, etc.), departamentos reales, valores en USD acordes al sector BESS en Perú (USD 500K–5M).

---

## Validaciones Obligatorias (Zod schemas)

```typescript
// RUC peruano
z.string().length(11).regex(/^\d{11}$/, "RUC debe tener 11 dígitos numéricos")

// Teléfono peruano
z.string().regex(/^(\+51)?[9]\d{8}$/, "Teléfono inválido")

// Email
z.string().email()

// Rango de probabilidad de oportunidad
z.number().min(0).max(100)

// Potencia BESS
z.number().positive().max(500_000) // máx 500 MW

// Energía BESS
z.number().positive().max(2_000_000) // máx 2 GWh
```

---

## Criterios de Aceptación

Al ejecutar `npm run build`, el output debe:
- Compilar sin errores TypeScript (`tsc --noEmit`)
- Generar bundle < 3MB (con code splitting por ruta)
- Abrir en Chrome/Edge/Brave/Opera sin extensiones ni servidor backend
- Funcionar completamente offline (sin internet) excepto las integraciones de calendario
- Persistir todos los datos al recargar la página
- Completar el cierre trimestral y descargar el Excel en < 5 segundos con 50 clientes

---

## Comandos de Desarrollo

```bash
# Inicializar proyecto
npm create vite@latest bess-crm -- --template react-ts
cd bess-crm
npm install

# Instalar dependencias
npm install tailwindcss @tailwindcss/vite
npm install zustand react-router-dom react-hook-form @hookform/resolvers zod
npm install date-fns xlsx @dnd-kit/core @dnd-kit/sortable
npm install recharts lucide-react sonner
npm install @radix-ui/react-dialog @radix-ui/react-tabs @radix-ui/react-sheet
npm install nanoid clsx tailwind-merge

# shadcn/ui
npx shadcn@latest init
npx shadcn@latest add button card badge dialog sheet tabs table input select textarea
npx shadcn@latest add skeleton toast dropdown-menu separator progress

# Desarrollo
npm run dev

# Build producción
npm run build
npm run preview
```
