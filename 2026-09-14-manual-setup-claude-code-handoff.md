# Manual de Setup — Claude Code (para auditoria e upgrade de outra instância)

> Gerado em 2026-09-14, a partir do ambiente `/root/` (ecossistema ConnectJÁ).
> Objetivo: outro Claude Code lê este arquivo, analisa o `~/.claude` de outra pessoa,
> e aplica tudo que for aplicável para elevar o nível daquele setup ao deste.
> Não é sobre os apps ConnectJÁ (isso é `agentic-os`) — é sobre a camada meta:
> como o Claude Code em si está configurado, equipado e instruído.

---

## 0. Como usar este manual (instrução para o Claude que for executar)

1. Rode o **diagnóstico do alvo** (seção 12) antes de tocar em qualquer arquivo.
2. Compare cada seção deste manual com o que existe no setup alvo.
3. Para cada gap, **pergunte antes de instalar/alterar** algo que grava estado
   (plugins, MCPs, hooks) — isso não é código local reversível, é configuração
   de ambiente. Ver `settings.json` e `.claude.json` são edições sensíveis.
4. Priorize pela seção 13 (ordem de impacto), não pela ordem deste documento.
5. Nunca copie credenciais, tokens ou paths específicos do ConnectJÁ — este
   manual descreve **estrutura e padrões**, não segredos.

---

## 1. Arquitetura de contexto em camadas (CLAUDE.md)

O setup usa **3 arquivos de contexto com responsabilidade separada**, em vez de
um CLAUDE.md monolítico:

| Arquivo | Escopo | Conteúdo |
|---|---|---|
| `~/.claude/CLAUDE.md` | **Como trabalhar** (global, todas as sessões) | Fluxo de desenvolvimento, comandos de apoio, regra de memória |
| `~/.claude/rules/*.md` | **Regras globais por tema** | Ex: `context7.md` — quando usar Context7 MCP |
| `<projeto>/CLAUDE.md` | **O que é este projeto** (checked-in no repo) | Missão, stack, MCPs específicos, design system |

**Por que isso importa:** um CLAUDE.md único vira lixão — mistura "como trabalhar"
(vale pra qualquer projeto) com "o que é este projeto" (só vale aqui). Separar os
dois permite que o global sirva em qualquer repo sem edição, e o local fique
enxuto e específico.

**Upgrade a aplicar num setup que só tem um CLAUDE.md gigante:** dividir em
`~/.claude/CLAUDE.md` (fluxo/regras pessoais) + `CLAUDE.md` do projeto (domínio).
`~/.claude/rules/*.md` é para regras que só se aplicam quando um gatilho aparece
(ex: nome de biblioteca conhecida → força uso de Context7 em vez de treino).

---

## 2. Sistema de memória persistente (fonte da verdade entre sessões)

Localização: `~/.claude/projects/<projeto-slug>/memory/`

Estrutura:
```
memory/
  MEMORY.md              ← índice, uma linha por memória, carregado sempre no contexto
  active-goal.md          ← objetivo ativo da sessão (ver /goal)
  <slug-do-assunto>.md    ← um arquivo por memória, com frontmatter
```

Frontmatter de cada memória:
```yaml
---
name: slug-kebab-case
description: uma linha, usada para decidir relevância em sessões futuras
metadata:
  type: user | feedback | project | reference
---
```

**4 tipos de memória, cada um com corpo estruturado diferente:**
- `user` — quem é o usuário, papel, conhecimento prévio (sem julgamento negativo)
- `feedback` — correções e confirmações de abordagem. Corpo: regra + **Why:** + **How to apply:**
- `project` — fatos/decisões do trabalho em andamento, com prazo. Corpo: fato + **Why:** + **How to apply:**
- `reference` — ponteiro para sistema externo (Linear, Slack, dashboard)

