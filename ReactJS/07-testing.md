# 📘 ReactJS - Phần 7: Testing

[⬅️ Advanced](./06-advanced.md) | [Phần tiếp: Tình huống thực tế ➡️](./08-tinh-huong-thuc-te.md)

---

## 1. React Testing Library

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

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 2. Testing Custom Hooks

**Câu hỏi: Cách test custom hook?**

**Trả lời:**

```jsx
import { renderHook, act, waitFor } from '@testing-library/react';

// Test useCounter hook
function useCounter(initial = 0) {
  const [count, setCount] = useState(initial);
  const increment = () => setCount(c => c + 1);
  const decrement = () => setCount(c => c - 1);
  const reset = () => setCount(initial);
  return { count, increment, decrement, reset };
}

test('useCounter increments and decrements', () => {
  const { result } = renderHook(() => useCounter(10));

  expect(result.current.count).toBe(10);

  act(() => { result.current.increment(); });
  expect(result.current.count).toBe(11);

  act(() => { result.current.decrement(); });
  expect(result.current.count).toBe(10);

  act(() => { result.current.reset(); });
  expect(result.current.count).toBe(10);
});

// Test hook with async operations
test('useFetch loads data', async () => {
  global.fetch = jest.fn().mockResolvedValue({
    json: () => Promise.resolve({ name: 'John' }),
  });

  const { result } = renderHook(() => useFetch('/api/user'));

  expect(result.current.loading).toBe(true);

  await waitFor(() => {
    expect(result.current.loading).toBe(false);
    expect(result.current.data).toEqual({ name: 'John' });
  });
});

// Test hook that depends on props (re-render with new props)
test('useFetch refetches when URL changes', async () => {
  const { result, rerender } = renderHook(
    ({ url }) => useFetch(url),
    { initialProps: { url: '/api/users/1' } },
  );

  await waitFor(() => expect(result.current.data).toBeTruthy());

  rerender({ url: '/api/users/2' }); // Simulate prop change

  await waitFor(() => {
    expect(fetch).toHaveBeenCalledWith('/api/users/2', expect.any(Object));
  });
});
```

---

### 3. Testing với Context / Redux Provider

```jsx
// Custom render helper với providers
function renderWithProviders(ui, { preloadedState = {}, store, ...options } = {}) {
  const testStore = store ?? configureStore({
    reducer: { auth: authReducer, cart: cartReducer },
    preloadedState,
  });

  function Wrapper({ children }) {
    return (
      <Provider store={testStore}>
        <BrowserRouter>
          <ThemeProvider>{children}</ThemeProvider>
        </BrowserRouter>
      </Provider>
    );
  }

  return { store: testStore, ...render(ui, { wrapper: Wrapper, ...options }) };
}

// Sử dụng
test('shows admin panel for admin users', () => {
  renderWithProviders(<Dashboard />, {
    preloadedState: {
      auth: { user: { role: 'admin' }, isAuthenticated: true },
    },
  });

  expect(screen.getByText('Admin Panel')).toBeInTheDocument();
});
```

---

### 4. MSW (Mock Service Worker) — Mock API tốt hơn jest.fn()

```jsx
import { setupServer } from 'msw/node';
import { http, HttpResponse } from 'msw';

const server = setupServer(
  http.get('/api/users', () => {
    return HttpResponse.json([
      { id: 1, name: 'John' },
      { id: 2, name: 'Jane' },
    ]);
  }),
  http.post('/api/users', async ({ request }) => {
    const body = await request.json();
    return HttpResponse.json({ id: 3, ...body }, { status: 201 });
  }),
);

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test('renders users from API', async () => {
  render(<UserList />);

  await waitFor(() => {
    expect(screen.getByText('John')).toBeInTheDocument();
    expect(screen.getByText('Jane')).toBeInTheDocument();
  });
});

test('handles server error', async () => {
  // Override handler cho test case cụ thể
  server.use(
    http.get('/api/users', () => {
      return HttpResponse.json({ message: 'Server Error' }, { status: 500 });
    }),
  );

  render(<UserList />);

  await waitFor(() => {
    expect(screen.getByText(/error/i)).toBeInTheDocument();
  });
});
```
