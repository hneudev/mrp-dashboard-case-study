# MRP Dashboard — Enterprise Production Planning

> A case study and reference implementation of an enterprise Manufacturing Resource Planning (MRP) frontend, built for a concrete pipe manufacturer. Originally delivered at Softtek (2023–2024) and reconstructed here with mock data.

![Next.js](https://img.shields.io/badge/Next.js-14-black?logo=next.js)
![TypeScript](https://img.shields.io/badge/TypeScript-5-blue?logo=typescript)
![GraphQL](https://img.shields.io/badge/API-GraphQL-e10098?logo=graphql)
![Redux](https://img.shields.io/badge/State-Redux_Toolkit-764abc?logo=redux)
![MUI](https://img.shields.io/badge/UI-Material_UI-007fff?logo=mui)

**[Live Demo →](https://mrp-dashboard.vercel.app)** · *All data is mock — no client information is exposed*

---

## The Problem

A concrete pipe manufacturer needed to replace a legacy spreadsheet-based production planning process. Planners were manually cross-referencing sales orders, raw material inventory, and machine capacity across multiple Excel files — a process prone to human error and unable to scale with growing order volume.

The goal was a web-based MRP system that could:

- Ingest sales orders and automatically generate production plans
- Surface inventory shortfalls and machine scheduling conflicts in real time
- Integrate with existing sales and production systems via API
- Be used daily by non-technical production planners, not developers

---

## My Role

I was the sole frontend engineer on a cross-functional team of three (backend, QA, and myself). I owned the entire frontend — from architecture decisions to component implementation, state management design, and stakeholder demos. I worked directly with production planners to validate UX decisions, and with the backend engineer to co-design the GraphQL schema.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Next.js Frontend (this repo)             │
│                                                             │
│   ┌─────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│   │  Pages /    │  │  Redux Store │  │  Apollo Client   │  │
│   │  App Router │  │  (UI state)  │  │  (server state)  │  │
│   └──────┬──────┘  └──────────────┘  └────────┬─────────┘  │
└──────────┼──────────────────────────────────────┼───────────┘
           │                                      │
           │                          ┌───────────▼──────────┐
           │                          │   GraphQL API        │
           │                          │   (Node / Express)   │
           │                          └──────┬───────┬───────┘
           │                                 │       │
           │                    ┌────────────▼┐  ┌───▼───────────┐
           │                    │  Sales DB   │  │  Production   │
           │                    │  (Orders)   │  │  DB (Plans)   │
           └────────────────────┴─────────────┴──┴───────────────┘
```

### State management split

One of the key architectural decisions was separating server state from UI state:

| Concern | Tool | Reason |
|---|---|---|
| Server data (orders, inventory, plans) | Apollo Client cache | Handles caching, re-fetching, optimistic updates automatically |
| UI state (selected rows, modal state, filters) | Redux Toolkit | Predictable, debuggable, easy to snapshot for QA |
| Form state | React Hook Form | Reduces re-renders on complex multi-step forms |

This split prevented the common pattern of storing server data in Redux and manually managing cache invalidation — a source of bugs in many enterprise frontends.

---

## Architecture Decision Records

### ADR-001 — GraphQL over REST

**Context:** The backend team had experience with REST. I advocated for GraphQL.

**Decision:** GraphQL with Apollo Client.

**Reasons:**
- Production planning screens require data from multiple entities simultaneously (orders + inventory + machine schedules). GraphQL allowed fetching exactly what each screen needed in one round trip.
- Apollo's normalized cache meant that updating a production plan on one screen automatically reflected on the dashboard without a manual refetch.
- Strongly typed schema aligned with our TypeScript-first frontend — types were generated from the schema using `graphql-codegen`.

**Trade-off:** Steeper learning curve for the backend engineer. We spent one sprint co-designing the schema before either side started implementation — an investment that paid off in fewer breaking changes later.

---

### ADR-002 — Material UI over a custom component library

**Context:** The team had a tight deadline. The client's design requirements were functional, not brand-driven.

**Decision:** Material UI (MUI) v5 with a custom theme.

**Reasons:**
- MUI's `DataGrid` component handled complex table requirements (sorting, filtering, inline editing, virtualization for large datasets) that would have taken weeks to build from scratch.
- The component library covers every UI pattern an enterprise dashboard needs — date pickers, autocomplete, dialogs — with full accessibility compliance.
- A custom MUI theme gave the client a distinctive look without custom component work.

**Trade-off:** MUI's bundle size is significant. We mitigated this with tree-shaking and lazy loading for heavy components (DataGrid, DatePicker). For a consumer product with strict performance budgets, a lighter alternative (shadcn/ui, Radix) would be preferable.

---

### ADR-003 — Next.js Pages Router (not App Router)

**Context:** This project was built in mid-2023 when the App Router was newly stable and had limited ecosystem support.

**Decision:** Pages Router.

**Reasons:**
- Apollo Client's SSR integration was mature and well-documented for Pages Router. App Router support was experimental at the time.
- The team needed to move fast — adopting a new paradigm mid-project was a risk we couldn't afford.

**What I'd do differently today:** The App Router's React Server Components would meaningfully improve this application. Heavy dashboard pages that fetch large datasets could become Server Components, eliminating client-side loading states and reducing the Apollo Client bundle sent to the browser. I'd migrate to App Router if rebuilding this today.

---

## Key Implementation Challenges

### 1. Dynamic production planning forms

The core workflow required planners to create a production plan by filling in a multi-step form that adapted based on previous inputs — selecting a product type changed which machine types were available, which changed which shifts were valid.

**Solution:** React Hook Form with `watch` and `useEffect` to drive dependent field resets. Zod schemas for validation, colocated with form components.

```typescript
// Dependent field reset pattern
const productType = watch('productType')

useEffect(() => {
  resetField('machineId')
  resetField('shiftId')
}, [productType, resetField])
```

### 2. Real-time inventory conflict detection

When a planner submitted a production plan, the system needed to immediately surface whether the required raw materials were available — without a full page reload.

**Solution:** Apollo optimistic updates + a conflict detection query triggered on form submission. The UI showed a "checking inventory..." state, then either confirmed the plan or surfaced specific shortfalls with line-item detail.

### 3. Large dataset performance

The orders table could display thousands of rows. Standard rendering caused visible lag on scroll.

**Solution:** MUI DataGrid's built-in row virtualization, combined with server-side pagination via GraphQL cursor-based pagination. Only the visible rows are in the DOM at any time.

---

## What I'd Improve

These are honest reflections, not excuses. Every item here represents something I'd do differently with more time or if rebuilding today:

1. **Migrate to App Router + RSC** — The biggest structural improvement. Server Components would eliminate loading skeletons on the dashboard's initial render and reduce JS sent to the client.

2. **Replace Apollo with TanStack Query + a typed REST client** — Apollo is powerful but heavy. For a project this size, TanStack Query with a `graphql-request` transport would give 80% of the benefit at half the bundle cost.

3. **Add E2E tests for the planning workflow** — The multi-step form was the most business-critical path and the hardest to manually test. Playwright tests covering the happy path and key error states would have saved hours of QA time per sprint.

4. **Storybook for the component library** — We built reusable components but they lived only in the codebase. Visual documentation would have helped onboard new team members and improved design-dev communication.

5. **Proper error boundary strategy** — Error handling was inconsistent — some components had try/catch, others relied on Apollo's error state. A top-level error boundary strategy with specific fallback UIs per section would make failure modes more predictable.

---

## Running the Demo

This repo uses [MSW (Mock Service Worker)](https://mswjs.io/) to intercept GraphQL queries and return realistic mock data. No backend required.

### Setup

```bash
git clone https://github.com/hneudev/mrp-dashboard-case-study.git
cd mrp-dashboard-case-study
npm install
npm run dev
```

The mock API starts automatically in development. To explore the mock data, see `src/mocks/handlers/`.

### Credentials (mock)

| Role | Email | Password |
|---|---|---|
| Production Planner | planner@demo.com | demo1234 |
| Supervisor | supervisor@demo.com | demo1234 |

---

## Project Structure

```
src/
├── pages/                  # Next.js Pages Router
│   ├── dashboard/          # Main planning dashboard
│   ├── orders/             # Sales orders list + detail
│   ├── plans/              # Production plans (create / edit / view)
│   └── inventory/          # Raw material inventory view
├── components/
│   ├── ui/                 # Base components (themed MUI wrappers)
│   ├── plans/              # Plan-specific components
│   ├── orders/             # Order-specific components
│   └── layout/             # Shell, Sidebar, Header
├── lib/
│   ├── apollo/             # Apollo Client setup, cache config
│   ├── graphql/            # Generated types + query definitions
│   └── store/              # Redux Toolkit slices
├── mocks/
│   ├── handlers/           # MSW GraphQL handlers
│   └── data/               # Seed data factories
└── types/                  # Shared TypeScript types
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| Framework | Next.js (Pages Router) |
| Language | TypeScript 5 |
| UI Library | Material UI v5 |
| API | GraphQL (Apollo Client) |
| State | Redux Toolkit + React Hook Form |
| Mock API | MSW (Mock Service Worker) |
| Type generation | graphql-codegen |
| Deployment | Vercel |

---

## Lessons Learned

**Design for the actual user, not the ideal user.** Production planners used this tool for 8+ hours a day. Small UX decisions — keyboard navigation in the DataGrid, sticky table headers, confirmation dialogs before destructive actions — mattered more to daily satisfaction than any architectural choice. I learned to weight user interviews as heavily as technical decisions.

**Co-design the API schema early.** The sprint we spent co-designing the GraphQL schema with the backend engineer was the highest-leverage work of the project. Every hour spent there saved three hours of "we need to add a field" back-and-forth later.

**Enterprise clients need change justification, not just change.** Every significant architectural decision I made — choosing GraphQL, structuring the Redux store a certain way — needed to be explained in business terms to stakeholders who weren't developers. Writing ADRs wasn't just good engineering hygiene; it was how I got decisions approved.

---

*Built by [Héctor Neudert](https://hneu.dev) · [LinkedIn](https://linkedin.com/in/hneudert) · [GitHub](https://github.com/hneudev)*  
*Original project delivered at [Softtek](https://softtek.com) — this repo uses entirely synthetic data.*
