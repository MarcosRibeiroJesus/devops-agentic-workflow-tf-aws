# Material de Apoio — DevOps com Agentes de IA
### Claude Code, Terraform e AWS · Pós-graduação em Engenharia de IA Aplicada — UNIPDS

Este documento acompanha os slides da live e existe para um único propósito: te dar profundidade suficiente em cada conceito para que, no dia, você possa falar sobre eles com segurança — os slides mostram só a definição resumida; aqui está o "porquê" e o "como funciona por baixo do capô", como nos módulos de fundamentos da Anthropic.

Todas as definições abaixo foram verificadas na documentação oficial (`code.claude.com/docs`, `platform.claude.com/docs`, `claude.com/blog`, `anthropic.com/engineering` e o anúncio original do MCP). As citações diretas estão em itálico.

> **Regra de manutenção**: este documento acompanha o `slides.html` linha a linha. Toda vez que um slide novo é criado ou um slide existente é alterado, esta versão é revisada no mesmo momento — nunca fica defasada em relação ao que está na tela.

---

## Sumário

1. [Módulo 0 — Do workflow tradicional ao agêntico](#módulo-0)
   - 0.1 O problema real
   - 0.2 Um incidente, dois jeitos de resolver
   - 0.3 Arquitetura do workshop
2. [Módulo 1 — Fundamentos](#módulo-1)
   - 1.1 O loop agêntico do Claude Code
   - 1.2 CLAUDE.md
   - 1.3 Skills
   - 1.4 Agents (Subagents)
   - 1.5 Rules
   - 1.6 Agents × Skills × Rules — critério de decisão
   - 1.7 O harness: como as peças viram um agente
   - 1.8 Terraform, em uma imagem
   - 1.9 AWS: onde a infraestrutura vai morar
   - 1.10 Memória (CLAUDE.md vs. Auto Memory)
   - 1.11 MCP — Model Context Protocol
3. [Módulo 2 — Segurança](#módulo-2)
   - 2.1 Por que instrução não é controle
   - 2.2 Hooks
   - 2.3 Permissions
   - 2.4 Guardrails para infraestrutura
4. [Módulo 3 — Integrações](#módulo-3)
5. [Módulo 4 — Demonstração ao vivo](#módulo-4)
   - 4.1 Estrutura do projeto e o CLAUDE.md real
   - 4.2 Anatomia completa de um SKILL.md
   - 4.3 As skills do projeto
   - 4.4 Exemplo real: a skill deploy
   - 4.5 O fluxo de trabalho, com e sem skills
   - 4.6 Ordem de execução ao vivo
6. [Glossário rápido](#glossário)
7. [Fontes](#fontes)

---

<a name="módulo-0"></a>
## Módulo 0 — Do workflow tradicional ao agêntico

### 0.1 O problema real

Um engenheiro DevOps tradicional gasta a maior parte do tempo em trabalho de **tradução**: pegar uma decisão de arquitetura já tomada (ou tomada na hora, sob pressão) e traduzi-la em artefatos de configuração — HCL do Terraform, YAML de pipelines de CI/CD, JSON de políticas IAM, manifests do Kubernetes, e por aí vai. Esse trabalho de tradução consome tempo, exige memorizar sintaxe e providers, e empurra a revisão de arquitetura e segurança para "quando sobrar tempo" — que, na prática, é sempre depois do prazo.

O workflow agêntico inverte essa equação. O engenheiro deixa de ser o tradutor e passa a ser o **revisor e definidor de regras**: ele descreve a intenção em linguagem natural, define as convenções e os limites de segurança do projeto (é exatamente isso que os mecanismos do Módulo 1 e 2 fazem), e o agente — Claude Code, no nosso caso — assume a tradução para HCL/YAML/JSON. O tempo do engenheiro se desloca de "como escrevo isso" para "isso está arquiteturalmente correto e seguro".

Isso não elimina a responsabilidade do engenheiro — pelo contrário, a concentra onde ela importa mais.

### 0.2 Um incidente, dois jeitos de resolver

A forma mais concreta de mostrar essa inversão é comparar o mesmo incidente resolvido dos dois jeitos. Cenário: *"o deploy está com erro 5xx."*

**Sem agente**, o DevOps é o executor de cada etapa, uma por uma: recebe o alerta (PagerDuty, Slack, ticket) → investiga sozinho, decidindo onde olhar (logs, métricas, traces) → formula uma hipótese e corrige o código, a config ou a infra → testa e abre um PR para revisão humana → faz o deploy e observa (canary/blue-green). Se o deploy falhar, o ciclo inteiro recomeça do zero, e cada uma dessas decisões passa pela mão do engenheiro.

**Com agente**, o DevOps deixa de executar comando por comando e passa a definir o objetivo e os guardrails: o que precisa ser resolvido, e dentro de quais limites. Esse par (objetivo + limites) é entregue ao **harness**, que carrega as permissões, políticas, testes e critérios de aprovação configurados de antemão. É o harness que então aciona um ciclo de investigar → implementar → validar, usando como referência o **"ground truth" do ambiente** (resultados de testes, scans de segurança, saída de um `terraform plan`) a cada passo — exatamente o mecanismo que a Anthropic descreve no artigo *"Building Effective Agents"*:

> *"During execution, it's crucial for the agents to gain 'ground truth' from the environment at each step (such as tool call results or code execution) to assess its progress."*

O resultado ainda passa por um PR com aprovação humana antes do deploy, e a observação pós-deploy alimenta o agente de volta caso algo dê errado — o humano continua no controle dos pontos de decisão que importam, só que não precisa mais operar cada comando manualmente. É o que a Anthropic chama de **autonomia limitada** (*bounded autonomy*):

> *"Agents can then pause for human feedback at checkpoints or when encountering blockers. The task often terminates upon completion, but it's also common to include stopping conditions (such as a maximum number of iterations) to maintain control."*

**A diferença mais profunda por trás disso** é a diferença entre um **workflow** e um **agente**, formalmente definida no mesmo artigo:

> *"Workflows are systems where LLMs and tools are orchestrated through predefined code paths."*
> *"Agents, on the other hand, are systems where LLMs dynamically direct their own processes and tool usage, maintaining control over how they accomplish tasks."*

Um pipeline de CI/CD tradicional é sempre a mesma esteira, A → B → C → D, determinística. Um workflow agêntico é um **loop de controle**: observar, agir, observar de novo, até bater a meta — adaptável a cada execução, porque a próxima ação depende do que aconteceu na anterior, não de um roteiro fixo escrito com antecedência.

### 0.3 Arquitetura do workshop

O projeto que construiremos ao vivo é deliberadamente simples na superfície — um site estático — para que toda a atenção da audiência fique no *processo* (como o agente chega lá), não na complexidade da infraestrutura em si:

- **Amazon S3**: bucket que armazena os arquivos estáticos do site (HTML/CSS/JS).
- **Amazon CloudFront**: CDN na frente do bucket, responsável por HTTPS (via certificado ACM), cache e distribuição global.
- **IAM**: política de acesso do CloudFront ao bucket seguindo o princípio de *least privilege* — o mínimo de permissão necessária, nada mais.
- **(Opcional) Route 53**: DNS customizado, se houver domínio próprio.

A regra combinada com a audiência: **nenhuma linha de Terraform ou YAML é escrita manualmente**. Todo `.tf`, todo `settings.json`, toda política IAM sai do agente — nosso trabalho é dar contexto e colocar guardrails.

---

<a name="módulo-1"></a>
## Módulo 1 — Fundamentos

Esta seção é o coração da live. Depois de entender o loop que move o Claude Code, os mecanismos seguintes (CLAUDE.md, Skills, Agents, Rules) respondem à mesma pergunta — *"como o Claude Code sabe o que fazer e quando"* — mas em momentos e granularidades diferentes. Entender a diferença entre eles é o que separa um uso raso do Claude Code de um setup verdadeiramente agêntico.

### 1.1 O loop agêntico do Claude Code

**O que é.** Segundo a documentação oficial, toda tarefa que você dá ao Claude Code passa por três fases que se repetem e se misturam:

> *"When you give Claude a task, it works through three phases: gather context, take action, and verify results. These phases blend together. Claude uses tools throughout, whether searching files to understand your code, editing to make changes, or running tests to check its work."*

Ou seja: **reunir contexto** (ler arquivos, rodar buscas, checar o estado atual) → **agir** (editar código, rodar comandos, chamar ferramentas) → **verificar** (rodar testes, conferir o resultado) → repetir quantas vezes for preciso, até a tarefa estar completa. Não é um script linear com passos fixos — a próxima ação do Claude depende do que ele aprendeu na etapa anterior:

> *"Claude decides what each step requires based on what it learned from the previous step, chaining dozens of actions together and course-correcting along the way."*

**Você está no centro do loop.** A documentação é explícita sobre isso: mesmo com o Claude operando de forma autônoma dentro do loop, você pode interromper a qualquer momento para redirecionar, adicionar contexto ou pedir uma abordagem diferente — o loop não te tira de controle, te tira apenas da execução manual de cada passo:

> *"You're part of this loop too. You can interrupt at any point to steer Claude in a different direction, provide additional context, or ask it to try a different approach. Claude works autonomously but stays responsive to your input."*

**O que empodera o loop** — dois componentes, mais a camada que os conecta:

- **Modelos**, que raciocinam: Sonnet (a maioria das tarefas de código), Opus (raciocínio mais forte para decisões arquiteturais complexas) e Haiku (respostas rápidas e mais baratas para tarefas simples).
- **Ferramentas**, que agem: leitura e edição de arquivos, busca em código, execução de comandos (`Bash`), navegação web e integração com ferramentas de inteligência de código. A documentação agrupa isso em cinco categorias oficiais: *File operations*, *Search*, *Execution*, *Web* e *Code intelligence*.
- **O harness do Claude Code**, que sustenta o loop e transforma o modelo bruto em um agente capaz:

> *"The agentic loop is powered by two components: models that reason and tools that act. Claude Code serves as the agentic harness around Claude: it provides the tools, context management, and execution environment that turn a language model into a capable coding agent."*

Na prática, o harness cobre três funções: **contexto** (o que o modelo sabe: CLAUDE.md, Auto Memory, arquivos do projeto), **execução** (quais ferramentas ele pode chamar e como) e **orquestração** (como o loop decide a próxima ação e quando parar). Os Módulos 1 e 2 inteiros são, no fundo, uma explicação detalhada de como cada peça do harness é configurada.

**Exemplo prático: um EKS mal configurado.** Suponha que um `terraform plan` falha porque um bloco do cluster EKS está com a sintaxe errada ou uma configuração obrigatória foi esquecida. O loop resolve assim:

1. **Reunir contexto** — lê os arquivos `.tf` relevantes, checa a configuração atual do cluster, localiza exatamente onde está o erro.
2. **Agir** — corrige o bloco com erro de sintaxe, ou adiciona a configuração que estava faltando.
3. **Verificar** — roda `terraform plan` de novo. Se o plano sai limpo, a tarefa está feita. Se não, o loop volta ao passo 1 com o novo erro como contexto atualizado.

Esse ciclo de três passos é literalmente o mesmo padrão usado em qualquer correção que o Claude Code faz em código, não só em infraestrutura — é por isso que a documentação chama de "loop", não de "pipeline": ele se adapta ao que encontra em cada iteração, em vez de seguir uma sequência fixa de antemão.

---

### 1.2 CLAUDE.md

**O que é.** CLAUDE.md é um arquivo Markdown que dá ao Claude Code instruções persistentes sobre um projeto, um workflow pessoal, ou uma organização inteira. Ele é lido no início de cada sessão e permanece no contexto durante toda a conversa.

> *"CLAUDE.md files are markdown files that give Claude persistent instructions for a project, your personal workflow, or your entire organization."*

**Por que existe.** Sem ele, cada sessão do Claude Code começa do zero — sem saber que comando roda os testes, sem saber a estrutura de diretórios, sem saber que o time usa `pnpm` e não `npm`. CLAUDE.md é a forma de você escrever uma vez o que, de outra forma, teria que reexplicar em toda conversa nova.

**Onde ele pode viver** (da abrangência mais ampla para a mais específica — e nessa ordem entram no contexto):

| Escopo | Local | Uso |
|---|---|---|
| Política gerenciada | `/etc/claude-code/CLAUDE.md` (Linux) | Padrões da organização, geridos por TI/DevOps — não pode ser sobrescrito |
| Usuário | `~/.claude/CLAUDE.md` | Preferências pessoais válidas em todos os projetos |
| Projeto | `./CLAUDE.md` ou `./.claude/CLAUDE.md` | Instruções compartilhadas com o time via controle de versão |
| Local | `./CLAUDE.local.md` | Preferências pessoais daquele projeto — deve entrar no `.gitignore` |

Todos os arquivos encontrados são **concatenados** no contexto (não se sobrescrevem), na ordem da raiz do sistema de arquivos até o diretório de trabalho — ou seja, instruções mais próximas de onde você rodou o Claude Code são lidas por último (e, na prática, tendem a ter mais peso).

**Quando adicionar algo ao CLAUDE.md** — regra prática da própria documentação: adicione quando o Claude comete o mesmo erro pela segunda vez, quando uma revisão de código pega algo que ele "deveria ter sabido", quando você digita a mesma correção duas sessões seguidas, ou quando um novo colega do time precisaria do mesmo contexto para ser produtivo.

**Boas práticas de escrita:**
- **Tamanho**: manter abaixo de 200 linhas por arquivo — arquivos maiores consomem mais contexto e reduzem a aderência do agente às instruções.
- **Estrutura**: usar cabeçalhos e bullets — o Claude "escaneia" a estrutura do mesmo jeito que uma pessoa lendo.
- **Especificidade**: prefira "use indentação de 2 espaços" a "formate o código direito"; prefira "rode `npm test` antes de commitar" a "teste suas mudanças".
- **Consistência**: instruções conflitantes entre arquivos fazem o Claude escolher uma arbitrariamente.

**Ponto crítico de segurança conceitual**: CLAUDE.md é entregue como uma mensagem de usuário após o system prompt — o Claude tenta segui-lo, mas **não há garantia de cumprimento estrito**. Não é uma camada de aplicação (enforcement); é contexto. Isso é o gancho para o Módulo 2 (Hooks e Permissions).

No nosso workshop, o CLAUDE.md do projeto contém, entre outras coisas, a stack (Terraform + provider AWS), a convenção de nomenclatura de recursos, a região AWS padrão, e a instrução mais importante do arquivo: nunca escrever Terraform ou CI/CD à mão — usar sempre a skill certa. Veja o trecho real na seção 4.1.

---

### 1.3 Skills

**O que é.** Skills estendem o que o Claude pode fazer. Você cria um arquivo `SKILL.md` com instruções, e o Claude adiciona isso ao seu conjunto de ferramentas — usando a skill quando é relevante, ou sendo invocado diretamente com `/nome-da-skill`.

> *"Only the name and description load at session start; the full body loads when Claude invokes the skill."*

**A diferença fundamental para o CLAUDE.md** é o carregamento em duas camadas: no início da sessão, só o **nome** e a **descrição** da skill entram no contexto (custo quase zero). O corpo completo — o passo a passo, os exemplos, os scripts auxiliares — só é carregado no momento em que a skill é efetivamente invocada. Isso significa que você pode ter dezenas de skills registradas sem pagar o custo de contexto de nenhuma delas até precisar.

**Quando criar uma skill** (critério oficial): quando você fica colando as mesmas instruções, checklist ou procedimento multi-etapa no chat repetidamente, ou quando uma seção do CLAUDE.md cresceu a ponto de virar um procedimento em vez de um fato estático.

**Detalhe de implementação relevante**: comandos customizados (antigo `.claude/commands/`) foram unificados com skills — um arquivo em `.claude/commands/deploy.md` e uma skill em `.claude/skills/deploy/SKILL.md` criam o mesmo comando `/deploy` e funcionam da mesma forma. Skills adicionam recursos extras: um diretório para arquivos de apoio, frontmatter para controlar quem invoca (você ou o próprio Claude), e a possibilidade de o Claude carregá-las automaticamente quando percebe que são relevantes.

No workshop, a skill `deploy` (entre outras) encapsula o checklist completo de publicação: build do site → upload para S3 → invalidação de cache do CloudFront → verificação de saúde do endpoint. Ela é uma das quatro skills que testamos ao vivo — veja a seção 4.3 para a lista completa e a 4.4 para o arquivo real.

---

### 1.4 Agents (Subagents)

**O que é.** Subagents são instâncias de agente separadas que o agente principal pode disparar para lidar com subtarefas focadas — cada um roda em sua própria janela de contexto, isolada da conversa principal.

> *"The only thing that returns to your main session is the subagent's final message plus metadata."*

**Os quatro motivos para usar um subagent**, segundo a documentação:

1. **Isolamento de contexto** — um subagent de pesquisa pode vasculhar dezenas de arquivos sem que nenhum desse conteúdo intermediário "suje" a conversa principal; só o resumo final volta.
2. **Paralelização** — múltiplos subagents rodam ao mesmo tempo, então subtarefas independentes terminam no tempo da mais lenta, não na soma de todas.
3. **Instruções e conhecimento especializados** — cada subagent pode ter um *system prompt* próprio, com expertise específica que seria ruído no agente principal.
4. **Restrição de ferramentas** — um subagent pode ser limitado a um subconjunto de ferramentas (ex: só `Read`, `Grep`, `Glob` — nunca `Write` ou `Bash`), reduzindo o raio de ação possível.

**Duas formas de definir um subagent:**
- **Baseado em arquivo** (o que faremos ao vivo): markdown em `.claude/agents/nome.md`, descoberto automaticamente pelo Claude Code.
- **Programático**: via SDK, passando um dicionário `agents` na configuração — recomendado para aplicações construídas sobre o Agent SDK.

**O que um subagent herda e o que não herda** — isso é essencial para entender por que ele é "isolado":

| Recebe | Não recebe |
|---|---|
| Seu próprio system prompt + o prompt da chamada | O histórico de conversa do agente pai |
| O CLAUDE.md do projeto | O system prompt do agente pai |
| As definições de ferramentas (todas, ou o subconjunto de `tools`) | Skills pré-carregadas (a menos que listadas explicitamente) |

**Invocação**: automática (o Claude decide, com base na `description` do subagent — por isso escrever uma descrição clara e específica importa) ou explícita (mencionar o nome do subagent no prompt: *"use o agente terraform-reviewer para..."*).

**Limites de escala** (relevantes se a audiência perguntar sobre custo/controle): é possível limitar a profundidade de subagents aninhados (`CLAUDE_CODE_MAX_SUBAGENT_SPAWN_DEPTH`, padrão 3), quantos rodam simultaneamente (`CLAUDE_CODE_MAX_CONCURRENT_SUBAGENTS`, padrão 20) e o gasto máximo em dólares de uma query inteira (`maxBudgetUsd`).

No workshop, o subagent `terraform-reviewer` terá acesso somente de leitura (`Read`, `Grep`, `Glob`) e a responsabilidade específica de revisar políticas IAM geradas antes de qualquer `apply` — um exemplo direto de restrição de ferramentas como controle de segurança.

---

### 1.5 Rules

**O que é.** Regras específicas organizadas em `.claude/rules/`, um arquivo por tópico, que podem ser carregadas sempre (como o CLAUDE.md) ou apenas quando arquivos que batem com um padrão de caminho (`paths:`) são tocados pelo Claude.

> *"Path-scoped rules allow you to load rule instructions only when they are relevant."*

**A diferença para CLAUDE.md**: CLAUDE.md é um bloco monolítico de contexto geral do projeto. Rules são modulares — cada arquivo cobre um tópico (`testing.md`, `security.md`, `api-design.md`) — e podem ser **condicionais**: uma regra com

```yaml
---
paths:
  - "infra/**/*.tf"
---
```

só entra no contexto quando o Claude está de fato trabalhando com arquivos que batem nesse padrão. Isso evita que uma regra sobre convenções de Terraform ocupe espaço de contexto quando o Claude está mexendo no frontend, por exemplo.

**A diferença para Skills**: rules carregam automaticamente (sempre, ou condicionalmente por caminho) — o Claude não *decide* usá-las, elas simplesmente entram no contexto quando o gatilho acontece. Skills são *invocadas* — sob demanda, seja pelo Claude decidindo que são relevantes, seja pelo usuário chamando explicitamente.

Regras também podem ser compartilhadas entre projetos via **symlink** (`.claude/rules/` aceita links simbólicos) e podem existir em nível de usuário (`~/.claude/rules/`), aplicando-se a todos os projetos na máquina.

No workshop, a regra `infra-security.md`, com `paths: ["**/*.tf"]`, vai carregar automaticamente sempre que o Claude tocar em qualquer arquivo Terraform, reforçando convenções como "toda IAM policy deve declarar recursos explicitamente, nunca usar `Resource: "*"`".

---

### 1.6 Agents × Skills × Rules — critério de decisão

| Conceito | Quando carrega | Quem decide usar | Use para |
|---|---|---|---|
| **Agents (subagents)** | Sob demanda, em contexto **isolado** | O Claude (automático) ou o usuário (explícito) | Tarefas paralelas ou especializadas que não devem poluir a conversa principal |
| **Skills** | Sob demanda — só nome/descrição no início, corpo ao ser invocada | O Claude (automático) ou o usuário (`/nome`) | Procedimentos repetíveis, checklists, workflows multi-etapa |
| **Rules** | Sempre (sem `paths`) ou condicionalmente (com `paths`) | Automático — não é "invocado", é carregado | Convenções e restrições específicas de um domínio ou tipo de arquivo |

Uma forma simples de decidir na prática: **é um fato que o Claude precisa saber sempre que mexe num certo tipo de arquivo?** → Rule. **É um procedimento que se repete, mas só quando pedido?** → Skill. **É uma tarefa que merece rodar isolada, possivelmente em paralelo com outras?** → Agent.

---

### 1.7 O harness: como as peças viram um agente

**O que é.** Como visto na seção 1.1, o harness é a camada que a própria documentação chama de "agentic harness": o programa Claude Code em si, que fornece as ferramentas, a gestão de contexto e o ambiente de execução que transformam um modelo de linguagem bruto em um agente capaz.

CLAUDE.md, Skills, Agents, Rules, Hooks, MCP e Memória não competem entre si — cada um resolve uma pergunta diferente (veja a tabela completa na seção 3.3), e juntos são exatamente o que compõe o harness na prática. Sem harness, o modelo esquece tudo entre sessões e não tem limites; é a diferença entre um script solto e a esteira de CI/CD ao redor dele — sozinho, o script não sabe onde rodar, quem aprova ou o que é permitido, e é o pipeline com suas etapas e guardrails que torna o processo confiável.

Vale reforçar: o harness não é um mecanismo a mais na lista — ele é a soma organizada de todos os outros. É por isso que esta seção fica no meio do Módulo 1, depois de Rules: a essa altura, já vimos peças suficientes para que "harness" pare de ser uma palavra abstrata e vire a estrutura visível ao redor delas.

---

### 1.8 Terraform, em uma imagem

**O que é.** Ferramenta de *Infrastructure as Code* da HashiCorp: você descreve os recursos de infraestrutura em arquivos declarativos (HCL) e o Terraform calcula e executa o que for preciso — criar, atualizar ou destruir — para que o estado real bata com o que foi declarado.

**Alternativas** que valem citar se a audiência perguntar: AWS CloudFormation e AWS CDK (proprietários da AWS), Pulumi (declarativo, mas em linguagens de programação de propósito geral em vez de HCL) e OpenTofu (fork open source do Terraform, mantido pela Linux Foundation depois da mudança de licença da HashiCorp).

**Os três comandos que levam até o deploy**, na ordem que usamos ao vivo:

```
$ terraform init      # baixa os providers e prepara o backend
$ terraform plan      # mostra o que vai mudar, antes de mudar
$ terraform apply     # ✓ infraestrutura criada na AWS
```

`plan` é o passo que mais importa para segurança: é o ponto onde um humano (ou um subagent revisor, como o `terraform-reviewer` da seção 1.4) pode ver exatamente o que vai mudar antes de qualquer coisa acontecer de verdade — é também o "ground truth" que o loop agêntico usa para verificar o próprio trabalho (seção 1.1).

---

### 1.9 AWS: onde a infraestrutura vai morar

**O que é.** O maior provedor de nuvem pública: um catálogo de serviços gerenciados que substitui o data center físico — você aluga exatamente o que precisa (computação, armazenamento, rede, IA), pelo tempo que precisa, cobrado pelo uso.

**Ponto técnico importante para a live**: nem o Terraform nem o Claude Code têm acesso próprio à AWS. Os dois autenticam cada chamada usando a mesma credencial IAM configurada na sua máquina local — normalmente em `~/.aws/credentials`:

```ini
[default]
aws_access_key_id = AKIA...
aws_secret_access_key = ••••••••••••
region = us-east-1
```

Terraform e Claude Code leem essa mesma credencial via AWS CLI/SDK, sem precisar de nenhuma configuração duplicada. Isso significa que qualquer limite de permissão colocado nessa credencial IAM (seção 1.4 e Módulo 2) vale para os dois igualmente — é uma camada de segurança que nem depende do Claude Code para funcionar.

Serviços da AWS usados ou citados no workshop: **S3** (armazenamento do site), **CloudFront** (CDN/HTTPS), **IAM** (permissões), **EC2** (computação, se necessário) e **Bedrock** (acesso a modelos, caso a organização prefira rodar Claude via AWS em vez da API direta da Anthropic).

---

### 1.10 Memória (CLAUDE.md vs. Auto Memory)

Cada sessão do Claude Code começa com uma janela de contexto vazia. Dois mecanismos carregam conhecimento entre sessões — e é importante não confundi-los:

| | CLAUDE.md | Auto Memory |
|---|---|---|
| **Quem escreve** | Você | O próprio Claude |
| **Conteúdo** | Instruções e regras | Aprendizados e padrões observados |
| **Escopo** | Projeto, usuário ou organização | Por repositório, compartilhado entre worktrees |
| **Uso ideal** | Padrões de código, arquitetura, workflows | Comandos de build descobertos, insights de debug, preferências que o Claude percebeu |

**Como a Auto Memory funciona por baixo do capô**: cada projeto tem seu próprio diretório de memória em `~/.claude/projects/<projeto>/memory/`, contendo um `MEMORY.md` (o índice, carregado em toda sessão) e arquivos de tópico (`debugging.md`, `api-conventions.md` etc., lidos sob demanda). Só os primeiros **200 linhas ou 25KB** do `MEMORY.md` — o que vier primeiro — são carregados automaticamente; por isso o próprio Claude Code mantém esse índice enxuto, movendo detalhe para arquivos de tópico.

**Um ponto importante de segurança conceitual, repetido de propósito**: tanto CLAUDE.md quanto Auto Memory são **contexto**, não configuração aplicada. O Claude trata como orientação, não como regra travada — para bloquear uma ação independentemente do que o Claude decida, o mecanismo correto é um Hook (Módulo 2).

No workshop, depois de algumas iterações, a Auto Memory vai reter, por exemplo, que o bucket de teste sempre precisa da tag `Environment=workshop` — um padrão que o Claude aprendeu ao ser corrigido, sem que ninguém precisasse editar o CLAUDE.md manualmente.

---

### 1.11 MCP — Model Context Protocol

**O que é.** Segundo o anúncio original da Anthropic, o MCP é um **padrão aberto para conectar assistentes de IA aos sistemas onde os dados residem** — repositórios de conteúdo, ferramentas de negócio, ambientes de desenvolvimento — através de conexões seguras e bidirecionais.

**O problema que ele resolve**: antes do MCP, cada combinação de assistente de IA × ferramenta externa exigia uma integração sob medida — um problema clássico de "N×M": N assistentes, M ferramentas, N×M integrações para manter. O MCP propõe um protocolo único: qualquer cliente compatível com MCP (Claude Code, por exemplo) pode falar com qualquer servidor MCP (uma ferramenta de terceiros ou construída internamente), sem integração customizada para cada par.

**No contexto do Claude Code**, um MCP server é o que permite o agente ler documentos de design no Google Drive, atualizar tickets no Jira, puxar dados do Slack, ou — o que nos interessa no workshop — consultar a API da AWS, o Terraform Registry, ou abrir um Pull Request no GitHub, tudo através do mesmo protocolo, sem que o Claude Code precise de código específico para cada uma dessas integrações.

**Configuração na prática**, o arquivo `.mcp.json` na raiz do projeto:

```json
{
  "mcpServers": {
    "aws": {
      "command": "uvx",
      "args": ["aws-mcp-server"],
      "env": { "AWS_PROFILE": "default" }
    }
  }
}
```

No workshop, vamos conectar um MCP server que dá ao Claude Code acesso a informações da conta AWS (por exemplo, verificar se um bucket já existe, consultar limites de serviço) e ao GitHub (para abrir o PR com as mudanças de infraestrutura antes do apply).

---

<a name="módulo-2"></a>
## Módulo 2 — Segurança

### 2.1 Por que instrução não é controle

Este é o ponto conceitual mais importante da live, e vale repetir com todas as letras: **CLAUDE.md, Rules e Auto Memory são contexto — o Claude tenta segui-los, mas não há garantia de cumprimento estrito.** Eles moldam o comportamento, não o aplicam.

> *"Settings rules are enforced by the client regardless of what Claude decides to do. CLAUDE.md instructions shape Claude's behavior but are not a hard enforcement layer."*

Para qualquer restrição que **precisa** valer sempre — "nunca rode `terraform destroy` sem confirmação humana", "nunca crie um usuário IAM com permissões administrativas" — a resposta correta não é "escrever isso mais enfaticamente no CLAUDE.md". É usar um mecanismo que é executado pelo *harness* (o programa Claude Code em si), não pelo modelo. Esses mecanismos são **Hooks** e **Permissions**.

A régua mental para a audiência: se a instrução pode, no limite, ser mal interpretada ou esquecida pelo modelo em um contexto muito longo, ela é *soft*. Se ela é aplicada por código determinístico antes mesmo de o modelo decidir algo, ela é *hard*.

---

### 2.2 Hooks

**O que é.** Comandos shell, endpoints HTTP ou prompts de LLM definidos pelo usuário, que executam automaticamente em pontos específicos do ciclo de vida do Claude Code — amarrados a um **evento**, não a uma instrução que o modelo escolhe seguir.

> *"Hooks are user-defined shell commands, HTTP endpoints, or LLM prompts that execute automatically at specific points in Claude Code's lifecycle."*

**Cadência dos eventos** — hooks disparam em três granularidades diferentes:

- **Uma vez por sessão**: `SessionStart`, `SessionEnd`.
- **Uma vez por turno**: `UserPromptSubmit` (antes do Claude processar o prompt), `Stop` (quando termina de responder).
- **A cada chamada de ferramenta** (o *agentic loop* propriamente dito, seção 1.1): `PreToolUse` (antes de executar — pode bloquear), `PermissionRequest`, `PostToolUse` (depois do sucesso), `PostToolUseFailure`.

Existem ainda eventos para subagents (`SubagentStart`, `SubagentStop`), compactação de contexto (`PreCompact`, `PostCompact`) e mudanças de configuração (`ConfigChange`, `InstructionsLoaded`).

**O mecanismo de bloqueio, na prática**: um hook `PreToolUse` recebe os dados da chamada de ferramenta prestes a acontecer (via stdin, em JSON) e pode retornar uma decisão estruturada:

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "deny",
    "permissionDecisionReason": "Comando destrutivo bloqueado pelo hook"
  }
}
```

Esse retorno acontece **antes** de qualquer execução — é o próprio harness que intercepta, não uma instrução que o modelo escolhe seguir. É essa distinção que torna o hook uma camada de aplicação real, e não apenas mais um texto de contexto — a analogia mais próxima é um hook de pre-commit ou um gate de pipeline: dispara sozinho toda vez que o evento acontece, sem depender de alguém lembrar de rodar aquilo manualmente.

No workshop, o hook `PreToolUse` mais importante barra qualquer chamada `Bash` que contenha `terraform destroy` ou `aws iam create-user`, retornando `deny` imediatamente — o agente literalmente não consegue executar o comando, independente do que "decidiu" fazer.

---

### 2.3 Permissions

**O que é.** A seção `permissions` do `settings.json` controla quais ferramentas e ações o Claude pode executar, através de três tipos de regra: `allow`, `deny` e `ask`.

```json
{
  "permissions": {
    "allow": ["Bash(terraform plan)", "Read(./infra/**)"],
    "deny": ["Bash(terraform destroy *)", "Bash(aws iam create-user *)", "Read(./.env)"]
  }
}
```

**Ordem de prioridade**: `deny` sempre vence, depois `allow`, depois `ask` como fallback quando nada mais se aplica.

**Camadas de configuração** — do maior para o menor alcance, e observe que elas se **mesclam** (diferente de outras configurações, que se sobrescrevem):

1. **Managed settings** — prioridade máxima, não pode ser sobrescrita pelo time nem pelo indivíduo. Ideal para políticas de compliance corporativas.
2. **Project settings** (`.claude/settings.json`) — compartilhado com o time via git.
3. **Local settings** (`.claude/settings.local.json`) — pessoal, no `.gitignore`.
4. **User settings** (`~/.claude/settings.json`) — aplicado a todos os projetos daquele usuário.

**Um detalhe de segurança relevante para times**: regras de `allow` vindas de `.claude/settings.json` de um projeto exigem verificação de confiança do workspace antes de valerem — isso evita que um repositório clonado de fonte não confiável já venha com permissões liberadas automaticamente. Regras em `settings.local.json`, por serem pessoais, não passam por essa verificação.

No workshop, o `settings.json` do projeto vai conter `deny` explícito para os comandos destrutivos e `allow` para o ciclo normal (`terraform plan`, `terraform apply` com aprovação, leitura de arquivos do projeto).

---

### 2.4 Guardrails para infraestrutura — o que efetivamente configurar

Combinando Hooks e Permissions, a defesa em profundidade recomendada para um workflow agêntico de infraestrutura tem pelo menos três camadas:

1. **Permissions (`deny`)** — bloqueia comandos conhecidos como perigosos antes mesmo de o Claude tentar: `terraform destroy`, `aws iam create-user`, `aws iam attach-user-policy`, exclusão de buckets S3 fora de um padrão de nome específico.
2. **Hooks (`PreToolUse`)** — para regras mais dinâmicas que um simples padrão de string não cobre (ex: "bloquear qualquer comando Terraform que toque um *state* de produção", inspecionando o conteúdo real do comando).
3. **Subagent com ferramentas restritas** — como o `terraform-reviewer` do Módulo 1.4, que só tem acesso de leitura e cuja função é justamente revisar antes de qualquer aplicação real.

A combinação das três camadas é o que permite dar ao agente autonomia real (ele pode planejar, escrever, iterar livremente) sem abrir mão de controle sobre as ações que realmente importam (aplicar mudanças destrutivas ou de alto privilégio).

---

<a name="módulo-3"></a>
## Módulo 3 — Integrações

### 3.1 MCP na prática

No workshop, conectamos pelo menos um MCP server que dá ao Claude Code visibilidade sobre o estado real da conta AWS — por exemplo, para verificar se um nome de bucket já está em uso, consultar quotas de serviço, ou puxar a lista de distribuições CloudFront existentes antes de criar uma nova. Isso evita que o agente "alucine" sobre o estado da infraestrutura: em vez de assumir, ele consulta.

Um segundo MCP server relevante é o do GitHub, usado para abrir automaticamente um Pull Request com as mudanças de Terraform propostas — mantendo um humano no laço de aprovação antes do `apply` chegar em produção, mesmo em um workflow altamente automatizado.

### 3.2 Memória entre sessões, na prática

Ao longo de várias sessões de trabalho no mesmo projeto, a Auto Memory (Módulo 1.10) acumula decisões e padrões — convenção de tags, região AWS padrão do time, formato de nome de bucket — sem que ninguém precise manter isso manualmente atualizado em um documento. Isso é particularmente valioso em um contexto de ensino/workshop: erros cometidos e corrigidos ao vivo "grudam" automaticamente para a próxima sessão.

### 3.3 Como tudo se combina

A stack agêntica completa que construímos ao longo da live tem oito peças, cada uma respondendo a uma pergunta diferente:

| Peça | Pergunta que responde |
|---|---|
| CLAUDE.md | O que este projeto é e como funciona? |
| Skills | Como executo este procedimento específico? |
| Agents | Quem faz esta tarefa isolada, e com que ferramentas? |
| Rules | Que convenção vale quando toco neste tipo de arquivo? |
| Hooks | O que é bloqueado ou validado, sempre, sem exceção? |
| Permissions | O que este agente pode, não pode, ou precisa perguntar? |
| MCP | A que sistemas externos este agente tem acesso? |
| Memória | O que este agente já aprendeu sobre este projeto? |

Nenhuma peça sozinha entrega um workflow agêntico seguro e produtivo — é a combinação das oito, orquestrada pelo harness (seção 1.7), que faz isso.

---

<a name="módulo-4"></a>
## Módulo 4 — Demonstração ao vivo

### 4.1 Estrutura do projeto e o CLAUDE.md real

A estrutura do repositório usado na live:

```
devops-agentic-workflow-tf-aws-webslides/
├── terraform/
│   └── main.tf
├── .github/workflows/
│   └── deploy.yml
├── .claude/
│   └── skills/, agents/
├── .mcp.json
└── CLAUDE.md          ← lido sempre
```

E o trecho real do CLAUDE.md do projeto, seção Skills:

```markdown
# CLAUDE.md

## Skills (.claude/skills/)
Não escreva Terraform ou CI/CD manualmente.
Use a skill certa. Skills de ação têm
disable-model-invocation: true

/scaffold-terraform → gera todo o Terraform
/tf-plan → terraform plan + riscos
/tf-apply → terraform apply + verifica
/deploy → sync S3 + invalida CloudFront
```

Esse trecho é o exemplo mais direto do que a seção 1.2 descreve em teoria: uma instrução persistente, curta e específica, que existe justamente para impedir que o Claude escreva HCL ou YAML à mão — a tradução vira sempre responsabilidade de uma skill testada, nunca de uma decisão ad hoc do modelo no meio de uma conversa.

---

### 4.2 Anatomia completa de um SKILL.md

Todo campo do frontmatter de uma skill é opcional — só `description` é recomendado, e sem ele o Claude não sabe quando usar a skill sozinho:

| Campo | Para que serve |
|---|---|
| `name` | Nome exibido (padrão: nome da pasta) |
| `description` | O que a skill faz e quando usar — **recomendado** |
| `when_to_use` | Contexto extra para o Claude decidir acionar sozinho |
| `argument-hint` | Dica de autocomplete ao digitar `/nome` |
| `arguments` | Nomeia argumentos posicionais (`$a`, `$b`...) |
| `disable-model-invocation` | `true` = só pode ser chamada manualmente, via `/nome` |
| `user-invocable` | `false` = só o Claude pode acionar, nunca o usuário |
| `allowed-tools` | Libera ferramentas específicas só durante esse turno |
| `disallowed-tools` | Remove ferramentas específicas durante esse turno |
| `model` | Troca o modelo usado só nesse turno |
| `effort` | Troca o nível de raciocínio (reasoning effort) |
| `context` | `fork` = roda em um subagente isolado |
| `agent` | Qual subagente usar, quando `context: fork` |
| `background` | `true` = roda o fork em segundo plano |
| `hooks` | Hooks registrados especificamente ao invocar essa skill |
| `paths` | Auto-ativa a skill quando arquivos desses caminhos são tocados |
| `shell` | `bash` (padrão) ou `powershell` |
| `metadata` | Dados livres, de uso arbitrário por outras ferramentas |
| `license` | Licença da skill |
| `compatibility` | Requisitos de ambiente (versão do Claude Code, SO etc.) |

Essa tabela é útil para a live porque mostra que uma skill não é só "um prompt salvo" — os campos `disable-model-invocation`, `allowed-tools`/`disallowed-tools` e `paths` são, na prática, uma segunda camada de controle sobre a mesma automação, no mesmo espírito das Permissions do Módulo 2.3.

---

### 4.3 As skills do projeto

O diretório `.claude/skills/` do projeto tem dez skills registradas, mas só quatro são chamadas ao vivo:

| Skill | Faz | Ativação |
|---|---|---|
| `scaffold-terraform` | Gera todo o Terraform | Manual |
| `tf-plan` | Plan + análise de risco | Manual |
| `tf-apply` | Aplica infra + verifica | Manual |
| `deploy` | Sync S3 + invalida CloudFront | Manual |

As demais existem no repositório mas ficam fora da demonstração ao vivo por tempo: `scaffold-cicd`, `infra-status`, `infra-audit`, `setup-gh-actions`, `tf-destroy` (todas manuais) e `project-scope` (essa, ao contrário das outras, carrega automaticamente — não tem `disable-model-invocation: true`).

Repare que as quatro skills demonstradas têm ativação **manual** — nenhuma delas pode ser acionada pelo Claude sozinho. Isso não é acidente: qualquer skill que gera ou aplica infraestrutura real é, por definição, uma ação de alto impacto, e o projeto trata isso como uma decisão explícita do humano, nunca uma automação silenciosa.

---

### 4.4 Exemplo real: a skill deploy

O arquivo completo, do jeito que vive no repositório (`.claude/skills/deploy/SKILL.md`):

```markdown
---
name: deploy
description: Sync site files to S3 and invalidate CloudFront cache. Use after terraform apply to push site content live.
allowed-tools: Bash, Read
disable-model-invocation: true
---

Deploy site files to S3 and invalidate CloudFront cache.

Steps:
- [ ] Get terraform outputs: terraform output -json
- [ ] Sync site files: aws s3 sync . s3://<bucket> --exclude "terraform/*" ... --delete
- [ ] Invalidate cache: aws cloudfront create-invalidation --distribution-id <dist-id> --paths "/*"
- [ ] Report the CloudFront URL and invalidation status

If any step fails, stop and report the error. Do not continue.
```

Note como esse arquivo real usa, na prática, quatro conceitos que já vimos separadamente: `description` clara (para o Claude entender o propósito, mesmo não podendo acioná-la sozinho), `allowed-tools` restrito a só `Bash` e `Read` (nenhuma edição de arquivo é necessária para um deploy), `disable-model-invocation: true` (Módulo 4.2 — ação de alto impacto, só manual) e um checklist determinístico com uma instrução explícita de **parar em caso de erro**, em vez de tentar "resolver sozinho" e potencialmente deixar o deploy em um estado inconsistente.

---

### 4.5 O fluxo de trabalho, com e sem skills

A mesma tarefa de deploy, duas formas bem diferentes de chegar lá:

**Sem skills**: você reexplica o contexto e os passos a cada pedido; a ordem das etapas muda dependendo de como o prompt foi escrito; é fácil pular uma etapa de segurança sem perceber; o resultado varia conforme quem chamou e como chamou.

**Com skills**: o passo a passo foi gravado uma vez em `SKILL.md` e é reaproveitado sempre; a sequência é sempre a mesma (`/scaffold-terraform → /tf-plan → /tf-apply → /deploy`); `disable-model-invocation` trava a etapa arriscada para o humano decidir; o resultado é previsível, não importa quem chamou o comando.

Essa comparação é, em miniatura, a mesma lógica da seção 0.2: transformar uma sequência de decisões ad hoc em um processo repetível e com pontos de controle claros — só que aplicada a uma única tarefa (deploy), em vez de a um incidente inteiro.

---

### 4.6 Ordem de execução ao vivo

Sequência sugerida para a demonstração (ajuste o ritmo conforme o tempo disponível):

1. **`/init`** — gera o `CLAUDE.md` inicial, analisando o projeto (estrutura, comandos de build, convenções detectáveis automaticamente).
2. **Mostrar a skill de deploy já pronta** (seção 4.4) e explicar o frontmatter (seção 4.2).
3. **Criar o subagent de revisão** — pedir a criação de `terraform-reviewer`, com acesso restrito a leitura, focado em segurança de IAM.
4. **Configurar hooks e permissions** — pedir explicitamente o bloqueio de `terraform destroy` e criação de usuários IAM.
5. **Testar as quatro skills, uma a uma, nesta ordem**: `/scaffold-terraform` (gera todo o Terraform) → `/tf-plan` (revisa antes de aplicar) → `/tf-apply` (provisiona na AWS) → `/deploy` (site no ar). É a mesma esteira de qualquer pipeline de release: gerar, revisar, aplicar, publicar — pular uma etapa é o jeito mais comum de derrubar produção.
6. **O prompt final, ao vivo**: *"crie e publique um site estático no S3 com CloudFront, seguindo nossas regras de segurança"* — este é o momento em que todas as peças anteriores atuam juntas: CLAUDE.md dá o contexto, as skills estruturam o processo, o subagent revisa o IAM, os hooks/permissions bloqueiam qualquer desvio perigoso, e o MCP consulta o estado real da AWS antes de criar recursos.
7. **Mostrar o resultado** — a URL do CloudFront funcionando, e reforçar: nenhuma linha de Terraform ou YAML foi digitada manualmente durante a live.

---

<a name="glossário"></a>
## Glossário rápido

- **Agentic loop (loop agêntico)**: o ciclo de três fases que o Claude Code executa em toda tarefa — reunir contexto, agir, verificar — repetindo até a tarefa estar completa, com o humano podendo interromper a qualquer ponto.
- **Harness**: "the agentic harness around Claude" — a camada que fornece as ferramentas, a gestão de contexto e o ambiente de execução que transformam o modelo bruto em um agente capaz. No Claude Code, é o programa em si, por oposição ao modelo, que decide o que fazer dentro dos limites que o harness permite.
- **Ground truth**: informação real do ambiente (resultado de um teste, saída de um `terraform plan`, erro de compilação) que o agente usa para avaliar se o que fez de fato funcionou, em vez de assumir que funcionou.
- **Bounded autonomy (autonomia limitada)**: o agente opera de forma independente dentro do loop, mas para em pontos de checagem definidos (aprovação de PR, limite de iterações, bloqueio de um hook) para feedback ou decisão humana.
- **Workflow vs. Agent**: workflow é orquestração por caminhos de código pré-definidos (determinístico); agente é um sistema em que o próprio modelo direciona dinamicamente o processo e o uso de ferramentas (adaptativo).
- **Least privilege**: princípio de segurança que diz que qualquer identidade (usuário, papel, serviço) deve ter apenas as permissões estritamente necessárias para sua função.
- **Context window**: a "memória de trabalho" do modelo em uma sessão — tudo que está carregado nela (CLAUDE.md, rules, histórico da conversa) consome espaço finito.
- **Human-in-the-loop**: ponto do processo em que uma decisão exige aprovação humana explícita antes de prosseguir — no nosso workshop, tipicamente antes de um `terraform apply`.

---

<a name="fontes"></a>
## Fontes

- [How Claude Code works — Claude Code Docs](https://code.claude.com/docs/en/how-claude-code-works)
- [Overview — Claude Code Docs](https://code.claude.com/docs/en/overview)
- [How Claude remembers your project — Claude Code Docs](https://code.claude.com/docs/en/memory)
- [Extend Claude with skills — Claude Code Docs](https://code.claude.com/docs/en/skills)
- [Hooks reference — Claude Code Docs](https://code.claude.com/docs/en/hooks)
- [Claude Code settings — Claude Code Docs](https://code.claude.com/docs/en/settings)
- [Subagents in the SDK — Claude Code Docs](https://code.claude.com/docs/en/agent-sdk/subagents)
- [Steering Claude Code: when to use CLAUDE.md, skills, hooks, and subagents — Claude by Anthropic](https://claude.com/blog/steering-claude-code-skills-hooks-rules-subagents-and-more)
- [Building Effective Agents — Anthropic Engineering](https://www.anthropic.com/engineering/building-effective-agents)
- [Effective harnesses for long-running agents — Anthropic Engineering](https://www.anthropic.com/engineering/effective-harnesses-for-long-running-agents)
- [Introducing the Model Context Protocol — Anthropic](https://www.anthropic.com/news/model-context-protocol)
- [Give Claude context: CLAUDE.md and better prompts — Claude Help Center](https://support.claude.com/en/articles/14553240-give-claude-context-claude-md-and-better-prompts)

---

*Documento de apoio preparado para a live "DevOps com Agentes de IA — Claude Code, Terraform e AWS", Pós-graduação em Engenharia de IA Aplicada, UNIPDS.*
