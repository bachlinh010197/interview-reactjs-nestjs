# 📘 ReactJS - Phần 5: Performance Optimization

[⬅️ Router](./04-router.md) | [Phần tiếp: Advanced ➡️](./06-advanced.md)

---

## 1. React.memo

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

## 2. Code Splitting & Lazy Loading

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

## 3. Tối ưu danh sách lớn (Virtualization)

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

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 4. Profiling & Debugging Re-renders

**Câu hỏi: Cách tìm và fix unnecessary re-renders?**

**Trả lời:**

```jsx
// 1. React DevTools Profiler: Record → xem component nào render, tại sao

// 2. React.memo + why-did-you-render (development)
// npm install @welldone-software/why-did-you-render
import whyDidYouRender from '@welldone-software/why-did-you-render';
whyDidYouRender(React, { trackAllPureComponents: true });

// 3. Custom hook để detect re-renders
function useRenderCount(componentName) {
  const count = useRef(0);
  count.current++;
  useEffect(() => {
    console.log(`${componentName} rendered ${count.current} times`);
  });
}

// 4. Patterns tránh re-render
// ❌ Inline object → reference mới mỗi render
<Child style={{ color: 'red' }} />

// ✅ Hoist ra ngoài hoặc useMemo
const style = useMemo(() => ({ color: 'red' }), []);
<Child style={style} />

// ❌ Inline function
<Child onClick={() => handleClick(id)} />

// ✅ useCallback
const handleItemClick = useCallback(() => handleClick(id), [id]);
<Child onClick={handleItemClick} />
```

---

### 5. React Compiler (React 19) — Auto memoization

**Câu hỏi: React Compiler là gì? Nó thay đổi cách tối ưu như thế nào?**

**Trả lời:**

React Compiler (trước đây là React Forget) tự động thêm memoization vào build time, loại bỏ nhu cầu viết `useMemo`, `useCallback`, `React.memo` thủ công.

```jsx
// Trước React Compiler — phải memo thủ công
function ProductList({ products }) {
  const sorted = useMemo(() =>
    [...products].sort((a, b) => a.price - b.price),
  [products]);

  const handleClick = useCallback((id) => {
    navigate(`/product/${id}`);
  }, [navigate]);

  return sorted.map(p => (
    <ProductCard key={p.id} product={p} onClick={handleClick} />
  ));
}
const ProductCard = React.memo(({ product, onClick }) => { ... });

// Sau React Compiler — viết bình thường, compiler tự optimize
function ProductList({ products }) {
  const sorted = [...products].sort((a, b) => a.price - b.price);

  const handleClick = (id) => {
    navigate(`/product/${id}`);
  };

  return sorted.map(p => (
    <ProductCard key={p.id} product={p} onClick={handleClick} />
  ));
}
function ProductCard({ product, onClick }) { ... }
// Compiler tự thêm memo ở build time!
```

**Điều kiện để Compiler hoạt động đúng:**
- Code phải tuân thủ **Rules of React** (pure components, no side effects in render)
- Không mutate props/state
- Hooks phải ở top level

---

### 6. Concurrent Features sâu — useDeferredValue vs useTransition

**Câu hỏi: So sánh useDeferredValue và useTransition.**

**Trả lời:**

Cả hai đều đánh dấu update là "low priority" nhưng dùng trong hoàn cảnh khác:

| | useTransition | useDeferredValue |
|---|---|---|
| Dùng khi | Bạn **kiểm soát** setState | Bạn **nhận** value từ props/parent |
| Wrap | Wrap **action** gây state update | Wrap **value** đã có |
| Return | `[isPending, startTransition]` | deferred value |

```jsx
// useTransition: bạn kiểm soát cả input + filter
function SearchPage() {
  const [query, setQuery] = useState('');
  const [isPending, startTransition] = useTransition();

  const handleChange = (e) => {
    setQuery(e.target.value); // Urgent: update input ngay
    startTransition(() => {
      setFilteredResults(filterData(e.target.value)); // Non-urgent
    });
  };

  return (
    <>
      <input value={query} onChange={handleChange} />
      {isPending ? <Spinner /> : <ResultsList />}
    </>
  );
}

// useDeferredValue: bạn nhận query từ parent, không kiểm soát setState
function ResultsList({ query }) {
  const deferredQuery = useDeferredValue(query);
  const isStale = query !== deferredQuery;

  const results = useMemo(() => filterData(deferredQuery), [deferredQuery]);

  return (
    <div style={{ opacity: isStale ? 0.5 : 1 }}>
      {results.map(r => <ResultItem key={r.id} data={r} />)}
    </div>
  );
}
```
