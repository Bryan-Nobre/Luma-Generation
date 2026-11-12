# Arquitetura — visão de portfólio

Documento público e deliberadamente **alto nível**.  
Serve para mostrar o tipo de sistema e a presença de **OCPP** no produto.  
Não descreve implementação interna, APIs, schema, regras de segurança detalhadas nem infraestrutura.

---

## Visão geral

O sistema é uma plataforma de recarga de veículos elétricos com três frentes:

1. **App do motorista** — uso no dia a dia da recarga  
2. **Painel da operadora** — gestão e acompanhamento das estações  
3. **Camada de carga** — comunicação com carregadores via **OCPP 1.6**

```text
  App do motorista          Painel da operadora
         │                          │
         └────────────┬─────────────┘
                      │
               Plataforma de dados
                      │
               Camada OCPP 1.6
                      │
                 Carregadores
```

---

## Papel do OCPP no produto

O **OCPP 1.6** é o protocolo usado para falar com os carregadores. No produto, essa camada permite:

- manter a estação conectada ao sistema
- refletir o status do equipamento
- iniciar e encerrar sessões de carga
- acompanhar a energia entregue durante a recarga

Trabalhei nessa integração (software ↔ carregador), com testes em ambiente controlado e validação no fluxo do produto.

---

## Organização lógica (sem estrutura interna)

| Frente | Função para o usuário |
|--------|------------------------|
| Motorista | Localizar estação, liberar e acompanhar a recarga |
| Operadora | Ver estações/sessões e operar o negócio |
| Carga (OCPP) | Ligar o software aos charge points |

Frontends web em React/TypeScript. Backend em Node.js com WebSocket para o protocolo de carga. Dados e autenticação em Postgres (Supabase).

---

## O que este documento não inclui

De forma explícita, **não** são publicados:

- código-fonte
- rotas de API
- modelo completo de banco
- políticas de acesso detalhadas
- variáveis de ambiente ou credenciais
- topologia de deploy ou hosts
- fluxos internos passo a passo

Para conversas técnicas mais profundas (entrevista, pair review), o detalhe fica sob NDA ou demonstração controlada.

---

## Resumo

| Tema | Em uma frase |
|------|----------------|
| Produto | Plataforma de recarga EV (app + painel) |
| Protocolo | OCPP 1.6 |
| Objetivo deste repo | Portfólio — provar experiência com OCPP e produto real |
| Código | Proprietário, fora deste repositório |
