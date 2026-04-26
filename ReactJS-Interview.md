# 📘 TÀI LIỆU PHỎNG VẤN REACTJS

> Tài liệu tổng hợp câu hỏi & trả lời phỏng vấn ReactJS từ cơ bản đến nâng cao.

---

## I. KIẾN THỨC NỀN TẢNG (FUNDAMENTALS)

### 1. React là gì? Tại sao nên dùng React?

**Trả lời:**

React là một thư viện JavaScript mã nguồn mở do Facebook (Meta) phát triển, dùng để xây dựng giao diện người dùng (UI), đặc biệt cho các ứng dụng Single Page Application (SPA).

**Lý do nên dùng React:**
- **Virtual DOM**: Tối ưu hiệu năng bằng cách chỉ cập nhật phần DOM thay đổi
- **Component-based**: Chia UI thành các component nhỏ, tái sử dụng được
- **One-way data flow**: Luồng dữ liệu một chiều, dễ debug
- **Hệ sinh thái lớn**: Cộng đồng đông đảo, nhiều thư viện hỗ trợ
- **React Native**: Có thể phát triển mobile app
- **SEO friendly**: Hỗ trợ Server-Side Rendering (Next.js)

---

### 2. Virtual DOM là gì? Hoạt động như thế nào?

**Trả lời:**

Virtual DOM là một bản sao nhẹ (lightweight copy) của Real DOM, được lưu trong bộ nhớ dưới dạng JavaScript object.

**Cách hoạt động (Reconciliation):**
1. Khi state/props thay đổi, React tạo một Virtual DOM mới
2. So sánh Virtual DOM mới với Virtual DOM cũ (Diffing Algorithm)
3. Tìm ra sự khác biệt nhỏ nhất (minimal changes)
4. Chỉ cập nhật những phần thay đổi lên Real DOM (Batch update)

```jsx
// React tự động tối ưu - chỉ update phần thay đổi
function Counter() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <h1>Title không đổi</h1> {/* Không re-render */}
      <p>{count}</p>             {/* Chỉ phần này được update */}
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}
```

---

### 3. JSX là gì?

**Trả lời:**

JSX (JavaScript XML) là một syntax extension cho JavaScript, cho phép viết HTML-like code trong JavaScript. JSX không phải HTML, nó được Babel compile thành `React.createElement()`.

```jsx
// JSX
const element = <h1 className="title">Hello, {name}</h1>;

// Sau khi compile bởi Babel
const element = React.createElement('h1', { className: 'title' }, `Hello, ${name}`);
```

**Lưu ý quan trọng:**
- Dùng `className` thay vì `class`
- Dùng `htmlFor` thay vì `for`
- Phải có một root element duy nhất (hoặc dùng `<Fragment>` / `<>`)
- Biểu thức JavaScript đặt trong `{}`
- Component phải viết hoa chữ cái đầu

---

### 4. Function Component vs Class Component

**Trả lời:**

| Tiêu chí | Function Component | Class Component |
|---|---|---|
| Cú pháp | Hàm JavaScript đơn giản | ES6 Class extends React.Component |
| State | Dùng `useState` hook | Dùng `this.state` |
| Lifecycle | Dùng `useEffect` hook | `componentDidMount`, `componentDidUpdate`... |
| Performance | Nhẹ hơn, không cần `this` | Nặng hơn do cơ chế class |
| Hiện tại | **Được khuyến khích dùng** | Legacy, ít dùng |

```jsx
// Function Component (khuyến khích)
function Welcome({ name }) {
  const [count, setCount] = useState(0);
  return <h1>Hello, {name}! Count: {count}</h1>;
}

// Class Component (legacy)
class Welcome extends React.Component {
  state = { count: 0 };
  render() {
    return <h1>Hello, {this.props.name}! Count: {this.state.count}</h1>;
  }
}
```

---

### 5. Props vs State

**Trả lời:**

| Tiêu chí | Props | State |
|---|---|---|
| Nguồn gốc | Truyền từ component cha | Quản lý nội bộ component |
| Thay đổi | **Immutable** (read-only) | **Mutable** (qua setState) |
| Mục đích | Giao tiếp giữa components | Lưu trữ dữ liệu thay đổi |
| Re-render | Khi props thay đổi từ cha | Khi gọi setState |

