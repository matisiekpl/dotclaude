# Vue Frontend Project Guidelines

A reference guide for structuring, building, and maintaining Vue 3 frontend projects using Pinia, Vue Router, shadcn,
and related tooling.

---

## Table of Contents

1. [Project Structure](#project-structure)
2. [Tech Stack](#tech-stack)
3. [Configuration](#configuration)
4. [Entry Point](#entry-point)
5. [Router](#router)
6. [API Layer](#api-layer)
7. [Stores](#stores)
8. [Views & Components](#views--components)
9. [Types](#types)
10. [Utilities](#utilities)
11. [Coding Style](#coding-style)

---

## Project Structure

```
src/
  api/                        # Axios API client files
    base.ts                   # Axios instance constructor with interceptors
    auth.ts                   # Example: per-domain API module
    utils.ts                  # Shared API helpers (e.g. error extraction)
  assets/                     # Static assets (images, icons, etc.)
  components/
    ui/                       # shadcn-vue components (do not modify directly)
  elements/                   # Custom shared Vue components (reusable across views)
  lib/                        # General TypeScript utilities (e.g. utils.ts)
  stores/                     # Pinia store files
  types/
    dto/                      # DTO types matching API request/response shapes
    models/                   # Domain model types
  views/                      # Vue view components, organized by domain
    Auth/
      LoginView.vue
      RegisterView.vue
    Dashboard/
      DashboardView.vue
  App.vue                     # Root Vue component
  config.ts                   # App-wide configuration constants
  router.ts                   # Vue Router setup
  style.css                   # Global styles
  main.ts                     # Application entry point
```

**File naming conventions:**

- Stores: `auth.store.ts`, `workspace.store.ts`
- DTO types: `authResponse.ts`, `articleData.ts`
- Model types: `user.ts`, `workspace.ts`
- Views (routable): `LoginView.vue`, `DashboardView.vue` — always use `View` suffix
- Elements (non-routable): `EditorTopBar.vue` — no `View` suffix
- Keep names short and concise

---

## Tech Stack

| Library                 | Purpose                                     |
|-------------------------|---------------------------------------------|
| Yarn                    | Package Manager                             |
| Vue 3 (Composition API) | UI framework                                |
| Pinia                   | State management                            |
| Vue Router              | Client-side routing                         |
| shadcn-vue              | UI component library (`src/components/ui/`) |
| Axios                   | HTTP client                                 |
| `vue-sonner`            | Toast notifications                         |
| `@vueuse/core`          | Vue composition utilities                   |
| `lucide-vue-next`       | Icon library                                |
| `jwt-decode`            | JWT decoding in the API interceptor         |
| shadcn-vue              | UI Framework                                |

---

## Configuration

App-wide constants (API base URL, feature flags, access keys, etc.) live in `src/config.ts`. The server URL switches
automatically based on the current hostname.
Don't use any environment variables config from `.env`.
Configure EVERY variable within that `config.ts`.

```typescript
// src/config.ts
export const Config = {
  serverUrl: window.location.href.includes('localhost')
          ? 'http://localhost:3000'
          : `${window.location.protocol}//${window.location.hostname}`,
  sampleAccessKey: 'sample-config-access-key',
};
```

> Add new config values here rather than scattering `import.meta.env` calls throughout the codebase.

---

## Entry Point

```typescript
// src/main.ts
import {createApp} from 'vue'
import './style.css'
import App from './App.vue'
import {router} from '@/router.ts'
import {createPinia} from 'pinia'

createApp(App).use(router).use(createPinia()).mount('#app')
```

---

## Router

All routes are defined in `src/router.ts`. Use named routes for navigation. Nest child routes under layout components
where appropriate.

```typescript
// src/router.ts
import {createRouter, createWebHistory} from 'vue-router'

const routes = [
  {path: '/login', component: LoginView, name: 'login'},
  {path: '/register', component: RegisterView, name: 'register'},
  {path: '/password/forgot', component: ForgotPasswordView, name: 'password.forgot'},
  {path: '/password/reset', component: ResetPasswordView, name: 'password.reset'},
  {
    path: '/workspaces/:workspaceID', component: DashboardView,
    children: [
      {path: 'collections', component: CollectionsListView, name: 'collections.list'},
      {path: 'collections/:collectionID/articles', component: ArticlesListView, name: 'collections.show'},
      {path: 'collections/:collectionID/articles/:articleID/editor', component: ArticleEditor, name: 'editor'},
    ]
  },
  {
    path: '/oauth/:provider/callback',
    component: ThirdPartyExchange
  },
]

export const router = createRouter({
  history: createWebHistory(),
  routes,
})
```

---

## API Layer

Each domain has its own file in `src/api/`. Every file exports a single named object (e.g. `authApi`) containing all
functions for that domain. Functions are `async` and return typed values.

### Axios Instance (`src/api/base.ts`)

The base Axios instance attaches the `Authorization` header on every request and redirects to `/login` if the stored JWT
has expired.

```typescript
// src/api/base.ts
import axios, {type InternalAxiosRequestConfig} from 'axios'
import {ACCESS_TOKEN} from '@/stores/auth.store.ts'
import {Config} from '@/config.ts'
import {jwtDecode} from 'jwt-decode'

const api = axios.create({
  baseURL: Config.serverUrl + '/api/',
})

api.interceptors.request.use(
        (config: InternalAxiosRequestConfig) => {
          const token = localStorage.getItem(ACCESS_TOKEN)
          if (token) {
            const decoded = jwtDecode(token) as { exp: number }
            const currentTime = Date.now() / 1000
            if (decoded && decoded.exp < currentTime && !window.location.href.includes('/widget/')) {
              localStorage.removeItem(ACCESS_TOKEN)
              window.location.href = '/login'
            }
            config.headers.Authorization = `Bearer ${token}`
          }
          return config
        },
)

export default api
```

### Error Helper (`src/api/utils.ts`)

Extracts the user-facing error message from an Axios error response and capitalizes the first letter.

```typescript
// src/api/utils.ts
import type {AxiosError} from 'axios'

export function getAppError(err: any): string {
  const message = (((err as AxiosError).response!.data! as any).message) as string
  return String(message).charAt(0).toUpperCase() + String(message).slice(1)
}
```

### Example API Module (`src/api/auth.ts`)

```typescript
// src/api/auth.ts
import api from '@/api/base.ts'
import type {AuthResponse} from '@/types/dto/authResponse.ts'
import type {User} from '@/types/models/user.ts'

async function login(email: string, password: string): Promise<string> {
  const response = await api.post('auth/login', {email, password})
  return (response.data as AuthResponse).token
}

async function register(name: string, email: string, password: string): Promise<string> {
  const response = await api.post('auth/register', {name, email, password})
  return (response.data as AuthResponse).token
}

async function me(): Promise<User> {
  const response = await api.get('auth/me')
  return response.data as User
}

async function update(user: User): Promise<User> {
  const response = await api.put('auth/me', user)
  return response.data as User
}

async function loginWith(provider: 'google' | 'github'): Promise<string> {
  const response = await api.get(`auth/${provider}/login`)
  return response.data['url'] as string
}

async function exchangeWith(provider: 'google' | 'github', code: string): Promise<string> {
  const response = await api.post(`auth/${provider}/exchange`, {code})
  return (response.data as AuthResponse).token
}

export const authApi = {
  login,
  register,
  me,
  update,
  loginWith,
  exchangeWith,
}
```

---

## Stores

Pinia stores live in `src/stores/`. Each store is named after its domain: `auth.store.ts`, `workspace.store.ts`.

**Rules:**

- Keep most business logic in stores, including loading states and error handling.
- Only move logic into a component if it is tightly coupled to that component's lifecycle.
- Use `toast.error(getAppError(err))` for user-facing error messages in catch blocks.
- Export localStorage key constants from the store file that owns them (e.g. `ACCESS_TOKEN` from `auth.store.ts`).

### Example Store (`src/stores/auth.store.ts`)

```typescript
import {defineStore} from 'pinia'
import {ref} from 'vue'
import {authApi} from '@/api/auth.ts'
import {getAppError} from '@/api/utils.ts'
import {toast} from 'vue-sonner'
import {useRouter, useRoute} from 'vue-router'
import type {User} from '@/types/models/user.ts'
import {WORKSPACE_ID} from '@/stores/workspace.store.ts'

export const ACCESS_TOKEN = 'ACCESS_TOKEN'
export const RETURN_URL = 'RETURN_URL'
export const CONNECT_PROVIDER = 'CONNECT_PROVIDER'

export const useAuthStore = defineStore('auth', () => {
  const name = ref<string>('')
  const email = ref<string>('')
  const password = ref<string>('')
  const router = useRouter()
  const route = useRoute()
  const loading = ref(false)
  const user = ref<User | undefined>(undefined)
  const initialized = ref(false)

  async function check() {
    if (localStorage.getItem(ACCESS_TOKEN)) {
      await router.push('/')
    }
    if (route.query.email) {
      email.value = route.query.email as string
    }
  }

  async function login() {
    try {
      loading.value = true
      const token = await authApi.login(email.value, password.value)
      localStorage.setItem(ACCESS_TOKEN, token)
      await init()
      const returnUrl = localStorage.getItem(RETURN_URL)
              || new URLSearchParams(window.location.search).get('return')
              || '/'
      localStorage.removeItem(RETURN_URL)
      await router.push(returnUrl)
    } catch (err) {
      toast.error(getAppError(err))
    } finally {
      loading.value = false
    }
  }

  async function register() {
    try {
      loading.value = true
      const token = await authApi.register(name.value, email.value, password.value)
      localStorage.setItem(ACCESS_TOKEN, token)
      await init()
      const returnUrl = localStorage.getItem(RETURN_URL)
              || new URLSearchParams(window.location.search).get('return')
              || '/'
      localStorage.removeItem(RETURN_URL)
      await router.push(returnUrl)
    } catch (err) {
      toast.error(getAppError(err))
    } finally {
      loading.value = false
    }
  }

  async function logout() {
    localStorage.removeItem(ACCESS_TOKEN)
    localStorage.removeItem(WORKSPACE_ID)
    await router.push('/login')
  }

  async function init() {
    user.value = await authApi.me()
    initialized.value = true
  }

  return {check, login, register, logout, init, name, email, password, loading, user, initialized}
})
```

---

## Views & Components

### Views (`src/views/`)

Views are organized into subdirectories by domain (e.g. `Auth/`, `Dashboard/`). All routable components use the `View`
suffix. Non-routable components (rendered inside a view) do not.

```
src/views/
  Auth/
    LoginView.vue      ← routable
    RegisterView.vue   ← routable
  Dashboard/
    DashboardView.vue  ← routable
    EditorTopBar.vue   ← NOT routable, no View suffix
```

### Example View (`src/views/Auth/LoginView.vue`)

```vue

<script setup lang="ts">
  import {Label} from '@/components/ui/label'
  import {Button} from '@/components/ui/button'
  import {Input} from '@/components/ui/input'
  import {Spinner} from '@/components/ui/spinner'
  import {useAuthStore} from '@/stores/auth.store.ts'
  import {onMounted} from 'vue'
  import {useTitle} from '@vueuse/core'
  import {useRoute} from 'vue-router'

  useTitle('Login — MyApp')
  const authStore = useAuthStore()
  const route = useRoute()

  onMounted(() => {
    authStore.check()
  })
</script>

<template>
  <div class="flex h-screen">
    <div class="m-auto flex flex-col gap-4 w-[330px]">
      <Label class="text-3xl">Login to MyApp</Label>
      <Label :secondary="true" class="!text-md">Welcome back</Label>

      <Button class="w-full" variant="outline" @click="authStore.loginWith('google')" type="button">
        <Spinner v-if="authStore.googleLoading" class="mr-2"/>
        <img src="../../assets/google.png" alt="Google" class="size-4 mt-0.5"/>
        Continue with Google
      </Button>

      <Button class="w-full" variant="outline" @click="authStore.loginWith('github')" type="button">
        <Spinner v-if="authStore.googleLoading" class="mr-2"/>
        <img src="../../assets/github.png" alt="GitHub" class="size-4 mt-0.5"/>
        Continue with GitHub
      </Button>

      <hr class="my-4"/>

      <form class="m-auto flex flex-col gap-4 w-[330px]" @submit.prevent="authStore.login()">
        <Label>Email</Label>
        <Input type="email" v-model="authStore.email" :disabled="!!route.query.email" autofocus required
               autocomplete="email"/>

        <div class="flex items-center justify-between">
          <Label>Password</Label>
          <RouterLink to="/password/forgot" class="text-sm underline hover:text-black text-zinc-500">
            Forgot password?
          </RouterLink>
        </div>
        <Input type="password" v-model="authStore.password" required autocomplete="new-password"/>

        <Button type="submit">
          <Spinner class="animate-spin" v-if="authStore.loading"/>
          Sign in
        </Button>

        <span class="text-zinc-500 text-sm">
          Don't have an account yet?
          <RouterLink
                  :to="`/register${route.query.return ? `?return=${encodeURIComponent(route.query.return as string)}` : ''}`"
                  class="underline hover:text-black">
            Sign up here
          </RouterLink>
        </span>
      </form>
    </div>
  </div>
</template>
```

### Elements (`src/elements/`)

Reusable components shared across multiple views. These are not tied to a specific domain and have no routing
association.

---

## Types

### Models (`src/types/models/`)

Domain entity shapes, typically mirroring backend model structs.

```typescript
// src/types/models/article.ts
import type {ArticleTranslation} from '@/types/models/articleTranslation.ts'

export interface Article {
  id: string
  createdAt: Date
  updatedAt: Date
  collectionID: string
  position: number
  articleTranslations: ArticleTranslation[]
}
```

### DTOs (`src/types/dto/`)

Shapes for API request and response payloads. Named after the operation they represent.

```typescript
// src/types/dto/authResponse.ts
export interface AuthResponse {
  token: string
}
```

**Naming examples:**

- `authResponse.ts` → `AuthResponse`
- `articleData.ts` → `ArticleData`
- `loginRequest.ts` → `LoginRequest`

---

## Utilities

General TypeScript helpers live in `src/lib/` (e.g. `utils.ts`). Keep them pure and stateless — no Vue or store imports.

---

## Coding Rules

- **Business logic belongs in stores.** Only keep logic in a component when it is inseparable from that component's
  template or lifecycle.
- **Loading states live in stores.** Expose a `loading` ref per async action where needed.
- **Error handling in stores.** Use `toast.error(getAppError(err))` in `catch` blocks consistently.
- **Short, concise names.** Prefer `article` over `currentArticle`, `workspace` over `activeWorkspaceData`.
- **Typed API calls.** Every API function must declare its return type explicitly.
- **Use `<script setup>`** for all components. Avoid Options API.
- **Use named routes** for `router.push` calls — avoid hardcoding path strings outside of `router.ts`.
- **localStorage keys as exported constants.** Define them at the top of the store file that owns them (e.g.
  `export const ACCESS_TOKEN = 'ACCESS_TOKEN'`).
- **Comments in rules file** – sometimes rules author put comment with filename at the top of the file for LLM context.
  Don't add those comments in produced code.
- **Use `useTitle` hook** to set page title.w
- **Use `useRoute` hook** to get current route.
- **Use `useRouter` hook** to navigate.
- **Use `shadcn-vue` CLI** to install components (`npx shadcn-vue@latest`).
- Don't run `yarn dev` or run the app. Developer would do that.
- Use Context7 MCP for checking out libraries documentations.
- When adding shadcn Button component, please add `cursor-pointer` class to it.
- When spinning up new Vue project, keep existing files. Use tutorial from: https://www.shadcn-vue.com/docs/installation/vite
- Don't run `yarn build` to test code changes. Output the correct code.
- For button loading state, use `<Spinner/>` component from shadcn-vue.
- When asked for login and register feature, add `/logout` route that will log out user.
- For sidebars, tables, section etc, use shadcn-vue components.
- Logic related to UI display, not business-logic should be inside components (for example dialog show ref).
- When implementing mutating CRUD actions in store, refetch data from server, don't modify data in memory directly to avoid dual source of truth.
- Don't introduce helper methods that do very small things. For small things, prefer inline functions calls.
- Never run application, never run typecheck. Developer would do that manually.