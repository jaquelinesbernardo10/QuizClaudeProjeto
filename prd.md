# PRD — Quiz Claude Code (Verdadeiro ou Falso)

> Documento de requisitos de produto para ser consumido pelo Claude Code na construção do projeto.
> Versão 1.0 · 2026-10-11 · Status: aprovado para MVP

---

## 1. Visão geral

### 1.1 Problema
Times internos estão adotando o Claude Code, mas o nível de entendimento é desigual: o pessoal de negócio não sabe bem o que a ferramenta é, onde roda e quando usá-la, e os técnicos conhecem só parte dos recursos. Falta um jeito rápido, leve e mensurável de nivelar esse conhecimento.

### 1.2 Solução
Um **quiz web de Verdadeiro ou Falso** sobre o Claude Code, com dificuldade **progressiva** (fácil → médio → difícil), **explicação imediata** após cada resposta, **ranking** para gerar engajamento entre os times e uma **área de administração** para manter o banco de perguntas sem mexer no código.

### 1.3 Objetivos de negócio
| # | Objetivo | Métrica de sucesso (primeiros 60 dias) |
|---|----------|----------------------------------------|
| O1 | Nivelar o conhecimento básico sobre Claude Code | Pontuação média das rodadas ≥ 60% |
| O2 | Gerar engajamento com a adoção da ferramenta | ≥ 70% das pessoas dos times-alvo completam ao menos 1 rodada |
| O3 | Incentivar o aprendizado contínuo | ≥ 30% dos participantes jogam mais de 1 rodada |
| O4 | Manter o conteúdo atualizado sem precisar de dev | Perguntas criadas ou editadas pelo admin sem deploy |

### 1.4 Público-alvo
- **Primário:** colaboradores de times internos (negócio e tecnologia) da empresa que está adotando o Claude Code.
- **Secundário:** quem facilita a adoção (líderes, champions, time de enablement), que mantém as perguntas pelo admin.

### 1.5 Fora de escopo (MVP)
- Login, SSO ou perfis de usuário com histórico
- Timer por pergunta
- Múltipla escolha ou outros formatos de pergunta (só V/F)
- Internacionalização (só PT-BR)
- Temas técnicos avançados (MCP, hooks, SDK, CI). Ficam para a v2 (ver §10)
- Autenticação na área admin (decisão consciente; ver §9)

---

## 2. Personas

| Persona | Descrição | O que espera |
|---------|-----------|--------------|
| **Ana, analista de negócio** | Ouviu falar do Claude Code e não programa | Entender o que é a ferramenta e para que serve, sem jargão |
| **Bruno, dev** | Já usa o Claude Code no dia a dia | Se desafiar nas perguntas difíceis e aparecer no topo do ranking |
| **Carla, champion de adoção** | Responsável por disseminar a ferramenta | Cadastrar e ajustar perguntas e acompanhar o engajamento pelo ranking |

---

## 3. Requisitos funcionais

### 3.1 Fluxo do jogador

```
[Home] → digita apelido → [Iniciar rodada]
   → 15 perguntas (5 fáceis → 5 médias → 5 difíceis)
      → para cada uma: responde V/F → feedback imediato (acertou/errou + explicação + link)
   → [Resultado] pontuação, acertos por nível, posição no ranking
   → [Jogar de novo] ou [Ver ranking]
```

#### RF-01: Home
- Título, uma frase explicando o quiz e um resumo das regras (15 perguntas, 3 níveis, pontuação).
- Campo **Apelido** (obrigatório, 2 a 20 caracteres, sem validação de unicidade). Fica salvo no `localStorage` para pré-preencher na próxima visita.
- Botão **Começar**, desabilitado enquanto o apelido for inválido.
- Link para **Ranking**.

#### RF-02: Montagem da rodada
- O servidor sorteia **5 perguntas ativas de cada nível** (fácil, médio, difícil) e as ordena por nível. Dentro de um mesmo nível a ordem é aleatória.
- Se algum nível tiver menos de 5 perguntas ativas, a rodada não começa e aparece uma mensagem amigável (“Banco de perguntas incompleto”).
- A rodada é registrada no banco (`Attempt`) com as perguntas sorteadas, para validar as respostas no servidor.

