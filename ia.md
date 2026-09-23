# Registro de uso de IA

Ferramenta: Claude (Claude Code). Data: 23/09/2026.

## 1. Estudo inicial da tarefa

**Pergunta:** enviei o enunciado e o código de partida e pedi um estudo completo antes de alterar o código.

**Resposta (resumo):** levantou os dados de `data()`, as diretivas e os métodos, e apontou problemas do original: excluir só existe em pendentes, texto salvo sem `trim`, Enter não adiciona e o modo escuro não pinta a página inteira. Alertou que, ao filtrar a lista, a exclusão por índice apagaria a tarefa errada.

**Aproveitei:** sim. Serviu de base para o Raio X, que revisei testando o `original.html`.

## 2. Decisões discutidas com a IA

Antes de gerar o código, a IA me perguntou como eu queria resolver alguns pontos. Minhas escolhas:

| Ponto | Opções apresentadas | Minha escolha |
|---|---|---|
| Identificar cada tarefa (key e exclusão) | Criar campo `id` / usar `indexOf(tarefa)` sem mudar os dados | Campo `id`, que também serve de `key` |
| Busca e mensagem | Ignorar maiúsculas / também ignorar acentos / aviso por seção | Ignorar maiúsculas, com uma mensagem única quando nada é encontrado |
| Adicionar tarefa com filtro ativo | Manter o filtro e documentar / limpar a busca ao adicionar | Manter o filtro e documentar |
| Escopo | Código + README, registro de IA, prints automáticos, corrigir bugs do original | Código, README e registro de IA; bugs do original só documentados; testes feitos por mim |

## 3. Perguntas para entender melhor a solução

### 3.1 Qual a diferença entre `computed` e `methods`? Por que o filtro não pode ser um método?

**Resposta:** um *método* é uma função que roda toda vez que é chamada. Um *computed* é um valor derivado de outros dados: o Vue sabe de quais dados ele depende e só recalcula quando algum deles muda.

O filtro até *poderia* ser um método (`v-for="tarefa in filtrar()"`) e funcionaria, mas:
- como o template usa o resultado em três lugares (as duas listas e o aviso), o método rodaria três vezes a cada renderização, e também quando mudasse algo que não tem nada a ver com o filtro, como o `modoEscuro` ou o texto de `novaTarefa`;
- o enunciado pede o filtro "como um valor calculado", que é exatamente o `computed`;
- no template, `tarefasFiltradas` se lê como um dado (o que a tela mostra), e não como uma ação.

Regra prática: **`methods` para ações** (adicionar, excluir), **`computed` para valores derivados** (lista filtrada, contagens).

**Aproveitei:** sim, é o argumento principal da decisão técnica no README.

### 3.2 O que é o cache do `computed` e quando ele é recalculado?

**Resposta:** na primeira vez que `tarefasFiltradas` é lido, o Vue executa a função e registra quais dados reativos foram acessados: `this.busca`, `this.tarefas` e o `texto` de cada tarefa. O resultado fica guardado. Nas próximas leituras, se nada disso mudou, o Vue devolve o valor guardado sem executar o `filter` de novo.

Ele é recalculado quando:
- o usuário digita na busca (`busca` muda);
- uma tarefa é adicionada ou excluída (`push`/`splice` em `tarefas`);
- o texto de alguma tarefa muda.

Ligar o modo escuro ou digitar em "Nova tarefa" **não** recalcula o filtro, porque ele não depende desses dados.

**Aproveitei:** sim, para explicar por que o `computed` é mais eficiente.

### 3.3 Por que o `filter` não altera o array original? O que aconteceria se eu usasse `splice` para filtrar?

**Resposta:** `filter` é um método de array que **cria um array novo** com os itens que passam no teste e não toca no original. Já `splice` **modifica o próprio array**, removendo ou inserindo itens.

