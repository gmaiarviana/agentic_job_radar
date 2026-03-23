# INFRASTRUCTURE — agentic_job_radar
Referência de usuários, portas, acessos e configurações do ambiente.
Atualizar sempre que houver mudança de infraestrutura.

---

## Usuários Windows

| Usuário | Tipo | Propósito |
|---|---|---|
| `guilh` | Admin | Trabalho normal; revisão de vagas; acesso ao Telegram |
| `sandbox` | Padrão | Roda Ollama (Windows); sem credenciais pessoais — OpenClaw roda no WSL2 como `openclaw` |
| `Paola` | — | Outro usuário da máquina; sem relação com o projeto |

---

## Usuários Linux (WSL2)

| Usuário | Propósito | Observações |
|---|---|---|
| `gmrv` | Usuário padrão existente | **NÃO usar para OpenClaw** — tem symlinks para credenciais do `guilh` (`.aws`, `.azure`) e acesso amplo ao home do `guilh` em `/mnt/c/Users/guilh/` |
| `openclaw` | Roda o OpenClaw | Sem symlinks para credenciais; isolamento reforçado com `options="metadata,umask=027,fmask=137"` no automount (ver WSL2 abaixo) |

### Symlinks perigosos no `gmrv` (não remover — só ignorar)
```
~/.aws   → /mnt/c/Users/guilh/.aws
~/.azure → /mnt/c/Users/guilh/.azure
```

---

## Dados do pipeline (WSL2 — filesystem Linux)

**Pastas Windows legadas:** `C:\SharedData\` foi criada no início do Bloco 1 e está **abandonada** no plano atual (não usar para o pipeline). `C:\SharedModels\` deixou de ser o caminho planejado para partilha com o WSL, mas o Ollama está a apontar para ela como *model location* atual — ver secção Ollama abaixo.

**Estado do diretório de trabalho:** criado em 22/03/2026; acessível pelo `guilh` via `\\wsl$\Ubuntu\home\openclaw\agentic_job_radar\`.

**Permissões (acesso pelo Explorer):** `/home/openclaw` e `/home/openclaw/agentic_job_radar` com modo `755` — necessário para o `guilh` enxergar o caminho via Windows Explorer.

| Item | Valor |
|---|---|
| Caminho no Ubuntu (WSL2) | `/home/openclaw/agentic_job_radar/` |
| Acesso pelo `guilh` (Windows Explorer) | `\\wsl$\Ubuntu\home\openclaw\agentic_job_radar\` |
| Quem grava | usuário Linux `openclaw` (workers / Lobster / OpenClaw no WSL2) |
| Revisão pelo `guilh` | leitura do HTML estático em `output/index.html`; feedback ativo via Telegram |

### Estrutura de `/home/openclaw/agentic_job_radar/`
```
raw/        → vagas brutas (um arquivo por vaga: {id_hash}.json)
filtered/   → após filtro básico de título e localização
enriched/   → com HTML completo da página da vaga
scored/     → após scoring + análise
logs/       → raciocínio passo a passo por vaga
output/     → HTML estático para revisão pelo guilh
seen_jobs.json → dedup persistente
```

---

## Ollama

| Item | Valor |
|---|---|
| Instalado em | instalação de sistema via PowerShell admin (não por utilizador) |
| Model location | `C:\SharedModels` (valor atual no ambiente; modelo `qwen3:8b` presente; Ollama apontado para este diretório) |
| Tipo de processo | processo Windows (não serviço systemd nem serviço Windows dedicado documentado aqui) |
| Bind address | `127.0.0.1` (não exposto na rede) |
| Porta | `11434` |
| Endpoint | `http://127.0.0.1:11434` (responde HTTP 200) |
| Modelo | `qwen3:8b` |
| Validação | `GET /api/tags` OK; `qwen3:8b` presente |
| "Expose to network" | desligado |

**Acesso pelo `guilh`:** via HTTP em `http://127.0.0.1:11434` — sem instância própria do Ollama no perfil do `guilh`.

**Acesso pelo WSL2 (`openclaw`):** `http://127.0.0.1:11434` com sucesso via *mirrored networking* — confirmado com `sudo -u openclaw curl` para o endpoint local.

---

## WSL2

| Item | Valor |
|---|---|
| Versão | WSL2 |
| Distro | Ubuntu 24.04.1 LTS |
| Networking | `mirrored` (configurado em `%USERPROFILE%\.wslconfig`) |
| DrvFs (`[automount]` em `/etc/wsl.conf`) | `options="metadata,umask=027,fmask=137"` — `metadata` habilita permissões POSIX no mount do Windows; `fmask=137` resulta em modo **640** para arquivos no DrvFs (dono lê/escreve, grupo só lê, outros sem acesso); em conjunto com `umask=027`, o usuário `openclaw` não acessa o home do `guilh` em `/mnt/c/Users/guilh/` (incl. `.ssh`) — ver `SECURITY.md` |
| Usuário padrão | `gmrv` (não usar para OpenClaw) |
| Usuário OpenClaw | `openclaw` |

### `.wslconfig` (em `C:\Users\guilh\.wslconfig`)
```ini
[wsl2]
networkingMode=mirrored
```

O mirrored networking garante que `127.0.0.1` dentro do WSL2 aponte para o mesmo `127.0.0.1` do Windows — necessário para o OpenClaw acessar o Ollama.

