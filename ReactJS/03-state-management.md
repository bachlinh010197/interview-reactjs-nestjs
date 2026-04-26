# 📘 ReactJS - Phần 3: State Management

[⬅️ Hooks](./02-hooks.md) | [Phần tiếp: Router ➡️](./04-router.md)

---

## 1. Context API vs Redux

**Câu hỏi: Khi nào dùng Context API, khi nào dùng Redux?**

**Trả lời:**

| Tiêu chí | Context API | Redux |
|---|---|---|
| Quy mô | Nhỏ - trung bình | Trung bình - lớn |
| Tần suất update | Thấp (theme, auth, locale) | Cao (real-time data) |
| Performance | Re-render tất cả consumer | Selective subscription |
| DevTools | Không | Redux DevTools mạnh mẽ |
| Middleware | Không | Thunk, Saga, etc. |
| Boilerplate | Ít | Nhiều (Redux Toolkit giảm bớt) |

**Dùng Context:** Theme, language, auth status, feature flags

**Dùng Redux:** Shopping cart, complex forms, real-time dashboard, undo/redo

---

## 2. Redux Toolkit (RTK)

**Câu hỏi: Giải thích Redux flow và cách dùng Redux Toolkit.**

**Trả lời:**

**Redux flow:** UI → dispatch(action) → Reducer → New State → UI re-render

```jsx
// store.js
import { configureStore } from '@reduxjs/toolkit';
import todoReducer from './todoSlice';

export const store = configureStore({
  reducer: {
    todos: todoReducer,
  },
});

// todoSlice.js
import { createSlice, createAsyncThunk } from '@reduxjs/toolkit';

// Async action
export const fetchTodos = createAsyncThunk('todos/fetch', async () => {
  const res = await fetch('/api/todos');
  return res.json();
});

const todoSlice = createSlice({
  name: 'todos',
  initialState: { items: [], loading: false, error: null },
  reducers: {
    addTodo: (state, action) => {
      // Immer cho phép "mutate" trực tiếp
      state.items.push(action.payload);
    },
    removeTodo: (state, action) => {
      state.items = state.items.filter(t => t.id !== action.payload);
    },
    toggleTodo: (state, action) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) todo.completed = !todo.completed;
    },
  },
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => { state.loading = true; })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      })
      .addCase(fetchTodos.rejected, (state, action) => {
        state.loading = false;
        state.error = action.error.message;
      });
  },
});

export const { addTodo, removeTodo, toggleTodo } = todoSlice.actions;
export default todoSlice.reducer;

// Component sử dụng
function TodoList() {
  const { items, loading } = useSelector(state => state.todos);
  const dispatch = useDispatch();

  useEffect(() => {
    dispatch(fetchTodos());
  }, [dispatch]);

  return (
    <ul>
      {items.map(todo => (
        <li key={todo.id} onClick={() => dispatch(toggleTodo(todo.id))}>
          {todo.text}
        </li>
      ))}
    </ul>
  );
}
```

---

## 3. Redux Middleware

**Câu hỏi: Redux middleware là gì? So sánh Thunk vs Saga.**

**Trả lời:**

Middleware là lớp trung gian giữa dispatch action và reducer, dùng để xử lý side effects.

| | Redux Thunk | Redux Saga |
|---|---|---|
| Cú pháp | Async/await, đơn giản | Generator functions |
| Learning curve | Thấp | Cao |
| Xử lý phức tạp | Khó với race condition | Mạnh: takeLatest, debounce... |
| Testing | Mock fetch | Dễ test generator |
| Dùng khi | CRUD đơn giản | Workflow phức tạp |

---

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 4. RTK Query — cách thay thế useEffect + useState cho API calls

**Câu hỏi: RTK Query giải quyết vấn đề gì? So sánh với React Query.**

**Trả lời:**

RTK Query là data fetching & caching tool tích hợp trong Redux Toolkit, loại bỏ boilerplate của createAsyncThunk.

```jsx
// apiSlice.js
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

export const apiSlice = createApi({
  reducerPath: 'api',
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  tagTypes: ['User', 'Post'],
  endpoints: (builder) => ({
    getUsers: builder.query({
      query: () => '/users',
      providesTags: ['User'],
    }),
    getUserById: builder.query({
      query: (id) => `/users/${id}`,
      providesTags: (result, error, id) => [{ type: 'User', id }],
    }),
    createUser: builder.mutation({
      query: (body) => ({ url: '/users', method: 'POST', body }),
      invalidatesTags: ['User'], // Auto refetch getUsers
    }),
    updateUser: builder.mutation({
      query: ({ id, ...body }) => ({ url: `/users/${id}`, method: 'PUT', body }),
      invalidatesTags: (result, error, { id }) => [{ type: 'User', id }],
    }),
  }),
});

export const {
  useGetUsersQuery,
  useGetUserByIdQuery,
  useCreateUserMutation,
  useUpdateUserMutation,
} = apiSlice;

// Component — không cần useState, useEffect, loading state!
function UserList() {
  const { data: users, isLoading, error } = useGetUsersQuery();
  const [createUser, { isLoading: isCreating }] = useCreateUserMutation();

  if (isLoading) return <Spinner />;
  return (
    <div>
      <button onClick={() => createUser({ name: 'New User' })} disabled={isCreating}>
        Add User
      </button>
      {users.map(user => <UserCard key={user.id} user={user} />)}
    </div>
  );
}
```

