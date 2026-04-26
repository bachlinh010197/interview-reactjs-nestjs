# 📘 ReactJS - Phần 2: React Hooks

[⬅️ Fundamentals](./01-fundamentals.md) | [Phần tiếp: State Management ➡️](./03-state-management.md)

---

## 1. useState

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

## 2. useEffect

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

## 3. useContext

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

## 4. useReducer

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

## 5. useMemo vs useCallback

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

## 6. useRef

**Câu hỏi: useRef dùng để làm gì? Có những use case nào?**

**Trả lời:**

`useRef` tạo một object `{ current: value }` tồn tại xuyên suốt lifecycle, **thay đổi không gây re-render**.

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

## 7. useEffect vs useLayoutEffect

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

## 8. Custom Hooks

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

## 9. Rules of Hooks

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

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 10. useImperativeHandle — khi nào cần dùng?

**Trả lời:**

`useImperativeHandle` cho phép customize instance value được expose khi parent dùng `ref`. Dùng khi bạn muốn **giới hạn** những gì parent có thể truy cập.

```jsx
const VideoPlayer = React.forwardRef(function VideoPlayer({ src }, ref) {
  const videoRef = useRef(null);
  const [isPlaying, setIsPlaying] = useState(false);

  // Chỉ expose play/pause, KHÔNG expose toàn bộ DOM element
  useImperativeHandle(ref, () => ({
    play() {
      videoRef.current.play();
      setIsPlaying(true);
    },
    pause() {
      videoRef.current.pause();
      setIsPlaying(false);
    },
    get isPlaying() {
      return isPlaying;
    },
  }), [isPlaying]);

  return <video ref={videoRef} src={src} />;
});

// Parent chỉ truy cập được play(), pause(), isPlaying
function App() {
  const playerRef = useRef(null);
  return (
    <>
      <VideoPlayer ref={playerRef} src="/video.mp4" />
      <button onClick={() => playerRef.current.play()}>Play</button>
      <button onClick={() => playerRef.current.pause()}>Pause</button>
    </>
  );
}
```

---

### 11. useSyncExternalStore — Subscribe external store an toàn

**Trả lời:**

`useSyncExternalStore` (React 18) giải quyết **tearing** — hiện tượng UI hiển thị 2 giá trị khác nhau từ cùng 1 store trong concurrent rendering.

```jsx
// Dùng cho external stores (không phải React state)
function useWindowWidth() {
  return useSyncExternalStore(
    // subscribe: đăng ký listener
    (callback) => {
      window.addEventListener('resize', callback);
      return () => window.removeEventListener('resize', callback);
    },
    // getSnapshot: lấy giá trị hiện tại (client)
    () => window.innerWidth,
    // getServerSnapshot: giá trị cho SSR (optional)
    () => 1024,
  );
}

// Dùng với custom store (Zustand-like)
function createStore(initialState) {
  let state = initialState;
  const listeners = new Set();

  return {
    getState: () => state,
    setState: (newState) => {
      state = typeof newState === 'function' ? newState(state) : newState;
      listeners.forEach(l => l());
    },
    subscribe: (listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);
    },
  };
}

const store = createStore({ count: 0 });

function Counter() {
  const count = useSyncExternalStore(
    store.subscribe,
    () => store.getState().count,
  );
  return <button onClick={() => store.setState(s => ({ count: s.count + 1 }))}>{count}</button>;
}
```

---

### 12. useEffect dependency hell — Cách xử lý dependencies phức tạp

**Trả lời:**

```jsx
// ❌ Vấn đề: object/function trong dependencies → infinite loop
function SearchPage({ filters }) {
  const [results, setResults] = useState([]);

  // filters là object → reference mới mỗi render → effect chạy liên tục!
  useEffect(() => {
    fetchResults(filters).then(setResults);
  }, [filters]); // ❌ infinite loop nếu parent re-render

  // ✅ FIX 1: Destructure primitive values
  const { query, category, page } = filters;
  useEffect(() => {
    fetchResults({ query, category, page }).then(setResults);
  }, [query, category, page]); // primitives → ổn định

  // ✅ FIX 2: Serialize object
  const filtersKey = JSON.stringify(filters);
  useEffect(() => {
    fetchResults(JSON.parse(filtersKey)).then(setResults);
  }, [filtersKey]);

  // ✅ FIX 3: useRef cho event handlers
  const onFetchRef = useRef(onFetch);
  onFetchRef.current = onFetch;
  useEffect(() => {
    onFetchRef.current(filters);
  }, [query]); // Không cần thêm onFetch vào deps
}
```

---

### 13. Tại sao KHÔNG nên dùng useEffect cho data fetching? (Senior)

**Trả lời:**

Đây là quan điểm từ React team (React 19+). `useEffect` cho data fetching có nhiều vấn đề:

**Vấn đề:**
- **Waterfall**: Parent fetch xong → child mới bắt đầu fetch
- **No caching**: Mỗi lần mount lại fetch mới
- **Race conditions**: Cần tự handle AbortController
- **No SSR support**: Chỉ chạy trên client
- **No preloading**: Không thể prefetch

**Giải pháp tốt hơn:**
```jsx
// ✅ Dùng React Query / TanStack Query
function UserProfile({ userId }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetchUser(userId),
    staleTime: 5 * 60 * 1000,  // Cache 5 phút
    retry: 2,
  });

  if (isLoading) return <Spinner />;
  if (error) return <Error message={error.message} />;
  return <div>{data.name}</div>;
}

// ✅ Dùng React Router loader (data trước khi render)
const router = createBrowserRouter([
  {
    path: '/users/:id',
    loader: ({ params }) => fetchUser(params.id),
    element: <UserProfile />,
  },
]);

function UserProfile() {
  const user = useLoaderData(); // Data đã sẵn sàng!
  return <div>{user.name}</div>;
}

// ✅ Next.js Server Components
async function UserProfile({ userId }) {
  const user = await fetchUser(userId); // Fetch trên server
  return <div>{user.name}</div>;
}
```
