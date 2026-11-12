# Arquitetura — Atomtech Pulse

Documentação técnica do monorepo. Descreve apenas o que está implementado no código e nas migrations; não substitui os planos de produto em `docs/`.

---

## 1. Visão geral da arquitetura

A plataforma é um **SaaS multi-tenant** composto por:

1. **Dashboard web** (`src/`) — SPA React para staff da operadora e super admin  
2. **App do motorista** (`app-para-mobile/pulse-charge-interface/`) — SPA React (PWA-like)  
3. **Backend CSMS** (`backend/`) — API HTTP + servidor **OCPP 1.6** + canal realtime WebSocket  
4. **Supabase** (`supabase/migrations/`) — Postgres, Auth, RLS, Realtime, Storage e Edge Functions  

```text
                    ┌─────────────────────────────┐
                    │   Supabase (mesmo projeto)  │
                    │ Postgres · Auth · Storage   │
                    │ Edge Functions · Realtime   │
                    └──────────────┬──────────────┘
                                   │
           ┌───────────────────────┼───────────────────────┐
           │                       │                       │
    ┌──────▼──────┐         ┌──────▼──────┐         ┌───────▼───────┐
    │  Dashboard  │         │ App mobile  │         │  backend/     │
    │  profiles   │         │  ev_users   │         │  OCPP + API   │
    │  companies  │         │  carteira   │         │  realtime WS  │
    │  estações   │         │  mapa/carga │         │               │
    └─────────────┘         └─────────────┘         └───────────────┘
                                                           ▲
                                                           │ OCPP 1.6
                                                    ┌──────┴──────┐
                                                    │ Carregadores│
                                                    └─────────────┘
```

Regra de ouro do produto: **nunca confiar só no frontend**; isolamento por `company_id` + RLS; motorista só lê o catálogo público e os próprios dados.

---

## 2. Organização dos módulos

| Pasta | Responsabilidade |
|-------|------------------|
| `src/` | Painel Luma Gen: páginas, componentes, hooks, serviços Supabase, landing |
| `backend/src/` | Express, handlers OCPP, serviços de estação/transação/carteira/carga, realtime |
| `supabase/migrations/` | Schema, RLS, RPCs, buckets e policies versionadas |
| `app-para-mobile/pulse-charge-interface/` | App Luma (projeto aninhado, publicação independente) |
| `docs/` | Arquitetura, planos de produto e runbooks |
| `scripts/` | Utilitários de tipos, assets e manutenção (não fazem parte do runtime de produção) |

O dashboard e o app mobile são **projetos frontend separados**, apontando para o **mesmo Supabase**. O backend Node é o único processo que fala OCPP com os carregadores e usa a service role do Supabase para persistência operacional.

---

## 3. Estrutura do frontend

### 3.1 Dashboard (`src/`)

- **Roteamento:** `App.tsx` (público: landing, privacidade) e `AppAuthenticated.tsx` (login, cadastro, `/admin/*`, `/dashboard/*`)
- **Estado remoto:** TanStack Query + cliente Supabase
- **UI:** Tailwind 3 + shadcn/ui (Radix) + Sonner
- **Mapas / gráficos:** Leaflet / React-Leaflet, Recharts
- **Exportação:** jsPDF e XLSX em listagens (ex.: estações)

Principais rotas autenticadas:

| Área | Exemplos de rotas |
|------|-------------------|
| Admin (super) | `/admin/companies`, leads, landing analytics |
| Operação | `/dashboard`, `/dashboard/stations`, `/dashboard/station/:id`, `/dashboard/sessions` |
| Negócio | `/dashboard/users`, `/dashboard/vouchers`, `/dashboard/financial`, `/dashboard/analytics` |
| Ops | `/dashboard/alerts`, `/dashboard/logs`, `/dashboard/push`, `/dashboard/settings` |

Autenticação de staff usa a tabela **`profiles`** (role + `company_id`). Super admin pode trocar o contexto de empresa no painel.

### 3.2 App do motorista (`app-para-mobile/pulse-charge-interface/`)

- **Roteamento:** React Router — rotas guest (`/`, `/login`, `/cadastro`) e protegidas (`/mapa`, `/qr`, `/recarga`, `/carteira`, …)
- **Organização:** feature folders (`features/auth`, `stations`, `charging`, `wallet`, `profile`, …)
- **Acesso a dados:** repositórios em `services/supabase/*` + contratos em `services/contracts/*`
- **Carga:** chama a API do backend (`/api/charge/*` e OCPP remoto) com JWT do motorista

Telas principais: boas-vindas, login, wizard de cadastro, mapa/lista/filtros, detalhe da estação, QR/pré-pago, sessão ativa, resumo da recarga, carteira, favoritos, perfil e notificações.

---

## 4. Estrutura do backend

Entrada: `backend/src/server.ts`.