```jsx
// Props: truyền từ cha sang con
function Parent() {
  return <Child name="React" />;
}

function Child({ name }) {
  // State: quản lý nội bộ
  const [count, setCount] = useState(0);
  return <p>{name}: {count}</p>;
}
```

---

### 6. One-way Data Binding (Luồng dữ liệu một chiều) là gì?

**Trả lời:**

Trong React, dữ liệu chỉ chảy theo một chiều: từ **component cha → component con** thông qua props. Component con **không thể trực tiếp thay đổi** props của cha.

Nếu con muốn thay đổi dữ liệu ở cha, cha phải truyền một **callback function** xuống con.

```jsx
function Parent() {
  const [message, setMessage] = useState("Hello");

  return <Child message={message} onUpdate={setMessage} />;
}

function Child({ message, onUpdate }) {
  return (
    <div>
      <p>{message}</p>
      <button onClick={() => onUpdate("Updated!")}>Update</button>
    </div>
  );
}
```

---

### 7. React Element vs React Component

**Trả lời:**

- **React Element**: Là một plain object mô tả những gì bạn muốn hiển thị trên DOM. Nó immutable và nhẹ.
- **React Component**: Là một function hoặc class trả về React Element. Nó có thể có state và lifecycle.

```jsx
// React Element (object)
const element = <h1>Hello</h1>;
// Tương đương: React.createElement('h1', null, 'Hello')

// React Component (function trả về element)
function Greeting() {
  return <h1>Hello</h1>; // Trả về React Element
}
```

---

### 8. Key trong React là gì? Tại sao quan trọng?

**Trả lời:**

`key` là một prop đặc biệt giúp React **nhận diện** element nào thay đổi, thêm, hoặc xóa trong danh sách. Key giúp tối ưu quá trình reconciliation.

```jsx
// ❌ Sai: dùng index làm key (gây bug khi reorder/delete)
{items.map((item, index) => (
  <Item key={index} data={item} />
))}

// ✅ Đúng: dùng unique id
{items.map((item) => (
  <Item key={item.id} data={item} />
))}
```

**Tại sao không nên dùng index làm key?**
- Khi thêm/xóa/sắp xếp lại, index thay đổi → React hiểu sai element nào cần update
- Gây ra bug về state (input value bị lẫn) và performance kém

---

### 9. Controlled vs Uncontrolled Components

**Trả lời:**

- **Controlled**: Giá trị form được quản lý bởi React state. Mọi thay đổi đều qua `onChange` handler.
- **Uncontrolled**: Giá trị form được quản lý bởi DOM, truy cập qua `ref`.

```jsx
// Controlled Component
function ControlledForm() {
  const [value, setValue] = useState('');
  return <input value={value} onChange={(e) => setValue(e.target.value)} />;
}

// Uncontrolled Component
function UncontrolledForm() {
  const inputRef = useRef(null);
  const handleSubmit = () => {
    console.log(inputRef.current.value);
  };
  return <input ref={inputRef} defaultValue="" />;
}
```

**Khi nào dùng gì?**
- **Controlled**: Form phức tạp, cần validation real-time, conditional rendering
- **Uncontrolled**: Form đơn giản, tích hợp thư viện non-React, file input

---

### 10. Synthetic Event trong React là gì?

**Trả lời:**

Synthetic Event là wrapper cross-browser của React bao bọc native event. Nó có cùng interface như native event nhưng hoạt động nhất quán trên mọi trình duyệt.

```jsx
function HandleClick() {
  const handleClick = (e) => {
    // e là SyntheticEvent, không phải native event
    e.preventDefault();
    console.log(e.target);        // Hoạt động giống native
    console.log(e.nativeEvent);   // Truy cập native event gốc
  };

  return <a href="#" onClick={handleClick}>Click</a>;
}
```

---

## II. REACT HOOKS

### 1. useState

**Câu hỏi: useState hoạt động như thế nào? Giải thích batching.**

**Trả lời:**

`useState` là hook cơ bản nhất để quản lý state trong function component.

```jsx
const [state, setState] = useState(initialValue);
```

**Batching**: React 18+ tự động gộp (batch) nhiều setState thành một lần re-render duy nhất.