#### RF-03: Tela de pergunta
- Indicador de progresso (“Pergunta 7 de 15”) e barra de progresso.
- Selo do nível atual (Fácil 🟢 / Médio 🟡 / Difícil 🔴), com uma pequena animação ou transição ao mudar de nível (“Subindo de nível!”).
- Enunciado em destaque.
- Dois botões grandes: **Verdadeiro** e **Falso**. Atalhos de teclado: `V`/`F` ou `←`/`→`.
- Pontuação acumulada visível.

#### RF-04: Feedback imediato
- Depois de responder, os botões ficam travados e aparece:
  - Status: ✅ “Acertou!” ou ❌ “Errou”, além da resposta correta.
  - **Explicação** curta (1 a 3 frases).
  - **Link “Saiba mais”** para a documentação oficial, se cadastrado.
  - Pontos ganhos naquela pergunta.
- Botão **Próxima** (ou `Enter`).
- A correção é feita **no servidor**. O cliente nunca recebe a resposta correta antes de responder.

#### RF-05: Resultado
- Pontuação final, acertos totais (x/15) e acertos por nível.
- Mensagem conforme o desempenho:
  - 0–40%: “Começando a jornada”
  - 41–70%: “Bom caminho”
  - 71–90%: “Quase expert”
  - 91–100%: “Mestre do Claude Code”
- Posição no ranking geral.
- Lista recolhível com as 15 perguntas, a resposta dada, a correta e a explicação.
- Botões **Jogar de novo** e **Ver ranking**.

#### RF-06: Pontuação
| Nível | Pontos por acerto |
|-------|-------------------|
| Fácil | 10 |
| Médio | 20 |
| Difícil | 30 |

- Máximo por rodada: 5×10 + 5×20 + 5×30 = **300 pontos**.
- Erro não desconta pontos.
- Duração da rodada (do início à última resposta) é registrada e serve de **critério de desempate** (quem terminou mais rápido fica na frente).

#### RF-07: Ranking
- Página `/ranking` com o **Top 20**: posição, apelido, pontuação, acertos e data.
- Filtro de período: **Geral** | **Últimos 7 dias**.
- Só entram rodadas **concluídas** (15 respostas).
- Como os apelidos são livres e podem se repetir, **cada rodada é uma linha** do ranking. Não há agregação por pessoa.
- Ordenação: pontuação ↓, duração ↑, data de conclusão ↑.

### 3.2 Área administrativa

#### RF-08: Lista de perguntas (`/admin`)
- Tabela com enunciado (truncado), nível, resposta (V/F), status (ativa/inativa) e data de atualização.
- Filtros por nível e status, e busca por texto.
- Contador por nível mostrando quantas perguntas ativas existem, com alerta se algum nível tiver menos de 5.

#### RF-09: Criar e editar pergunta
- Campos:
  - **Enunciado** (obrigatório, 10 a 300 caracteres)
  - **Resposta correta** (obrigatório, Verdadeiro/Falso)
  - **Nível** (obrigatório, Fácil/Médio/Difícil)
  - **Explicação** (obrigatório, 10 a 500 caracteres)
  - **Link de referência** (opcional, URL válida)
  - **Tema** (opcional, texto livre, ex.: “O que é”, “Onde roda”, “Planos e custo”)
  - **Ativa** (checkbox, padrão: marcado)
- Validação no cliente e no servidor (mesmo schema Zod).

#### RF-10: Desativar ou excluir
- **Desativar** tira a pergunta do sorteio e preserva o histórico. É a ação recomendada.
- **Excluir** só é permitido se a pergunta nunca apareceu em nenhuma rodada; caso contrário, o sistema sugere desativar. Pede confirmação.

#### RF-11: Moderação do ranking
- Na área admin, uma aba **Ranking** lista as rodadas concluídas, com opção de **ocultar** uma rodada (por exemplo, por apelido ofensivo). Ocultar é reversível.

