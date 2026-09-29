---
name: global-ssh-mcp
description: Instala o servidor MCP ssh (diegofornalha/ssh-mcp, binario ssh-mcp-stdio em Rust) para todos os usuarios da maquina - binario root-owned em /usr/local/bin e plugin ssh-mcp forcado via enabledPlugins no managed settings. Use quando pedirem MCP ssh global, para todo mundo, em outro Mac ou Linux, ou para atualizar/remover o binario.
---

# MCP ssh global (todos os usuarios)

Objetivo: toda conta da maquina abre o `claude` com o servidor MCP `ssh`
disponivel, sem cada pessoa rodar `claude mcp add`.

## Como funciona

Dois pedacos, ambos fora do home de qualquer usuario:

1. **Binario** `ssh-mcp-stdio` em `/usr/local/bin`, root-owned. E o servidor
   em si (Rust, transporte stdio, fonte em
   https://github.com/diegofornalha/ssh-mcp).
2. **Plugin `ssh-mcp`** deste marketplace. O `.mcp.json` dele aponta para o
   binario. Quem tem o plugin habilitado ganha o servidor.

Para valer para todo mundo sem instalacao manual, o managed settings forca o
marketplace e o plugin com `extraKnownMarketplaces` + `enabledPlugins`.

## Por que nao usar managed-mcp.json ou managedMcpServers

- `managedMcpServers` (chave do managed-settings.json) so aceita servidores
  `http`/`sse` com URL `https://`. Recusa `command`, entao nao serve para um
  servidor stdio.
- `managed-mcp.json` aceita stdio, mas e **controle exclusivo**: os usuarios
  perdem todos os outros MCPs (os proprios, os de plugins e os conectores
  claude.ai, a menos que se ligue `allowAllClaudeAiMcps`), e `claude mcp add`
  passa a falhar para todo mundo. So use se a intencao for justamente travar
  a lista de MCPs da maquina. Formato, caso queira:
  ```json
  { "mcpServers": { "ssh": { "type": "stdio", "command": "/usr/local/bin/ssh-mcp-stdio" } } }
  ```
  em `/Library/Application Support/ClaudeCode/managed-mcp.json` (macOS) ou
  `/etc/claude-code/managed-mcp.json` (Linux), root-owned 644.

O plugin nao tem nenhum desses efeitos: cada usuario continua com os MCPs que
ja tinha e ganha o `ssh` por cima.

## 1. Obter o binario

Nao ha release publicada; e build a partir do fonte, ou copia de um binario ja
compilado na mesma arquitetura.

**Copiar de uma maquina/conta que ja tem** (mais rapido):

```bash
file ~/.local/bin/ssh-mcp-stdio     # confira a arquitetura: arm64 nao roda em x86_64 e vice-versa
uname -m
```

**Compilar** (precisa de Rust; `curl https://sh.rustup.rs -sSf | sh` se nao tiver):

```bash
git clone https://github.com/diegofornalha/ssh-mcp.git
cd ssh-mcp
cargo build --release
ls -la target/release/ssh-mcp-stdio
```

Leia o fonte antes de instalar para todo mundo: o servidor abre conexoes SSH
com as chaves em `~/.ssh` de quem estiver usando e executa comandos remotos.

## 2. Instalar o binario (root; a pessoa roda com `!`, um por linha)

macOS:

```
! sudo cp ~/.local/bin/ssh-mcp-stdio /usr/local/bin/ssh-mcp-stdio
! sudo chown root:wheel /usr/local/bin/ssh-mcp-stdio
! sudo chmod 755 /usr/local/bin/ssh-mcp-stdio
! sudo codesign -f -s - /usr/local/bin/ssh-mcp-stdio
! ls -la /usr/local/bin/ssh-mcp-stdio
```

Troque a origem por `target/release/ssh-mcp-stdio` se compilou. O `codesign`
ad-hoc evita o macOS matar o binario copiado. Confira `-rwxr-xr-x` no `ls`;
rode o `chmod` numa linha separada.

Linux (mesmo caminho, `root:root`, sem codesign):

```bash
sudo cp target/release/ssh-mcp-stdio /usr/local/bin/ssh-mcp-stdio
sudo chown root:root /usr/local/bin/ssh-mcp-stdio
sudo chmod 755 /usr/local/bin/ssh-mcp-stdio
```

Teste rapido (deve logar "starting ssh-mcp stdio transport" no stderr e
encerrar ao fechar o stdin):

```bash
echo | /usr/local/bin/ssh-mcp-stdio 2>&1 | head -3
```

## 3. Forcar o plugin para todos (managed settings)

Merge no `managed-settings.json` existente, preservando o resto. O repositorio
precisa estar **publicado no GitHub com o plugin `ssh-mcp`** antes, porque a
fonte e `github` com `autoUpdate`.

```
! sudo cp "/Library/Application Support/ClaudeCode/managed-settings.json" "/Library/Application Support/ClaudeCode/managed-settings.json.bak-$(date +%Y%m%d-%H%M%S)"
! jq '. + {"extraKnownMarketplaces": ((.extraKnownMarketplaces // {}) + {"box-admin": {"source": {"source": "github", "repo": "agentesgrok/ClaudeCode"}, "autoUpdate": true}}), "enabledPlugins": ((.enabledPlugins // {}) + {"ssh-mcp@box-admin": true, "macos-admin@box-admin": true})}' "/Library/Application Support/ClaudeCode/managed-settings.json" | sudo tee "/Library/Application Support/ClaudeCode/managed-settings.json.new" > /dev/null && sudo mv "/Library/Application Support/ClaudeCode/managed-settings.json.new" "/Library/Application Support/ClaudeCode/managed-settings.json"
! sudo chown root:admin "/Library/Application Support/ClaudeCode/managed-settings.json" && sudo chmod 644 "/Library/Application Support/ClaudeCode/managed-settings.json" && jq . "/Library/Application Support/ClaudeCode/managed-settings.json"
```

No Linux, o arquivo e `/etc/claude-code/managed-settings.json`, `root:root`.

Resultado esperado no JSON:

```json
"extraKnownMarketplaces": {
  "box-admin": {
    "source": { "source": "github", "repo": "agentesgrok/ClaudeCode" },
    "autoUpdate": true
  }
},
"enabledPlugins": {
  "ssh-mcp@box-admin": true,
  "macos-admin@box-admin": true
}
```

Se algum usuario ja tinha um marketplace `box-admin` apontando para uma pasta
local, o managed **nao** substitui sozinho; veja "Verificar" abaixo.

Sem managed settings, cada pessoa instala por conta propria:

```bash
claude plugin marketplace add agentesgrok/ClaudeCode
claude plugin install ssh-mcp@box-admin
```

## 4. Limpar o servidor por usuario, se existia

Quem ja tinha o `ssh` adicionado com `claude mcp add` passa a ver dois. O do
plugin aparece com prefixo do plugin; o antigo, com o nome cru. Remova o antigo
na conta em questao:

```bash
claude mcp get ssh          # Scope: User config -> e o antigo
claude mcp remove ssh -s user
```

## Verificar

O auto-install de `enabledPlugins` acontece na **primeira sessao** de cada
conta depois do managed settings, nao ao rodar `claude plugin list`. Abra um
`claude` (ou `claude -p ok`) antes de conferir.

```bash
claude plugin list --json               # os dois com "scope": "managed", "enabled": true
claude mcp list                         # plugin:ssh-mcp:ssh: /usr/local/bin/ssh-mcp-stdio - Connected
claude doctor 2>&1 | grep -i -A3 "invalid settings"   # tem que sair vazio
```

Comportamentos que confundem, mas sao normais:

- `claude plugin list` **sem** `--json` responde "No plugins installed" para
  plugins managed. Use `--json`.
- O servidor aparece como `plugin:ssh-mcp:ssh`, nao como `ssh`. As ferramentas
  ficam com prefixo `mcp__plugin_ssh-mcp_ssh__`.
- O plugin vai para `~/.claude/plugins/cache/box-admin/ssh-mcp/<versao>/` de
  cada usuario; `installed_plugins.json` continua vazio.

Se a conta ja tinha um marketplace `box-admin` apontando para outro lugar (por
exemplo uma pasta local que nao existe mais), o managed nao substitui sozinho.
Remova o antigo e abra uma sessao nova:

```bash
claude plugin marketplace remove box-admin
claude -p ok
claude plugin marketplace list          # box-admin  Source: GitHub (agentesgrok/ClaudeCode)
```

Dentro do `claude`, `/mcp` lista o servidor e `/plugin` mostra o plugin como
managed.

## Atualizar o binario

Compile ou copie a versao nova e repita o passo 2. O plugin nao muda; so o
binario. Sessoes abertas continuam com o processo antigo ate reiniciar.

## Remover

- Binario: `! sudo rm /usr/local/bin/ssh-mcp-stdio`
- Plugin forcado: tire `"ssh-mcp@box-admin"` de `enabledPlugins` no managed
  settings (ou defina `false`).

## Armadilhas conhecidas

- O `.mcp.json` do plugin usa caminho absoluto `/usr/local/bin/ssh-mcp-stdio`.
  Se o binario nao estiver la, o plugin carrega mas o servidor falha ao
  conectar; `claude mcp list` mostra "Failed".
- Binario arm64 copiado para um Mac Intel nao roda. Compile na arquitetura
  certa.
- `enabledPlugins` no managed settings com fonte `github` exige rede na
  primeira sessao de cada usuario, para clonar o marketplace.
- Nao coloque o binario dentro do plugin (11 MB, por arquitetura, e o
  diretorio do plugin muda a cada update). O lugar dele e `/usr/local/bin`.
