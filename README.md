# agentic_job_radar

Plano de Projeto (versao 2.4)

Este repositorio e a base do `agentic_job_radar`, uma evolucao do `job_radar` com foco em:
- pipeline local (custo zero de API)
- enrichimento com fetch completo da pagina antes de filtrar/pontuar
- uso de agentes para enriquecer dados e, em seguida, aplicar filtros e scoring

## Contexto e motivacao

O `job_radar` original (pipeline com scoring via API) teve limitacoes:
- ~12 vagas/semana com `location mismatch`
- ~3 vagas relevantes/semana apos filtros (volume insuficiente)
- custo de API para scoring como limitador de escala

Objetivo: pipeline local com determinismo do orquestrador e execucao em workers locais.

## Hardware (referencia do ambiente)

- Dell Inspiron 16 Plus 7640
- Intel Core Ultra 7 155H, 32GB RAM
- NVIDIA RTX 4060 8GB VRAM (modelos 7-8B inteiros na GPU)

## Arquitetura

```
OpenClaw (interface: Telegram -> guilh)
  -> Lobster (orquestrador deterministico do pipeline)
     -> step 1: coletor      -> raw/
     -> step 2: filtro       -> filtered/
     -> step 3: enriquecedor -> enriched/
     -> step 4: scorer       -> scored/

Builder (script Python) -> scored/ -> output/index.html (revisao estatico)
```

**Nota:** o enriquecedor roda após o filtro básico — fetch HTTP só para vagas que sobreviveram à triagem de título e localização. Reduz carga e falhas em vagas que seriam eliminadas de qualquer jeito.

### Componentes

1. **OpenClaw**
   - Interface e plataforma de agentes (Telegram -> usuario `guilh`)
   - Suporte a expansao futura; multi-agente de codigo fica no Bloco 4 (Symphony vs Deep Agents — ver decisao abaixo)

2. **Lobster**
   - Orquestrador deterministico do pipeline
   - YAML/flow fixo: `coletor -> filtro -> enriquecedor -> scorer -> builder`
   - Passa JSON entre etapas; LLMs executam analise, Lobster controla o fluxo

3. **Workers Python**
   - Portados/adaptados do `job_radar`
   - Contrato de entrada/saida via JSON

4. **Builder**
   - Gera `output/index.html` a partir de `scored/`
   - Sem servidor: o `guilh` abre o HTML via caminho WSL (`\\wsl$\Ubuntu\...\output\index.html`) para revisão estática; feedback ativo entra pelo Telegram

## Decisoes de arquitetura (principais)

| Decisao | Escolha | Motivo |
|---|---|---|
| Runtime de modelo | Ollama | gratuito, suporte a GPU, compativel com OpenAI API |
| Modelo inicial | `qwen3:8b` | geração mais recente que o Qwen 2.5 7B; melhor raciocínio; cabe nos 8GB de VRAM |
| Interface e plataforma de agente | OpenClaw + Lobster | OpenClaw: extensivel; Lobster: determinismo nativo |
| Canal de interface | Telegram | API oficial do Telegram; reduz risco comparado a WhatsApp (Baileys) |
| Orquestracao | Lobster (nao LangChain) | pipeline de vagas sequencial fixo; determinismo > dinamismo; Lobster permanece |
| Isolamento de seguranca | `sandbox` (Ollama no Windows) + `openclaw` (OpenClaw/Lobster/workers no WSL2) | Ollama sem credenciais pessoais; agentes e dados do pipeline no Linux com `openclaw` separado do `gmrv`; Docker removido |
| Containerização | Sem Docker | Docker foi avaliado e removido: é uma opção do OpenClaw, nao pre-requisito; `sandbox` user cumpre o papel de isolamento com menos friccao |
| Skills externas | nenhuma (ClawHub) | reduzir risco de submissions maliciosas |
| Revisao | HTML estatico + Telegram | revisão por leitura do `output/index.html`; feedback ativo pelo Telegram (sem escrita direta do `guilh` nos artefatos do pipeline) |
| Dedup | `seen_jobs.json` | mesmo schema/lógica do `job_radar` |
| Bind address OpenClaw | `127.0.0.1` | nao expor na rede (0.0.0.0) |
| Ordem do pipeline | filtro antes do enriquecedor | fetch HTTP é mais lento e fragil; rodar filtro basico primeiro elimina vagas invalidas sem custo de rede |
| Granularidade dos arquivos | Um arquivo por vaga (`{id_hash}.json`) | retomada apos falha por vaga; debug direto; rastreamento do ciclo de vida de cada vaga entre diretórios |
| JS rendering | Playwright (backlog) | maioria das fontes atuais nao requer JS; avaliar impacto real antes de implementar |
| Multi-agente de codigo (futuro, Bloco 4) | Deep Agents ou Symphony | mesmo problema (agente de codigo autonomo); Deep Agents CLI (LangChain) roda no terminal, interativo ou headless via pipe; Symphony no ecossistema OpenClaw — comparar na hora e escolher |