### `/etc/wsl.conf` (dentro do Ubuntu — valor real do ambiente)
```ini
[automount]
options = "metadata,umask=027,fmask=137"
```

**Notas:** `metadata` permite que o WSL aplique semântica de permissões POSIX ao DrvFs. `fmask=137` faz com que arquivos montados fiquem em modo **640** (mais restritivo para “outros” do que só `umask=027` sem `fmask` explícito). O `umask=027` continua a atuar nos bits de diretórios e no conjunto das máscaras do automount.

---

## Node.js

| Item | Valor |
|---|---|
| Instalado via | `nvm` (não via apt/sistema) — sem necessidade de `sudo` |
| nvm path | `/home/openclaw/.nvm/` |
| Para carregar em qualquer comando | `source ~/.nvm/nvm.sh` |
| Motivo | `openclaw` fora do sudoers; `nvm` instala no home do usuário |
| Node | `v22.22.1` |
| npm | `10.9.4` |
| Pacotes globais npm | ficam dentro do `nvm` (não usar `npm config set prefix` — conflita com `nvm`) |
| Nota de debug | comandos `wsl -u openclaw` executados do PowerShell geram `chdir(/mnt/c/Windows/system32) failed 13` — é inofensivo (`fmask` bloqueia acesso do `openclaw` ao `system32`) |

---

## OpenClaw

| Item | Valor |
|---|---|
| Roda como | usuário Linux `openclaw` no WSL2 |
| Status | instalado e rodando (Mar 2026) |
| Versão | `2026.3.22 (4dcc39c)` |
| Instalado via | `npm install -g openclaw@latest` (nvm, sem `sudo`) |
| Config | `/home/openclaw/.openclaw/openclaw.json` |
| Config backup | `/home/openclaw/.openclaw/openclaw.json.bak` |
| Workspace | `/home/openclaw/.openclaw/workspace/` |
| Gateway service | `systemd user service` em `/home/openclaw/.config/systemd/user/openclaw-gateway.service` |
| Lingering | habilitado (sobrevive a logout do `openclaw`) |
| Runtime | Node (obrigatório para Telegram) |
| Log file | `/tmp/openclaw/openclaw-2026-03-23.log` (padrão: `/tmp/openclaw/openclaw-{data}.log`) |
| Bind address | `127.0.0.1` |
| Porta Gateway | `18789` |
| Dashboard | `http://127.0.0.1:18789` |
| Modelo | Ollama → `qwen3:8b` via `http://127.0.0.1:11434` |
| Control UI | `assets` ausentes (cosmético — `pnpm ui:build` para gerar, não é necessário para o pipeline) |
| Web search | `DuckDuckGo` selecionado no onboard, mas pede API key (inesperado — documentado como “key-free”). Web search irrelevante para o pipeline; resolver se necessário no futuro. |
| Skills externas | nenhuma instalada (todas skipped no onboard) |
| Hooks | nenhum habilitado |

### Útil para debug (guilh -> openclaw via PowerShell)

Comandos do guilh para o `openclaw` seguem o padrão:
```powershell
wsl -u openclaw -- bash -c "source ~/.nvm/nvm.sh && <comando>"
```

Ver logs do gateway:
```powershell
wsl -u openclaw -- bash -c "journalctl --user --no-pager -n 50 -u openclaw-gateway"
```

Status do serviço:
```powershell
wsl -u openclaw -- bash -c "systemctl --user status openclaw-gateway"
```

Reiniciar gateway:
```powershell
wsl -u openclaw -- bash -c "systemctl --user restart openclaw-gateway"
```

Ver config:
```powershell
wsl -u openclaw -- bash -c "source ~/.nvm/nvm.sh && openclaw config get gateway.auth.token"
```

---

## Lobster

| Item | Valor |
|---|---|
| Instalado via | incluído com OpenClaw (npm — não é instalação separada) |
| Roda como | usuário Linux `openclaw` no WSL2 |
| Workflows | `/home/openclaw/agentic_job_radar/` (filesystem Linux; não usar `C:\SharedData\`) |
| Status | incluído com OpenClaw (instalado junto, Mar 2026) |

---

## Telegram Bot

| Item | Valor |
|---|---|
| Status | criado e conectado (Mar 2026) |
| Username | `@agentic_job_radar_bot` |
| Criado via | @BotFather |
| Conectado ao | OpenClaw (usuário `openclaw` no WSL2) |
| User ID autorizado (pairing) | `942075913 (guilh)` |
| Pairing mode | padrão (primeiro usuário pareado) |
| DM access policy | pairing (padrão) |
| Token do bot | armazenado na config do OpenClaw (`openclaw.json`) |

---

## Portas em uso

| Porta | Serviço | Bind |
|---|---|---|
| `11434` | Ollama | `127.0.0.1` |
| `18789` | OpenClaw Gateway | `127.0.0.1` |
| `18791` | Browser control (OpenClaw) | `127.0.0.1` |

---

## Senhas e credenciais

Não armazenar senhas aqui. Referência de onde cada credencial vive:

| Credencial | Onde fica |
|---|---|
| Senha do usuário `sandbox` | Windows — definida pelo `guilh` |
| Token do bot Telegram | OpenClaw config (`openclaw.json`) |
| Token do OpenClaw Gateway | gerado no onboarding (a documentar após instalação) |

---

*Última atualização: Mar 2026 — Bloco 1 concluído*
