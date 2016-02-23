# Babel Plugin Handbook

Questo documento include le linee guida per la creazione di [plugins](https://babeljs.io/docs/advanced/plugins/) per [Babel](https://babeljs.io).

[![cc-by-4.0](https://licensebuttons.net/l/by/4.0/80x15.png)](http://creativecommons.org/licenses/by/4.0/)

This handbook is available in other languages, see the [README](/README.md) for a complete list.

# Sommario

  * [Introduzione](#toc-introduction)
  * [Nozioni di base](#toc-basics) 
      * [AST](#toc-asts)
      * [Stadi primari di Babel](#toc-stages-of-babel)
      * [Parse](#toc-parse) 
          * [Analisi Lessicale](#toc-lexical-analysis)
          * [Analisi Sintattica](#toc-syntactic-analysis)
      * [Transform](#toc-transform)
      * [Generate](#toc-generate)
      * [Navigazione (traverse)](#toc-traversal)
      * [Visitatori (visitor)](#toc-visitors)
      * [Percorsi (path)](#toc-paths) 
          * [Oggetto "path" disponibile nei visitatori](#toc-paths-in-visitors)
      * [Stato (state)](#toc-state)
      * [Scope](#toc-scopes) 
          * [Binding](#toc-bindings)
  * [API](#toc-api) 
      * [babylon](#toc-babylon)
      * [babel-traverse](#toc-babel-traverse)
      * [babel-types](#toc-babel-types)
      * [Definizioni](#toc-definitions)
      * [Costruttori](#toc-builders)
      * [Validatori](#toc-validators)
      * [Convertitori](#toc-converters)
      * [babel-generator](#toc-babel-generator)
      * [babel-template](#toc-babel-template)
  * [Scrivere il primo plugin per Babel](#toc-writing-your-first-babel-plugin)
  * [Operazioni di trasformazione](#toc-transformation-operations) 
      * [Navigazione AST](#toc-visiting)
      * [Verificare se un nodo è di uno specifico tipo](#toc-check-if-a-node-is-a-certain-type)
      * [Verificare se esistono referenze ad un identifier](#toc-check-if-an-identifier-is-referenced)
      * [Manipolazione AST](#toc-manipulation)
      * [Sostituire un nodo](#toc-replacing-a-node)
      * [Sostituire un nodo con più nodi](#toc-replacing-a-node-with-multiple-nodes)
      * [Sostituire un nodo con una stringa di codice](#toc-replacing-a-node-with-a-source-string)
      * [Inserire un nodo adiacente](#toc-inserting-a-sibling-node)
      * [Rimuovere un nodo](#toc-removing-a-node)
      * [Sostituire un nodo padre](#toc-replacing-a-parent)
      * [Rimuovere un nodo padre](#toc-removing-a-parent)
      * [Scope (e variabili)](#toc-scope)
      * [Verificare se una variabile locale è associata ad uno scope](#toc-checking-if-a-local-variable-is-bound)
      * [Generare un nuovo nome di variabile (UID)](#toc-generating-a-uid)
      * [Promuovere la dichiarazione di una variabile ad uno scope padre](#toc-pushing-a-variable-declaration-to-a-parent-scope)
      * [Rinominare una variabile (e tutte le sue referenze)](#toc-rename-a-binding-and-its-references)
  * [Opzioni per i propri plugin](#toc-plugin-options)
  * [Generare nodi (per inserimenti e trasformazioni)](#toc-building-nodes)
  * [Best practice](#toc-best-practices) 
      * [Evitare navigazioni non necessarie dell'AST](#toc-avoid-traversing-the-ast-as-much-as-possible)
      * [Aggregare i visitatori quando possibile](#toc-merge-visitors-whenever-possible)
      * [Evitare navigazioni programmatiche nidificate](#toc-do-not-traverse-when-manual-lookup-will-do)
      * [Ottimizzare i visitatori nidificati](#toc-optimizing-nested-visitors)
      * [Essere consapevoli delle strutture nidificate](#toc-being-aware-of-nested-structures)

# <a id="toc-introduction"></a>Introduzione

Babel è un compilatore multiuso e generalista per JavaScript. Nello specifico è un insieme di moduli che possono essere usati per differenti operazioni di analisi statica del codice.

> Per analisi statica si intende il processo di analisi di codice senza che questo venga eseguito. (L'analisi del codice effettuata durante la sua esecuzioni è definita analisi dinamica). Le finalità dell'analisi statica possono essere moteplici. It can be used for linting, compiling, code highlighting, code transformation, optimization, minification, and much more.

È possibile utilizzare Babel per costruire diversi tipi di strumenti che consentono di essere più produttivi e scrivere programmi migliori.

> ***Per aggiornamenti futuri, segui [@thejameskyle](https://twitter.com/thejameskyle) su Twitter.***

* * *

# <a id="toc-basics"></a>Nozioni di base

Babel è un compilatore JavaScript, specificamente un compilatore di fonte a fonte, spesso chiamato un "transpiler". Ciò significa che puoi dare Babel del codice JavaScript che viene modificato da Babel, il quale genera del nuovo codice.

## <a id="toc-asts"></a>AST

Ognuno di questi passaggi comportano la creazione o l'utilizzo con un [Albero Sintattico Astratto](https://en.wikipedia.org/wiki/Abstract_syntax_tree) o AST.

> Babel uses an AST modified from [ESTree](https://github.com/estree/estree), with the core spec located [here](https://github.com/babel/babel/blob/master/doc/ast/spec.md).

```js
function square(n) {
  return n * n;
}
```

> Check out [AST Explorer](http://astexplorer.net/) to get a better sense of the AST nodes. [Here](http://astexplorer.net/#/Z1exs6BWMq) is a link to it with the example code above pasted in.

Questo stesso programma può essere rappresentato come un elenco come questo:

```md
- FunctionDeclaration:
  - id:
    - Identifier:
      - name: square
  - params [1]
    - Identifier
      - name: n
  - body:
    - BlockStatement
      - body [1]
        - ReturnStatement
          - argument
            - BinaryExpression
              - operator: *
              - left
                - Identifier
                  - name: n
              - right
                - Identifier
                  - name: n
```

Oppure come un oggetto JavaScript come questo:

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

Noterete che ogni livello di un AST ha una struttura simile:

```js
{
  type: "FunctionDeclaration",
  id: {...},
  params: [...],
  body: {...}
}
```

```js
{
  type: "Identifier",
  name: ...
}
```

```js
{
  type: "BinaryExpression",
  operator: ...,
  left: {...},
  right: {...}
}
```

> Nota: Alcune proprietà sono state rimosse per semplicità.

Ognuno di questi è conosciuto come un **Nodo**. Un AST può essere costituito da un singolo nodo, o centinaia se non migliaia di nodi. Insieme, essi sono in grado di descrivere la sintassi di un programma che può essere utilizzato per l'analisi statica.

Ogni Nodo ha questa interfaccia:

```typescript
interface Node {
  type: string;
}
```

La proprietà `type` è una stringa che rappresenta il tipo di oggetto del Nodo (ie. `"FunctionDeclaration"`, `"Identificatore"` o `"BinaryExpression"`). Ogni tipo di Nodo definisce un ulteriore insieme di proprietà che descrivono il tipo di Nodo in particolare.

Ci sono altre proprietà su ogni nodo che Babel genera che descrivono la posizione del nodo nel codice sorgente originale.

```js
{
  type: ...,
  start: 0,
  end: 38,
  loc: {
    start: {
      line: 1,
      column: 0
    },
    end: {
      line: 3,
      column: 1
    }
  },
  ...
}
```

Queste proprietà `start`, `end`, `loc`, appaiono in ogni singolo Nodo.

## <a id="toc-stages-of-babel"></a>Stadi primari di Babel

Le tre fasi primarie di Babel sono **parse**, **trasformare**, **generare**.

### <a id="toc-parse"></a>Parse

Lo stage di **parse** ha come input del codice e come output un AST. Ci sono due fasi di analisi in Babel: [**Analisi Lessicale**](https://en.wikipedia.org/wiki/Lexical_analysis) e [**Analisi Sintattica**](https://en.wikipedia.org/wiki/Parsing).

#### <a id="toc-lexical-analysis"></a>Analisi Lessicale

Durante l'analisi lessicale, Babel analizza una stringa di codice e la trasforma in una serie di **tokens**.

Si può pensare ai tokens come un vettore che contiene pezzi di sintassi del linguaggio.

```js
n * n;
```

```js
[
  { type: { ... }, value: "n", start: 0, end: 1, loc: { ... } },
  { type: { ... }, value: "*", start: 2, end: 3, loc: { ... } },
  { type: { ... }, value: "n", start: 4, end: 5, loc: { ... } },
  ...
]
```

Ciascun `type` ha un insieme di proprietà che descrivono il token:

```js
{
  type: {
    label: 'name',
    keyword: undefined,
    beforeExpr: false,
    startsExpr: true,
    rightAssociative: false,
    isLoop: false,
    isAssign: false,
    prefix: false,
    postfix: false,
    binop: null,
    updateContext: null
  },
  ...
}
```

In modo simile ai Nodi AST, anche i tokens hanno le proprietà `start`, `end` e `loc`.

#### <a id="toc-syntactic-analysis"></a>Analisi Sintattica

L'analisi sintattica prende una serie di tokens e costruisce una rappresentazione AST. Utilizzando le informazioni nei token, questa fase li riformatterà come un AST che rappresenta la struttura del codice.

### <a id="toc-transform"></a>Transform

La fase di [trasformazione](https://en.wikipedia.org/wiki/Program_transformation) prende un AST e lo attraversa aggiungendo, modificando e rimuovendo Nodi durante l'attraversamento. Questa è di gran lunga la parte più complessa di Babel o qualsiasi altro compilatore. Questa fase è dove operano i plugin ed è quindi l'argomento principale trattato in questo manuale. Quindi non andremo in profondo in questa fase in questa sezione.

### <a id="toc-generate"></a>Generate

La fase di [generazione del codice](https://en.wikipedia.org/wiki/Code_generation_(compiler)) prende l'AST finale e lo trasforma in una stringa di codice, ed allo stesso tempo crea una [mappa di origine](http://www.html5rocks.com/en/tutorials/developertools/sourcemaps/).

Code generation is pretty simple: you traverse through the AST depth-first, building a string that represents the transformed code.

## <a id="toc-traversal"></a>Navigazione (traverse)

When you want to transform an AST you have to [traverse the tree](https://en.wikipedia.org/wiki/Tree_traversal) recursively.

Per esempio prendiamo un nodo di tipo `FunctionDeclaration`. Ha diverse proprietà: `id`, `params` e `body`. Ognuno di essi ha uno o più nodi nidificati.

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  params: [{
    type: "Identifier",
    name: "n"
  }],
  body: {
    type: "BlockStatement",
    body: [{
      type: "ReturnStatement",
      argument: {
        type: "BinaryExpression",
        operator: "*",
        left: {
          type: "Identifier",
          name: "n"
        },
        right: {
          type: "Identifier",
          name: "n"
        }
      }
    }]
  }
}
```

Così iniziamo dalla `FunctionDeclaration` e conosciamo le proprietà interne quindi visitiamo ciascuno di essi e i loro figli in ordine.

La proprietà successiva è `id` è un `identificatore`. L'`identificatore ` non ha altri nodi nidificati quindi andiamo avanti.

Successivamente troviamo la proprietà `params`, ovvero un vettore di nodi. Visitiamo ciascuno di essi. In questo caso è un singolo nodo che è anche un `identificatore`, quindi andiamo avanti.

Adesso arriviamo alla proprietà `body` che è un `BlockStatement` con una proprietà `body` che è un vettore di nodi, quindi andiamo a ciascuno di essi.

Qui l'unico elemento è un nodo di `ReturnStatement` che ha un `argument`, andiamo all' `argument` e troviamo una `BinaryExpression`.

L'oggetto `BinaryExpression` ha le proprietà `operator`, `left` e `right`. L'operator non è un nodo ma solo un valore - "*", quindi visitiamo le proprietà `left` e `right`.

Questo processo di attraversamento dell'AST avviene durante lo stage di trasformazione di Babel.

### <a id="toc-visitors"></a>Visitatori (visitor)

Quando si parla di "andare" a un nodo, intendiamo in realtà la **visita** di un nodo. La ragione per che cui usiamo quel termine è perché è sfruttato il concetto di [**visitatore**](https://en.wikipedia.org/wiki/Visitor_pattern).

I visitatori sono un modello utilizzato in attraversamento di AST tra i diversi linguaggi. In poche parole sono un oggetto con metodi definiti per accettare tipi di nodi particolari in un albero. È un po' astratto, quindi diamo un'occhiata a un esempio.

```js
const MyVisitor = {
  Identifier() {
    console.log("Called!");
  }
};
```

> **Nota:** `Identifier() { ... }` è l'abbreviazione di `Identifier: {enter () { ... }}`.

Questo è un visitatore base che quando utilizzato durante un attraversamento chiamerà il metodo `Identifier()` per ogni `identifier` nell'albero.

Per esempio con questo codice il metodo `Identifier()` verrà chiamato quattro volte con ogni `identificatore` (tra cui `square`).

```js
function square(n) {
  return n * n;
}
```

```js
Called!
Called!
Called!
Called!
```

Queste chiamate sono tutte eseguite mentre si sta per**entrare** nel nodo. Però c'è anche la possibilità di eseguire un metodo di un visitatore durante **l'uscita** dal nodo.

Immaginate di avere questa struttura ad albero:

```js
- FunctionDeclaration
  - Identifier (id)
  - Identifier (params[0])
  - BlockStatement (body)
    - ReturnStatement (body)
      - BinaryExpression (argument)
        - Identifier (left)
        - Identifier (right)
```

Durante l'attraversamento di ogni ramo dell'albero, eventualmente raggiungiamo dei nodi senza altri rami, quindi per arrivare al nodo successivo c'è bisogno di ritornare in su per arrivare al nodo successivo. Andando verso il basso l'albero **entriamo** ciascun nodo, quindi tornando indietro **usciamo** da ogni nodo.

Adesso andiamo *passo per passo* attraverso il processo che viene eseguito per l'AST descritto in precedenza.

  * Entra nella `FunctionDeclaration` 
      * Entra nell' `Identifier (id)`
      * Vicolo cieco (non ci sono ulteriori nodi dato che id ha un solo valore)
      * Esci dall' `Identifier(id)`
      * Entra nell'`Identifier (params[0])`
      * Vicolo cieco (non ci sono ulteriori nodi dato che id ha un solo valore)
      * Esci dall'`Identifier (params[0])`
      * Entra nel `BlockStatement (body)`
      * Entra nella `ReturnStatement (body)` 
          * Entra nella `BinaryExpression (argument)`
          * Entra nella `Identifier (left)` 
              * Vicolo cieco (non ci sono ulteriori nodi dato che id ha un solo valore)
          * Esci dall'`Identifier (left)`
          * Entra nella `Identifier (right)` 
              * Vicolo cieco (non ci sono ulteriori nodi dato che id ha un solo valore)
          * Esci dall'`Identifier (right)`
          * Esci dalla `BinaryExpression (argument)`
      * Esci dal `ReturnStatement (body)`
      * Esci dal `BlockStatement (body)`
  * Esci dalla `FunctionDeclaration`

Quindi durante la creazione di un visitatore hai due opportunità per visitare il nodo.

```js
const MyVisitor = {
  Identifier: {
    enter() {
      console.log("Entered!");
    },
    exit() {
      console.log("Exited!");
    }
  }
};
```

### <a id="toc-paths"></a>Percorsi (path)

Un AST ha generalmente molti nodi, ma come sono collegati tra di loro? Potremmo esporre un oggetto modificabile con pieno accesso, o possiamo semplificare questo con **percorsi**.

Un **percorso** è una rappresentazione in forma di oggetto del collegamento tra due nodi.

Per esempio se prendiamo il seguente nodo e suo figlio:

```js
{
  type: "FunctionDeclaration",
  id: {
    type: "Identifier",
    name: "square"
  },
  ...
}
```

E rappresentiamo l'`identifier` come un percorso, può essere definito come segue:

```js
{
  "parent": {
    "type": "FunctionDeclaration",
    "id": {...},
    ....
  },
  "node": {
    "type": "Identifier",
    "name": "square"
  }
}
```

Inoltre un percorso ha anche metadati aggiuntivi sul percorso:

```js
{
  "parent": {...},
  "node": {...},
  "hub": {...},
  "contexts": [],
  "data": {},
  "shouldSkip": false,
  "shouldStop": false,
  "removed": false,
  "state": null,
  "opts": null,
  "skipKeys": null,
  "parentPath": null,
  "context": null,
  "container": null,
  "listKey": null,
  "inList": false,
  "parentKey": null,
  "key": null,
  "scope": null,
  "type": null,
  "typeAnnotation": null
}
```

E ha anche molti metodi che permettono di aggiungere, aggiornare, spostare e rimuovere nodi. Questi metodi verranno visitati in sezioni successive.

In un certo senso, i percorsi sono una rappresentazione **reattiva** della posizione di un nodo nell'albero e di tutte le informazioni sul nodo. Ogni volta che si esegue un metodo che modifica la struttura dell'albero, queste informazioni vengono aggiornate. Babel gestisce tutto questo per voi, per rendere il lavoro con i nodi il più semplice possibile.

#### <a id="toc-paths-in-visitors"></a>Oggetto "path" disponibile nei visitatori

Quando si dispone di un visitatore che ha un metodo di `Identifier()`, in realtà stai visitando il percorso invece del nodo. In questo modo si lavora principalmente con la rappresentazione reattiva di un nodo anziché il nodo stesso.

```js
const MyVisitor = {
  Identifier(path) {
    console.log("Visiting: " + path.node.name);
  }
};
```

```js
a + b + c;
```

```js
Visiting: a
Visiting: b
Visiting: c
```

### <a id="toc-state"></a>Stato (state)

Lo "stato" è un nemico quando si parla di trasformazione di AST. Lo stato la morderà più e più volte e le tue ipotesi sullo stato saranno quasi sempre smentite da alcune sintassi che non hai considerato.

Prendete il seguente codice:

```js
function square(n) {
  return n * n;
}
```

Scriviamo un visitatore velocemente che rinomina `n` a `x`.

```js
let paramName;

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    paramName = param.name;
    param.name = "x";
  },

  Identifier(path) {
    if (path.node.name === paramName) {
      path.node.name = "x";
    }
  }
};
```

Questo potrebbe funzionare per il codice sopra riportato, ma potrebbe facilmente non funzionare con questo codice:

```js
function square(n) {
  return n * n;
}
n;
```

Il modo migliore per affrontare questo problema è la ricorsione. Quindi cerchiamo di imitare un film di Christopher Nolan e mettere un visitatore all'interno di un visitatore.

```js
const updateParamNameVisitor = {
  Identifier(path) {
    if (path.node.name === this.paramName) {
      path.node.name = "x";
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    const param = path.node.params[0];
    const paramName = param.name;
    param.name = "x";

    path.traverse(updateParamNameVisitor, { paramName });
  }
};
```

Naturalmente, questo è solo un esempio ma viene illustrato come eliminare lo stato globale dai tuoi visitatori.

### <a id="toc-scopes"></a>Scope

Adesso introduciamo il concetto di [**scope**](https://en.wikipedia.org/wiki/Scope_(computer_science)). JavaScript ha uno [scoping lessicale](https://en.wikipedia.org/wiki/Scope_(computer_science)#Lexical_scoping_vs._dynamic_scoping), che è una struttura ad albero dove i blocchi creano nuovo scope.

```js
// global scope

function scopeOne() {
  // scope 1

  function scopeTwo() {
    // scope 2
  }
}
```

Ogni volta che si crea un riferimento in JavaScript, sia che si tratti di una variabile, funzione, classe, parametro, import, etichetta, ecc., appartiene allo scope corrente.

```js
var global = "I am in the global scope";

function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var two = "I am in the scope created by `scopeTwo()`";
  }
}
```

Il codice all'interno di uno scope più profondo può utilizzare un riferimento da uno scope superiore.

```js
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    one = "I am updating the reference in `scopeOne` inside `scopeTwo`";
  }
}
```

Uno scope inferiore potrebbe anche creare un riferimento dello stesso nome senza modificarlo.

```js
function scopeOne() {
  var one = "I am in the scope created by `scopeOne()`";

  function scopeTwo() {
    var one = "I am creating a new `one` but leaving reference in `scopeOne()` alone.";
  }
}
```

Quando scriviamo un file di trasformazione, bisogna essere cauti nei confronti dello scope. Dobbiamo essere certi di non rompere il codice esistente durante la modifica di diverse parti del codice.

We may want to add new references and make sure they don't collide with existing ones. Or maybe we just want to find where a variable is referenced. We want to be able to track these references within a given scope.

A scope can be represented as:

```js
{
  path: path,
  block: path.node,
  parentBlock: path.parent,
  parent: parentScope,
  bindings: [...]
}
```

When you create a new scope you do so by giving it a path and a parent scope. Then during the traversal process it collects all the references ("bindings") within that scope.

Once that's done, there's all sorts of methods you can use on scopes. We'll get into those later though.

#### <a id="toc-bindings"></a>Binding

References all belong to a particular scope; this relationship is known as a **binding**.

```js
function scopeOnce() {
  var ref = "This is a binding";

  ref; // This is a reference to a binding

  function scopeTwo() {
    ref; // This is a reference to a binding from a lower scope
  }
}
```

A single binding looks like this:

```js
{
  identifier: node,
  scope: scope,
  path: path,
  kind: 'var',

  referenced: true,
  references: 3,
  referencePaths: [path, path, path],

  constant: false,
  constantViolations: [path]
}
```

With this information you can find all the references to a binding, see what type of binding it is (parameter, declaration, etc.), lookup what scope it belongs to, or get a copy of its identifier. You can even tell if it's constant and if not, see what paths are causing it to be non-constant.

Being able to tell if a binding is constant is useful for many purposes, the largest of which is minification.

```js
function scopeOne() {
  var ref1 = "This is a constant binding";

  becauseNothingEverChangesTheValueOf(ref1);

  function scopeTwo() {
    var ref2 = "This is *not* a constant binding";
    ref2 = "Because this changes the value";
  }
}
```

* * *

# <a id="toc-api"></a>API

Babel is actually a collection of modules. In this section we'll walk through the major ones, explaining what they do and how to use them.

> Note: This is not a replacement for detailed API documentation which will be available elsewhere shortly.

## <a id="toc-babylon"></a>[`babylon`](https://github.com/babel/babel/tree/master/packages/babylon)

Babylon is Babel's parser. Started as a fork of Acorn, it's fast, simple to use, has plugin-based architecture for non-standard features (as well as future standards).

First, let's install it.

```sh
$ npm install --save babylon
```

Let's start by simply parsing a string of code:

```js
import * as babylon from "babylon";

const code = `function square(n) {
  return n * n;
}`;

babylon.parse(code);
// Node {
//   type: "File",
//   start: 0,
//   end: 38,
//   loc: SourceLocation {...},
//   program: Node {...},
//   comments: [],
//   tokens: [...]
// }
```

We can also pass options to `parse()` like so:

```js
babylon.parse(code, {
  sourceType: "module", // default: "script"
  plugins: ["jsx"] // default: []
});
```

`sourceType` can either be `"module"` or `"script"` which is the mode that Babylon should parse in. `"module"` will parse in strict mode and allow module declarations, `"script"` will not.

> **Note:** `sourceType` defaults to `"script"` and will error when it finds `import` or `export`. Pass `sourceType: "module"` to get rid of these errors.

Since Babylon is built with a plugin-based architecture, there is also a `plugins` option which will enable the internal plugins. Note that Babylon has not yet opened this API to external plugins, although may do so in the future.

To see a full list of plugins, see the [Babylon README](https://github.com/babel/babel/blob/master/packages/babylon/README.md#plugins).

## <a id="toc-babel-traverse"></a>[`babel-traverse`](https://github.com/babel/babel/tree/master/packages/babel-traverse)

The Babel Traverse module maintains the overall tree state, and is responsible for replacing, removing, and adding nodes.

Install it by running:

```sh
$ npm install --save babel-traverse
```

We can use it alongside Babylon to traverse and update nodes:

```js
import * as babylon from "babylon";
import traverse from "babel-traverse";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

traverse(ast, {
  enter(path) {
    if (
      path.node.type === "Identifier" &&
      path.node.name === "n"
    ) {
      path.node.name = "x";
    }
  }
});
```

## <a id="toc-babel-types"></a>[`babel-types`](https://github.com/babel/babel/tree/master/packages/babel-types)

Babel Types is a Lodash-esque utility library for AST nodes. It contains methods for building, validating, and converting AST nodes. It's useful for cleaning up AST logic with well thought out utility methods.

You can install it by running:

```sh
$ npm install --save babel-types
```

Then start using it:

```js
import traverse from "babel-traverse";
import * as t from "babel-types";

traverse(ast, {
  enter(path) {
    if (t.isIdentifier(path.node, { name: "n" })) {
      path.node.name = "x";
    }
  }
});
```

### <a id="toc-definitions"></a>Definizioni

Babel Types has definitions for every single type of node, with information on what properties belong where, what values are valid, how to build that node, how the node should be traversed, and aliases of the Node.

A single node type definition looks like this:

```js
defineType("BinaryExpression", {
  builder: ["operator", "left", "right"],
  fields: {
    operator: {
      validate: assertValueType("string")
    },
    left: {
      validate: assertNodeType("Expression")
    },
    right: {
      validate: assertNodeType("Expression")
    }
  },
  visitor: ["left", "right"],
  aliases: ["Binary", "Expression"]
});
```

### <a id="toc-builders"></a>Costruttori

You'll notice the above definition for `BinaryExpression` has a field for a `builder`.

```js
builder: ["operator", "left", "right"]
```

This is because each node type gets a builder method, which when used looks like this:

```js
t.binaryExpression("*", t.identifier("a"), t.identifier("b"));
```

Which creates an AST like this:

```js
{
  type: "BinaryExpression",
  operator: "*",
  left: {
    type: "Identifier",
    name: "a"
  },
  right: {
    type: "Identifier",
    name: "b"
  }
}
```

Which when printed looks like this:

```js
a * b
```

Builders will also validate the nodes they are creating and throw descriptive errors if used improperly. Which leads into the next type of method.

### <a id="toc-validators"></a>Validatori

The definition for `BinaryExpression` also includes information on the `fields` of a node and how to validate them.

```js
fields: {
  operator: {
    validate: assertValueType("string")
  },
  left: {
    validate: assertNodeType("Expression")
  },
  right: {
    validate: assertNodeType("Expression")
  }
}
```

This is used to create two types of validating methods. The first of which is `isX`.

```js
t.isBinaryExpression(maybeBinaryExpressionNode);
```

This tests to make sure that the node is a binary expression, but you can also pass a second parameter to ensure that the node contains certain properties and values.

```js
t.isBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
```

There is also the more, *ehem*, assertive version of these methods, which will throw errors instead of returning `true` or `false`.

```js
t.assertBinaryExpression(maybeBinaryExpressionNode);
t.assertBinaryExpression(maybeBinaryExpressionNode, { operator: "*" });
// Error: Expected type "BinaryExpression" with option { "operator": "*" }
```

### <a id="toc-converters"></a>Convertitori

> [WIP]

## <a id="toc-babel-generator"></a>[`babel-generator`](https://github.com/babel/babel/tree/master/packages/babel-generator)

Babel Generator is the code generator for Babel. It takes an AST and turns it into code with sourcemaps.

Run the following to install it:

```sh
$ npm install --save babel-generator
```

Then use it

```js
import * as babylon from "babylon";
import generate from "babel-generator";

const code = `function square(n) {
  return n * n;
}`;

const ast = babylon.parse(code);

generate(ast, null, code);
// {
//   code: "...",
//   map: "..."
// }
```

You can also pass options to `generate()`.

```js
generate(ast, {
  retainLines: false,
  compact: "auto",
  concise: false,
  quotes: "double",
  // ...
}, code);
```

## <a id="toc-babel-template"></a>[`babel-template`](https://github.com/babel/babel/tree/master/packages/babel-template)

Babel Template is another tiny but incredibly useful module. It allows you to write strings of code with placeholders that you can use instead of manually building up a massive AST.

```sh
$ npm install --save babel-template
```

```js
import template from "babel-template";
import generate from "babel-generator";
import * as t from "babel-types";

const buildRequire = template(`
  var IMPORT_NAME = require(SOURCE);
`);

const ast = buildRequire({
  IMPORT_NAME: t.identifier("myModule"),
  SOURCE: t.stringLiteral("my-module")
});

console.log(generate(ast).code);
```

```js
var myModule = require("my-module");
```

# <a id="toc-writing-your-first-babel-plugin"></a>Scrivere il primo plugin per Babel

Now that you're familiar with all the basics of Babel, let's tie it together with the plugin API.

Start off with a `function` that gets passed the current [`babel`](https://github.com/babel/babel/tree/master/packages/babel-core) object.

```js
export default function(babel) {
  // plugin contents
}
```

Since you'll be using it so often, you'll likely want to grab just `babel.types` like so:

```js
export default function({ types: t }) {
  // plugin contents
}
```

Then you return an object with a property `visitor` which is the primary visitor for the plugin.

```js
export default function({ types: t }) {
  return {
    visitor: {
      // visitor contents
    }
  };
};
```

Let's write a quick plugin to show off how it works. Here's our source code:

```js
foo === bar;
```

Or in AST form:

```js
{
  type: "BinaryExpression",
  operator: "===",
  left: {
    type: "Identifier",
    name: "foo"
  },
  right: {
    type: "Identifier",
    name: "bar"
  }
}
```

We'll start off by adding a `BinaryExpression` visitor method.

```js
export default function({ types: t }) {
  return {
    visitor: {
      BinaryExpression(path) {
        // ...
      }
    }
  };
}
```

Then let's narrow it down to just `BinaryExpression`s that are using the `===` operator.

```js
visitor: {
  BinaryExpression(path) {
    if (path.node.operator !== "===") {
      return;
    }

    // ...
  }
}
```

Now let's replace the `left` property with a new identifier:

```js
BinaryExpression(path) {
  if (path.node.operator !== "===") {
    return;
  }

  path.node.left = t.identifier("sebmck");
  // ...
}
```

Already if we run this plugin we would get:

```js
sebmck === bar;
```

Now let's just replace the `right` property.

```js
BinaryExpression(path) {
  if (path.node.operator !== "===") {
    return;
  }

  path.node.left = t.identifier("sebmck");
  path.node.right = t.identifier("dork");
}
```

And now for our final result:

```js
sebmck === dork;
```

Awesome! Our very first Babel plugin.

* * *

# <a id="toc-transformation-operations"></a>Operazioni di trasformazione

## <a id="toc-visiting"></a>Navigazione AST

### <a id="toc-check-if-a-node-is-a-certain-type"></a>Verificare se un nodo è di uno specifico tipo

If you want to check what the type of a node is, the preferred way to do so is:

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left)) {
    // ...
  }
}
```

You can also do a shallow check for properties on that node:

```js
BinaryExpression(path) {
  if (t.isIdentifier(path.node.left, { name: "n" })) {
    // ...
  }
}
```

This is functionally equivalent to:

```js
BinaryExpression(path) {
  if (
    path.node.left != null &&
    path.node.left.type === "Identifier" &&
    path.node.left.name === "n"
  ) {
    // ...
  }
}
```

### <a id="toc-check-if-an-identifier-is-referenced"></a>Verificare se esistono referenze ad un identifier

```js
Identifier(path) {
  if (path.isReferencedIdentifier()) {
    // ...
  }
}
```

Alternatively:

```js
Identifier(path) {
  if (t.isReferenced(path.node, path.parent)) {
    // ...
  }
}
```

## <a id="toc-manipulation"></a>Manipolazione AST

### <a id="toc-replacing-a-node"></a>Sostituire un nodo

```js
BinaryExpression(path) {
  path.replaceWith(
    t.binaryExpression("**", path.node.left, t.numberLiteral(2))
  );
}
```

```diff
  function square(n) {
-   return n * n;
+   return n ** 2;
  }
```

### <a id="toc-replacing-a-node-with-multiple-nodes"></a>Sostituire un nodo con più nodi

```js
ReturnStatement(path) {
  path.replaceWithMultiple([
    t.expressionStatement(t.stringLiteral("Is this the real life?")),
    t.expressionStatement(t.stringLiteral("Is this just fantasy?")),
    t.expressionStatement(t.stringLiteral("(Enjoy singing the rest of the song in your head)")),
  ]);
}
```

```diff
  function square(n) {
-   return n * n;
+   "Is this the real life?";
+   "Is this just fantasy?";
+   "(Enjoy singing the rest of the song in your head)";
  }
```

> **Note:** When replacing an expression with multiple nodes, they must be statements. This is because Babel uses heuristics extensively when replacing nodes which means that you can do some pretty crazy transformations that would be extremely verbose otherwise.

### <a id="toc-replacing-a-node-with-a-source-string"></a>Sostituire un nodo con una stringa di codice

```js
FunctionDeclaration(path) {
  path.replaceWithSourceString(`function add(a, b) {
    return a + b;
  }`);
}
```

```diff
- function square(n) {
-   return n * n;
+ function add(a, b) {
+   return a + b;
  }
```

> **Note:** It's not recommended to use this API unless you're dealing with dynamic source strings, otherwise it's more efficient to parse the code outside of the visitor.

### <a id="toc-inserting-a-sibling-node"></a>Inserire un nodo adiacente

```js
FunctionDeclaration(path) {
  path.insertBefore(t.expressionStatement(t.stringLiteral("Because I'm easy come, easy go.")));
  path.insertAfter(t.expressionStatement(t.stringLiteral("A little high, little low.")));
}
```

```diff
+ "Because I'm easy come, easy go.";
  function square(n) {
    return n * n;
  }
+ "A little high, little low.";
```

> **Note:** This should always be a statement or an array of statements. This uses the same heuristics mentioned in [Replacing a node with multiple nodes](#replacing-a-node-with-multiple-nodes).

### <a id="toc-removing-a-node"></a>Rimuovere un nodo

```js
FunctionDeclaration(path) {
  path.remove();
}
```

```diff
- function square(n) {
-   return n * n;
- }
```

### <a id="toc-replacing-a-parent"></a>Sostituire un nodo padre

```js
BinaryExpression(path) {
  path.parentPath.replaceWith(
    t.expressionStatement(t.stringLiteral("Anyway the wind blows, doesn't really matter to me, to me."))
  );
}
```

```diff
  function square(n) {
-   return n * n;
+   "Anyway the wind blows, doesn't really matter to me, to me.";
  }
```

### <a id="toc-removing-a-parent"></a>Rimuovere un nodo padre

```js
BinaryExpression(path) {
  path.parentPath.remove();
}
```

```diff
  function square(n) {
-   return n * n;
  }
```

## <a id="toc-scope"></a>Scope (e variabili)

### <a id="toc-checking-if-a-local-variable-is-bound"></a>Verificare se una variabile locale è associata ad uno scope

```js
FunctionDeclaration(path) {
  if (path.scope.hasBinding("n")) {
    // ...
  }
}
```

This will walk up the scope tree and check for that particular binding.

You can also check if a scope has its **own** binding:

```js
FunctionDeclaration(path) {
  if (path.scope.hasOwnBinding("n")) {
    // ...
  }
}
```

### <a id="toc-generating-a-uid"></a>Generare un nuovo nome di variabile (UID)

This will generate an identifier that doesn't collide with any locally defined variables.

```js
FunctionDeclaration(path) {
  path.scope.generateUidIdentifier("uid");
  // Node { type: "Identifier", name: "_uid" }
  path.scope.generateUidIdentifier("uid");
  // Node { type: "Identifier", name: "_uid2" }
}
```

### <a id="toc-pushing-a-variable-declaration-to-a-parent-scope"></a>Promuovere la dichiarazione di una variabile ad uno scope padre

Sometimes you may want to push a `VariableDeclaration` so you can assign to it.

```js
FunctionDeclaration(path) {
  const id = path.scope.generateUidIdentifierBasedOnNode(path.node.id);
  path.remove();
  path.scope.parent.push({ id, init: path.node });
}
```

```diff
- function square(n) {
+ var _square = function square(n) {
    return n * n;
- }
+ };
```

### <a id="toc-rename-a-binding-and-its-references"></a>Rinominare una variabile (e tutte le sue referenze)

```js
FunctionDeclaration(path) {
  path.scope.rename("n", "x");
}
```

```diff
- function square(n) {
-   return n * n;
+ function square(x) {
+   return x * x;
  }
```

Alternatively, you can rename a binding to a generated unique identifier:

```js
FunctionDeclaration(path) {
  path.scope.rename("n");
}
```

```diff
- function square(n) {
-   return n * n;
+ function square(_n) {
+   return _n * _n;
  }
```

* * *

# <a id="toc-plugin-options"></a>Opzioni per i propri plugin

If you would like to let your users customize the behavior of your Babel plugin you can accept plugin specific options which users can specify like this:

```js
{
  plugins: [
    ["my-plugin", {
      "option1": true,
      "option2": false
    }]
  ]
}
```

These options then get passed into plugin visitors through the `state` object:

```js
export default function({ types: t }) {
  return {
    visitor: {
      FunctionDeclaration(path, state) {
        console.log(state.opts);
        // { option1: true, option2: false }
      }
    }
  }
}
```

These options are plugin-specific and you cannot access options from other plugins.

* * *

# <a id="toc-building-nodes"></a>Generare nodi (per inserimenti e trasformazioni)

When writing transformations you'll often want to build up some nodes to insert into the AST. As mentioned previously, you can do this using the [builder](#builder) methods in the [`babel-types`](#babel-types) package.

The method name for a builder is simply the name of the node type you want to build except with the first letter lowercased. For example if you wanted to build a `MemberExpression` you would use `t.memberExpression(...)`.

The arguments of these builders are decided by the node definition. There's some work that's being done to generate easy-to-read documentation on the definitions, but for now they can all be found [here](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions).

A node definition looks like the following:

```js
defineType("MemberExpression", {
  builder: ["object", "property", "computed"],
  visitor: ["object", "property"],
  aliases: ["Expression", "LVal"],
  fields: {
    object: {
      validate: assertNodeType("Expression")
    },
    property: {
      validate(node, key, val) {
        let expectedType = node.computed ? "Expression" : "Identifier";
        assertNodeType(expectedType)(node, key, val);
      }
    },
    computed: {
      default: false
    }
  }
});
```

Here you can see all the information about this particular node type, including how to build it, traverse it, and validate it.

By looking at the `builder` property, you can see the 3 arguments that will be needed to call the builder method (`t.memberExpression`).

```js
builder: ["object", "property", "computed"],
```

> Note that sometimes there are more properties that you can customize on the node than the `builder` array contains. This is to keep the builder from having too many arguments. In these cases you need to set the properties manually. An example of this is [`ClassMethod`](https://github.com/babel/babel/blob/bbd14f88c4eea88fa584dd877759dd6b900bf35e/packages/babel-types/src/definitions/es2015.js#L238-L276).

You can see the validation for the builder arguments with the `fields` object.

```js
fields: {
  object: {
    validate: assertNodeType("Expression")
  },
  property: {
    validate(node, key, val) {
      let expectedType = node.computed ? "Expression" : "Identifier";
      assertNodeType(expectedType)(node, key, val);
    }
  },
  computed: {
    default: false
  }
}
```

You can see that `object` needs to be an `Expression`, `property` either needs to be an `Expression` or an `Identifier` depending on if the member expression is `computed` or not and `computed` is simply a boolean that defaults to `false`.

So we can construct a `MemberExpression` by doing the following:

```js
t.memberExpression(
  t.identifier('object'),
  t.identifier('property')
  // `computed` is optional
);
```

Which will result in:

```js
object.property
```

However, we said that `object` needed to be an `Expression` so why is `Identifier` valid?

Well if we look at the definition of `Identifier` we can see that it has an `aliases` property which states that it is also an expression.

```js
aliases: ["Expression", "LVal"],
```

So since `MemberExpression` is a type of `Expression`, we could set it as the `object` of another `MemberExpression`:

```js
t.memberExpression(
  t.memberExpression(
    t.identifier('member'),
    t.identifier('expression')
  ),
  t.identifier('property')
)
```

Which will result in:

```js
member.expression.property
```

It's very unlikely that you will ever memorize the builder method signatures for every node type. So you should take some time and understand how they are generated from the node definitions.

You can find all of the actual [definitions here](https://github.com/babel/babel/tree/master/packages/babel-types/src/definitions) and you can see them [documented here](https://github.com/babel/babel/blob/master/doc/ast/spec.md)

* * *

# <a id="toc-best-practices"></a>Best practice

> I'll be working on this section over the coming weeks.

## <a id="toc-avoid-traversing-the-ast-as-much-as-possible"></a>Evitare navigazioni non necessarie dell'AST

Traversing the AST is expensive, and it's easy to accidentally traverse the AST more than necessary. This could be thousands if not tens of thousands of extra operations.

Babel optimizes this as much as possible, merging visitors together if it can in order to do everything in a single traversal.

### <a id="toc-merge-visitors-whenever-possible"></a>Aggregare i visitatori quando possibile

When writing visitors, it may be tempting to call `path.traverse` in multiple places where they are logically necessary.

```js
path.traverse({
  Identifier(path) {
    // ...
  }
});

path.traverse({
  BinaryExpression(path) {
    // ...
  }
});
```

However, it is far better to write these as a single visitor that only gets run once. Otherwise you are traversing the same tree multiple times for no reason.

```js
path.traverse({
  Identifier(path) {
    // ...
  },
  BinaryExpression(path) {
    // ...
  }
});
```

### <a id="toc-do-not-traverse-when-manual-lookup-will-do"></a>Evitare navigazioni programmatiche nidificate

It may also be tempting to call `path.traverse` when looking for a particular node type.

```js
const visitorOne = {
  Identifier(path) {
    // ...
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    path.get('params').traverse(visitorOne);
  }
};
```

However, if you are looking for something specific and shallow, there is a good chance you can manually lookup the nodes you need without performing a costly traversal.

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    path.node.params.forEach(function() {
      // ...
    });
  }
};
```

## <a id="toc-optimizing-nested-visitors"></a>Ottimizzare i visitatori nidificati

When you are nesting visitors, it might make sense to write them nested in your code.

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    path.traverse({
      Identifier(path) {
        // ...
      }
    });
  }
};
```

However, this creates a new visitor object everytime `FunctionDeclaration()` is called above, which Babel then needs to explode and validate every single time. This can be costly, so it is better to hoist the visitor up.

```js
const visitorOne = {
  Identifier(path) {
    // ...
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    path.traverse(visitorOne);
  }
};
```

If you need some state within the nested visitor, like so:

```js
const MyVisitor = {
  FunctionDeclaration(path) {
    var exampleState = path.node.params[0].name;

    path.traverse({
      Identifier(path) {
        if (path.node.name === exampleState) {
          // ...
        }
      }
    });
  }
};
```

You can pass it in as state to the `traverse()` method and have access to it on `this` in the visitor.

```js
const visitorOne = {
  Identifier(path) {
    if (path.node.name === this.exampleState) {
      // ...
    }
  }
};

const MyVisitor = {
  FunctionDeclaration(path) {
    var exampleState = path.node.params[0].name;
    path.traverse(visitorOne, { exampleState });
  }
};
```

## <a id="toc-being-aware-of-nested-structures"></a>Essere consapevoli delle strutture nidificate

Sometimes when thinking about a given transform, you might forget that the given structure can be nested.

For example, imagine we want to lookup the `constructor` `ClassMethod` from the `Foo` `ClassDeclaration`.

```js
class Foo {
  constructor() {
    // ...
  }
}
```

```js
const constructorVisitor = {
  ClassMethod(path) {
    if (path.node.name === 'constructor') {
      // ...
    }
  }
}

const MyVisitor = {
  ClassDeclaration(path) {
    if (path.node.id.name === 'Foo') {
      path.traverse(constructorVisitor);
    }
  }
}
```

We are ignoring the fact that classes can be nested and using the traversal above we will hit a nested `constructor` as well:

```js
class Foo {
  constructor() {
    class Bar {
      constructor() {
        // ...
      }
    }
  }
}
```

> ***Per aggiornamenti futuri, segui [@thejameskyle](https://twitter.com/thejameskyle) su Twitter.***