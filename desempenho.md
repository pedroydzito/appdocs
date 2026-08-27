# Desempenho — regras reutilizáveis

Este documento é o par do `design-system.md`: lá está como o app **parece**,
aqui está como ele **responde**.

Tudo o que segue é regra geral, com os números que o app de referência usa hoje
e o motivo de cada um — números vindos de um app em produção, não de um artigo.
Quando uma regra só vale para um recurso específico (gravar voz, editor de texto
rico, exportar PDF), ela está na §14, e é para ser aplicada **se** o seu app
tiver aquele recurso.

**Índice**

1. [Medir antes de otimizar](#1-medir-antes-de-otimizar)
2. [Orçamento de bundle](#2-orçamento-de-bundle)
3. [Carregar código sob demanda](#3-carregar-código-sob-demanda)
4. [Cache de dados](#4-cache-de-dados)
5. [Listas longas](#5-listas-longas)
6. [Render: onde o React custa caro](#6-render-onde-o-react-custa-caro)
7. [Adiar, esperar e cancelar](#7-adiar-esperar-e-cancelar)
8. [Offline-first](#8-offline-first)
9. [Imagens](#9-imagens)
10. [Web Workers](#10-web-workers)
11. [Rede e API](#11-rede-e-api)
12. [Banco de dados](#12-banco-de-dados)
13. [Observabilidade](#13-observabilidade)
14. [Recursos pesados específicos](#14-recursos-pesados-específicos)
15. [Checklist](#15-checklist)

---

## 1. Medir antes de otimizar

Três métricas de campo, não de laboratório: **LCP** (a tela apareceu), **INP**
(o toque respondeu) e **CLS** (nada pulou). Instrumente com uma ferramenta de
RUM já na primeira versão — otimizar sem medir é escolher o alvo pelo palpite.

Onde cada uma costuma quebrar:

| Métrica | Suspeito número 1                                              |
| ------- | -------------------------------------------------------------- |
| LCP     | a tela inicial: lista grande, imagem sem tamanho, fonte tardia |
| INP     | o editor de texto rico e a caixa de busca                      |
| CLS     | conteúdo entrando sem esqueleto do mesmo formato               |

**Regra:** nenhuma otimização entra sem antes existir o número que ela melhora.

---

## 2. Orçamento de bundle

Um script no CI lê a saída do build e reprova quando passa do teto:

| Métrica                         | Teto   |
| ------------------------------- | ------ |
| Carregamento inicial            | 1,5 MB |
| Por rota (navegação interna)    | 1,2 MB |
| Aviso para um chunk sob demanda | 0,4 MB |

Um teto que ninguém checa não é teto. Rode junto do `typecheck`, do lint e dos
testes, e quebre o build.

---

## 3. Carregar código sob demanda

Regra: **o que não aparece na primeira tela não entra no primeiro download.**

Duas ferramentas:

- **`next/dynamic` com `ssr: false`** para componentes pesados que só existem no
  cliente — editor de texto, gráficos, visualizador com zoom, player de áudio,
  abas do detalhe.
- **`import()` dentro da função** para bibliotecas usadas em uma ação — gerar
  PDF, soltar confete, ler um módulo de vibração.

Candidatos típicos, e o que fazer com cada um:

| Biblioteca                   | Estratégia                                        |
| ---------------------------- | ------------------------------------------------- |
| Editor de texto rico         | dinâmico + montagem adiada (ver §14.1)            |
| Gráficos                     | dinâmico, por rota                                |
| Geração de PDF / captura DOM | `import()` só no clique de exportar               |
| Confete / animação de festa  | `import()` no momento da celebração               |
| Zoom/pan de imagem           | dinâmico                                          |
| Suavização de rolagem        | estático (é global), mas desligada onde atrapalha |

Some a isso a otimização de importação por barril (`optimizePackageImports` para
pacotes de ícones, gráficos e datas): ela reescreve `import { X } from "pacote"`
para o módulo interno e evita arrastar o índice inteiro.

**Contraexemplo:** não adie o que o usuário vê em menos de um segundo. Carregar
dinamicamente o cabeçalho da página só troca um custo por um piscar.

---

## 4. Cache de dados

### 4.1 O padrão

**Stale-while-revalidate com o cache num módulo**, não dentro do componente:

1. Na montagem, entrega o que estiver em cache — a tela aparece cheia.
2. Em paralelo, busca de novo e atualiza se mudou.
3. Passado o TTL, o cache é descartado e a tela volta ao esqueleto.

O cache vive num `Map` de módulo justamente porque o roteador destrói e recria
componentes ao navegar: guardar em estado de componente é guardar em nada.

### 4.2 Os números

| Parâmetro              | Valor                      | Motivo                                                                      |
| ---------------------- | -------------------------- | --------------------------------------------------------------------------- |
| TTL de tela            | 30 min                     | 5 min passava rápido demais e a tela voltava ao esqueleto numa sessão curta |
| TTL de item individual | 60 s                       | detalhe aberto e fechado em seguida não deve refazer a busca                |
| Tamanho de página      | 20                         | o suficiente para encher a tela sem pesar a consulta                        |
| Cache de URL assinada  | 24 h, expirando 60 s antes | a margem evita o link vencer no meio do carregamento                        |

### 4.3 Regras

- **A chave descreve o pedido inteiro.** Numa lista com filtros, a chave é o
  filtro normalizado e serializado — senão trocar o filtro devolve o resultado
  errado.
- **Guarde as páginas já roladas.** Ao revalidar, substitua só a primeira fatia
  e mantenha o resto: quem rolou até a página 5 e voltou não pode ser jogado
  para o topo.
- **Invalide por prefixo**, não item por item.
- **Limpe tudo no logout.** Cache de módulo sobrevive à troca de conta, e isso é
  um vazamento de dados, não um detalhe de desempenho.
- **Estado de interface** (busca digitada, filtro escolhido) vai para
  `sessionStorage`; preferência de longo prazo (tema, contraste) vai para
  `localStorage`.
- **Se o backend tiver realtime**, use-o só para **invalidar** o cache, não para
  aplicar o dado recebido. Invalidar é uma linha; sincronizar dois caminhos de
  escrita é uma classe de bug.

### 4.4 Escrita otimista

Atualize a interface antes da resposta e guarde o valor anterior para desfazer
em caso de erro. Vale para curtir, favoritar, renomear, arquivar — qualquer
mudança pequena e reversível. Não vale para criar nem para pagar.

---

## 5. Listas longas

### 5.1 Virtualizar

Acima de ~50 itens visíveis, virtualize. Duas decisões importam:

- **Granularidade**: virtualize o **grupo** (o dia, a seção), não o item, quando
  o layout já agrupa. Menos nós medidos e o cabeçalho do grupo não pisca.
- **Estimativa de altura**: use uma fórmula, não um número. Um grupo de _n_
  itens mede `base + n × altura`. Errar a estimativa é o que faz a barra de
  rolagem tremer.

Sobra de itens fora da tela (`overscan`): **2 no celular, 5 no desktop**. Mais
que isso desperdiça o que a virtualização economizou.

### 5.2 Paginar

Rolagem infinita com `IntersectionObserver` num sentinela no fim da lista,
página de 20. Paginação por `offset/limit` no banco é suficiente até a casa das
dezenas de milhares; acima disso, chave composta (keyset).

### 5.3 Revelar sem observador

Item que entra na tela pode animar **na montagem** em vez de esperar um
`IntersectionObserver`: numa lista virtualizada a linha só monta perto da
viewport, então montar já é o sinal. Um observador a menos por item.

---

## 6. Render: onde o React custa caro

### 6.1 Isole o campo que muda a cada tecla

Uma caixa de busca dentro do componente da lista rerenderiza a lista inteira a
cada letra. Extraia o campo para um componente **memoizado** que guarda o texto
localmente e só propaga o valor depois da pausa. Foi a mudança de maior impacto
em INP no app.

### 6.2 Memoize a célula da lista

O cartão da lista é `memo()`. Numa lista virtualizada ele rerenderiza a cada
quadro da rolagem se não for.

Para a comparação rasa do `memo` funcionar, a camada de dados precisa **trocar o
objeto só quando o dado muda de verdade**. Se cada busca recria todos os
objetos, o `memo` não segura nada.

### 6.3 `useMemo` onde há derivação real

Filtro, agrupamento, ordenação e contagem sobre a lista inteira: sim. Concatenar
duas strings: não. `useMemo` tem custo próprio; usá-lo em tudo só adiciona
trabalho.

### 6.4 Anime layout com FLIP, não com layout

Quando uma lista se reordena, meça a posição antes e depois, aplique a inversão
sem transição e devolva ao normal em uma transição de `transform`. Animar
`width`/`top` recalcula layout a cada quadro; `transform` não.

---

## 7. Adiar, esperar e cancelar

### 7.1 Ocioso

Trabalho que não é para agora (indexar, pré-calcular, aquecer cache) vai para
`requestIdleCallback` com **timeout de 2s** — o timeout garante que ele aconteça
mesmo num aparelho que nunca fica ocioso. Sem `requestIdleCallback` (Safari),
caia para `setTimeout`.

### 7.2 Esperas

| Situação                                         | Espera          |
| ------------------------------------------------ | --------------- |
| Busca literal enquanto digita                    | 300 ms          |
| Busca cara (servidor, IA, embeddings)            | 600 ms          |
| Salvamento automático enquanto digita            | 1200 ms         |
| Salvamento após uma ação (não digitação)         | 300 ms          |
| Barra de progresso de rota: atraso para aparecer | 180 ms          |
| Permissão do sistema (localização etc.)          | 3500 ms de teto |

A barra de progresso merece nota: ela **espera 180ms antes de aparecer**. Sem
isso, cada navegação instantânea pisca uma barra — o que faz o app parecer mais
lento, não mais rápido.

### 7.3 Cancelar

Todo pedido disparado por digitação, por rolagem ou por navegação leva
`AbortController` e é cancelado quando o efeito é limpo. Sem isso, a resposta
velha chega depois da nova e sobrescreve o resultado certo.

### 7.4 Repetir

| Onde                         | Política                                           |
| ---------------------------- | -------------------------------------------------- |
| Chamada cara no servidor     | 3 tentativas, espera 600 → 1200 → 2400 ms + jitter |
| Trabalho de fundo no cliente | 3 tentativas, espera `tentativa × 2000 ms`         |
| Fila offline                 | 5 tentativas, depois desiste e avisa               |

O jitter não é enfeite: sem ele, mil clientes que falharam juntos voltam juntos.

E **só repita erro de rede**. Repetir um 400 é repetir um erro seu.

---

## 8. Offline-first

Três camadas, e cada uma resolve uma coisa:

### 8.1 Service worker — a casca

| Recurso            | Estratégia                                  |
| ------------------ | ------------------------------------------- |
| Navegação          | rede primeiro, cache como rede de segurança |
| Estáticos com hash | cache primeiro (o hash já é a versão)       |
| Imagens do usuário | cache e revalida em paralelo                |
| API                | **sempre rede**, nunca cache                |

Detalhes que evitam dor:

- **Versione o cache** (`app-v8`) e apague os antigos ao ativar. Cache de app
  sem versão é a causa clássica do "atualizei e continua velho".
- **Cache de estáticos e de imagens atravessa versões** — não faz sentido baixar
  de novo o que tem hash no nome.
- **Ponha teto e descarte o mais antigo**: 120 arquivos estáticos, 400 imagens.
  Sem teto, o armazenamento cresce até o sistema apagar tudo de uma vez.
- **Normalize a chave da imagem** removendo o token da URL assinada — senão cada
  assinatura nova vira uma entrada nova no cache.
- **Pré-carregue** as rotas principais e uma página de "sem conexão".
- Sirva o próprio arquivo do service worker com `Cache-Control: no-store`.
- **Registre só em produção**, depois do `load`. Em desenvolvimento ele só
  atrapalha.

### 8.2 Banco local — o que ainda não subiu

IndexedDB com uma conexão compartilhada (uma promessa em módulo) e tratamento
para múltiplas abas. Guarde ali: rascunhos, arquivos binários pesados e a fila.

### 8.3 Fila — a garantia

Uma operação por vez, em ordem, parando na primeira falha de rede. Entra na fila
**só** o que falhou por rede. Item na fila aparece na interface com identidade
provisória e uma marca de pendente — a pessoa vê o que escreveu, com o aviso de
que ainda não subiu.

Tente esvaziar na montagem e no evento `online`. Mostre o indicador de
sincronização **apenas quando houver pendências**.

---

## 9. Imagens

### 9.1 Comprima no cliente, antes de subir

| Uso       | Lado maior | Qualidade |
| --------- | ---------- | --------- |
| Exibição  | 1600 px    | 0.82      |
| Miniatura | 400 px     | 0.75      |

Formato: tente AVIF, caia para WebP, caia para JPEG. **Nunca aumente** uma
imagem (`escala = min(1, maxLado / ladoMaior)`).

Suba **duas versões** — a de exibição e a miniatura, com nome derivado
(`{id}.webp` e `{id}-mini.webp`). Uma listagem que carrega 25 KB por miniatura
em vez de 300 KB é a diferença entre uma galeria usável e uma inusável, e evita
depender de redimensionamento sob demanda no servidor a cada acesso.

Faça isso num worker (§10).

### 9.2 Exiba com disciplina

- `loading="lazy"` e `decoding="async"` em imagem fora da primeira tela.
- **Sempre** `width`/`height` (ou proporção fixa): é o que evita o salto de
  layout.
- `sizes` coerente com o layout, para o navegador não baixar a maior.
- Em lista virtualizada, assine/carregue **só os grupos visíveis**.
- Peça a versão grande **sob demanda**, ao ampliar.

### 9.3 URLs assinadas

Assine **em lote** (até 100 por chamada), guarde em memória e em
`sessionStorage`, e expire 60s antes do prazo real. Junte os pedidos do mesmo
ciclo de render com `queueMicrotask` — a lista inteira vira uma chamada.

---

## 10. Web Workers

Mande para um worker todo laço que passe de ~50ms na thread principal:

| Trabalho                             | Por quê                                |
| ------------------------------------ | -------------------------------------- |
| Decodificar e recomprimir imagem     | uma foto de 3–8 MB congela a interface |
| Transformação pixel a pixel          | milhões de pixels, 1–2s parados        |
| Grafo/agregação sobre coleção grande | O(n²) engasga a rolagem                |

Padrão:

- **Um worker por chamada, encerrado no fim** (`terminate()`), ou encerrado na
  limpeza do efeito. Pool de workers é otimização prematura para trabalho
  esporádico.
- Correlacione pedido e resposta por um `id` na mensagem.
- Passe **dados achatados** — a mensagem é clonada estruturalmente e não carrega
  funções nem classes.
- Transfira `ArrayBuffer`/`ImageBitmap` em vez de copiar.
- **Tenha sempre um caminho síncrono de reserva**: se o worker falhar ao criar,
  o recurso não pode sumir.

---

## 11. Rede e API

### 11.1 Tempo limite por tipo de rota

Declare o teto de duração de cada rota, e que ele seja proporcional ao trabalho:
30s para uma busca, 60s para uma operação comum, 120s para geração longa, 300s
para um trabalho em lote. Rota sem teto declarado morre no teto padrão da
plataforma, no meio do trabalho.

### 11.2 Streaming

Resposta longa gerada por modelo vai em stream, com:

```
Content-Type: text/plain; charset=utf-8
Cache-Control: no-store
X-Accel-Buffering: no
```

O último cabeçalho desliga o buffer do proxy — sem ele o stream chega inteiro no
fim e você pagou a complexidade sem ganhar nada.

Onde o modelo permitir escolher o esforço de raciocínio, use o **mais baixo** no
que é conversa: a diferença medida foi ~1s contra ~14s até o primeiro token, e o
primeiro token é o que a pessoa sente.

### 11.3 Limite de uso

Limite na portaria, por rota e por usuário, com perfis:

| Perfil      | Janela             |
| ----------- | ------------------ |
| padrão      | 30 chamadas / 60 s |
| caro        | 10 chamadas / 60 s |
| alto volume | 40 chamadas / 60 s |

Responda **429** com `Retry-After`. Um contador em memória por processo já
resolve para uma instância; se houver várias, mova para armazenamento
compartilhado.

Valide o tamanho da entrada com um esquema (texto até 60 mil caracteres,
pergunta até 4 mil). O limite de tamanho é limite de custo.

### 11.4 Pré-carregar o provável

- `prefetch` nos links de navegação.
- No `pointerenter`/`pointerdown` de um cartão, pré-carregue **a rota e o dado**.
  O intervalo entre passar o mouse e clicar é de graça.
- `preconnect` + `dns-prefetch` para o domínio do backend, no `<head>`.

---

## 12. Banco de dados

### 12.1 Nunca `select("*")`

Liste as colunas. O caso concreto: uma coluna de vetor de embeddings respondia
por **86% do tráfego** da tela inicial (9.762 de 11.389 bytes por linha) e não
era usada em nada ali. Tenha um conjunto de colunas por caso de uso — "colunas
do cartão" é diferente de "colunas do detalhe" — e um teste que garante que a
coluna gorda ficou fora.

### 12.2 Índices declarados junto da consulta

Para cada consulta de tela, um índice:

| Consulta                        | Índice                            |
| ------------------------------- | --------------------------------- |
| Listagem cronológica            | `(usuario, criado_em)`            |
| Filtro por data                 | `(usuario, data)`                 |
| Filtro por etiqueta (array)     | GIN                               |
| Busca textual                   | GIN sobre `tsvector` no idioma    |
| Subconjunto pequeno e frequente | índice parcial (`where favorito`) |
| Busca por similaridade vetorial | HNSW                              |

### 12.3 Evite o N+1

- Uma consulta por tela, não uma por item.
- Assinatura de URLs em lote.
- Contagem agregada numa **view**, em vez de trazer as linhas para contar no
  cliente.
- `Promise.all` quando as consultas são independentes.
- **Teste o orçamento**: um teste estático que conta as chamadas ao banco por
  arquivo de tela e reprova quando passa do teto. É o único jeito de o N+1 não
  voltar sozinho.

### 12.4 Uma camada de acesso, e só uma

Todo acesso passa por um módulo. É lá que ficam cache, escrita otimista,
tolerância a coluna ausente e o `join` certo. Componente que fala direto com o
banco é onde o N+1 nasce.

---

## 13. Observabilidade

- **RUM em produção** para LCP/INP/CLS por rota.
- **Log estruturado** em JSON, com uma lista de campos proibidos (conteúdo do
  usuário, token, e-mail). Log que vaza conteúdo é incidente, não observabilidade.
- **Relato de erro do cliente** para o servidor, com `keepalive: true` (o
  `fetch` precisa sobreviver ao fechamento da aba) e pilha truncada em ~4000
  caracteres.
- **Ouvintes globais** de `error` e `unhandledrejection`.
- **Validação de ambiente no boot**: falte uma variável e o processo diz qual,
  em vez de quebrar na primeira requisição.

---

## 14. Recursos pesados específicos

Aplique **se** o app tiver o recurso.

### 14.1 Editor de texto rico

O maior custo de INP de um app assim. Três camadas:

1. `next/dynamic` com `ssr: false`.
2. **Montagem adiada**: mostre o texto como HTML estático e só monte o editor
   quando (a) a pessoa clicar/focar, ou (b) o navegador ficar ocioso (timeout de
   2,5s; 1,2s de `setTimeout` onde não houver `requestIdleCallback`). Quem só
   veio ler nunca paga o editor.
3. Salvamento automático com 1200ms de espera.

### 14.2 Gravação de voz

- Grave em pedaços (`MediaRecorder.start(1000)`) — um pedaço por segundo.
- **Salve o rascunho a cada 15s** em banco local. Gravação perdida por travada
  ou aba fechada é dado perdido, e é o pior tipo de bug de desempenho: aquele em
  que o usuário perde trabalho.
- Peça **wake lock** durante a gravação, e solte no fim.
- Transcrição/pós-processamento em background, com espera proporcional à
  duração (~4s para menos de 1 min, 10s até 3 min, 20s acima).
- Desenhe a forma de onda em canvas, nunca em DOM por amostra.
- Nunca deixe o arquivo de áudio sem compressão subir na thread principal.

### 14.3 Gráficos

Dinâmicos por rota. Agregue os dados **antes** de entregar ao gráfico (de
preferência no banco ou num worker) — biblioteca de gráfico não é ferramenta de
agregação. Anime a entrada por CSS escalonado, não recalculando a série.

### 14.4 Exportar PDF / capturar a tela

Biblioteca de captura de DOM e de PDF são das mais pesadas que existem: só por
`import()` no clique. Quando o resultado for uma imagem simples (um cartão para
compartilhar), desenhe em `canvas` à mão — evita centenas de KB de dependência.

### 14.5 Suavização de rolagem

Um `lerp` de **0.12** dá o peso certo sem parecer atraso. Não sincronize o
toque (o gesto nativo do celular já é bom, e roubá-lo custa quadros). Marque as
áreas de rolagem interna com um atributo para a biblioteca não capturá-las, e
**pare** a suavização nas telas de escrita — rolagem animada embaixo do cursor
é desconfortável.

### 14.6 Chamadas a modelos de IA

- Streaming sempre que a resposta for longa (§11.2).
- Trabalho de indexação/embedding fora do caminho crítico, no ocioso.
- Estime e registre o custo por chamada; sem isso ninguém descobre a rota cara
  antes da fatura.
- Um modo degradado quando não há chave configurada — o app continua de pé, só
  com menos recurso.

---

## 15. Checklist

Antes de dar uma tela por pronta:

- [ ] A lista longa está virtualizada e paginada.
- [ ] O campo que muda a cada tecla está isolado e memoizado.
- [ ] Toda busca tem espera e `AbortController`.
- [ ] Nada de `select("*")`; existe índice para a consulta desta tela.
- [ ] Cada consulta acontece uma vez, não uma por item.
- [ ] Imagem tem tamanho declarado, `lazy` fora da primeira tela e miniatura
      própria.
- [ ] Biblioteca pesada entra por `dynamic` ou `import()`.
- [ ] Trabalho de mais de 50ms está num worker ou no ocioso.
- [ ] A tela tem estado de carregando com o **formato** do conteúdo.
- [ ] O que falha por rede vai para a fila; o que falha por erro do cliente, não.
- [ ] O cache tem chave que descreve o pedido e é limpo no logout.
- [ ] O orçamento de bundle continua passando.