| Camada | Conteúdo |
|--------|----------|
| **HTTP** | `api/stations.routes`, `ocppRemote.routes`, `charge.routes`, `payments.routes`, `logs.routes` |
| **OCPP** | `ocpp/ocppServer`, `ocppHandlers`, `ocppOutgoing`, sweeps de sessão/estação |
| **Serviços** | estações, conectores, transações, charge session/authorization, wallet hold, prepaid cap, vouchers, alertas, push enqueue, retenção de logs |
| **Realtime** | `realtime/websocketServer` — eventos para o painel |
| **Banco** | cliente Supabase com service role (`database/supabaseClient`) |

Health checks: `GET /health` e `GET /health/details` (flags de runtime, sem exposição de segredos).

### Protocolo OCPP (implementado)

Mensagens tratadas no CSMS incluem:

- Do carregador: `BootNotification`, `Heartbeat`, `Authorize`, `StatusNotification`, `StartTransaction`, `StopTransaction`, `MeterValues`
- Para o carregador: `RemoteStartTransaction`, `RemoteStopTransaction` (e demais comandos remotos expostos nas rotas autenticadas do painel)

Conectores são criados/descobertos conforme o carregador reporta (`ensureConnectorExists` / boot com conectores iniciais). A estação é localizada pelo `charge_point_id` da URL WebSocket.

---

## 5. Banco de dados e relacionamentos

O schema evolui por migrations em `supabase/migrations/`. Entidades centrais:

```text
companies
   ├── profiles          (staff do painel)
   ├── stations
   │      └── connectors
   │      └── transactions  (também ligadas a ev_users quando a sessão é do app)
   ├── tariffs
   ├── vouchers
   └── ev_users          (motoristas)
          ├── ev_user_wallets / ev_user_wallet_transactions
          ├── ev_user_favorites
          ├── ev_user_vouchers
          ├── ev_user_notifications / device_tokens
          └── charge_authorizations  (pré-pago por sessão)

ocpp_command_logs
station_alerts
lead_submissions
```

### Relacionamentos relevantes

- **`stations.company_id` → `companies`** — isolamento operacional do painel  
- **`connectors.station_id` → `stations`** — bocas do carregador  
- **`transactions`** — sessão OCPP; `ev_user_id` opcional vincula o motorista  
- **`charge_authorizations`** — hold pré-pago (`amount_brl`, `id_tag` no formato `EV-{id}`, status `pending|ready|in_use|consumed|expired|cancelled`)  
- **`tariffs`** — preço por kWh por operadora (RPCs como `get_current_tariff` / `get_tariff_for_company` / `get_tariffs_for_companies`)  
- **Views** — ex.: `v_sessions_list` para listagens do app  

Transações não carregam `company_id` direto: o isolamento do painel passa por `station_id` → estação da empresa (avaliar coluna própria apenas se volume exigir — nota de desempenho no plano do produto).

---

## 6. Autenticação e permissões

### Dois públicos

| Público | Identidade | Tabela | Observação |
|---------|------------|--------|------------|
| Staff do painel | Supabase Auth + `profiles` | `profiles.role`, `profiles.company_id` | Roles: `super_admin`, `company_admin`, `manager`, `viewer` |
| Motorista | Supabase Auth + `ev_users` | `ev_users.auth_user_id`, `company_id` | Metadata de driver; cadastro via Edge Functions (`signup-ev-driver`, etc.) |

Motorista **não** deve receber `profiles` com role de painel. Staff e motorista compartilham Auth, mas políticas e tabelas são distintas.

### RBAC (painel)

Escrita tipicamente restrita (policies de migration): criar/editar para `company_admin` + `manager`; exclusões sensíveis para `company_admin`; `super_admin` com acesso amplo. `viewer` é leitura.

### API de carga (app → backend)

Rotas `/api/charge/*` exigem `Authorization: Bearer <JWT>`, resolvem o `ev_users` pelo `auth_user_id` e só então executam prepare/confirm/cancel. Há bypass exclusivo para gates de teste locais (nunca habilitado em produção).

### Funções auxiliares no Postgres

Exemplos presentes no schema tipado: `current_ev_user_id`, `current_ev_company_id`, `same_company`, `is_super_admin`, `list_company_catalog`, RPCs de carteira e tarifas.

---

## 7. Segurança e validações

- **RLS** em tabelas sensíveis (`stations`, `transactions`, `ev_users`, carteira, autorizações, etc.)
- **Filtro por empresa** obrigatório no acesso staff; vazamento cross-tenant é tratado como erro grave de produto
- **Validação de payload** no backend (limites de body JSON, checagem de IDs e valores de carga)
- **Zod / React Hook Form** em formulários dos frontends
- **Segredos** apenas em variáveis de ambiente (URL/keys Supabase, service role no backend, tokens de pagamento quando configurados) — nunca commitados
- **Carteira:** hold antes da carga; settle/refund no fim; saldo insuficiente bloqueia prepare/confirm
- **Authorize / StartTransaction** consultam autorização pré-paga válida (não aceitam início “solto” no fluxo de produto)