---

## 4. Requisitos não funcionais

| ID | Requisito |
|----|-----------|
| RNF-01 | **Responsivo:** funciona bem de 360px (celular) a desktop, sem scroll horizontal |
| RNF-02 | **Desempenho:** primeira carga < 2s em rede corporativa; resposta da API < 300ms |
| RNF-03 | **Acessibilidade:** contraste AA, navegação por teclado, foco visível, `aria-live` no feedback, sem depender só de cor para indicar certo/errado (usar ícone e texto) |
| RNF-04 | **Integridade:** resposta correta e pontuação são calculadas só no servidor; cada pergunta da rodada só pode ser respondida uma vez |
| RNF-05 | **Idioma:** toda a interface em PT-BR |
| RNF-06 | **Visual:** leve e gamificado: cores vivas, cantos arredondados, microanimações (respeitando `prefers-reduced-motion`), suporte a modo escuro |
| RNF-07 | **Simplicidade operacional:** roda com `npm install && npm run dev`, sem serviços externos |

---

## 5. Arquitetura técnica

### 5.1 Stack
| Camada | Tecnologia |
|--------|------------|
| Framework | **Next.js 15** (App Router) + **TypeScript** (strict) |
| UI | **Tailwind CSS** + componentes próprios (shadcn/ui opcional) |
| Animações | CSS/Tailwind (Framer Motion opcional) |
| ORM | **Prisma** |
| Banco | **SQLite** (arquivo local `prisma/dev.db`) |
| Validação | **Zod** (schemas compartilhados entre cliente e servidor) |
| Testes | **Vitest** (unidade) + **Playwright** (e2e do fluxo principal) |
| Lint/format | ESLint + Prettier |

### 5.2 Estrutura de pastas sugerida
```
/
├── prisma/
│   ├── schema.prisma
│   └── seed.ts              # 30 perguntas iniciais
├── src/
│   ├── app/
│   │   ├── page.tsx                     # Home
│   │   ├── quiz/[attemptId]/page.tsx    # Jogo
│   │   ├── resultado/[attemptId]/page.tsx
│   │   ├── ranking/page.tsx
│   │   ├── admin/
│   │   │   ├── page.tsx                 # Lista de perguntas
│   │   │   ├── perguntas/nova/page.tsx
│   │   │   ├── perguntas/[id]/page.tsx
│   │   │   └── ranking/page.tsx         # Moderação
│   │   └── api/                         # Route handlers (ver §5.4)
│   ├── components/
│   ├── lib/
│   │   ├── db.ts            # Prisma client singleton
│   │   ├── scoring.ts       # Regras de pontuação (puro, testável)
│   │   ├── draw.ts          # Sorteio das perguntas (puro, testável)
│   │   └── schemas.ts       # Schemas Zod
│   └── types/
├── tests/
│   ├── unit/
│   └── e2e/
└── README.md
```

### 5.3 Modelo de dados (Prisma)
```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

enum Level {
  EASY
  MEDIUM
  HARD
}

model Question {
  id            String           @id @default(cuid())
  statement     String
  correctAnswer Boolean
  level         Level
  explanation   String
  referenceUrl  String?
  topic         String?
  active        Boolean          @default(true)
  createdAt     DateTime         @default(now())
  updatedAt     DateTime         @updatedAt
  attemptItems  AttemptQuestion[]

  @@index([level, active])
}

model Attempt {
  id          String            @id @default(cuid())
  nickname    String
  score       Int               @default(0)
  correctCount Int              @default(0)
  startedAt   DateTime          @default(now())
  finishedAt  DateTime?
  durationMs  Int?
  hidden      Boolean           @default(false)
  items       AttemptQuestion[]

  @@index([finishedAt, score])
}

model AttemptQuestion {
  id          String    @id @default(cuid())
  attemptId   String
  questionId  String
  position    Int       // 1..15
  answer      Boolean?  // null = ainda não respondida
  isCorrect   Boolean?
  points      Int       @default(0)
  answeredAt  DateTime?
  attempt     Attempt   @relation(fields: [attemptId], references: [id], onDelete: Cascade)
  question    Question  @relation(fields: [questionId], references: [id])

  @@unique([attemptId, position])
  @@unique([attemptId, questionId])
}
```
> Observação: o SQLite com Prisma aceita `enum` nas versões recentes do Prisma. Se a versão usada não aceitar, troque por `String` e valide com Zod.

