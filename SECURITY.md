# SECURITY — agentic_job_radar
Decisões de segurança, riscos conhecidos e mitigações.

---

## Modelo de ameaça

O risco principal é o OpenClaw ser comprometido via **prompt injection** — instruções maliciosas embutidas em dados externos (JDs de vagas, respostas de APIs) que façam o agente executar ações não intencionais com acesso às credenciais do usuário.

---

## Decisões de segurança

### WSL2: usuário `openclaw` separado do `gmrv`

**Problema identificado:** o usuário Linux padrão `gmrv` tem symlinks diretos para credenciais do `guilh`:
```
~/.aws   → /mnt/c/Users/guilh/.aws
~/.azure → /mnt/c/Users/guilh/.azure
```
Além disso, o `gmrv` tem acesso de leitura e escrita irrestrito a `/mnt/c/Users/guilh/` via filesystem do WSL2.

**Decisão:** criar usuário Linux `openclaw` sem esses symlinks e sem acesso às pastas do `guilh`. OpenClaw roda exclusivamente como `openclaw`.

**Limitação residual:** o WSL2 monta o filesystem Windows em `/mnt/c/` para todos os usuários Linux. O `openclaw` ainda tem acesso a `/mnt/c/` em geral — mas não tem os symlinks explícitos para as credenciais do `guilh`, reduzindo a superfície de ataque acidental.

**O `gmrv` não é removido** — é o usuário padrão do WSL2 e pode ser necessário para outras tarefas. Simplesmente não é usado para o OpenClaw.

---

### Bind address: `127.0.0.1`

Tanto Ollama quanto OpenClaw Gateway ficam em `127.0.0.1` — não expostos na rede local nem na internet.

---

### Skills externas: nenhuma

ClawHub desabilitado. Análise da Cisco mostrou que ~17-26% das submissions contêm código malicioso ou prompt injection. Apenas skills próprias são permitidas.

---

### Autenticação Telegram: backlog

O bot Telegram atualmente não tem whitelist — qualquer pessoa que encontre o bot pode interagir com ele. Isso é aceitável durante o desenvolvimento (Blocos 1-3) porque o bot ainda não tem acesso a dados sensíveis.

**Antes de usar em produção (Bloco 4):** configurar `allowFrom` no OpenClaw com o user ID do Telegram do `guilh`.

---

## Riscos conhecidos e aceitos

| Risco | Mitigação | Status |
|---|---|---|
| `openclaw` acessa `/mnt/c/` em geral via WSL2 | Usuário separado sem symlinks explícitos; sem ClawHub | Aceito |
| Bot Telegram sem whitelist | Apenas durante desenvolvimento; configurar no Bloco 4 | Aceito temporariamente |
| Prompt injection via JDs de vagas | Sem skills externas; agente com escopo restrito ao pipeline | Mitigado parcialmente |
| `gmrv` com acesso às credenciais do `guilh` | `gmrv` não é usado para o OpenClaw | Aceito (não é possível remover o acesso WSL2 ao filesystem Windows sem desabilitar a integração) |

---

*Última atualização: Mar 2026 — Bloco 1 em andamento*
