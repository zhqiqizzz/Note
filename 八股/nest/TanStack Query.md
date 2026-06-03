**接口数据怎么优雅管理**。

很多初学者会这样写：

```ts
const loading = ref(false);
const error = ref('');
const tasks = ref([]);

async function loadTasks() {
  loading.value = true;
  try {
    tasks.value = await getTasks();
  } catch {
    error.value = '加载失败';
  } finally {
    loading.value = false;
  }
}
```

这没错，但项目一大，你会到处重复：

```
loading
error
重新请求
缓存
分页
提交后刷新列表
详情页复用
窗口聚焦重新请求
```

这就是 TanStack Query 解决的问题。

**1. TanStack Query 是什么**

TanStack Query 原来叫 React Query，但它也支持 Vue。

它专门管理 **服务端状态**。

服务端状态就是：

```
来自后端 API 的数据
可能过期
需要重新请求
需要缓存
多个组件可能共享
```

比如：

```
任务列表
用户信息
Agent 运行记录
工具调用日志
消息历史
文件列表
```

它不适合管理：

```
弹窗开关
输入框内容
侧边栏展开
主题色
当前 tab
```

这些是客户端状态，放 ref 或 Pinia。

一句话区分：

```
后端来的数据：TanStack Query
前端自己的状态：ref / reactive / Pinia
```

**2. 安装**

```bash
npm install @tanstack/vue-query
```

在 main.ts 注册：

```ts
import { createApp } from 'vue';
import {
  QueryClient,
  VueQueryPlugin,
} from '@tanstack/vue-query';
import App from './App.vue';

const queryClient = new QueryClient();

createApp(App)
  .use(VueQueryPlugin, { queryClient })
  .mount('#app');
```

**3. useQuery：查询数据**

比如任务列表。

API：

```ts
// src/api/tasks.api.ts
import { http } from './http';

export type Task = {
  id: number;
  title: string;
  status: 'todo' | 'doing' | 'done';
};

export function getTasks() {
  return http.get<Task[]>('/tasks');
}
```

封装 query：

```ts
// src/features/tasks/queries.ts
import { useQuery } from '@tanstack/vue-query';
import { getTasks } from '@/api/tasks.api';

export function useTasksQuery() {
  return useQuery({
    queryKey: ['tasks'],
    queryFn: getTasks,
  });
}
```

组件使用：

```vue
<script setup lang="ts">
import { useTasksQuery } from '@/features/tasks/queries';

const { data: tasks, isLoading, isError, error, refetch } = useTasksQuery();
</script>

<template>
  <div v-if="isLoading">加载中...</div>
  <div v-else-if="isError">加载失败</div>
  <div v-else-if="tasks?.length === 0">暂无任务</div>

  <ul v-else>
    <li v-for="task in tasks" :key="task.id">
      {{ task.title }} - {{ task.status }}
    </li>
  </ul>

  <button @click="refetch()">刷新</button>
</template>
```

你不用手写：

```
loading
error
cache
refetch
```

它都帮你管理。

**4. queryKey 很重要**

```ts
queryKey: ['tasks']
```

这个 key 表示“任务列表缓存”。

如果是任务详情：

```ts
queryKey: ['tasks', taskId]
```

如果是按状态筛选：

```ts
queryKey: ['tasks', { status }]
```

规则：

```
同一个 queryKey 共享同一份缓存
queryKey 变化会重新请求
```

例如：

```ts
export function useTasksQuery(status?: Ref<string | undefined>) {
  return useQuery({
    queryKey: ['tasks', status],
    queryFn: () => getTasks({ status: status.value }),
  });
}
```

**5. useMutation：修改数据**

`useQuery` 用来 GET。  
`useMutation` 用来 POST/PATCH/DELETE。

API：

```ts
export function createTask(data: { title: string }) {
  return http.post<Task>('/tasks', data);
}
```

Mutation：

```ts
import { useMutation, useQueryClient } from '@tanstack/vue-query';
import { createTask } from '@/api/tasks.api';

export function useCreateTaskMutation() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: createTask,
    onSuccess: () => {
      queryClient.invalidateQueries({
        queryKey: ['tasks'],
      });
    },
  });
}
```

组件：

```vue
<script setup lang="ts">
import { ref } from 'vue';
import { useCreateTaskMutation } from '@/features/tasks/queries';

const title = ref('');
const createTaskMutation = useCreateTaskMutation();

async function handleSubmit() {
  await createTaskMutation.mutateAsync({
    title: title.value,
  });

  title.value = '';
}
</script>

<template>
  <form @submit.prevent="handleSubmit">
    <input v-model="title" />
    <button :disabled="createTaskMutation.isPending.value">
      创建
    </button>
  </form>
</template>
```