## Estrutura de dados (WSL2 — filesystem Linux)

Os dados do pipeline ficam no **filesystem Linux do WSL2** (Ubuntu), não em pasta Windows compartilhada — evita problemas de permissões do DrvFs e alinha com o isolamento do usuário `openclaw` (ver `SECURITY.md`).

**Caminho no Ubuntu:** `/home/openclaw/agentic_job_radar/`

**Acesso pelo `guilh` no Windows Explorer:** `\\wsl$\Ubuntu\home\openclaw\agentic_job_radar\` (leitura para revisão; HTML estático em `output\index.html`).

```
guilh (admin, Windows)
  -> revisão: leitura via \\wsl$\Ubuntu\home\openclaw\agentic_job_radar\ (ex.: output\index.html)
  -> feedback ativo para o agente: Telegram (OpenClaw) — não grava diretamente nos arquivos do pipeline

sandbox (Windows)
  -> Ollama instalado; modelos no diretório padrão (ex.: C:\Users\sandbox\.ollama\models\)
  -> guilh consome Ollama via HTTP em localhost:11434 (sem instância própria)

openclaw (Linux no WSL2)
  -> OpenClaw + Lobster + workers Python; leitura/gravação em /home/openclaw/agentic_job_radar/

/home/openclaw/agentic_job_radar/
  raw/                                    -> vagas brutas coletadas
  filtered/                               -> apos filtro basico (titulo + localizacao)
  enriched/                               -> vagas filtradas com HTML completo da pagina
  scored/                                 -> apos scoring + analise
  seen_jobs.json                          -> dedup persistente (mesmo schema do `job_radar`)
  logs/                                   -> raciocinio/explicacoes passo a passo por vaga
  output/                                 -> HTML para revisao (index.html)