| | RTK Query | React Query (TanStack) |
|---|---|---|
| Bundled with | Redux Toolkit | Standalone |
| Redux integration | Native | Cần adapter |
| Cache invalidation | Tag-based | Key-based |
| Bundle size | Nhỏ hơn (nếu đã dùng Redux) | ~13KB |
| DevTools | Redux DevTools | React Query DevTools |
| Khi nào dùng | Đã dùng Redux | Không dùng Redux |

---

### 5. Zustand — Lightweight alternative cho Redux

**Câu hỏi: Zustand là gì? Khi nào dùng thay Redux?**

**Trả lời:**

Zustand là state management library siêu nhẹ (~1KB), không cần Provider, không boilerplate.

```jsx
import { create } from 'zustand';
import { devtools, persist } from 'zustand/middleware';

// Tạo store — đơn giản hơn Redux rất nhiều
const useCartStore = create(
  devtools(
    persist(
      (set, get) => ({
        items: [],
        totalItems: 0,

        addItem: (product) => set((state) => {
          const existing = state.items.find(i => i.id === product.id);
          if (existing) {
            return {
              items: state.items.map(i =>
                i.id === product.id ? { ...i, qty: i.qty + 1 } : i
              ),
              totalItems: state.totalItems + 1,
            };
          }
          return {
            items: [...state.items, { ...product, qty: 1 }],
            totalItems: state.totalItems + 1,
          };
        }),

        removeItem: (id) => set((state) => ({
          items: state.items.filter(i => i.id !== id),
          totalItems: state.totalItems - state.items.find(i => i.id === id)?.qty,
        })),

        // Async action — không cần middleware!
        checkout: async () => {
          const { items } = get();
          await api.createOrder(items);
          set({ items: [], totalItems: 0 });
        },
      }),
      { name: 'cart-storage' }, // persist to localStorage
    ),
  ),
);

// Sử dụng — tự động selective subscription (không re-render thừa)
function CartCount() {
  const totalItems = useCartStore((state) => state.totalItems);
  return <span>Cart ({totalItems})</span>;
}

function CartItems() {
  const items = useCartStore((state) => state.items);
  const removeItem = useCartStore((state) => state.removeItem);
  return items.map(item => (
    <div key={item.id}>
      {item.name} x{item.qty}
      <button onClick={() => removeItem(item.id)}>Remove</button>
    </div>
  ));
}
```

**Khi nào dùng Zustand thay Redux?**
- App nhỏ-trung bình, không cần middleware phức tạp
- Muốn API đơn giản, ít boilerplate
- Cần performance tốt (selective subscription built-in)
- Không cần Redux DevTools ecosystem

---

### 6. State Machine Pattern với useReducer (Senior)

**Câu hỏi: Cách dùng state machine pattern trong React?**

**Trả lời:**

State machine đảm bảo app chỉ ở **một trạng thái hợp lệ** tại bất kỳ thời điểm nào, ngăn impossible states.

```jsx
// ❌ Boolean soup — dễ gây impossible states
const [isLoading, setIsLoading] = useState(false);
const [error, setError] = useState(null);
const [data, setData] = useState(null);
// Có thể xảy ra: isLoading=true AND error!=null → vô lý!

// ✅ State machine — chỉ 1 trạng thái tại 1 thời điểm
const initialState = { status: 'idle', data: null, error: null };

function fetchReducer(state, action) {
  switch (state.status) {
    case 'idle':
      if (action.type === 'FETCH') return { status: 'loading', data: null, error: null };
      return state;
    case 'loading':
      if (action.type === 'SUCCESS') return { status: 'success', data: action.payload, error: null };
      if (action.type === 'ERROR') return { status: 'error', data: null, error: action.payload };
      return state;
    case 'success':
      if (action.type === 'FETCH') return { status: 'loading', data: null, error: null };
      return state;
    case 'error':
      if (action.type === 'FETCH') return { status: 'loading', data: null, error: null };
      return state;
    default:
      return state;
  }
}

function UserProfile({ userId }) {
  const [state, dispatch] = useReducer(fetchReducer, initialState);

  useEffect(() => {
    dispatch({ type: 'FETCH' });
    fetchUser(userId)
      .then(data => dispatch({ type: 'SUCCESS', payload: data }))
      .catch(err => dispatch({ type: 'ERROR', payload: err.message }));
  }, [userId]);

  switch (state.status) {
    case 'idle': return null;
    case 'loading': return <Spinner />;
    case 'error': return <Error message={state.error} onRetry={() => dispatch({ type: 'FETCH' })} />;
    case 'success': return <div>{state.data.name}</div>;
  }
}
```
