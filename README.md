# agentic_job_radar

Plano de Projeto (versao 2.1)

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
     -> step 2: enriquecedor -> enriched/
     -> step 3: filtro       -> filtered/
     -> step 4: scorer       -> scored/

Builder (script Python) -> scored/ -> output/index.html (revisao estatico)
```

### Componentes

1. **OpenClaw**
   - Interface e plataforma de agentes (Telegram -> usuario `guilh`)
   - Suporte a expansao futura de pipelines multi-agente

2. **Lobster**
   - Orquestrador deterministico do pipeline
   - YAML/flow fixo: `coletor -> enriquecedor -> filtro -> scorer -> builder`
   - Passa JSON entre etapas; LLMs executam analise, Lobster controla o fluxo

3. **Workers Python**
   - Portados/adaptados do `job_radar`
   - Contrato de entrada/saida via JSON

4. **Builder**
   - Gera `output/index.html` a partir de `scored/`
   - Sem servidor: abre no browser do usuario `guilh`

## Decisoes de arquitetura (principais)

| Decisao | Escolha | Motivo |
|---|---|---|
| Runtime de modelo | Ollama | gratuito, suporte a GPU, compativel com OpenAI API |
| Modelo inicial | Qwen 2.5 7B | bom suporte para PT e tool-use para 7B |
| Interface e plataforma de agente | OpenClaw + Lobster | OpenClaw: extensivel; Lobster: determinismo nativo |
| Canal de interface | Telegram | API oficial do Telegram; reduz risco comparado a WhatsApp (Baileys) |
| Orquestracao | Lobster (nao LangChain) | pipeline sequencial fixo; determinismo > dinamismo |
| Isolamento de seguranca | usuario `sandbox` (sem Docker) | evita acesso a credenciais pessoais |
| Skills externas | nenhuma (ClawHub) | reduzir risco de submissions maliciosas |
| Revisao | HTML estatico | sem servidor; simples e previsivel |
| Dedup | `seen_jobs.json` | mesmo schema/lógica do `job_radar` |
| Bind address OpenClaw | `127.0.0.1` | nao expor na rede (0.0.0.0) |
| Expansao futura | Symphony | pipelines de codigo autônomas no ecossistema |

## Estrutura de dados (Windows)

```
guilh (admin)
  -> consumo/revisao
  -> leitura de C:\SharedData\agentic_job_radar\
  -> OpenClaw via Telegram

sandbox (padrao)
  -> Ollama instalado (OLLAMA_MODELS em `C:\SharedModels\`)
  -> OpenClaw instalado e rodando como servico
  -> scripts Python rodam diretamente no `sandbox`, lendo/gravando em `C:\SharedData\agentic_job_radar\`
  -> guilh consome Ollama via HTTP em `localhost:11434` (sem instancia propria)

C:\SharedModels\                          -> modelo (Qwen 2.5 7B) compartilhado
C:\SharedData\agentic_job_radar\
  raw\                                    -> vagas brutas
  enriched\                               -> html completo da pagina + metadados adicionados
  filtered\                               -> apos filtros (ex.: localizacao/titulo)
  scored\                                 -> apos scoring + analise
  seen_jobs.json                          -> deduplicacao persistente
  logs\                                   -> raciocinio/explicacoes passo a passo por vaga
  output\                                 -> HTML para revisao
```

## Seguranca (contrato do ambiente)

- Isolamento por usuario `sandbox` (sem containerização)
- OpenClaw bind configurado para `127.0.0.1` (nao `0.0.0.0`)
- nenhum usuario `sandbox` com credenciais pessoais do `guilh`
- nenhuma skill de terceiros

## Portao go/no-go (Bloco 2)

Sobre uma amostra do mesmo dataset do `job_radar`:
- o agente de filtro deve chegar nos mesmos vereditos (passa/nao passa) em pelo menos 80% dos casos
- nenhuma vaga claramente US-only deve passar
- nenhuma vaga reconhecidamente relevante deve ser eliminada

Se falhar: iterar no prompt/criterio antes de seguir para o Bloco 3.

## Roadmap (visao v2.1)

### Bloco 0 - Definicao de schema (antes do Bloco 2)
Definir campos minimos de cada diretoria:
- `raw/`      -> o que o coletor grava
- `enriched/` -> o que o enriquecedor adiciona ao raw
- `filtered/` -> o que sobra apos filtro de localizacao e titulo
- `scored/`   -> o que o scorer grava

Isso garante contratos claros entre agentes.

### Bloco 1 - Infraestrutura base
1. Criar usuario `sandbox` no Windows
2. Criar `C:\SharedModels\` e `C:\SharedData\agentic_job_radar\` com permissoes para `guilh` e `sandbox`
3. Instalar Ollama no `sandbox`; apontar `OLLAMA_MODELS` para `C:\SharedModels\`; configurar bind em `127.0.0.1`
4. Baixar Qwen 2.5 7B via Ollama no `sandbox` e validar execucao com GPU
5. Instalar OpenClaw no `sandbox`; bind em `127.0.0.1`; configurar para usar apenas `C:\SharedData\agentic_job_radar\` (sem container)
6. Instalar Lobster no `sandbox`; validar integracao com OpenClaw
7. Criar bot no Telegram (@BotFather); conectar ao OpenClaw; parear com conta do `guilh`
8. Experimento simples: message via Telegram -> resposta local usando Qwen
9. Validar que `guilh` consome Ollama via HTTP (localhost:11434)

### Bloco 2 - Primeiro agente (portao de qualidade)
1. Skill de filtro de localizacao em Python; expor como step do Lobster
2. Rodar sobre amostra do `job_radar` (copiar manualmente para `enriched/` para teste)
3. Comparar com `filter.py` atual e avaliar contra o portao
4. Executar go/no-go; iterar antes de seguir se necessario

### Bloco 3 - Pipeline completo (apos portao)
1. Coletor (portar do `job_radar`)
2. Enriquecedor (fetch completo da pagina da vaga)
3. Scorer (requisitos -> senioridade -> dominio -> score)
4. Builder (gerar `output/index.html` a partir de `scored/`)
5. Workflow Lobster completo: coletor -> enriquecedor -> filtro -> scorer -> builder
6. Validar end-to-end com 1 semana de dados reais

### Bloco 4 - Expansao (backlog)
- Symphony para multi-agente de codigo
- Playwright para sites com JS rendering
- Validador de fontes (`source_candidates.yaml`)
- Transparencia de raciocinio em tempo real
- Novos coletores

## Relacao com o `job_radar`

O `job_radar` continua rodando via GitHub Actions enquanto `agentic_job_radar` nao for validado.
Quando o novo pipeline produzir qualidade igual ou superior por 2 semanas, o `job_radar` pode ser desativado.