**Regras que fazem esse sistema funcionar:**
- `MEMORY.md` é só índice (uma linha, <150 char cada), nunca conteúdo — evita estourar contexto
- Linkar memórias relacionadas com `[[slug]]`, mesmo que o alvo ainda não exista
- **Não salvar** o que já é derivável do código (padrões, arquitetura, paths) — isso o Claude relê
- Salvar feedback tanto em correção quanto em confirmação silenciosa ("perfeito, mantém assim") —
  se só se grava correção, o sistema fica cada vez mais conservador e perde ganhos validados
- Antes de agir sobre uma memória que cita função/arquivo/porta específicos: **verificar que ainda existe**
  (grep, `ls`, `pm2 list`) — memória é uma foto do passado, não garantia do presente

**Espelho humano-legível:** sync para Obsidian vault (`sync_from_memory.sh`), e captura de
insights ad-hoc via `brain-save.sh "título" "conteúdo"` → `vault/outputs/`.

**Absorção automática:** hook de `Stop` roda um script que lê a sessão e grava um resumo
diário em `vault/outputs/YYYY-MM-DD-brain-absorb.md`. Próxima sessão começa lendo esse arquivo
via hook de `UserPromptSubmit` — contexto entra sozinho, sem o usuário precisar colar nada.

**Upgrade a aplicar num setup sem isso:** o ganho real não é "ter um arquivo de memória", é o
**índice + arquivo-por-assunto + tipos com corpo padronizado**. Um único MEMORY.md gigante
sem por-assunto vira ilegível rápido. Se o alvo já tem algo parecido, comparar contra os 4 tipos
acima — geralmente falta o tipo `feedback` (a maioria só salva fatos de projeto, não como
o usuário gosta de ser corrigido).

---

## 3. Fluxo de trabalho padrão

```
/goal → pedido → /spec → /build → /review → /loop → entrega
```

| Comando | Papel |
|---|---|
| `/goal [descrição]` | bússola da sessão — registra objetivo, marcos, contexto ativado, fora de escopo |
| `/goal status` | mostra progresso |
| `/goal done` | fecha, salva no vault, limpa o goal ativo |
| `/spec` | antes de codar: objetivo, requisitos, restrições, riscos, plano — só pergunta se faltar info crítica |
| `/build` | implementação com decisões técnicas explicadas |
| `/review` | revisão crítica obrigatória — nunca pular. Classifica ✅/⚠️/❌ |
| `/loop` | corrige o que a revisão apontou, reavalia, para quando o ganho for marginal |
| `/RE` | **volta e descarta** — quando a direção saiu errada, não tenta "consertar por cima"; volta
a uma mensagem anterior e descarta o contexto poluído por tentativas fracassadas |

Regra que sustenta tudo: **nunca codar sem spec, nunca entregar sem review crítico.**
Para pedidos rápidos, o fluxo roda internamente e só o resultado final aparece — o usuário
não precisa ver as 5 etapas toda vez.

**Comandos de apoio** (fora do fluxo principal, chamados quando o caso pede):

| Comando | Quando |
|---|---|
| `/escopo [pedido]` | tarefa que tende a crescer sozinha — trava quais arquivos podem ser tocados antes de editar |
| `/gargalo [app/endpoint]` | algo lento — obriga medir antes de otimizar, reverte o que não melhorou o número |
| `/edge [função/fluxo]` | antes de subir código que mexe em dado de cliente — varre entrada, estado, fuso, concorrência, escala, permissão |

**Upgrade a aplicar:** isso é um plugin/skill leve (~7 slash commands custom), não requer
infraestrutura nova. Portável para qualquer stack. O valor está em `/RE` — a maioria dos setups
não tem um jeito explícito de "descartar contexto poluído" e o usuário acaba tentando "conserta
isso" repetidamente sobre uma base já errada.

---

## 4. Persona operacional — Ponytail (lazy senior developer)

Hook de `SessionStart` injeta uma persona comportamental fixa, independente do projeto:
regra de "escada" antes de escrever qualquer código —

