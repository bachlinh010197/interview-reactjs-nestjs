# 📘 ReactJS - Phần 1: Kiến Thức Nền Tảng

[⬅️ Mục lục](../README.md) | [Phần tiếp: Hooks ➡️](./02-hooks.md)

---

## 1. React là gì? Tại sao nên dùng React?

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

## 2. Virtual DOM là gì? Hoạt động như thế nào?

**Trả lời:**

Virtual DOM là một bản sao nhẹ (lightweight copy) của Real DOM, được lưu trong bộ nhớ dưới dạng JavaScript object.

**Cách hoạt động (Reconciliation):**
1. Khi state/props thay đổi, React tạo một Virtual DOM mới
2. So sánh Virtual DOM mới với Virtual DOM cũ (Diffing Algorithm)
3. Tìm ra sự khác biệt nhỏ nhất (minimal changes)
4. Chỉ cập nhật những phần thay đổi lên Real DOM (Batch update)

```jsx
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

## 3. JSX là gì?

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

## 4. Function Component vs Class Component

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

## 5. Props vs State

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

## 6. One-way Data Binding (Luồng dữ liệu một chiều)

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

## 7. React Element vs React Component

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

## 8. Key trong React là gì? Tại sao quan trọng?

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

## 9. Controlled vs Uncontrolled Components

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

## 10. Synthetic Event trong React là gì?

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

## 🔥 CÂU HỎI LEVEL MIDDLE → SENIOR

### 11. React Fiber là gì? Tại sao React cần Fiber Architecture?

**Trả lời:**

React Fiber là engine reconciliation được viết lại hoàn toàn trong React 16, thay thế stack-based reconciler cũ.

**Vấn đề của Stack Reconciler cũ:**
- Synchronous rendering: khi cây component lớn, UI bị freeze
- Không thể dừng giữa chừng (non-interruptible)

**Fiber giải quyết bằng cách:**
- **Incremental rendering**: Chia công việc render thành các đơn vị nhỏ (units of work)
- **Interruptible**: Có thể pause, resume, abort render
- **Priority-based**: Gán priority cho từng update (animation > data fetch)
- **Concurrent Mode**: Render nhiều version của UI cùng lúc

```
// Fiber Node structure (simplified)
{
  type: 'div',           // Component type
  key: null,
  stateNode: DOMElement, // Tham chiếu đến DOM node
  child: FiberNode,      // Con đầu tiên
  sibling: FiberNode,    // Anh em kế tiếp
  return: FiberNode,     // Parent
  pendingProps: {},
  memoizedState: {},
  effectTag: 'UPDATE',   // Side effect cần thực hiện
}
```

**2 phases:**
1. **Render phase** (interruptible): Duyệt cây fiber, tìm changes — có thể bị interrupt
2. **Commit phase** (synchronous): Apply changes lên DOM — không thể interrupt

---

### 12. Giải thích React Reconciliation Algorithm chi tiết

**Trả lời:**

React dùng heuristic O(n) algorithm thay vì O(n³) tree diff:

**2 giả định chính:**
1. Hai elements khác type → tạo cây mới hoàn toàn
2. `key` prop giúp nhận diện element nào ổn định qua các render

```jsx
// Case 1: Khác type → unmount cây cũ, mount cây mới
// Before
<div><Counter /></div>
// After → Counter bị unmount hoàn toàn, state mất
<span><Counter /></span>

// Case 2: Cùng type → giữ instance, chỉ update props
// Before
<div className="old" />
// After → chỉ update className, không tạo DOM mới
<div className="new" />

// Case 3: List với key → React biết element nào thêm/xóa/di chuyển
// Before
<li key="a">A</li>
<li key="b">B</li>
// After → React biết chỉ cần thêm C ở đầu
<li key="c">C</li>
<li key="a">A</li>
<li key="b">B</li>
```

---

### 13. Closure trap trong React Hooks — giải thích và cách tránh

**Trả lời:**

Closure trap xảy ra khi callback "bắt" (close over) giá trị cũ của state/props do JavaScript closure.

```jsx
function Timer() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const id = setInterval(() => {
      // ❌ BUG: count luôn = 0 vì closure bắt giá trị lúc mount
      console.log(count); // Luôn in 0
      setCount(count + 1); // Luôn set thành 1
    }, 1000);
    return () => clearInterval(id);
  }, []); // dependency array rỗng

  // ✅ FIX 1: Dùng functional update
  useEffect(() => {
    const id = setInterval(() => {
      setCount(prev => prev + 1); // Luôn lấy giá trị mới nhất
    }, 1000);
    return () => clearInterval(id);
  }, []);

  // ✅ FIX 2: Dùng useRef để lưu giá trị mới nhất
  const countRef = useRef(count);
  countRef.current = count;

  useEffect(() => {
    const id = setInterval(() => {
      console.log(countRef.current); // Luôn đúng
    }, 1000);
    return () => clearInterval(id);
  }, []);
}
```

---

### 14. React Render vs Commit — Component render nhưng DOM không update?

**Trả lời:**

**Render ≠ DOM update.** React render có 2 phase:

1. **Render phase**: React gọi component function → tạo React Elements → so sánh (diff) với lần trước
2. **Commit phase**: Nếu có diff → cập nhật DOM. **Nếu không có diff → không cập nhật DOM**

```jsx
function Parent() {
  const [count, setCount] = useState(0);
  console.log('Parent rendered'); // Gọi mỗi lần state thay đổi

  return (
    <div>
      <button onClick={() => setCount(c => c + 1)}>{count}</button>
      <Child /> {/* Child cũng bị render lại dù không dùng count! */}
    </div>
  );
}

function Child() {
  console.log('Child rendered'); // Gọi mỗi lần Parent render
  return <p>Static content</p>; // Nhưng DOM KHÔNG update vì output giống hệt
}

// Fix: React.memo ngăn re-render
const Child = React.memo(function Child() {
  console.log('Child rendered'); // CHỈ gọi khi props thay đổi
  return <p>Static content</p>;
});
```

---

### 15. Khi nào setState KHÔNG gây re-render?

**Trả lời:**

React sử dụng `Object.is()` để so sánh. Nếu giá trị mới **giống hệt** giá trị cũ → **bỏ qua re-render** (bailout).

```jsx
const [count, setCount] = useState(0);
setCount(0); // Không re-render vì Object.is(0, 0) = true

const [user, setUser] = useState({ name: 'John' });
setUser({ name: 'John' }); // CÓ re-render vì {} !== {} (reference khác)

// ❌ Mutate object → React không phát hiện thay đổi → KHÔNG re-render
user.name = 'Jane';
setUser(user); // Object.is(user, user) = true → bỏ qua!

// ✅ Tạo object mới
setUser({ ...user, name: 'Jane' }); // Reference mới → re-render
```
