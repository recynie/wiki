---
title: Vue.js 基础教程
date: 2026-04-11
description: Vue.js 渐进式前端框架基础教程，涵盖响应式原理、Composition API、路由和状态管理
tags:
  - vue
  - frontend
  - javascript
draft: false
permalink:
---

## Vue简介

Vue.js 是一个渐进式 JavaScript 框架，用于构建用户界面。与其他大型框架不同，Vue 被设计为可以自底向上逐层应用。[^1]

### 核心特点

- **渐进式框架**：可以从简单的视图层开始，逐步集成更多功能
- **响应式数据绑定**：数据变化自动更新视图
- **组件系统**：基于组件的架构便于复用和维护
- **轻量级**：核心库体积小，性能优秀

### MVVM模式

Vue 实现了 MVVM（Model-View-ViewModel）模式：

| 层次 | 作用 | Vue对应 |
|------|------|---------|
| Model | 数据层 | `data` 选项 |
| View | 视图层 | 模板（Template） |
| ViewModel | 逻辑层 | Vue 实例 |

```javascript
const app = new Vue({
  el: '#app',
  data: {
    message: 'Hello Vue!'
  }
})
```

```html
<div id="app">
  {{ message }}  <!-- View -->
</div>
```

### 响应式原理

Vue 3 使用 Proxy 替代了 Vue 2 的 Object.defineProperty，实现真正的响应式：

```javascript
const data = { count: 0 }
const proxy = new Proxy(data, {
  get(target, key) {
    return Reflect.get(target, key)
  },
  set(target, key, value) {
    console.log('数据更新:', key, value)
    return Reflect.set(target, key, value)
  }
})
```

---

## 核心概念

### 模板语法

#### 插值表达式

```html
<!-- 文本插值 -->
<span>{{ message }}</span>

<!-- JavaScript 表达式 -->
<span>{{ count + 1 }}</span>
<span>{{ ok ? 'Yes' : 'No' }}</span>

<!-- 原始 HTML（慎用） -->
<div v-html="rawHtml"></div>
```

#### 指令（Directives）

指令是带有 `v-` 前缀的特殊属性。

##### v-bind

动态绑定 HTML 属性：

```html
<!-- 完整语法 -->
<img v-bind:src="imageUrl" alt="图片">

<!-- 简写 -->
<img :src="imageUrl" alt="图片">

<!-- 绑定多个属性 -->
<div v-bind="{ id: someId, class: someClass }"></div>
```

##### v-on

监听 DOM 事件：

```html
<!-- 完整语法 -->
<button v-on:click="handleClick">点击</button>

<!-- 简写 -->
<button @click="handleClick">点击</button>

<!-- 传递参数 -->
<button @click="handleClick('arg', $event)">点击</button>

<!-- 事件修饰符 -->
<form @submit.prevent="onSubmit">...</form>
<a @click.stop="handleClick">链接</a>
<input @keyup.enter="submit">
```

##### v-if / v-show

条件渲染：

```html
<!-- v-if：真正的条件渲染 -->
<div v-if="show">条件为 true 时显示</div>
<div v-else-if="show2">第二个条件</div>
<div v-else>都不满足</div>

<!-- v-show：始终渲染，切换 display -->
<div v-show="show">通过 display 切换</div>
```

| 特性 | v-if | v-show |
|------|------|--------|
| 渲染时机 | 条件满足时渲染 | 初始渲染，后切换 display |
| 切换开销 | 高（可能触发组件生命周期） | 低（仅切换 CSS） |
| 初始开销 | 低 | 高 |

##### v-for

列表渲染：

```html
<!-- 遍历数组 -->
<ul>
  <li v-for="(item, index) in items" :key="item.id">
    {{ index }}: {{ item.name }}
  </li>
</ul>

<!-- 遍历对象 -->
<div v-for="(value, key, index) in object" :key="key">
  {{ index }}. {{ key }}: {{ value }}
</div>

<!-- 遍历数字 -->
<span v-for="n in 10" :key="n">{{ n }}</span>
```