Se eu usasse `splice` para "filtrar", as tarefas que não batem com a busca seriam **apagadas de verdade** de `tarefas`. Ao limpar a busca, elas não voltariam, porque foram perdidas. É isso que o enunciado proíbe com "sem alterar os dados apenas para exibir o resultado". Com `filter`, `tarefas` continua completo e a busca só muda o que é exibido.

**Aproveitei:** sim, usei no README e é o motivo de o teste 4b (tarefa que reaparece ao limpar a busca) funcionar.

### 3.4 Por que marcar o checkbox numa lista filtrada altera a tarefa original?

**Resposta:** em JavaScript, objetos são guardados **por referência**. O array novo criado pelo `filter` não contém cópias das tarefas, contém **os mesmos objetos** que estão em `tarefas`. É como ter duas listas de endereços apontando para as mesmas casas.

Quando o `v-model="tarefa.concluida"` muda o objeto pela lista filtrada, está mudando o mesmo objeto que está em `tarefas`. O Vue detecta a mudança, o `v-if` passa a mostrar a tarefa em "Concluídas" e, ao limpar a busca, ela continua concluída.

A exclusão é diferente: ela precisa remover o item **do array** `tarefas`, e não só mudar o objeto. Por isso ela usa o `id` para achar a posição no array original.

**Aproveitei:** sim, explica o teste 5a.

### 3.5 Para que serve a `key` no `v-for`, e o que dá errado usando o índice?

**Resposta:** quando a lista muda, o Vue tenta **reaproveitar** os elementos HTML que já existem em vez de recriar tudo. A `key` diz ao Vue "este `<li>` pertence a esta tarefa". Com uma key estável (o `id`), o Vue sabe exatamente qual elemento remover, mover ou manter.

Com o índice como key, a key representa a **posição**, não a tarefa. Se eu excluo o primeiro item, todos os outros "mudam de key" (o que era 1 vira 0) e o Vue pode reaproveitar o elemento errado. Com filtro fica pior: o índice é a posição **na lista filtrada**, então `excluirTarefa(indice)` fazia `splice` na posição errada de `tarefas` e apagava outra tarefa. Exemplo: com `[Estudar, Ler capítulo, Estudar Vue]` e busca "vue", "Estudar Vue" aparece com índice 0, e excluir apagaria "Estudar".

**Aproveitei:** sim, motivou o campo `id` e o teste 5b.

### 3.6 Por que o `v-if` está num `<li>` dentro de um `<template v-for>` e não no mesmo elemento do `v-for`?

**Resposta:** no Vue 3, quando `v-if` e `v-for` estão no **mesmo elemento**, o `v-if` é avaliado **antes**. Nesse momento a variável `tarefa` do `v-for` ainda não existe, então `v-if="!tarefa.concluida"` daria erro. A documentação do Vue recomenda não misturar os dois no mesmo elemento.

A solução do código original é usar um `<template>` para o `v-for`. O `<template>` não gera nenhuma tag no HTML final, só agrupa. O `v-if` fica no `<li>` de dentro, onde `tarefa` já existe. Uma alternativa seria criar dois `computed` (pendentes e concluídas) e eliminar o `v-if`, mas mantive a estrutura original para mexer menos no código.

**Aproveitei:** sim, para o Raio X, item 3.

### 3.7 Como o Vue sabe que precisa atualizar a tela quando faço `push` no array?

**Resposta:** o que é retornado em `data()` é transformado pelo Vue em um objeto **reativo**. No Vue 3 isso é feito com um `Proxy` do JavaScript, um "intermediário" que intercepta toda leitura e escrita nos dados.

- Quando o template (ou um `computed`) **lê** `tarefas`, o Proxy anota que aquele trecho depende de `tarefas`.
- Quando um método **escreve**, com `push`, `splice` ou `tarefa.concluida = true`, o Proxy percebe a mudança e avisa todos que dependem daquele dado.
- O Vue então redesenha só as partes afetadas.

Por isso os métodos nunca mexem no HTML diretamente: basta mudar os dados que a tela acompanha.

**Aproveitei:** sim, para o Raio X, item 5.