### 5.4 API (Route Handlers)

| Método | Rota | Descrição |
|--------|------|-----------|
| `POST` | `/api/attempts` | Body `{ nickname }`. Sorteia 15 perguntas e cria `Attempt` + `AttemptQuestion`. Retorna `{ attemptId }` |
| `GET` | `/api/attempts/:id/current` | Retorna a próxima pergunta não respondida: `{ position, total, level, statement, score }`. **Sem `correctAnswer`** |
| `POST` | `/api/attempts/:id/answer` | Body `{ position, answer }`. Valida se é a pergunta atual e se ainda não foi respondida, corrige e pontua. Retorna `{ isCorrect, correctAnswer, explanation, referenceUrl, points, score, finished }`. Na 15ª resposta, preenche `finishedAt` e `durationMs` |
| `GET` | `/api/attempts/:id/result` | Resumo final (só se a rodada estiver concluída) com a revisão das 15 perguntas e a posição no ranking |
| `GET` | `/api/ranking?period=all\|7d` | Top 20 das rodadas concluídas e não ocultas |
| `GET` | `/api/admin/questions` | Lista com filtros `level`, `active`, `q` |
| `POST` | `/api/admin/questions` | Cria pergunta |
| `PUT` | `/api/admin/questions/:id` | Edita pergunta |
| `DELETE` | `/api/admin/questions/:id` | Exclui (retorna `409` se a pergunta já foi usada em alguma rodada) |
| `PATCH` | `/api/admin/attempts/:id` | Body `{ hidden }`. Oculta ou reexibe uma rodada no ranking |

Server Actions podem substituir as rotas admin, desde que as regras sejam as mesmas.

**Erros:** respostas JSON no formato `{ error: { code, message } }`, com mensagens em PT-BR. Códigos: `400` (validação), `404`, `409` (pergunta já respondida, exclusão bloqueada, rodada não concluída), `422` (banco de perguntas insuficiente).

### 5.5 Regras de negócio no servidor
- `draw.ts`: `drawQuestions(pool, perLevel = 5)` recebe as perguntas ativas, embaralha com Fisher–Yates dentro de cada nível e devolve 15 em ordem EASY → MEDIUM → HARD. Lança erro se faltar pergunta em algum nível.
- `scoring.ts`: `pointsFor(level, isCorrect)` retorna 10/20/30 ou 0.
- A resposta só é aceita para a **menor posição ainda não respondida** (impede pular perguntas e responder fora de ordem).
- Apelido: aplicar `trim`, colapsar espaços e limitar a 20 caracteres. Sanitização de HTML fica por conta do React (sem `dangerouslySetInnerHTML`).

### 5.6 Configuração
`.env.example`:
```
DATABASE_URL="file:./dev.db"
```
Scripts no `package.json`:
- `dev`, `build`, `start`, `lint`, `test`, `test:e2e`
- `db:migrate` → `prisma migrate dev`
- `db:seed` → `prisma db seed`
- `db:reset` → `prisma migrate reset`

---

## 6. Conteúdo: banco de perguntas inicial (seed)

### 6.1 Diretrizes
- **30 perguntas:** 10 fáceis, 10 médias e 10 difíceis.
- **Tema do MVP:** *Conceitos e negócio*: o que é o Claude Code, o que ele faz, onde roda (terminal/CLI, app desktop, web, extensões de IDE), quem desenvolve, para que tipo de tarefa serve, como ele interage com o projeto (lê arquivos, executa comandos, pede permissão), planos e custo, privacidade e boas práticas de adoção.
- A progressão de dificuldade vem da **profundidade do conceito**, não de detalhes técnicos de configuração:
  - **Fácil:** o que é, quem faz e para que serve.
  - **Médio:** onde roda, como interage com o projeto, papel humano, permissões.
  - **Difícil:** nuances de uso, limites, custo, governança e conceitos como `CLAUDE.md`, MCP e modos de permissão em nível conceitual.
