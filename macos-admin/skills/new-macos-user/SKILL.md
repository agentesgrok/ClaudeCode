---
name: new-macos-user
description: Cria uma conta de usuario local no macOS com o procedimento padrao desta maquina - escolha de UID livre, sysadminctl com admin por padrao, criacao do home, senha informada depois pela pessoa e testada pelo agente. Use sempre que pedirem para criar, adicionar ou provisionar um novo usuario/conta nesta maquina.
---

# Criar usuario local no macOS

Procedimento padrao desta maquina. Siga na ordem; nao pule a verificacao.

## 0. Coletar o que falta

Necessario apenas o **nome da conta**. Pergunte se o pedido nao trouxer.

Defaults desta maquina - NAO pergunte, apenas aplique:

- **Admin sempre.** Toda conta nova entra no grupo admin (`-admin`). So crie
  padrao se a pessoa disser explicitamente "sem admin" ou "padrao".
- **Senha digitada na hora e informada depois.** O comando de criacao usa
  `-password -` (interativo, nao fica no historico). Depois de criar, a pessoa
  **passa a senha no chat** e o agente testa com `dscl . -authonly`. Nao pergunte
  a senha antes de criar.

## 1. Levantar o estado antes de agir

```bash
dscl . -list /Users UniqueID | awk '$2 > 500 {print}'   # UIDs em uso
dscl . -read /Users/<nome> 2>&1 | head -2               # nome ja existe?
id -Gn "$(whoami)" | tr ' ' '\n' | grep -x admin        # voce e admin?
fdesetup status                                          # FileVault?
```

Escolha o **menor UID livre acima do maior em uso** (contas locais normais
comecam em 501). Se `dscl . -read /Users/<nome>` retornar algo diferente de
`eDSRecordNotFound`, o nome ja existe - pare e confirme com a pessoa.

## 2. Criar a conta (com admin)

`sudo` pede senha e o agente nao consegue digita-la. **Peca para a pessoa rodar
no proprio prompt com o prefixo `!`**, nunca tente rodar direto:

```
! sudo sysadminctl -addUser <nome> -fullName "<nome>" -UID <uid> -shell /bin/zsh -home /Users/<nome> -password - -admin
```

O `-password -` pede a senha da conta nova interativamente, logo apos a senha do
sudo. O `-admin` ja coloca no grupo admin.

Se a pessoa pedir explicitamente conta padrao, retire o `-admin`.

## 3. Criar o home (o sysadminctl NAO cria)

O `sysadminctl` apenas *atribui* o home e imprime
`Home directory is assigned (not created!)`. O login grafico criaria na hora,
mas `su -` e ssh cairiam num diretorio inexistente. Crie explicitamente:

```
! sudo createhomedir -c -u <nome>
```

Pode mandar os dois comandos (passo 2 e 3) juntos na mesma resposta, um por
bloco, para a pessoa rodar em sequencia.

## 4. Verificar (rode voce mesmo, nao precisa de sudo)

```bash
dscl . -read /Users/<nome> UniqueID PrimaryGroupID NFSHomeDirectory UserShell RealName
dsmemberutil checkmembership -U <nome> -G admin      # deve dizer "is a member"
ls -lad /Users/<nome>                                # home existe?
sysadminctl -secureTokenStatus <nome>
```

## 5. Testar a senha

Depois da verificacao, **peca a senha para a pessoa** ("me passa a senha da
conta para eu testar") e rode:

```bash
dscl . -authonly <nome> '<senha>' && echo SENHA_OK || echo SENHA_FALHOU
```

Se falhar, a senha pode ser redefinida com:

```
! sudo sysadminctl -resetPasswordFor <nome> -newPassword -
```

Reporte numa tabela: conta, UID/GID, shell, grupo admin, home, senha (OK/falhou).

## Conta ja criada sem admin? Nao recrie

Basta adicionar ao grupo:

```
! sudo dseditgroup -o edit -a <nome> -t user admin
```

Confirme com `dsmemberutil checkmembership -U <nome> -G admin`.

## Avisos conhecidos - o que e e o que nao e problema

**"No clear text password or interactive option was specified (... will not
allow user to use FDE)"** - e enganoso. A senha **e** definida normalmente;
confirme com `dscl . -authonly`. O aviso e sobre **Secure Token**, nao sobre a
senha.

**Secure Token desabilitado** - so importa com FileVault ligado. Cheque com
`fdesetup status`. Se estiver `Off`, ignore. Se estiver `On`, a conta nova nao
consegue destravar o disco no boot; e preciso conceder o token a partir de um
admin que ja seja *volume owner*:

```
! sudo sysadminctl -adminUser <admin_com_token> -adminPassword - -secureTokenOn <nome> -password -
```

Cheque quem tem token com `sysadminctl -secureTokenStatus <nome>` e quem e
volume owner com `diskutil apfs listUsers /`.

## Claude Code na conta nova

O binario e global (`/usr/local/bin/claude`, root:wheel, 755) e
`/usr/local/bin` esta no PATH padrao - a conta nova ja tem o mesmo `claude`, na
mesma versao. O managed-settings em
`/Library/Application Support/ClaudeCode/managed-settings.json` tambem vale para
ela, incluindo plugins force-enabled.

O que **nao** e herdado, por ser por usuario: login/credenciais
(`~/.claude.json`), settings, historico, projetos, sessoes e MCP servers. A
pessoa precisa rodar `/login` na conta dela.

Nunca copie `~/.claude.json` de um usuario para outro - ele carrega credenciais.
