# Texto e acabamento

O quarto documento. Os outros três dizem como o app **parece**, **responde** e
**existe**; este diz o que **não deve estar na tela**. É a tradução, para este
sistema, das regras do [unslop-ui](https://github.com/felipebasurto/unslop-ui)
(Felipe Basurto, licença MIT), que vem inteiro em
[`ferramentas/unslop-ui/`](ferramentas/unslop-ui/) e roda como skill.

A ideia em uma frase: **interface é usada, não lida**. IA decora tudo o que
constrói — rótulo em caixa alta em cima de todo título, `01 //` antes de cada
recurso, subtítulo que repete o título, legenda explicando o botão que a pessoa
está vendo, um ponto verde "AO VIVO" ligado a nada. Cada um desses é um sinal de
que ninguém revisou a tela.

## 1. Quando usar

- **Ao terminar cada tela**, antes do checklist do design system §15.
- **Ao receber uma tela pronta** (sua, de outra IA, de um template).
- **Nunca sozinha.** A skill tem `disable-model-invocation: true`: cortar texto é
  decisão de quem é dono do produto, então ela só roda chamada.

```
/unslop-ui app/(telas)/inicio/page.tsx
/unslop-ui components/ --surface
```

Neste repositório a skill já está registrada em `.claude/skills/unslop-ui`
(link para `ferramentas/unslop-ui/skills/unslop-ui`). Em outro projeto:
`npx skills add felipebasurto/unslop-ui -g`. Agente sem skills: abra
[`SKILL.md`](ferramentas/unslop-ui/skills/unslop-ui/SKILL.md) e siga à mão.

## 2. Os vinte sinais

Os ids são estáveis; cite-os como U1–U20 no relatório.

### Texto de interface

| Id  | Sinal                    | Exemplo                                                       | O que fazer                                                        |
| --- | ------------------------ | ------------------------------------------------------------- | ------------------------------------------------------------------ |
| U1  | Sobrancelha decorativa   | `RECURSOS` em cima de "Recursos"; pílula `● NOVIDADES`        | Apagar. Ver §3 — a nossa `.rotulo` só fica se disser outra coisa.  |
| U2  | Numeração decorativa     | `01 //`, `02 //` em recursos sem ordem                        | Apagar, salvo ordem real (passos de cadastro).                     |
| U3  | Subtítulo por reflexo    | "Análises" / "Entenda melhor seus dados"                      | Fica só se disser um fato que o título não diz.                    |
| U4  | Explicar o que se vê     | "Toque no botão abaixo para começar"                          | Apagar. Vazio, erro e confirmação ficam, reescritos (§13 do DS).   |
| U5  | Nota de projeto vazada   | "(atualiza em tempo real)", "(com IA)"                        | Apagar, ou levar para dica se alguém procuraria.                   |
| U6  | Fileira de estados       | `12 itens · Atualizado há 2 min · Sincronizado · v2.1`        | Fica o fato em que a pessoa age.                                   |
| U7  | Rotular tudo             | Rótulo, placeholder e ajuda dizendo "e-mail"                  | Um sinal só. No formulário, fica o rótulo visível.                 |
| U8  | Slogan picotado          | "Finanças. Simples. Enfim."                                   | Uma frase que nomeia a coisa, com palavras que a tela já usa.      |

Título de seção, de cartão ou legenda dentro do app: **até 5 palavras**.

### Decoração

O teste de estado real (U6, U9, U10): o elemento nasce de uma condição ou de um
valor (`{aoVivo && …}`, uma prop, o resultado de uma consulta). Marcação que
sempre aparece com texto fixo é enfeite.

| Id  | Sinal                    | O que fazer                                                                                     |
| --- | ------------------------ | ----------------------------------------------------------------------------------------------- |
| U9  | Badge sem estado         | `NOVO`, `IA`, `Beta`, `🔥 Popular` de enfeite: apagar.                                           |
| U10 | Ponto de status          | `animate-pulse` ao lado de dado parado: apagar. O esqueleto de carregamento não é isto.          |
| U11 | Emoji e ícone decorativo | Em título, botão, rótulo de KPI, ícone em quadrado tingido sobre cada cartão: apagar com a caixa. |

O ícone **fica** quando é o próprio controle (botão de ícone com `aria-label`),
quando carrega sentido que o texto não tem, ou numa barra/menu de ações em que
todo item tem um. Os ícones da nossa navegação (§9 do DS) passam por isso: são o
alvo de toque da pílula no celular.

### Gráficos e painéis

| Id  | Sinal                          | O que fazer                                                                |
| --- | ------------------------------ | -------------------------------------------------------------------------- |
| U12 | Título que nomeia o gráfico    | "Gráfico de barras da receita" → "Receita mensal".                         |
| U13 | Instrução de leitura           | "Cada barra é um mês", "Passe o mouse": apagar.                            |
| U14 | Dizer duas vezes               | Uma codificação por quantidade. KPI: número, uma seta, o período.          |
| U15 | Andaime competindo com o dado  | Moldura, grade escura, sombra em barra: apagar ou virar linha fina.        |
| U16 | Legenda onde cabe rótulo       | Nome no fim da linha, com o último valor. Legenda só com mais de 4 séries. |

As peças de gráfico do design system (§8.23) e a distribuição (§8.26) já nascem
assim; U12–U16 é o que conferir quando alguém as desmontou.

### Superfície (só com `--surface`)

| Id  | Sinal               | Neste sistema, o alvo é                                                                       |
| --- | ------------------- | --------------------------------------------------------------------------------------------- |
| U17 | Degradê e brilho    | A superfície e a sombra dos tokens (§3, §5 do DS). O véu de hover de cor única não é degradê. |
| U18 | Espaço fora da grade | Os degraus de §5 do DS.                                                                      |
| U19 | Peso 800/900        | 500–700 conforme §4. A `.display` e o `.numeral` em 700 estão certos.                          |
| U20 | Movimento enfeite   | `--tempo-rapido` / `--tempo-normal` e `--curva` (§6). Sem `transition-all`, sem escala no hover de cartão. |

**Os tokens deste repositório ganham dos valores de reserva** do
[`tokens.md`](ferramentas/unslop-ui/skills/unslop-ui/references/tokens.md) da
skill — a própria skill manda assim. Onde ela diz "150ms ease-out", aqui vale o
token; onde ela diz "press 0.97–0.98", aqui vale a tabela de §6 do DS (que já
inclui 0.995 para cartão e 0.94 para controle pequeno). As animações de
celebração (`.animar-salvo`, `.animar-tique`) explicam que algo aconteceu: são o
"movimento que explica" que a regra U20 preserva.

## 3. Onde as regras se encontram com as nossas

Três pontos em que foi preciso decidir — e a decisão vale para os dois lados:

1. **Sobrancelha (`.rotulo`, DS §4.2) × U1.** A nossa sobrancelha existe e
   continua. Ela passa no U1 quando diz o que o título não diz: "47 entradas até
   aqui", "março de 2026", o nome da conta. Repetir o nome da seção ("VISÃO
   GERAL" acima de "Visão geral") é U1 e sai. É o mesmo que o DS §13 já dizia —
   "rótulo é contexto, não decoração" —, agora com nome e número.
2. **Texto vem do dicionário (DS §15) × guarda-corpo de i18n.** A skill não
   escreve texto no componente: edita na origem ou lista como tarefa. Aqui a
   origem é sempre o dicionário — corrija lá.
3. **Vazio convida (DS §13) × U4.** O estado vazio não é apagado: é reescrito em
   uma ou duas frases que dizem o que aparece ali e qual o primeiro passo, sem
   "Ops!" nem exclamação. As duas regras dizem o mesmo.

Regras de escrita que a skill traz (as do *unslop*, citadas como "unslop 13")
foram escritas para inglês. Em português, valem as do DS §13; onde elas não
dizem nada, siga o espírito: frase curta, sem travessão de efeito, sem
superlativo que o código não prova.

## 4. Guarda-corpos

- **Cortar primeiro, escrever pouco.** Texto novo só pode afirmar o que a tela
  já mostra. Nenhum número, velocidade ou benefício inventado.
- Nunca apagar `aria-label`, `alt`, texto só para leitor de tela. Se o texto
  cortado era o único nome acessível, ele vai para o `aria-label`.
- Campo de formulário mantém rótulo visível ligado ao `input`.
- Afirmação que o código não prova (número de clientes, depoimento, logo de
  cliente): não mexer — listar como tarefa.
- Tela limpa volta intocada. O veredito "nenhum" é um resultado.
- Ao apagar um elemento, apague o que só existia por ele: invólucro vazio,
  margem, import.

## 5. O relatório

Toda passada termina com a tabela de contagem (antes, depois, mantidos) e até
três listas: **Reescrito** (todo texto que a IA escreveu, antigo → novo — é o
que o dono precisa conferir), **Mantido de propósito** e **Tarefas**. Formato
completo em [`SKILL.md`](ferramentas/unslop-ui/skills/unslop-ui/SKILL.md).

O contador (`scripts/count_tells.py`) é uma heurística de regex: acha
candidatos, não decide. U3, U4, U5, U7, U8 e U14 só aparecem lendo.

```bash
python3 ferramentas/unslop-ui/skills/unslop-ui/scripts/count_tells.py --surface app/
```

## 6. Checklist

Antes de dar uma tela por pronta, junto com o DS §15:

- [ ] Rodou `/unslop-ui` (com `--surface` se a tela é nova) e o relatório foi lido.
- [ ] Nenhuma sobrancelha repete o título; toda `.rotulo` diz um fato.
- [ ] Nenhum subtítulo, ajuda ou legenda narrando o que a tela mostra.
- [ ] Nenhum badge, ponto pulsante ou "ao vivo" sem condição no código.
- [ ] Nenhum emoji em título, botão ou rótulo; nenhum ícone em quadrado tingido.
- [ ] Gráfico com título do dado, sem instrução de leitura, rótulo direto.
- [ ] A lista **Reescrito** foi conferida por quem é dono do produto.
- [ ] As **Tarefas** foram resolvidas ou registradas.

## 7. Manter em dia

`ferramentas/unslop-ui/` é cópia do repositório original (commit `cd89116`,
2026-11-18), sem o vídeo e o GIF de demonstração. Para atualizar, substitua a
pasta e rode os testes:

```bash
cd ferramentas/unslop-ui && python3 -m unittest discover -s tests
```

`evals/run.py` é outra coisa: chama um agente de verdade (`claude -p`) sobre
os casos de `evals/fixtures/` e confere o que ele preservou. Gasta tokens e o
resultado depende do agente; rode à mão quando mudar a skill, com ela instalada.