:::warning
始终在 `v-for` 中使用 `:key` 属性，以维护组件状态和提高渲染性能。
:::

### 响应式数据

#### data 选项

组件的响应式数据源：

```javascript
export default {
  data() {
    return {
      count: 0,
      name: 'Vue',
      user: {
        age: 18
      }
    }
  }
}
```

#### computed 计算属性

基于响应式数据衍生的计算值：

```javascript
export default {
  data() {
    return {
      firstName: 'John',
      lastName: 'Doe'
    }
  },
  computed: {
    // 只读计算属性
    fullName() {
      return this.firstName + ' ' + this.lastName
    },
    // 可写计算属性
    fullNameWritable: {
      get() {
        return this.firstName + ' ' + this.lastName
      },
      set(value) {
        const [first, last] = value.split(' ')
        this.firstName = first
        this.lastName = last
      }
    }
  }
}
```

| 特性 | methods | computed |
|------|---------|----------|
| 缓存 | 无 | 有（基于响应式依赖） |
| 调用 | 每次调用 | 访问时计算 |
| 适用场景 | 方法调用 | 派生数据 |

#### watch 侦听器

监听响应式数据变化：

```javascript
export default {
  data() {
    return {
      question: '',
      answer: '请输入问题'
    }
  },
  watch: {
    // 基础用法
    question(newValue, oldValue) {
      this.fetchAnswer()
    },
    // 深度监听对象
    user: {
      handler(newUser) {
        console.log('用户信息更新:', newUser)
      },
      deep: true
    },
    // 立即执行
    message: {
      handler(newMsg) {
        console.log('立即执行:', newMsg)
      },
      immediate: true
    }
  }
}
```

### 组件系统

#### 组件定义与使用

```javascript
// Button.vue
<template>
  <button class="btn" @click="handleClick">
    <slot></slot>
  </button>
</template>

<script>
export default {
  name: 'AppButton',
  props: {
    type: {
      type: String,
      default: 'primary',
      validator: v => ['primary', 'secondary'].includes(v)
    }
  },
  emits: ['click'],
  setup(props, { emit }) {
    const handleClick = () => {
      emit('click', 'clicked')
    }
    return { handleClick }
  }
}
</script>
```

#### props 父子通信

父组件向子组件传递数据：

```html
<!-- 父组件 -->
<template>
  <UserCard 
    :name="user.name"
    :age="user.age"
    :is-vip="user.isVip"
  />
</template>

<script>
export default {
  data() {
    return {
      user: { name: 'Alice', age: 20, isVip: true }
    }
  }
}
</script>
```

```javascript
// 子组件
export default {
  // 声明 props
  props: {
    name: String,
    age: {
      type: Number,
      required: true
    },
    isVip: {
      type: Boolean,
      default: false
    }
  }
}
```

#### emit 子父通信

子组件向父组件发送事件：

```javascript
// 子组件
export default {
  emits: ['update', 'delete'],
  setup(props, { emit }) {
    const update = (newValue) => {
      emit('update', newValue)
    }
    const remove = () => {
      emit('delete', this.id)
    }
    return { update, remove }
  }
}
```

```html
<!-- 父组件 -->
<template>
  <ChildComponent 
    @update="handleUpdate"
    @delete="handleDelete"
  />
</template>
```

#### slot 插槽

内容分发机制：

```html
<!-- 父组件 -->
<Card>
  <template #header>
    <h2>卡片标题</h2>
  </template>
  
  <p>默认插槽内容</p>
  
  <template #footer>
    <button>确定</button>
  </template>
</Card>
```

```html
<!-- Card.vue -->
<template>
  <div class="card">
    <slot name="header"></slot>
    <div class="content">
      <slot></slot>
    </div>
    <slot name="footer"></slot>
  </div>
</template>
```

| 插槽类型 | 用途 |
|----------|------|
| 默认插槽 | 无 name 的单一插槽 |
| 具名插槽 | `v-slot:name` 或 `#name` |
| 作用域插槽 | 父组件访问子组件数据 |

---

## Composition API

Composition API 是 Vue 3 引入的新特性，提供更灵活的逻辑组织方式。[^2]

