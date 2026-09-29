# O que realmente prepara o sistema

Premissa deste repositorio: os agentes que rodam nesta maquina vao ficar
mais capazes e mais autonomos. O sistema deve continuar seguro e util **sem
depender de o agente obedecer instrucoes**. Nao precisamos decidir se e AGI
para nos preparar: os controles abaixo valem igual para um modelo de hoje e
para um que passe em qualquer definicao.

## Regra de arquitetura

**Enforcement e hook ou managed settings. Nunca skill, nunca CLAUDE.md.**
Skill e procedimento; CLAUDE.md e contexto. Os dois ajudam o agente a fazer
certo, nenhum garante.

| Vale com qualquer modelo | Depende do modelo cooperar |
|---|---|
| hook `PreToolUse` / `PostToolUse` no managed settings | skill que pede "consulte X antes" |
| `permissions.deny` no managed settings | CLAUDE.md dizendo "nunca faca Y" |
| conta sem admin + `sudo` na mao de uma pessoa | "o agente sabe que nao deve" |
| log root-only em `/var/log` | `audit.log` que o proprio agente escreve na pasta dele |
| managed settings (ninguem sobrescreve) | `~/.claude/settings.json` de cada usuario |

Foi por isso que a skill `jev-gatekeeper` saiu do `macos-admin`: gatekeeper
em forma de texto nao intercepta nada.

## Os controles

1. **Hooks em vez de skills** para tudo que precisa acontecer sempre.
   `PreToolUse` no `managed-settings.json` roda antes de cada comando, em
   qualquer modo, e ninguem desliga.
2. **Trilha de auditoria fora do alcance do agente.** Hook `PostToolUse`
   gravando comando, usuario e hora em `/var/log/claude-code/audit.log`,
   root-only. Sem isso nao ha como saber o que aconteceu.
3. **Deny list fixa** em `permissions.deny` para o que nunca faz sentido,
   independente de contexto. Vale mesmo em bypass.
4. **Separacao por conta.** Agente roda em conta sem admin. O que precisa de
   root passa por uma pessoa: o fluxo "agente prepara, pessoa roda com `!`"
   que todas as skills seguem. Isso escala com a capacidade do modelo;
   bypass irrestrito em conta admin nao.
5. **Reversivel e versionado.** Managed settings, hooks e scripts neste git,
   com backup datado antes de cada troca.
6. **Servico externo so com fallback decidido antes.** Aprovacao remota e
   `PreToolUse` com servico real. `fallback: deny` trava a administracao
   quando o servico cai; `fallback: allow` nao protege nada. Escolher
   conscientemente, nao herdar.

## Estado (2026-09-29)

Feito, via managed settings: status line global, `bypassPermissions` para
todas as contas, marketplace `box-admin` com `macos-admin` e `ssh-mcp`
forcados, MCP `ssh` global com binario root-owned em `/usr/local/bin`.

Falta, nesta ordem:

- [ ] Hook `PostToolUse` de auditoria em `/var/log/claude-code/audit.log`.
- [ ] `permissions.deny`: apagar `/Library/Application Support/ClaudeCode`,
      editar `/etc/sudoers`, `rm -rf` em home alheio, alterar
      `managed-settings.json` por dentro de uma sessao.
- [ ] Contas de agente (`svc`, `agent`) sem admin, revisadas.

O bypass global ja instalado e compativel com tudo isso; ele so deixa de ser
o unico controle. Alias em `/etc/zshrc` e wrapper `claude-bypass` sao
conveniencia, nao controle.
