# Operação — do primeiro `npm install` ao app instalado no celular

O `design-system.md` diz como o app **parece**, o `desempenho.md` diz como ele
**responde**. Este diz como ele **existe**: variáveis de ambiente, banco,
autenticação, arquivos, tarefas agendadas, notificação, instalação como
aplicativo, integração contínua, publicação e o que fazer quando algo quebra.

Como os outros dois, é uma receita. Os valores concretos vêm de um app em
produção; a estrutura serve para qualquer um.

**Índice**

1. [A pilha de referência](#1-a-pilha-de-referência)
2. [Variáveis de ambiente](#2-variáveis-de-ambiente)
3. [Rodar na sua máquina](#3-rodar-na-sua-máquina)
4. [Tornar o app instalável (PWA)](#4-tornar-o-app-instalável-pwa)
5. [Funcionar sem internet](#5-funcionar-sem-internet)
6. [Login e isolamento de dados](#6-login-e-isolamento-de-dados)
7. [Banco de dados e migrações](#7-banco-de-dados-e-migrações)
8. [Arquivos do usuário](#8-arquivos-do-usuário)
9. [Rotas de API](#9-rotas-de-api)
10. [Tarefas agendadas](#10-tarefas-agendadas)
11. [Notificações](#11-notificações)
12. [Integração contínua](#12-integração-contínua)
13. [Publicar](#13-publicar)
14. [Versão e release](#14-versão-e-release)
15. [Observar e diagnosticar](#15-observar-e-diagnosticar)
16. [Segredos e troca de chaves](#16-segredos-e-troca-de-chaves)
17. [Backup, exportação e apagar a conta](#17-backup-exportação-e-apagar-a-conta)
18. [Checklist de lançamento](#18-checklist-de-lançamento)

---

## 1. A pilha de referência

| Camada        | Escolha                   | Trocável por                              |
| ------------- | ------------------------- | ----------------------------------------- |
| Framework     | Next.js (App Router)      | qualquer um com SSR e rotas de API        |
| Hospedagem    | Vercel                    | Netlify, Fly, contêiner próprio           |
| Banco + auth  | Supabase (Postgres + RLS) | Neon + Auth.js, Firebase, backend próprio |
| Arquivos      | Supabase Storage          | S3, R2                                    |
| IA (opcional) | Gemini                    | qualquer provedor com API de texto        |
| Instalação    | PWA (manifest + SW)       | —                                         |

Duas escolhas merecem justificativa, porque moldam o resto:

- **Banco com segurança por linha (RLS)** em vez de checagem no código. A regra
  "cada um só vê o que é seu" mora no banco, uma vez, e vale para toda consulta —
  inclusive as que você vai escrever daqui a seis meses e esquecer de filtrar.
- **PWA em vez de app nativo.** Um código, instalação sem loja, atualização sem
  revisão. O preço está na §4.6: o que um PWA ainda não faz.

---

## 2. Variáveis de ambiente

### 2.1 Declare uma vez, com gravidade

O erro clássico é cada arquivo ler a variável do seu jeito — `!` (que mente para
o compilador e estoura em produção), `?? "padrão"` ou um `Boolean(...)` que
devolve tela vazia sem dizer por quê. Uma variável esquecida vira "comportamento
estranho". No app de referência isso custou meses de lembrete que nunca
disparava, porque faltava um segredo e nada dizia isso em lugar nenhum.

Faça uma lista única, tipada, com **nome, para que serve e o que se perde sem
ela**, em três gravidades:

| Gravidade   | Significado                                       |
| ----------- | ------------------------------------------------- |
| `essencial` | sem isto o app não sobe                           |
| `recurso`   | sem isto um recurso inteiro some, o resto anda    |
| `servidor`  | só no servidor, e só para tarefas administrativas |

No boot, valide a lista e **imprima o que faltou, com a consequência**. Nada
disso derruba o app: um diário sem IA continua sendo um diário. O que não pode é
falhar em silêncio.

### 2.2 O conjunto típico

| Variável                         | Gravidade | Sem ela                               |
| -------------------------------- | --------- | ------------------------------------- |
| `NEXT_PUBLIC_<BACKEND>_URL`      | essencial | nada carrega                          |
| `NEXT_PUBLIC_<BACKEND>_ANON_KEY` | essencial | nada carrega                          |
| `<BACKEND>_SECRET_KEY`           | servidor  | rotas administrativas e scripts param |
| `<IA>_API_KEY`                   | recurso   | o app cai num modo degradado          |
| `NEXT_PUBLIC_VAPID_PUBLIC_KEY`   | recurso   | sem notificação                       |
| `VAPID_PRIVATE_KEY`              | recurso   | sem notificação                       |
| `CRON_SECRET`                    | recurso   | as tarefas agendadas voltam 401       |

### 2.3 Onde elas vivem

**Dois lugares, e só dois**: `.env.local` na máquina de quem desenvolve (fora do
git) e as variáveis do projeto na hospedagem. Um terceiro lugar é onde a
divergência nasce.

Mantenha um `.env.example` **comentado** no repositório, com todas as chaves
vazias e uma linha dizendo para que cada uma serve. É a documentação que ninguém
esquece de atualizar, porque sem ela o app novo não sobe.

Regra de ouro: o prefixo público (`NEXT_PUBLIC_`) significa **está no bundle do
navegador**. A chave anônima do backend é pública por definição e não é segredo
— quem protege os dados é o RLS. A chave secreta ignora o RLS e **nunca** pode
aparecer num arquivo com `"use client"`.

---

## 3. Rodar na sua máquina

```bash
npm install
cp .env.example .env.local   # preencha
npm run dev
```

O conjunto mínimo de scripts que todo app deste tipo precisa ter:

| Script                    | O que faz                                     |
| ------------------------- | --------------------------------------------- |
| `dev`                     | servidor local                                |
| `build` / `start`         | produção                                      |
| `typecheck`               | tipos, sem emitir                             |
| `lint`                    | ESLint                                        |
| `format` / `format:check` | Prettier (escrever / conferir)                |
| `test`                    | testes unitários                              |
| `e2e`                     | testes de ponta a ponta                       |
| `conferir:bundle`         | orçamento de tamanho (ver `desempenho.md` §2) |
| `conferir:schema`         | código e banco batem                          |
| `tipos`                   | regera os tipos do banco                      |
| `testar:rls`              | duas contas de verdade, uma não vê a outra    |
| `icones`                  | gera os ícones do PWA                         |

**Antes de qualquer commit**: `format` → `typecheck` → `lint` → `test`. Se o
projeto adota versionamento semântico manual, suba a versão no `package.json`
**nesse momento**, não a cada mudança.

---

## 4. Tornar o app instalável (PWA)

São cinco peças. Faltando qualquer uma, o navegador não oferece a instalação.

### 4.1 O manifesto

`public/manifest.webmanifest`, referenciado no metadata da raiz
(`manifest: "/manifest.webmanifest"`):

```json
{
  "name": "Nome Completo do App",
  "short_name": "Nome",
  "description": "Uma frase sobre o que o app faz.",
  "start_url": "/",
  "scope": "/",
  "display": "standalone",
  "orientation": "portrait",
  "background_color": "#f5f4f0",
  "theme_color": "#f5f4f0",
  "lang": "pt-BR",
  "dir": "ltr",
  "categories": ["productivity"],
  "icons": [
    { "src": "/icones/icone-192.png", "sizes": "192x192", "type": "image/png", "purpose": "any" },
    { "src": "/icones/icone-512.png", "sizes": "512x512", "type": "image/png", "purpose": "any" },
    {
      "src": "/icones/icone-mascarado-512.png",
      "sizes": "512x512",
      "type": "image/png",
      "purpose": "maskable"
    }
  ]
}
```

O que cada campo decide, na prática:

- **`short_name`** é o que cabe embaixo do ícone na tela inicial. Passou de ~12
  caracteres, o sistema corta.
- **`display: "standalone"`** tira a barra do navegador. É o campo que faz
  parecer aplicativo.
- **`scope`** delimita o que abre dentro do app; um link fora do escopo abre no
  navegador.
- **`background_color`** é a cor da tela de abertura, antes de o app pintar.
  Use o **mesmo** `--fundo` do tema claro, senão a abertura pisca.
- **`theme_color`** tinge a barra de status. Declare também no viewport, com
  variação por tema:

```ts
export const viewport = {
  width: "device-width",
  initialScale: 1,
  viewportFit: "cover", // necessário para as áreas seguras
  themeColor: [
    { media: "(prefers-color-scheme: light)", color: "#f5f4f0" },
    { media: "(prefers-color-scheme: dark)", color: "#0a0a0a" },
  ],
};
```

### 4.2 Ícones

| Arquivo                   | Tamanho | `purpose`  | Para quê                            |
| ------------------------- | ------- | ---------- | ----------------------------------- |
| `icone-192.png`           | 192     | `any`      | mínimo exigido para instalar        |
| `icone-512.png`           | 512     | `any`      | tela de abertura, lojas de PWA      |
| `icone-mascarado-512.png` | 512     | `maskable` | Android recorta em círculo/squircle |
| `apple-icon.png`          | 180     | —          | iOS (via metadata `icons.apple`)    |
| `favicon` / `icon.png`    | 32/48   | —          | aba do navegador                    |

O **maskable** é o mais esquecido: o Android recorta o ícone na forma do sistema
e come as bordas. Desenhe com a arte dentro dos 80% centrais — fora disso, o
ícone aparece cortado só em alguns aparelhos, que é o pior tipo de bug visual.

Gere os ícones por script a partir do símbolo da marca e da cor de acento
(`npm run icones`), em vez de exportar à mão. Trocar o acento passa a ser trocar
uma constante.

### 4.3 Metadados de iOS

O iPhone ignora metade do manifesto e usa os seus próprios:

```ts
appleWebApp: {
  capable: true,
  statusBarStyle: "black-translucent",  // o app pinta atrás da barra
  title: "Nome",
},
formatDetection: { telephone: false },  // impede o iOS de virar número em link
```

Com `black-translucent` + `viewportFit: "cover"`, o conteúdo passa **por baixo**
do entalhe e da barra inferior — e aí as áreas seguras deixam de ser detalhe:

```css
.pt-segura {
  padding-top: env(safe-area-inset-top);
}
.pb-segura {
  padding-bottom: env(safe-area-inset-bottom);
}
```

Toda barra fixa no rodapé precisa da folga de baixo, e o conteúdo precisa
reservar a altura dela (ver `design-system.md` §9.6).

### 4.4 Service worker

Sem service worker o navegador não oferece instalação. Registre **só em
produção** e depois do `load` — em desenvolvimento ele serve arquivo velho e
custa uma hora de confusão por semana:

```tsx
useEffect(() => {
  if (!("serviceWorker" in navigator)) return;
  if (process.env.NODE_ENV !== "production") return;
  const registrar = () =>
    navigator.serviceWorker.register("/sw.js", { scope: "/" }).catch(console.warn);
  if (document.readyState === "complete") registrar();
  else window.addEventListener("load", registrar, { once: true });
}, []);
```

Sirva o próprio `/sw.js` com `Cache-Control: no-cache, no-store,
must-revalidate`. Um service worker cacheado é um app congelado numa versão
antiga, e não há botão de atualizar que resolva.

As estratégias de cache estão em `desempenho.md` §8.1. O que é **operação**:

- **Versione o cache** (`app-v8`) e apague os antigos no `activate`. Suba a
  versão sempre que um arquivo pré-cacheado mudar — foi o que já segurou um
  ícone antigo do PWA por semanas.
- **`skipWaiting()`** no install faz a versão nova assumir sem esperar o
  fechamento de todas as abas. Bom para app pessoal; pense duas vezes se você
  tem estado longo aberto na tela.
- Guarde uma página `/offline.html` no pré-cache e sirva-a quando a navegação
  falhar. Ela não pode depender de nada da rede.

### 4.5 Atalhos e receber compartilhamento

Dois campos do manifesto que fazem o PWA parecer nativo:

```json
"shortcuts": [
  { "name": "Criar agora", "short_name": "Criar", "url": "/nova?modo=texto",
    "icons": [{ "src": "/icones/icone-192.png", "sizes": "192x192" }] }
],
"share_target": {
  "action": "/receber",
  "method": "POST",
  "enctype": "multipart/form-data",
  "params": {
    "title": "titulo", "text": "texto", "url": "url",
    "files": [{ "name": "arquivos", "accept": ["image/*", "audio/*", "application/pdf"] }]
  }
}
```

Os **atalhos** aparecem ao segurar o ícone. Coloque de dois a três, e só ações
que valem por si.

O **alvo de compartilhamento** põe o app na folha "Compartilhar com…" do
sistema. Com `method: "POST"` e arquivos, o destino recebe um `multipart/form-data`
— e aqui mora a pegadinha: em vários navegadores esse POST é entregue **ao
service worker**, não ao servidor. O padrão que funciona é interceptar o POST no
service worker, guardar os arquivos num cache temporário e responder com um
redirecionamento 303 para uma tela que os lê de lá.

### 4.5-b Receber arquivo no COMPUTADOR — três caminhos, e o primeiro não existe

Vale escrever com todas as letras porque custa meio dia descobrir: **o Web Share
Target é do Android (e do ChromeOS). No macOS e no Windows, nenhum app web
aparece na folha de compartilhar do sistema.** Não há configuração que resolva.

O que existe lá, em ordem de esforço:

**1. Arrastar e soltar na janela.** Funciona em qualquer navegador, sem
instalar nada, e é o caminho mais curto. Escute na `window`, não numa área:

```ts
const temArquivo = (e: DragEvent) => Array.from(e.dataTransfer?.types ?? []).includes("Files");

// `dragenter`/`dragleave` chegam aninhados: conte, não use booleano.
let dentro = 0;
window.addEventListener("dragenter", (e) => {
  if (temArquivo(e)) {
    dentro++;
    acender();
  }
});
window.addEventListener("dragleave", (e) => {
  if (temArquivo(e) && --dentro === 0) apagar();
});
// Sem `preventDefault` no dragover, o navegador ABRE o arquivo no lugar da página.
window.addEventListener("dragover", (e) => {
  if (temArquivo(e)) e.preventDefault();
});
```

Se a pessoa já está na tela que recebe arquivos, **avise em vez de navegar**:
trocar de rota ali pode matar uma gravação em andamento.

**2. "Abrir com" no Finder / soltar no ícone do Dock.** É o `file_handlers` do
manifesto, para o app instalado no Chrome/Edge:

```json
"file_handlers": [
  { "action": "/nova?modo=documento",
    "accept": { "audio/*": [".mp3", ".m4a"], "image/*": [".jpg", ".png"], "application/pdf": [".pdf"] },
    "launch_type": "single-client" }
]
```

O arquivo chega por `launchQueue.setConsumer(({ files }) => …)`, como
`FileSystemFileHandle`. **Só vale para o app instalado**, e a associação é feita
no momento da instalação — quem já tinha o app instalado antes precisa
reinstalar para o sistema registrar os tipos.

**3. Parâmetros na URL.** O caminho que nenhum sistema bloqueia: aceite
`?texto=`, `?titulo=` e `?url=` na tela de criação. Com isso, um atalho do app
Atalhos (macOS/iOS), um bookmarklet ou qualquer automação consegue mandar o
trecho selecionado para dentro do app — porque _abrir uma URL_ todo sistema
sabe fazer.

Passe o texto pelo **mesmo saneador** do share do celular: é conteúdo de fora
entrando no editor. Escape o HTML e só transforme em link o que for `http(s)` —
`javascript:` num `href` é a forma mais antiga que existe de transformar
"compartilhar" em "executar".

E aceite-o como **estado inicial**, não num efeito: a tela nasce escrita, sem o
quadro em branco antes de o parágrafo aparecer.

### 4.5-c O app instalado serve uma versão antiga

Sintoma que parece bug do app e não é: no PWA instalado, um recurso não funciona
— e no navegador, funciona. Quase sempre é o service worker servindo uma casca
antiga, presa desde a instalação.

Duas providências: (1) o aviso de atualização precisa existir na interface (§4.4)
e (2) quando alguém relatar algo assim, a primeira pergunta é "no navegador
também acontece?". Se não acontece, reinstalar o app resolve — e o defeito real
é o seu ciclo de atualização, não o recurso.

### 4.6 O que o navegador exige, e o que ele ainda não dá

Para o prompt de instalação aparecer: **HTTPS** (ou `localhost`), manifesto
válido com `name`, `short_name`, `start_url`, `display` e ícones de 192 e 512, e
um service worker registrado com handler de `fetch`.

O que um PWA **não** faz, e é melhor saber antes de prometer: alarme em horário
exato com o app fechado, acesso a contatos/SMS, execução longa em segundo plano
e, no iOS, notificação push só a partir do iOS 16.4 **e só depois de a pessoa
instalar o app na tela inicial**.

### 4.7 Como testar a instalação

| Plataforma       | Caminho                                                         |
| ---------------- | --------------------------------------------------------------- |
| Android / Chrome | menu → "Instalar aplicativo"; DevTools → Application → Manifest |
| iOS / Safari     | Compartilhar → "Adicionar à Tela de Início" (**só no Safari**)  |
| Desktop          | ícone de instalar na barra de endereço                          |

Depois de instalar, confira sempre: o ícone certo (não o antigo), a cor da tela
de abertura, o app abrindo **sem** barra de navegador, as áreas seguras no
aparelho com entalhe, e o app abrindo em modo avião.

---

## 5. Funcionar sem internet

Três camadas, cada uma resolvendo uma coisa (detalhe técnico em
`desempenho.md` §8):

1. **Service worker** — a casca: HTML, JS, CSS e imagens já vistas.
2. **Banco local** (IndexedDB) — rascunhos, arquivos binários e a fila.
3. **Fila de envio** — o que foi criado offline, com no máximo 5 tentativas.

A regra de produto: **nada do que a pessoa escreveu pode depender da rede para
existir**. Salve primeiro no aparelho, mostre na interface com uma marca de
pendente e suba depois. Item na fila aparece na tela — some da tela só quando
subir de verdade.

Teste isto a cada release, no aparelho, com o modo avião ligado: abrir o app,
criar algo, fechar o app, religar a rede, conferir que subiu.

---

## 6. Login e isolamento de dados

### 6.1 Sessão em cookie, não em `localStorage`

Use a integração de servidor do provedor de auth (no Supabase, o pacote `ssr`),
para que o middleware, os componentes de servidor e o navegador enxerguem a
**mesma** sessão. Sessão em `localStorage` não existe para o servidor, e aí toda
rota protegida vira uma tela que pisca.

No middleware/proxy: valide o token, renove quando preciso e redirecione quem
não tem sessão — **menos** nas rotas públicas (entrar, cadastrar, página
offline). Nas rotas de API, responda **401** em vez de redirecionar: um `fetch`
que recebe HTML de login é um erro de parse difícil de ler.

E deixe o app passar quando o backend não estiver configurado. Sem isso, quem
clona o repositório sem credenciais vê uma tela morta em vez de uma mensagem.

### 6.2 A regra mora no banco

Ative RLS em **toda** tabela com dado de usuário, e escreva as políticas de
`select`, `insert`, `update` e `delete` comparando o dono da linha com o usuário
da sessão. Feito isso, o cliente nunca precisa filtrar por usuário — e não
filtrar deixa de ser um vazamento.

### 6.3 Prove que funciona

Um teste que cria **duas contas de verdade**, grava dado em uma e tenta ler pela
outra, esperando zero linhas. Ele roda no CI, em job separado, depois do resto
passar (§12). É o único teste que prova a promessa central do produto, e o único
que não dá para deduzir lendo o código.

---

## 7. Banco de dados e migrações

### 7.1 Migrações idempotentes, na ordem, versionadas

Um arquivo `schema.sql` com o estado inicial e migrações numeradas
(`migracao-002.sql`, …). **Tudo idempotente** (`create ... if not exists`,
`drop policy if exists` antes de criar): a migração vai ser rodada de novo,
inteira, por alguém que não sabe onde parou.

Mantenha no `README` do diretório uma tabela com o número, o que a migração faz
e se já foi aplicada. Sem essa tabela, ninguém sabe o estado do banco de
produção — e essa é a informação mais cara de recuperar.

Duas armadilhas que já custaram caro:

- O editor SQL roda o arquivo inteiro **numa transação**. Um erro no passo 5
  desfaz os quatro primeiros, e você fica achando que aplicou.
- `create or replace function` **falha** quando o tipo de retorno muda
  (`42P13`). Derrube a função antes de recriar.

### 7.2 Garanta que o código e o banco batem

Um script (`conferir:schema`) que lê as colunas que o código grava e lê, compara
com o schema real e falha se faltar alguma. Escrito depois de a divergência
quebrar o app duas vezes.

Complemente com **tipos gerados** do banco (`npm run tipos`) e um teste que
falha quando o arquivo gerado sai de sincronia.

### 7.3 Tolerância a coluna ausente

A camada de acesso sabe seguir sem um conjunto declarado de colunas opcionais:
se o banco recusar, o registro é salvo sem elas em vez de se perder. É rede de
segurança para quem ainda não aplicou a migração — **não** substitui aplicá-la.

### 7.4 Nunca teste migração em produção

O ambiente de pré-visualização usa o **mesmo banco** que produção, salvo se você
montar outro. Migração nova se testa num projeto separado. Vale repetir porque a
pré-visualização parece um ambiente isolado e não é.

---

## 8. Arquivos do usuário

- Um **bucket por natureza de arquivo** (áudio, foto, anexo), todos privados.
- Política de acesso pelo caminho: o primeiro segmento é o id do dono.
- Nada de URL pública: gere **URL assinada** com validade (24h serve para mídia
  pessoal), em lote, com cache — ver `desempenho.md` §9.3.
- Comprima no cliente antes de subir e guarde uma miniatura ao lado do arquivo
  de exibição (`desempenho.md` §9.1).
- Ao apagar um registro, apague os arquivos dele. Storage órfão é a linha da
  fatura que ninguém entende seis meses depois.
- Se houver lixeira, a faxina do que passou do prazo é tarefa agendada (§10).

---

## 9. Rotas de API

- **Declare o tempo limite** de cada rota, proporcional ao trabalho: 30s para
  busca, 60s para operação comum, 120s para geração longa, 300s para lote. Sem
  declarar, ela morre no teto padrão da plataforma, no meio do trabalho.
- **Valide a entrada com esquema** (Zod ou equivalente), inclusive o tamanho.
  Limite de tamanho é limite de custo.
- **Limite a taxa** por rota e por usuário, com perfis diferentes para rota
  barata e rota cara; responda 401/429 com `Retry-After`.
- **Respostas longas em stream**, com `Cache-Control: no-store` e
  `X-Accel-Buffering: no`.
- Escolha o runtime com consciência: Node onde precisar de SDK e de tempo; edge
  onde precisar de latência e não precisar de biblioteca pesada.
- Nunca registre conteúdo do usuário nem chave em log (§15).

---

## 10. Tarefas agendadas

Na Vercel, `vercel.json`:

```json
{
  "crons": [
    { "path": "/api/lembrete", "schedule": "0 22 * * *" },
    { "path": "/api/manutencao", "schedule": "30 4 * * *" }
  ]
}
```

Regras:

- **Proteja a rota** com um segredo comparado ao cabeçalho `Authorization`. Rota
  de cron aberta é um botão de "faça isso mil vezes" na internet.
- O horário é **UTC**. `0 22 * * *` é 19h em Brasília — e muda quando o fuso
  muda de horário de verão.
- **Cron não roda em pré-visualização**, só em produção. Para testar, chame a
  rota à mão com o segredo.
- Toda tarefa precisa ser **idempotente** e registrar quanto trabalhou: a
  plataforma pode repetir a execução.
- Candidatas típicas: lembrete diário, resumo semanal, esvaziar lixeira vencida,
  limpar arquivo órfão, reindexar o que ficou para trás.

---

## 11. Notificações

Web Push, com um par de chaves VAPID (`NEXT_PUBLIC_VAPID_PUBLIC_KEY` e
`VAPID_PRIVATE_KEY`).

Fluxo: a pessoa aceita → o navegador devolve uma inscrição → você guarda no
banco, junto do fuso e do horário preferido → a tarefa agendada percorre as
inscrições e dispara.

O que dói:

- **Peça a permissão no momento certo**, depois de a pessoa ligar o lembrete —
  nunca na abertura. Permissão negada não se pede de novo.
- **A chave pública é parte da inscrição.** Trocá-la invalida **todas** as
  inscrições existentes, e cada pessoa precisa reativar o lembrete na mão. Só
  troque com motivo.
- Inscrição morta responde 404/410: **apague** ao receber, ou a base cresce com
  lixo e cada disparo fica mais lento.
- No iOS, só funciona depois de instalar o app na tela inicial (§4.6).

---

## 12. Integração contínua

Um workflow no push para `main` e em todo pull request, com cancelamento do
anterior no mesmo branch. Os passos, nesta ordem (o mais barato primeiro):

1. `typecheck`
2. `lint` — **com teto de avisos** (`--max-warnings N`). Teto é o que impede a
   conta de crescer sem ninguém olhar; quando subir, o commit explica por quê.
3. `format:check`
4. `test`
5. `build` — precisa das chaves **públicas** nos segredos do repositório, senão
   o pré-render das páginas que criam o cliente do backend falha
6. `conferir:bundle` — o orçamento de tamanho

E um **job separado**, depois do primeiro passar, para o teste de isolamento
entre contas (§6.3). Ele só roda no repositório de origem: um pull request de
fork não recebe segredos, e falharia por falta de chave, não por falha de
isolamento.

### 12.1 "Preciso mesmo disso? Meus outros projetos só têm o repo e a Vercel."

Pergunta legítima, e a resposta honesta é: **a Vercel só responde uma pergunta —
"compila?"**. Ela não sabe se o app ficou mais lento, se uma tradução sumiu, se
uma conta passou a enxergar os dados da outra. Se o build passar, ela publica.

O que cada passo compra, e o que acontece sem ele:

| Passo             | O que ele pega                                | Sem ele                                                        |
| ----------------- | --------------------------------------------- | -------------------------------------------------------------- |
| `typecheck`       | campo renomeado, chave de dicionário faltando | a tela quebra no aparelho de quem usa, não no seu              |
| `lint` (com teto) | efeito sem dependência, hook condicional      | bug de render que só aparece em condição de corrida            |
| `format:check`    | formatação divergente                         | diff de 300 linhas onde mudou uma                              |
| `test`            | a regra de negócio que você quebrou sem ver   | descoberto pelo usuário                                        |
| `build`           | erro de pré-render, variável faltando         | deploy quebrado — a Vercel também pega, mas depois de publicar |
| `conferir:bundle` | a tela que engordou 15% num commit            | **nada**: o app fica lento um pouco por vez, e ninguém nota    |
| isolamento (RLS)  | uma conta lendo os dados da outra             | vazamento de dados                                             |

Os dois últimos são os que **nenhuma plataforma faz por você**, e são os dois
mais caros de descobrir tarde. O de bundle é o mais subestimado: nenhum commit
deixa o app lento — cem commits deixam, e sem um teto que reprova, não existe o
dia em que alguém decide reagir.

**A versão mínima que já vale a pena**, se o projeto é pequeno: `typecheck` +
`build`. São dez linhas de YAML e cobrem a maior parte do estrago. O resto entra
quando o app começar a ter usuário que não é você.

**O que o CI não é:** ele não substitui rodar o app. Nenhum teste aqui percebe
que o espaçamento ficou feio ou que o texto está confuso. Ele existe para você
não gastar atenção com o que a máquina confere melhor — e sobrar atenção para o
que só uma pessoa vê.

### 12.2 O teto de avisos, e por que não zero

Um lint com zero aviso vira um lint que ninguém pode adicionar regra nova, e
acaba desligado. Um lint sem teto vira um lint que ninguém lê. O meio-termo é
`--max-warnings N` com o N **escrito no workflow, com um comentário dizendo o
que são aqueles avisos** e por que são aceitáveis. Quando o número sobe, o
commit precisa dizer por quê — e essa frase é a revisão.

---

## 13. Publicar

### 13.1 Ambientes

| Ambiente         | Quando            | Cuidado                               |
| ---------------- | ----------------- | ------------------------------------- |
| Desenvolvimento  | sua máquina       | sem service worker                    |
| Pré-visualização | todo pull request | **mesmo banco** de produção; sem cron |
| Produção         | merge em `main`   | —                                     |

Cadastre as variáveis nos **três** ambientes da hospedagem. Variável de ambiente
só passa a valer no próximo build — mudou, redeploy.

### 13.2 Domínio e HTTPS

HTTPS é requisito do PWA, não preferência. Aponte o domínio, deixe a plataforma
emitir o certificado e escolha **um** domínio canônico: o app instalado guarda a
origem, e trocar de domínio depois é pedir para todo mundo reinstalar.

### 13.3 Primeiro deploy, na ordem

1. Criar o projeto no backend, rodar `schema.sql` e as migrações.
2. Criar os buckets e as políticas.
3. Cadastrar as variáveis nos três ambientes.
4. Conectar o repositório à hospedagem e deployar.
5. Conferir: entrar, criar, recarregar, sair, entrar de novo.
6. Rodar `testar:rls` contra o projeto novo.
7. Instalar no celular e repetir o teste de modo avião (§5).

---

## 14. Versão e release

- **Versione a cada commit** que vai para `main`, no `package.json`: `+0.0.1`
  para ajuste e correção, `+0.1.0` para um conjunto de mudanças ou recurso
  grande, `+1.0.0` para o que muda o núcleo do app.
- **Mostre a versão no app** (rodapé dos ajustes, ou da barra lateral). É o que
  transforma "está esquisito aqui" em um relato utilizável.
- **Suba a versão do service worker** junto, quando algo pré-cacheado mudar.
- O histórico do git é o changelog. Mensagem de commit no imperativo, dizendo o
  efeito e não o arquivo tocado.
- Decisão de arquitetura que você vai ter que defender de novo em seis meses
  vira um ADR curto em `docs/adr/` — contexto, decisão, consequência.

---

## 15. Observar e diagnosticar

| Peça                       | Para quê                                            |
| -------------------------- | --------------------------------------------------- |
| RUM (LCP/INP/CLS por rota) | saber o que está lento **para quem usa**            |
| Log estruturado (JSON)     | achar o evento sem `grep` em texto livre            |
| Relato de erro do cliente  | ver o que quebra no aparelho dos outros             |
| Ouvintes globais           | `error` e `unhandledrejection`                      |
| Validação de ambiente      | falhar no boot dizendo o nome da variável que falta |

Duas regras não negociáveis:

- **Lista de campos proibidos no log**: conteúdo do usuário, token, e-mail,
  localização. Log que vaza conteúdo é incidente, não observabilidade.
- **Relato de erro com `keepalive: true`** e pilha truncada (~4000 caracteres):
  o envio precisa sobreviver ao fechamento da aba.

Com menos de ~100 visitas, o painel de campo oscila demais para significar
alguma coisa. Até lá, o número que vale é o do orçamento de bundle, que é
determinístico.

---

## 16. Segredos e troca de chaves

Ordem da troca — nesta sequência não há janela de app fora do ar:

1. **Criar a chave nova** sem revogar a antiga (a maioria dos provedores aceita
   duas válidas ao mesmo tempo).
2. **Atualizar a hospedagem** nos três ambientes e **redeployar**.
3. **Conferir que funciona** com uma operação que atravesse tudo — criar um
   registro com upload e processamento cobre banco, arquivos e IA de uma vez.
4. **Só então revogar a antiga**, e atualizar o `.env.local` de cada máquina.

Exceções:

- **Chave que ignora o RLS**: se vazou, o passo 1 não se aplica. Revogue
  primeiro, conserte depois.
- **Par VAPID**: ver §11 — trocar derruba todas as inscrições.
- **Segredo de cron**: pode trocar a qualquer momento; o efeito é a tarefa
  responder 401 até o redeploy.
- **Chave pública do backend**: não é segredo, está no bundle. Trocá-la é
  operação de rotina.

Se um segredo entrar no histórico do git, **rotacione**. Reescrever o histórico
não desfaz o vazamento; só a chave nova desfaz.

---

## 17. Backup, exportação e apagar a conta

Se o app guarda o que a pessoa escreveu, isso é dela — e um dia ela vai querer
levar embora, ou sumir.

- **Exportar tudo** num arquivo só (um `.zip` com JSON + as mídias), gerado no
  cliente a partir dos dados que ele já tem acesso.
- **Importar de volta**, aceitando o próprio formato. Exportação sem importação
  é uma promessa pela metade.
- **Apagar a conta** de verdade: registros, arquivos e inscrições de
  notificação. Com lixeira, defina o prazo (30 dias é o costume) e deixe a
  faxina para a tarefa agendada.
- **Backup do banco**: confira que o plano contratado faz backup automático e,
  mais importante, **teste uma restauração** antes de precisar dela.

### 17.1 Exportar não é o mesmo que fazer backup

Confundir os dois produz um dos erros mais silenciosos que existem, e vale
separar antes de escrever o primeiro deles:

|                    | Exportar                          | Backup                               |
| ------------------ | --------------------------------- | ------------------------------------ |
| Para que serve     | ler, imprimir, guardar noutro app | restaurar a conta                    |
| Quem lê            | uma pessoa                        | o próprio app                        |
| Formato            | Markdown, PDF, HTML               | JSON completo, opcionalmente cifrado |
| Conteúdo protegido | **respeita a proteção**           | vai inteiro                          |
| Perder um campo    | aceitável                         | **quebra a restauração**             |

A linha que importa é a penúltima. Se o app tem qualquer conteúdo que a
interface trata como protegido — trancado por senha, marcado como privado —, a
**exportação legível** tem de respeitar isso, e o **backup** não pode. Um
backup que perde conteúdo não restaura nada; uma exportação que despeja o
conteúdo trancado quebra a única promessa que aquele cadeado fazia.

Escreva os dois motivos em comentário, junto do código, para ninguém
"corrigir" um deles achando que é inconsistência.

### 17.2 Montar HTML na mão? Escape.

A exportação para impressão costuma montar HTML por concatenação, e é o único
lugar do app moderno onde isso ainda acontece — os componentes escapam sozinhos,
esta função não.

```ts
// Texto puro virando conteúdo de HTML. O `&` primeiro, senão as entidades
// recém-criadas viram "&amp;amp;".
const escaparHtml = (t: string) =>
  t.replace(/&/g, "&amp;").replace(/</g, "&lt;").replace(/>/g, "&gt;").replace(/"/g, "&quot;");
```

O que precisa passar por ela: título, rótulo, nome de categoria, nome de
pessoa — tudo que é **texto puro**. O corpo do conteúdo, se já é HTML saneado
na gravação, entra inteiro (é ele que carrega os parágrafos).

E lembre que a janela aberta com `window.open("")` herda a **sua origem**: o que
entra ali executa como se fosse o app. Não é "só um preview".

Duas outras coisas que essa janela costuma esquecer:

- **Pop-up bloqueado** devolve `null`. Sem tratar, o botão simplesmente não faz
  nada e não há como a pessoa descobrir por quê. Diga.
- **A data do conteúdo é a que a pessoa declarou**, não a de criação do
  registro. Quem escreve de madrugada, ou data o item para ontem, recebia o dia
  errado impresso.

---

## 18. Checklist de lançamento

- [ ] `.env.example` completo e comentado; as variáveis cadastradas nos três
      ambientes.
- [ ] Boot valida o ambiente e diz o que falta.
- [ ] RLS ligado em toda tabela; o teste de isolamento passa no CI.
- [ ] Migrações aplicadas e a tabela de estado atualizada; `conferir:schema` passa.
- [ ] Buckets privados, URLs assinadas, arquivos apagados junto do registro.
- [ ] Rotas de API com tempo limite, validação de entrada e limite de taxa.
- [ ] Tarefas agendadas protegidas por segredo e idempotentes.
- [ ] Manifesto válido, ícones 192/512/maskable/apple, `theme_color` por tema.
- [ ] Service worker registrado só em produção, com versão e página offline.
- [ ] Instalado no celular: ícone certo, sem barra de navegador, áreas seguras
      respeitadas, funciona em modo avião.
- [ ] CI verde nos seis passos, orçamento de bundle dentro do teto — por rota.
- [ ] Exportação legível respeita o que a interface diz estar protegido; o
      backup vai inteiro, e os dois motivos estão em comentário (§17.1).
- [ ] HTML montado à mão escapa o texto puro que entra nele (§17.2).
- [ ] HTTPS, domínio canônico definido.
- [ ] Métricas de campo chegando; log sem campo proibido.
- [ ] Exportar, importar e apagar a conta funcionam.
- [ ] Versão visível no app.