### setup 函数

`setup` 是 Composition API 的入口点：

```javascript
import { ref, computed, onMounted } from 'vue'

export default {
  setup() {
    // 响应式状态
    const count = ref(0)
    const double = computed(() => count.value * 2)
    
    // 方法
    const increment = () => {
      count.value++
    }
    
    // 生命周期钩子
    onMounted(() => {
      console.log('组件挂载完成')
    })
    
    // 返回给模板使用
    return {
      count,
      double,
      increment
    }
  }
}
```

:::tip
在 `setup` 中访问组件实例时，`this` 不可用。使用 `setup(props, context)` 获取。
:::

### ref 和 reactive

两种创建响应式数据的方式：

```javascript
import { ref, reactive, toRefs } from 'vue'

// ref：用于基本类型
const count = ref(0)
const name = ref('Vue')

// 访问值需要 .value
count.value++

// reactive：用于对象/数组
const state = reactive({
  count: 0,
  name: 'Vue',
  user: { age: 18 }
})

// 访问直接使用，无需 .value
state.count++

// 将 reactive 对象转换为 ref
const { count, name } = toRefs(state)
```

| API | 适用类型 | 访问方式 | 示例 |
|-----|----------|----------|------|
| ref | 基本类型 | .value | `const n = ref(0)` |
| reactive | 对象/数组 | 直接访问 | `const s = reactive({n: 0})` |
| toRefs | reactive 转 ref | .value | `const {n} = toRefs(s)` |

### computed 计算属性

```javascript
import { ref, computed } from 'vue'

const firstName = ref('John')
const lastName = ref('Doe')

// 只读
const fullName = computed(() => {
  return firstName.value + ' ' + lastName.value
})

// 可写
const fullNameWritable = computed({
  get() {
    return firstName.value + ' ' + lastName.value
  },
  set(value) {
    const [first, last] = value.split(' ')
    firstName.value = first
    lastName.value = last
  }
})
```

### watch 侦听器

```javascript
import { ref, reactive, watch } from 'vue'

const count = ref(0)
const user = reactive({ name: 'Alice', age: 18 })

// 监听 ref
watch(count, (newVal, oldVal) => {
  console.log(`count: ${oldVal} -> ${newVal}`)
})

// 监听 reactive（自动深度）
watch(user, (newUser, oldUser) => {
  console.log('用户更新:', newUser)
}, { deep: true })

// 监听多个数据源
watch([count, user], ([newCount, newUser], [oldCount, oldUser]) => {
  console.log('任一数据变化')
})

// 立即执行
watch(count, (val) => {
  console.log('立即执行:', val)
}, { immediate: true })
```

### 生命周期钩子

| 选项式 API | Composition API |
|------------|-----------------|
| created | setup() |
| mounted | onMounted |
| updated | onUpdated |
| unmounted | onUnmounted |
| beforeCreate | - |
| beforeMount | onBeforeMount |
| beforeUpdate | onBeforeUpdate |
| beforeUnmount | onBeforeUnmount |
| errorCaptured | onErrorCaptured |

```javascript
import { 
  onMounted, 
  onUpdated, 
  onUnmounted,
  onBeforeMount,
  onBeforeUpdate,
  onBeforeUnmount 
} from 'vue'

export default {
  setup() {
    onBeforeMount(() => {
      console.log('即将挂载')
    })
    
    onMounted(() => {
      console.log('挂载完成')
      // 适合 DOM 操作、第三方库初始化
    })
    
    onBeforeUpdate(() => {
      console.log('即将更新')
    })
    
    onUpdated(() => {
      console.log('更新完成')
    })
    
    onBeforeUnmount(() => {
      console.log('即将卸载')
    })
    
    onUnmounted(() => {
      console.log('卸载完成')
      // 清理定时器、事件监听器
    })
    
    return {}
  }
}
```

### 依赖注入 provide/inject

跨层级组件通信：