```

**Nota:** `C:\SharedData\` e `C:\SharedModels\` foram abandonadas no plano atual.

## Schema de Dados (Bloco 0)

Cada arquivo segue o padrão `{id_hash}.json` — um arquivo por vaga, rastreavel entre diretorios pelo mesmo nome.

### `raw/{id_hash}.json`
Gravado pelo coletor. Campos obrigatorios:

```json
{
  "id_hash": "string",
  "source": "string",
  "title": "string",
  "company": "string",
  "location": "string",
  "salary": "string | null",
  "url": "string",
  "date": "string",
  "collected_at": "string (ISO 8601)",
  "jd_full": "string"
}
```

`jd_full` vem da API/RSS da fonte — pode estar incompleto. O enriquecedor adiciona o conteudo real da pagina sem sobrescrever este campo.

### `filtered/{id_hash}.json`
Gravado pelo filtro. Herda todos os campos do `raw/` e adiciona:

```json
{
  "filtered_at": "string (ISO 8601)",
  "filter_reason": "string | null"
}
```

Apenas vagas que passaram chegam aqui. `filter_reason` registra o motivo caso o filtro tenha tido duvida mas deixado passar (zona cinzenta).

### `enriched/{id_hash}.json`
Gravado pelo enriquecedor. Herda todos os campos do `filtered/` e adiciona:

```json
{
  "jd_fetched": "string",
  "fetch_status": "ok | failed",
  "fetched_at": "string (ISO 8601)"
}
```

`jd_full` original preservado para debug. `jd_fetched` contem o conteudo completo da pagina. O scorer usa `jd_fetched` se disponivel, senão cai para `jd_full`.

### `scored/{id_hash}.json`
Gravado pelo scorer. Herda todos os campos do `enriched/` e adiciona:

```json
{
  "score": "int",
  "score_ceiling": "int",
  "ceiling_reason": "string",
  "justification": "string",
  "main_gap": "string",
  "core_requirements": "array",
  "seniority_comparison": "object",
  "scored_at": "string (ISO 8601)"
}
```

## Contrato de I/O dos Scripts Python (pre-requisito Lobster)

Todo script worker deve respeitar o contrato do Lobster:
- Aceita o path do arquivo de entrada como argumento (ou JSON no `stdin`)
- Produz JSON no `stdout` com o resultado do step
- Retorna exit code 0 em sucesso; non-zero em falha (interrompe o pipeline)
- Grava o arquivo de saida em seu diretorio correspondente antes de responder no `stdout`

## Seguranca

- OpenClaw bind address configurado para `127.0.0.1` (nao `0.0.0.0`)
- nenhuma credencial pessoal no usuario `sandbox`
- nenhuma skill de terceiros — apenas skills proprias
- GPU acessivel via Ollama no usuario `sandbox`
- Autenticacao do bot Telegram: a definir (backlog — configurar whitelist por user ID antes de expor o bot)

## Portao go/no-go (Bloco 2)

Sobre uma amostra do mesmo dataset do `job_radar`:
- o agente de filtro deve chegar nos mesmos vereditos (passa/nao passa) em pelo menos 80% dos casos
- nenhuma vaga claramente US-only deve passar
- nenhuma vaga reconhecidamente relevante deve ser eliminada

Se falhar: iterar no prompt/criterio antes de seguir para o Bloco 3.

## Proximos Passos

### Bloco 0 — Definicao de schema e contrato de I/O *(concluido)*
- Schema de cada diretorio definido (`raw/`, `filtered/`, `enriched/`, `scored/`)
- Granularidade: um arquivo por vaga (`{id_hash}.json`)
- Contrato de I/O JSON documentado
- Ordem do pipeline ajustada: filtro antes do enriquecedor

### Bloco 1 — Infraestrutura base *(em andamento)*
1. ✅ Usuário `sandbox` criado no Windows
2. ✅ WSL2 configurado (`umask=027`, `networkingMode=mirrored`)
3. ✅ Usuário Linux `openclaw` criado no WSL2 (sem symlinks para credenciais do `guilh`)
4. ✅ Estrutura `/home/openclaw/agentic_job_radar/` criada e acessível via `\\wsl$\Ubuntu\home\openclaw\agentic_job_radar\`
5. ✅ Ollama instalado no `sandbox`; modelo `qwen3:8b` instalado e validado
6. ✅ `C:\SharedData\` e `C:\SharedModels\` criadas mas abandonadas (podem ser deletadas)
- Validar que `guilh` consome Ollama via HTTP sem instância própria
- Instalar OpenClaw + Lobster no WSL2 como `openclaw`
- Criar bot Telegram e conectar ao OpenClaw
- Experimento: Telegram → OpenClaw responde usando `qwen3:8b`

### Bloco 2 — Primeiro agente (portao de qualidade)
*(Bloco 0 concluido)*
10. Construir `filtro.py`; respeitar contrato JSON de I/O; expor como step Lobster
11. Rodar sobre amostra de vagas do `job_radar` (copiar manualmente para `raw/` para teste)
12. Comparar resultado com `filter.py` atual — avaliar contra criterio do portao (julgamento do usuario)
13. **Portao go/no-go:** se qualidade insatisfatoria, iterar no prompt antes de continuar

### Bloco 3 — Pipeline completo (só apos portao)
14. Construir `coletor.py` (portar coletores do `job_radar`)
15. Construir `enriquecedor.py` (fetch HTTP da URL da vaga)
16. Construir `scorer.py` (multi-step: requisitos -> senioridade -> dominio -> score)
17. Construir Builder (gera `output/index.html` a partir de `scored/`)
18. Montar workflow Lobster completo: coletor -> filtro -> enriquecedor -> scorer -> builder
19. Validar pipeline end-to-end com 1 semana de dados reais

### Bloco 4 — Expansao (backlog)
- Autenticacao Telegram: whitelist por user ID no OpenClaw
- Multi-agente de codigo: avaliar **Symphony** (OpenClaw) vs **Deep Agents CLI** (LangChain) — ambos cobrem o caso; Deep Agents pode ser alternativa mais madura; escolher apos comparacao no Bloco 4
- Playwright para sites com JS rendering (avaliar quantas fontes realmente precisam antes de implementar)
- Validador de fontes (`source_candidates.yaml`)
- Transparencia de raciocinio em tempo real
- Novos coletores

## Relacao com o `job_radar`

`job_radar` continua rodando via GitHub Actions enquanto `agentic_job_radar` nao estiver validado.
Migracao gradual — quando o novo pipeline produzir qualidade igual ou superior por pelo menos 2 semanas, o `job_radar` pode ser desativado.

---
*Versao 2.4 — Mar 2026*

