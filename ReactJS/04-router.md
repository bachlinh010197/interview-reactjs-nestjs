# 📘 ReactJS - Phần 4: React Router

[⬅️ State Management](./03-state-management.md) | [Phần tiếp: Performance ➡️](./05-performance.md)

---

## 1. React Router v6

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
