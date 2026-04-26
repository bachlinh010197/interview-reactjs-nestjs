# 📘 ReactJS - Phần 8: Câu Hỏi Tình Huống & Tips

[⬅️ Testing](./07-testing.md) | [🏠 Mục lục](../README.md)

---

## 1. Cách xử lý form phức tạp?

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

## 2. Cách xử lý Authentication?

**Trả lời:**

```jsx
const AuthContext = createContext(null);

function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
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

function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

---

## 3. Cách thiết kế component tái sử dụng?

**Trả lời:**

Áp dụng **Compound Components Pattern**:

```jsx
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

## 4. React 18+ có gì mới?

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
    setQuery(e.target.value);

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

## 5. So sánh Next.js App Router vs Pages Router

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

## 🎯 TIPS PHỎNG VẤN

1. **Trả lời có cấu trúc**: Định nghĩa → Cách hoạt động → Ví dụ → Khi nào dùng
2. **Nêu trade-offs**: Không có giải pháp hoàn hảo, show bạn hiểu ưu/nhược điểm
3. **Liên hệ thực tế**: Đề cập kinh nghiệm thực tế khi có thể
4. **Hỏi lại nếu cần**: Không ngại hỏi để hiểu rõ câu hỏi
5. **Code sạch**: Khi live coding, đặt tên biến rõ ràng, giải thích từng bước

---

## 🔥 CÂU HỎI TÌNH HUỐNG LEVEL SENIOR

### 6. Thiết kế hệ thống Notification Real-time

**Câu hỏi:** Thiết kế UI notification real-time (như Facebook/Slack) trong React.

**Trả lời:**

```jsx
// 1. Custom hook quản lý WebSocket connection
function useNotifications() {
  const [notifications, setNotifications] = useState([]);
  const [unreadCount, setUnreadCount] = useState(0);
  const wsRef = useRef(null);

  useEffect(() => {
    const ws = new WebSocket('wss://api.example.com/ws/notifications');
    wsRef.current = ws;

    ws.onmessage = (event) => {
      const notification = JSON.parse(event.data);
      setNotifications(prev => [notification, ...prev]);
      setUnreadCount(c => c + 1);

      // Browser notification
      if (Notification.permission === 'granted') {
        new Notification(notification.title, { body: notification.message });
      }
    };

    ws.onclose = () => {
      // Reconnect logic với exponential backoff
      setTimeout(() => {
        wsRef.current = new WebSocket('wss://api.example.com/ws/notifications');
      }, 3000);
    };

    return () => ws.close();
  }, []);

  const markAsRead = useCallback((id) => {
    setNotifications(prev =>
      prev.map(n => n.id === id ? { ...n, read: true } : n)
    );
    setUnreadCount(c => Math.max(0, c - 1));
    api.markNotificationRead(id); // Fire-and-forget
  }, []);

  const markAllAsRead = useCallback(() => {
    setNotifications(prev => prev.map(n => ({ ...n, read: true })));
    setUnreadCount(0);
    api.markAllNotificationsRead();
  }, []);

  return { notifications, unreadCount, markAsRead, markAllAsRead };
}

// 2. NotificationProvider (global state)
const NotificationContext = createContext();

function NotificationProvider({ children }) {
  const notificationState = useNotifications();
  return (
    <NotificationContext.Provider value={notificationState}>
      {children}
    </NotificationContext.Provider>
  );
}
```

---

### 7. Micro-Frontend Architecture

**Câu hỏi:** Cách triển khai Micro-Frontend với React?

**Trả lời:**

**3 Approaches phổ biến:**

| Approach | Pros | Cons |
|---|---|---|
| **Module Federation** (Webpack 5) | Runtime loading, share deps | Webpack-specific |
| **Single-SPA** | Framework agnostic | Complex setup |
| **iframe** | Hoàn toàn isolated | Performance, UX kém |

```jsx
// Module Federation — Webpack 5
// Host app webpack config
new ModuleFederationPlugin({
  name: 'host',
  remotes: {
    productApp: 'productApp@https://products.example.com/remoteEntry.js',
    cartApp: 'cartApp@https://cart.example.com/remoteEntry.js',
  },
  shared: ['react', 'react-dom'],
});

// Host app component
const ProductList = React.lazy(() => import('productApp/ProductList'));
const CartWidget = React.lazy(() => import('cartApp/CartWidget'));

function App() {
  return (
    <Suspense fallback={<Spinner />}>
      <Header>
        <CartWidget />
      </Header>
      <ProductList />
    </Suspense>
  );
}
```

---

### 8. Cách xử lý Memory Leak trong React

**Câu hỏi:** Memory leak thường xảy ra ở đâu trong React? Cách phòng tránh?

**Trả lời:**

```jsx
// ❌ Leak 1: setState sau khi unmount
useEffect(() => {
  fetchData().then(data => {
    setData(data); // Component có thể đã unmount!
  });
}, []);

// ✅ Fix: AbortController
useEffect(() => {
  const controller = new AbortController();
  fetchData({ signal: controller.signal })
    .then(setData)
    .catch(err => { if (err.name !== 'AbortError') throw err; });
  return () => controller.abort();
}, []);

// ❌ Leak 2: Event listener không cleanup
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
  // Quên return cleanup!
}, []);

// ✅ Fix
useEffect(() => {
  window.addEventListener('scroll', handleScroll);
  return () => window.removeEventListener('scroll', handleScroll);
}, []);

// ❌ Leak 3: setInterval không clear
useEffect(() => {
  setInterval(pollData, 5000); // Không bao giờ clear!
}, []);

// ✅ Fix
useEffect(() => {
  const id = setInterval(pollData, 5000);
  return () => clearInterval(id);
}, []);

// ❌ Leak 4: Closure giữ reference lớn
useEffect(() => {
  const hugeData = generateHugeData(); // 100MB
  const handler = () => console.log(hugeData.length);
  window.addEventListener('click', handler);
  // hugeData bị giữ trong closure của handler mãi mãi!
  return () => window.removeEventListener('click', handler);
}, []);
```