### 3.8 Qual a diferença entre `v-if` e `v-show`? Qual faz mais sentido para o aviso de "nenhuma tarefa"?

**Resposta:**
- `v-if` **cria ou remove** o elemento do HTML. Quando a condição é falsa, o elemento nem existe na página.
- `v-show` sempre cria o elemento e só alterna o CSS `display: none`.

`v-show` compensa quando algo alterna muitas vezes e é caro de criar. `v-if` é melhor quando a condição muda pouco ou quando o elemento normalmente não deve existir.

Para o aviso, os dois funcionariam. Usei `v-if` porque o aviso só aparece em um caso específico, porque é a diretiva que está nos PDFs e porque é a mesma usada no resto do código. Além disso, o aviso interpola `{{ busca }}`: com `v-if`, esse trecho nem é processado quando não é exibido.

**Aproveitei:** sim, justificativa do `v-if` no aviso.

### 3.9 Como eu faria a busca ignorar acentos, e por que ficou fora da entrega?

**Resposta:** daria para "limpar" o texto antes de comparar:

```js
texto.normalize('NFD').replace(/[̀-ͯ]/g, '').toLowerCase()
```

O `normalize('NFD')` separa a letra do acento ("í" vira "i" + "´") e o `replace` com expressão regular remove os acentos. Assim "capitulo" encontraria "capítulo".

Ficou fora porque `normalize` e expressões regulares não estão nos PDFs da disciplina. O enunciado pede para usar esse conhecimento, e eu teria que explicar um código que não foi trabalhado em aula. Fica registrado como melhoria futura.

**Aproveitei:** não no código; a limitação foi documentada.

### 3.10 Que perguntas o professor pode fazer na apresentação, e como eu responderia?

**Resposta:** perguntas prováveis e respostas curtas:

| Pergunta provável | Resposta |
|---|---|
| Por que `computed` e não um método? | O filtro é um valor derivado de `tarefas` e `busca`; o `computed` recalcula sozinho quando eles mudam e guarda o resultado em cache. É o que o enunciado pede. |
| Onde está a lista filtrada guardada? | Em lugar nenhum de `data()`. Ela é calculada a partir de `tarefas`, e não duplico nada. |
| Por que com a busca vazia aparecem todas? | `includes('')` é sempre verdadeiro, então nenhuma tarefa é filtrada. |
| Por que você criou o `id`? | Com filtro, o índice da tela deixa de ser o índice do array. Sem `id`, excluir apagaria a tarefa errada. |
| Adicionei uma tarefa e ela não apareceu. É bug? | Não. Ela foi adicionada em `tarefas`, mas não contém o termo da busca. Ao limpar a busca, aparece. |
| Por que marcar como concluída na busca funciona? | O `filter` devolve os mesmos objetos (referência), então o `v-model` altera o objeto original. |
| Mostre o fluxo quando digito na busca. | `v-model` → `busca` muda → `tarefasFiltradas` recalcula → os `v-for` redesenham → o `v-if` do aviso é avaliado. |
| O que você mudaria se o produto crescesse? | Separar pendentes e concluídas em dois `computed`, salvar as tarefas no navegador e permitir excluir as concluídas. |

Dica: saber abrir o `index.html`, digitar na busca e excluir uma tarefa filtrada ao vivo, explicando cada passo.

**Aproveitei:** sim, para me preparar para a apresentação.

## 4. Perguntas sobre possíveis alterações no código

Perguntei como eu faria outras mudanças no produto para praticar e entender melhor o código. **Nenhuma delas foi aplicada no `index.html`**, porque o enunciado pede uma adição pequena (a busca). Ficam como estudo e ideias para evoluir o produto.

### Melhorias na busca

#### 4.1 Como mostrar quantas tarefas foram encontradas? (Exemplo: "2 de 5 tarefas".)

**Resposta:** basta interpolar o tamanho dos dois arrays. Como `tarefasFiltradas` é um `computed`, o número atualiza sozinho a cada tecla:

