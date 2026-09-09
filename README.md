# Base para aplicativos

Três documentos que descrevem, por inteiro, como um aplicativo web deve ser
desenhado, como deve responder e como deve ser operado. Eles saíram de um app
real em produção, mas nada aqui depende do assunto dele.

A ideia é simples: **aponte uma IA para este repositório e peça o app que você
quer**. Ela recebe o sistema pronto — cor, tipografia, movimento, componentes,
navegação, cache, offline, PWA, deploy — e gasta o esforço no que é seu, que é o
conteúdo.

| Documento                              | O que responde                                        |
| -------------------------------------- | ----------------------------------------------------- |
| [`design-system.md`](design-system.md) | Como o app **parece** e como ele **reage** ao toque   |
| [`desempenho.md`](desempenho.md)       | Como o app **responde**: cache, listas, offline, rede |
| [`operacao.md`](operacao.md)           | Como o app **existe**: banco, login, PWA, deploy, CI  |

> **Já tem um app e ele está lento?** Vá direto para
> [`desempenho.md` §16](desempenho.md#16-diagnóstico-o-meu-app-demora-para-carregar-as-telas):
> é um roteiro na ordem certa, e os quatro primeiros itens respondem pela quase
> totalidade dos casos. O nº 1 — o cache do roteador — costuma ser uma linha de
> configuração.

---

## Como usar

Cole isto na conversa com a IA, trocando o que está entre colchetes:

> Use como base o repositório https://github.com/pedroydzito/appdocs — leia os
> três documentos (`design-system.md`, `desempenho.md`, `operacao.md`) por
> inteiro antes de escrever qualquer código, e siga-os à risca.
>
> Quero construir: **[descreva o app em duas ou três frases: para quem é, o que
> a pessoa faz nele, qual é a ação principal]**.
>
> - Cor de acento: **[#hexadecimal, ou "escolha e justifique"]**
> - Nome do app: **[nome]** (curto: **[até 12 caracteres]**)
> - Destinos da navegação: **[4 a 8 itens]**
> - Ação principal (o botão "+"): **[o que ele cria]**
> - Categorias coloridas, se houver: **[lista]**
> - Precisa de: **[login / offline / instalável / notificação / IA / upload de
>   imagem / gravação de voz]**
>
> Siga a §14 do design system para adaptar o acento e as categorias. Monte a
> página `/estilo` com os quatro temas. Não invente token, componente nem
> duração fora do que está especificado.

Se algum item ficar em branco, a IA deve **perguntar antes de começar** — não
adivinhar.

---

## O que muda de app para app

| Muda                                            | Onde está a receita        |
| ----------------------------------------------- | -------------------------- |
| Cor de acento e os onze degraus                 | design system §14.1        |
| Categorias coloridas (se houver)                | design system §3.4 e §14.2 |
| Destinos e ação principal da navegação          | design system §14.3        |
| Nome, ícones, atalhos, alvo de compartilhamento | operação §4                |
| Telas e conteúdo                                | você                       |

### Se a ideia é refazer ESTE app com outra cor

É o caso mais fácil, e é literalmente três coisas:

1. **A escala do acento** — onze degraus derivados de um hexadecimal (design
   system §14.1). Troque o 500 e derive o resto; a receita está lá.
2. **Os quatro temas** — só o que depende do acento muda. Fundo, superfícies,
   bordas e texto continuam iguais nos dois temas, e o alto contraste continua
   sendo um modificador combinável.
3. **`--perigo`** — se o novo acento **não** for avermelhado, aí sim o perigo
   volta a ser vermelho por conta própria (§3.2 explica por que, com acento
   coral, ele é o próprio acento a 20° de matiz).

Depois disso, rode o teste de contraste (§3.6) — ele reprova o par que cair
abaixo de AA — e olhe a página `/estilo`, que mostra todo componente nos quatro
temas lado a lado. Nada mais precisa ser tocado: nenhum componente conhece a
cor, todos leem o token.

## O que **não** muda

Se mexer nisto, o resultado deixa de parecer o mesmo produto:

- Título de tela em caixa baixa, com a display de tracking apertado.
- Fundo quente no tema claro, preto de verdade no escuro, alto contraste como
  modificador combinável com os dois.
- Um véu de hover só, seguindo a luz do tema.
- Duas durações e duas curvas de movimento — e tudo morrendo sob
  `prefers-reduced-motion`.
- Cartão com raio de 1.25rem; botão, campo e busca com 0.75rem.
- Chip como pílula com fundo tingido em 5%.
- Barra de navegação como pílula flutuante com desfoque no celular, coluna fixa
  no desktop.
- Contraste cobrado por teste, quebrando o build.

---

## Ordem de trabalho sugerida

1. **Tokens e temas** (`globals.css`) — os quatro temas completos, mais o teste
   de contraste. Nada mais começa antes disto.
2. **Componentes** (`components/ui/`) — botão, campo, chip, folha, nota,
   esqueleto, estado vazio, chave, trilho.
3. **Página `/estilo`** — os quatro temas e todo componente lado a lado. É onde
   um desvio aparece antes de virar dívida.
4. **Moldura** — casca, navegação nos dois formatos, animação de rota.
5. **Backend** — banco com RLS, login, arquivos (operação §6–§8).
6. **Telas** do app, usando só o que existe nos passos 2 e 4.
7. **PWA** — manifesto, ícones, service worker (operação §4).
8. **CI e deploy** (operação §12–§13).

Ao final, os três checklists — o do design system §15, o do desempenho §15 e o
de lançamento em operação §18 — precisam passar item a item.

---

## Como estes documentos foram escritos

Cada regra aqui tem um custo pago atrás dela. Onde o texto explica **por que** a
decisão é aquela, é porque a alternativa foi tentada e falhou — o segundo tom de
cor para texto que estragava a tela, o `filter` de hover que ia para o lado
errado no tema escuro, o cache de 5 minutos que devolvia a tela ao esqueleto no
meio da sessão, a coluna de embeddings que era 86% do tráfego da tela inicial.

Não trate as justificativas como enfeite: elas são o que impede a decisão de ser
desfeita por engano seis meses depois.
