
<script>
  let studentName = 'Palak'
  import { supabase } from './lib/supabase.js'

let user = null
let sessionLoading = true

supabase.auth.getSession().then(({ data, error }) => {
  if (error) {
    authMessage = error.message
  } else {
    user = data.session?.user ?? null
  }

  sessionLoading = false
})

supabase.auth.onAuthStateChange((_event, session) => {
  user = session?.user ?? null
})
let email = ''
let password = ''
let authMode = 'signin'
let authMessage = ''
let authLoading = false
async function handleAuth(event) {
  event.preventDefault()
  authLoading = true
  authMessage = ''

  try {
    const { data, error } = authMode === 'signup'
      ? await supabase.auth.signUp({ email, password })
      : await supabase.auth.signInWithPassword({ email, password })

    if (error) throw error

    user = data.user
    await loadTasks()
    authMessage = authMode === 'signup'
      ? 'Account created! Check your email if confirmation is required.'
      : 'Signed in successfully!'
  } catch (error) {
    authMessage = error.message
  } finally {
    authLoading = false
  }
}
  let newTask = ''
  let newPriority = 'medium'
  let newDueDate = ''
  let tasks = JSON.parse(
  localStorage.getItem('focusflow-tasks') || '[]'
)
  let activeFilter = 'all'
  let searchQuery = ''

 async function loadTasks() {
  if (!user) return

  const { data, error } = await supabase
    .from('tasks')
    .select('*')
    .eq('user_id', user.id)
    .order('created_at', { ascending: false })

  if (error) {
    authMessage = error.message
    return
  }

  tasks = data.map(task => ({
    id: task.id,
    title: task.title,
    completed: task.completed,
    priority: task.priority,
    dueDate: task.due_date ?? ''
  }))
}
 
async function addTask() {
  const title = newTask.trim()
  if (!title || !user) return

  const { data, error } = await supabase
    .from('tasks')
    .insert({
      user_id: user.id,
      title,
      priority: newPriority,
      due_date: newDueDate || null
    })
    .select()
    .single()

  if (error) {
    authMessage = error.message
    return
  }

  tasks = [
    {
      id: data.id,
      title: data.title,
      completed: data.completed,
      priority: data.priority,
      dueDate: data.due_date ?? ''
    },
    ...tasks
  ]

  newTask = ''
  newDueDate = ''
  authMessage = ''
}


  
async function toggleTask(id) {
  const task = tasks.find(task => task.id === id)
  if (!task || !user) return

  const { error } = await supabase
    .from('tasks')
    .update({ completed: !task.completed })
    .eq('id', id)
    .eq('user_id', user.id)

  if (error) {
    authMessage = error.message
    return
  }

  tasks = tasks.map(task =>
    task.id === id
      ? { ...task, completed: !task.completed }
      : task
  )
}


  
async function deleteTask(id) {
  if (!user) return

  const { error } = await supabase
    .from('tasks')
    .delete()
    .eq('id', id)
    .eq('user_id', user.id)

  if (error) {
    authMessage = error.message
    return
  }

  tasks = tasks.filter(task => task.id !== id)
}


  $: completedCount = tasks.filter(task => task.completed).length
  $: pendingCount = tasks.length - completedCount
  $: progress = tasks.length
    ? Math.round((completedCount / tasks.length) * 100)
    : 0

    $: completionRate = tasks.length
  ? Math.round((completedCount / tasks.length) * 100)
  : 0
  $: filteredTasks = tasks.filter(task => {
  const matchesFilter =
    activeFilter === 'pending' ? !task.completed :
    activeFilter === 'completed' ? task.completed :
    true

  const matchesSearch = task.title
    .toLowerCase()
    .includes(searchQuery.trim().toLowerCase())

  return matchesFilter && matchesSearch
})
</script>

<svelte:head>
  <title>FocusFlow — Your Student Workspace</title>
  <meta
    name="description"
    content="A calmer way to manage your student life."
  />
</svelte:head>

