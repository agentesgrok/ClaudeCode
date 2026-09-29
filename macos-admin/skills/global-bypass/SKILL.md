---
name: global-bypass
description: Faz o Claude Code abrir direto em modo bypassPermissions para todos os usuarios desta maquina, via managed settings em /Library/Application Support/ClaudeCode, com opcionais de alias global no zsh e wrapper claude-bypass. Use quando pedirem para digitar claude e ja entrar sem pedir permissao, para todo mundo, global, ou para reverter/proibir isso.
---

# Bypass global (todos os usuarios)

Objetivo: qualquer pessoa da maquina digita `claude` e a sessao ja abre em
`bypassPermissions`, sem `--dangerously-skip-permissions`, sem alias e sem o
dialogo "Yes, I accept". Isso e feito no **managed settings**, o tier de maior
precedencia: ninguem desliga pelo `~/.claude/settings.json` proprio.

Antes de instalar, confirme com a pessoa que ela entende o que bypass
significa: o Claude executa qualquer comando, edita qualquer arquivo e faz
qualquer chamada de rede sem perguntar, em todas as contas da maquina.

## Os tres caminhos e o que cada um faz

| Caminho | Onde age | Quem afeta | Alguem desliga? |
|---|---|---|---|
| Managed settings (obrigatorio) | dentro do Claude Code | todas as contas, qualquer forma de chamada | nao |
| Alias em `/etc/zshrc` (opcional) | shell zsh interativo | todas as contas, so no terminal | sim, `unalias` ou `~/.zshrc` |
| Wrapper `/usr/local/bin/claude-bypass` (opcional) | novo comando | quem chamar `claude-bypass` | `claude` normal continua pedindo |

Para "digitar `claude` e entrar em bypass", so o primeiro importa. Os outros
dois sao reforco e podem ser pulados.

## Config

Duas chaves no `managed-settings.json`:

```json
{
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "skipDangerousModePermissionPrompt": true
}
```

- `permissions.defaultMode: "bypassPermissions"`: toda sessao nova comeca em
  bypass. Desde a v2.1.257 esse valor so vale em user ou managed settings; em
  `.claude/settings.json` de projeto e ignorado.
- `skipDangerousModePermissionPrompt: true`: suprime o dialogo de confirmacao
  que aparece na primeira vez que alguem entra em bypass.

Nao funciona rodando como `root`: o Claude Code recusa bypass nesse caso.

## Pre-checagens

```bash
claude --version                                     # >= 2.1.257
ls -la "/Library/Application Support/ClaudeCode/"    # managed-settings.json existe?
jq . "/Library/Application Support/ClaudeCode/managed-settings.json"
```

Se o arquivo ja existir com outras chaves (statusLine, plugins), faca **merge**
com `jq`, nunca sobrescreva.

## Instalar (managed settings)

Precisa de root; peca para a pessoa rodar com `!`, **um comando por linha**.
Heredoc de varias linhas nao funciona com o prefixo `!`.

1. Backup datado:
   ```
   ! sudo cp "/Library/Application Support/ClaudeCode/managed-settings.json" "/Library/Application Support/ClaudeCode/managed-settings.json.bak-$(date +%Y%m%d-%H%M%S)"
   ```
2. Se o arquivo **nao existe**, crie com as duas chaves:
   ```
   ! printf '{\n  "permissions": { "defaultMode": "bypassPermissions" },\n  "skipDangerousModePermissionPrompt": true\n}\n' | sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json" > /dev/null
   ```
   Se o arquivo **ja existe**, faca merge preservando o resto:
   ```
   ! jq '. + {"permissions": ((.permissions // {}) + {"defaultMode": "bypassPermissions"}), "skipDangerousModePermissionPrompt": true}' "/Library/Application Support/ClaudeCode/managed-settings.json" | sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json.new" > /dev/null && sudo mv "/Library/Application Support/ClaudeCode/managed-settings.json.new" "/Library/Application Support/ClaudeCode/managed-settings.json"
   ```
