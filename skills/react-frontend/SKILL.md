---
name: react-frontend
description: React frontend development for Home Assistant apps. Use when writing or modifying React components, state management, API integration, theming, or responsive layouts for HA app (add-on) web interfaces served through ingress.
---

# React Frontend — Non-Obvious Reference

This skill covers React frontend patterns for HA app web interfaces. Only non-obvious details and HA-specific concerns — standard React knowledge applies for everything else.

## 1. Ingress Base Path Handling

The most common source of bugs in HA app frontends. Your app is served at a dynamic path, not root.

**In `index.html`:**
```html
<!-- The server must inject the correct base path -->
<base href="%BASE_PATH%/" />
```

**In your app:**
```javascript
// Read base path from the <base> tag
const basePath = document.querySelector('base')?.getAttribute('href') || '/';

// Use for all API calls
const apiFetch = (path, options) =>
  fetch(`${basePath}api${path}`, options);

// Use for React Router
<BrowserRouter basename={basePath}>
```

**Common mistakes:**
```javascript
// ❌ WRONG — hardcoded paths break under ingress
fetch('/api/data')
<Link to="/settings">

// ✅ CORRECT — relative to base path
fetch(`${basePath}api/data`)
<Link to="settings">  // React Router handles basename
```

## 2. React Query for Server State

```javascript
import { QueryClient, QueryClientProvider, useQuery, useMutation } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: 1,
      staleTime: 30 * 1000,        // 30 seconds before refetch
      refetchOnWindowFocus: true,   // Refetch when user returns to tab
    },
  },
});

// Fetching data
function useEntities() {
  return useQuery({
    queryKey: ['entities'],
    queryFn: () => apiFetch('/entities').then(r => r.json()),
  });
}

// Mutations with cache invalidation
function useCreateEntity() {
  return useMutation({
    mutationFn: (data) => apiFetch('/entities', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data),
    }),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['entities'] });
    },
  });
}
```

**Key patterns:**
- Use `queryKey` arrays for hierarchical cache invalidation
- Invalidate on mutation success, don't manually update cache unless needed
- `staleTime` prevents unnecessary refetches — tune per data type

## 3. Authentication Integration

**With HA ingress (recommended):**
No separate auth needed — HA handles authentication before requests reach your app. The user is already logged into HA.

**With standalone access (direct port):**
```javascript
// Store JWT in memory (not localStorage for security)
const AuthContext = createContext();

function AuthProvider({ children }) {
  const [token, setToken] = useState(null);

  const login = async (username, password) => {
    const response = await fetch(`${basePath}api/auth/login`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ username, password }),
    });
    const data = await response.json();
    setToken(data.access_token);
  };

  return (
    <AuthContext.Provider value={{ token, login, logout }}>
      {children}
    </AuthContext.Provider>
  );
}
```

**Include token in API calls:**
```javascript
const apiFetch = (path, options = {}) => {
  const headers = { ...options.headers };
  if (token) headers['Authorization'] = `Bearer ${token}`;
  return fetch(`${basePath}api${path}`, { ...options, headers });
};
```

## 4. Theming (Dark/Light Mode)

HA users expect dark mode support:

```javascript
// Detect system preference
const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;

// Or read from HA theme (when served through ingress)
const haTheme = document.body.getAttribute('data-theme'); // 'dark' or 'light'
```

**CSS custom properties approach:**
```css
:root {
  --bg-primary: #ffffff;
  --bg-secondary: #f5f5f5;
  --text-primary: #1a1a1a;
  --text-secondary: #666666;
  --accent: #0366d6;
  --danger: #d32f2f;
  --success: #2e7d32;
  --border: #e0e0e0;
}

[data-theme="dark"] {
  --bg-primary: #1a1a1a;
  --bg-secondary: #2d2d2d;
  --text-primary: #e0e0e0;
  --text-secondary: #a0a0a0;
  --accent: #58a6ff;
  --danger: #f44336;
  --success: #4caf50;
  --border: #404040;
}
```

## 5. Responsive Design

HA apps can be viewed on phones, tablets, and desktops — especially through the HA companion app.

```css
/* Mobile first */
.container { padding: 8px; }
.grid { display: grid; grid-template-columns: 1fr; gap: 8px; }

/* Tablet */
@media (min-width: 768px) {
  .container { padding: 16px; }
  .grid { grid-template-columns: repeat(2, 1fr); gap: 16px; }
}

/* Desktop */
@media (min-width: 1024px) {
  .container { padding: 24px; max-width: 1200px; margin: 0 auto; }
  .grid { grid-template-columns: repeat(3, 1fr); }
}
```

## 6. Error Boundaries

```javascript
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, info) {
    console.error('React error:', error, info);
  }

  render() {
    if (this.state.hasError) {
      return (
        <div className="error-page">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ hasError: false })}>
            Try again
          </button>
        </div>
      );
    }
    return this.props.children;
  }
}
```

Wrap the top-level app and individual route components.

## 7. Loading and Empty States

Always handle all states — don't show blank screens:

```javascript
function EntityList() {
  const { data, isLoading, error } = useEntities();

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage message={error.message} />;
  if (!data?.length) return <EmptyState message="No entities found" />;

  return <div>{data.map(entity => <EntityCard key={entity.id} entity={entity} />)}</div>;
}
```

## 8. Form Handling with Validation

```javascript
function EntityForm({ onSubmit }) {
  const [errors, setErrors] = useState({});

  const validate = (data) => {
    const errors = {};
    if (!data.name?.trim()) errors.name = 'Name is required';
    if (data.port && (data.port < 1 || data.port > 65535)) errors.port = 'Invalid port';
    return errors;
  };

  const handleSubmit = (e) => {
    e.preventDefault();
    const formData = Object.fromEntries(new FormData(e.target));
    const validationErrors = validate(formData);
    if (Object.keys(validationErrors).length) {
      setErrors(validationErrors);
      return;
    }
    onSubmit(formData);
  };

  return (
    <form onSubmit={handleSubmit}>
      <input name="name" className={errors.name ? 'error' : ''} />
      {errors.name && <span className="error-text">{errors.name}</span>}
      <button type="submit">Save</button>
    </form>
  );
}
```

## 9. Build Configuration

```javascript
// vite.config.js (recommended over CRA/webpack)
import { defineConfig } from 'vite';
import react from '@vitejs/plugin-react';

export default defineConfig({
  plugins: [react()],
  base: './',                    // Relative paths for ingress compatibility
  build: {
    outDir: 'build',
    sourcemap: false,            // Smaller bundle for embedded devices
  },
  server: {
    port: 3000,
    proxy: {
      '/api': 'http://localhost:8000',  // Proxy to backend in development
    },
  },
});
```

**Critical:** `base: './'` ensures all asset paths are relative, which is required for ingress to work.

## 10. Testing Patterns

```javascript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

// Test wrapper with providers
function renderWithProviders(ui) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(
    <QueryClientProvider client={queryClient}>
      {ui}
    </QueryClientProvider>
  );
}

// Debounced search components
test('search filters results after debounce', async () => {
  jest.useFakeTimers();
  renderWithProviders(<SearchComponent />);
  fireEvent.change(screen.getByRole('searchbox'), { target: { value: 'test' } });
  jest.advanceTimersByTime(300);
  await waitFor(() => expect(screen.getByText('test result')).toBeInTheDocument());
  jest.useRealTimers();
});
```