```jsx
function Example() {
  const [count, setCount] = useState(0);
  const [flag, setFlag] = useState(false);

  const handleClick = () => {
    // React 18: cả 2 setState chỉ gây 1 lần re-render (automatic batching)
    setCount(c => c + 1);
    setFlag(f => !f);
  };

  // Functional update: dùng khi state mới phụ thuộc state cũ
  const increment = () => {
    setCount(prev => prev + 1); // ✅ Đúng
    // setCount(count + 1);     // ❌ Có thể sai khi gọi nhiều lần
  };

  return <button onClick={handleClick}>{count}</button>;
}
```

**Lazy initialization:** Khi initialValue tốn chi phí tính toán:
```jsx
// ❌ Tính toán mỗi lần render
const [data, setData] = useState(expensiveComputation());

// ✅ Chỉ tính toán lần đầu
const [data, setData] = useState(() => expensiveComputation());
```

---

### 2. useEffect

**Câu hỏi: Giải thích useEffect, dependency array, và cleanup function.**

**Trả lời:**

`useEffect` dùng để thực hiện side effects: fetch data, subscriptions, DOM manipulation...

```jsx
useEffect(() => {
  // Side effect code

  return () => {
    // Cleanup function (optional)
    // Chạy khi component unmount hoặc trước khi effect chạy lại
  };
}, [dependencies]); // Dependency array
```

**3 trường hợp dependency array:**

```jsx
// 1. Không có dependency array → chạy sau MỖI lần render
useEffect(() => { console.log('Mỗi render'); });

// 2. Array rỗng → chạy MỘT LẦN sau mount (tương tự componentDidMount)
useEffect(() => { console.log('Chỉ lần đầu'); }, []);

// 3. Có dependencies → chạy khi dependency thay đổi
useEffect(() => { console.log('count thay đổi'); }, [count]);
```

**Ví dụ thực tế - Fetch data và cleanup:**

```jsx
function UserProfile({ userId }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    const controller = new AbortController();

    async function fetchUser() {
      try {
        const res = await fetch(`/api/users/${userId}`, {
          signal: controller.signal,
        });
        const data = await res.json();
        setUser(data);
      } catch (err) {
        if (err.name !== 'AbortError') console.error(err);
      }
    }

    fetchUser();

    // Cleanup: abort request cũ khi userId thay đổi hoặc unmount
    return () => controller.abort();
  }, [userId]);

  return user ? <div>{user.name}</div> : <p>Loading...</p>;
}
```

---

### 3. useContext

**Câu hỏi: useContext giải quyết vấn đề gì?**

**Trả lời:**

`useContext` giải quyết vấn đề **prop drilling** - truyền props qua nhiều tầng component trung gian.

```jsx
// Tạo Context
const ThemeContext = React.createContext('light');

// Provider: bọc ở component cha
function App() {
  const [theme, setTheme] = useState('dark');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Header />
    </ThemeContext.Provider>
  );
}

// Consumer: dùng ở bất kỳ component con nào
function DeepNestedButton() {
  const { theme, setTheme } = useContext(ThemeContext);
  return (
    <button
      style={{ background: theme === 'dark' ? '#333' : '#fff' }}
      onClick={() => setTheme(theme === 'dark' ? 'light' : 'dark')}
    >
      Toggle Theme
    </button>
  );
}
```

**Nhược điểm:** Mọi consumer đều re-render khi context value thay đổi, kể cả khi chỉ dùng một phần của value. Giải pháp: tách context nhỏ hoặc dùng `useMemo`.

---

### 4. useReducer

**Câu hỏi: Khi nào dùng useReducer thay vì useState?**

**Trả lời:**

Dùng `useReducer` khi:
- State phức tạp (object/array lồng nhau)
- Các state transitions liên quan đến nhau
- Logic cập nhật state phức tạp
- Cần tách logic ra khỏi component

```jsx
const initialState = { count: 0, step: 1 };

function reducer(state, action) {
  switch (action.type) {
    case 'increment':
      return { ...state, count: state.count + state.step };
    case 'decrement':
      return { ...state, count: state.count - state.step };
    case 'setStep':
      return { ...state, step: action.payload };
    case 'reset':
      return initialState;
    default:
      throw new Error(`Unknown action: ${action.type}`);
  }
}

function Counter() {
  const [state, dispatch] = useReducer(reducer, initialState);

  return (
    <div>
      <p>Count: {state.count} (step: {state.step})</p>
      <button onClick={() => dispatch({ type: 'increment' })}>+</button>
      <button onClick={() => dispatch({ type: 'decrement' })}>-</button>
      <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
    </div>
  );
}
```

