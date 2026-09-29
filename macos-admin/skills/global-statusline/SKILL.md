---
name: global-statusline
description: Instala, atualiza ou remove a status line global do Claude Code desta maquina, via managed settings em /Library/Application Support/ClaudeCode. Use quando pedirem para colocar, trocar, atualizar ou tirar a status line para todos os usuarios.
---

# Status line global (todos os usuarios)

A status line vale para a maquina toda porque mora no **managed settings**, o
tier de maior precedencia. Isso significa que **nenhum usuario consegue
sobrescrever ou desligar** pelo `~/.claude/settings.json` dele.

## Layout instalado

```
/Library/Application Support/ClaudeCode/
├── managed-settings.json   root:admin 644  <- chave "statusLine"
└── statusline.sh           root:wheel 755  <- o script
```

O script fica root-owned de proposito: usuario sem root nao consegue adulterar
um comando que roda a cada render do prompt.

O script de referencia vem junto com o plugin, em
`${CLAUDE_PLUGIN_ROOT}/scripts/statusline.sh`. Ele mostra modelo, uso de
contexto, email/assento da conta e o limite da sessao. Depende de `jq` e `curl`,
e le o token OAuth do proprio usuario (Keychain ou `~/.claude/.credentials.json`)
para consultar `api.anthropic.com/api/oauth/profile`. Para instalar outro script,
troque o caminho de origem nos comandos abaixo.

## Config

```json
"statusLine": {
  "type": "command",
  "command": "bash '/Library/Application Support/ClaudeCode/statusline.sh'"
}
```

Aspas simples no caminho sao obrigatorias - ha espaco em "Application Support".
Campos opcionais: `padding`, `refreshInterval` (segundos), `hideVimModeIndicator`.

## Instalar ou trocar o script

1. Leia o script inteiro antes. Ele roda a cada atualizacao do prompt, no
   contexto de todo usuario da maquina. Procure por: chamadas de rede e para
   onde vao, leitura de credenciais (Keychain, `.credentials.json`), `eval`,
   base64, `curl | sh`.
2. Teste isolado antes de instalar, com um JSON de exemplo no stdin:
   ```bash
   echo '{"model":{"display_name":"Opus 5"},"context_window":{"used_percentage":23,"context_window_size":1000000,"current_usage":{"input_tokens":180000}}}' \
     | COLUMNS=120 bash "${CLAUDE_PLUGIN_ROOT}/scripts/statusline.sh"
   ```
   Cheque saida e tempo - o comando roda com frequencia, entao deve ser rapido
   (dezenas de ms). Confirme as dependencias: `jq`, `curl`, `python3`.
3. Instale (precisa de root; peca para a pessoa rodar com `!`):
   ```
   ! sudo cp "${CLAUDE_PLUGIN_ROOT}/scripts/statusline.sh" "/Library/Application Support/ClaudeCode/statusline.sh"
   ! sudo chown root:wheel "/Library/Application Support/ClaudeCode/statusline.sh"
   ! sudo chmod 755 "/Library/Application Support/ClaudeCode/statusline.sh"
   ```
4. Se a chave `statusLine` ainda nao existir no managed-settings, faca backup
   datado antes de editar:
   ```
   ! sudo cp "/Library/Application Support/ClaudeCode/managed-settings.json" \
             "/Library/Application Support/ClaudeCode/managed-settings.json.bak-$(date +%Y%m%d-%H%M%S)"
   ```
5. Reinicie o Claude Code - a status line so e lida na inicializacao da sessao.

## Verificar

```bash
claude doctor 2>&1 | grep -A3 -i "invalid settings"   # tem que sair vazio
python3 -c "import json;print(json.load(open('/Library/Application Support/ClaudeCode/managed-settings.json'))['statusLine'])"
```

## Remover

Apague a chave `statusLine` do managed-settings (ou restaure o backup datado) e
remova o `statusline.sh`. Reinicie o Claude Code.

## Armadilha conhecida

Instaladores de statusline de terceiros costumam escrever em
`~/.claude/settings.json`, que e **por usuario** e perde para o managed
settings. Se alguem rodar um desses, a config fica inerte e confunde o
diagnostico. Prefira sempre o caminho global acima, e cheque se sobrou um
`statusLine` orfao no settings do usuario.
