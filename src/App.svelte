<script>
  import { onMount } from 'svelte'
  import { sortBy } from 'lodash'
  import PouchDB from 'pouchdb-browser'

  import TodoItem from './todo-item.svelte'

  let db = new PouchDB('db')
  const replication = PouchDB.sync('db', 'http://localhost:8080/', {
    live: true,
    retry: true
  }).on('change', async function (info) {
    await updateTodos()
  }).on('error', function (err) {
    console.log('Replication error:', err)
  })

  let newTodoText = ''
  let sortByWhat = 'createdAt'
  let filterByWhat = ''
  let isLoading = true

  let todos = []
  $: sortedAndFilteredTodos = sortBy(todos, [sortByWhat]).filter((todo) => {
    const [filterKey, filterValue] = filterByWhat.split(':')

    return filterKey && filterValue ? todo[filterKey].toString() === filterValue : true
  })


  async function updateTodos() {
    const allDocs = await db.allDocs({
      include_docs: true
    })
    todos = allDocs.rows.map(row => row.doc)
    isLoading = false
  }


  async function add(event) {
    const newTodo = {
      text: newTodoText,
      complete: false,
      createdAt: new Date().toISOString()
    }
    if (newTodoText != ''){
      const addition = await db.post(newTodo)
      if (addition.ok) {
        await updateTodos()
      }
    }else{
      alert('Veuillez rentré un tache valide')
    }
    

    console.log(newTodoText)

  }

  async function updateStatus(event) {
    const { todo } = event.detail
    const update = await db.put(todo)
    if (update.ok) {
      await updateTodos()
    }
  }

  async function removeItem(event) {
    const { todo: todoToRemove } = event.detail
    const removal = await db.remove(todoToRemove)
    if (removal.ok) {
      todos = todos.filter((todo) => {
        return todo._id !== todoToRemove._id
        })
    }
  }

  onMount(async () => {
    await updateTodos()
  })
</script>

<style>
  ul {
    list-style: none;
    padding: 0;
  }
  button {
    margin-left: 0.75em;
  }
  input[type='text'] {
    width: 440px;
  }
</style>

{#if isLoading}
  <h1>
    Chargement des taches…
  </h1>
{:else}
  {#if todos.length === 0}
    <h1>
      Zéro tache! Nice ✊
    </h1>
  {:else}
    <h1>
      Il reste {sortedAndFilteredTodos.length} sur {todos.length} taches
    </h1>
  {/if}
{/if}

<div>
  <label>Trié par:</label>
  <select bind:value={sortByWhat}>
    <option value='createdAt'>Date de création</option>
    <option value='text'>Alphabétique</option>
    <option value='complete'>Completion</option>
  </select>
</div>
<div>
  <label>Filtrer:</label>
  <select bind:value={filterByWhat}>
    <option value=''>Voir toutes les taches</option>
    <option value='complete:true'>Voir les taches complète</option>
    <option value='complete:false'>Voir les taches non complète</option>
  </select>
</div>

<ul>
  {#each sortedAndFilteredTodos as todo (todo._id)}
    <TodoItem todo={todo} on:remove={removeItem} on:update={updateStatus}/>
  {/each}
</ul>

<form on:submit|preventDefault={add}>
  <input type='text' bind:value={newTodoText}>
  <button type='submit'>➕ Ajouté</button>
</form>