---

### 5. useMemo vs useCallback

**Câu hỏi: Sự khác nhau giữa useMemo và useCallback? Khi nào nên dùng?**

**Trả lời:**

| | useMemo | useCallback |
|---|---|---|
| Cache | **Giá trị** tính toán | **Hàm** reference |
| Trả về | Kết quả của function | Chính function đó |
| Dùng khi | Tính toán nặng | Truyền callback cho child component |

```jsx
function ProductList({ products, filter }) {
  // useMemo: cache kết quả tính toán nặng
  const filteredProducts = useMemo(() => {
    return products.filter(p => p.category === filter);
  }, [products, filter]);

  // useCallback: cache function reference
  const handleDelete = useCallback((id) => {
    // delete logic
  }, []);

  return filteredProducts.map(p => (
    <ProductItem key={p.id} product={p} onDelete={handleDelete} />
  ));
}

// Child component dùng React.memo - chỉ re-render khi props thay đổi
const ProductItem = React.memo(({ product, onDelete }) => {
  return (
    <div>
      <span>{product.name}</span>
      <button onClick={() => onDelete(product.id)}>Delete</button>
    </div>
  );
});
```

**⚠️ Không lạm dụng:** Chỉ dùng khi thực sự cần thiết. Memoization cũng tốn memory và có overhead so sánh dependencies.

---

### 6. useRef

**Câu hỏi: useRef dùng để làm gì? Có những use case nào?**

**Trả lời:**

`useRef` tạo một object `{ current: value }` tồn tại xuyên suốt lifecycle, **thay đổi không gây re-render**.

**Use cases:**

```jsx
function Example() {
  // 1. Truy cập DOM element
  const inputRef = useRef(null);
  const focusInput = () => inputRef.current.focus();

  // 2. Lưu giá trị previous
  const prevCountRef = useRef();
  const [count, setCount] = useState(0);
  useEffect(() => {
    prevCountRef.current = count;
  });

  // 3. Lưu timer/interval ID
  const timerRef = useRef(null);
  useEffect(() => {
    timerRef.current = setInterval(() => {/* ... */}, 1000);
    return () => clearInterval(timerRef.current);
  }, []);

  // 4. Track mounted status
  const isMounted = useRef(true);
  useEffect(() => {
    return () => { isMounted.current = false; };
  }, []);

  return <input ref={inputRef} />;
}
```

---

### 7. useEffect vs useLayoutEffect

**Câu hỏi: Khi nào dùng useLayoutEffect thay vì useEffect?**

**Trả lời:**

| | useEffect | useLayoutEffect |
|---|---|---|
| Thời điểm | Sau khi paint (async) | Sau DOM update, trước paint (sync) |
| Blocking | Không block UI | Block UI cho đến khi hoàn thành |
| Use case | Fetch data, subscriptions | Đo DOM, prevent flicker |

```jsx
function Tooltip({ targetRef }) {
  const tooltipRef = useRef(null);
  const [position, setPosition] = useState({ top: 0, left: 0 });

  // useLayoutEffect: đo DOM trước khi paint → không bị flicker
  useLayoutEffect(() => {
    const rect = targetRef.current.getBoundingClientRect();
    setPosition({
      top: rect.bottom + 10,
      left: rect.left,
    });
  }, [targetRef]);

  return (
    <div ref={tooltipRef} style={{ position: 'absolute', ...position }}>
      Tooltip content
    </div>
  );
}
```

---

### 8. Custom Hooks

**Câu hỏi: Custom Hook là gì? Cho ví dụ thực tế.**

**Trả lời:**

Custom Hook là hàm JavaScript bắt đầu bằng `use`, cho phép tái sử dụng logic stateful giữa các component.

