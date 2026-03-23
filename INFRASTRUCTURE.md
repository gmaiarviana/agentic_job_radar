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
| `openclaw` | Roda o OpenClaw | Sem symlinks para credenciais; isolamento reforçado com `umask=027` (ver WSL2 abaixo) |

### Symlinks perigosos no `gmrv` (não remover — só ignorar)
```
~/.aws   → /mnt/c/Users/guilh/.aws
~/.azure → /mnt/c/Users/guilh/.azure
```

---

## Dados do pipeline (WSL2 — filesystem Linux)

**Pastas Windows legadas:** `C:\SharedData\` e `C:\SharedModels\` foram criadas no início do Bloco 1 mas estão **abandonadas** no plano atual (não usar). Ainda existem no disco — o `guilh` pode apagá-las manualmente se quiser recuperar espaço.

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
| Instalado em | usuário Windows `sandbox` — raiz da instalação `C:\Users\sandbox\.ollama\` |
| Model location | diretório padrão do Ollama: `C:\Users\sandbox\.ollama\models\` |
| Bind address | `127.0.0.1` (não exposto na rede) |
| Porta | `11434` |
| Endpoint | `http://127.0.0.1:11434` |
| Modelo | `qwen3:8b` |
| Pull / cache | `qwen3:8b` instalado e validado |
| "Expose to network" | desligado |

**Acesso pelo `guilh`:** via HTTP em `http://127.0.0.1:11434` — funciona graças ao mirrored networking do WSL2. Não requer instância própria do Ollama.

---

## WSL2

| Item | Valor |
|---|---|
| Versão | WSL2 |
| Distro | Ubuntu 24.04.1 LTS |
| Networking | `mirrored` (configurado em `%USERPROFILE%\.wslconfig`) |
| `umask` | `027` em `/etc/wsl.conf` — grupos/outros sem leitura em arquivos novos do `guilh` no DrvFs; o usuário `openclaw` não acessa o home do `guilh` em `/mnt/c/Users/guilh/` (incl. `.ssh`) — ver `SECURITY.md` |
| Usuário padrão | `gmrv` (não usar para OpenClaw) |
| Usuário OpenClaw | `openclaw` |

### `.wslconfig` (em `C:\Users\guilh\.wslconfig`)
```ini
[wsl2]
networkingMode=mirrored
```

O mirrored networking garante que `127.0.0.1` dentro do WSL2 aponte para o mesmo `127.0.0.1` do Windows — necessário para o OpenClaw acessar o Ollama.

### `/etc/wsl.conf` (dentro do Ubuntu — exemplo)
```ini
[automount]
options = "umask=027"
```

Ajuste conforme a linha real do ambiente; o efeito desejado é `umask=027` aplicado ao mount do DrvFs.

---

## OpenClaw

| Item | Valor |
|---|---|
| Roda como | usuário Linux `openclaw` no WSL2 |
| Bind address | `127.0.0.1` |
| Porta Gateway | `18789` |
| Dashboard | `http://127.0.0.1:18789` |
| Modelo | Ollama → `qwen3:8b` via `http://127.0.0.1:11434` |
| Skills externas | nenhuma (ClawHub desabilitado) |
| Status | a instalar |

---

## Lobster

| Item | Valor |
|---|---|
| Instalado via | npm (junto com OpenClaw) |
| Roda como | usuário Linux `openclaw` no WSL2 |
| Workflows | `/home/openclaw/agentic_job_radar/` (filesystem Linux; não usar `C:\SharedData\`) |
| Status | a instalar |

---

## Telegram Bot

| Item | Valor |
|---|---|
| Criado via | @BotFather |
| Conectado ao | OpenClaw (usuário `openclaw` no WSL2) |
| Usuário autorizado | `guilh` (whitelist por user ID — a configurar no Bloco 4) |
| Status | a criar |

---

## Portas em uso

| Porta | Serviço | Bind |
|---|---|---|
| `11434` | Ollama | `127.0.0.1` |
| `18789` | OpenClaw Gateway | `127.0.0.1` |

---

## Senhas e credenciais

Não armazenar senhas aqui. Referência de onde cada credencial vive:

| Credencial | Onde fica |
|---|---|
| Senha do usuário `sandbox` | Windows — definida pelo `guilh` |
| Token do bot Telegram | OpenClaw config (a documentar após criação) |
| Token do OpenClaw Gateway | gerado no onboarding (a documentar após instalação) |

---

*Última atualização: Mar 2026 — Bloco 1 em andamento*
