# Web2 · Tarefa 001b — Raio X e evolução do produto

Dupla: Rafael Lourenço

Arquivos:

| Arquivo | Conteúdo |
|---|---|
| `original.html` | Aplicação de partida, sem alterações (base do Raio X) |
| `index.html` | Aplicação final, com a busca |
| `README.md` | Este documento |
| `ia.md` | Registro das perguntas feitas à IA e do que foi aproveitado |
| `evidencias/` | Prints dos testes |

---

## Parte 1 · Raio X (código original)

### 1. Dados em `data()`

| Dado | Tipo | Para que serve |
|---|---|---|
| `modoEscuro` | boolean | Guarda se o tema escuro está ligado. Começa `false`. |
| `novaTarefa` | string | Guarda o que o usuário está digitando no campo "Nova tarefa". |
| `tarefas` | array de objetos `{ texto, concluida }` | A lista de tarefas. É a única fonte das duas listas da tela. |

### 2. Elementos que usam interpolação

Só `{{ tarefa.texto }}`, dentro de cada `<li>` das listas **Pendentes** e **Concluídas**. Ela mostra o texto de cada objeto do array. Se o texto mudar no objeto, a tela muda junto.

### 3. Onde aparecem as diretivas e o que acontece na tela

**`v-model`**
- `v-model="modoEscuro"` no switch. Ao clicar no switch, `modoEscuro` vira `true`/`false` e todos os `v-bind:class` que dependem dele trocam as cores na hora.
- `v-model="novaTarefa"` no campo de texto. Cada tecla digitada atualiza `novaTarefa`. Quando o método limpa `novaTarefa`, o campo também fica vazio (a ligação funciona nos dois sentidos).
- `v-model="tarefa.concluida"` nos checkboxes de cada tarefa. Ao marcar, o objeto passa a ter `concluida: true`. Com isso o `v-if` da lista Pendentes deixa de mostrar o item e o da lista Concluídas passa a mostrar: **a tarefa "pula" de lista**. Ao desmarcar, ela volta.

**`v-for`**
- Aparece duas vezes, uma em cada `<ul>`. As duas percorrem o **array inteiro** `tarefas` e criam um `<li>` para cada tarefa. Quando o array ganha ou perde um item, as listas são redesenhadas.

**`v-if`**
- `v-if="!tarefa.concluida"` na primeira lista e `v-if="tarefa.concluida"` na segunda. Cada tarefa é percorrida nas duas listas, mas só aparece em uma delas. É isso que separa pendentes de concluídas.

**`v-bind`**
- `v-bind:class` no `#app`, no card e em cada `<li>`: escolhe as classes Bootstrap claras (`bg-light`, `bg-white text-dark`) ou escuras (`bg-dark text-white border-light`) conforme `modoEscuro`.
- `v-bind:key="indice"`: identifica cada item para o Vue saber qual elemento atualizar.

**`v-on`**
- `v-on:click="adicionarTarefa()"` no botão **adicionar**.
- `v-on:click="excluirTarefa(indice)"` no botão **excluir** (só existe na lista de pendentes).

### 4. O que cada método faz

- **`adicionarTarefa()`**: se `novaTarefa` não for vazio (nem só espaços, por causa do `trim()`), coloca um novo objeto `{ texto, concluida: false }` no fim de `tarefas` e limpa o campo. Na tela, a tarefa aparece em Pendentes e o input fica vazio.
- **`excluirTarefa(indice)`**: remove do array o item da posição `indice` com `splice`. Na tela, o `<li>` some.

### 5. Qual alteração de estado faz a tela atualizar

A tela é consequência do estado. Qualquer mudança em `modoEscuro`, `novaTarefa` ou `tarefas` (incluindo `push`, `splice` e mudar `concluida` de um objeto) é detectada pela reatividade do Vue, que redesenha só as partes do template que usam aquele dado. Os métodos nunca mexem no HTML diretamente, só nos dados.

### 6. Trecho que seria difícil de alterar se o produto crescesse

- **As duas listas são blocos duplicados**: o mesmo `v-for` sobre o array inteiro, diferenciado só pelo `v-if`. Qualquer mudança (um filtro, por exemplo) precisa ser feita duas vezes, e cada tarefa é percorrida duas vezes.
- **A identificação por índice** (`key` e `excluirTarefa(indice)`). O índice só vale enquanto a lista exibida é igual ao array. Se a lista exibida for filtrada ou ordenada, o índice da tela deixa de ser o índice do array e a exclusão **apaga a tarefa errada**. Foi exatamente isso que tivemos de resolver na Parte 2.
- O ternário de classes do modo escuro se repete em 3 lugares.

### Problemas observados no original (não corrigidos, fora do escopo)

- Tarefas concluídas não têm botão excluir.
- O texto é validado com `trim()`, mas é salvo **sem** `trim()` (espaços nas pontas ficam no texto).
- Enter não adiciona a tarefa, só o clique.
- O modo escuro pinta só o `#app`: o fundo da página fora do container continua claro.
- Nenhum dado é salvo: recarregar a página volta às 2 tarefas iniciais.

---

## Parte 2 · Adição: busca por texto

### O que mudou

```js
data() {
  return {
    ...
    busca: '',                // texto digitado na busca
    proximoId: 3,             // gerador de id
    tarefas: [
      { id: 1, texto: 'Estudar', concluida: false },
      { id: 2, texto: 'Ler capítulo', concluida: true }
    ]
  }
},
computed: {
  tarefasFiltradas() {
    const termo = this.busca.trim().toLowerCase()
    return this.tarefas.filter(tarefa => tarefa.texto.toLowerCase().includes(termo))
  }
}
```

