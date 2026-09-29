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

Duas skills, ambas escritas para o fluxo em que o agente prepara os comandos e a
pessoa roda os que precisam de `sudo` com o prefixo `!`:

- **`/macos-admin:global-statusline`** — instala, troca ou remove a status line
  global via `managed-settings.json`, o tier de maior precedencia (nenhum usuario
  sobrescreve pelo settings dele). Traz um `scripts/statusline.sh` de referencia
  que mostra modelo, uso de contexto, email/assento e limite da sessao.
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
├── managed-settings.json     root:admin 644  chave "statusLine"
├── statusline.sh             root:wheel 755  copia de macos-admin/scripts/statusline.sh
├── README.md                                 este arquivo (clone do repositorio)
├── .claude-plugin/marketplace.json
└── macos-admin/
    ├── .claude-plugin/plugin.json
    ├── scripts/statusline.sh
    └── skills/{global-statusline,new-macos-user}/SKILL.md
```

O clone e os arquivos do repositorio pertencem ao usuario que clonou; apenas
`managed-settings.json` e `statusline.sh` sao root-owned. Editar esses dois
exige `sudo`.

Por estar no managed settings, a status line vale para **todos os usuarios** da
maquina e tem precedencia sobre `~/.claude/settings.json` e `.claude/settings.json`
de projeto: ninguem consegue sobrescrever ou desligar pelo settings proprio.

Hoje o `managed-settings.json` contem so a status line:

```json
{
  "statusLine": {
    "type": "command",
    "command": "bash '/Library/Application Support/ClaudeCode/statusline.sh'"
  }
}
```

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