---

## 8. Integrações externas

| Integração | Uso no produto |
|------------|----------------|
| **Supabase Auth / Postgres / Realtime / Storage** | Identidade, dados, fotos de estação, eventos |
| **Supabase Edge Functions** | Cadastro/completação de motorista, listagem de operadoras no signup, criação de usuário no painel |
| **OCPP 1.6 (WebSocket)** | Comunicação com charge points físicos ou simuladores |
| **ViaCEP** (app) | Autopreenchimento de endereço no cadastro |
| **WhatsApp** (app) | Link de suporte configurável (`VITE_SUPPORT_WHATSAPP`) |
| **Provedor de pagamento** | Rotas/estrutura de top-up presentes; integração real de produção em evolução |

---

## 9. Fluxos principais

### 9.1 Recarga pré-paga (motorista)

```text
1. App: escolhe estação/conector e valor (R$)
2. POST /api/charge/prepare  → charge_authorization (pending) + valida saldo
3. POST /api/charge/confirm  → hold na carteira → status ready → RemoteStart (idTag EV-{id})
4. CP: Authorize / StartTransaction (só com auth válida)
5. MeterValues atualizam energia; prepaidCap pode RemoteStop ao atingir ~95% do hold
6. StopTransaction → settle (custo + estorno da sobra) → status consumed
7. App: tela de sessão ao vivo e depois resumo (kWh, reservado, cobrado, devolvido)
```

### 9.2 Estação online (OCPP)

```text
1. Dashboard cadastra station com charge_point_id
2. CP abre WSS /ocpp/{chargePointId} (subprotocolo ocpp1.6)
3. BootNotification → station online + conectores iniciais
4. StatusNotification por boca → cria/atualiza connectors
5. Painel recebe eventos via canal realtime do backend
```

### 9.3 Operação no painel

Staff autenticado consulta/escreve via Supabase com RLS. Comandos remotos (start/stop, disponibilidade) passam pelo backend autenticado (`/api/ocpp/...`), que envia CALL OCPP ao WebSocket do charge point conectado.

---

## 10. Decisões arquiteturais

| Decisão | Motivo / efeito |
|---------|-----------------|
| **Dois frontends, um banco** | Separar UX de operadora e motorista sem duplicar domínio de dados |
| **Auth staff ≠ auth motorista** | Evita misturar papéis e policies; cadastros e Edge Functions distintos |
| **Backend Node como CSMS** | OCPP exige processo stateful (WS) fora do serverless puro |
| **Pré-pago com hold** | Plugar o cabo não inicia energia; pagamento/autorização libera RemoteStart |
| **idTag `EV-{ev_user_id}`** | Liga sessão OCPP ao motorista do app |
| **Catálogo multi-operadora no app** | Motorista vê estações habilitadas; roaming simples com tarifa/company da estação |
| **RLS como última linha** | Frontend pode falhar; o Postgres ainda isola tenants |
| **Gates de teste no backend** | Scripts `gate:*` validam auth, wallet, prepaid, catálogo e RLS sem depender só de UI |

---

## 11. Possibilidades de escalabilidade

Já observadas no plano de desempenho e no código:

1. **Índices e RPCs em lote** — ex.: tarifas por várias empresas em uma chamada; índices parciais de catálogo  
2. **Realtime seletivo** — painel filtra por empresa/estações carregadas; debounce e pausa em aba oculta  
3. **Retenção de `ocpp_command_logs`** — limpeza periódica configurável (dados financeiros não são apagados)  
4. **Sweeps OCPP** — intervalo ajustável para marcar estações stale/offline  
5. **Próximos passos naturais** (quando o volume crescer):
   - `company_id` denormalizado em `transactions` + índices
   - code-split dos bundles dos frontends
   - filtragem geográfica de estações no app (~centenas+)
   - filas/workers para push e webhooks de pagamento

O desenho atual (CSMS único + Postgres + RLS) cobre o estágio de produto com dezenas de estações e volume moderado de sessões; a separação clara entre API de carga, OCPP e frontends permite evoluir cada peça de forma independente.

---

## Referências internas

| Documento | Assunto |
|-----------|---------|
| [../README.md](../README.md) | Apresentação de portfólio |
| `PLANO.md` | Status de produto e desempenho |
| `PLANO_CARGA_PREPAGA.md` | Detalhe do fluxo pré-pago |
| `OCPP_BACKEND_E_CONECTORES.md` | Conectores e comportamento OCPP |
| `STACK_DASHBOARD_VS_MOBILE.md` | Comparativo histórico das stacks (pode estar parcialmente desatualizado frente ao app atual) |

---

*Última revisão baseada no estado do monorepo (dashboard, backend, migrations e app mobile) — sem exposição de credenciais.*