```
1. Isso precisa existir? (YAGNI)
2. Já existe no codebase? (reuso > reescrita)
3. Stdlib resolve?
4. Feature nativa da plataforma resolve?
5. Dependência já instalada resolve?
6. Cabe em uma linha?
7. Só então: código mínimo que funciona.
```

Regras adicionais: sem abstração não pedida, deleção > adição, diff mais curto vence
(só depois de entender o problema — nunca antes), comentário só quando explica um "porquê"
não-óbvio, saída em código-primeiro com no máximo 3 linhas de explicação.
Nunca simplifica: validação em fronteira de confiança, tratamento de erro que evita perda de
dado, segurança, acessibilidade, o que foi pedido explicitamente.

Toggle: `/ponytail lite|full|ultra`, ou "stop ponytail" para desligar.

**Upgrade a aplicar:** isso é um hook de `SessionStart` (ou um plugin) que injeta texto de
persona. Baixo custo, alto impacto em setups onde o Claude tende a over-engineer. Só portar se
o estilo do usuário alvo combinar — é uma escolha de tom, não uma correção objetiva.

---

## 5. Hooks configurados

| Hook | Evento | Efeito |
|---|---|---|
| Ponytail | `SessionStart` | injeta a persona da seção 4 |
| Brain-context | `UserPromptSubmit` | injeta caminho do último `brain-absorb.md` para o Claude ler antes de responder |
| MCPmarket sync | `SessionStart` | sincroniza skills baseline de um marketplace de skills |
| Brain-absorb | `Stop` | roda script que resume a sessão e grava em `vault/outputs/` |

**Upgrade a aplicar:** o padrão replicável é **hook de Stop grava resumo → hook de
SessionStart/UserPromptSubmit relê o resumo mais recente**. Isso fecha o loop de continuidade
entre sessões sem exigir que o usuário repita contexto. É o upgrade de maior alavancagem se o
setup alvo não tem nenhuma forma de memória entre sessões.

---

## 6. Skills locais (customizadas para este ambiente — não vêm de nenhum plugin)

Ficam em `~/.claude/skills/<nome>/SKILL.md`. Lista atual (46):

`agentic-os`, `artifacts-builder`, `brand-guidelines`, `canvas-design`,
`changelog-generator`, `competitive-ads-extractor`, `connect`, `connect-apps`,
`content-research-writer`, `context7-mcp`, `developer-growth-analysis`, `docx`,
`domain-name-brainstormer`, `file-organizer`, `find-skills`,
`google-ads-api-account-diagnostics`, `google-ads-api-mcp-setup`,
`google-ads-api-quickstart`, `gsap-motion`, `humanizer`, `image-enhancer`,
`impeccable`, `internal-comms`, `invoice-organizer`, `langsmith-fetch`,
`lead-research-assistant`, `mcp-builder`, `meeting-insights-analyzer`, `pdf`,
`pptx`, `raffle-winner-picker`, `scroll-world`, `skill-creator`, `skill-share`,
`slack-gif-creator`, `tailored-resume-generator`, `task-observer`,
`theme-factory`, `threejs-webgl`, `twitter-algorithm-optimizer`,
`ugc-higgsfield`, `video-downloader`, `webapp-testing`, `xlsx`

**As que valem a pena portar para qualquer setup técnico (não são ConnectJÁ-specific):**
- `impeccable` — auditoria/polimento de UI existente (`audit`, `polish`, `layout`)
- `gsap-motion` / `threejs-webgl` / `scroll-world` — trio de motion/3D/scrollytelling web
- `mcp-builder` — construir servidor MCP próprio (Python FastMCP ou Node/TS SDK)
- `skill-creator` — criar novas skills seguindo o formato certo
- `find-skills` — descobrir skills instaláveis quando falta capacidade
- `webapp-testing` — Playwright para testar app local antes de dar por concluído
- `file-organizer` / `docx` / `pdf` / `pptx` / `xlsx` — utilitários de documento genéricos
- `changelog-generator` — git log → changelog user-facing
- `task-observer` — (ver plugins community, seção 8)