```html
<small v-if="busca.trim()">
  {{ tarefasFiltradas.length }} de {{ tarefas.length }} tarefas
</small>
```

O `v-if` faz o contador aparecer só quando há busca. Sem busca, os dois números seriam iguais e a informação não teria utilidade.

#### 4.2 Como colocar um botão "limpar busca" que só aparece quando há texto no campo?

**Resposta:** colocar o campo num `input-group` do Bootstrap e usar `v-if` no botão. O clique só precisa zerar o dado: o `v-model` esvazia o campo e o `computed` volta a mostrar tudo.

```html
<div class="input-group mb-3">
  <input type="search" class="form-control" v-model="busca" placeholder="Buscar tarefa">
  <button class="btn btn-outline-secondary" v-if="busca" v-on:click="busca = ''">limpar</button>
</div>
```

Não precisa criar um método, porque a ação é uma única atribuição. Se crescesse (limpar a busca e também o filtro de status, por exemplo), valeria criar um método `limparBusca()`.

#### 4.3 Como destacar o termo buscado dentro do texto da tarefa?

**Resposta:** a tentação é usar `v-html` montando `<mark>` numa string, mas isso é **perigoso**: `v-html` insere HTML sem proteção, e uma tarefa com o texto `<img src=x onerror=alert(1)>` executaria código (XSS). O jeito seguro é dividir o texto em três pedaços e deixar a interpolação `{{ }}` cuidar do resto, porque ela sempre trata o conteúdo como texto:

```js
methods: {
  destacar(texto) {
    const termo = this.busca.trim().toLowerCase()
    const i = texto.toLowerCase().indexOf(termo)
    if (!termo || i === -1) {
      return { antes: texto, meio: '', depois: '' }
    }
    return {
      antes: texto.slice(0, i),
      meio: texto.slice(i, i + termo.length),
      depois: texto.slice(i + termo.length)
    }
  }
}
```

```html
{{ destacar(tarefa.texto).antes }}<mark>{{ destacar(tarefa.texto).meio }}</mark>{{ destacar(tarefa.texto).depois }}
```

Aqui faz sentido usar **método** e não `computed`, porque recebe um parâmetro (o texto de cada tarefa). Limitação: destaca só a primeira ocorrência do termo.

#### 4.4 E se eu quisesse que a busca só começasse a partir de 2 letras?

**Resposta:** a regra fica dentro do `computed`. Com menos de 2 letras, ele devolve a lista inteira:

```js
tarefasFiltradas() {
  const termo = this.busca.trim().toLowerCase()
  if (termo.length < 2) {
    return this.tarefas
  }
  return this.tarefas.filter(tarefa => tarefa.texto.toLowerCase().includes(termo))
}
```

A condição do aviso também muda, senão com 1 letra ele poderia aparecer sem a busca estar valendo: `v-if="busca.trim().length >= 2 && tarefasFiltradas.length === 0"`. Devolver `this.tarefas` diretamente não duplica nada, é o mesmo array.

### Organização do código

#### 4.5 Como separar pendentes e concluídas em dois `computed` e eliminar o `v-if` de dentro do `v-for`?

**Resposta:** criar dois `computed` que partem de `tarefasFiltradas`, porque um `computed` pode depender de outro:

```js
computed: {
  tarefasFiltradas() { /* igual ao atual */ },
  pendentes() {
    return this.tarefasFiltradas.filter(tarefa => !tarefa.concluida)
  },
  concluidas() {
    return this.tarefasFiltradas.filter(tarefa => tarefa.concluida)
  }
}
```

```html
<li class="list-group-item" v-for="tarefa in pendentes" v-bind:key="tarefa.id" ...>
```

O `<template>` e o `v-if` somem: cada lista percorre só as suas tarefas, em vez de o array inteiro duas vezes. Essa é a melhoria indicada no Raio X, item 6. Ao marcar um checkbox, `concluida` muda, os dois `computed` recalculam e a tarefa troca de lista, como antes.

#### 4.6 Como mostrar um aviso "nenhuma pendente" ou "nenhuma concluída" em cada seção?

