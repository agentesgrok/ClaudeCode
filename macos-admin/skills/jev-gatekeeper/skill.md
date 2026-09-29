---
name: jev-gatekeeper
description: Intercepta acoes sensiveis (sudo, escrita em /usr/local/bin e /Library/Application Support, managed-settings.json, delecao, chmod/chown), consulta o classificador Jev (TypeSafe AI) e so executa se allow com confianca alta; caso contrario bloqueia. Use quando pedirem gatekeeper Jev, aprovacao de comando sensivel via Jev, ou auditoria allow/deny antes de rodar sudo ou alterar arquivos de sistema.
---

# Jev gatekeeper

Antes de executar qualquer acao sensivel nesta maquina, monte o payload, consulte o endpoint Jev configurado em `config.json`, aplique o limiar de confianca e registre o resultado em `audit.log` nesta pasta da skill.

## Quando interceptar

Trate como sensivel se o comando ou o alvo bater em qualquer um destes criterios (veja tambem `config.json`):

- Comandos: `sudo`, `rm`, `mv`, `cp`, `chmod`, `chown`, `codesign` (e variantes com path absoluto)
- Escrita ou alteracao sob: `/usr/local/bin`, `/Library/Application Support`, `/etc`, `/var`, `/System`
- Arquivo `managed-settings.json` (qualquer path sob ClaudeCode)
- Delecao (`rm`, `unlink`, truncate destrutivo)
- Mudanca de dono/permissao (`chmod`, `chown`, `chgrp`)

Se a skill nao souber se algo e sensivel (comando novo, path ambiguo, flag pouco clara), **pesquise na web** (docs Claude Code, man pages, TypeSafe/Jev) **antes** de perguntar ao usuario.

## Fluxo

1. Detecte a acao candidata (comando completo + alvo + contexto curto: quem pediu, o que se pretende).
2. Monte o payload JSON:
   ```json
   {
     "comando": "<full shell command>",
     "alvo": "<primary path or resource>",
     "contexto": "<short why / who asked>"
   }
   ```
3. POST no `jev_endpoint` de `config.json` com o state = payload e as `questions` listadas la (Choice/Noul conforme API TypeSafe). Traduza a resposta tipada do Jev para o contrato abaixo.
4. Normalize a resposta para:
   ```json
   {
     "decisão": "allow" | "deny",
     "confiança": 0.0,
     "motivo": "texto"
   }
   ```
5. Decisao:
   - Se `confiança >= confidence_threshold` (default `0.85`) **e** `decisão == "allow"`: execute a acao e logue.
   - Caso contrario: **nao execute**, mostre o `motivo` ao usuario, logue.
6. **Fallback**: se o endpoint Jev falhar, timeout, HTTP nao-2xx, JSON invalido ou campos ausentes: **negar tudo** (`fallback_policy: deny`). Nao execute. Logue com `decisão=deny` e motivo explicando a falha.

## Prompt exato (ingles, com placeholders)

Use este texto como instrucoes / state textual enviado ao Jev (substitua os placeholders):

```
You are a security gatekeeper for a macOS admin agent. Decide whether the following action should be ALLOWED or DENIED.

Command: {{comando}}
Target: {{alvo}}
Context: {{contexto}}

Sensitive paths under policy: /usr/local/bin, /Library/Application Support, /etc, /var, /System
Sensitive commands under policy: sudo, rm, mv, cp, chmod, chown, codesign
Also treat managed-settings.json changes and destructive deletes as sensitive.

Answer these questions about the action:
1. Is this action destructive?
2. Does it affect system files?
3. Does it require sudo?
4. Is there a security risk?

Then return a single JSON object with exactly these keys (Portuguese key names as shown):
{"decisão":"allow"|"deny","confiança":<number 0..1>,"motivo":"<short English or Portuguese reason>"}

Rules:
- Prefer deny when unsure.
- Deny if the action can wipe, overwrite system config, escalate privileges unexpectedly, or install unsigned binaries without clear admin intent in Context.
- Allow only when the action is clearly intentional, scoped, and low residual risk given Context.
- confiança must be calibrated: use values >= 0.85 only when you are highly sure.
```

## Logging (`audit.log`)

Apos cada decisao (incluindo fallback), append uma linha em:

`macos-admin/skills/jev-gatekeeper/audit.log`

Formato (uma linha por evento, ISO-8601 local ou UTC marcado):

```
<timestamp>	<comando>	<decisão>	<confiança>	<motivo>
```

Nao commite `audit.log` se estiver vazio de ruido operacional; se o arquivo nascer com secrets, limpe antes do push. Preferir criar o arquivo localmente na primeira decisao.

## Config

Leia `config.json` nesta pasta para `jev_endpoint`, `confidence_threshold`, `fallback_policy`, `questions`, `sensitive_paths` e `sensitive_commands`. Nao hardcode valores diferentes sem atualizar o JSON.

## Notas

- Esta skill e procedimento para o agente: nao e um hook binario do shell. Em bypassPermissions o agente ainda deve consultar o gatekeeper antes das acoes listadas.
- Credenciais da API TypeSafe (se houver) nao ficam neste repo; use env / secret manager da sessao.
