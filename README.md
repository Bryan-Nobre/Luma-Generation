# Luma Generation

<p align="center">
  <strong>Case de portfólio — plataforma de recarga para veículos elétricos</strong><br/>
  Experiência do motorista · Painel da operadora · Integração com carregadores via <strong>OCPP 1.6</strong>
</p>

<p align="center">
  <img src="https://i.ibb.co/zhwYxf1V/landing-hero.png" alt="Landing Luma Gen" width="820" />
</p>

<p align="center">
  <em>Landing da marca Luma Gen.</em>
</p>

---

## Sobre o projeto

**Luma Generation** é um produto de mobilidade elétrica em que atuei no desenvolvimento de software: da interface do motorista à operação das estações e à comunicação com carregadores.

Este repositório é um **case de portfólio**. O objetivo é apresentar o tipo de produto e as tecnologias com as quais trabalho — em especial o protocolo **OCPP** para controle de estações de recarga.  
**O código-fonte do produto não está publicado aqui.**

---

## O que o produto cobre

- **App do motorista** — encontrar estações, iniciar recarga e acompanhar a sessão
- **Painel da operadora** — acompanhar estações, sessões e a operação do dia a dia
- **Comunicação com carregadores** — integração com charge points usando **OCPP 1.6** (WebSocket)

---

## Destaque técnico: OCPP

Trabalhei na integração de software com carregadores reais e simulados através do protocolo **OCPP 1.6**, incluindo o ciclo de vida da conexão da estação e o acompanhamento de sessões de carga no produto.

Isso envolve, em alto nível:

- comunicação contínua com o carregador
- sincronização de status da estação/conectores
- início e fim de sessões de recarga
- leitura de energia durante a carga

> Detalhes de implementação, endpoints, schema e infraestrutura **não são documentados publicamente**.

---

## Stack (visão geral)

| Área | Tecnologias |
|------|-------------|
| Frontends | React, TypeScript, Vite, Tailwind CSS |
| Backend | Node.js, TypeScript, WebSocket |
| Dados & Auth | Postgres (Supabase) |
| Protocolo de carga | **OCPP 1.6** |

---

## Interfaces

<p align="center">
  <img src="https://i.ibb.co/zhwYxf1V/landing-hero.png" alt="Landing Luma Gen" width="780" />
</p>
<p align="center"><em><strong>Landing</strong> — apresentação do produto.</em></p>

<br/>

<p align="center">
  <img src="https://i.ibb.co/XfmbFjpR/dashboard-login.png" alt="Login do painel" width="520" />
</p>
<p align="center"><em><strong>Painel</strong> — acesso da operadora.</em></p>

<br/>

<p align="center">
  <img src="https://i.ibb.co/tPvGmB9x/mobile-welcome.png" alt="Boas-vindas do app" width="320" />
  &nbsp;&nbsp;
  <img src="https://i.ibb.co/JRyN71Qg/mobile-login.png" alt="Login do app" width="320" />
</p>
<p align="center"><em><strong>App do motorista</strong> — entrada e autenticação.</em></p>

---

## Documentação

Visão técnica resumida (sem detalhes sensíveis): **[arquitetura.md](arquitetura.md)**

---

## Status

Produto real em uso operacional, com evolução contínua.  
Este espaço no GitHub serve apenas como **vitrine de portfólio**.

**Código-fonte:** proprietário · **não incluído neste repositório**
