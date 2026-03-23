# INFRASTRUCTURE — agentic_job_radar
Referência de usuários, portas, acessos e configurações do ambiente.
Atualizar sempre que houver mudança de infraestrutura.

---

## Usuários Windows

| Usuário | Tipo | Propósito |
|---|---|---|
| `guilh` | Admin | Trabalho normal; revisão de vagas; acesso ao Telegram |
| `sandbox` | Padrão | Roda Ollama e OpenClaw; sem credenciais pessoais |
| `Paola` | — | Outro usuário da máquina; sem relação com o projeto |

---

## Usuários Linux (WSL2)

| Usuário | Propósito | Observações |
|---|---|---|
| `gmrv` | Usuário padrão existente | **NÃO usar para OpenClaw** — tem symlinks para credenciais do `guilh` (`.aws`, `.azure`) |
| `openclaw` | Roda o OpenClaw | Criado sem symlinks; sem acesso às credenciais do `guilh` |

### Symlinks perigosos no `gmrv` (não remover — só ignorar)
```
~/.aws   → /mnt/c/Users/guilh/.aws
~/.azure → /mnt/c/Users/guilh/.azure
```

---

## Pastas Compartilhadas

| Pasta | Propósito | Permissões |
|---|---|---|
| `C:\SharedModels\` | Modelos Ollama | `guilh` + `sandbox`: leitura e escrita |
| `C:\SharedData\agentic_job_radar\` | Dados do pipeline | `guilh` + `sandbox`: leitura e escrita |

### Estrutura de `C:\SharedData\agentic_job_radar\`
```
raw\        → vagas brutas (um arquivo por vaga: {id_hash}.json)
filtered\   → após filtro básico de título e localização
enriched\   → com HTML completo da página da vaga
scored\     → após scoring + análise
logs\       → raciocínio passo a passo por vaga
output\     → HTML estático para revisão pelo guilh
seen_jobs.json → dedup persistente
```

---

## Ollama

| Item | Valor |
|---|---|
| Instalado em | usuário Windows `sandbox` |
| Model location | `C:\SharedModels\` |
| Bind address | `127.0.0.1` (não exposto na rede) |
| Porta | `11434` |
| Endpoint | `http://127.0.0.1:11434` |
| Modelo | `qwen3:8b` |
| "Expose to network" | desligado |

**Acesso pelo `guilh`:** via HTTP em `http://127.0.0.1:11434` — funciona graças ao mirrored networking do WSL2. Não requer instância própria do Ollama.

---

## WSL2

| Item | Valor |
|---|---|
| Versão | WSL2 |
| Distro | Ubuntu 24.04.1 LTS |
| Networking | `mirrored` (configurado em `%USERPROFILE%\.wslconfig`) |
| Usuário padrão | `gmrv` (não usar para OpenClaw) |
| Usuário OpenClaw | `openclaw` (a criar) |

### `.wslconfig` (em `C:\Users\guilh\.wslconfig`)
```ini
[wsl2]
networkingMode=mirrored
```

O mirrored networking garante que `127.0.0.1` dentro do WSL2 aponte para o mesmo `127.0.0.1` do Windows — necessário para o OpenClaw acessar o Ollama.

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
| Workflows | `C:\SharedData\agentic_job_radar\` (montado via `/mnt/c/`) |
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
