# SECURITY — agentic_job_radar
Decisões de segurança, riscos conhecidos e mitigações.

---

## Modelo de ameaça

O risco principal é o OpenClaw ser comprometido via **prompt injection** — instruções maliciosas embutidas em dados externos (JDs de vagas, respostas de APIs) que façam o agente executar ações não intencionais com acesso às credenciais do usuário.

---

## Decisões de segurança

### Dados do pipeline no filesystem Linux (não no DrvFs)

**Problema:** pastas Windows montadas em `/mnt/c/` (DrvFs) herdam semântica de permissões diferente da do Linux; compartilhar `C:\SharedData\` entre `sandbox`/`guilh` e o WSL gerava atrito e risco de exposição indevida.

**Decisão:** o pipeline grava em `/home/openclaw/agentic_job_radar/` no Ubuntu WSL2. O `guilh` revisa por leitura via `\\wsl$\Ubuntu\home\openclaw\agentic_job_radar\` (ex.: `output/index.html`). **Tradeoff aceito:** o `openclaw` não depende de dados em `C:\` via `/mnt/c/` para o pipeline.

---

### WSL2: `umask=027` e isolamento do home do `guilh`

**Configuração:** `umask=027` em `/etc/wsl.conf` (efeito típico via opções de automount do DrvFs — ver `INFRASTRUCTURE.md`).

**Efeito para o `openclaw`:** bloqueia acesso do usuário `openclaw` ao filesystem Windows sob `/mnt/c/Users/guilh/`, incluindo a chave SSH em `.ssh/id_ed25519` e demais arquivos do perfil do `guilh` montados com permissões restritivas.

**Limitação residual:** o `openclaw` ainda pode ter acesso de **leitura** a outras áreas de `/mnt/c/` em geral, conforme permissões herdadas do Windows — **não** há symlinks explícitos no `openclaw` para `.aws`, `.azure` ou outras credenciais do `guilh` (contraste com o `gmrv`).

---

### WSL2: usuário `openclaw` separado do `gmrv`

**Problema identificado:** o usuário Linux padrão `gmrv` tem symlinks diretos para credenciais do `guilh`:
```
~/.aws   → /mnt/c/Users/guilh/.aws
~/.azure → /mnt/c/Users/guilh/.azure
```
Além disso, o `gmrv` tem acesso amplo a `/mnt/c/Users/guilh/` via DrvFs.

**Decisão:** OpenClaw roda exclusivamente como o usuário Linux `openclaw`, sem esses symlinks e com o reforço do `umask=027` sobre o acesso ao home do `guilh` em `/mnt/c/`.

**O `gmrv` não é removido** — é o usuário padrão do WSL2 e pode ser necessário para outras tarefas. Simplesmente não é usado para o OpenClaw.

---

### Feedback: Telegram em vez de escrita direta nos artefatos

**Decisão:** o feedback ativo do `guilh` entra pelo **Telegram** (OpenClaw), não por edição direta dos JSON/HTML do pipeline.

**Benefício:** elimina a necessidade de conceder ao `guilh` escrita nos dados do pipeline no WSL2; a revisão permanece por leitura (HTML estático e Explorer em `\\wsl$\...`).

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
| `openclaw` com leitura possível em partes de `/mnt/c/` (DrvFs) | Sem symlinks para credenciais do `guilh`; `umask=027` bloqueia `/mnt/c/Users/guilh/` para o `openclaw`; sem ClawHub | Aceito com mitigação |
| Bot Telegram sem whitelist | Apenas durante desenvolvimento; configurar no Bloco 4 | Aceito temporariamente |
| Prompt injection via JDs de vagas | Sem skills externas; agente com escopo restrito ao pipeline | Mitigado parcialmente |
| `gmrv` com acesso às credenciais e ao home do `guilh` | `gmrv` não é usado para o OpenClaw | Aceito |

---

*Última atualização: Mar 2026 — Bloco 1 em andamento (Ollama validado)*
