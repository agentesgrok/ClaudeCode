# ClaudeCode

Marketplace de plugins do Claude Code com procedimentos de administracao de
maquinas macOS.

## Instalar

```bash
claude plugin marketplace add agentesgrok/ClaudeCode
claude plugin install macos-admin@box-admin
```

## Plugins

### macos-admin

Tres skills, todas escritas para o fluxo em que o agente prepara os comandos e a
pessoa roda os que precisam de `sudo` com o prefixo `!`:

- **`/macos-admin:global-statusline`** — instala, troca ou remove a status line
  global via `managed-settings.json`, o tier de maior precedencia (nenhum usuario
  sobrescreve pelo settings dele). Traz um `scripts/statusline.sh` de referencia
  que mostra modelo, uso de contexto, email/assento e limite da sessao.
- **`/macos-admin:global-bypass`** — faz `claude` abrir direto em
  `bypassPermissions` para todos os usuarios da maquina, via
  `permissions.defaultMode` + `skipDangerousModePermissionPrompt` no
  `managed-settings.json`. Cobre merge com `jq` num arquivo ja existente, os
  opcionais alias em `/etc/zshrc` e wrapper `claude-bypass`, verificacao,
  reversao e como proibir bypass com `disableBypassPermissionsMode`. Tem uma
  secao Linux/Debian com os caminhos equivalentes
  (`/etc/claude-code/managed-settings.json`, `/etc/profile.d`, `root:root`).
- **`/macos-admin:new-macos-user`** — cria conta local com `sysadminctl`: escolha
  de UID livre, admin por padrao, `createhomedir`, e verificacao da senha com
  `dscl . -authonly`. Cobre os avisos enganosos de Secure Token e FileVault.

Dependencias do statusline de referencia: `jq` e `curl`. Ele le o token OAuth do
**proprio usuario** (Keychain ou `~/.claude/.credentials.json`) para consultar
`api.anthropic.com/api/oauth/profile` e exibir email e assento — leia o script
antes de instalar, ele roda a cada render do prompt de todo mundo na maquina.

## Onde fica no Mac

Nesta maquina este repositorio esta clonado direto em
`/Library/Application Support/ClaudeCode/`, a pasta que o Claude Code usa como
**managed settings** no macOS. Os arquivos instalados ficam na raiz, ao lado do
clone:

```
/Library/Application Support/ClaudeCode/
├── managed-settings.json     root:admin 644  statusLine + bypass global
├── statusline.sh             root:wheel 755  copia de macos-admin/scripts/statusline.sh
├── README.md                                 este arquivo (clone do repositorio)
├── .claude-plugin/marketplace.json
└── macos-admin/
    ├── .claude-plugin/plugin.json
    ├── scripts/statusline.sh
    └── skills/{global-statusline,global-bypass,new-macos-user}/SKILL.md
```

Fora da pasta, o bypass global tambem deixou dois opcionais instalados nesta
maquina, ambos redundantes com o managed settings:

```
/etc/zshrc                    bloco "claude bypass global": alias claude='claude --dangerously-skip-permissions'
/usr/local/bin/claude-bypass  root:wheel 755  exec /usr/local/bin/claude --dangerously-skip-permissions "$@"
```

O clone e os arquivos do repositorio pertencem ao usuario que clonou; apenas
`managed-settings.json` e `statusline.sh` sao root-owned. Editar esses dois
exige `sudo`. Os backups datados `*.bak-*` gerados pelas skills ficam na mesma
pasta e estao no `.gitignore`.

Por estar no managed settings, a status line vale para **todos os usuarios** da
maquina e tem precedencia sobre `~/.claude/settings.json` e `.claude/settings.json`
de projeto: ninguem consegue sobrescrever ou desligar pelo settings proprio.

