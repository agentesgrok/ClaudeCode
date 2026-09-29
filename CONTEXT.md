# Contexto: Paradigma AGI — Humano no Design

## Premissa
AGI já existe (meados 2026). GPT-6 Astra (OpenAI, set/2026) e declaração de Jensen Huang ("AGI has arrived") marcam a transição. Sem consenso formal sob definições estritas, mas a realidade operacional já reflete essa capacidade.

## Mudança de paradigma
"Human in the loop" (humano assina cada comando com "!") está obsoleto. Novo modelo: "human in the design" — humano desenha regras, AGI executa autonomamente.

- Humano define políticas, não aprova cada ação.
- `managed-settings.json` = constituição (allowlist explícita, default-deny, sem bypass permissivo).
- Skills/plugins = lei (procedimentos autônomos, sem supervisão por comando).
- MCP = sistema nervoso (conecta AGI ao mundo real).
- Marketplaces = distribuição (versionados, auto-atualizáveis, via MDM em frota).

## Implicações para este repo
- Todo plugin/skill projetado para operação autônoma.
- Hooks `PreToolUse` são o enforcement real (skills são só documentação).
- Fallback de gatekeeper externo: deny all, nunca allow all.
- Auditoria imutável e versionada, não em cache de plugin.
- Versionamento em `plugin.json` obrigatório a cada mudança.

## Plugins neste repo
- `macos-admin`: new-macos-user, global-statusline, global-bypass + statusline.sh
- `ssh-mcp`: MCP SSH global em /usr/local/bin/ssh-mcp-stdio