```jsx
// Custom hook: useLocalStorage
function useLocalStorage(key, initialValue) {
  const [value, setValue] = useState(() => {
    try {
      const item = localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch {
      return initialValue;
    }
  });

  useEffect(() => {
    localStorage.setItem(key, JSON.stringify(value));
  }, [key, value]);

  return [value, setValue];
}

// Custom hook: useFetch
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    const controller = new AbortController();
    setLoading(true);

    fetch(url, { signal: controller.signal })
      .then(res => res.json())
      .then(data => { setData(data); setLoading(false); })
      .catch(err => {
        if (err.name !== 'AbortError') {
          setError(err); setLoading(false);
        }
      });

    return () => controller.abort();
  }, [url]);

  return { data, loading, error };
}

// Custom hook: useDebounce
function useDebounce(value, delay = 300) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedValue(value), delay);
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// Sử dụng
function SearchComponent() {
  const [query, setQuery] = useState('');
  const debouncedQuery = useDebounce(query, 500);
  const { data, loading } = useFetch(`/api/search?q=${debouncedQuery}`);
  const [theme, setTheme] = useLocalStorage('theme', 'dark');

  return (/* ... */);
}
```

---

### 9. Rules of Hooks

**Câu hỏi: Có những quy tắc nào khi sử dụng Hooks?**

**Trả lời:**

1. **Chỉ gọi Hooks ở top level** - Không gọi trong vòng lặp, điều kiện, hay hàm lồng nhau
2. **Chỉ gọi Hooks trong React function component hoặc custom hooks**

```jsx
// ❌ SAI: gọi hook trong điều kiện
function Bad({ isLoggedIn }) {
  if (isLoggedIn) {
    const [user, setUser] = useState(null); // ❌
  }
}

// ✅ ĐÚNG: điều kiện bên trong hook
function Good({ isLoggedIn }) {
  const [user, setUser] = useState(null);

  useEffect(() => {
    if (isLoggedIn) {
      fetchUser().then(setUser);
    }
  }, [isLoggedIn]);
}
```

---

## III. STATE MANAGEMENT

### 1. Context API vs Redux

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

### 2. Redux Toolkit (RTK)

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

### 3. Redux Middleware

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

## IV. REACT ROUTER

### 1. React Router cơ bản

**Câu hỏi: Giải thích cách sử dụng React Router v6.**

**Trả lời:**

```jsx
import { BrowserRouter, Routes, Route, Navigate, Outlet } from 'react-router-dom';

function App() {
  return (
    <BrowserRouter>
      <Routes>
        <Route path="/" element={<Layout />}>
          <Route index element={<Home />} />
          <Route path="products" element={<Products />} />
          <Route path="products/:id" element={<ProductDetail />} />

          {/* Protected Routes */}
          <Route element={<ProtectedRoute />}>
            <Route path="dashboard" element={<Dashboard />} />
            <Route path="profile" element={<Profile />} />
          </Route>

          {/* 404 */}
          <Route path="*" element={<NotFound />} />
        </Route>
      </Routes>
    </BrowserRouter>
  );
}

// Protected Route component
function ProtectedRoute() {
  const { isAuthenticated } = useAuth();
  return isAuthenticated ? <Outlet /> : <Navigate to="/login" replace />;
}

// Sử dụng hooks
function ProductDetail() {
  const { id } = useParams();
  const navigate = useNavigate();
  const [searchParams, setSearchParams] = useSearchParams();
  const tab = searchParams.get('tab') || 'info';

  return (
    <div>
      <h1>Product {id}</h1>
      <button onClick={() => navigate(-1)}>Go Back</button>
      <button onClick={() => navigate('/products', { replace: true })}>
        All Products
      </button>
    </div>
  );
}

// Lazy loading routes
const Dashboard = React.lazy(() => import('./pages/Dashboard'));

function App() {
  return (
    <Suspense fallback={<Loading />}>
      <Routes>
        <Route path="/dashboard" element={<Dashboard />} />
      </Routes>
    </Suspense>
  );
}
```

---

## V. PERFORMANCE OPTIMIZATION

### 1. React.memo

**Câu hỏi: React.memo hoạt động thế nào?**

**Trả lời:**

`React.memo` là HOC giúp component chỉ re-render khi props thay đổi (shallow comparison).

```jsx
const ExpensiveList = React.memo(function ExpensiveList({ items, onItemClick }) {
  console.log('ExpensiveList rendered');
  return (
    <ul>
      {items.map(item => (
        <li key={item.id} onClick={() => onItemClick(item.id)}>
          {item.name}
        </li>
      ))}
    </ul>
  );
});

// Custom comparison function
const UserCard = React.memo(
  function UserCard({ user, config }) {
    return <div>{user.name}</div>;
  },
  (prevProps, nextProps) => {
    // Return true nếu KHÔNG cần re-render
    return prevProps.user.id === nextProps.user.id
        && prevProps.user.name === nextProps.user.name;
  }
);
```