No template:
- novo `<input type="search" v-model="busca">`;
- os dois `v-for` passaram a percorrer `tarefasFiltradas` em vez de `tarefas`;
- `v-bind:key="tarefa.id"` no lugar do índice;
- o botão excluir chama `excluirTarefa(tarefa.id)`;
- um aviso `v-if="busca.trim() && tarefasFiltradas.length === 0"` com a mensagem _Nenhuma tarefa encontrada para "…"_.

### Fluxo dos dados

1. O usuário digita na busca → `v-model` atualiza `busca`.
2. `tarefasFiltradas` depende de `busca` e de `tarefas`, então o Vue recalcula o valor.
3. Os `v-for` usam `tarefasFiltradas` → as listas são redesenhadas só com as tarefas que contêm o termo.
4. Se o resultado for vazio (e a busca não estiver vazia), o `v-if` do aviso fica verdadeiro e a mensagem aparece.

### Decisões técnicas

- **Por que `computed` e não um método ou uma segunda lista?** O filtro é um valor *derivado* de `tarefas` + `busca`. Com `computed`, o Vue recalcula sozinho quando qualquer um dos dois muda e guarda o resultado em cache enquanto eles não mudam. Uma segunda lista em `data()` duplicaria os dados, e teríamos que lembrar de atualizá-la em todo `push`, `splice` e checkbox. O enunciado pede exatamente isso: não duplicar a lista original e não alterar os dados só para exibir.
- **`filter` não altera `tarefas`**: ele devolve um array novo, mas com **os mesmos objetos**. Por isso marcar o checkbox de uma tarefa filtrada (`v-model="tarefa.concluida"`) altera o objeto original, e a tarefa muda de lista normalmente.
- **Campo vazio mostra tudo**: `'qualquer texto'.includes('')` é sempre `true`, então sem termo nenhuma tarefa é filtrada. Não foi preciso criar um `if` especial.
- **`toLowerCase()`**: "estudar" encontra "Estudar". O `trim()` faz com que espaços digitados sem querer não atrapalhem.
- **Por que criamos `id`?** Com filtro, o índice que o `v-for` fornece é a posição na **lista filtrada**, não em `tarefas`. Exemplo: com `[Estudar, Ler capítulo, Estudar Vue]` e busca "vue", "Estudar Vue" aparece com índice 0. O `excluirTarefa(0)` original apagaria "Estudar". Com um `id` único por tarefa, a exclusão procura a posição real com `findIndex` e remove a tarefa certa. O `id` também passou a ser a `key`, que deve identificar o item, não a posição.
- **Adicionar com filtro ativo**: a nova tarefa entra em `tarefas` normalmente, mas só aparece se o texto contiver o termo da busca. Isso é o comportamento esperado do filtro (a tela mostra a busca atual). Ao limpar a busca, a tarefa aparece. Optamos por não limpar a busca automaticamente, para o filtro continuar previsível.

---

## Parte 3 · Testes

Dados iniciais: "Estudar" (pendente) e "Ler capítulo" (concluída).

| # | Cenário | Passos | Resultado esperado | Obtido | Evidência |
|---|---|---|---|---|---|
| 1 | Campo vazio | Abrir a página sem digitar na busca | As 2 tarefas aparecem, cada uma na sua lista; sem aviso | _ | `evidencias/01-vazio.png` |
| 2 | Palavra existente | Digitar `estudar` | Só "Estudar" em Pendentes; Concluídas vazia; sem aviso | _ | `evidencias/02-existente.png` |
| 3 | Sem resultado | Digitar `xyz` | Listas vazias; aviso _Nenhuma tarefa encontrada para "xyz"_ | _ | `evidencias/03-sem-resultado.png` |
| 4a | Incluir com filtro (casa) | Busca `estudar`, adicionar "Estudar Vue" | "Estudar Vue" aparece em Pendentes junto com "Estudar" | _ | `evidencias/04a-incluir-casa.png` |
| 4b | Incluir com filtro (não casa) | Busca `estudar`, adicionar "Comprar pão"; depois limpar a busca | Não aparece enquanto filtra; aparece ao limpar a busca | _ | `evidencias/04b-incluir-nao-casa.png` |
| 5a | Concluir tarefa filtrada | Busca `vue`, marcar "Estudar Vue" | Vai para Concluídas; ao limpar a busca continua concluída | _ | `evidencias/05a-concluir.png` |
| 5b | Excluir tarefa filtrada | Tarefas: Estudar, Estudar Vue (pendentes). Busca `vue`, excluir "Estudar Vue" | Some só "Estudar Vue"; ao limpar a busca, "Estudar" continua lá | _ | `evidencias/05b-excluir.png` |
| 6a | Modo escuro sem filtro | Ligar modo escuro, busca vazia | Card e itens escuros; todas as tarefas visíveis | _ | `evidencias/06a-escuro.png` |
| 6b | Modo escuro com filtro | Modo escuro + busca `xyz` e depois `estudar` | Aviso legível no tema escuro; itens filtrados com classes escuras | _ | `evidencias/06b-escuro-filtro.png` |

> O teste 5b é o mais importante: no código original (exclusão por índice) ele apagaria a tarefa errada.
