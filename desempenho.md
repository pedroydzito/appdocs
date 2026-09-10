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
16. [Diagnóstico: "o meu app demora"](#16-diagnóstico-o-meu-app-demora-para-carregar-as-telas)

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

| Métrica                         | Teto   | O que ele responde                         |
| ------------------------------- | ------ | ------------------------------------------ |
| Carregamento inicial            | 1,5 MB | quanto custa ABRIR o app                   |
| Por rota (navegação interna)    | 1,2 MB | quanto custa CHEGAR em cada tela           |
| Aviso para um chunk sob demanda | 0,4 MB | o que está grande, mas corretamente adiado |

Um teto que ninguém checa não é teto. Rode junto do `typecheck`, do lint e dos
testes, e **quebre o build**.

### 2.1 Meça por rota, não só o total

O total sozinho esconde a regressão que importa. Ele mede o esqueleto que toda
visita baixa; uma tela pode **dobrar de peso sem mexer nesse número**, porque o
peso dela está num chunk que só desce quando alguém navega até lá. É por rota
que uma dependência pesada entra sem ninguém ver.

O caso real: uma tela de Ajustes passou de 1,08 para 1,24 MB — 15% num commit —
porque um utilitário de exportação importava um compactador **no topo do
arquivo**. Quem abria Ajustes para trocar o tema baixava a biblioteca inteira de
gerar arquivo `.zip`. O total inicial não se mexeu um byte, e sem o teto por
rota a regressão teria passado.

### 2.2 A regra que evita 90% disso

> Biblioteca que só serve **depois de um clique** entra por `await import()`
> **dentro da função**, nunca no topo do arquivo.

```ts
// ✗ Todo mundo que abre a tela baixa o compactador.
import JSZip from "jszip";
export async function exportarTudo(itens) {
  const zip = new JSZip();
}

// ✓ Só quem clica em "exportar" baixa.
export async function exportarTudo(itens) {
  const { default: JSZip } = await import("jszip");
  const zip = new JSZip();
}
```

Vale para: compactador, gerador de PDF, captura de tela, confete, leitor de
código de barras, reconhecimento de voz, editor de imagem, qualquer parser.

### 2.3 Como achar o culpado de uma rota pesada

1. Rode o build e leia o peso por rota (o script abaixo).
2. Na rota estourada, liste os `import` do topo do arquivo da página.
3. Pergunte de cada um: **isto é preciso antes de qualquer clique?** Se não, é
   candidato a `await import()` ou a `next/dynamic`.
4. Meça de novo. Uma dependência bem escolhida costuma valer 100–200 KB.

O script é curto: o Next escreve, em `.next/app-build-manifest.json`, quais
arquivos cada rota carrega. Somar os tamanhos desses arquivos é o peso da rota.
Some `.next/static/chunks` para o carregamento inicial. Menos de cem linhas, sem
dependência nenhuma, e é o que segura o app leve por anos.

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
| Gráficos                     | dinâmico, **num módulo só** (ver abaixo)          |
| Geração de PDF / captura DOM | `import()` só no clique de exportar               |
| Confete / animação de festa  | `import()` no momento da celebração               |
| Zoom/pan de imagem           | dinâmico                                          |
| Suavização de rolagem        | estático (é global), mas desligada onde atrapalha |

Some a isso a otimização de importação por barril (`optimizePackageImports` para
pacotes de ícones, gráficos e datas): ela reescreve `import { X } from "pacote"`
para o módulo interno e evita arrastar o índice inteiro.

**Um import dinâmico por componente pode duplicar a biblioteca.** Dois gráficos
em dois arquivos, cada um com o seu `next/dynamic`, viraram **dois chunks de
330 KB** — a mesma biblioteca, baixada duas vezes, na mesma tela. O empacotador
não junta o que você separou. Se dois componentes dinâmicos dependem da mesma
biblioteca pesada, ponha-os **no mesmo módulo**: um módulo, um chunk, uma cópia.
Confira medindo o que a tela baixa, não lendo o código.

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

### 4.5 O cache do ROTEADOR — a causa nº 1 de "tudo demora"

Esta é, isolada, a diferença entre um app que parece nativo e um que parece um
site. Se você só for aplicar uma regra deste documento inteiro, aplique esta.

**O problema.** Num roteador moderno de React (o App Router do Next, e os
equivalentes), navegar **destrói a árvore da rota anterior**. Sair da lista,
abrir um item e voltar não devolve a tela de antes: monta tudo de novo, do zero.
E como o padrão do Next 15+ é `staleTimes.dynamic = 0`, o payload da rota
também não é reaproveitado — ele é buscado outra vez no servidor. O resultado é
exatamente a queixa "abro, volto, e tenho que esperar de novo".

**A correção, em duas camadas — e as duas são necessárias:**

```ts
// next.config.ts — camada 1: o roteador guarda a rota já buscada.
experimental: {
  staleTimes: { dynamic: 180, static: 300 },
}
```

Não pesa nada: o que fica guardado é o resultado que o navegador **já** tinha
baixado, na memória da própria aba, e some ao fechar. Três minutos cobre o
vaivém real de quem usa o app; passado isso, busca de novo.

```ts
// camada 2: o SEU cache de dados vive num módulo, fora do React.
const cache = new Map<string, Pagina>();
```

A camada 1 evita a ida ao servidor pelo _código e pelo payload_ da rota. A
camada 2 evita a ida ao servidor pelos _dados_. Sem a 2, a rota volta instantânea
e a lista dentro dela volta vazia — que é meio caminho e parece defeito.

**Guarde TODAS as páginas roladas, não só a primeira.** Voltar para a lista
depois de ter rolado até março e cair em janeiro é o mesmo defeito com outra
cara.

### 4.6 O `getUser()` do middleware, uma vez por navegação

O padrão que todo guia de autenticação ensina — validar a sessão no middleware —
cobra **uma ida à rede ao provedor de identidade em cada requisição**, inclusive
em cada payload de rota que o roteador busca ao navegar. É o custo fixo mais
caro do vaivém entre telas, e ele não aparece em nenhum perfil de JavaScript.

Na **abertura a frio** é pior, e é lá que se percebe: a instância também está
fria, então é um TLS novo inteiro na frente do primeiro byte do HTML — nada da
página começa antes disso. Ver §8.4.

A correção é ler o vencimento do token **do próprio cookie** e só ir à rede
quando falta pouco (uma margem de ~120 s):

```ts
const restam = segundosRestantes(request); // exp do JWT, lido localmente
if (!portaDeEntrada && restam !== null && restam > 120) return response;
```

Três cuidados, e o terceiro é o que quebra:

1. **Isto não é autorização.** É só a decisão de renovar ou não. Quem autoriza é
   a checagem no servidor da própria tela, e a regra de acesso no banco.
2. Sem cookie, ou com formato inesperado, **caia no caminho completo**.
3. **As portas de entrada ficam de fora do atalho.** `/`, `/entrar` e
   `/cadastro` decidem para onde a pessoa vai: mandar alguém para a tela inicial
   com um token que o servidor já recusou faz a tela devolver para o login, e o
   login devolver para a tela inicial — um pingue-pongue infinito.

### 4.7 A janela de frescor: 20 segundos sem perguntar nada

Ter cache não basta se toda montagem dispara a revalidação assim mesmo: a tela
abre cheia, e um segundo depois pisca com a mesma resposta.

> Se o que está guardado tem menos de **20 segundos**, não consulte nada.

Vinte segundos é o tempo de abrir um item, ler e voltar. Menos que isso e a ida
à rede volta a acontecer em toda navegação; mais e a lista começa a parecer
velha para quem usa dois aparelhos.

Duas janelas, com prazos diferentes:

| O quê                     | Janela | Por quê                                           |
| ------------------------- | ------ | ------------------------------------------------- |
| Lista / tela              | 20 s   | é o vaivém de navegação                           |
| Item individual (detalhe) | 60 s   | detalhe aberto e fechado não deve refazer a busca |

**A exceção que não pode faltar:** quem **escreve** invalida a janela. Salvar um
item acontece em outra rota, com a lista desmontada — nenhum componente estaria
escutando. Por isso a assinatura é **do módulo**, não do componente:

```ts
// No repositório de dados, fora do React:
inscrever(() => {
  for (const [chave, pagina] of cache) cache.set(chave, { ...pagina, em: 0 });
});
```

Sem isso, voltar para a lista nos 20 segundos seguintes a salvar mostra a lista
de antes — sem a coisa que a pessoa acabou de criar. É pior do que lentidão.

### 4.8 Não pinte o que não mudou

A revalidação de fundo quase sempre devolve **exatamente** o que já está na
tela. Trocar o estado assim mesmo faz a tela inteira renderizar de novo — e, se
houver animação de transição, um fade da página um segundo depois de ela abrir.
A pessoa lê isso como "recarregou".

```ts
const mesmaLista =
  novos.length === atuaisRef.current.length &&
  novos.every((e, i) => {
    const atual = atuaisRef.current[i];
    return atual?.id === e.id && atual?.atualizadoEm === e.atualizadoEm;
  });

if (mesmaLista) {
  setCarregando(false); // só sai do esqueleto, se ainda estiver nele
  return; // ninguém pinta nada
}
```

Compare por **identidade + carimbo de atualização**, não por conteúdo inteiro:
é O(n) e não depende de serializar objeto.

### 4.9 Respostas fora de ordem

Trocar de filtro depressa dispara buscas que voltam **fora de ordem**, e a mais
lenta chega por último e pinta a lista dela por cima da tela que já mostra
outro filtro. Duas guardas, e as duas são precisas:

```ts
const aindaVale = chaveDe(f) === chaveDe(filtroAtualRef.current);
if (!aindaVale) return;                       // 1. não pinta
// ...
finally {
  // 2. e também NÃO desliga o esqueleto do filtro que ainda está vindo
  if (chaveDe(f) === chaveDe(filtroAtualRef.current)) setCarregando(false);
}
```

Esquecer a segunda é sutil e apareceu em produção: a resposta obsoleta desligava
o esqueleto do filtro novo, e a lista anterior ficava na tela sem sinal nenhum
de carregamento — como se aquele fosse o resultado.

### 4.10 Cuidado: Suspense recria efeitos

Quando um componente carregado por `next/dynamic` suspende, o React **esconde a
árvore e, ao mostrá-la de volta, recria os efeitos** — mantendo estado e refs.
Todo `useEffect` daquele ramo roda de novo, com os refs no valor em que ficaram.

Isso quebra qualquer guarda do tipo "só na primeira montagem" escrita com ref:

```ts
// ✗ Frágil: o ref já foi gasto, e o efeito roda de novo depois do Suspense.
const primeiroRender = useRef(true);
useEffect(() => {
  if (primeiroRender.current) {
    primeiroRender.current = false;
    return;
  }
  animarSaida(); // dispara sem nunca ter havido entrada
}, [aberto]);

// ✓ Guarde o FATO, não a ordem.
const jaAbriu = useRef(false);
useEffect(() => {
  if (aberto) {
    jaAbriu.current = true;
    return;
  }
  if (!jaAbriu.current) return; // nunca abriu: não há saída para animar
  animarSaida();
}, [aberto]);
```

O sintoma real disso: ao entrar numa tela cujo painel é dinâmico, dois diálogos
fechados apareciam por meio segundo e sumiam.

---

### 4.11 A lista pesada não bloqueia a casca

Um layout que busca dados para toda a árvore acaba esperando pela **maior** das
listas antes de mandar o primeiro byte — inclusive nas telas que não a leem. É o
custo mais fácil de não enxergar: cada tela parece lenta por igual, porque todas
pagam a mesma conta.

O conserto não é buscar menos, é **não esperar**. O servidor entrega a promessa
sem `await`; quem precisa do dado a lê com `use()` e suspende só o Suspense mais
próximo. A casca, a navegação e o cabeçalho da tela pintam com o resto.

**A regra que faz isso funcionar: o provedor não lê a promessa.**

```tsx
// ERRADO: use() aqui suspende a árvore inteira — a navegação volta a esperar.
function Provedor({ promessa, children }) {
  const lista = use(promessa);
  return <Ctx.Provider value={lista}>{children}</Ctx.Provider>;
}

// CERTO: o provedor guarda a promessa; quem lê é que suspende.
function Provedor({ promessa, children }) {
  return <Ctx.Provider value={{ promessa }}>{children}</Ctx.Provider>;
}
export function useLista() {
  return use(useContext(Ctx).promessa);
}
```

Três coisas que vêm junto:

- **Todo leitor precisa de um Suspense acima dele.** As rotas que já têm o
  esqueleto de carregamento estão cobertas; as que não têm derrubam o fallback
  para o do layout — que é a casca inteira, exatamente o que você estava
  evitando. Ponha um Suspense de segurança em volta do conteúdo da rota.
- **O que vive na navegação não pode ler a lista.** Uma busca global dentro da
  barra lateral, lendo as transações, prende a barra na promessa. Separe: a
  barra é a casca; o **painel** que abre no clique é quem lê, sob o Suspense
  dele.
- **O otimista continua funcionando** se a fila de patches for `useOptimistic`
  sobre a **lista vazia**, aplicada em cima do que a promessa devolveu. O React
  descarta a fila sozinho quando a transição termina, e você não precisa de uma
  cópia local da lista para nada.

Revalidar cria uma promessa nova, e uma promessa nova suspenderia de novo — mas
isso acontece dentro de uma transição (a ação do servidor, o `refresh` do
roteador), e transição não revela fallback: a tela anterior fica no lugar. Se
você vir o esqueleto piscar a cada escrita, o que faltou foi a transição.

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

### 6.3 `useMemo` morre na navegação — a derivação cara vive no módulo

`useMemo` guarda o resultado **enquanto o componente vive**. Numa moldura que
remonta a tela a cada navegação — e ela remonta, porque é o `key={caminho}` que
faz a animação de entrada rodar de novo (§9.3 do design system) — toda derivação
cara é refeita do zero **a cada ida e volta entre abas**. É invisível no perfil
de uma tela só, e é metade do "trocar de aba demora".

O cache vai para o módulo, fora do React, com a **identidade dos dados** como
chave:

```ts
export function memoModulo<C extends readonly unknown[], V>(calcular: (...c: C) => V) {
  let anterior: { chaves: C; valor: V } | null = null;
  return (...chaves: C): V => {
    if (anterior && anterior.chaves.every((c, i) => c === chaves[i])) return anterior.valor;
    const valor = calcular(...chaves);
    anterior = { chaves, valor };
    return valor;
  };
}
```

Guardar **só o último** resultado é de propósito: é o que o vaivém usa, e não
vira vazamento. Funciona porque a camada de dados troca o objeto só quando o
dado muda de verdade (§6.2) — se cada busca recria os arrays, isto não segura
nada, e o `memo()` das células também não.

E, antes de memoizar, **conte as varreduras**. Um gráfico de doze meses escrito
com um `filter` por mês e por tipo faz 72 varreduras completas da lista a cada
montagem; a mesma conta numa passada só, com um mapa de mês para índice, é
O(n). Memoizar uma conta O(72n) é esconder o problema, não resolvê-lo.

### 6.4 `useMemo` onde há derivação real

Filtro, agrupamento, ordenação e contagem sobre a lista inteira: sim. Concatenar
duas strings: não. `useMemo` tem custo próprio; usá-lo em tudo só adiciona
trabalho.

### 6.5 Anime layout com FLIP, não com layout

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

### 7.5 A promessa recusada que trava a tela para sempre

A família de defeito mais comum num app assíncrono, e a mais difícil de
enxergar em revisão: **um `await` fora do `try`**, entre ligar e desligar um
estado de "ocupado".

```ts
// ✗ Se a promessa é recusada, `salvando` fica ligado PARA SEMPRE.
setSalvando(true);
await salvar(dados);
setSalvando(false);

// ✓
setSalvando(true);
try {
  await salvar(dados);
} finally {
  setSalvando(false);
}
```

O sintoma nunca é "deu erro": é um botão girando sem fim, um teclado inerte, uma
frase de "confirmando…" que não sai da tela. A pessoa não vê defeito, vê
lentidão — e é por isso que isto está no documento de desempenho.

Uma busca mecânica acha quase todos:

```
set<Algo>(true) … await … set<Algo>(false)   sem try/catch/finally no meio
```

Os quatro lugares onde ele costuma se esconder:

| Lugar                                     | O que fica preso                         |
| ----------------------------------------- | ---------------------------------------- |
| Conferência de senha/PIN                  | o teclado inteiro, e o conteúdo trancado |
| Sair da conta                             | o botão girando, sessão em limbo         |
| Espera por webhook depois de um pagamento | "confirmando…" sobre um plano já ativo   |
| Salvar formulário                         | botão girando, sem mensagem nenhuma      |

**A variante silenciosa** é pior: o `.then` que faz a fila andar.

```ts
// ✗ Se `criar` é recusada, `seguirParaOProximo` nunca roda.
void criar(nome).then((feita) => {
  aplicar(feita);
  seguirParaOProximo();
});

// ✓ O `catch` ANTES do `then` mantém o fluxo andando.
void criar(nome)
  .catch(() => null)
  .then((feita) => {
    if (feita) aplicar(feita);
    seguirParaOProximo();
  });
```

Aqui não trava um botão: trava o **fluxo**. Um diálogo que não avança para a
próxima pergunta, um assistente parado no passo 2, e nada no console.

### 7.6 O `await` que devolve depois de a tela morrer

Todo recurso de aparelho — microfone, câmera, geolocalização, sensor — é
pedido com uma promessa que pode demorar **o tempo de a pessoa ler um diálogo
de permissão**. Nesse intervalo ela pode sair da tela. E aí:

```ts
// ✗ A limpeza do efeito já rodou, com o ref ainda vazio. O microfone fica
//   ligado, com o ponto vermelho do sistema aceso, até fechar a aba.
const fluxo = await navigator.mediaDevices.getUserMedia({ audio: true });
trilhasRef.current = fluxo.getTracks();

// ✓
const fluxo = await navigator.mediaDevices.getUserMedia({ audio: true });
if (desmontadoRef.current) {
  fluxo.getTracks().forEach((t) => t.stop());
  return;
}
trilhasRef.current = fluxo.getTracks();
```

A regra geral: **depois de todo `await` que devolve um recurso, pergunte se a
tela ainda existe** — e, se não existir, devolva o recurso ali mesmo. A limpeza
do `useEffect` não alcança o que ainda não tinha nascido quando ela rodou.

Vale igual para `MediaRecorder` (parar o gravador, não só as trilhas),
`AudioContext` (fechar), `WakeLock` (liberar) e handles do sistema de arquivos.

---

### 7.7 O app "recarrega" quando a pessoa volta para ele

Sintoma: a pessoa sai do app, atende uma mensagem, volta — e a tela remonta do
zero, perdendo a rolagem e o estado. Parece descarte de aba pelo sistema, e às
vezes é. Mas na maioria dos casos é o **bfcache desligado por um cabeçalho**.

> O navegador não guarda no bfcache uma página cujo documento veio com
> `Cache-Control: no-store` — e `no-store` é o padrão de toda rota renderizada
> sob demanda no Next (e equivalentes).

Trocar por `private, no-cache, max-age=0, must-revalidate` mantém a revalidação
em toda navegação (nada obsoleto é servido) e devolve o bfcache: voltar para o
app passa a restaurar a página **como ela estava**, instantaneamente, sem
requisição nenhuma.

Onde escrever isso: no middleware, na resposta de navegação — e **só nela**. As
rotas de API têm cabeçalho próprio (um stream de modelo precisa do `no-store`
dele), então exclua `/api/` da regra.

As outras coisas que desligam o bfcache, e que vale conferir junto:

- um ouvinte de `unload` (use `pagehide`);
- uma conexão aberta que o navegador não sabe pausar (`WebSocket`, `EventSource`)
  — feche-a em `pagehide` e reabra em `pageshow`;
- `window.opener` vivo.

## 8. Offline-first

Três camadas, e cada uma resolve uma coisa:

### 8.1 Service worker — a casca

| Recurso                            | Estratégia                                               |
| ---------------------------------- | -------------------------------------------------------- |
| Navegação                          | **cache guardado na hora**, revalidando atrás (ver §8.4) |
| Navegação sem cópia deste endereço | rede com prazo, cache como rede de segurança             |
| Estáticos com hash                 | cache primeiro (o hash já é a versão)                    |
| Imagens do usuário                 | cache e revalida em paralelo                             |
| API                                | **sempre rede**, nunca cache                             |

Detalhes que evitam dor:

- **Versione o cache** (`app-v8`) e apague os antigos ao ativar. Cache de app
  sem versão é a causa clássica do "atualizei e continua velho".
- **Cache de estáticos e de imagens atravessa versões** — não faz sentido baixar
  de novo o que tem hash no nome.
- **Ponha teto e descarte o mais antigo**: 120 arquivos estáticos, 400 imagens.
  Sem teto, o armazenamento cresce até o sistema apagar tudo de uma vez.
- **Normalize a chave da imagem** removendo o token da URL assinada — senão cada
  assinatura nova vira uma entrada nova no cache.
- **Cache-primeiro tem de ser cache-primeiro.** Montar o `fetch` antes do `if`
  que decide a estratégia faz toda imagem já guardada disparar uma requisição
  cujo resultado é descartado. Numa galeria são dezenas de idas à rede por
  rolagem, gastando dado de quem está no 4G para rebaixar arquivos idênticos.
  Monte a busca dentro de uma função e chame-a só no ramo que precisa dela.
- **Rejeição pendurada.** Se a navegação tem prazo e cai para o cache, a
  promessa de rede continua correndo e, ao falhar, vira `unhandled rejection`
  dentro do service worker. Marque-a como tratada (`daRede.catch(() => {})`)
  logo depois de criá-la.
- **Nunca guarde uma resposta `redirected`.** Sem sessão, o servidor desvia para
  a tela de entrar; o `fetch` da navegação SEGUE o desvio, e o que chega no
  worker é um 200 com o HTML do login e `resposta.ok` verdadeiro. Guardar isso
  sob a chave da rota pedida faz o app abrir na tela de entrar **mesmo depois de
  entrar**, e ninguém liga uma coisa na outra. A condição é
  `resposta.ok && !resposta.redirected`, e ela vale para toda gravação de HTML.
- **Pré-carregue** as rotas principais e uma página de "sem conexão".
- Sirva o próprio arquivo do service worker com `Cache-Control: no-store`.
- **Registre só em produção, e no primeiro OCIOSO — não no `load`.** Instalar o
  worker dispara o pré-cache inteiro: quinze rotas mais os chunks de cada uma.
  Em `load`, essas dezenas de requisições começam enquanto o app ainda hidrata e
  faz a primeira sincronização, competindo com exatamente o que a pessoa está
  esperando ver — e para servir à PRÓXIMA abertura, não a esta.
  `requestIdleCallback` com `timeout` de 3s (o teto para quem nunca fica
  ocioso). Em desenvolvimento ele só atrapalha.

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

### 8.4 A abertura: o app instalado que demora cinco segundos com a logo na tela

Esta é a espera que quase nenhum guia de desempenho cobre, porque ela não
aparece em perfil de JavaScript nenhum e some dentro de um número que ninguém
mede: o tempo entre tocar no ícone e a tela existir.

O sintoma tem duas caras, e é o **mesmo** problema visto de dois lugares:

- no navegador, digitar o endereço e ficar ~3 s com a barra girando antes de
  qualquer coisa aparecer;
- no app instalado, ~5 s com a tela de abertura do sistema (a logo sobre a cor
  de fundo) antes do app.

A segunda é a mais reveladora. **A tela de abertura do sistema fica no ar até o
primeiro quadro da página.** Ela não é uma animação com duração própria; ela é a
medida exata do que o app faz antes de pintar. Cinco segundos de logo são cinco
segundos de espera de verdade.

**A causa quase sempre é a mesma, e é contraintuitiva: o app tinha tudo o que
precisava, guardado, e foi perguntar ao servidor se havia algo melhor.**

Três esperas se somam, nesta ordem de tamanho.

**1. A navegação do service worker era "rede primeiro".**

É a estratégia que todo guia recomenda, e por um bom motivo: ninguém quer ficar
preso a um HTML de duas versões atrás. Costuma vir com um prazo — 3 s, 3,5 s —
para não ficar tela branca num 4G ruim.

No navegador esse prazo quase não aparece, porque a rede geralmente responde
antes. No app instalado ele aparece **inteiro**, porque é a abertura a frio: o
worker acorda, o aparelho negocia a rede, e a logo fica lá. O HTML estava a um
centímetro de distância.

A correção é servir o guardado **na hora**, e revalidar atrás:

```js
const daRede = (async () => {
  const resposta = await fetch(request);
  if (resposta.ok && !resposta.redirected) {
    await cache.put(chave, resposta.clone());
    void aquecerEstaticos(resposta); // o HTML novo sem os chunks novos é tela branca
  }
  return resposta;
})();
daRede.catch(() => {}); // a rejeição não pode ficar solta

// O caminho rápido: esta rota já está guardada.
const desteEndereco = await cache.match(chave);
if (desteEndereco) return desteEndereco;

// Sem cópia DESTE endereço, segue o caminho antigo: rede com prazo,
// e o que houver guardado como rede de segurança.
```

Duas decisões dentro disso, e as duas importam:

- **Só a cópia EXATA do endereço vale para o caminho rápido.** A casca de outra
  rota também abriria o app — o HTML de um roteador de cliente é o mesmo casco,
  e o roteador lê o endereço e monta a tela certa. Mas isso custa uma remontagem
  no cliente e um piscar da tela errada. É um ótimo negócio contra "sem
  internet"; é um péssimo negócio contra "meio segundo".
- **A versão nova entra na abertura seguinte.** Isso soa pior do que é: o app já
  tinha esse contrato para o próprio service worker (instala, avisa, a pessoa
  recarrega). Servir HTML de uma versão atrás por uma abertura é o mesmo
  compromisso, e a alternativa é cobrar a espera de todo mundo, toda vez, para
  cobrir o dia do deploy.

**2. A portaria ia à rede antes do primeiro byte do HTML.**

O middleware validando a sessão a cada requisição — §4.6. Na navegação entre
telas isso é caro; na abertura a frio é pior, porque a instância também está
fria: é um TLS novo inteiro na frente de tudo, e nada da página começa antes.
Ler o vencimento do próprio cookie e só ir à rede quando falta pouco tira essa
ida do caminho crítico.

**3. O pré-cache começava enquanto o app ainda hidratava.**

Registrar o worker em `load` dispara a instalação — e a instalação baixa as
rotas essenciais e os chunks de cada uma. São dezenas de requisições disputando
banda e CPU com a hidratação e com a primeira sincronização, **para servir à
próxima abertura**. Nada ali é urgente. Vai para o primeiro ocioso.

**E o que sobra na frente: as fontes.** `preload` de fonte é prioridade alta,
antes do JavaScript que desenha a tela. Vale para a fonte que está no primeiro
quadro de toda tela (a de interface, a de título). Não vale para a que só
aparece dentro de um item aberto: ali ela é um par de arquivos competindo com os
chunks para desenhar um texto que ainda não está na tela. Com `display: swap`
nada pisca por tirá-la do `preload` — o texto nasce na fonte de reserva e troca
quando a dela chega, que é exatamente a hora certa.

**A guarda que torna tudo isso seguro.** Servir a casca guardada e restaurar do
bfcache (§7.7) têm o mesmo efeito colateral: a tela aparece sem a portaria ter
decidido nada. Quem perdeu a sessão no meio-tempo — saiu da conta noutra aba,
num computador emprestado — voltaria a ver o app. Então o cliente confere:

```tsx
// Em rota que exige conta. Ouça o sinal ASSENTADO — não pergunte no meio.
cliente.auth.onAuthStateChange((evento, sessao) => {
  if (sessao) return;
  if (evento === "SIGNED_OUT")
    sair(); // acabou de verdade, sem ambiguidade
  else if (evento === "INITIAL_SESSION") void confirmarEsair(); // pede segunda opinião
});
```

**O erro que essa guarda quase sempre comete na primeira versão, e que é caro
porque aparece em TODA abertura:** decidir a partir de um `getSession()` solto
na montagem. `getSession()` responde `null` durante a janela em que o cliente
ainda está renovando um token vencido — e essa janela vira o caso comum
justamente quando você aplica §4.6, porque o token deixa de chegar fresco do
middleware e passa a ser renovado no cliente, pela rede, em quase um segundo.

O resultado é um piscar caríssimo: o app abre, vai para a tela de entrar, o
servidor vê a sessão boa e manda de volta — em quem estava perfeitamente
logado. `onAuthStateChange` emite `INITIAL_SESSION` **depois** que o cliente
terminou de recuperar (ou de não recuperar) a sessão; é esse o sinal.

E vale pedir uma segunda opinião antes de expulsar por `INITIAL_SESSION`: um
`getSession()` um segundo depois. O custo de um falso positivo aqui é alto e
visível; o de esperar um segundo é zero, porque a casca certa já está desenhada
e ninguém está olhando para a tela errada.

Mais dois detalhes que custam caro se forem esquecidos:

- **Vá direto para a tela de entrar, não para o mesmo endereço.** Recarregar o
  caminho atual pareceria mais limpo — a portaria decidiria de novo — mas o
  service worker devolveria a mesma casca guardada, a guarda rodaria outra vez, e
  o app entra num **laço de recarregamentos**. A tela de entrar é pública: a
  guarda não roda lá.
- **Offline, a guarda não expulsa ninguém.** Sem rede, `getSession()` não
  consegue renovar um token vencido e devolve `null` — e aí você estaria
  fechando o diário na cara de quem abriu o app no avião justamente para ler o
  que já está no aparelho. Sem internet a página fica; quem recusa é a próxima
  chamada ao servidor.

**Como medir.** Não use o perfil de JavaScript, que começa tarde demais. Use o
`PerformanceNavigationTiming`: `responseStart − requestStart` é a portaria (2),
`responseStart` perto de zero com o worker no controle é o caminho rápido
funcionando (1), e `loadEventEnd` contra o começo do pré-cache mostra (3). No
app instalado, o cronômetro honesto é o mais simples que existe: filme a tela e
conte os quadros de logo.

### 8.5 A versão nova: aplicar na porta, nunca no meio do uso

Um app com service worker tem um problema que um site normal não tem: **quem
está no comando é o worker antigo, e ele não sai de cena sozinho.** Publicar não
atualiza ninguém — instala uma versão nova que fica _esperando_.

A recomendação padrão é avisar e deixar a pessoa decidir, e ela existe por um
bom motivo: trocar o JavaScript por baixo de uma gravação em curso perde
contexto sem ninguém entender o porquê. O problema é o outro extremo — quem
ignora o aviso fica semanas numa versão velha, e a correção que você publicou
não chega em quem mais precisava dela.

A regra que resolve os dois lados: **aplica na porta, nunca no meio do uso.**

| Quando a versão nova aparece                                                   | O que fazer           |
| ------------------------------------------------------------------------------ | --------------------- |
| Já estava esperando quando o app ABRIU                                         | assume, sem perguntar |
| Terminou de instalar nos primeiros ~10s, sem ninguém ter tocado em nada        | assume também         |
| Apareceu com o app já em uso                                                   | convida               |
| Tela onde há conteúdo em curso (escrever, conversar, alvo de compartilhamento) | convida, sempre       |

O pior caso passa a ser **uma abertura atrás**, nunca mais que isso: o que foi
convite hoje está esperando amanhã, e a primeira linha da tabela o aplica.

**Isto não custa desempenho, e o detalhe é qual chamada você usa.**
`getRegistration()` só consulta o que este navegador já sabe — não é ida ao
servidor —, então pode rodar nos primeiros milissegundos da abertura sem
disputar nada. O `register()` e o pré-cache continuam no ocioso (§8.1).

```ts
// Na porta: local, imediato, sem rede.
if (telaComConteudoEmCurso(location.pathname)) return;
if (sessionStorage.getItem(CHAVE_JA_ASSUMIU)) return; // uma vez por aba
const registro = await navigator.serviceWorker.getRegistration();
if (!registro?.waiting || !navigator.serviceWorker.controller) return;
sessionStorage.setItem(CHAVE_JA_ASSUMIU, "1");
registro.waiting.postMessage("assumir-agora"); // o worker chama skipWaiting()
// `controllerchange` recarrega uma vez.
```

Quatro guardas, e cada uma cobre um jeito de isto dar errado:

- **`controller` tem de existir.** Sem ele não é atualização, é a primeira
  instalação — e recarregar a primeira visita de alguém é gratuito e grosseiro.
- **Uma vez por aba** (`sessionStorage`). Se a ativação falhar e o worker
  continuar esperando, sem essa marca o app recarrega em laço. Armazenamento
  bloqueado (aba anônima) conta como "já tentou": melhor perder a troca do que
  arriscar o laço.
- **"Ainda na porta" é sem TOQUE, não só sem tempo.** Um relógio sozinho
  recarrega em cima de quem abriu o app e começou a ler nos primeiros segundos.
  Marque o primeiro `pointerdown` / `keydown` / `wheel` / `touchstart` com um
  ouvinte `once` e pare de assumir a partir dali.
- **As telas de conteúdo em curso ficam fora, sempre** — inclusive na porta. O
  alvo de compartilhamento abre JÁ com um arquivo que outro app mandou;
  recarregar ali perde exatamente aquilo.

**E lembre do custo do outro lado.** Se a sua versão de cache é apagada ao
ativar (e deve ser — HTML velho apontando para chunks velhos é o pior dos
mundos), a abertura logo depois da troca vai à rede. É uma abertura lenta por
publicação, contra a garantia de que ninguém fica para trás. Vale; só não
confunda essa abertura com a otimização de §8.4 tendo falhado.

### 8.6 O que roda na abertura não pode abrir janela

Este não é bem um item de desempenho — é um item de **abertura**, e ele entra
aqui porque só aparece depois que você conserta §8.4.

Todo app acumula um punhado de tarefas que rodam "no ocioso, quando o app
abre": sincronizar, aquecer cache, um backup automático, renovar uma permissão.
Enquanto o app demora três segundos para abrir, boa parte dessas tarefas nunca
chega a rodar — a pessoa já está fazendo outra coisa, ou fechou. **Quando a
abertura fica instantânea, todas elas passam a rodar, cedo e sempre.** É por
isso que uma otimização de abertura costuma vir acompanhada de um bug que
"apareceu do nada": ele estava lá, disparando de vez em quando.

A regra: **nada que rode sem um toque da pessoa pode abrir uma janela, um
popup, uma aba ou um diálogo do sistema.**

E o jeito mais fácil de violá-la é acreditar num nome. O caso que custou caro:

> O cliente de **token** do Google Identity Services abre uma janela SEMPRE.
> `prompt: "none"` pula a tela de consentimento — **não** a janela.

No computador ela pisca e fecha, então a suposição errada sobrevive anos. No
celular, com o app instalado, ela vira uma aba do navegador por cima do app,
com o cartão de carregamento do provedor, em toda abertura.

Dois agravantes que transformam "às vezes" em "sempre", e que valem para
qualquer tarefa periódica:

- **Só marque como feito o que de fato aconteceu** — e então cuide para que o
  fracasso não seja gratuito. Se a marca de "última execução" só é gravada no
  sucesso, uma tarefa que falha continua vencida e **tenta de novo em toda
  abertura, para sempre**. Ou a tentativa é barata e calada (o certo), ou
  precisa de uma espera crescente entre tentativas.
- **Uma automação que só funciona com autorização interativa não é uma
  automação.** Ou ela roda com uma credencial que renova sozinha (do lado do
  servidor), ou ela é oportunista — roda quando a credencial já está viva — e a
  interface diz a verdade sobre quando rodou pela última vez. O que não pode é
  o meio-termo: tentar interativamente sem ninguém pedindo.

**Como auditar isto no seu app**, e vale meia hora: liste tudo o que dispara no
`load` e no primeiro ocioso, e para cada item pergunte se ele pode abrir alguma
coisa — `window.open`, um cliente de OAuth, `requestPermission` de notificação
ou de geolocalização, um seletor de arquivo, `showInstallPrompt`. Toda API que
pede permissão ao sistema deve nascer de um gesto; o navegador até bloqueia
algumas fora do gesto, mas as que ele deixa passar são justamente as que
aparecem em cima do seu app.

O teste que segura isso é simples e vale a pena escrever: **transforme a janela
num espião**. Se o caminho automático encostar nela, o teste cai.

```ts
const requestAccessToken = vi.fn();
// … roda o caminho automático três vezes, como três aberturas do app
expect(requestAccessToken).not.toHaveBeenCalled();
```

Escreva-o de forma que ele **falhe com o código antigo** antes de dar por
resolvido — um teste que passaria dos dois jeitos não protege nada.

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

**E dê segunda chance.** Este é um defeito que só aparece em produção e é
insidioso: o efeito que assina roda quando a lista de caminhos visíveis muda. Se
a assinatura falha — a sessão do backend ainda subindo no primeiro segundo, a
rede piscando —, **nada muda depois**: nem os caminhos, nem o mapa de URLs. O
efeito não roda de novo, e a grade fica em retângulos vazios até a pessoa
recarregar a página. Trocar de aba "conserta", o que despista a investigação:
não é o foco que arruma, é a revalidação trocando a lista.

```ts
const tentativas = useRef(0);
const MAX = 5;

const reagendar = () => {
  if (tentativas.current >= MAX) return;
  const espera = 400 * 2 ** tentativas.current; // 0,4s · 0,8 · 1,6 · 3,2 · 6,4
  tentativas.current += 1;
  timer = setTimeout(tentar, espera);
};

const tentar = () => {
  void assinar(faltando).then(
    (mapa) => {
      if (!mapa.size) return reagendar();
      tentativas.current = 0; // veio algo: o teto zera
      acumularNoMapa(mapa);
    },
    () => reagendar(), // recusa também reagenda
  );
};
```

Duas sutilezas que fazem diferença: o teto zera **a cada resposta útil** (rolar
uma galeria inteira não gasta o limite — ele é do trecho que está falhando, não
da sessão), e o mapa **só cresce** (quem já foi assinado não é pedido de novo,
então a lista virtualizada pede só o que entrou na tela).

Ponha isso num hook único e use-o em toda tela que mostra imagem privada. Três
telas fazendo a mesma coisa de três jeitos é como o defeito sobrevive: corrigido
numa, continua nas outras.

### 9.4 `createObjectURL` é alocação: alguém precisa revogar

`URL.createObjectURL(blob)` prende o blob na memória do documento **até alguém
chamar `revokeObjectURL`**. Não há coleta de lixo para isso: a URL é uma
referência forte, e o navegador não sabe que você parou de usá-la.

Duas regras, e as duas foram aprendidas do jeito caro:

**1. Cache de URL nunca sobrescreve sem revogar.**

```ts
// ✗ Cada gravação por cima deixa o blob anterior vivo até a aba fechar.
cache.set(caminho, URL.createObjectURL(blob));

// ✓ Uma função, e todo mundo passa por ela.
function guardarUrlDeBlob(caminho: string, blob: Blob): string {
  const anterior = cache.get(caminho);
  if (anterior?.startsWith("blob:")) URL.revokeObjectURL(anterior);
  const url = URL.createObjectURL(blob);
  cache.set(caminho, url);
  return url;
}
```

O sintoma é o pior possível de diagnosticar: a aba vai ficando pesada ao longo
da sessão, sem nenhum pico, sem nenhum erro. Numa lista de avatares
pré-carregada a cada visita, são megabytes por sessão.

**2. A prévia local sai quando a definitiva entra — nem antes.**

Ao subir uma imagem, o padrão é mostrar um `blob:` local enquanto a rede
trabalha. O erro é revogá-lo no `finally` do upload: o caminho novo já está no
estado, mas a **URL assinada** dele ainda não chegou, e a imagem pisca para o
placeholder no meio.

Duas formas corretas, dependendo do caso:

- Guarde na prévia o caminho que ela virou, e **descarte-a quando a lista do
  servidor contiver esse caminho** — a troca acontece no mesmo render, sem vão.
- Ou, no caso simples de um avatar só, adie a revogação por um quadro
  (`requestAnimationFrame`), para o novo `src` já ter sido pintado.

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

### 11.5 O botão VOLTAR é o mais lento do app — e ninguém percebe

Telas de tela cheia (criar, editar, ler) costumam **não ter a navegação do app**.
Isso quer dizer que nada nelas chamou `prefetch` da tela de origem: o roteador
só começa a buscar a rota de destino no instante do toque. Três segundos de
botão parado, e a pessoa toca de novo.

Duas linhas resolvem:

```tsx
// Ao MONTAR a tela cheia, a rota de volta já vai ficando pronta.
useEffect(() => {
  router.prefetch("/");
}, [router]);
```

E o outro lado do mesmo problema: **um indicador de rota que só escuta clique em
`<a>` não cobre botão nenhum.** Voltar quase sempre é um `<button>` chamando
`router.back()`. Exponha um disparo manual e chame-o de dentro do botão:

```ts
export function comecarProgressoRota() {
  window.dispatchEvent(new CustomEvent("progresso-rota"));
}
```

Só acenda a barra quando o toque **de fato troca de rota**. Se ele apenas fecha
um painel ou volta uma etapa dentro da mesma tela, a barra fica pendurada
esperando uma navegação que não vem — e o indicador passa a mentir.

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
- [ ] O orçamento de bundle continua passando — **por rota**, não só o total.
- [ ] `staleTimes` do roteador está configurado; voltar não remonta a tela.
- [ ] Existe janela de frescor (20 s), e escrever a invalida.
- [ ] O middleware não paga um `getUser()` por navegação quando não há o que
      renovar — e as portas de entrada ficam fora do atalho.
- [ ] O documento de navegação **não** vai com `no-store` — senão não há
      bfcache, e voltar para o app recarrega tudo.
- [ ] Derivação cara de tela vive num memo de módulo, não num `useMemo` que
      morre na navegação.
- [ ] O service worker serve a casca guardada **na hora** na abertura, e
      revalida atrás — não fica segundos com a logo na tela perguntando à rede
      por um HTML que ele já tem.
- [ ] Nenhuma resposta `redirected` vai para o cache de HTML (senão o app abre
      na tela de entrar mesmo depois de entrar).
- [ ] O service worker é registrado no primeiro OCIOSO, não em `load` — o
      pré-cache não disputa banda com a hidratação.
- [ ] Quando o HTML pode vir do cache ou do bfcache, existe uma guarda de sessão
      no cliente — e ela não expulsa ninguém offline, nem recarrega o mesmo
      endereço (laço).
- [ ] A guarda de sessão decide pelo sinal ASSENTADO (`INITIAL_SESSION` /
      `SIGNED_OUT`), nunca por um `getSession()` solto na montagem — que
      responde `null` enquanto um token vencido está sendo renovado.
- [ ] A versão nova do worker é aplicada na PORTA (esperando na abertura, ou
      instalada antes do primeiro toque) e convidada no meio do uso — ninguém
      fica mais de uma abertura atrás.
- [ ] Nada que roda no `load` ou no primeiro ocioso abre janela, popup ou
      diálogo de permissão. Auditado item a item — e não pelo nome da opção
      (`prompt: "none"` não quer dizer "sem janela").
- [ ] Só as famílias e os pesos de fonte que a interface desenha de fato, e no
      `preload` só as que estão no primeiro quadro.
- [ ] Dois componentes dinâmicos que usam a mesma biblioteca pesada moram no
      mesmo módulo (senão são duas cópias).
- [ ] Revalidação que devolve o mesmo resultado **não** repinta a tela.
- [ ] Resposta obsoleta não pinta nem desliga o esqueleto da atual.
- [ ] Tela cheia faz `prefetch` da rota de volta ao montar.
- [ ] Botão que navega acende o indicador de rota (link não é o único caminho).
- [ ] Toda assinatura de URL tem repetição com espera dobrando.
- [ ] Nenhuma promessa fica sem `catch` num caminho que desliga carregamento.
- [ ] Todo `createObjectURL` tem dono: o cache revoga antes de sobrescrever, e a
      prévia local só é revogada quando a definitiva já está na tela.
- [ ] Depois de todo `await` que devolve um recurso do aparelho, há a pergunta
      "a tela ainda existe?" — e a devolução do recurso se ela não existir.

---

## 16. Diagnóstico: "o meu app demora para carregar as telas"

Na ordem. Pare quando achar — os quatro primeiros respondem pela quase
totalidade dos casos, e nenhum deles é "otimizar o React".

**1. A navegação remonta tudo?**
Abra uma tela, vá para outra e volte. Se voltar mostra esqueleto, o roteador
está descartando a rota. → §4.5. É a correção de maior efeito e a mais barata:
uma linha de configuração e um `Map` em módulo.

**2. A tela abre cheia e "recarrega" um segundo depois?**
Três causas, nesta ordem de frequência: revalidação repintando resultado
idêntico (§4.8), o detalhe refazendo a busca a cada aviso do repositório (§4.7,
janela de 60 s), e transição de view rodando em toda revalidação (§4.8).

**3. Quanto pesa a ROTA, não o app?**
Meça por rota (§2.1). Um único `import` no topo de um arquivo pode custar
200 KB numa tela que nunca usa aquilo. → §2.2.

**3-b. Voltar para o app recarrega tudo?**
Não é o sistema descartando a aba: é `no-store` no documento desligando o
bfcache. → §7.7. Uma linha no middleware.

**3-b-2. Depois de acelerar a abertura, apareceu um popup/aba do nada?**
Não apareceu: ele já existia e agora dá tempo de disparar. Tarefas de ocioso
que antes quase nunca rodavam passam a rodar em toda sessão. → §8.6.

**3-c. A ABERTURA demora, mas a navegação já está rápida?**
São problemas diferentes e não se resolvem no mesmo lugar. Se o app instalado
fica segundos na tela de abertura do sistema, quase sempre é o service worker
indo perguntar à rede antes de servir o HTML que ele já tem guardado — mais a
portaria e o pré-cache na frente da hidratação. → §8.4. A pista que separa este
caso de todos os outros: o app é rápido **depois** que abre.

**4. O primeiro toque de cada sessão demora?**
Alguma checagem síncrona de servidor está no caminho do clique — permissão,
plano, sessão. Leia do espelho local (síncrono) para decidir a interface, e
confirme com o servidor atrás. Guarde a última confirmação por 60 s.

**5. A lista trava ao rolar ou ao digitar?**
Virtualize (§5.1) e isole o campo (§6.1). Nessa ordem: virtualizar resolve a
rolagem, isolar resolve a digitação, e um não substitui o outro.

**6. As imagens chegam tarde ou não chegam?**
Miniatura própria (§9.1), tamanho declarado (§9.2), e **repetição na
assinatura**: se a URL assinada falha porque a sessão ainda está subindo, nada
muda depois e a foto fica em branco até recarregar. Reagende com espera
dobrando.

**7. Só então**: profile de render, memoização, workers.

### O erro de método mais comum

Otimizar o que é fácil de medir (o tamanho do JavaScript) em vez do que a pessoa
sente (o tempo entre o toque e a tela). Um app com 400 KB que remonta a tela a
cada navegação parece mais lento do que um de 1,2 MB que devolve a tela pronta.
**Meça o vaivém real**: abra a lista, entre num item, volte, entre em outro,
volte. É esse ciclo que define a impressão de velocidade.