---

### 2. Code Splitting & Lazy Loading

**Câu hỏi: Cách tối ưu bundle size với code splitting?**

**Trả lời:**

```jsx
import React, { Suspense, lazy } from 'react';

// Lazy load components
const AdminPanel = lazy(() => import('./AdminPanel'));
const Analytics = lazy(() => import('./Analytics'));

function App() {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      <Routes>
        <Route path="/admin" element={<AdminPanel />} />
        <Route path="/analytics" element={<Analytics />} />
      </Routes>
    </Suspense>
  );
}

// Named exports
const Dashboard = lazy(() =>
  import('./Dashboard').then(module => ({ default: module.Dashboard }))
);
```

---

### 3. Tối ưu danh sách lớn (Virtualization)

**Câu hỏi: Cách render danh sách hàng nghìn item mà không lag?**

**Trả lời:**

Dùng **Virtualization** - chỉ render các item hiện đang visible trên viewport.

```jsx
import { FixedSizeList } from 'react-window';

function VirtualizedList({ items }) {
  const Row = ({ index, style }) => (
    <div style={style}>
      {items[index].name}
    </div>
  );

  return (
    <FixedSizeList
      height={600}           // Chiều cao container
      width="100%"
      itemCount={items.length}
      itemSize={50}           // Chiều cao mỗi item
    >
      {Row}
    </FixedSizeList>
  );
}
```

---

## VI. ADVANCED CONCEPTS

### 1. Higher-Order Component (HOC)

**Câu hỏi: HOC là gì? Cho ví dụ.**

**Trả lời:**

HOC là một **function nhận vào component và trả về component mới** với thêm props/logic.

```jsx
// HOC: thêm loading state
function withLoading(WrappedComponent) {
  return function WithLoadingComponent({ isLoading, ...props }) {
    if (isLoading) return <div>Loading...</div>;
    return <WrappedComponent {...props} />;
  };
}

// HOC: thêm authentication check
function withAuth(WrappedComponent) {
  return function WithAuthComponent(props) {
    const { isAuthenticated } = useAuth();
    if (!isAuthenticated) return <Navigate to="/login" />;
    return <WrappedComponent {...props} />;
  };
}

const ProtectedDashboard = withAuth(Dashboard);
const LoadableUserList = withLoading(UserList);
```

**Lưu ý:** Trong React hiện đại, Custom Hooks thường được ưu tiên hơn HOC vì đơn giản và dễ đọc hơn.

---

### 2. Error Boundaries

**Câu hỏi: Error Boundary là gì?**

**Trả lời:**

Error Boundary là component bắt JavaScript error trong cây component con, log error, và hiển thị fallback UI thay vì crash toàn app.

```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    // Log error to monitoring service
    logErrorToService(error, errorInfo);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div>
          <h2>Something went wrong!</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try Again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}

// Sử dụng
<ErrorBoundary>
  <UserProfile />
</ErrorBoundary>
```

**Lưu ý:** Error Boundary chỉ bắt error trong rendering, lifecycle, constructor. **Không bắt:** event handlers, async code, SSR.

---

### 3. React Portals

**Câu hỏi: Portal dùng để làm gì?**

**Trả lời:**

Portal cho phép render children vào một DOM node nằm **ngoài** parent DOM hierarchy, thường dùng cho modal, tooltip, dropdown.

```jsx
import { createPortal } from 'react-dom';

function Modal({ isOpen, onClose, children }) {
  if (!isOpen) return null;

  return createPortal(
    <div className="modal-overlay" onClick={onClose}>
      <div className="modal-content" onClick={(e) => e.stopPropagation()}>
        {children}
        <button onClick={onClose}>Close</button>
      </div>
    </div>,
    document.getElementById('modal-root') // Render ngoài #root
  );
}
```

---

### 4. forwardRef

**Câu hỏi: forwardRef dùng khi nào?**

**Trả lời:**

`forwardRef` cho phép truyền `ref` từ component cha xuống DOM element bên trong component con.