```javascript
// 祖先组件
import { provide, ref } from 'vue'

export default {
  setup() {
    const theme = ref('dark')
    
    provide('theme', theme)
    provide('toggleTheme', () => {
      theme.value = theme.value === 'dark' ? 'light' : 'dark'
    })
  }
}
```

```javascript
// 后代组件
import { inject } from 'vue'

export default {
  setup() {
    const theme = inject('theme')
    const toggleTheme = inject('toggleTheme')
    
    return { theme, toggleTheme }
  }
}
```

---

## 路由 Vue Router

Vue Router 是 Vue.js 的官方路由管理器。[^3]

### 路由配置

```javascript
// router/index.js
import { createRouter, createWebHistory } from 'vue-router'
import Home from './views/Home.vue'
import About from './views/About.vue'
import User from './views/User.vue'

const routes = [
  {
    path: '/',
    name: 'Home',
    component: Home
  },
  {
    path: '/about',
    name: 'About',
    component: About
  },
  {
    // 动态路由
    path: '/user/:id',
    name: 'User',
    component: User,
    props: true  // 将路由参数作为 props 传递
  },
  {
    // 多段匹配
    path: '/user/:id/post/:postId',
    component: () => import('./views/UserPost.vue')
  },
  {
    path: '/:pathMatch(.*)*',
    redirect: '/'
  }
]

const router = createRouter({
  history: createWebHistory(),
  routes
})

export default router
```

```javascript
// main.js
import { createApp } from 'vue'
import App from './App.vue'
import router from './router'

createApp(App)
  .use(router)
  .mount('#app')
```

### 路由使用

```html
<!-- 声明式导航 -->
<template>
  <nav>
    <router-link to="/">首页</router-link>
    <router-link :to="{ name: 'User', params: { id: 1 }}">用户</router-link>
    <router-link replace to="/about">关于（替换历史）</router-link>
  </nav>
  
  <router-view></router-view>
  <router-view name="sidebar"></router-view>
</template>
```

```javascript
// 编程式导航
export default {
  setup() {
    const router = useRouter()
    const route = useRoute()
    
    // 导航到路径
    router.push('/about')
    router.push({ name: 'User', params: { id: 1 }})
    
    // 导航到当前位置（刷新）
    router.go(0)
    
    // 替换当前记录
    router.replace('/about')
    
    // 获取路由参数
    const id = route.params.id
    const query = route.query
  }
}
```

### 导航守卫

```javascript
// 全局前置守卫
router.beforeEach((to, from, next) => {
  // to: 目标路由
  // from: 来源路由
  // next: 放行函数
  
  if (to.meta.requiresAuth && !isAuthenticated) {
    next('/login')
  } else {
    next()
  }
})

// 全局后置守卫
router.afterEach((to, from) => {
  console.log('导航到:', to.path)
  // 适合页面标题修改、滚动位置等
})

// 路由独享守卫
const routes = [
  {
    path: '/admin',
    component: Admin,
    beforeEnter: (to, from, next) => {
      if (isAdmin) {
        next()
      } else {
        next('/')
      }
    }
  }
]

// 组件内守卫
export default {
  setup() {
    onBeforeRouteEnter((to, from, next) => {
      // 路由进入前，this 不可用
      next(vm => {
        // 可以在回调中访问组件实例
      })
    })
    
    onBeforeRouteUpdate((to, from) => {
      // 路由更新（同一组件，参数变化）
    })
    
    onBeforeRouteLeave((to, from) => {
      // 路由离开
      const answer = window.confirm('确定离开？')
      if (answer) {
        next()
      } else {
        next(false)
      }
    })
  }
}
```

| 守卫类型 | 调用时机 |
|----------|----------|
| beforeEach | 路由进入前（全局） |
| beforeResolve | 路由确认前（全局） |
| afterEach | 路由进入后（全局） |
| beforeEnter | 路由进入前（路由独享） |
| beforeRouteEnter | 组件渲染前调用 |
| beforeRouteUpdate | 路由更新时调用 |
| beforeRouteLeave | 路由离开时调用 |

---

## 状态管理 Pinia

Pinia 是 Vue 3 的新一代状态管理库，API 设计类似 Vuex。[^4]