- Equilíbrio de respostas: entre 40% e 60% das perguntas devem ser **Falso**.
- Afirmações falsas precisam ser **plausíveis** (nada de pegadinha absurda) e a explicação deve corrigir o equívoco.
- Toda pergunta tem explicação e, sempre que possível, `referenceUrl` apontando para a documentação oficial (`https://docs.claude.com/` ou `https://docs.anthropic.com/en/docs/claude-code/`).
- ⚠️ **Antes de finalizar o seed, confira cada afirmação na documentação oficial atual.** O produto evolui rápido, então evite números exatos de preço, limites ou nomes de planos que possam mudar.

### 6.2 Exemplos (formato de referência)
```ts
{
  statement: "O Claude Code é uma ferramenta de programação com IA desenvolvida pela Anthropic.",
  correctAnswer: true,
  level: "EASY",
  topic: "O que é",
  explanation: "O Claude Code é a ferramenta de programação agêntica da Anthropic, que trabalha diretamente no seu projeto.",
  referenceUrl: "https://docs.anthropic.com/en/docs/claude-code/overview"
},
{
  statement: "O Claude Code só pode ser usado pelo navegador.",
  correctAnswer: false,
  level: "MEDIUM",
  topic: "Onde roda",
  explanation: "Ele roda no terminal (CLI) e também está disponível no app desktop, na web e em extensões de IDE como VS Code e JetBrains."
},
{
  statement: "Por padrão, o Claude Code pode alterar arquivos e executar comandos sem nunca pedir permissão.",
  correctAnswer: false,
  level: "MEDIUM",
  topic: "Permissões",
  explanation: "Por padrão ele pede aprovação para ações como editar arquivos e rodar comandos. O usuário escolhe o modo de permissão mais adequado."
},
{
  statement: "O arquivo CLAUDE.md serve para registrar instruções e contexto do projeto que o Claude Code carrega automaticamente.",
  correctAnswer: true,
  level: "HARD",
  topic: "Contexto",
  explanation: "O CLAUDE.md guarda convenções, comandos e contexto do projeto, e é lido automaticamente no início das sessões."
},
{
  statement: "O Claude Code não consegue se conectar a ferramentas ou fontes de dados externas.",
  correctAnswer: false,
  level: "HARD",
  topic: "Integrações",
  explanation: "Pelo Model Context Protocol (MCP), o Claude Code se conecta a ferramentas e fontes de dados externas."
}
```

---

## 7. UX e design

- **Paleta:** cor primária laranja/terracota (remetendo ao Claude), verde para acerto, vermelho para erro, e amarelo/âmbar para o nível médio. Defina tudo como variáveis CSS (tokens) com tema claro e escuro.
- **Tipografia:** sans-serif legível (por exemplo, Inter), enunciado com pelo menos 20px no desktop e 18px no celular.
- **Gamificação leve:**
  - Barra de progresso animada
  - Transição “Subindo de nível!” ao passar de fácil para médio e de médio para difícil
  - Pequeno confete ou brilho no resultado ≥ 71% (desligado com `prefers-reduced-motion`)
  - Medalhas 🥇🥈🥉 no top 3 do ranking
- **Estados obrigatórios:** carregando (skeleton), vazio (ranking sem rodadas), erro (com botão de tentar de novo) e banco insuficiente.
- **Admin:** visual funcional e sóbrio (tabela e formulário), sem a gamificação.

---

## 8. Critérios de aceite (MVP)

