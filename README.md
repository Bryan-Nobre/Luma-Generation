# Atomtech Pulse

<p align="center">
  <strong>Plataforma SaaS multi-tenant para recarga de veículos elétricos</strong><br/>
  Painel da operadora · Backend OCPP 1.6 · App do motorista
</p>

<p align="center">
  <img src="https://i.ibb.co/zhwYxf1V/landing-hero.png" alt="Landing Luma Gen" width="820" />
</p>

<p align="center">
  <em>Landing pública da marca Luma Gen — entrada comercial da plataforma.</em>
</p>

---

## Visão geral

**Atomtech Pulse** é um produto completo de gestão de recarga elétrica: da operação das estações ao pagamento e liberação da carga pelo motorista.

A plataforma conecta três superfícies:

| Superfície | Público | Função |
|------------|---------|--------|
| **Painel web (Luma Gen)** | Operadoras e administradores | Gestão de estações, sessões, clientes, tarifas, vouchers e financeiro |
| **Backend CSMS** | Carregadores + APIs internas | Protocolo **OCPP 1.6**, comandos remotos, autorização pré-paga e realtime |
| **App do motorista (Luma)** | Condutores de VE | Mapa, carteira, QR, recarga pré-paga e histórico pessoal |

O isolamento multi-tenant por `company_id` (com RLS no Postgres) é a base de segurança do sistema: cada operadora enxerga apenas os próprios dados operacionais; o motorista acessa o catálogo de estações e somente as próprias sessões e movimentações de carteira.

---

## Principais funcionalidades

### Painel da operadora
- Visão geral com métricas de período, mapa de estações e sessões recentes
- Cadastro e monitoramento de estações e conectores (status OCPP em tempo quase real)
- Histórico de sessões de carga e detalhes por estação
- Gestão de motoristas (`ev_users`), tarifas e vouchers/cupons
- Módulos financeiro, alertas, logs e configurações
- Papéis de acesso: `super_admin`, `company_admin`, `manager`, `viewer`
- Landing page, leads e analytics de aquisição (visão administrativa)

### App do motorista
- Cadastro em etapas e autenticação
- Mapa e lista de estações (catálogo multi-operadora, filtros e detalhe da estação)
- Favoritos, perfil, veículo e estatísticas de uso
- Carteira digital, vouchers e notificações
- Fluxo **pagar e liberar**: reserva de valor → RemoteStart → sessão ao vivo → resumo ao concluir
- Leitura de QR / código do carregador

### Backend de carga
- Servidor **OCPP 1.6** via WebSocket (`BootNotification`, `Heartbeat`, `Authorize`, `StatusNotification`, `StartTransaction`, `StopTransaction`, `MeterValues`, RemoteStart/Stop)
- Autorização pré-paga com hold na carteira e acerto ao finalizar a sessão
- Limite automático de energia conforme o valor reservado
- Canal realtime para atualização de status no painel

---

## Destaques técnicos

- **Multi-tenant de ponta a ponta** — isolamento por empresa no banco (RLS) e no backend
- **Dois públicos, um banco** — staff (`profiles`) e motoristas (`ev_users`) no mesmo Supabase, com fluxos de auth separados
- **Carga pré-paga real** — prepare → hold → RemoteStart → MeterValues → settle / estorno da sobra
- **Roaming simples** — motorista pode carregar em estação de outra operadora; a sessão fica vinculada à empresa da estação e ao histórico do motorista
- **Frontend modular** — dashboard e app mobile como SPAs React independentes, compartilhando o mesmo backend de dados e OCPP

---

## Stack tecnológica