### Store 定义

```javascript
// stores/user.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  // 状态
  const name = ref('Alice')
  const age = ref(18)
  const isVip = ref(false)
  
  // 计算属性
  const isAdult = computed(() => age.value >= 18)
  const level = computed(() => isVip.value ? 'VIP' : '普通用户')
  
  // 方法（actions）
  const setUser = (newName, newAge) => {
    name.value = newName
    age.value = newAge
  }
  
  const upgradeToVip = () => {
    isVip.value = true
  }
  
  const reset = () => {
    name.value = 'Alice'
    age.value = 18
    isVip.value = false
  }
  
  return {
    name,
    age,
    isVip,
    isAdult,
    level,
    setUser,
    upgradeToVip,
    reset
  }
})
```

### getters 和 actions

```javascript
// stores/cart.js
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useCartStore = defineStore('cart', () => {
  // 状态
  const items = ref([])
  
  // getters
  const totalItems = computed(() => items.value.length)
  
  const totalPrice = computed(() => {
    return items.value.reduce((sum, item) => {
      return sum + item.price * item.quantity
    }, 0)
  })
  
  const isEmpty = computed(() => items.value.length === 0)
  
  // actions
  const addItem = (product) => {
    const existing = items.value.find(i => i.id === product.id)
    if (existing) {
      existing.quantity++
    } else {
      items.value.push({ ...product, quantity: 1 })
    }
  }
  
  const removeItem = (id) => {
    const index = items.value.findIndex(i => i.id === id)
    if (index > -1) {
      items.value.splice(index, 1)
    }
  }
  
  const clearCart = () => {
    items.value = []
  }
  
  return {
    items,
    totalItems,
    totalPrice,
    isEmpty,
    addItem,
    removeItem,
    clearCart
  }
})
```

### 组件中使用 Store

```html
<script setup>
import { useUserStore } from './stores/user'
import { useCartStore } from './stores/cart'

const userStore = useUserStore()
const cartStore = useCartStore()

// 直接访问状态
console.log(userStore.name)

// 调用 actions
const handleLogin = () => {
  userStore.setUser('Bob', 25)
}

const handleBuy = (product) => {
  cartStore.addItem(product)
}
</script>

<template>
  <div>
    <p>用户名: {{ userStore.name }}</p>
    <p>会员等级: {{ userStore.level }}</p>
    <p>购物车: {{ cartStore.totalItems }} 件商品</p>
  </div>
</template>
```

:::tip
Pinia 支持直接修改状态（响应式），但建议通过 actions 修改，以保持状态变化的可追踪性。
:::

---

## 实战示例：TODO 应用

一个完整的 TODO 应用，综合运用组件、响应式、localStorage 持久化。

### 项目结构

```
src/
├── components/
│   ├── TodoInput.vue
│   ├── TodoItem.vue
│   └── TodoFilters.vue
├── stores/
│   └── todo.js
├── App.vue
└── main.js
```

### Store 定义

```javascript
// stores/todo.js
import { defineStore } from 'pinia'
import { ref, computed, watch } from 'vue'

export const useTodoStore = defineStore('todo', () => {
  const todos = ref(JSON.parse(localStorage.getItem('todos') || '[]'))
  const filter = ref('all') // all, active, completed
  
  // 持久化
  watch(todos, (newTodos) => {
    localStorage.setItem('todos', JSON.stringify(newTodos))
  }, { deep: true })
  
  // getters
  const filteredTodos = computed(() => {
    switch (filter.value) {
      case 'active':
        return todos.value.filter(t => !t.completed)
      case 'completed':
        return todos.value.filter(t => t.completed)
      default:
        return todos.value
    }
  })
  
  const stats = computed(() => ({
    total: todos.value.length,
    active: todos.value.filter(t => !t.completed).length,
    completed: todos.value.filter(t => t.completed).length
  }))
  
  // actions
  const addTodo = (text) => {
    if (!text.trim()) return
    todos.value.push({
      id: Date.now(),
      text: text.trim(),
      completed: false,
      createdAt: new Date().toISOString()
    })
  }
  
  const toggleTodo = (id) => {
    const todo = todos.value.find(t => t.id === id)
    if (todo) {
      todo.completed = !todo.completed
    }
  }
  
  const deleteTodo = (id) => {
    const index = todos.value.findIndex(t => t.id === id)
    if (index > -1) {
      todos.value.splice(index, 1)
    }
  }
  
  const editTodo = (id, newText) => {
    const todo = todos.value.find(t => t.id === id)
    if (todo && newText.trim()) {
      todo.text = newText.trim()
    }
  }
  
  const clearCompleted = () => {
    todos.value = todos.value.filter(t => !t.completed)
  }
  
  return {
    todos,
    filter,
    filteredTodos,
    stats,
    addTodo,
    toggleTodo,
    deleteTodo,
    editTodo,
    clearCompleted
  }
})
```