<main>
{#if !user}
  <section class="tasks">
    <h2>Welcome to FocusFlow</h2>
    <p>Sign in to sync your tasks across devices.</p>

    <form onsubmit={handleAuth}>
      <input
        type="email"
        bind:value={email}
        placeholder="Your email"
        required
      />

      <input
        type="password"
        bind:value={password}
        placeholder="Password (at least 6 characters)"
        minlength="6"
        required
      />

      <button class="add-button" type="submit" disabled={authLoading}>
        {authLoading
          ? 'Please wait...'
          : authMode === 'signup'
            ? 'Create account'
            : 'Sign in'}
      </button>
    </form>

    <button
      type="button"
      onclick={() => authMode = authMode === 'signin' ? 'signup' : 'signin'}
    >
      {authMode === 'signin'
        ? 'New here? Create an account'
        : 'Already registered? Sign in'}
    </button>

    {#if authMessage}
      <p>{authMessage}</p>
    {/if}
  </section>
{:else}
  <header class="hero">
    <p class="eyebrow">YOUR PERSONAL WORKSPACE</p>
    <h1>FocusFlow<span>.</span></h1>
    <p class="subtitle">A calmer way to manage your student life.</p>
  </header>

  <section class="welcome">
    <div>
      <p class="eyebrow">YOUR DAILY SPACE</p>
      <h2>Welcome back, {studentName}!</h2>
      <p>Small steps today. Big progress tomorrow.</p>
    </div>
    <div class="welcome-icon">✦</div>
  </section>

  <section class="dashboard">
    <article class="stat-card">
      <span class="stat-icon purple">☷</span>
      <p>Total tasks</p>
      <h3>{tasks.length}</h3>
    </article>

    <article class="stat-card">
      <span class="stat-icon orange">◷</span>
      <p>Pending</p>
      <h3>{pendingCount}</h3>
    </article>

    <article class="stat-card">
  <span class="stat-icon green">✓</span>
  <p>Completed</p>
  <h3>{completedCount}</h3>
  <p>{completionRate}% completion rate</p>
</article>
  </section>

  <section class="tasks">
    <div class="section-heading">
      <div>
        <p class="eyebrow">MAKE IT HAPPEN</p>
        <h2>Your tasks</h2>
      </div>
      <span class="task-count">{pendingCount} left</span>
    </div>

<form onsubmit={(event) => {
  event.preventDefault()
  addTask()
}}>
  <input
    bind:value={newTask}
    placeholder="What needs your attention?"
    aria-label="New task"
  />

  <select bind:value={newPriority} aria-label="Task priority">
    <option value="high">🔴 High</option>
    <option value="medium">🟡 Medium</option>
    <option value="low">🟢 Low</option>
  </select>
  <input
  type="date"
  bind:value={newDueDate}
  aria-label="Task due date"
/>

  <button class="add-button" type="submit">+ Add task</button>
</form>

   

    <div class="progress-section">
      <div class="progress-label">
        <span>Daily progress</span>
        <strong>{progress}%</strong>
      </div>
      <div
        class="progress-track"
        role="progressbar"
        aria-label="Task completion progress"
        aria-valuemin="0"
        aria-valuemax="100"
        aria-valuenow={progress}
      >
        <div class="progress-fill" style={`width: ${progress}%`}></div>
      </div>
    </div>

    <input
  class="search-input"
  type="search"
  bind:value={searchQuery}
  placeholder="🔎 Search your tasks..."
  aria-label="Search tasks"
/>
    <div class="filters" aria-label="Filter tasks">
      {#each [
        { id: 'all', label: 'All tasks' },
        { id: 'pending', label: 'Pending' },
        { id: 'completed', label: 'Completed' }
      ] as filter}
        <button
          type="button"
          class:active={activeFilter === filter.id}
          aria-pressed={activeFilter === filter.id}
          onclick={() => activeFilter = filter.id}
        >
          {filter.label}
        </button>
      {/each}
    </div>

    {#if filteredTasks.length === 0}
      <div class="empty-state">
        <div class="empty-icon">✧</div>
        <h3>
          {tasks.length === 0
            ? 'A fresh start.'
            : 'Nothing here yet.'}
        </h3>
        <p>
          {tasks.length === 0
            ? 'Add your first task and make today count.'
            : 'Try another filter or add a new task.'}
        </p>
      </div>
    {:else}
      <div class="task-list">
        {#each filteredTasks as task (task.id)}
          <div class="task-row">
            <label class="task-label">
              <input
                type="checkbox"
                checked={task.completed}
                onchange={() => toggleTask(task.id)}
              />
             <span class:completed={task.completed}>
  {task.title}
</span>

<span class={`priority-badge ${task.priority ?? 'medium'}`}>
  {task.priority ?? 'medium'}
</span>
{#if task.dueDate}
  <span class="due-date">
    Due: {task.dueDate}
  </span>
{/if}
            </label>

            <button
              class="delete-button"
              type="button"
              aria-label={`Delete ${task.title}`}
              onclick={() => deleteTask(task.id)}
            >
              Delete
            </button>
          </div>
        {/each}
      </div>
    {/if}
  </section>

  <footer>
    <span>FocusFlow</span>
    <span>One step at a time. ✨</span>
  </footer>
  {/if}
</main>

<style>
  :global(*) {
    box-sizing: border-box;
  }

  :global(body) {
    margin: 0;
    font-family: Inter, Arial, sans-serif;
    background: #f5f6fc;
    color: #22243b;
  }

  :global(button), :global(input) {
    font: inherit;
  }

  main {
    max-width: 850px;
    margin: 0 auto;
    padding: 48px 24px 24px;
  }

  .hero {
    text-align: center;
    margin-bottom: 34px;
  }

  .eyebrow {
    color: #7161db;
    font-size: 11px;
    font-weight: 800;
    letter-spacing: 2px;
    margin: 0 0 10px;
  }

  h1 {
    font-size: clamp(42px, 7vw, 64px);
    letter-spacing: -3px;
    margin: 0;
    color: #22243b;
  }

  h1 span {
    color: #7966e8;
  }

  .subtitle {
    color: #777c91;
    margin-top: 8px;
    font-size: 16px;
  }

  .welcome {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 16px;
    background: linear-gradient(120deg, #fff, #f0edff);
    padding: 28px;
    border: 1px solid #e9e7f7;
    border-radius: 22px;
    margin-bottom: 18px;
    box-shadow: 0 8px 30px #28294a08;
  }

  .welcome h2,
.tasks h2 {
  margin: 0 0 9px;
  font-size: 24px;
  color: #22243b;
}

  .welcome p:last-child {
    margin: 0;
    color: #777c91;
  }

  .welcome-icon {
    color: #7966e8;
    background: white;
    border-radius: 18px;
    width: 56px;
    height: 56px;
    display: grid;
    place-items: center;
    font-size: 30px;
    flex-shrink: 0;
  }

  .dashboard {
    display: grid;
    grid-template-columns: repeat(3, minmax(0, 1fr));
    gap: 14px;
    margin-bottom: 18px;
  }

  .stat-card {
    padding: 20px;
    background: white;
    border: 1px solid #ececf4;
    border-radius: 18px;
  }

  .stat-icon {
    display: grid;
    place-items: center;
    width: 36px;
    height: 36px;
    border-radius: 12px;
    font-size: 20px;
  }

  .purple { background: #f0edff; color: #705cda; }
  .orange { background: #fff1e5; color: #d9823c; }
  .green { background: #e4f8ef; color: #25875a; }

  .stat-card p {
    margin: 14px 0 5px;
    font-size: 13px;
    color: #777c91;
  }

  .stat-card h3 {
    font-size: 28px;
    margin: 0;
  }

  .tasks {
    padding: 28px;
    background: white;
    border: 1px solid #ececf4;
    border-radius: 22px;
    box-shadow: 0 8px 30px #28294a08;
  }

  .section-heading, .progress-label {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
  }

  .task-count {
    background: #f0edff;
    color: #705cda;
    padding: 8px 12px;
    border-radius: 20px;
    font-size: 12px;
    font-weight: 700;
  }

  form {
    display: flex;
    gap: 10px;
    margin: 22px 0 24px;
  }

  form input {
    min-width: 0;
    flex: 1;
    padding: 14px 16px;
    border: 1px solid #e2e3ed;
    border-radius: 12px;
    outline: none;
    background: #fbfbfe;
    color: #22243b;
  }

  form input:focus {
    border-color: #7966e8;
    box-shadow: 0 0 0 3px #7966e81c;
  }

  button {
    cursor: pointer;
    transition: background 0.2s, transform 0.2s;
  }

  button:focus-visible, input:focus-visible {
    outline: 3px solid #a69bf8;
    outline-offset: 3px;
  }

  .add-button {
    background: #7161db;
    color: white;
    border: 0;
    border-radius: 12px;
    padding: 0 18px;
    font-weight: 700;
    white-space: nowrap;
  }

  .add-button:hover {
    background: #5d4bc8;
  }

  .progress-section {
    margin-bottom: 25px;
  }

  .progress-label {
    font-size: 13px;
    color: #777c91;
    margin-bottom: 10px;
  }

  .progress-label strong {
    color: #7161db;
  }

  .progress-track {
    height: 8px;
    background: #eeedf7;
    border-radius: 20px;
    overflow: hidden;
  }

  .progress-fill {
    height: 100%;
    background: linear-gradient(90deg, #8e7cf1, #6555d1);
    border-radius: 20px;
    transition: width 0.25s ease;
  }

  .filters {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    padding-bottom: 16px;
    border-bottom: 1px solid #eeeef5;
  }

  .filters button {
    border: 0;
    background: #f5f5fa;
    color: #74778b;
    padding: 9px 14px;
    border-radius: 10px;
    font-size: 13px;
  }

  .filters button.active {
    color: white;
    background: #7161db;
  }

  .task-list {
    display: grid;
    gap: 10px;
    margin-top: 16px;
  }

  .task-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 15px;
    border: 1px solid #eeeef5;
    border-radius: 12px;
    background: #fff;
  }

  .task-label {
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
    overflow-wrap: anywhere;
  }

  .task-label input {
    width: 17px;
    height: 17px;
    accent-color: #7161db;
    flex-shrink: 0;
  }

  .completed {
    color: #999bad;
    text-decoration: line-through;
  }

  .delete-button {
    flex-shrink: 0;
    border: 0;
    background: #fff0f0;
    color: #c34e58;
    padding: 8px 10px;
    border-radius: 8px;
    font-size: 12px;
  }

  .delete-button:hover {
    background: #ffe0e0;
  }

  .empty-state {
    text-align: center;
    padding: 35px 10px 18px;
  }

  .empty-icon {
    font-size: 34px;
    color: #8c7ae8;
  }

  .empty-state h3 {
    margin: 12px 0 8px;
  }

  .empty-state p {
    color: #777c91;
    font-size: 14px;
    margin: 0;
  }

  footer {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    padding: 24px 4px 0;
    color: #898ba0;
    font-size: 12px;
  }

  footer span:first-child {
    color: #7161db;
    font-weight: 800;
  }

  @media (max-width: 520px) {
    main {
      padding: 32px 14px 20px;
    }

    .welcome, .tasks {
      padding: 20px;
    }

    .dashboard {
      gap: 8px;
    }

    .stat-card {
      padding: 13px;
    }

    .stat-card h3 {
      font-size: 24px;
    }

    form {
      flex-direction: column;
    }

    .add-button {
      padding: 13px;
    }

    .welcome h2, .tasks h2 {
      font-size: 21px;
    }
  }
  .priority-badge {
  padding: 5px 9px;
  border-radius: 20px;
  font-size: 11px;
  font-weight: 700;
  text-transform: capitalize;
  white-space: nowrap;
}

.priority-badge.high {
  background: #fff0f0;
  color: #c34e58;
}

.priority-badge.medium {
  background: #fff5df;
  color: #986a10;
}

.priority-badge.low {
  background: #e4f8ef;
  color: #25875a;
}
.due-date {
  font-size: 11px;
  color: #7161db;
  background: #f0edff;
  padding: 5px 9px;
  border-radius: 20px;
  white-space: nowrap;
}
.search-input {
  width: 100%;
  padding: 13px 15px;
  margin: 16px 0;
  border: 1px solid #e2e3ed;
  border-radius: 12px;
  background: #fbfbfe;
  color: #22243b;
  outline: none;
}

.search-input:focus {
  border-color: #7966e8;
  box-shadow: 0 0 0 3px #7966e81c;
}
</style>