- [ ] Um jogador informa o apelido, joga 15 perguntas na ordem 5 fáceis → 5 médias → 5 difíceis e vê o resultado.
- [ ] Depois de cada resposta, aparecem acerto/erro, resposta correta, explicação e link (se houver).
- [ ] A resposta correta nunca aparece na rede antes de o jogador responder (conferível nas DevTools).
- [ ] Responder a mesma pergunta duas vezes ou fora de ordem retorna `409` e não altera a pontuação.
- [ ] A pontuação segue 10/20/30 e o máximo é 300.
- [ ] O ranking mostra o Top 20 ordenado por pontuação ↓ e duração ↑, com filtro Geral/7 dias, e não inclui rodadas incompletas nem ocultas.
- [ ] O admin cria, edita, desativa e exclui perguntas, e a exclusão de pergunta já usada é bloqueada.
- [ ] Uma pergunta desativada não aparece em rodadas novas.
- [ ] O admin oculta e reexibe rodadas do ranking.
- [ ] Com menos de 5 perguntas ativas em algum nível, aparece uma mensagem clara e a rodada não começa.
- [ ] O seed cria 30 perguntas válidas (10 por nível).
- [ ] O layout funciona em 360px sem scroll horizontal e o jogo pode ser jogado só com o teclado.
- [ ] Testes unitários de `draw.ts` e `scoring.ts` passando, e e2e do fluxo completo de uma rodada passando.
- [ ] O README explica como instalar, rodar a migração e o seed e subir o projeto.

---

## 9. Riscos e decisões conscientes

| Risco | Decisão no MVP | Mitigação futura |
|-------|----------------|------------------|
| **Admin sem autenticação:** qualquer pessoa com a URL pode alterar perguntas e o ranking | Aceito por ser uso interno. **Não expor publicamente na internet**; publicar só na rede interna/VPN | v2: senha via variável de ambiente ou SSO |
| **Apelido livre:** nomes ofensivos ou falsificação de identidade | Moderação manual (ocultar rodada, RF-11) | Lista de palavras bloqueadas; identificação por e-mail/SSO |
| **Ranking manipulável** (várias rodadas, scripts) | Correção no servidor e ordem forçada das respostas | Rate limit por IP; limite de rodadas por período |
| **Conteúdo desatualizado:** o Claude Code evolui rápido | Admin edita sem deploy; campo `referenceUrl` | Revisão trimestral do banco; campo “revisado em” |
| **Banco pequeno (30):** as rodadas se repetem rápido | Sorteio aleatório de 5 em 10 por nível | Crescer para 90+ perguntas e ampliar os temas |
| **SQLite** em hospedagem serverless (sistema de arquivos efêmero) | Fazer deploy em ambiente com disco persistente (VM, container com volume) | Migrar para Postgres trocando o `provider` do Prisma |

---

## 10. Roadmap pós-MVP

1. **v1.1:** proteção do admin (senha por variável de ambiente), rate limit e exportação de resultados em CSV.
2. **v2:** novos temas (Uso no dia a dia, Extensibilidade, Avançado/Automação), trilhas por tema, ranking por time.
3. **v3:** SSO corporativo, histórico por pessoa, painel de analytics para os champions (taxa de acerto por pergunta, para identificar lacunas de conhecimento).

---

## 11. Instruções para o Claude Code (implementação)

1. Crie o projeto Next.js com TypeScript, Tailwind, ESLint e App Router.
2. Configure Prisma + SQLite com o schema da §5.3, crie a migração inicial e escreva o `seed.ts` com as **30 perguntas** seguindo a §6 (confira o conteúdo na documentação oficial).
3. Implemente primeiro `lib/draw.ts`, `lib/scoring.ts` e `lib/schemas.ts`, já com testes unitários.
4. Implemente as rotas da §5.4, garantindo as regras da §5.5 (correção só no servidor, ordem forçada).
5. Construa as telas na ordem: Home → Quiz → Resultado → Ranking → Admin.
6. Aplique o design da §7 com tokens de tema claro e escuro.
7. Escreva o teste e2e do fluxo completo e confira todos os critérios da §8.
8. Documente no `README.md`: requisitos, setup, scripts e o aviso de que o `/admin` não tem autenticação.