### TodoInput 组件

```html
<!-- components/TodoInput.vue -->
<template>
  <div class="todo-input">
    <input 
      v-model="inputText"
      @keyup.enter="handleAdd"
      placeholder="输入新的待办事项..."
      class="input"
    />
    <button @click="handleAdd" class="btn-add">添加</button>
  </div>
</template>

<script setup>
import { ref } from 'vue'
import { useTodoStore } from '../stores/todo'

const todoStore = useTodoStore()
const inputText = ref('')

const handleAdd = () => {
  todoStore.addTodo(inputText.value)
  inputText.value = ''
}
</script>

<style scoped>
.todo-input {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
}

.input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 4px;
  font-size: 14px;
}

.btn-add {
  padding: 8px 16px;
  background: #42b983;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}

.btn-add:hover {
  background: #3aa87a;
}
</style>
```

### TodoItem 组件

```html
<!-- components/TodoItem.vue -->
<template>
  <div class="todo-item" :class="{ completed: todo.completed }">
    <input 
      type="checkbox"
      :checked="todo.completed"
      @change="todoStore.toggleTodo(todo.id)"
    />
    
    <span 
      v-if="!isEditing" 
      class="text"
      @dblclick="startEdit"
    >
      {{ todo.text }}
    </span>
    
    <input 
      v-else
      v-model="editText"
      @keyup.enter="saveEdit"
      @blur="saveEdit"
      @keyup.escape="cancelEdit"
      class="edit-input"
      ref="editInputRef"
    />
    
    <button @click="deleteTodo" class="btn-delete">删除</button>
  </div>
</template>

<script setup>
import { ref, nextTick } from 'vue'
import { useTodoStore } from '../stores/todo'

const props = defineProps({
  todo: {
    type: Object,
    required: true
  }
})

const todoStore = useTodoStore()
const isEditing = ref(false)
const editText = ref('')
const editInputRef = ref(null)

const startEdit = () => {
  isEditing.value = true
  editText.value = props.todo.text
  nextTick(() => {
    editInputRef.value?.focus()
  })
}

const saveEdit = () => {
  if (editText.value.trim()) {
    todoStore.editTodo(props.todo.id, editText.value)
  }
  isEditing.value = false
}

const cancelEdit = () => {
  isEditing.value = false
}

const deleteTodo = () => {
  todoStore.deleteTodo(props.todo.id)
}
</script>

<style scoped>
.todo-item {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 8px;
  border-bottom: 1px solid #eee;
}

.todo-item.completed .text {
  text-decoration: line-through;
  color: #999;
}

.text {
  flex: 1;
  cursor: pointer;
}

.edit-input {
  flex: 1;
  padding: 4px 8px;
  border: 1px solid #42b983;
  border-radius: 4px;
}

.btn-delete {
  padding: 4px 8px;
  background: #ff6b6b;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style>
```

### TodoFilters 组件