```jsx
const CustomInput = React.forwardRef(function CustomInput({ label, ...props }, ref) {
  return (
    <label>
      {label}
      <input ref={ref} {...props} />
    </label>
  );
});

function Form() {
  const inputRef = useRef(null);
  return (
    <>
      <CustomInput ref={inputRef} label="Name" />
      <button onClick={() => inputRef.current.focus()}>Focus Input</button>
    </>
  );
}
```

---

### 5. Server-Side Rendering (SSR) vs Client-Side Rendering (CSR)

**Câu hỏi: So sánh SSR và CSR. Khi nào dùng SSR?**

**Trả lời:**

| | CSR | SSR |
|---|---|---|
| Render | Trên browser | Trên server → gửi HTML |
| First Paint | Chậm (tải JS trước) | Nhanh (HTML có sẵn) |
| SEO | Kém | Tốt |
| Interactivity | Nhanh sau load | Cần hydration |
| Server load | Thấp | Cao |
| Framework | Create React App | **Next.js**, Remix |

**Dùng SSR khi:** SEO quan trọng (blog, e-commerce), first load performance critical, social media sharing (meta tags).

**Dùng CSR khi:** Dashboard, admin panel, ứng dụng nội bộ không cần SEO.

---

## VII. TESTING

### 1. React Testing Library

**Câu hỏi: Cách viết test cho React component?**

**Trả lời:**

```jsx
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';

// Test render cơ bản
test('renders greeting message', () => {
  render(<Greeting name="React" />);
  expect(screen.getByText('Hello, React!')).toBeInTheDocument();
});

// Test user interaction
test('increments counter on click', async () => {
  const user = userEvent.setup();
  render(<Counter />);

  const button = screen.getByRole('button', { name: /increment/i });
  await user.click(button);

  expect(screen.getByText('Count: 1')).toBeInTheDocument();
});

// Test async operations
test('loads and displays user data', async () => {
  // Mock fetch
  global.fetch = jest.fn(() =>
    Promise.resolve({
      json: () => Promise.resolve({ name: 'John' }),
    })
  );

  render(<UserProfile userId="1" />);

  expect(screen.getByText('Loading...')).toBeInTheDocument();

  await waitFor(() => {
    expect(screen.getByText('John')).toBeInTheDocument();
  });
});

// Test form submission
test('submits form with correct data', async () => {
  const handleSubmit = jest.fn();
  const user = userEvent.setup();

  render(<LoginForm onSubmit={handleSubmit} />);

  await user.type(screen.getByLabelText(/email/i), 'test@example.com');
  await user.type(screen.getByLabelText(/password/i), 'password123');
  await user.click(screen.getByRole('button', { name: /login/i }));

  expect(handleSubmit).toHaveBeenCalledWith({
    email: 'test@example.com',
    password: 'password123',
  });
});
```

---

## VIII. CÂU HỎI TÌNH HUỐNG THỰC TẾ

### 1. Cách xử lý form phức tạp?

**Trả lời:**

Dùng thư viện **React Hook Form** kết hợp **Zod/Yup** cho validation:

```jsx
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { z } from 'zod';

const schema = z.object({
  email: z.string().email('Email không hợp lệ'),
  password: z.string().min(8, 'Ít nhất 8 ký tự'),
  confirmPassword: z.string(),
}).refine(data => data.password === data.confirmPassword, {
  message: 'Mật khẩu không khớp',
  path: ['confirmPassword'],
});

function RegisterForm() {
  const { register, handleSubmit, formState: { errors, isSubmitting } } = useForm({
    resolver: zodResolver(schema),
  });

  const onSubmit = async (data) => {
    await api.register(data);
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('email')} placeholder="Email" />
      {errors.email && <span>{errors.email.message}</span>}

      <input {...register('password')} type="password" />
      {errors.password && <span>{errors.password.message}</span>}

      <input {...register('confirmPassword')} type="password" />
      {errors.confirmPassword && <span>{errors.confirmPassword.message}</span>}

      <button disabled={isSubmitting}>
        {isSubmitting ? 'Đang xử lý...' : 'Đăng ký'}
      </button>
    </form>
  );
}
```

---

### 2. Cách xử lý Authentication?

**Trả lời:**