**Resposta:** com os `computed` da 4.5, é um `v-if` por seção:

```html
<p class="text-muted" v-if="tarefasFiltradas.length > 0 && pendentes.length === 0">Nenhuma pendente.</p>
<p class="text-muted" v-if="tarefasFiltradas.length > 0 && concluidas.length === 0">Nenhuma concluída.</p>
```

O `tarefasFiltradas.length > 0` evita repetir o aviso: se a busca não encontrou nada, já aparece a mensagem geral, e mostrar três avisos seria poluição visual.

#### 4.7 Como tirar o ternário do modo escuro que se repete em 3 lugares?

**Resposta:** um `computed` para cada classe. A regra do tema fica num lugar só:

```js
computed: {
  classeApp() {
    return this.modoEscuro ? 'bg-dark' : 'bg-light'
  },
  classeCard() {
    return this.modoEscuro ? 'bg-dark text-white' : 'bg-white text-dark'
  },
  classeItem() {
    return this.modoEscuro ? 'bg-dark text-white border-light' : ''
  }
}
```

```html
<li class="list-group-item" v-bind:class="classeItem">
```

A vantagem é que, para trocar as cores do tema escuro, mexe-se num lugar só. Hoje o ternário dos itens está copiado nas duas listas, e esquecer uma das cópias deixaria o tema inconsistente.

#### 4.8 Como mostrar um contador de tarefas no título, como "Pendentes (3)"?

**Resposta:** com os `computed` da 4.5, é só interpolação:

```html
<h6>Pendentes ({{ pendentes.length }})</h6>
<h6>Concluídas ({{ concluidas.length }})</h6>
```

Há uma decisão a tomar: contar só as que aparecem na busca (`pendentes`, como acima) ou o total real (`tarefas.filter(t => !t.concluida).length`). Contar as filtradas é mais coerente com o que o usuário está vendo.

### Corrigir os problemas do original

#### 4.9 Como adicionar tarefa apertando Enter?

**Resposta:** o `v-on` aceita **modificadores de tecla**. Basta um atributo no campo:

```html
<input type="text" class="form-control" v-model="novaTarefa"
       v-on:keyup.enter="adicionarTarefa()" placeholder="Nova tarefa">
```

O `.enter` faz o Vue chamar o método só quando a tecla solta é o Enter. O método é o mesmo do botão, então a validação com `trim()` vale para os dois caminhos.

#### 4.10 Como colocar o botão excluir também nas concluídas?

**Resposta:** copiar o botão para o `<li>` da lista de concluídas:

```html
<button class="btn btn-sm btn-danger ms-2" v-on:click="excluirTarefa(tarefa.id)">excluir</button>
```

No código original, isso exigiria cuidado com o índice. Com o `id`, funciona sem nenhuma mudança no método, porque `findIndex` acha a tarefa em qualquer posição do array. Isso mostra outra vantagem do `id`: o método não depende de onde o botão está.

#### 4.11 Como salvar o texto já com `trim()` e impedir tarefas duplicadas?

**Resposta:**

```js
adicionarTarefa() {
  const texto = this.novaTarefa.trim()
  if (!texto) {
    return
  }
  const existe = this.tarefas.some(tarefa => tarefa.texto.toLowerCase() === texto.toLowerCase())
  if (existe) {
    return
  }
  this.tarefas.push({ id: this.proximoId, texto: texto, concluida: false })
  this.proximoId++
  this.novaTarefa = ''
}
```

- Guardar o resultado do `trim()` numa variável corrige o problema do original, que validava sem espaços mas salvava com espaços.
- `some` devolve `true` se **alguma** tarefa satisfaz a condição. A comparação em minúsculas faz "Estudar" e "estudar" contarem como duplicadas.
- Na duplicada, o campo não é limpo, e o usuário pode corrigir o texto. Para ficar claro, daria para colocar um dado `erro` em `data()` e mostrá-lo com `v-if`.

#### 4.12 Como fazer o modo escuro pintar a página inteira, e não só o card?