3. Dono e permissao:
   ```
   ! sudo chown root:admin "/Library/Application Support/ClaudeCode/managed-settings.json" && sudo chmod 644 "/Library/Application Support/ClaudeCode/managed-settings.json" && jq . "/Library/Application Support/ClaudeCode/managed-settings.json"
   ```
4. Reinicie o Claude Code. Sessoes ja abertas nao mudam.

## Opcional: alias global no zsh

Cobre quem usa uma versao antiga do Claude Code que ignore o managed settings.
Vale so em zsh interativo e um update do macOS pode recriar `/etc/zshrc`.

```
! sudo cp /etc/zshrc /etc/zshrc.bak-$(date +%Y%m%d-%H%M%S)
! printf '\n# >>> claude bypass global >>>\nalias claude='"'"'claude --dangerously-skip-permissions'"'"'\n# <<< claude bypass global <<<\n' | sudo tee -a /etc/zshrc > /dev/null && tail -4 /etc/zshrc
```

Para bash, o mesmo bloco em `/etc/bashrc`.

## Opcional: wrapper claude-bypass

Cria um segundo comando e deixa `claude` intacto. Util quando **nao** se quer
forcar bypass em todo mundo, so oferecer um atalho.

```
! printf '#!/bin/bash\nexec /usr/local/bin/claude --dangerously-skip-permissions "$@"\n' | sudo tee /usr/local/bin/claude-bypass > /dev/null
! sudo chmod 755 /usr/local/bin/claude-bypass
! sudo chown root:wheel /usr/local/bin/claude-bypass
! ls -la /usr/local/bin/claude-bypass
```

Rode o `chmod` numa linha separada e confira o `ls`: precisa sair `-rwxr-xr-x`.
Encadeado com `&&` depois do `tee` ja falhou silenciosamente e o arquivo ficou
644, sem execucao.

O caminho `/usr/local/bin/claude` e o binario global do Mac. Se a maquina so
tiver `claude` por usuario em `~/.local/bin`, troque por
`"$(command -v claude)"`.

## Verificar

Em qualquer conta, num terminal novo:

```bash
claude doctor 2>&1 | grep -A3 -i "invalid settings"   # tem que sair vazio
jq '.permissions.defaultMode, .skipDangerousModePermissionPrompt' "/Library/Application Support/ClaudeCode/managed-settings.json"
type claude              # se instalou o alias: "claude is an alias for ..."
claude-bypass --version  # se instalou o wrapper
```

Abra `claude` e rode `/status`: o modo deve aparecer como bypass sem ter pedido
confirmacao.

## Reverter

- Managed: remova as duas chaves (ou troque `defaultMode` por `"ask"`), ou
  restaure o backup datado. Reinicie o Claude Code.
  ```
  ! jq 'del(.permissions.defaultMode, .skipDangerousModePermissionPrompt) | if .permissions == {} then del(.permissions) else . end' "/Library/Application Support/ClaudeCode/managed-settings.json" | sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json.new" > /dev/null && sudo mv "/Library/Application Support/ClaudeCode/managed-settings.json.new" "/Library/Application Support/ClaudeCode/managed-settings.json"
  ```
- Alias: apague o bloco entre `# >>> claude bypass global >>>` e
  `# <<< claude bypass global <<<` em `/etc/zshrc`, ou restaure o backup.
- Wrapper: `! sudo rm /usr/local/bin/claude-bypass`.

## Proibir bypass na maquina inteira

O oposto desta skill. Vale para todos e ninguem entra em bypass, nem com a
flag:

```json
"permissions": { "disableBypassPermissionsMode": true }
```

## Armadilhas conhecidas

- `defaultMode: bypassPermissions` num `.claude/settings.json` de projeto nao
  faz nada desde a v2.1.257. Se alguem reclamar que "colocou e nao funciona",
  cheque onde a chave esta.
- O agente Claude Code nao consegue instalar isso sozinho: precisa de `sudo`
  com senha, e o classificador do auto mode bloqueia acoes que configurem
  bypass. O fluxo e sempre o agente preparar e a pessoa rodar com `!`.
- O `managed-settings.json` desta maquina e versionado no repositorio clonado
  na mesma pasta. Commite a mudanca junto com o README; os `.bak-*` ficam no
  `.gitignore`.