```html
<!-- components/TodoFilters.vue -->
<template>
  <div class="filters">
    <span class="stats">
      {{ stats.active }} 项待完成 / {{ stats.completed }} 项已完成
    </span>
    
    <div class="filter-buttons">
      <button 
        v-for="f in filters" 
        :key="f.value"
        :class="{ active: filter === f.value }"
        @click="todoStore.filter = f.value"
      >
        {{ f.label }}
      </button>
    </div>
    
    <button 
      v-if="stats.completed > 0"
      @click="todoStore.clearCompleted"
      class="btn-clear"
    >
      清除已完成
    </button>
  </div>
</template>

<script setup>
import { useTodoStore } from '../stores/todo'

const todoStore = useTodoStore()
const { filter, stats } = todoStore

const filters = [
  { label: '全部', value: 'all' },
  { label: '进行中', value: 'active' },
  { label: '已完成', value: 'completed' }
]
</script>

<style scoped>
.filters {
  display: flex;
  align-items: center;
  gap: 16px;
  padding: 8px 0;
}

.stats {
  color: #666;
  font-size: 14px;
}

.filter-buttons {
  display: flex;
  gap: 4px;
}

.filter-buttons button {
  padding: 4px 12px;
  border: 1px solid #ddd;
  background: white;
  border-radius: 4px;
  cursor: pointer;
}

.filter-buttons button.active {
  background: #42b983;
  color: white;
  border-color: #42b983;
}

.btn-clear {
  padding: 4px 12px;
  background: #ff6b6b;
  color: white;
  border: none;
  border-radius: 4px;
  cursor: pointer;
}
</style>
```

### App.vue 主组件

```html
<!-- App.vue -->
<template>
  <div class="app">
    <h1>我的 TODO</h1>
    
    <TodoInput />
    
    <div class="todo-list">
      <TodoItem 
        v-for="todo in filteredTodos" 
        :key="todo.id"
        :todo="todo"
      />
      
      <p v-if="filteredTodos.length === 0" class="empty">
        {{ filter === 'all' ? '暂无待办事项' : '没有符合筛选条件的待办' }}
      </p>
    </div>
    
    <TodoFilters />
  </div>
</template>

<script setup>
import TodoInput from './components/TodoInput.vue'
import TodoItem from './components/TodoItem.vue'
import TodoFilters from './components/TodoFilters.vue'
import { useTodoStore } from './stores/todo'

const todoStore = useTodoStore()
const { filteredTodos, filter } = todoStore
</script>

<style>
* {
  box-sizing: border-box;
}

.app {
  max-width: 500px;
  margin: 40px auto;
  padding: 20px;
  font-family: Arial, sans-serif;
}

h1 {
  text-align: center;
  color: #42b983;
}

.todo-list {
  border: 1px solid #ddd;
  border-radius: 4px;
  margin-bottom: 16px;
}

.empty {
  text-align: center;
  padding: 20px;
  color: #999;
}
</style>
```

### main.js 入口文件

```javascript
// main.js
import { createApp } from 'vue'
import { createPinia } from 'pinia'
import App from './App.vue'

const app = createApp(App)
const pinia = createPinia()

app.use(pinia)
app.mount('#app')
```

---

## 总结

本教程涵盖了 Vue.js 的核心知识点：

| 模块 | 主要内容 |
|------|----------|
| [[../frontend/vue-basics\|Vue 基础]] | 模板语法、响应式数据、组件系统 |
| [[../frontend/vue-composition-api\|Composition API]] | setup、ref/reactive、生命周期 |
| [[../frontend/vue-router\|Vue Router]] | 路由配置、导航守卫 |
| [[../frontend/vue-pinia\|Pinia]] | Store、getters、actions |

进一步学习建议：
- 深入学习 [[../frontend/vue3-advanced\|Vue 3 高级特性]]
- 了解 [[../frontend/vue-ecology\|Vue 生态圈]]（Vite、Vitest、VueUse）
- 实践 [[../frontend/vue3-project\|Vue 3 项目开发]]

[^1]: 本段来自 [Vue.js 官方文档](https://vuejs.org/)
[^2]: 本段来自 [Vue Composition API 指南](https://vuejs.org/guide/extras/composition-api-faq.html)
[^3]: 本段来自 [Vue Router 官方文档](https://router.vuejs.org/)
[^4]: 本段来自 [Pinia 官方文档](https://pinia.vuejs.org/)
