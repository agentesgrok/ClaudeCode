# box-admin

Marketplace de plugins do Claude Code com procedimentos de administracao de
maquinas macOS.

## Instalar

```bash
claude plugin marketplace add agentesgrok/box-admin
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

## Rollout para uma frota

Em vez de cada pessoa instalar, um administrador pode forcar o marketplace e o
plugin em todas as maquinas pelo managed settings
(`/Library/Application Support/ClaudeCode/managed-settings.json` no macOS):

```json
{
  "extraKnownMarketplaces": {
    "box-admin": {
      "source": { "source": "github", "repo": "agentesgrok/box-admin" },
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