**Resposta:** hoje o `#app` tem a classe `container`, que tem largura máxima e fica centralizado. O resto do `<body>` fica fora do Vue e continua branco. A solução é o `#app` ocupar a tela toda e o `container` ficar dentro dele:

```html
<div id="app" class="min-vh-100 py-4" v-bind:class="modoEscuro ? 'bg-dark' : 'bg-light'">
  <div class="container">
    <div class="card" ...>
      ...
    </div>
  </div>
</div>
```

- `min-vh-100` (Bootstrap) faz o `#app` ter no mínimo a altura da janela.
- Sem `container` no `#app`, ele ocupa a largura inteira.
- Ainda sobra a margem padrão do `<body>` (8px). O Bootstrap zera essa margem, então na prática fica tudo escuro.

#### 4.13 Como colocar `<label>` nos checkboxes para poder clicar no texto da tarefa?

**Resposta:** colocar o checkbox **dentro** do `<label>`. Assim clicar no texto também marca o checkbox:

```html
<label>
  <input type="checkbox" v-model="tarefa.concluida">
  {{ tarefa.texto }}
</label>
```

Além de aumentar a área de clique, o leitor de tela passa a anunciar "Estudar, caixa de seleção" em vez de só "caixa de seleção" (acessibilidade). O switch do modo escuro tem o mesmo problema: o `<label>` dele não está ligado ao input. A correção é colocar `id="modoEscuro"` no input e `for="modoEscuro"` no label.

### Para testar se entendi (mais difíceis)

#### 4.14 O que acontece se eu voltar o `excluirTarefa` para usar o índice? Mostre o bug acontecendo.

**Resposta:** trocando no `index.html`:

```html
<template v-for="(tarefa, indice) in tarefasFiltradas" v-bind:key="tarefa.id">
  ...
  <button ... v-on:click="excluirTarefa(indice)">excluir</button>
```

```js
excluirTarefa(indice) {
  this.tarefas.splice(indice, 1)
}
```

Passo a passo:

| Passo | `tarefas` | `tarefasFiltradas` | O que aparece |
|---|---|---|---|
| 1. Adicionar "Estudar Vue" | [0] Estudar, [1] Ler capítulo, [2] Estudar Vue | as 3 | as 3 |
| 2. Buscar "vue" | igual | [0] Estudar Vue | só "Estudar Vue" |
| 3. Clicar em excluir | `splice(0, 1)` remove **Estudar** → [0] Ler capítulo, [1] Estudar Vue | [0] Estudar Vue | "Estudar Vue" **continua na tela** |
| 4. Limpar a busca | igual ao passo 3 | as 2 | "Estudar" sumiu |

Na tela, parece que o botão não funcionou (a tarefa clicada continua lá). Na verdade o usuário perdeu outra tarefa, sem perceber. O `v-for` sobre a lista filtrada fornece o índice **da lista filtrada** (0), mas o `splice` usa esse número no array **original**, onde a posição 0 é outra tarefa. Com o `id`, o `findIndex` procura a tarefa pelo que ela é, não por onde ela aparece.

#### 4.15 Como transformar o filtro em um método e comparar com o `computed`?

**Resposta:** deixar os dois lado a lado, com um `console.log` em cada:

```js
computed: {
  tarefasFiltradas() {
    console.log('computed rodou')
    const termo = this.busca.trim().toLowerCase()
    return this.tarefas.filter(tarefa => tarefa.texto.toLowerCase().includes(termo))
  }
},
methods: {
  filtrarTarefas() {
    console.log('método rodou')
    const termo = this.busca.trim().toLowerCase()
    return this.tarefas.filter(tarefa => tarefa.texto.toLowerCase().includes(termo))
  }
}
```

Depois troca-se o template para usar `filtrarTarefas()` nos 3 lugares (as duas listas e o aviso) e compara-se o console (F12) digitando em cada campo.