注意：

```
创建成功后 invalidate ['tasks']
TanStack Query 会自动重新请求任务列表
```

这比你手动维护列表更不容易乱。

**6. invalidateQueries 是什么**

invalidateQueries 的意思是：

```
这个缓存过期了，请重新请求
```

比如：

```ts
queryClient.invalidateQueries({
  queryKey: ['tasks'],
});
```

代表：

```
所有 tasks 相关数据都需要刷新
```

常见场景：

```
创建任务后刷新列表
删除任务后刷新列表
修改状态后刷新列表和详情
```

如果更新详情：

```
点击完成
  -> 等接口成功
  -> 刷新列表
```

**7. 乐观更新**

有时你希望用户操作后 UI 立刻变化，不等接口返回。

比如切换任务状态。

普通流程：

```
点击完成
  -> 等接口成功
  -> 刷新列表
```

乐观更新：

```
点击完成
  -> 立刻把 UI 改成 done
  -> 后端成功：保持
  -> 后端失败：回滚
```

适合：

```
点赞
收藏
切换状态
简单操作
```

但复杂业务别乱用。

**8. staleTime 和 gcTime**

两个重要配置。

staleTime：

```
数据多久内算新鲜
```

例如：

```ts
useQuery({
  queryKey: ['tasks'],
  queryFn: getTasks,
  staleTime: 30_000,
});
```

意思是 30 秒内不认为过期。

gcTime：

`缓存无组件使用后，多久被清理`

默认通常够用。

常见理解：

`staleTime 控制重新请求频率 gcTime 控制缓存保留多久`

**9. enabled：条件请求**

有些请求要等条件满足。

比如有 token 才请求当前用户：

`const authStore = useAuthStore(); useQuery({ queryKey: ['me'], queryFn: getMe, enabled: computed(() => Boolean(authStore.accessToken)), });`

或者有 id 才请求详情：

`useQuery({ queryKey: ['agent-run', runId], queryFn: () => getAgentRun(runId.value), enabled: computed(() => Boolean(runId.value)), });`

**10. 分页**

分页 queryKey 要包含页码：

`const page = ref(1); const query = useQuery({ queryKey: ['tasks', { page }], queryFn: () => getTasks({ page: page.value, pageSize: 20, }), });`

后端返回：

`type PageResult<T> = { items: T[]; total: number; page: number; pageSize: number; };`

前端根据 total 渲染分页器。

**11. Agent 场景怎么用**

Agent 任务列表：

`useQuery({ queryKey: ['agent-runs'], queryFn: getAgentRuns, });`

Agent 任务详情：

`useQuery({ queryKey: ['agent-run', runId], queryFn: () => getAgentRun(runId), });`

创建 Agent 任务：

``useMutation({ mutationFn: createAgentRun, onSuccess: (data) => { router.push(`/agent-runs/${data.runId}`); }, });``

如果是 SSE 流式输出，TanStack Query 可以负责初始详情，SSE 负责实时增量：

`useQuery 加载 run 基础信息 SSE 接收 token/tool/status 事件 实时更新页面局部状态 任务完成后 invalidate agent-run`

这是很常见的组合。

**12. 它和 Pinia 怎么配合**

推荐分工：

`Pinia： accessToken currentUser 主题 全局 UI 状态 TanStack Query： 任务列表 任务详情 Agent runs 文件列表 会话历史 工具调用记录`

不要把所有后端数据放 Pinia。  
否则你要自己处理缓存、刷新、过期、重复请求，越写越累。

**13. 什么时候不用 TanStack Query**

小页面可以不用。

比如：

`一个很简单的登录表单 一次性提交 不需要缓存 不需要复用`

直接调用 API 就行。

适合用它的场景：

`列表 详情 分页 多个组件复用同一接口 需要刷新 需要缓存 需要 mutation 后同步 UI`

**14. 学习重点**

你先掌握这几个就够：

`useQuery useMutation queryKey invalidateQueries enabled staleTime 分页 mutation 后刷新列表`

一句话总结：

`TanStack Query for Vue 是管理后端接口数据的工具，让你少写重复的 loading/error/cache/refetch 逻辑。`

在 Vue 全栈/Agent 项目里，它通常和 Pinia 分工：

`Pinia 管用户登录态和前端全局状态 TanStack Query 管从后端来的数据`