**As específicas do ConnectJÁ (não portar, mas usar como *padrão* de skill de domínio):**
`connect`, `connect-apps`, `google-ads-api-*`, `ugc-higgsfield`, `agentic-os`.
O padrão que vale copiar: **uma skill por domínio de negócio, com trigger explícito em
português E inglês, listando exatamente quais MCPs ela deve acionar.**

---

## 7. Skills de sistema / community mais relevantes (via plugins)

Não são locais — vêm de plugins instalados. As que carregam mais peso operacional:

| Skill | O que faz | Por que importa |
|---|---|---|
| `superpowers:using-superpowers` | meta-skill: força checar se existe skill aplicável ANTES de qualquer ação, inclusive perguntas de esclarecimento | Resolve o problema clássico de "esqueci que existe uma skill pra isso" |
| `superpowers:brainstorming` | obrigatório antes de qualquer trabalho criativo (features, componentes) | Trava contra "implementar direto sem explorar intenção" |
| `superpowers:systematic-debugging` | obrigatório antes de propor fix em bug/teste falhando | Trava contra "achismo de causa raiz" |
| `superpowers:writing-plans` / `executing-plans` | plano formal antes de tarefa multi-etapa | |
| `superpowers:verification-before-completion` | não declarar tarefa pronta sem verificar | |
| `superpowers:using-git-worktrees` | isolamento de workspace para feature work | |
| `superpowers:requesting-code-review` / `receiving-code-review` | protocolo de review entre agentes | |
| `impeccable` | design/redesign/auditoria de UI, cobre acessibilidade, hierarquia, tokens | |
| `code-review` | `/code-review [low..max|ultra] [--comment|--fix]` — review do diff atual ou de um PR | |
| `simplify` | review focado só em qualidade (reuso/simplificação/eficiência), sem caçar bug | |
| `security-review` | | |
| `fewer-permission-prompts` | escaneia transcript, gera allowlist pro `settings.json` | Reduz atrito de permissão sem abrir mão de segurança |
| `update-config` | edita `settings.json`/`settings.local.json` — hooks, permissões, env vars | Sempre que o pedido for "sempre que X, faça Y" — isso é hook, não memória |
| `loop` | roda um prompt/slash-command em intervalo recorrente, com auto-pace | |
| `schedule` | agentes cloud em cron | |
| `claude-mem:*` (plugin) | memória automática por observação (diferente do sistema de memory/ manual da seção 2) — indexa sessões passadas e injeta contexto relevante na 2ª sessão em diante | Complementa (não substitui) o sistema de memória manual |
| `artifact-design` / `artifact-capabilities` / `artifact-diagramming` / `dataviz` | ler ANTES de publicar qualquer Artifact — calibra investimento de design e capacidades de runtime | |
| `frontend-design` (oficial Anthropic) | direção de arte para não cair em "cara de IA genérica" | Aqui o setup **sobrepõe** essa skill com identidade de marca (ver seção abaixo) |