| Ação | Com `computed` | Com método |
|---|---|---|
| Digitar uma letra na busca | roda 1 vez | roda 3 vezes (uma por uso no template) |
| Digitar uma letra em "Nova tarefa" | não roda | roda 3 vezes |
| Ligar o modo escuro | não roda | roda 3 vezes |

O método roda a cada nova renderização do componente, mesmo quando a busca não mudou. O `computed` só roda quando `busca` ou `tarefas` mudam, e o resultado é reaproveitado nos 3 lugares. Com 2 tarefas a diferença é invisível; com milhares, o método deixaria a digitação lenta.

#### 4.16 Como editar o texto de uma tarefa existente?

**Resposta:** dois dados novos e três métodos:

```js
data() {
  return {
    ...
    editandoId: null,     // id da tarefa em edição (null = nenhuma)
    textoEditado: ''
  }
},
methods: {
  iniciarEdicao(tarefa) {
    this.editandoId = tarefa.id
    this.textoEditado = tarefa.texto
  },
  salvarEdicao(tarefa) {
    if (this.textoEditado.trim()) {
      tarefa.texto = this.textoEditado.trim()
    }
    this.editandoId = null
  },
  cancelarEdicao() {
    this.editandoId = null
  }
}
```

```html
<template v-if="editandoId === tarefa.id">
  <input v-model="textoEditado" v-on:keyup.enter="salvarEdicao(tarefa)" v-on:keyup.esc="cancelarEdicao()">
</template>
<template v-else>
  {{ tarefa.texto }}
  <button class="btn btn-sm btn-secondary ms-2" v-on:click="iniciarEdicao(tarefa)">editar</button>
</template>
```

- A edição é feita num dado separado (`textoEditado`) para poder cancelar sem estragar o texto original.
- `salvarEdicao` altera o objeto recebido, que é o mesmo de `tarefas` (referência, como na pergunta 3.4).
- Detalhe interessante: se a busca estiver ativa e o novo texto não contiver mais o termo, a tarefa **some da lista** ao salvar. O `computed` depende do `texto` de cada tarefa e recalcula.

#### 4.17 Como filtrar também por status, com botões "todas / pendentes / concluídas"?

**Resposta:** um dado novo e uma condição a mais no mesmo `computed`:

```js
data() {
  return {
    ...
    status: 'todas'   // 'todas', 'pendentes' ou 'concluidas'
  }
},
computed: {
  tarefasFiltradas() {
    const termo = this.busca.trim().toLowerCase()
    return this.tarefas.filter(tarefa => {
      if (!tarefa.texto.toLowerCase().includes(termo)) {
        return false
      }
      if (this.status === 'pendentes') {
        return !tarefa.concluida
      }
      if (this.status === 'concluidas') {
        return tarefa.concluida
      }
      return true
    })
  }
}
```

```html
<div class="btn-group mb-3">
  <button class="btn btn-outline-primary" v-bind:class="{ active: status === 'todas' }" v-on:click="status = 'todas'">todas</button>
  <button class="btn btn-outline-primary" v-bind:class="{ active: status === 'pendentes' }" v-on:click="status = 'pendentes'">pendentes</button>
  <button class="btn btn-outline-primary" v-bind:class="{ active: status === 'concluidas' }" v-on:click="status = 'concluidas'">concluídas</button>
</div>
```

- Os dois filtros se combinam: a tarefa precisa passar nos dois.
- `v-bind:class="{ active: ... }"` é a forma de **objeto** do `v-bind:class`: a classe `active` é aplicada só quando a condição é verdadeira, destacando o botão escolhido.
- Nada muda em `tarefas`. Continua sendo só um jeito diferente de exibir, o mesmo princípio da busca.
- O `computed` agora depende de `busca`, `tarefas` **e** `status`. Mudar qualquer um dos três recalcula a lista.

## O que não aproveitei

- Correção dos bugs do original: ficaram apenas registrados no Raio X, fora do escopo pedido.
- Busca sem acentos: fora dos PDFs, documentada como melhoria futura.
- Prints automáticos: fiz os testes manualmente.
