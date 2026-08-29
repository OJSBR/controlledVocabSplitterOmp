# Controlled Vocabulary Splitter — OMP plugin

[![OMP](https://img.shields.io/badge/OMP-3.5-brightgreen)](https://pkp.sfu.ca/omp/)
[![Version](https://img.shields.io/badge/version-1.0.0.1-blue)](version.xml)
[![License](https://img.shields.io/badge/license-GPL--3.0-lightgrey)](LICENSE)

**⬇️ Install package:** [OMP 3.5](https://github.com/OJSBR/controlledVocabSplitterOmp/releases/download/1.0.0.1/controlledVocabSplitter-1.0.0.1.tar.gz) — or browse all [Releases](../../releases).

A generic plugin for **Open Monograph Press (OMP)** that splits **keywords, subjects,
disciplines and supporting agencies pasted as a single line** into the separate terms the
author meant — in the field itself and on every save — **without patching OMP core**.

> **Developed and maintained by [OJSBR](https://ojsbr.com).** See the
> [Credits & authorship](#credits--authorship) section below.

## Compatibility & branches

| OMP version | Branch | Plugin release |
|-------------|--------|----------------|
| OMP 3.5.x   | [`stable-3_5_0`](../../tree/stable-3_5_0) *(default)* | 1.0.0.1 |

**Looking for the OJS edition?** It lives in its own repository,
[OJSBR/controlledVocabSplitter](https://github.com/OJSBR/controlledVocabSplitter). This repository is the same plugin with the
terminology and the data model of a book publisher.

## The problem

Authors select the keyword line in their manuscript, copy it, and paste the whole thing into
the keyword field:

```
Palatal Expansion Technique. Clinical Protocol. Orthopedic appliance.
```

One keystroke later that is **one** term in `controlled_vocab_entries`. The reader sees a
sentence where a tag should be, the keyword cloud shows the whole phrase, and
`citation_keywords` goes out as a single meta tag — which is exactly what Google Scholar and
the indexers read.

It is not rare. In one press we found it in **every published book, in all three
languages**: 24 records that should have been 107 terms.

## What it does

Two levels, one rule set:

- **In the browser.** The vocabulary field splits what is pasted into it, on the spot, so the
  author sees separate tags and can fix any term that was cut in the wrong place. Pressing
  Enter on a typed list does the same. This is a courtesy, not the guarantee.
- **On the server.** The controlled-vocabulary repository is replaced by one that applies the
  same rules on **every** write: metadata form, submission wizard, REST API and native XML
  import — including the browsers where the script never ran.

### The splitting rules

Only **one** separator is used per entry: the first of the ones below that occurs in it.
A list written with semicolons and commas is therefore cut on the semicolons alone, which
keeps an inverted subject heading such as `Hypertension, Pregnancy-Induced` in one piece.

| Order | Separator | Example |
|-------|-----------|---------|
| 1 | Semicolon | `Ozone therapy; Oxidative stress; Regulation` |
| 2 | Comma followed by a space | `Ozone therapy, Oxidative stress, Regulation` |
| 3 | Period followed by a space | `Ozone therapy. Oxidative stress. Regulation` |

What is **never** cut:

- A period with no space after it, so legal references such as `Lei 13.964/2019` and
  `Decreto 12.456/2025` stay whole — cutting one destroys the citation.
- A period that closes a single letter, because that is an initial: `S. aureus`, `E. coli`
  and `C. albicans` stay whole.
- A comma with no space after it, so `1,5 mm` stays whole.

Terms are also cleaned: exotic spaces (no-break, en, thin, ideographic) and zero-width
characters become plain spaces, Unicode is normalised to NFC, and punctuation left dangling
at the edges is dropped. Duplicates within the same language are removed, case-insensitively.

**The comma is the risky one.** An inverted subject heading is a single term and would be cut
in two. Presses that index with MeSH or DeCS should turn the comma off.

## Installation

1. Download the package from the link at the top of this README.
2. In OMP, go to **Settings → Website → Plugins → Upload a new plugin** and upload the
   `.tar.gz`, or unpack it into `plugins/generic/` and clear `cache/t_cache` and
   `cache/t_compile`.
3. Enable **Controlled Vocabulary Splitter** in the plugin list.

> **Keep the folder name.** PKP 3.5 derives the plugin's class namespace from its installation
> directory, so the folder must be `controlledVocabSplitter` — the `Omp` suffix belongs to the repository
> name only. The release tarball already unpacks with the right name.

## Configuration

**Settings → Website → Plugins → Controlled Vocabulary Splitter → Settings.**

- **Vocabularies** — which of the four are split. Everything is split by default; anything
  unticked is stored exactly as it was typed.
- **Separators** — which of the three are honoured. All are on by default.

Settings are per press.

### Repairing an existing archive

The plugin only acts when something is saved, so an archive built before it was installed
keeps its concatenated terms. The bundled script repairs it in one pass, and prints what it
would do without changing anything unless you ask:

```bash
php plugins/generic/controlledVocabSplitter/tools/fixExistingVocabs.php
php plugins/generic/controlledVocabSplitter/tools/fixExistingVocabs.php --write
```

`--context=1`, `--publication=13`, `--field=keywords` and `--separators=semicolon,period`
narrow it down. Run it as the account that owns the files, never as root, and clear
`cache/t_cache`, `cache/t_compile` and any keyword-cloud cache afterwards.

## How it works (technical)

**Server side.** Every write of these vocabularies goes through
`Repo::controlledVocab()->insertBySymbolic()` — the publication DAO (metadata form, wizard and
REST API), the native XML import filter, and any plugin that stores vocabulary. The facade
resolves that repository from the container on every call, so the plugin binds a subclass
instead of chasing each entry point with its own hook. Nothing else is overridden: reads,
sequencing and deletion stay as core wrote them, and vocabularies that are not a publication's
own (user interests, for one) are passed through untouched.

Overriding a method whose signature the parent no longer has is a fatal error PHP raises while
compiling the class, which no `try`/`catch` can recover from. So `isCoreSignatureKnown()`
inspects the parent by reflection **before** the subclass is ever loaded; on a core release
that changed it, the plugin falls back to the browser-side split instead of taking the site
down.

**Browser side.** The plugin wraps the `FieldControlledVocab` component, adding a `paste`
handler and an override of `selectSuggestion` — a suggestion picked from the list is never
touched, only free text. Form fields are **not** resolved from the global registry: FormGroup
keeps its own `components` map inside the compiled `js/build.js`, and that is the copy Vue
renders, so the plugin walks the component tree and replaces every occurrence. The script is
published with `STYLE_SEQUENCE_LAST`, which lands it after `js/build.js` and before the inline
`pkp.registry.init()` call.

## Tests

Verified on **OMP 3.5.0.4** against a live press:

- the plugin's own repository really replaces the core one at runtime
  (`SplittingControlledVocabRepository`), and the `insertBySymbolic()` signature guard
  recognises the OMP core;
- a real write through `Repo::controlledVocab()->insertBySymbolic()` turned the pasted line
  `"Palatal Expansion. Clinical Protocol. Orthopedic appliance."` into **three** rows in
  `controlled_vocab_entry_settings`;
- `"Lei 13.964/2019"` survived intact next to a second term — the period only splits when a
  space follows it.

The Cypress test and the two PHP regression suites of the OJS edition are **not** shipped
here yet: they target OJS fixtures (sections, `abstractsNotRequired`) that do not exist in
OMP. Porting them is tracked for a later release.

## Languages

Ships in **7 languages**: English, Portuguese (Brazil), Portuguese (Portugal), Spanish, French,
Italian and German. Note the French folder is `locale/fr` — PKP 3.5 has no `fr_FR` locale, so a
`fr_FR` folder would never be loaded.

## Credits & authorship

- **Developed and maintained by** [OJSBR](https://ojsbr.com) — original plugin.
- Distributed under the **GNU GPL v3**.

## Contributing

Issues and pull requests are welcome. Please target the branch matching the OMP version you
are working against. See [`CONTRIBUTING.md`](CONTRIBUTING.md).

## License

Distributed under the **GNU GPL v3**. See [`LICENSE`](LICENSE) and `docs/COPYING`.

---

## 🇧🇷 Português

Plugin genérico para o **Open Monograph Press (OMP)** que separa **palavras-chave, assuntos,
áreas do conhecimento e agências de fomento coladas em uma linha só** nos termos que o autor
quis dizer — no próprio campo e em toda gravação — **sem alterar o núcleo do OMP**.

> **Desenvolvido e mantido pela [OJSBR](https://ojsbr.com).** Veja a seção
> [Créditos e autoria](#créditos-e-autoria) abaixo.

### Compatibilidade e branches

| Versão do OMP | Branch | Release do plugin |
|---------------|--------|-------------------|
| OMP 3.5.x     | [`stable-3_5_0`](../../tree/stable-3_5_0) *(padrão)* | 1.0.0.1 |

**Procurando a versão de OJS?** Ela tem repositório próprio,
[OJSBR/controlledVocabSplitter](https://github.com/OJSBR/controlledVocabSplitter). Este repositório é o mesmo plugin com a
terminologia e o modelo de dados de uma editora de livros.

### O problema

O autor seleciona a linha de palavras-chave do manuscrito, copia e cola tudo de uma vez no
campo:

```
Técnica de Expansão Palatina. Protocolo Clínico. Aparelho Ortopédico.
```

Um Enter depois, isso é **um** registro em `controlled_vocab_entries`. O leitor vê uma frase
onde deveria haver etiquetas, a nuvem de palavras-chave mostra a frase inteira e o
`citation_keywords` sai como uma meta tag só — justamente o que o Google Scholar e os
indexadores leem.

Não é raro. Em uma editora o problema estava em **todos os livros publicados, nos três
idiomas**: 24 registros que deveriam ser 107 termos.

### O que faz

Duas camadas, uma regra só:

- **No navegador.** O campo separa o que é colado, na hora, para o autor ver as etiquetas e
  corrigir qualquer termo cortado no lugar errado. Digitar a lista e apertar Enter faz o
  mesmo. Isso é cortesia, não é a garantia.
- **No servidor.** O repositório de vocabulário controlado é trocado por um que aplica as
  mesmas regras em **toda** gravação: formulário de metadados, assistente de submissão, API
  REST e importação XML nativa — inclusive nos navegadores em que o script não rodou.

#### As regras de separação

Só **um** separador é usado por registro: o primeiro da lista abaixo que aparecer nele. Uma
lista escrita com ponto e vírgula e vírgulas é cortada só nos pontos e vírgulas, o que mantém
inteiro um descritor invertido como `Hipertensão Induzida pela Gravidez, Síndrome`.

| Ordem | Separador | Exemplo |
|-------|-----------|---------|
| 1 | Ponto e vírgula | `Ozonioterapia; Estresse oxidativo; Regulamentação` |
| 2 | Vírgula seguida de espaço | `Ozonioterapia, Estresse oxidativo, Regulamentação` |
| 3 | Ponto seguido de espaço | `Ozonioterapia. Estresse oxidativo. Regulamentação` |

O que **nunca** é cortado:

- Ponto sem espaço depois, o que mantém inteiras referências legais como `Lei 13.964/2019` e
  `Decreto 12.456/2025` — quebrar uma delas destrói a citação.
- Ponto que fecha uma letra sozinha, porque ali é inicial: `S. aureus`, `E. coli` e
  `C. albicans` ficam inteiros.
- Vírgula sem espaço depois, o que mantém inteiro `1,5 mm`.

Os termos também são limpos: espaços exóticos (NBSP, EN SPACE, THIN SPACE, ideográfico) e
caracteres de largura zero viram espaço comum, o Unicode é normalizado para NFC e a pontuação
solta nas bordas cai. Duplicatas no mesmo idioma são removidas, sem diferenciar maiúsculas.

**A vírgula é a arriscada.** Um descritor invertido é um termo só e seria cortado em dois.
Editora que indexa com MeSH ou DeCS deve desligar a vírgula.

### Instalação

1. Baixe o pacote pelo link no topo deste README.
2. No OMP, vá em **Configurações → Website → Plugins → Enviar um novo plugin** e envie o
   `.tar.gz`, ou descompacte em `plugins/generic/` e limpe `cache/t_cache` e
   `cache/t_compile`.
3. Habilite o **Separador de Vocabulário Controlado** na lista de plugins.

> **Não renomeie a pasta.** O PKP 3.5 deriva o namespace da classe do diretório de instalação,
> então a pasta precisa se chamar `controlledVocabSplitter` — o sufixo `Omp` é só do repositório. O pacote
> de release já descompacta com o nome certo.

### Configuração

**Configurações → Website → Plugins → Separador de Vocabulário Controlado → Configurações.**

- **Vocabulários** — quais dos quatro são separados. Por padrão, todos; o que for desmarcado
  é gravado exatamente como foi digitado.
- **Separadores** — quais dos três valem. Por padrão, todos.

A configuração é por editora.

#### Corrigir um acervo já gravado

O plugin só age na gravação, então um acervo montado antes da instalação continua com os
termos colados. O script que acompanha o plugin corrige tudo de uma vez, e só simula até você
mandar gravar:

```bash
php plugins/generic/controlledVocabSplitter/tools/fixExistingVocabs.php
php plugins/generic/controlledVocabSplitter/tools/fixExistingVocabs.php --write
```

`--context=1`, `--publication=13`, `--field=keywords` e `--separators=semicolon,period`
restringem o alcance. Rode como o dono dos arquivos, nunca como root, e depois limpe
`cache/t_cache`, `cache/t_compile` e o cache de qualquer bloco de nuvem de palavras-chave.

### Testes

Verificado no **OMP 3.5.0.4** em uma editora real:

- o repositório do plugin de fato substitui o do núcleo em tempo de execução
  (`SplittingControlledVocabRepository`), e a trava que confere a assinatura de
  `insertBySymbolic()` reconhece o núcleo do OMP;
- uma gravação real por `Repo::controlledVocab()->insertBySymbolic()` transformou a linha
  colada `"Palatal Expansion. Clinical Protocol. Orthopedic appliance."` em **três** registros
  em `controlled_vocab_entry_settings`;
- `"Lei 13.964/2019"` ficou inteira ao lado de um segundo termo — o ponto só separa quando vem
  espaço depois.

O teste Cypress e as duas suítes PHP de regressão da versão de OJS **ainda não** vêm neste
repositório: elas dependem de estruturas que só existem no OJS (seções,
`abstractsNotRequired`). O porte está previsto para uma versão futura.

### Idiomas

Em **7 idiomas**: inglês, português (Brasil), português (Portugal), espanhol, francês, italiano
e alemão. A pasta do francês é `locale/fr` — o PKP 3.5 não tem o locale `fr_FR`, então uma pasta
`fr_FR` nunca seria carregada.

### Créditos e autoria

- **Desenvolvido e mantido pela** [OJSBR](https://ojsbr.com) — plugin autoral.
- Distribuído sob a **GNU GPL v3**.

### Licença

Distribuído sob a **GNU GPL v3**. Veja [`LICENSE`](LICENSE) e `docs/COPYING`.
