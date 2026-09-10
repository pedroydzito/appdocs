# Design system — especificação completa

Cor, tipografia, espaço, forma, movimento, interação, componentes, navegação e
padrões de tela — a especificação inteira de uma interface.

Ela nasceu de um app real (um diário pessoal), mas nada aqui depende do
assunto: quem seguir este documento constrói um aplicativo **diferente** — outro
conteúdo, outra cor de acento — que parece o mesmo produto. Onde uma decisão
depende do domínio, está dito em voz alta, com a instrução de como resolver no
seu caso.

**Monte uma página `/estilo`**, fora da navegação, mostrando todo componente e
os quatro temas lado a lado. Este documento é a lei; aquela página é a prova, e
é onde um componente fora do padrão aparece antes de chegar ao usuário.

**Índice**

1. [Princípios](#1-princípios)
2. [Base técnica](#2-base-técnica)
3. [Cor](#3-cor)
4. [Tipografia](#4-tipografia)
5. [Espaço, largura, forma e elevação](#5-espaço-largura-forma-e-elevação)
6. [Movimento](#6-movimento)
7. [Interação](#7-interação)
8. [Componentes](#8-componentes)
9. [Moldura e navegação](#9-moldura-e-navegação)
10. [Padrões de tela](#10-padrões-de-tela)
11. [Gestos](#11-gestos)
12. [Acessibilidade](#12-acessibilidade)
13. [Voz e escrita](#13-voz-e-escrita)
14. [Como adaptar para outro app](#14-como-adaptar-para-outro-app)
15. [Checklist](#15-checklist)

---

## 1. Princípios

Sete regras. Tudo o que vem depois é consequência delas.

1. **Um token, um valor.** Nenhum hexadecimal, nenhuma duração e nenhum raio
   escrito à mão dentro de um componente. Se um valor aparece duas vezes, ele
   vira token.
2. **Um componente, um gesto.** Botão, campo, chip, folha e cartão existem uma
   vez só. Antes de escrever uma classe de estilo, procure o componente — foi a
   falta disso que gerou 62 cópias manuais do botão de acento, cada uma com o
   seu padding, raio e hover.
3. **O acento é identidade, não semântica.** A cor da marca aparece com
   parcimônia e não significa "perigo" nem "sucesso" — cada um desses tem token
   próprio.
4. **Hover é confirmação, não anúncio.** O véu é de 5,5%. A resposta ao toque
   chega antes da resposta da rede.
5. **Movimento conta uma história.** Nada anima por enfeite: cada transição diz
   de onde veio ou para onde foi. Tudo é `transform`/`opacity`, e tudo morre sob
   `prefers-reduced-motion`.
6. **Contraste é teste, não opinião.** O build reprova quando um par de cores
   cai abaixo de AA (AAA no alto contraste).
7. **Caixa baixa, verbo no infinitivo, português sem jargão.** A voz do produto
   é parte do design.

---

## 2. Base técnica

| Peça      | Escolha                                                               |
| --------- | --------------------------------------------------------------------- |
| Framework | Next.js (App Router), React 19                                        |
| CSS       | Tailwind v4 (`@import "tailwindcss"`), tokens em `globals.css`        |
| Ícones    | `lucide-react`                                                        |
| Rolagem   | `lenis` (suavização de roda)                                          |
| Animação  | CSS + View Transitions API + uma mola própria. **Sem framer-motion.** |
| Fontes    | `next/font/google`                                                    |

### Tema por classe, não por `prefers-color-scheme`

O tema é uma classe no `<html>`, aplicada por um script que roda **antes da
primeira pintura** (senão o app pisca branco para quem usa escuro). No Tailwind
v4 isso exige redefinir a variante:

```css
@custom-variant dark (&:where(.dark, .dark *));
```

São **quatro** combinações, não duas: `claro`, `escuro`, e o modificador
`.contraste-alto`, que combina com os dois. O script lê `localStorage` e liga as
classes:

```js
var escuro =
  salvo === "escuro" || (salvo === "sistema" && matchMedia("(prefers-color-scheme: dark)").matches);
raiz.classList.toggle("dark", escuro);
raiz.style.colorScheme = escuro ? "dark" : "light";
raiz.classList.toggle("contraste-alto", contraste === "alto");
```

### Duas armadilhas do CSS por utilitário

As duas custaram uma tarde cada, e nenhuma delas dá erro:

- **Variante não pega em classe de componente.** `hover:cartao`,
  `focus:superficie-2`, `data-[state=open]:superficie-2` — nada disso existe: a
  variante só compõe com **utilidade**, e uma classe declarada em
  `@layer components` não é uma. O CSS compila, a classe some, e o hover
  simplesmente não acontece. Ou exponha o token como cor no `@theme inline`
  (`hover:bg-superficie-2`) ou escreva a regra `:hover` dentro da própria classe.
- **Variável declarada e não usada é removida.** Ver §3.1: a escala inteira
  precisa existir em `:root`, não só dentro de `@theme`.

### Fontes

Três famílias, e **só os pesos que a interface desenha de fato** — eram doze
arquivos, metade nunca usada.

| Papel     | Família             | Pesos                | Variável CSS        | Token            |
| --------- | ------------------- | -------------------- | ------------------- | ---------------- |
| Display   | Bricolage Grotesque | 700                  | `--font-bricolage`  | `--font-display` |
| Interface | Geist               | 400, 500, 600, 700   | `--font-geist`      | `--font-sans`    |
| Leitura   | Newsreader          | 400 normal + itálico | `--font-newsreader` | `--font-serif`   |

Todas com `display: "swap"`. **As variáveis de fonte vão no `<html>`, não no
`<body>`**: o `@theme` do Tailwind declara `--font-display` em `:root`, e um
`var()` dentro de custom property é resolvido no elemento que a declara — com as
fontes só no `<body>` a declaração inteira vira inválida e tudo cai na fonte do
sistema.

A serifada é o texto longo (o conteúdo que a pessoa escreveu). A display é só
título. A sans é o resto.

---

## 3. Cor

### 3.1 A escala do acento

Uma cor de marca, e uma escala derivada dela — mesmo matiz e mesma saturação,
variando só a luminosidade. Não são "outras cores da marca": existem para os
lugares que precisam de mais ou menos contraste.

```css
@theme {
  --color-acento-50: #fef0ee;
  --color-acento-100: #fce0de;
  --color-acento-200: #f9bdb8;
  --color-acento-300: #f68c83;
  --color-acento-400: #f55b4f;
  --color-acento-500: #f43a2b; /* a cor da marca, e só ela */
  --color-acento-600: #e41c0c;
  --color-acento-700: #b9190c;
  --color-acento-800: #92160c;
  --color-acento-900: #70140c;
  --color-acento-950: #400e0a;
}
```

**Declare os onze degraus em `:root` e faça a ponte no `@theme inline`.** Uma
variável que só existe dentro de `@theme` e que nenhuma regra usa é **removida
na compilação** — e o resultado é sutil: o botão continua certo (o 500 é usado
pelos papéis), mas a página de estilo mostra metade da escala transparente, e
qualquer `bg-acento-200` escrito depois não pinta nada.

```css
:root {
  --acento-50: #fef0ee;
  /* … os onze … */
}
@theme inline {
  --color-acento-50: var(--acento-50);
  /* … os onze … */
}
```

Assim as utilidades do Tailwind continuam existindo, o alto contraste continua
podendo reescrever a escala inteira, e nada some por falta de uso.

### 3.2 Papéis, não cores

Nenhuma tela usa `--color-acento-500` diretamente para decidir contraste. Ela
usa o **papel**:

| Token                                            | Papel                                                      |
| ------------------------------------------------ | ---------------------------------------------------------- |
| `--acento-solido`                                | Preenchimento (botão primário, chip selecionado)           |
| `--acento-solido-texto`                          | O que vai **por cima** do preenchimento                    |
| `--acento-texto`                                 | Acento como **letra** (menção, erro, link de sistema)      |
| `--perigo` / `--perigo-texto` / `--perigo-tenue` | Destruir                                                   |
| `--sucesso` / `--sucesso-tenue`                  | Confirmar                                                  |
| `--aviso` / `--aviso-tenue`                      | Alertar                                                    |
| `--favorito`                                     | A estrela. Não é aviso nem acento — é o dourado de marcar. |
| `--link`                                         | Link dentro de texto corrido                               |
| `--anel-foco`                                    | Halo de foco de teclado                                    |
| `--veu-hover` / `--veu-ativo`                    | A camada de hover e de toque                               |
| `--chip-tinta`                                   | Quanto de cor entra no fundo de um chip tingido            |

Duas decisões que parecem erradas e não são:

- **`--perigo` é o próprio acento.** Houve uma tentativa de dar ao perigo um
  vermelho só dele (um rosado a 20° do acento) para "excluir" não parecer
  "salvar". Na prática ficaram dois vermelhos parecidos disputando a mesma tela,
  e o de perigo lia como um acento errado. Destruir é destaque, e o acento já é
  a cor de destaque. **Se o seu acento não for vermelho**, aí sim `--perigo`
  precisa ser uma cor própria — ver §14.
- **`--link` é azul e sublinhado**, contra a identidade. É a única convenção que
  a web inteira ensinou, e aqui ela precisa vencer a marca: um link com a cor do
  acento se confundiria com um botão.

### 3.3 Os quatro temas, por inteiro

**Claro** — papel quente, não branco puro. Branco puro em tela cheia cansa a
vista e é o que dá ar de template genérico.

```css
:root {
  --fundo: #f5f4f0;
  --superficie: #fffefb;
  --superficie-2: #eceae3;
  --superficie-3: #e2dfd6;
  --borda: #e0ddd4;
  --borda-forte: #cdc9bd;
  --texto: #16130f;
  --texto-2: #4a453e;
  --texto-3: #6b645b;

  --sombra-baixa: 0 1px 2px rgb(22 19 15 / 0.04);
  --sombra: 0 1px 2px rgb(22 19 15 / 0.04), 0 4px 16px -6px rgb(22 19 15 / 0.08);
  --sombra-alta: 0 2px 4px rgb(22 19 15 / 0.05), 0 12px 32px -8px rgb(22 19 15 / 0.16);

  --perigo: #f43a2b;
  --perigo-tenue: rgb(244 58 43 / 0.1);
  --perigo-texto: #ffffff;
  --sucesso: #0f7a52;
  --sucesso-tenue: rgb(15 122 82 / 0.1);
  --aviso: #9a5b12;
  --aviso-tenue: rgb(154 91 18 / 0.1);
  --favorito: #b87a14;
  --link: #1f5fd0;

  --veu-hover: rgb(0 0 0 / 0.055);
  --veu-ativo: rgb(0 0 0 / 0.09);

  --acento-solido: #f43a2b;
  --acento-solido-texto: #ffffff;
  --acento-texto: #b9190c;
  --anel-foco: rgb(244 58 43 / 0.4);

  --sobre-emocao: #ffffff;
  --chip-tinta: 5%;
}
```

**Escuro** — preto de verdade, não cinza-azulado.

```css
.dark {
  --fundo: #0a0a0a;
  --superficie: #141414;
  --superficie-2: #1c1c1c;
  --superficie-3: #262626;
  --borda: #242424;
  --borda-forte: #363636;
  --texto: #fafaf9;
  --texto-2: #b8b5b0;
  --texto-3: #8b8781;

  --sombra-baixa: 0 1px 2px rgb(0 0 0 / 0.3);
  --sombra: 0 1px 2px rgb(0 0 0 / 0.4);
  --sombra-alta: 0 12px 32px -8px rgb(0 0 0 / 0.6);

  --perigo-tenue: rgb(244 58 43 / 0.16);
  --sucesso: #4ade9b;
  --sucesso-tenue: rgb(74 222 155 / 0.14);
  --aviso: #e0a458;
  --aviso-tenue: rgb(224 164 88 / 0.14);
  --favorito: #f5c451;
  --link: #7cb2ff;

  /* No escuro o véu é BRANCO: a superfície clareia ao receber o ponteiro. */
  --veu-hover: rgb(255 255 255 / 0.07);
  --veu-ativo: rgb(255 255 255 / 0.11);

  --acento-texto: #f55b4f;
  --anel-foco: rgb(244 58 43 / 0.5);
  --sobre-emocao: #000000;
}
```

Repare no que **não** é redeclarado no escuro: `--acento-solido`,
`--acento-solido-texto` e `--perigo-texto`. O vermelho da marca é o mesmo nos
dois temas de propósito — não existe "o vermelho do claro" e "o vermelho do
escuro". Só `--acento-texto` muda, porque o problema é o oposto: no claro o
`#f43a2b` era escuro demais como letra fina; no escuro é claro demais sobre o
fundo tingido.

**Alto contraste** — não é um terceiro tema, é um **modificador** que combina
com os outros dois. Sobe o mínimo de 4.5 (AA) para 7 (AAA), engrossa a borda e
**mata toda a cor**: só preto, branco e cinza. É assim que funcionam os modos de
alto contraste do sistema operacional — cor decorativa é a primeira coisa que
atrapalha quem depende desse modo.

```css
.contraste-alto {
  --fundo: #ffffff;
  --superficie: #ffffff;
  --superficie-2: #ededed;
  --superficie-3: #dcdcdc;
  --borda: #767676;
  --borda-forte: #1a1a1a;
  --texto: #000000;
  --texto-2: #1f1f1f;
  --texto-3: #3d3d3d;

  /* A escala INTEIRA vira cinza — assim `bg-acento-500/10` e qualquer
     utilidade do Tailwind acompanham de graça, porque no v4 elas compilam
     para var(--color-acento-*) e não para o hexadecimal cru. */
  --color-acento-500: #4d4d4d;
  --color-acento-700: #1a1a1a;
  /* … os onze degraus, de #f7f7f7 a #000000 … */

  --acento-solido: #000000;
  --acento-solido-texto: #ffffff;
  --acento-texto: #000000;
  --perigo: #000000;
  --sucesso: #000000;
  --aviso: #000000;
  --favorito: #000000;
  --link: #000000;
  --veu-hover: rgb(0 0 0 / 0.12);
  --veu-ativo: rgb(0 0 0 / 0.2);
  --anel-foco: rgb(0 0 0 / 0.75);
}

.dark.contraste-alto {
  --fundo: #000000;
  --superficie: #0d0d0d;
  --superficie-2: #1f1f1f;
  --superficie-3: #2e2e2e;
  --borda: #8f8f8f;
  --borda-forte: #ffffff;
  --texto: #ffffff;
  --texto-2: #ededed;
  --texto-3: #c9c9c9;

  /* O botão primário se INVERTE: fundo branco, rótulo preto. É o oposto do
     fundo da página, e é isso que o faz saltar sem depender de matiz. */
  --acento-solido: #ffffff;
  --acento-solido-texto: #000000;
  --acento-texto: #ffffff;
  --veu-hover: rgb(255 255 255 / 0.14);
  --veu-ativo: rgb(255 255 255 / 0.22);
  --anel-foco: rgb(255 255 255 / 0.85);
}
```

### 3.4 Cor de categoria

Quase todo app tem um conjunto fechado de categorias que precisa de cor:
prioridade, status, projeto, tipo de despesa, disciplina, humor. Ele **não** faz
parte do sistema — é conteúdo. O que é sistema é a forma de declará-lo:

- Um token por categoria, **e um valor por tema** (`--categoria-<id>`). A mesma
  cor quase nunca serve no claro e no escuro: no papel claro ela precisa ser
  escura o bastante para se ler; no preto, clara o bastante.
- O catálogo em código devolve `{ id, rotulo, cor: "var(--categoria-x)" }`.
  **Nunca um hexadecimal** — é isso que faz a cor trocar sozinha com o tema.
- No alto contraste elas perdem a cor mas **não** a diferença: viram degraus de
  cinza na mesma ordem, e o gráfico continua distinguindo uma da outra.
- Se o seu app não tem categorias coloridas, apague o grupo inteiro. Nada mais
  depende dele.

Duas regras aprendidas a duras penas:

- **Uma cor por categoria, e só uma.** Ela é o pontinho, o ícone, a barra do
  gráfico E a palavra escrita no chip. Houve uma fase com um segundo tom,
  escurecido só para o texto: garantia os 4.5 da WCAG e estragava a tela — todo
  laranja virava marrom no tema claro, e o mesmo chip aparecia numa cor na lista
  e noutra na edição. O tom vivo ficou, com o piso de 3:1 que o teste cobra.
- **A cor nunca é o único sinal.** O nome está sempre escrito ao lado.

### 3.5 O véu de hover

Um padrão para o app inteiro. Antes cada componente inventava o seu: o primário
clareava (`brightness-110`), o de superfície desbotava (`opacity-80`), o fantasma
ganhava fundo cinza. Três respostas para o mesmo gesto, e no escuro uma delas ia
para o lado errado.

A regra segue a luz: **no claro o véu é preto** (tudo escurece um pouco), **no
escuro é branco** (tudo clareia). A superfície se aproxima do dedo.

É `background-image`, não `filter`:

```css
background-image: linear-gradient(var(--veu-hover), var(--veu-hover));
```

O filtro clareia em qualquer tema e não alcança botão de fundo transparente. O
gradiente de uma cor só é uma camada plana pintada dentro da caixa — funciona
sobre fundo pintado, transparente ou imagem. E não mexe em `background-color`,
então quem tem cor própria (o primário, o de perigo, o chip da emoção) a mantém
com o véu por cima.

### 3.6 O contraste é testado

Um teste lê o `globals.css`, extrai os tokens dos quatro temas e reprova o build
quando um par cai abaixo do mínimo. O que ele cobra:

| Par                                                                     | Mínimo |
| ----------------------------------------------------------------------- | ------ |
| `--texto`, `--texto-2`, `--texto-3` sobre fundo/superfície/superfície-2 | 4.5    |
| `--sucesso`, `--aviso`, `--acento-texto`, `--link` como texto           | 4.5    |
| `--perigo`, `--acento-solido`, `--favorito` como preenchimento          | 3      |
| Rótulo sobre preenchimento (`--acento-solido-texto` etc.)               | 3      |
| Cor de categoria sobre a superfície e sobre o próprio chip tingido      | 3      |
| `--borda-forte` contra a superfície que ela separa                      | 1.4    |
| Tudo o que é texto, no alto contraste                                   | 7      |

O `--texto-3` já precisou ser corrigido duas vezes por ficar abaixo do mínimo, e
nas duas o problema só apareceu depois de ir para a tela. Por isso é teste.

---

## 4. Tipografia

### 4.1 A escala

Oito degraus, no lugar dos 100+ tamanhos avulsos (`text-[13.5px]` e afins). São
`@utility` e não classes soltas, porque só assim o Tailwind gera as variantes —
sem isso `sm:txt-sm` não existe e as telas voltam para o valor arbitrário no
responsivo.

| Classe           | Tamanho                      | Entrelinha | Papel                  |
| ---------------- | ---------------------------- | ---------- | ---------------------- |
| `.txt-xs`        | 0.6875rem (11px)             | 1.45       | metadado               |
| `.txt-sm`        | 0.8125rem (13px)             | 1.5        | apoio, chip            |
| `.txt-md`        | 0.9375rem (15px)             | 1.6        | corpo padrão           |
| `.txt-lg`        | 1.0625rem (17px)             | 1.65       | corpo longo            |
| `.txt-xl`        | 1.1875rem (19px)             | 1.5        | leitura serifada       |
| `.txt-titulo`    | 1.5rem (24px)                | 1.25       | cabeçalho de seção     |
| `.txt-display-2` | `clamp(2.25rem, 5vw, 3rem)`  | 1          | título de painel/folha |
| `.txt-display`   | `clamp(2.5rem, 6vw, 3.5rem)` | 0.98       | título de tela         |

O `clamp` substitui o par `2.5rem` / `lg:3.5rem` que sete telas repetiam à mão,
sem o degrau seco no breakpoint.

Mais dois degraus, para **número grande** — a estatística de um cartão, o
contador de uma tela vazia:

| Classe          | Tamanho                        | Papel                          |
| --------------- | ------------------------------ | ------------------------------ |
| `.txt-numero`   | `clamp(1.75rem, 4vw, 2.25rem)` | número dentro de cartão        |
| `.txt-numero-g` | `clamp(2.5rem, 8vw, 4.5rem)`   | número que É a tela (destaque) |

Eles existem porque quatro tamanhos diferentes tinham sido inventados para o
mesmo papel (1.75/2.25, 2.25/2.75, 2.5 fixo e 3.5/4.5). Quando um papel aparece
duas vezes com números diferentes, o certo não é escolher um: é **dar nome aos
dois degraus** e usar só eles.

E um terceiro, que só aparece quando o cartão **divide a linha** com outro:

| Classe                  | Tamanho                                    | Papel                                  |
| ----------------------- | ------------------------------------------ | -------------------------------------- |
| `.txt-numero-estreito`  | `.txt-xl` até `sm`, depois `.txt-numero`   | número em cartão que divide a linha    |

O `.txt-numero` foi medido para um cartão de largura inteira. Num mosaico de
duas colunas no celular, o cartão tem ~45% da tela, e um valor com separador de
milhar (`R$ 2.479,50`) **transborda a caixa** — o texto sai pela borda do
cartão, que é o defeito visual mais fácil de deixar passar porque só aparece
com dado real e comprido. Não invente um tamanho no meio: componha os degraus
que já existem, e mude no ponto de quebra.

### 4.2 Os três gestos tipográficos

```css
/* Título: caixa baixa e tracking apertado. É o que dá personalidade
   e afasta da cara de template. */
.display {
  font-family: var(--font-display);
  font-weight: 700;
  letter-spacing: -0.035em;
  line-height: 0.95;
}

/* Numerais grandes de estatística. */
.numeral {
  font-family: var(--font-display);
  font-weight: 700;
  letter-spacing: -0.05em;
  font-variant-numeric: tabular-nums;
  line-height: 0.85;
}

/* Sobrancelha: o contexto acima do título. */
.rotulo {
  font-size: 0.6875rem;
  font-weight: 500;
  letter-spacing: 0.08em;
  text-transform: uppercase;
  color: var(--texto-3);
}
```

**A sobrancelha é curta, e não é rótulo de tudo.** Duas armadilhas, as duas
vistas em produção:

- **Sobrancelha que vira duas linhas deixou de ser sobrancelha.** Caixa alta com
  `letter-spacing` ocupa ~30% mais que a mesma frase em caixa baixa; uma frase
  como "contas a pagar e receber, parcelas e recorrências" quebra em duas linhas
  no celular e passa a competir com o título que ela deveria apresentar. Se não
  cabe numa linha estreita, não é contexto: é descrição, e desce para o corpo.
- **Rótulo de ajuste e de campo de formulário é frase, não sobrancelha.** Numa
  tela de ajustes com dez linhas, dez rótulos em caixa alta viram ruído e a
  descrição embaixo fica ilegível por contraste de peso. O rótulo de um campo de
  formulário usa a sobrancelha (ele é um por bloco, e é o contexto do controle);
  o de uma **linha de ajuste** é `.txt-sm font-medium`, com a descrição em
  `.txt-xs` na cor de metadado.

### 4.3 Cor do texto

`.t2` → `--texto-2` (apoio). `.t3` → `--texto-3` (metadado). O texto principal
não precisa de classe: o `body` já o define.

### 4.4 Base

**O `body` declara o tamanho do corpo.** Sem isso, todo texto sem classe cai nos
**16px do navegador** — que é maior que qualquer degrau da escala. O sintoma não
é "um texto errado": é a sensação de que a interface tem dois tamanhos
brigando, porque metade dos textos passou pela escala e a outra metade não. A
escala só é a escala quando o padrão também é dela.

```css
body {
  background-color: var(--fundo);
  color: var(--texto);
  font-family: var(--font-sans);
  font-size: 0.9375rem; /* .txt-md — o degrau do corpo */
  line-height: 1.6;
  -webkit-font-smoothing: antialiased;
  text-rendering: optimizeLegibility;
  overscroll-behavior-y: none;
}

/* Evita o zoom automático do iOS ao focar um campo. */
input,
textarea,
select {
  font-size: max(16px, 1em);
}

::selection {
  background-color: var(--color-acento-500);
  color: white;
}
```

---

## 5. Espaço, largura, forma e elevação

### 5.1 Espaçamento

A escala do Tailwind já é boa; o que faltava era dizer **onde cada passo entra**,
para parar de alternar entre `gap-2.5` e `gap-3` sem motivo.

| Passo | px  | Onde                                           |
| ----- | --- | ---------------------------------------------- |
| 1     | 4   | dentro de um controle — ícone colado ao rótulo |
| 2     | 8   | entre irmãos próximos — chips de uma linha     |
| 3     | 12  | entre campos de um mesmo grupo                 |
| 4     | 16  | padding de linha de lista, entre grupos        |
| 5     | 20  | padding de **cartão**                          |
| 6     | 24  | entre blocos de uma seção, cartão grande       |
| 8     | 32  | entre seções                                   |

Meios-passos (1.5, 2.5, 3.5) **só** quando um passo cheio quebra o alinhamento
de um ícone. Nada de valor arbitrário.

O passo 5 entrou depois, e a razão vale ser dita: ele já era o padding de
cartão mais usado do app, e não estava nesta tabela. A lista dizia 4 ou 6, a
tela fazia 5, e cada cartão novo era um sorteio entre os três. Escala que não
descreve a realidade não é seguida — é contornada.

### 5.2 Largura

| Token                 | Valor | Uso                                      |
| --------------------- | ----- | ---------------------------------------- |
| `.pagina` (max-width) | 72rem | moldura externa de qualquer tela         |
| `--largura-conteudo`  | 48rem | `.coluna-conteudo` — formulário, detalhe |
| `--largura-leitura`   | 68ch  | `.coluna-leitura` — texto corrido        |

68 caracteres é onde o olho não se perde na volta da linha.

### 5.3 Raio

| Token           | Valor   | Onde                                     |
| --------------- | ------- | ---------------------------------------- |
| `--raio-p`      | 0.5rem  | ladrilho de ícone, esqueleto de linha    |
| `--raio-m`      | 0.75rem | **botão, campo, campo de busca, nota**   |
| `--raio-g`      | 1rem    | painel interno                           |
| `--raio-cartao` | 1.25rem | cartão, folha, menu                      |
| `--raio-pilula` | 999px   | chip, botão de ícone, barra de navegação |

**Botão e campo compartilham o raio.** Um campo de busca com moldura de cartão
(1.25rem) fica visivelmente mais arredondado que o botão ao lado — foi
exatamente esse o defeito que originou a classe `.campo-busca`.

### 5.4 Elevação

Três sombras, e o critério é a distância do dedo: `--sombra-baixa` para o que
está encostado, `--sombra` para o que levanta no hover, `--sombra-alta` para o
que flutua (barra de navegação, folha, menu).

### 5.5 Superfícies

```css
.superficie {
  background-color: var(--superficie);
}
.superficie-2 {
  background-color: var(--superficie-2);
}
.superficie-3 {
  background-color: var(--superficie-3);
}

.cartao {
  background-color: var(--superficie);
  border: 1px solid var(--borda);
  border-radius: 1.25rem;
}

.cartao-plano {
  /* sem borda, um degrau acima do fundo */
  background-color: var(--superficie-2);
  border-radius: 1.25rem;
}

/* Campo de busca: parece cartão, mas é CAMPO — o raio é o de campo. */
.campo-busca {
  background-color: var(--superficie);
  border: 1px solid var(--borda);
  border-radius: var(--raio-m);
}
```

### 5.6 Véus

O que fica **entre** uma mídia e um controle por cima dela — o X de remover, o
giro de "enviando", a lupa. Dois passos, e só dois:

```css
--veu-midia: rgb(0 0 0 / 0.5); /* repouso */
--veu-midia-forte: rgb(0 0 0 / 0.7); /* toque */
```

Eram cinco opacidades escritas à mão (35%, 40%, 50%, 60%, 70%) para a mesma
função, e o mesmo botão aparecia mais escuro numa tela do que na outra. Preto
fixo nos dois temas: o véu existe para dar contraste **contra a foto**, não
contra o fundo do app, e a foto não muda com o tema.

Um terceiro véu, de papel diferente, é o do **alvo de soltar arquivo**:

```css
/* Aqui o fundo do app se APAGA para o alvo ficar sozinho no meio. Por isso
   é a própria cor de fundo, e não preto — e o color-mix cobre os dois temas. */
--veu-solta: color-mix(in srgb, var(--fundo) 88%, transparent);
```

**Um alvo de soltar, e só um.** Se a casca do app já cobre a janela inteira
quando um arquivo paira, a tela específica **não** pode ter o seu: os dois
acendem juntos, e o resultado é uma moldura tracejada em volta de tudo, outra em
volta da lista e um cartão no meio das duas. Três desenhos para uma coisa só.
Fica o de cima, e o tracejado é do **cartão**, não da página — moldura
tracejada em volta da janela parece erro de layout.

---

## 6. Movimento

### 6.1 Tokens

```css
--tempo-rapido: 150ms; /* hover, cor, foco, :active */
--tempo-normal: 280ms; /* trilho, folha, troca de painel */
--curva: cubic-bezier(0.22, 1, 0.36, 1); /* tudo que ENTRA */
--curva-saida: cubic-bezier(0.4, 0, 1, 1); /* tudo que SAI */
```

Duas durações e duas curvas dão conta de tudo. Antes havia 0.4s, 0.3s e 0.2s
misturados sem critério.

### 6.2 As distâncias

O vocabulário inteiro do app cabe nesta tabela. Reutilize estes números:

| Gesto                          | Valor                           |
| ------------------------------ | ------------------------------- |
| Cartão sobe no hover           | 2px                             |
| Cartão afunda no clique        | `scale(0.995)`, 80ms            |
| Controle pequeno afunda        | `scale(0.94)`, 80ms             |
| Botão afunda                   | `scale(0.98)`                   |
| Qualquer outro clicável afunda | `scale(0.97)`, 80ms             |
| Tela entra                     | 8px em Y, 180ms                 |
| Tela entra voltando            | −14px em X, 200ms               |
| Bloco encadeado entra          | 10px em Y, 280ms, atraso 48ms×i |
| Painel sai / entra             | ∓56px em X, 200ms / 260ms       |
| Mês do calendário entra        | ±28px em X, 220ms               |
| Ícone cresce no hover          | `scale(1.12)`                   |
| Miniatura cresce no hover      | `scale(1.045)`, 280ms           |

### 6.3 Catálogo de animações

**Entrada e saída**

| Classe                  | Duração | O que faz                                        |
| ----------------------- | ------- | ------------------------------------------------ |
| `.animar-surgir`        | 400ms   | opacidade 0→1 subindo 10px. O padrão de entrada. |
| `.animar-fade-in`       | 200ms   | só opacidade — para fundo escuro de menu         |
| `.animar-surgir-baixo`  | 200ms   | sobe 10px encolhido em 0.95                      |
| `.animar-fade-out`      | 160ms   | some (curva de saída)                            |
| `.brota-do-botao`       | 200ms   | `scale(0.82) translateY(8px)` → normal           |
| `.volta-para-o-botao`   | 150ms   | o inverso, para fechar menu                      |
| `.folha-fechando`       | 160ms   | encolhe e some                                   |
| `.animar-folha`         | 240ms   | sobe 24px com `scale(0.985)`                     |
| `.animar-tela`          | 180ms   | a tela nova entra subindo 8px                    |
| `.animar-tela-voltando` | 200ms   | a tela entra pela esquerda (−14px)               |
| `.revela-ao-rolar`      | 340ms   | seção aparece ao montar perto da viewport        |
| `.dissolvendo`          | 380ms   | `blur(3px) scale(0.96)` + some — descartar       |

**Encadeamento.** Um bloco marcado `data-encadeado` com `style={{ "--i": n }}`
entra 48ms depois do anterior. Quando um contêiner `.animar-tela` **tem** filhos
encadeados, ele mesmo não anima — só os filhos:

```css
.animar-tela:has([data-encadeado]) {
  animation: none;
}
.animar-tela [data-encadeado] {
  animation: entrar-encadeado 280ms var(--curva) both;
  animation-delay: calc(var(--i) * 48ms);
}
```

Na primeira visita do dia a coreografia fica mais lenta e mais teatral: 520ms
com 90ms entre os itens.

**Micro-interações**

| Classe                      | Efeito                                              |
| --------------------------- | --------------------------------------------------- |
| `.animar-pulsar`            | 1.8s, opacidade 1↔0.4 — "a IA está pensando"        |
| `.animar-digitando`         | 1.1s, três pontinhos subindo 4px                    |
| `.ponto-salvando`           | 1.6s, ponto encolhe até 0.72                        |
| `.animar-salvo`             | 420ms, pulso `scale` 0.86→1.1→1                     |
| `.animar-tique`             | 320ms, o check se desenha (`stroke-dashoffset`)     |
| `.animar-destaque-favorito` | 300ms, `scale` até 1.4                              |
| `.respira-contador`         | 620ms, `scale(1.16)` quando o número muda           |
| `tremer`                    | 400ms, ±3–7px — senha errada                        |
| `.esqueleto::after`         | 1.6s, brilho atravessando (`translateX -100%→100%`) |

**Dados entrando** — sempre escalonados por índice, com teto para a lista não
demorar:

| Classe           | Duração | Atraso                     |
| ---------------- | ------- | -------------------------- |
| `.barra-cresce`  | 520ms   | `min(--i, 12) * 55ms`      |
| `.barra-assenta` | 620ms   | `min(--i, 6) * 70ms`       |
| `.dia-acende`    | 260ms   | `min(--semana, 26) * 18ms` |
| `.tag-caindo`    | 320ms   | `min(--i, 10) * 55ms`      |

**Rolagem que dirige a animação.** O cabeçalho encolhe conforme a página rola —
sem JavaScript, e só onde o navegador suporta:

```css
@supports (animation-timeline: scroll()) {
  @media (prefers-reduced-motion: no-preference) {
    .cabecalho-encolhe {
      animation: encolher-cabecalho linear both;
      animation-timeline: scroll();
      animation-range: 0 220px; /* detalhe da entrada usa 0 180px */
    }
  }
}
```

**View Transitions.** Três usos, todos com queda limpa quando a API não existe:

| Efeito                | Duração | Como                                                     |
| --------------------- | ------- | -------------------------------------------------------- |
| Onda ao trocar o tema | 520ms   | `clip-path: circle(0% → raio)` a partir do botão clicado |
| Foto ampliando        | 320ms   | `view-transition-name` na miniatura e na foto grande     |
| Botão + → tela nova   | 420ms   | `view-transition-name: nova-fab` nos dois lados          |

O raio da onda é `Math.hypot(max(x, W−x), max(y, H−y))` — a diagonal até o canto
mais distante do clique. E enquanto uma View Transition roda, a animação de rota
é desligada (`html[data-transicao] .animar-tela { animation: none }`), senão as
duas brigam.

**Cuidado com `backdrop-filter`:** um elemento com `view-transition-name` é
fotografado e promovido para a camada da transição, e lá dentro o
`backdrop-filter` não tem o que amostrar. Foi isso que fazia o desfoque da barra
de navegação sumir "de vez em quando". A barra não tem nome de transição, e é de
propósito.

### 6.4 Movimento em JavaScript

O que o CSS não alcança passa por um guarda único:

```ts
export function querMenosMovimento() {
  return (
    typeof window !== "undefined" && window.matchMedia("(prefers-reduced-motion: reduce)").matches
  );
}
```

**Mola própria**, no lugar de uma biblioteca de animação (rigidez `0.14`,
amortecimento `0.78`, para quando `|destino − valor| ≤ 0.4` e `|velocidade| ≤
0.4`). Sob movimento reduzido ela salta direto para o destino. É o retorno
elástico do puxar-para-atualizar — e é tudo o que uma biblioteca de spring
faria ali.

**Confete** (import dinâmico): 40 partículas, espalhamento 60, origem `{ x: 0.8,
y: 0.3 }`, cores da categoria ou o dourado do favorito. Não roda sob movimento
reduzido.

**Vibração**, cinco padrões, falhando em silêncio onde não existe:

| Nome      | ms           |
| --------- | ------------ |
| `leve`    | 8            |
| `medio`   | 18           |
| `sucesso` | [10, 40, 14] |
| `aviso`   | [16, 32, 16] |
| `forte`   | [12, 48, 12] |

**Contagem animada**: 900ms com `easeOutCubic` em `requestAnimationFrame` (nunca
`setInterval`), disparada por `IntersectionObserver` com `threshold: 0.2`.

**FLIP para reacomodar**: filhos marcados `data-reacomodar` têm a posição medida
antes e depois; a inversão é aplicada sem transição e a volta em
`--tempo-normal`, depois de dois `requestAnimationFrame`. Movimento menor que
0.5px é ignorado.

### 6.5 Movimento reduzido

Duas frentes. No CSS, o interruptor geral:

```css
@media (prefers-reduced-motion: reduce) {
  .interativo:hover,
  .interativo:active,
  .pressionavel:active,
  .zoom-suave:hover img {
    transform: none !important;
  }

  *,
  *::before,
  *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

No JavaScript, `querMenosMovimento()` desliga: View Transitions, suavização de
rolagem, confete, mola, troca de painel com fantasma, gesto de voltar,
puxar-para-atualizar, coreografia de abertura, contagem animada, FLIP, parallax
e o morph do gráfico.

**Regra:** toda animação em JS pergunta antes. Se você escreveu `setTimeout` ou
`requestAnimationFrame` para mover alguma coisa, faltou a pergunta.

---

## 7. Interação

### 7.1 O piso

Todo controle responde ao ponteiro e ao clique, **mesmo o que ninguém lembrou de
estilizar**. Antes isso dependia de cada arquivo pôr o seu `hover:` à mão, e
sobrava: linha de lista, aba, item de menu, rótulo de arquivo — coisas clicáveis
que não davam sinal nenhum de que eram.

```css
@media (prefers-reduced-motion: no-preference) {
  :is(button, [role="button"], summary, label[for], a[href], .pressionavel, .interativo):not(
    :disabled,
    [aria-disabled="true"]
  ) {
    transition-property:
      transform, opacity, background-color, background-image, color, border-color, box-shadow;
  }

  @media (hover: hover) {
    :is(button, [role="button"], summary, label[for], a[href], .pressionavel, .interativo):not(
        :disabled,
        [aria-disabled="true"]
      ):hover {
      background-image: linear-gradient(var(--veu-hover), var(--veu-hover));
    }
  }

  :is(...):active {
    background-image: linear-gradient(var(--veu-ativo), var(--veu-ativo));
  }

  /* Quem não tem gesto próprio afunda 3%. */
  :is(button, [role="button"], summary, label[for], a[href]):not(
      :disabled,
      [aria-disabled="true"],
      .interativo,
      .pressionavel,
      .cartao
    ):active {
    transform: scale(0.97);
    transition-duration: 80ms;
  }
}
```

É um **piso**, não uma regra: quem já tem gesto próprio fica de fora pelo
`:not()`.

`@media (hover: hover)` em toda regra de hover — no celular o hover gruda depois
do toque e o elemento fica "aceso" até você tocar em outro lugar.

**Link dentro de texto corrido não recebe véu.** Ele não é um controle com caixa
própria: pintar um retângulo atrás de uma palavra mancha a linha inteira. O que
ele faz é engrossar o sublinhado.

### 7.2 Os dois utilitários de toque

```css
/* Coisa grande que se clica: cartão, linha de lista. */
.interativo {
  transition:
    transform,
    box-shadow,
    border-color,
    background-color var(--tempo-rapido) var(--curva);
}
@media (hover: hover) {
  .interativo:hover {
    transform: translateY(-2px);
    box-shadow: var(--sombra);
    border-color: var(--borda-forte);
  }
}
.interativo:active {
  transform: translateY(0) scale(0.995);
  box-shadow: none;
  transition-duration: 80ms;
}

/* Controle pequeno: botão, chip, ícone. NÃO levanta — numa barra de botões
   um deles subindo fica torto. Só encolhe. */
.pressionavel {
  transition:
    transform,
    background-color,
    color,
    opacity var(--tempo-rapido) var(--curva);
}
.pressionavel:active:not(:disabled) {
  transform: scale(0.94);
  transition-duration: 80ms;
}
```

Um pixel de elevação é de propósito: o suficiente para o olho registrar, pouco
para não parecer que a página está pulando.

**Quando o filho pressiona o pai.** Um cabeçalho clicável dentro de um cartão
deve encolher o **cartão inteiro**, não a si mesmo — senão a área cinza do
toque fica menor que o cartão e parece recortada. Isso se escreve com `:has()`:

```css
/* O botão em si não anima… */
.pasta > h2 > button.pasta-cabecalho:active {
  transform: none;
}
/* …quem anima é o cartão que o contém. */
@media (prefers-reduced-motion: no-preference) {
  .pasta:has(> h2 > button.pasta-cabecalho:active) {
    transform: scale(0.99);
    transition-duration: 80ms;
  }
}
```

E aqui vai uma armadilha que custou duas tentativas: **`:is(button, [role="button"], …)` pesa como
classe**, porque a especificidade de `:is()` é a do argumento mais forte — e um
seletor de atributo vale uma classe. Um `button:active { transform: none }`
escrito com seletor de elemento (0,0,1) perde para o piso genérico e é ignorado
**em silêncio**. Se uma regra de anulação "não está pegando", conte a
especificidade dos dois lados antes de mexer em qualquer outra coisa; e prefira
anular pelo caminho completo (`.pasta > h2 > button.x:active`), que é explícito
e não depende de sorte.

### 7.3 Ícones que reagem

Um ícone sozinho dentro de botão ou link cresce 12% no hover e encolhe para 90%
no clique. Além disso, alguns **gestos nomeados** por `aria-label` ou
`data-icone` — o ícone faz o que a ação faz:

| Ação              | Gesto                              |
| ----------------- | ---------------------------------- |
| Apagar            | a tampa da lixeira levanta (420ms) |
| Lembrete          | o sino balança ±14° (620ms)        |
| Nova / Criar      | `rotate(12deg) scale(1.06)`        |
| Voltar / Avançar  | desliza ∓2px                       |
| Restaurar         | `rotate(-10deg) scale(1.06)`       |
| Desfazer          | `rotate(10deg) scale(1.06)`        |
| Buscar            | `scale(1.1) rotate(-6deg)`         |
| Fechar            | `rotate(12deg)`                    |
| Enviar / Exportar | `translate(2px, -2px) scale(1.06)` |
| Gravar / Ouvir    | pulsa até `scale(1.22)` (520ms)    |
| Favoritar         | `rotate(10deg) scale(1.1)`         |
| Editar            | rabisca — ±1.5px e ±8° (460ms)     |

As rotações são pequenas de propósito: 10–12°, não 20. Passou disso vira
desenho animado.

### 7.4 Foco

```css
:focus-visible {
  outline: 2px solid var(--color-acento-500);
  outline-offset: 2px;
  border-radius: 4px;
}
```

Só `:focus-visible`, para o anel não aparecer em clique de mouse.

Trinta e dois controles usavam `outline-none` — em Tailwind isso vence o
`:focus-visible` global, e o campo ficava sem nenhum sinal para quem navega por
teclado. Para eles existem duas utilidades, que devolvem o anel por `box-shadow`
(propriedade diferente, então não disputa com o `outline-none`):

```css
@utility foco-anel {
  &:focus-visible {
    box-shadow: 0 0 0 3px var(--anel-foco);
  }
}
@utility foco-anel-dentro {
  &:focus-within {
    box-shadow: 0 0 0 3px var(--anel-foco);
  }
}
```

`foco-anel-dentro` é para invólucro: campo com ícone ou botão dentro tem a
moldura no `<span>` que envolve tudo, e o anel precisa cercar o conjunto.

**Regra inviolável: `outline-none` sem `foco-anel` junto é bug.**

### 7.5 Cursor e alvo

```css
html {
  -webkit-tap-highlight-color: transparent;
}
button:not(:disabled),
[role="button"],
label:has(> input[type="file"]),
summary,
a[href] {
  cursor: pointer;
}
button:disabled,
[aria-disabled="true"] {
  cursor: not-allowed;
}
```

Alvo de toque mínimo: **44px**. Onde o ícone precisa ser maior que o alvo
confortável, cresce o ícone e não o botão.

---

## 8. Componentes

Todos moram em `components/ui/`. A regra: **antes de escrever uma classe,
procure o componente.**

### 8.0 A regra número um, e por que ela é a número um

Esta é a única regra deste documento que, sozinha, decide se o app continua
parecendo um produto depois de seis meses. Ela é violada sempre da mesma forma:
não por rebeldia, mas porque escrever `className="rounded-xl bg-… px-4 py-2"`
custa dez segundos e procurar o componente custa trinta.

O preço, medido no app de referência:

| O que foi copiado à mão | Cópias | O que deu errado                                    |
| ----------------------- | -----: | --------------------------------------------------- |
| Preenchimento de acento |     62 | cada uma com o seu padding, raio e hover            |
| Botão "← Voltar"        |      8 | nenhuma com anel de foco; 6 ficaram sem o indicador |
| Tamanho de tipografia   |    209 | `text-[13.5px]` e afins, sem escala                 |
| Raio de canto           |    244 | quatro raios para o mesmo papel                     |
| Véu sobre foto          |      5 | o mesmo botão mais escuro numa tela que na outra    |
| Estado vazio            |      3 | dois na MESMA tela, com linguagens diferentes       |
| Ladrilho de ícone       |      6 | três tamanhos e duas formas para o mesmo papel      |
| Duração de transição    |      5 | 300 ms, 320 ms e 500 ms para gestos irmãos          |

Repare no padrão: **a cópia nunca dói na hora**. Ela dói no dia em que uma
mudança precisa acontecer nos oito lugares e chega em dois — e o que sobra não é
"inconsistência visual", é defeito funcional (o botão sem foco, o indicador que
não acende).

**Como detectar, sem depender de disciplina.** Um `grep` periódico responde:

```bash
# quantas vezes o mesmo bloco de classes aparece?
grep -rno 'rounded-\[[0-9]' src/ --include=*.tsx     # raio escrito à mão
grep -rno 'duration-[0-9]\+' src/ --include=*.tsx    # duração fora do token
grep -rno 'text-\[[0-9.]*px\]' src/ --include=*.tsx   # tipografia arbitrária
```

Três ocorrências do mesmo desenho é o limite: na terceira, extraia. E toda vez
que extrair, some a variação — se as três cópias divergiam, uma delas estava
errada, e agora dá para saber qual.

### 8.1 Botão

```tsx
<Botao variante="primario|superficie|fantasma|perigo" tamanho="p|m|g" largo carregando />
<BotaoLink href="…" />   // mesma cara, mas navega
<BotaoIcone rotulo="…" /> // quadrado, rótulo acessível OBRIGATÓRIO
```

```
base:      inline-flex shrink-0 cursor-pointer items-center justify-center
           rounded-[var(--raio-m)] font-semibold
           transition-[background-color,opacity,transform] duration-[var(--tempo-rapido)]
           active:scale-[0.98] disabled:opacity-40 disabled:cursor-not-allowed

primario:   bg-[var(--acento-solido)] text-[var(--acento-solido-texto)]
superficie: superficie-2 text-[var(--texto)]
fantasma:   text-[var(--texto-2)] hover:text-[var(--texto)]
perigo:     bg-[var(--perigo)] text-[var(--perigo-texto)]

p: h-9  px-3 txt-sm  gap-1.5
m: h-11 px-4 text-sm gap-2
g: h-14 px-5 text-sm gap-2
```

**Nenhuma variante define hover** — quem faz isso é o véu global. Aqui fica só o
que a variante **é**, não como ela reage. A exceção é o fantasma, que muda a cor
do texto: parado ele é quase invisível, e o véu sozinho não o traria para a
frente.

`BotaoIcone` é `rounded-[var(--raio-pilula)]`, lado 36/44/56px conforme o
tamanho, e exige `rotulo` (vira `aria-label` e `title`).

`BotaoLink` existe para não virar `<a>` dentro de `<button>`. Para navegar de
dentro de um cartão clicável, use `router.push` num `<button>` — **nunca aninhe
`<a>` dentro de `<a>`**.

### 8.2 Campo

```tsx
<Campo rotulo dica erro>{children}</Campo>   // só a moldura
<CampoTexto rotulo dica erro adorno … />
<AreaTexto rotulo dica erro rows … />
```

```
w-full rounded-[var(--raio-m)] bg-[var(--superficie-2)] px-4 py-2.5 txt-md
outline-none foco-anel transition-colors duration-[var(--tempo-rapido)]
placeholder:text-[var(--texto-3)] disabled:opacity-50
```

- Moldura: `flex flex-col gap-1.5` → `.rotulo`, dica em `.t3 .txt-xs`, campo,
  erro em `role="alert"` com `--acento-texto`.
- Erro adiciona `ring-1 ring-[var(--perigo)]` e `aria-invalid`.
- **Com adorno** (botão dentro do campo), a moldura sai do `<input>` e vai para
  o invólucro com `foco-anel-dentro` — senão o foco desenha a borda só em volta
  do texto e deixa o botão de fora.
- O nome é `CampoTexto` e não `Entrada` porque "entrada" é o substantivo central
  do domínio. **Não dê a um componente de formulário o nome de uma entidade do
  produto.**

### 8.3 Chip

Tag, categoria, pessoa e filtro são **a mesma peça**.

```tsx
<Chip variante="neutro|acento|solido|contorno" cor={corDaCategoria} title>…</Chip>
<ChipBotao ativo aoClicar cor desabilitado>…</ChipBotao>
```

```
base: inline-flex items-center gap-1.5 rounded-[var(--raio-pilula)]
      px-3 py-2 txt-xs font-semibold leading-none

neutro:   superficie-2 t3
acento:   fundo color-mix(acento, --chip-tinta) + text-[var(--acento-texto)]
solido:   bg-[var(--texto)] text-[var(--fundo)]
contorno: border + bg-[var(--superficie)] + t2
com cor:  fundo color-mix(cor, --chip-tinta) + a própria cor como letra
```

Um padding só, o do chip clicável. O chip estático era mais apertado (`py-1`
contra `py-2`) e a mesma linha misturava duas alturas de pílula sem que nada
justificasse.

O fundo tingido é **5%** (`--chip-tinta`) — baixo de propósito. Em 15% o chip
virava um bloco colorido e competia com a palavra que carrega; o fundo existe
para agrupar, não para gritar. Quem dá a cor é a letra e o ícone.

`ChipBotao` acrescenta `.pressionavel` e `aria-pressed`. Ativo com cor: fundo
sólido na cor e letra em `--sobre-emocao`.

### 8.4 Folha (modal)

```tsx
<Folha aberto aoFechar titulo descricao rodape largura="max-w-md">
  …
</Folha>
```

**Um componente, dois formatos**: folha subindo no celular (`items-end`,
`rounded-t-[var(--raio-cartao)]`), modal centrado a partir de `sm`
(`sm:items-center`, `sm:rounded-[var(--raio-cartao)]`).

| Peça             | Valor                                                           |
| ---------------- | --------------------------------------------------------------- |
| Camada           | `createPortal` no `body`, `z-[80]`                              |
| Fundo escuro     | `bg-black/40 backdrop-blur-[2px]`, entra com `.animar-surgir`   |
| Caixa            | `max-h-[85dvh]`, `--superficie`, `--sombra-alta`                |
| Cabeçalho        | `border-b px-5 py-4` — `h2.display text-lg` + fechar (`X` 16px) |
| Corpo            | `overflow-y-auto px-5 py-4`, com `data-lenis-prevent`           |
| Rodapé           | `border-t px-5 py-4`                                            |
| Alça (só mobile) | `h-1 w-10 rounded-full bg-[var(--borda-forte)]`                 |
| Saída            | fica montada 160ms depois de fechar, para a animação rodar      |

Resolve num lugar só o que cada diálogo resolvia (ou esquecia) por conta
própria: `role="dialog"`, `aria-modal`, Esc, travar a rolagem do fundo, prender
o Tab dentro e **devolver o foco a quem abriu**.

Detalhe que custou caro: o `aoFechar` vai por `ref`, e o efeito depende só de
`aberto`. Com `aoFechar` nas dependências, uma arrow inline em quem chama (o
caso normal) recriava o efeito a cada render, e o "foco de quem abriu" era
sobrescrito pelo foco do momento.

### 8.5 Trilho (indicador deslizante)

```tsx
<Trilho ativo={chave} className="superficie-2 rounded-xl" instantaneo? estilo? />
```

O fundo do item ativo **viaja** de um item para o outro em vez de trocar de
lugar num quadro. É o deslize que conta de onde para onde se foi.

- O trilho é **irmão** dos itens, não filho: se fosse filho, mudar de item
  significaria desmontá-lo e remontá-lo, e não haveria de onde animar.
- Quem usa marca cada item com `data-trilho="chave"` e põe `relative` no
  contêiner; os itens sobem para `z-10`.
- Mede por `offsetLeft/Top/Width/Height` (relativo ao contêiner, não à janela —
  a barra do celular é fixa e coordenadas de viewport a deslocariam a cada
  rolagem), com `ResizeObserver` no pai e nos filhos.
- Transição: `transform, width, height, background-color` em `--tempo-normal`.
- `instantaneo` desliga o deslize, para onde outro movimento já conta a história.

### 8.6 TrocaPainel

```tsx
<TrocaPainel chave={aba} duracaoMs={280}>
  {conteudo}
</TrocaPainel>
```

Saída e entrada ao mesmo tempo: o painel que sai desliza 56px para a esquerda
(200ms); o que entra chega de 56px à direita (260ms, com 90ms de atraso). É o
que faz a troca parecer um deslize, e não um fundo viajando de um rótulo para o
outro.

O painel que sai é um **fantasma** renderizado em `useLayoutEffect` — precisa ser
pintado no mesmo quadro em que o conteúdo novo monta, senão o olho vê um frame
só com a aba nova. Sob movimento reduzido, a troca é instantânea.

Quando há `TrocaPainel`, o `Trilho` das abas vai `instantaneo`: o olho segue o
conteúdo, e a pílula viajando competia com ele.

### 8.7 Chave (liga/desliga)

Trilho `h-7 w-12` pílula; polegar `h-6 w-6` que anda `translate-x-5`. Ligado:
`--acento-solido`. Desligado: `--superficie-3`. `role="switch"`, `aria-checked`,
`foco-anel`. Estado ocupado mostra um `Loader2` de 14px sobre o polegar.

### 8.8 Nota

```tsx
<Nota tom="info|aviso|erro" semIcone>
  …
</Nota>
```

`rounded-[var(--raio-m)] px-3.5 py-3 txt-xs leading-relaxed`, ícone de 16px
alinhado ao topo.

| Tom   | Fundo            | Cor              | Extra          |
| ----- | ---------------- | ---------------- | -------------- |
| info  | `--superficie-2` | `--texto-2`      | —              |
| aviso | `--aviso-tenue`  | `--aviso`        | —              |
| erro  | `--perigo-tenue` | `--acento-texto` | `role="alert"` |

Existia repetido em quatro telas, e cada cópia pintava o fundo com o acento —
inclusive as de erro. Dizer "não consegui salvar" com a mesma cor do botão
"Salvar" não avisa ninguém.

### 8.9 Sanfona (`<details>`)

Use `<details>/<summary>` para pergunta-e-resposta: ele já traz teclado,
`aria-expanded` e busca-na-página de graça. Duas linhas o colocam no sistema:

```css
summary::-webkit-details-marker { display: none; }  /* o triângulo do navegador */
```

…e o indicador vira um ícone de 16px que gira 180° com `group-open:rotate-180`.
Deixar o marcador nativo é a única parte do app com um glifo que não é do
sistema de ícones — e ele muda de desenho a cada navegador.

### 8.10 Esqueleto

```tsx
<Esqueleto className="h-8 w-40" arredondado="var(--raio-m)" />
<EsqueletoTexto linhas={4} />
<EsqueletoCartao /> <EsqueletoLista itens={3} />
```

```css
.esqueleto {
  background-color: var(--superficie-2);
  position: relative;
  overflow: hidden;
}
.esqueleto::after {
  content: "";
  position: absolute;
  inset: 0;
  background: linear-gradient(
    90deg,
    transparent,
    color-mix(in srgb, var(--texto) 6%, transparent),
    transparent
  );
  animation: brilho 1.6s infinite; /* translateX(-100%) → 100% */
}
```

**A regra é espelhar o formato do que vai chegar.** Um retângulo genérico faz a
tela pular quando o conteúdo real entra — o esqueleto do cartão tem metadado,
título, duas linhas de prévia e duas pílulas, nas mesmas medidas do cartão.

### 8.11 Estado vazio

```tsx
<EstadoVazio icone={<Users />} titulo="…" descricao="…" acao={<BotaoLink …/>} />
```

`flex flex-col items-center px-6 py-14 text-center`; ícone 36px em `.t3` com
entrada flutuante suave; título `.display text-xl`; descrição `.t3 .txt-sm
max-w-xs`; ação 20px abaixo.

**Diz o que houve e oferece a saída.** Vazio sem botão é beco sem saída.

### 8.12 Botão de salvar

Três estados numa peça só: parado → salvando (encolhe para `w-14` círculo em
320ms, com o rótulo sumindo antes) → salvo (pulso `.animar-salvo` de 420ms e o
check se desenhando em `.animar-tique`).

### 8.13 Avisos (toast)

`useAviso()` com `.info() .sucesso() .erro()`, sobre `z-70`.

**Nunca use `alert()`.**

### 8.14 Cartão

Não é componente de biblioteca, é padrão: `.cartao` + `.interativo` + `p-5`,
com esta ordem interna:

```
metadado (t3 txt-xs)      — hora, contagem, marcadores
título   (.display txt-xl leading-[1.1])
prévia   (t2 line-clamp-3 txt-sm leading-[1.6])
rodapé   (flex flex-wrap gap-1.5) — chips
```

O cartão tingido por categoria usa `::before` com um `radial-gradient` da cor em
22%, que aparece no hover junto de uma borda em 45% da mesma cor — a cor entra
como luz, não como preenchimento.

### 8.15 Botão de voltar

```tsx
<BotaoVoltar aoVoltar={() => router.back()} trocaDeRota />
```

Parece pequeno demais para virar componente. Não é — e a razão não é estética.

Num app com telas cheias (criar, editar, ler, responder), o "← Voltar" acaba
copiado em seis, oito lugares, sempre a mesma linha de cem caracteres de classe.
Enquanto foram cópias, duas coisas aconteceram: **nenhuma** tinha anel de foco,
e quando o indicador de rota passou a acender no clique, **só duas das oito**
ganharam a chamada — as outras seis continuavam parecendo que não tinham sido
tocadas enquanto a tela de destino montava.

`trocaDeRota` existe porque voltar nem sempre navega: às vezes fecha um painel
ou volta uma etapa dentro da mesma tela. Acender o indicador nesses casos o
deixa pendurado esperando uma navegação que não vem.

### 8.16 Só com internet / só com o recurso

```tsx
<SoOnline motivo="Isto precisa de internet">{botao}</SoOnline>
```

**O que exige rede não some: fica na tela, apagado e inerte.** Sumir faz a
pessoa procurar o botão e duvidar da própria memória; apagado ela entende que
volta quando a internet voltar.

Duas exigências de implementação:

- O atributo **`inert`** é o que de fato desliga o conteúdo — clique, foco,
  teclado e leitor de tela. `pointer-events: none` sozinho deixa o link
  alcançável por Tab, o que é pior que nada.
- Opacidade de **35%**, e um `title`/`aria-label` dizendo o motivo.

O mesmo invólucro serve para recurso de plano pago (`SoComIa` e afins): mesma
gramática, motivo diferente.

### 8.17 Seleção múltipla

Um padrão, usado igual em toda lista que permite apagar ou mover em lote:

1. Botão **"Selecionar"** no cabeçalho da tela (vira "Cancelar" quando ativo).
2. Com a seleção ligada, **o cartão inteiro é a caixa de marcar** — clicar não
   abre mais o item. Abrir no meio de uma seleção é perder o que já foi marcado.
3. Uma **marca redonda** à esquerda do cartão e um anel de acento em volta do
   cartão marcado. Redonda porque, numa lista que já tem foto, ícone e chip, um
   quadrado vazio vira mais um enfeite.
4. As ações do cartão **somem** durante a seleção: quem manda é a barra.
5. Barra **grudada embaixo**, com "Selecionar todos", a contagem e as ações. No
   fim de uma lista longa, um botão no rodapé custaria a rolagem inteira de
   volta. Se ela for `fixed` (e não `sticky`), ela é a `<BarraFlutuante>` de
   §8.18 — com portal —, senão a animação de entrada da tela a prende no fim do
   conteúdo.

Em lote, aja **em série** e conte o que falhou, em vez de morrer no primeiro
erro: dez escritas em paralelo na mesma coleção produzem escrita por cima de
escrita.

### 8.18 Barra flutuante — e por que ela precisa de portal

A barra que aparece acima da navegação (o total da seleção, um "desfazer", um
aviso de rascunho) parece um `position: fixed` e nada mais. Não é.

> **Um ancestral com `transform` animado vira bloco de contenção, e o `fixed`
> passa a se medir por ele.**

Como toda tela deste sistema entra animando (`.animar-tela`) e os blocos entram
encadeados (`transform` em `@keyframes`), **qualquer** barra fixa escrita dentro
da página cai nessa armadilha. O sintoma engana: a barra aparece no **fim do
conteúdo** em vez do rodapé da janela, e some ao rolar — e ninguém suspeita da
animação, porque o CSS da barra está certo. Pior: some quando a animação
termina? Não — a animação com `fill-mode: both` continua "aplicando" o valor
final, e o bloco de contenção continua de pé.

A correção é uma linha de arquitetura, não de CSS:

```tsx
return createPortal(<div className="fixed …">{children}</div>, document.body);
```

No `body` não há ancestral que a capture. E, já que ela é um componente:

| Peça               | Valor                                                                     |
| ------------------ | ------------------------------------------------------------------------- |
| Camada             | `--z-avisos` — acima do conteúdo, abaixo de folha e visor                 |
| Celular            | `inset-x-4`, `bottom: calc(var(--altura-nav) + safe-area + 0.5rem)`       |
| Desktop            | canto inferior direito, com respiro para não colidir com a bolha do chat  |
| Entrada            | `.animar-surgir`                                                          |

**Vale para tudo que é `fixed` dentro de uma tela animada**: menu que brota de
um botão, tooltip posicionado à mão, folha escrita sem o componente. Se algo
"fixo" apareceu no lugar errado, o primeiro suspeito é o ancestral animado.

### 8.19 Estado com prazo: data + selo

Quando um cartão precisa dizer "até quando" e "isto se renova sozinho?", **não
escreva uma frase**. Uma frase diz as duas coisas ao mesmo tempo e não deixa
nenhuma visível de relance:

> ~~Pago até 5 de outubro. Não renova sozinho — quando acabar, você escolhe de
> novo.~~

Vire duas informações, na mesma linha: à esquerda, ícone de calendário + a data;
à direita, um **selo** com a resposta binária ("Renovação automática" / "Sem
renovação automática"). Perto do fim do prazo, acrescente "· acaba em N dias" na
cor de aviso.

E **ação só quando ela faz sentido**: um botão "Renovar" ao lado de um prazo com
meses pela frente é um convite a pagar de novo por algo que a pessoa já tem.
Mostre-o na janela em que ele é útil (aqui: os últimos 15 dias).

---

## 9. Moldura e navegação

### 9.1 A casca

```tsx
<html className={fontes} data-scroll-behavior="smooth">
  <head>{/* preconnect + dns-prefetch do backend */}</head>
  <body className="antialiased">
    <ProvedorTema /> {/* script antes da primeira pintura */}
    <RolagemSuave /> {/* Lenis */}
    <ProvedorAvisos>
      <div id="app" className="lg:flex lg:min-h-dvh">
        <Navegacao />
        <AreaPrincipal>{children}</AreaPrincipal>
      </div>
    </ProvedorAvisos>
    {/* camadas: tranca, progresso de rota, gesto de voltar, vigia de erros,
        menu ⌘K, atalhos, sincronizador, service worker, métricas */}
  </body>
</html>
```

### 9.2 A página

```css
.pagina {
  width: 100%;
  max-width: 72rem;
  margin-inline: auto;
  padding-inline: 1.25rem;
  padding-block: 1.75rem;
}
@media (min-width: 1024px) {
  .pagina {
    padding-inline: 2.5rem;
    padding-block: 3rem;
  }
}
```

Todas as telas usam a mesma largura máxima e as mesmas margens. Antes cada uma
escolhia a sua (de `max-w-3xl` a `max-w-6xl`) e o conteúdo "pulava" de largura ao
trocar de aba.

### 9.3 `<main>`

- `key={caminho}` — é o que faz a animação de entrada rodar de novo a cada tela.
  Sem ela o React reaproveita o mesmo `<main>`, a animação já terminou e a troca
  acontece sem transição nenhuma.
- Classe `animar-tela` ou `animar-tela-voltando`, conforme a direção. Voltar
  desfaz o caminho: a tela entra **pela esquerda**, do lado para onde ela tinha
  ido. O App Router não anima a saída, mas inverter o lado de entrada já conta a
  mesma história. A direção vem de um `popstate` guardado numa bandeira, lida com
  `useMemo` no caminho (ler a cada render a perderia).
- Reserva de rodapé para a barra flutuante:
  `pb-[calc(var(--altura-nav)+env(safe-area-inset-bottom))] lg:pb-0`.

### 9.4 Telas imersivas

Escrever, conversar, entrar e a retrospectiva **não têm barra de navegação** —
nem a folga que ela exige, nem a animação de entrada. Ali a tela abre com o
cursor já no campo, e qualquer movimento atrapalha quem começou a digitar. Elas
usam `flex h-[100svh] flex-col overflow-hidden`.

Defina isso como uma função só (`telaImersiva(caminho)`) e consulte-a nos dois
lugares que precisam saber (a navegação e o `<main>`).

### 9.5 Navegação — desktop

Coluna fixa à esquerda a partir de `lg`:

```
aside: sticky top-0 z-40 h-dvh w-64 xl:w-72 shrink-0 border-r px-4 py-6
       hidden lg:flex flex-col
```

- Logo no topo (30px), 32px de folga abaixo.
- `<nav className="relative flex flex-col gap-1">` com um `<Trilho>` deslizante
  ao fundo (`superficie-2 rounded-xl`).
- Item: `relative z-10 flex items-center gap-3 rounded-xl px-2.5 py-2.5 text-sm
font-medium`; ativo em `--texto`, inativo em `.t2`. Ícone de 18px,
  `strokeWidth 2`, **acento só no ativo**.
- Ações principais logo abaixo da lista: a primária larga, a secundária em
  variante superfície.
- Versão do app no rodapé, em `.t3 .txt-xs`.

### 9.6 Navegação — celular

Pílula flutuante no rodapé, não barra colada:

```
fixed inset-x-0 bottom-0 z-40 flex justify-center px-4 pb-3 pb-segura lg:hidden
 └ nav: flex items-center gap-1 rounded-full border p-1.5
        shadow-[var(--sombra-alta)] backdrop-blur-xl
        background: color-mix(in srgb, var(--superficie) 82%, transparent)
```

- **Só os dois primeiros destinos** ficam visíveis; o resto vai para um menu que
  brota do botão `⌄` (`.brota-do-botao`, 200ms) e volta para ele ao fechar
  (`.volta-para-o-botao`, 150ms — o mesmo número no CSS e no `setTimeout` que
  adia o desmonte).
- **O rótulo só aparece no item ativo**, para a pílula não inchar. O `Trilho`
  acompanha a mudança de largura via `ResizeObserver`.
- Separador de 1px, e a ação principal como botão de ícone de 44px com o ícone
  em 30px — é o gesto mais disputado da barra; cresce o ícone, não o alvo.
- A barra **não some ao rolar**. Ela é o chão do app.

```css
/* 44px do item + 6px de padding em cima e embaixo + 12px de folga. O <main>
   reserva exatamente isso, em vez de uma faixa fixa bem maior que a barra. */
--altura-nav: 4.75rem;
.pt-segura {
  padding-top: env(safe-area-inset-top);
}
.pb-segura {
  padding-bottom: env(safe-area-inset-bottom);
}
```

### 9.7 Cabeçalho de tela

No celular, uma faixa com a marca e as ações; no desktop ela some (a marca já
está na coluna).

```tsx
<CabecalhoMobile>{acoes}</CabecalhoMobile>   {/* mb-6 flex justify-between lg:hidden */}
<header className="mb-8">
  <p className="rotulo mb-2.5">contexto em caixa alta</p>
  <h1 className="display txt-display">título em caixa baixa</h1>
</header>
```

O rótulo pode ser dinâmico e dar o estado ("47 entradas até aqui"). O título é
sempre curto e em caixa baixa.

### 9.8 Progresso de rota

Barra de 2px no topo, e uma barra **mentirosa** de propósito: espera 180ms antes
de aparecer (navegação rápida não pisca nada), avança a cada 220ms por
`p + (0.9 − p) × 0.12` — nunca chega a 90% sozinha — e completa quando a rota
chega, sumindo em 280ms.

### 9.9 Camadas (z-index)

**Tokens, não números soltos.** Enquanto foram números escritos em cada
arquivo, a conta de quem fica na frente de quem só existia na cabeça de quem
tinha escrito por último — e foi assim que uma confirmação de saída nasceu
_atrás_ da tela de bloqueio, invisível, e uma foto ampliada precisou de
`z-[9999]` (o número que se escolhe quando não existe escala).

```css
--z-avisos: 70; /* toast: acima de tudo que é conteúdo */
--z-folha: 80; /* modal / folha */
--z-visor: 90; /* mídia em tela cheia, acima da folha que a abriu */
--z-tranca: 100; /* a tela de bloqueio cobre o app inteiro */
--z-folha-tranca: 110; /* e o que a PRÓPRIA tranca abre fica acima dela */
```

| Camada                   | z                        |
| ------------------------ | ------------------------ |
| Conteúdo                 | 0                        |
| Fundo do menu do celular | 30                       |
| Navegação                | 40                       |
| Dropdown de busca        | 50                       |
| Avisos                   | `--z-avisos` (70)        |
| Folha / modal            | `--z-folha` (80)         |
| Mídia em tela cheia      | `--z-visor` (90)         |
| Tela de bloqueio         | `--z-tranca` (100)       |
| Folha aberta pela tranca | `--z-folha-tranca` (110) |

Regra: **camada nova entra na escala, com nome**. Se você precisou de um número
que não está aqui, ou falta um degrau (dê nome a ele) ou o problema não é de
camada.

### 9.10 Pontos de quebra

Mobile-first. Só `lg` muda a estrutura.

| Prefixo | ≥      | O que muda                                                       |
| ------- | ------ | ---------------------------------------------------------------- |
| (base)  | —      | pílula no rodapé, cabeçalho com marca, listas em uma coluna      |
| `sm`    | 640px  | duas colunas em mosaico, rótulos completos nos filtros           |
| `md`    | 768px  | fileira horizontal de cartões com encaixe                        |
| `lg`    | 1024px | **coluna lateral no lugar da pílula**, `.pagina` com folga maior |
| `xl`    | 1280px | coluna lateral mais larga, três colunas em mosaico               |

### 9.11 Ícones

| Tamanho  | Onde                                             |
| -------- | ------------------------------------------------ |
| 12px     | metadado dentro de cartão                        |
| 14px     | seta de dropdown, marcadores secundários         |
| **16px** | **o padrão** — botão, campo, nota                |
| 18–19px  | navegação (ajuste fino)                          |
| 20px     | botão de ícone grande, busca do menu de comandos |
| 36px     | estado vazio                                     |

`strokeWidth`: 2 é o padrão; 2.1 na navegação; 2.5–2.6 em "criar" e "enviar";
1.2–1.8 em ícone decorativo grande (quanto maior o ícone, mais fino o traço); 3
no check animado.

---

## 10. Padrões de tela

### 10.1 Listagem (a tela inicial)

```
CabecalhoMobile (ações)
header  — rótulo + título        [data-encadeado --i:0]
alertas                          [--i:1]
busca + filtros                  [--i:2]
lista                            [--i:3]
```

- **Busca**: `.campo-busca` com ícone à esquerda que fica em acento quando
  focado, limpar à direita, e 300ms de espera antes de propagar. O campo é
  isolado num componente memoizado — digitar não pode rerenderizar a lista.
- **Filtros**: `ChipBotao`. Ao mudar o filtro, os cartões que saem são
  empurrados para fora (fantasma `fixed`, 420ms) e os que ficam se reacomodam
  por FLIP.
  - **Chips não quebram linha.** Quatro filtros numa tela estreita viram duas
    fileiras, com um chip órfão embaixo. Uma fileira só, com rolagem lateral
    (`overflow-x: auto` e barra escondida), e os chips com `shrink-0`.
  - **Uma escolha por linha de controle, e o dropdown vem antes do chip.** Um
    campo de busca, uma fileira de dropdowns e uma fileira de chips empilhados
    ocupam meia tela antes do primeiro item da lista; no celular, dois
    dropdowns dividem a mesma linha.
- **Ladrilho do ícone**: um componente, e ele **nunca fica vazio**. Ele aparece
  em toda lista do app — foi assim que virou seis cópias com três tamanhos. Sem
  categoria (ou sem cor), o ladrilho é neutro, ou fala pelo tipo do item: um
  ícone solto, sem a caixa, desalinha todas as linhas em volta e parece defeito
  de carregamento.
- **Lista**: agrupada por dia; cada grupo entra com `.revela-ao-rolar` ao montar.
  No desktop os cartões do mesmo dia viram uma fileira horizontal com encaixe:

```css
.encaixa-lateral {
  scroll-snap-type: x proximity;
  scroll-padding-left: 1.25rem;
}
.encaixa-lateral > * {
  scroll-snap-align: start;
}
```

- **Mosaico** (calendário, galeria): `columns` com `column-gap: 0.875rem` e
  `break-inside: avoid` nos filhos.
- **Carregando**: esqueletos com as alturas variadas do conteúdo real, nunca um
  bloco só.

### 10.2 Detalhe com abas

```
cabeçalho que encolhe ao rolar   [--i:0]
abas: Trilho instantaneo         [--i:1]
TrocaPainel com o conteúdo       [--i:2]
coluna lateral (lg+)
```

### 10.3 Formulário

`.coluna-conteudo`, campos em `flex flex-col gap-4`, seções em cartão com
`p-5 sm:p-6`, ação principal larga no rodapé. Explicação vai em `<Nota>`, não em
parágrafo solto.

### 10.4 Leitura

`.coluna-leitura` (68ch), fonte serifada em `.txt-xl`, e o cabeçalho encolhendo
conforme a rolagem.

### 10.5 Erro

Um componente, e cada rota só passa o contexto:

```tsx
<ErroTela error={error} reset={reset} onde="galeria" descricao="…" />
```

`flex min-h-[60dvh] flex-col items-center justify-center gap-5 text-center`;
ícone de 28px num ladrilho de 56px tingido com o acento; título
`.display .txt-titulo` ("algo deu errado"); descrição **específica daquela
tela**; dois botões — "Tentar de novo" e voltar ao início. O identificador do
erro aparece em `.t3 font-mono .txt-xs` quando existe.

**Toda rota tem o seu limite de erro.** Sem ele, um erro em qualquer tela derruba
o app inteiro.

### 10.6 Carregamento

Três ferramentas, nesta ordem de preferência: esqueleto com o formato do
conteúdo → `TrocaSuave` (crossfade de 180ms entre esqueleto e conteúdo) →
indicador girando (só quando não dá para prever o formato).

A roda girando é a terceira opção, não a primeira. **Se você sabe que vem uma
lista de cartões, desenhe cartões cinzas** — a tela não muda de forma quando o
conteúdo chega, e a espera fica menor porque o olho já tem onde pousar. Uma
tela do app de referência tinha ficado com o único giro do produto inteiro, e
ele destoava por ser o único que não dizia o que vinha depois.

### 10.7 Vazio: são DOIS, e quase sempre um está esquecido

Toda tela com lista tem dois estados vazios, e eles dizem coisas diferentes:

| Qual                   | O que a pessoa fez | O que a tela diz                     |
| ---------------------- | ------------------ | ------------------------------------ |
| **Nunca teve nada**    | chegou agora       | o que isto é, e o botão para começar |
| **O filtro não achou** | buscou ou filtrou  | não achei isso — e nada mais         |

O primeiro é um convite e leva ação. O segundo é uma resposta e **não** leva
ação: oferecer "criar" para quem acabou de buscar é responder outra pergunta.

O erro que se repete: o primeiro vira `<EstadoVazio>` e o segundo, por ser "só
uma frase", nasce à mão — e a mesma tela passa a ter duas linguagens a dois
`else` de distância. Os dois usam o componente.

### 10.8 Uma promessa da interface vale em TODOS os caminhos de saída

A regra mais fácil de quebrar sem perceber, porque cada caminho é escrito num
dia diferente.

Se a interface promete que um conteúdo está protegido — uma entrada trancada
por senha, um item marcado como privado, um campo escondido —, essa promessa
precisa valer em **cada lugar por onde o conteúdo sai**:

- a prévia no cartão da lista
- a busca (inclusive a indexação para busca semântica)
- o que é mandado a um modelo de IA
- a exportação legível (texto, PDF, ZIP)
- a impressão
- o link público de leitura
- a notificação

No app de referência, os cinco primeiros estavam corretos — com comentários
explicando o cuidado — e a **exportação** ficou de fora por meses. É o caminho
que produz um arquivo para guardar em qualquer lugar ou mandar para uma
gráfica, ou seja, exatamente o pior lugar para o furo estar.

Faça a lista dos caminhos de saída **uma vez**, escreva-a no código junto do
estado que protege, e confira-a inteira toda vez que um caminho novo nascer.

A exceção legítima é o **backup**: ele é arquivo de restauração, não de leitura,
e um backup que perde conteúdo não restaura nada. Diga isso no comentário, para
ninguém "corrigir" depois.

---

## 11. Gestos

Todos só em `(pointer: coarse)` e todos desligados sob movimento reduzido. A
regra da resistência: **o dedo anda três, a tela anda um** — é o que impede um
deslize distraído de disparar a ação.

| Gesto                     | Limiar | Resistência   | Sinal                                      |
| ------------------------- | ------ | ------------- | ------------------------------------------ |
| Voltar (borda esquerda)   | 90px   | ÷3, máx 50%   | faixa de 24px na borda; volta em 260ms     |
| Ação no cartão (deslizar) | 72px   | ÷3            | ícone aparecendo atrás                     |
| Puxar para atualizar      | 72px   | ÷2, máx 108px | ícone gira `puxada × 3°` e cresce até 1.12 |
| Fechar folha (arrastar)   | 64px   | ÷3            | alça + fundo clareando                     |

O gesto de voltar só começa se o movimento for horizontal e passar de 12px — o
vertical continua sendo rolagem. O de arrastar a folha só começa se a rolagem
interna estiver no topo, e só conta movimento para baixo.

O retorno elástico do puxar-para-atualizar usa a mola de §6.4.

---

## 12. Acessibilidade

Piso, não aspiração:

- **Contraste** testado no build (§3.6).
- **Foco visível** em tudo; `outline-none` só acompanhado de `foco-anel`.
- **Botão só com ícone** sempre tem `aria-label` (o `BotaoIcone` exige).
- **Diálogo** passa pela `<Folha>`, que traz `role`, `aria-modal`, Esc, prisão de
  Tab e devolução do foco. Não reimplemente isso.
- **Movimento** respeitado no CSS e no JS (§6.5).
- **Alvo de toque** de 44px.
- **Cor nunca é o único sinal**: o nome está escrito ao lado do pontinho.
- **`alt`** descritivo nas imagens de conteúdo, `alt=""` nas decorativas.
- **Erro** anunciado com `role="alert"`.
- **Alto contraste** como modificador combinável, não como tema à parte.
- **Zoom do iOS**: campo com `font-size: max(16px, 1em)`.

O que nenhum código garante: ouvir o app com leitor de tela. Isso é trabalho de
gente.

---

## 13. Voz e escrita

- **Título de tela em caixa baixa**, com a display. Sempre.
- **Botão no infinitivo e dizendo o que acontece**: "Salvar entrada", não "OK".
- **Erro diz o que houve e como resolver, sem pedir desculpa**: "Não consegui
  salvar. Tente de novo." — não "Ops! Algo deu errado 😢".
- **Português do Brasil, sem jargão de sistema.** Nada de "sincronizar payload".
- **Rótulo é contexto, não decoração**: "47 entradas até aqui" vale mais que
  "visão geral".
- **Vazio convida**: diga o que aparece ali quando houver algo, e ofereça o
  primeiro passo.
- **O código também fala português.** Componentes, props e classes CSS têm nomes
  em português (`Botao`, `variante`, `.cartao`, `--fundo`). É o que mantém o
  vocabulário do produto e o do código na mesma língua.

---

## 14. Como adaptar para outro app

O que segue é a receita de troca. Nada aqui exige repensar o sistema.

### 14.1 Trocar a cor de acento

1. Escolha **um** hexadecimal. Ele é o `--color-acento-500`.
2. Gere os onze degraus **do mesmo matiz e da mesma saturação**, variando só a
   luminosidade. Não invente uma segunda cor de marca.
3. Preencha os papéis:
   - `--acento-solido` = o 500, se ele passar de 3:1 contra o fundo. Se não
     passar (amarelos, verdes-limão), use o 600 ou 700.
   - `--acento-solido-texto` = branco ou preto, o que der mais contraste.
   - `--acento-texto` = o degrau que passa **4.5:1** como letra sobre a
     superfície, no claro; e o degrau que passa 4.5:1 no escuro (normalmente um
     mais claro que o 500).
   - `--anel-foco` = o 500 com 40% de alfa (50% no escuro).
4. **`--perigo`**: se o seu acento for vermelho/coral, aponte o perigo para ele —
   destruir é destaque e dois vermelhos parecidos na mesma tela é pior. Se o
   acento for qualquer outra coisa, `--perigo` vira um vermelho próprio, com
   `--perigo-texto` e `--perigo-tenue` (10% no claro, 16% no escuro).
5. Rode o teste de contraste. Ele vai reprovar o que estiver frouxo — corrija o
   token, não o teste.

O resto do app acompanha sozinho, porque nada usa a cor crua.

### 14.2 Definir as categorias coloridas

Siga a §3.4: um token por categoria, um valor por tema, catálogo devolvendo o
token e nunca o hexadecimal, e o teste cobrando **3:1** contra a superfície e
contra o próprio chip tingido.

Para escolher os tons: pegue a família de matizes que faz sentido no domínio,
tire a versão **escura** para o tema claro e a **clara** para o escuro, e
confira uma a uma no teste. Cores vizinhas no círculo cromático são o problema
mais comum — se duas categorias forem confundíveis num gráfico, afaste o matiz
antes de mexer na luminosidade.

### 14.3 Trocar a navegação

A estrutura (coluna à esquerda no desktop, pílula flutuante no celular, dois
destinos visíveis + menu) funciona de 4 a 8 destinos. Ajuste:

- **Menos de 4 destinos**: mostre todos na pílula e dispense o menu `⌄`.
- **Mais de 8**: agrupe. Uma pílula com dez itens não é navegação, é um índice.
- A **ação principal** (o "+") fica sempre à direita da pílula e na base da
  coluna. Se o seu app não tem uma ação principal única, tire o botão em vez de
  inventar uma.

**A ação principal é da TELA, não do app.** O "+" da navegação cria uma conta em
"contas", um cartão em "cartões", um agendamento na agenda — e cai na ação do
app só onde a tela não registrou nenhuma. Isso resolve, de uma vez, o defeito
que aparece sozinho em todo app com listas: **cada tela nasce com o seu próprio
"+"**, um no topo, outro flutuando, e a pessoa passa a ter dois botões de criar
na mesma tela, em lugares diferentes, fazendo coisas diferentes.

A mecânica é um contexto e um hook:

```tsx
// Na tela:
useDefinirAcaoPrincipal("Nova conta", () => setAbrindo(true));
```

Dois detalhes de implementação, e o segundo é o que quebra:

- O registro vive num contexto acima da navegação **e** do `<main>`, senão a
  barra não enxerga o que a tela registrou.
- **Guarde o callback num `ref` e registre um invólucro estável.** `aoAcionar` é
  quase sempre uma arrow inline, nova a cada render: registrá-la direto é um
  `setState` por render — laço infinito, tela branca, e um "Maximum update
  depth" como única pista.

E as ações **secundárias** da tela (sincronizar, importar, filtrar) não disputam
esse lugar: no celular vão para a faixa da marca; no desktop, para o cabeçalho.

### 14.4 O que é tela e não é sistema

Categorias coloridas, exportação em PDF, editor de texto rico, gravação de voz,
transcrição, retrospectiva — nada disso é infraestrutura do design system. São
telas que usam os mesmos tokens e os mesmos componentes. Acrescente as suas sem
medo; o sistema não muda por causa delas.

### 14.4-b O texto é dado, não código

Se o app vai ter mais de um idioma — e mesmo que não vá —, **nenhuma frase mora
no componente**. O dicionário fica em arquivos por área (`telas`, `componentes`,
`entrada`…), com um objeto por idioma e o tipo derivado do idioma-base:

```ts
const pt = { salvar: "Salvar", fotos: (n: number) => `${n} fotos` };
export type Textos = typeof pt;
export const textos: Record<Idioma, Textos> = { pt, en, es };
```

Derivar o tipo do idioma-base é o detalhe que faz isto funcionar: **esquecer uma
chave em `en` vira erro de compilação**, não uma frase em português na tela de
alguém. Sem isso, a tradução apodrece em silêncio.

O que escapa quase sempre, e é o que a revisão deve procurar: `aria-label`,
`title`, `placeholder`, mensagem de modal de confirmação, e o rótulo "Enviando…"
dentro de um botão que carrega.

### 14.5 O que **não** mudar

Se você mexer nisto, o resultado deixa de parecer o mesmo produto:

- Título em caixa baixa com display de tracking apertado.
- Fundo quente no claro, preto de verdade no escuro.
- Véu de hover único, seguindo a luz do tema.
- Chips como pílula com fundo tingido a 5%.
- Cartão de raio 1.25rem, campo e botão de raio 0.75rem.
- Duas durações e duas curvas.
- Barra de navegação como pílula flutuante com desfoque.
- Encadeamento de 48ms na entrada da tela.

---

## 15. Checklist

Antes de dar uma tela por pronta:

- [ ] Nenhum hexadecimal, `ms`, `px` de raio ou `text-[..px]` escrito à mão.
- [ ] O `body` declara o tamanho do corpo — nada cai nos 16px do navegador.
- [ ] Nenhuma variante apontando para classe de componente (`hover:superficie-2`
      não existe, e falha em silêncio).
- [ ] Número em cartão que divide a linha usa o degrau estreito: com dado real e
      comprido, o texto não sai pela borda.
- [ ] Sobrancelha cabe em uma linha; rótulo de ajuste é frase, não caixa alta.
- [ ] O que é `fixed` dentro da tela vai por portal — a animação de entrada é
      bloco de contenção.
- [ ] A tela registrou a sua ação principal, e não criou um "+" próprio.
- [ ] Todo controle é um componente do sistema, não uma `<div>` estilizada.
- [ ] Quem tem `outline-none` tem `foco-anel` junto.
- [ ] Botão só com ícone tem rótulo acessível.
- [ ] Nenhum `<a>` dentro de `<a>` ou de `<button>`.
- [ ] Nenhum `alert()`.
- [ ] Funciona nos quatro temas (confira em `/estilo`).
- [ ] Passa o teste de contraste.
- [ ] Toda animação em JS pergunta por `querMenosMovimento()`.
- [ ] Hover só dentro de `@media (hover: hover)`.
- [ ] Estado vazio, de carregando e de erro existem — os três.
- [ ] Alvo de toque de 44px.
- [ ] O texto está em caixa baixa no título e no infinitivo no botão.
- [ ] Componente novo foi acrescentado à página `/estilo`.
- [ ] Nenhuma frase escrita no componente: **todo texto vem do dicionário**,
      inclusive `aria-label`, `title` e mensagem de modal.
- [ ] O que exige rede está apagado e `inert`, não escondido.
- [ ] Camada nova entrou na escala `--z-*`, com nome.
- [ ] Existe **um** alvo de soltar arquivo, não dois.
- [ ] Estado vazio usa o componente — inclusive o "nada encontrado" do filtro,
      que é o que mais escapa (a mesma tela costuma ter dois).
- [ ] Nenhum bloco de classes repetido três vezes (ver §8.0).
- [ ] Os DOIS vazios da tela existem e usam o componente (§10.7).
- [ ] Carregamento tem o formato do conteúdo; a roda girando é a terceira opção.
- [ ] Todo caminho de saída respeita o que a interface diz estar protegido
      (§10.8) — lista, busca, IA, exportação, impressão, link público.
