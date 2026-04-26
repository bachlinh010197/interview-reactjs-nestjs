# 📘 ReactJS - Phần 6: Advanced Concepts

[⬅️ Performance](./05-performance.md) | [Phần tiếp: Testing ➡️](./07-testing.md)

---

## 1. Higher-Order Component (HOC)

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

## 2. Error Boundaries

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

## 3. React Portals

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
    document.getElementById('modal-root')
  );
}
```

---

## 4. forwardRef

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

## 5. Server-Side Rendering (SSR) vs Client-Side Rendering (CSR)

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

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 6. React Server Components (RSC) — Paradigm mới

**Câu hỏi: React Server Components là gì? Khác gì SSR?**

**Trả lời:**

| | SSR | React Server Components |
|---|---|---|
| Render | HTML trên server → hydrate trên client | Component chạy **chỉ trên server**, không gửi JS |
| JS Bundle | Toàn bộ component code gửi xuống client | **Zero JS** cho server components |
| Interactivity | Cần hydration → interactive | Không interactive (dùng client component cho tương tác) |
| Data fetching | `getServerSideProps` (Next.js) | `async/await` trực tiếp trong component |
| Re-render | Toàn page | Có thể refetch riêng server component |

```jsx
// Server Component — chạy CHỈ trên server, KHÔNG có trong JS bundle
// Có thể dùng async/await, truy cập DB trực tiếp
async function ProductPage({ productId }) {
  // Truy cập DB trực tiếp — KHÔNG cần API endpoint!
  const product = await db.product.findUnique({ where: { id: productId } });
  const reviews = await db.review.findMany({ where: { productId } });

  return (
    <div>
      <h1>{product.name}</h1>
      <p>{product.description}</p>
      {/* Client Component cho phần interactive */}
      <AddToCartButton productId={product.id} price={product.price} />
      <ReviewList reviews={reviews} />
    </div>
  );
}

// Client Component — có 'use client' directive
'use client';
function AddToCartButton({ productId, price }) {
  const [adding, setAdding] = useState(false);

  const handleAdd = async () => {
    setAdding(true);
    await addToCart(productId);
    setAdding(false);
  };

  return (
    <button onClick={handleAdd} disabled={adding}>
      {adding ? 'Adding...' : `Add to Cart - $${price}`}
    </button>
  );
}
```

**Quy tắc quan trọng:**
- Server Component **KHÔNG thể** dùng useState, useEffect, event handlers
- Client Component **CÓ THỂ** import Server Component (nhưng không ngược lại — chỉ qua children)
- Mặc định tất cả component là Server Component (Next.js App Router)

---

### 7. useOptimistic (React 19) — Optimistic UI Updates

**Câu hỏi: Optimistic update là gì? Cách implement?**

**Trả lời:**

Optimistic update là kỹ thuật **update UI ngay lập tức** trước khi server xác nhận, rollback nếu fail.

```jsx
// React 19: useOptimistic hook
function TodoList({ todos, addTodoAction }) {
  const [optimisticTodos, addOptimisticTodo] = useOptimistic(
    todos,
    (currentTodos, newTodo) => [
      ...currentTodos,
      { ...newTodo, id: 'temp-' + Date.now(), pending: true },
    ],
  );

  async function handleAdd(formData) {
    const title = formData.get('title');
    addOptimisticTodo({ title }); // UI update ngay!
    await addTodoAction(title);   // Server call → khi xong, todos prop update → optimistic state reset
  }

  return (
    <div>
      <form action={handleAdd}>
        <input name="title" />
        <button type="submit">Add</button>
      </form>
      {optimisticTodos.map(todo => (
        <div key={todo.id} style={{ opacity: todo.pending ? 0.5 : 1 }}>
          {todo.title} {todo.pending && '(saving...)'}
        </div>
      ))}
    </div>
  );
}

// Trước React 19: implement thủ công
function useOptimisticMutation(mutationFn) {
  const [optimisticData, setOptimisticData] = useState(null);
  const [error, setError] = useState(null);

  const mutate = async (data, { optimisticUpdate, rollback }) => {
    setOptimisticData(optimisticUpdate);
    try {
      const result = await mutationFn(data);
      setOptimisticData(null);
      return result;
    } catch (err) {
      setOptimisticData(null);
      rollback?.();
      setError(err);
    }
  };

  return { mutate, optimisticData, error };
}
```

---

### 8. Design Patterns nâng cao: Render Props vs Hooks vs HOC

**Câu hỏi: So sánh 3 patterns chia sẻ logic. Khi nào dùng pattern nào?**

**Trả lời:**

```jsx
// 1. Custom Hook (ưu tiên — React hiện đại)
function useMousePosition() {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  return pos;
}

function App() {
  const { x, y } = useMousePosition();
  return <p>Mouse: {x}, {y}</p>;
}

// 2. Render Props (khi cần kiểm soát RENDER output)
function MouseTracker({ render }) {
  const { x, y } = useMousePosition();
  return render({ x, y });
}

<MouseTracker render={({ x, y }) => <Crosshair x={x} y={y} />} />

// 3. HOC (legacy — wrapper hell, khó debug)
function withMousePosition(Component) {
  return function(props) {
    const pos = useMousePosition();
    return <Component {...props} mousePos={pos} />;
  };
}
```

| Pattern | Ưu điểm | Nhược điểm | Dùng khi |
|---|---|---|---|
| **Custom Hook** | Đơn giản, composable, type-safe | Không kiểm soát render | **Mặc định chọn** |
| **Render Props** | Kiểm soát render | Callback hell nếu lồng nhiều | Cần flexible rendering |
| **HOC** | Wrap component sẵn | Wrapper hell, props collision | Legacy code, 3rd party |

---

### 9. Hydration Mismatch — nguyên nhân và cách fix

**Câu hỏi: Hydration mismatch là gì? Làm sao debug và fix?**

**Trả lời:**

Hydration mismatch xảy ra khi HTML server render **khác** với output client render lần đầu.

```jsx
// ❌ Gây mismatch: giá trị khác nhau giữa server và client
function Timestamp() {
  return <p>{new Date().toISOString()}</p>; // Server: 10:00:00, Client: 10:00:01
}

// ❌ Gây mismatch: window/document không tồn tại trên server
function WindowSize() {
  return <p>{window.innerWidth}</p>; // Server: error!
}

// ✅ FIX: Dùng useEffect + state (chỉ chạy trên client)
function Timestamp() {
  const [time, setTime] = useState(null); // Server: null, Client initial: null → match!
  useEffect(() => {
    setTime(new Date().toISOString()); // Update sau hydration
  }, []);
  return <p>{time ?? 'Loading...'}</p>;
}

// ✅ FIX: suppressHydrationWarning cho nội dung dynamic
<time suppressHydrationWarning>{new Date().toISOString()}</time>

// ✅ FIX: Custom hook useIsClient
function useIsClient() {
  const [isClient, setIsClient] = useState(false);
  useEffect(() => { setIsClient(true); }, []);
  return isClient;
}

function DynamicContent() {
  const isClient = useIsClient();
  if (!isClient) return <Skeleton />;
  return <p>Window: {window.innerWidth}px</p>;
}
```