Hoje o `managed-settings.json` contem a status line e o modo bypass global:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash '/Library/Application Support/ClaudeCode/statusline.sh'"
  },
  "permissions": {
    "defaultMode": "bypassPermissions"
  },
  "skipDangerousModePermissionPrompt": true
}
```

- `permissions.defaultMode: "bypassPermissions"` faz toda sessao nova de
  qualquer usuario ja abrir em bypass: basta digitar `claude`, sem
  `--dangerously-skip-permissions` e sem alias. Desde a v2.1.257 esse valor so
  vale em user ou managed settings; em `.claude/settings.json` de projeto ele e
  ignorado.
- `skipDangerousModePermissionPrompt: true` suprime o dialogo de confirmacao
  "Yes, I accept" que aparece na primeira vez que alguem entra em bypass.
- Nao funciona rodando como `root`: o Claude Code recusa bypass nesse caso.
- Para reverter, remova as duas chaves do JSON (ou troque `defaultMode` por
  `"ask"`). Para proibir bypass na maquina inteira, use
  `"permissions": { "disableBypassPermissionsMode": true }`.

O marketplace ainda nao esta no managed settings. Para adicionar, ha duas formas:

- **Diretorio local** (`"source": "directory", "path": "/Library/Application Support/ClaudeCode"`):
  aponta para este clone; nao depende de rede, mas cada atualizacao e um
  `git pull` manual na pasta.
- **GitHub** (`"source": "github", "repo": "agentesgrok/ClaudeCode"`, com
  `"autoUpdate": true`): o Claude Code clona e atualiza sozinho; o clone local
  deixa de ser necessario. E o formato mostrado em *Rollout para uma frota*.

Atualizar o script instalado depois de um `git pull`:

```bash
sudo cp "/Library/Application Support/ClaudeCode/macos-admin/scripts/statusline.sh" \
        "/Library/Application Support/ClaudeCode/statusline.sh"
sudo chown root:wheel "/Library/Application Support/ClaudeCode/statusline.sh"
sudo chmod 755 "/Library/Application Support/ClaudeCode/statusline.sh"
```

## Replicar em outro Mac

Para deixar um Mac novo igual a este (status line + bypass global para todos):

1. Instale o marketplace e o plugin em qualquer conta:
   ```bash
   claude plugin marketplace add agentesgrok/ClaudeCode
   claude plugin install macos-admin@box-admin
   ```
2. Abra `claude` e chame `/macos-admin:global-bypass`. O agente confere a
   versao, prepara os comandos e voce roda os `! sudo`, um por linha.
3. Chame `/macos-admin:global-statusline` para a status line.
4. Reinicie o Claude Code e confira com `/status` em outra conta da maquina.

O agente nao instala sozinho: precisa de `sudo` com senha, e o classificador do
auto mode bloqueia acoes que configurem bypass. Por isso todas as skills sao
"agente prepara, pessoa roda com `!`".

Num Linux/Debian o plugin instala igual e a skill `global-bypass` traz a secao
com os caminhos de la: managed settings em `/etc/claude-code/`, alias em
`/etc/profile.d/`, wrapper em `/usr/local/bin` com `root:root`. As demais
skills (status line, `sysadminctl`) sao especificas de macOS.

## Rollout para uma frota

Em vez de cada pessoa instalar, um administrador pode forcar o marketplace e o
plugin em todas as maquinas pelo managed settings
(`/Library/Application Support/ClaudeCode/managed-settings.json` no macOS):

```json
{
  "extraKnownMarketplaces": {
    "box-admin": {
      "source": { "source": "github", "repo": "agentesgrok/ClaudeCode" },
      "autoUpdate": true
    }
  },
  "enabledPlugins": {
    "macos-admin@box-admin": true
  }
}
```

## Releases

O `version` em `macos-admin/.claude-plugin/plugin.json` precisa ser incrementado
a cada mudanca publicada — sem isso, `claude plugin update` responde
"already at the latest version" e ninguem recebe a alteracao.