| Camada | Tecnologias |
|--------|-------------|
| **Dashboard** | React 18, TypeScript, Vite 5, React Router, TanStack Query, Tailwind CSS 3, shadcn/ui (Radix), Leaflet, Recharts, Zod |
| **App motorista** | React 19, TypeScript, Vite 7, React Router, TanStack Query, Tailwind CSS 4, shadcn/ui (Radix), Leaflet, Recharts, Zod |
| **Backend** | Node.js, Express, WebSocket (`ws`), TypeScript |
| **Dados & Auth** | Supabase (Postgres, Auth, RLS, Realtime, Edge Functions, Storage) |
| **Protocolo de carga** | OCPP 1.6 (JSON over WebSocket) |
| **Qualidade** | ESLint, Vitest (dashboard), scripts de gate no backend |

---

## Principais interfaces

<p align="center">
  <img src="https://i.ibb.co/zhwYxf1V/landing-hero.png" alt="Landing Luma Gen" width="780" />
</p>
<p align="center"><em><strong>Landing Luma Gen</strong> — apresentação da rede, parceiros e chamada para o produto.</em></p>

<br/>

<p align="center">
  <img src="https://i.ibb.co/XfmbFjpR/dashboard-login.png" alt="Login do painel" width="520" />
</p>
<p align="center"><em><strong>Login do painel</strong> — acesso de operadoras e administradores à gestão de carregadores.</em></p>

<br/>

<p align="center">
  <img src="https://i.ibb.co/tPvGmB9x/mobile-welcome.png" alt="Boas-vindas do app Luma" width="320" />
  &nbsp;&nbsp;
  <img src="https://i.ibb.co/JRyN71Qg/mobile-login.png" alt="Login do app Luma" width="320" />
</p>
<p align="center"><em><strong>App Luma</strong> — boas-vindas e login do motorista (mapa, carteira e recarga após autenticação).</em></p>

Outras interfaces implementadas no código (sem captura pública neste repositório): overview operacional, estações/conectores, sessões, financeiro, vouchers, mapa com filtros, sessão de recarga ativa e tela de conclusão com kWh e valores.

---

## Arquitetura (resumo)

```text
┌────────────────┐   ┌────────────────┐   ┌────────────────────┐
│  Dashboard     │   │  App motorista │   │  Carregadores      │
│  (React SPA)   │   │  (React SPA)   │   │  OCPP 1.6          │
└───────┬────────┘   └───────┬────────┘   └─────────┬──────────┘
        │                    │                      │
        │         JWT / REST │                      │ WSS OCPP
        ▼                    ▼                      ▼
┌──────────────────────────────────┐     ┌─────────────────────┐
│         Supabase                 │◄────│  Backend Node       │
│  Postgres · Auth · RLS · EF      │     │  Express + OCPP +   │
│  Realtime · Storage              │     │  Realtime WS        │
└──────────────────────────────────┘     └─────────────────────┘
```

Detalhes de módulos, fluxos, segurança e decisões: **[docs/arquitetura.md](docs/arquitetura.md)**.

---

## Documentação

| Documento | Conteúdo |
|-----------|----------|
| [docs/arquitetura.md](docs/arquitetura.md) | Arquitetura técnica, módulos, auth, fluxos e escalabilidade |
| `docs/` (demais arquivos) | Planos de produto, OCPP, carga pré-paga e runbooks internos |

> O código-fonte deste repositório permanece **privado**. Este README descreve o produto para avaliação técnica e de portfólio.

---

## Status do projeto

| Área | Status |
|------|--------|
| Painel multi-tenant (estações, clientes, sessões, tarifas, vouchers) | Em produção / estável |
| Backend OCPP 1.6 + comandos remotos | Em produção / estável |
| Carga pré-paga (prepare → hold → RemoteStart → settle) | Implementada e validada em testes |
| App motorista (mapa, carteira, QR, sessão e resumo) | Implementado e em evolução |
| Integração de top-up com provedor de pagamento | Em andamento |
| Push nativo (FCM) e suíte E2E ampliada | Planejado |

**Tipo de repositório:** privado · **Natureza:** case de produto SaaS (EnergyTech / mobilidade elétrica)