```jsx
// AuthContext.jsx
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // Check token on mount
    const token = localStorage.getItem('token');
    if (token) {
      api.getMe(token)
        .then(setUser)
        .catch(() => localStorage.removeItem('token'))
        .finally(() => setLoading(false));
    } else {
      setLoading(false);
    }
  }, []);

  const login = async (email, password) => {
    const { token, user } = await api.login(email, password);
    localStorage.setItem('token', token);
    setUser(user);
  };

  const logout = () => {
    localStorage.removeItem('token');
    setUser(null);
  };

  if (loading) return <FullPageSpinner />;

  return (
    <AuthContext.Provider value={{ user, login, logout, isAuthenticated: !!user }}>
      {children}
    </AuthContext.Provider>
  );
}

// Custom hook
function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

---

### 3. Cách thiết kế component tái sử dụng?

**Trả lời:**

Áp dụng **Compound Components Pattern**:

```jsx
// Button component linh hoạt
function Button({ variant = 'primary', size = 'md', isLoading, children, ...props }) {
  return (
    <button
      className={`btn btn-${variant} btn-${size}`}
      disabled={isLoading || props.disabled}
      {...props}
    >
      {isLoading ? <Spinner size="sm" /> : children}
    </button>
  );
}

// Compound Component: Dropdown
const DropdownContext = createContext();

function Dropdown({ children }) {
  const [isOpen, setIsOpen] = useState(false);
  return (
    <DropdownContext.Provider value={{ isOpen, toggle: () => setIsOpen(!isOpen) }}>
      <div className="dropdown">{children}</div>
    </DropdownContext.Provider>
  );
}

Dropdown.Trigger = function Trigger({ children }) {
  const { toggle } = useContext(DropdownContext);
  return <button onClick={toggle}>{children}</button>;
};

Dropdown.Menu = function Menu({ children }) {
  const { isOpen } = useContext(DropdownContext);
  if (!isOpen) return null;
  return <ul className="dropdown-menu">{children}</ul>;
};

Dropdown.Item = function Item({ children, onClick }) {
  return <li onClick={onClick}>{children}</li>;
};

// Sử dụng
<Dropdown>
  <Dropdown.Trigger>Options</Dropdown.Trigger>
  <Dropdown.Menu>
    <Dropdown.Item onClick={handleEdit}>Edit</Dropdown.Item>
    <Dropdown.Item onClick={handleDelete}>Delete</Dropdown.Item>
  </Dropdown.Menu>
</Dropdown>
```

---

### 4. React 18+ có gì mới?

**Trả lời:**

- **Automatic Batching**: Gộp nhiều setState trong mọi context (setTimeout, promises...)
- **Transitions** (`useTransition`, `startTransition`): Đánh dấu update không urgent
- **Suspense for data fetching**: Kết hợp với lazy loading
- **Concurrent Rendering**: Render có thể bị interrupt
- **useId**: Tạo unique ID cho SSR hydration
- **useSyncExternalStore**: Subscribe external store an toàn

```jsx
function SearchResults() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    // Urgent: update input ngay
    setQuery(e.target.value);

    // Non-urgent: update kết quả search có thể chậm
    startTransition(() => {
      setSearchResults(filterResults(e.target.value));
    });
  };

  return (
    <div>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <Results />}
    </div>
  );
}
```

---

### 5. So sánh Next.js App Router vs Pages Router

**Trả lời:**

| | Pages Router | App Router (Next.js 13+) |
|---|---|---|
| Folder | `pages/` | `app/` |
| Components | Client by default | **Server by default** |
| Data fetching | `getServerSideProps`, `getStaticProps` | `async` component, `fetch()` |
| Layout | `_app.js`, `_document.js` | `layout.tsx` (nested) |
| Loading | Custom | `loading.tsx` built-in |
| Error | Custom | `error.tsx` built-in |

---

## IX. TIPS PHỎNG VẤN

1. **Trả lời có cấu trúc**: Định nghĩa → Cách hoạt động → Ví dụ → Khi nào dùng
2. **Nêu trade-offs**: Không có giải pháp hoàn hảo, show bạn hiểu ưu/nhược điểm
3. **Liên hệ thực tế**: Đề cập kinh nghiệm thực tế khi có thể
4. **Hỏi lại nếu cần**: Không ngại hỏi để hiểu rõ câu hỏi
5. **Code sạch**: Khi live coding, đặt tên biến rõ ràng, giải thích từng bước