**Padrão de precedência que vale copiar:** quando uma skill genérica ("evite roxo, evite Inter,
é clichê de IA") conflita com identidade de marca do usuário, o CLAUDE.md do projeto resolve
explicitamente o conflito por escrito — em vez de deixar ambíguo qual instrução vence.

---

## 8. MCPs conectados — inventário funcional

| MCP | Categoria | Uso típico |
|---|---|---|
| Meta ADS | Marketing | campanhas/ad sets/criativos/insights/públicos/pixels/catálogos/Ad Library |
| Supermetrics | Dados | métricas de Google Ads/Meta/GA4/LinkedIn/TikTok — **assíncrono**, precisa poll |
| Windsor.ai | Dados/Ação | 350+ conectores, leitura E escrita de volta na plataforma origem |
| Higgsfield | Criativo | imagem/vídeo/áudio, upscale, dublagem, previsão de viralidade, website builder |
| Google Calendar | Produtividade | eventos, agenda, sugestão de horário |
| Computer Use | Controle de desktop | screenshot + clique/teclado no desktop do usuário, com controle de acesso por app/tier |
| Chrome (claude-in-chrome) | Navegador | DOM-aware, mais rápido que computer-use pra tarefa web |
| Playwright | Automação/teste | browser automation determinístico, ótimo pra `webapp-testing` |
| connectja-infra (custom) | Infra própria | `pm2_apps`, `listening_ports`, `check_path` — MCP caseiro pra não inventar porta/path |
| Serena | Código | navegação semântica de símbolo (find/rename/refactor), memória de projeto própria |
| Sentry | Observabilidade | busca de issue, análise com Seer, eventos |
| Context7 | Docs | documentação atualizada de biblioteca — **usar sempre que citar lib/framework/SDK**, mesmo achando que já sabe |
| Google Ads (nativo) | Marketing | `list_accessible_customers`, `search` (GAQL) |
| mcp-registry | Meta | descobrir/sugerir outros MCPs conectáveis |
| vault-notes | Arquivo | ler/escrever no vault Obsidian via filesystem MCP |
| scheduled-tasks | Automação | criar/listar/rodar tarefas agendadas |

**MCP mais replicável e barato de construir:** `connectja-infra`. É um MCP caseiro de ~3 tools
que resolve um problema real e recorrente — "não confiar em memória de porta/path sem
verificar". Qualquer setup com múltiplos serviços locais/PM2 se beneficia do mesmo padrão:
um MCP fino que só expõe `list running services / list ports / check path exists`, pra que o
Claude pare de inventar porta e sempre valide contra o estado real. Construível com a skill
`mcp-builder` em menos de 100 linhas.

**Regra operacional que vale copiar:** MCPs de dados assíncronos (Supermetrics) exigem
**poll explícito** documentado na instrução do próprio servidor — não assumir resposta síncrona.

---

## 9. Plugins instalados (marketplaces)

`codex-image`, `obsidian-skills`, `firecrawl`, `mcpmarket-me`, e o pacote grande
`anthropic-skills` (dezenas de skills oficiais — design, ads, docx/pptx/xlsx, memory tools),
mais os pacotes verticais: `design:*`, `sales:*`, `marketing:*`, `data:*`,
`small-business:*`, `stripe:*`, `adobe-for-creativity:*`, `cloudinary:*`, `superpowers:*`,
`ponytail:*`, `claude-mem:*`, `postiz:*`.

**Padrão que vale copiar:** os plugins verticais (`marketing:*`, `small-business:*`, `sales:*`)
funcionam como **kits de skill prontos por área de negócio** — antes de escrever uma skill
custom do zero para algo genérico ("relatório de performance", "revisão de contrato",
"triagem de lead"), checar se já existe no plugin oficial da área.

**Orquestrador local — `marketing-senior`:** plugin/skill própria que não executa nada sozinha,
só **roteia** para a combinação certa de skill+MCP na ordem certa, com gate de qualidade antes
da entrega. Dispara em qualquer menção a marketing/ads/copy/campanha, mesmo sem pedido explícito
de skill. Esse é o padrão de maior alavancagem pra portar: **um orquestrador por domínio de alto
volume de pedido**, que decide a combinação certa em vez de o usuário ter que lembrar qual skill
+ qual MCP usar toda vez.

---

## 10. Design system / padrões de produto (se o alvo também builda frontend)

Não é sobre Claude Code em si, mas é um padrão de instrução que vale copiar **como técnica**,
não como valores: o CLAUDE.md do projeto trava paleta de cor, fonte, raio de borda, padrão de
glassmorphism, e qual stack é obrigatória (vanilla vs React) por *tipo* de app — evita que cada
sessão nova reinvente a decisão visual. Se o alvo tem mais de 2-3 frontends internos, vale a
pena ele documentar o próprio design system da mesma forma.

---

## 11. Meta-regra de comportamento operacional (a mais importante deste manual)

O CLAUDE.md do projeto define um checklist de 3 perguntas **antes de qualquer tarefa**:

```
1. Existe skill que cobre isso? → leia o SKILL.md e use, não pergunte.
2. Existe MCP que executa a ação real (não só descreve)? → use, ofereça executar.
3. Existe combinação skill+MCP que entrega mais que o pedido? → use, sem pedir autorização.
```

Regra companheira: **"EXECUTE PRIMEIRO — entregue, explique depois se necessário."**

Isso é o que faz o resto do setup (seções 1-10) funcionar como sistema em vez de inventário
morto de ferramentas. Um setup pode ter todos os MCPs e skills do mundo instalados e ainda
assim o Claude default para resposta genérica se não houver essa regra explícita mandando
verificar-e-usar antes de responder.

**Isto é o upgrade #1 a aplicar em qualquer setup-alvo, antes de qualquer instalação de skill
ou MCP novo.** Sem essa regra, instalar mais capacidade só aumenta a lista que o Claude ignora.

---

## 12. Checklist de auditoria do setup-alvo

Rodar nesta ordem, com dados reais (nunca assumir):

```bash
# 1. O que já existe
cat ~/.claude/CLAUDE.md 2>/dev/null
ls ~/.claude/skills/ 2>/dev/null
cat ~/.claude/settings.json 2>/dev/null
ls ~/.claude/rules/ 2>/dev/null
find ~/.claude/projects -maxdepth 2 -name "memory" -type d 2>/dev/null

# 2. Plugins e MCPs configurados
claude mcp list 2>/dev/null
cat ~/.claude.json 2>/dev/null | grep -A5 '"mcpServers"'
```

Depois, comparar contra cada seção deste manual e listar gaps em 3 baldes:

- **Estrutural** (fácil, sem risco): dividir CLAUDE.md em camadas (seção 1), criar
  `memory/MEMORY.md` com o formato de 4 tipos (seção 2), adicionar checklist skill→MCP
  (seção 11)
- **Comportamental** (precisa confirmação do usuário-alvo, é escolha de estilo): persona tipo
  Ponytail (seção 4), fluxo `/goal→spec→build→review→loop` (seção 3)
- **Infraestrutural** (requer instalar plugin/MCP novo — **sempre perguntar antes**): hooks de
  Stop/SessionStart para memória automática (seção 5), MCP caseiro tipo `connectja-infra`
  (seção 8), plugins verticais por área de negócio (seção 9)

Nunca aplicar o balde 3 sem confirmação explícita — instalar plugin/MCP é mudança de ambiente,
não edição de arquivo reversível por `git`.

---

## 13. Ordem de prioridade se o usuário disser só "aplica tudo que der pra melhorar"

1. Regra de comportamento da seção 11 (checklist skill→MCP antes de responder) — maior
   alavancagem, custo zero, é só texto no CLAUDE.md
2. Separar CLAUDE.md em camadas (seção 1) — se já for um arquivo único grande
3. Sistema de memória com os 4 tipos + índice (seção 2) — se não existir nada parecido
4. `/RE` como conceito (seção 3) — mesmo sem implementar os outros 8 comandos, este sozinho
   já muda o hábito de "não tentar consertar por cima de direção errada"
5. Hook Stop→SessionStart para continuidade entre sessões (seção 5) — **perguntar antes**,
   é mudança de `settings.json`
6. MCPs/skills específicos — só depois do usuário confirmar quais domínios de negócio ele
   realmente opera (marketing? dados? design? infra?) — não instalar por completude
