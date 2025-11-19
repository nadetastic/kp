---
name: "react-dev"
version: "1.0.0"
displayName: "React Development"
description: "Best practices, patterns, and workflows for building React applications with hooks, state management, and testing"
keywords: ["react", "hooks", "components", "jsx", "state", "testing", "typescript"]
---

# React Development Power

## Overview

Comprehensive guide for building modern React applications with best practices, common patterns, and workflows. Covers component creation, hooks usage, state management, TypeScript integration, testing strategies, and performance optimization. This is a documentation-only power that provides guidance without executable tools.

Whether you're creating new components, refactoring class components to hooks, setting up testing, or optimizing performance, this power provides clear patterns and examples for common React development tasks.

## When to Use This Power

- Creating new React components with proper structure
- Implementing custom hooks for reusable logic
- Managing component state with useState, useReducer, useContext
- Handling side effects with useEffect properly
- Setting up TypeScript types for React components
- Writing tests for React components and hooks
- Optimizing performance with useMemo, useCallback, React.memo
- Implementing common patterns (forms, data fetching, routing)
- Refactoring class components to functional components
- Setting up project structure for scalability
- Debugging common React issues

## React Component Patterns

### Functional Component with TypeScript

```typescript
import React from 'react';

interface ButtonProps {
  label: string;
  onClick: () => void;
  variant?: 'primary' | 'secondary';
  disabled?: boolean;
  children?: React.ReactNode;
}

export const Button: React.FC<ButtonProps> = ({
  label,
  onClick,
  variant = 'primary',
  disabled = false,
  children
}) => {
  return (
    <button
      className={`btn btn-${variant}`}
      onClick={onClick}
      disabled={disabled}
      aria-label={label}
    >
      {children || label}
    </button>
  );
};
```

### Component with State and Effects

```typescript
import React, { useState, useEffect } from 'react';

interface User {
  id: number;
  name: string;
  email: string;
}

export const UserProfile: React.FC<{ userId: number }> = ({ userId }) => {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    let cancelled = false;

    const fetchUser = async () => {
      try {
        setLoading(true);
        const response = await fetch(`/api/users/${userId}`);
        if (!response.ok) throw new Error('Failed to fetch user');
        const data = await response.json();
        
        if (!cancelled) {
          setUser(data);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Unknown error');
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    };

    fetchUser();

    return () => {
      cancelled = true;
    };
  }, [userId]);

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;
  if (!user) return <div>No user found</div>;

  return (
    <div className="user-profile">
      <h2>{user.name}</h2>
      <p>{user.email}</p>
    </div>
  );
};
```

## Custom Hooks Patterns

### Data Fetching Hook

```typescript
import { useState, useEffect } from 'react';

interface UseFetchResult<T> {
  data: T | null;
  loading: boolean;
  error: string | null;
  refetch: () => void;
}

export function useFetch<T>(url: string): UseFetchResult<T> {
  const [data, setData] = useState<T | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [refetchTrigger, setRefetchTrigger] = useState(0);

  useEffect(() => {
    let cancelled = false;

    const fetchData = async () => {
      try {
        setLoading(true);
        const response = await fetch(url);
        if (!response.ok) throw new Error(`HTTP error! status: ${response.status}`);
        const json = await response.json();
        
        if (!cancelled) {
          setData(json);
          setError(null);
        }
      } catch (err) {
        if (!cancelled) {
          setError(err instanceof Error ? err.message : 'Unknown error');
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    };

    fetchData();

    return () => {
      cancelled = true;
    };
  }, [url, refetchTrigger]);

  const refetch = () => setRefetchTrigger(prev => prev + 1);

  return { data, loading, error, refetch };
}

// Usage
const UserList: React.FC = () => {
  const { data: users, loading, error, refetch } = useFetch<User[]>('/api/users');

  if (loading) return <div>Loading...</div>;
  if (error) return <div>Error: {error}</div>;

  return (
    <div>
      <button onClick={refetch}>Refresh</button>
      {users?.map(user => <div key={user.id}>{user.name}</div>)}
    </div>
  );
};
```

### Form Hook

```typescript
import { useState, ChangeEvent, FormEvent } from 'react';

interface UseFormOptions<T> {
  initialValues: T;
  onSubmit: (values: T) => void | Promise<void>;
  validate?: (values: T) => Partial<Record<keyof T, string>>;
}

export function useForm<T extends Record<string, any>>({
  initialValues,
  onSubmit,
  validate
}: UseFormOptions<T>) {
  const [values, setValues] = useState<T>(initialValues);
  const [errors, setErrors] = useState<Partial<Record<keyof T, string>>>({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const handleChange = (e: ChangeEvent<HTMLInputElement | HTMLTextAreaElement>) => {
    const { name, value } = e.target;
    setValues(prev => ({ ...prev, [name]: value }));
    // Clear error for this field
    if (errors[name as keyof T]) {
      setErrors(prev => ({ ...prev, [name]: undefined }));
    }
  };

  const handleSubmit = async (e: FormEvent) => {
    e.preventDefault();
    
    if (validate) {
      const validationErrors = validate(values);
      if (Object.keys(validationErrors).length > 0) {
        setErrors(validationErrors);
        return;
      }
    }

    setIsSubmitting(true);
    try {
      await onSubmit(values);
    } finally {
      setIsSubmitting(false);
    }
  };

  const reset = () => {
    setValues(initialValues);
    setErrors({});
  };

  return {
    values,
    errors,
    isSubmitting,
    handleChange,
    handleSubmit,
    reset
  };
}

// Usage
interface LoginForm {
  email: string;
  password: string;
}

const LoginComponent: React.FC = () => {
  const { values, errors, isSubmitting, handleChange, handleSubmit } = useForm<LoginForm>({
    initialValues: { email: '', password: '' },
    onSubmit: async (values) => {
      await fetch('/api/login', {
        method: 'POST',
        body: JSON.stringify(values)
      });
    },
    validate: (values) => {
      const errors: Partial<Record<keyof LoginForm, string>> = {};
      if (!values.email) errors.email = 'Email is required';
      if (!values.password) errors.password = 'Password is required';
      return errors;
    }
  });

  return (
    <form onSubmit={handleSubmit}>
      <input
        name="email"
        value={values.email}
        onChange={handleChange}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email}</span>}
      
      <input
        name="password"
        type="password"
        value={values.password}
        onChange={handleChange}
        placeholder="Password"
      />
      {errors.password && <span>{errors.password}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Logging in...' : 'Login'}
      </button>
    </form>
  );
};
```

## State Management Patterns

### Context + Reducer Pattern

```typescript
import React, { createContext, useContext, useReducer, ReactNode } from 'react';

// State type
interface AppState {
  user: User | null;
  theme: 'light' | 'dark';
  notifications: Notification[];
}

// Action types
type AppAction =
  | { type: 'SET_USER'; payload: User | null }
  | { type: 'TOGGLE_THEME' }
  | { type: 'ADD_NOTIFICATION'; payload: Notification }
  | { type: 'REMOVE_NOTIFICATION'; payload: string };

// Reducer
function appReducer(state: AppState, action: AppAction): AppState {
  switch (action.type) {
    case 'SET_USER':
      return { ...state, user: action.payload };
    case 'TOGGLE_THEME':
      return { ...state, theme: state.theme === 'light' ? 'dark' : 'light' };
    case 'ADD_NOTIFICATION':
      return { ...state, notifications: [...state.notifications, action.payload] };
    case 'REMOVE_NOTIFICATION':
      return {
        ...state,
        notifications: state.notifications.filter(n => n.id !== action.payload)
      };
    default:
      return state;
  }
}

// Context
const AppContext = createContext<{
  state: AppState;
  dispatch: React.Dispatch<AppAction>;
} | undefined>(undefined);

// Provider
export const AppProvider: React.FC<{ children: ReactNode }> = ({ children }) => {
  const [state, dispatch] = useReducer(appReducer, {
    user: null,
    theme: 'light',
    notifications: []
  });

  return (
    <AppContext.Provider value={{ state, dispatch }}>
      {children}
    </AppContext.Provider>
  );
};

// Hook
export function useApp() {
  const context = useContext(AppContext);
  if (!context) {
    throw new Error('useApp must be used within AppProvider');
  }
  return context;
}

// Usage
const UserDisplay: React.FC = () => {
  const { state, dispatch } = useApp();

  const handleLogout = () => {
    dispatch({ type: 'SET_USER', payload: null });
  };

  return (
    <div>
      {state.user ? (
        <>
          <p>Welcome, {state.user.name}</p>
          <button onClick={handleLogout}>Logout</button>
        </>
      ) : (
        <p>Please log in</p>
      )}
    </div>
  );
};
```

## Performance Optimization

### useMemo and useCallback

```typescript
import React, { useState, useMemo, useCallback } from 'react';

interface Item {
  id: number;
  name: string;
  price: number;
  category: string;
}

const ProductList: React.FC<{ items: Item[] }> = ({ items }) => {
  const [filter, setFilter] = useState('');
  const [sortBy, setSortBy] = useState<'name' | 'price'>('name');

  // Memoize expensive filtering/sorting
  const filteredAndSortedItems = useMemo(() => {
    console.log('Filtering and sorting...');
    return items
      .filter(item => item.name.toLowerCase().includes(filter.toLowerCase()))
      .sort((a, b) => {
        if (sortBy === 'name') return a.name.localeCompare(b.name);
        return a.price - b.price;
      });
  }, [items, filter, sortBy]);

  // Memoize callback to prevent child re-renders
  const handleFilterChange = useCallback((e: React.ChangeEvent<HTMLInputElement>) => {
    setFilter(e.target.value);
  }, []);

  return (
    <div>
      <input
        type="text"
        value={filter}
        onChange={handleFilterChange}
        placeholder="Filter products..."
      />
      <select value={sortBy} onChange={(e) => setSortBy(e.target.value as 'name' | 'price')}>
        <option value="name">Sort by Name</option>
        <option value="price">Sort by Price</option>
      </select>
      
      {filteredAndSortedItems.map(item => (
        <ProductItem key={item.id} item={item} />
      ))}
    </div>
  );
};

// Memoized child component
const ProductItem = React.memo<{ item: Item }>(({ item }) => {
  console.log('Rendering ProductItem:', item.name);
  return (
    <div className="product-item">
      <h3>{item.name}</h3>
      <p>${item.price}</p>
    </div>
  );
});
```

## Testing Patterns

### Component Testing with React Testing Library

```typescript
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { UserProfile } from './UserProfile';

describe('UserProfile', () => {
  it('renders loading state initially', () => {
    render(<UserProfile userId={1} />);
    expect(screen.getByText('Loading...')).toBeInTheDocument();
  });

  it('renders user data after fetch', async () => {
    global.fetch = vi.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({ id: 1, name: 'John Doe', email: 'john@example.com' })
      } as Response)
    );

    render(<UserProfile userId={1} />);

    await waitFor(() => {
      expect(screen.getByText('John Doe')).toBeInTheDocument();
      expect(screen.getByText('john@example.com')).toBeInTheDocument();
    });
  });

  it('renders error state on fetch failure', async () => {
    global.fetch = vi.fn(() =>
      Promise.resolve({
        ok: false,
        status: 404
      } as Response)
    );

    render(<UserProfile userId={1} />);

    await waitFor(() => {
      expect(screen.getByText(/Error:/)).toBeInTheDocument();
    });
  });
});
```

### Hook Testing

```typescript
import { renderHook, act, waitFor } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { useFetch } from './useFetch';

describe('useFetch', () => {
  it('fetches data successfully', async () => {
    const mockData = { id: 1, name: 'Test' };
    global.fetch = vi.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve(mockData)
      } as Response)
    );

    const { result } = renderHook(() => useFetch('/api/test'));

    expect(result.current.loading).toBe(true);

    await waitFor(() => {
      expect(result.current.loading).toBe(false);
      expect(result.current.data).toEqual(mockData);
      expect(result.current.error).toBeNull();
    });
  });

  it('refetches data when refetch is called', async () => {
    global.fetch = vi.fn(() =>
      Promise.resolve({
        ok: true,
        json: () => Promise.resolve({ id: 1 })
      } as Response)
    );

    const { result } = renderHook(() => useFetch('/api/test'));

    await waitFor(() => expect(result.current.loading).toBe(false));

    act(() => {
      result.current.refetch();
    });

    expect(result.current.loading).toBe(true);
    await waitFor(() => expect(result.current.loading).toBe(false));
    expect(global.fetch).toHaveBeenCalledTimes(2);
  });
});
```

## Common Workflows

### Creating a New Feature Component

1. **Create component file with TypeScript interface**
2. **Implement component with proper props typing**
3. **Add state management if needed (useState, useReducer, or context)**
4. **Implement side effects with useEffect (with cleanup)**
5. **Add error handling and loading states**
6. **Create tests for component behavior**
7. **Add accessibility attributes (aria-labels, roles)**
8. **Optimize with React.memo, useMemo, useCallback if needed**

### Refactoring Class to Functional Component

```typescript
// Before: Class Component
class UserList extends React.Component<Props, State> {
  state = { users: [], loading: true };

  componentDidMount() {
    this.fetchUsers();
  }

  componentDidUpdate(prevProps: Props) {
    if (prevProps.filter !== this.props.filter) {
      this.fetchUsers();
    }
  }

  componentWillUnmount() {
    this.cancelled = true;
  }

  fetchUsers = async () => {
    // fetch logic
  };

  render() {
    return <div>{/* render logic */}</div>;
  }
}

// After: Functional Component
const UserList: React.FC<Props> = ({ filter }) => {
  const [users, setUsers] = useState<User[]>([]);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let cancelled = false;

    const fetchUsers = async () => {
      setLoading(true);
      try {
        const response = await fetch(`/api/users?filter=${filter}`);
        const data = await response.json();
        if (!cancelled) {
          setUsers(data);
        }
      } finally {
        if (!cancelled) {
          setLoading(false);
        }
      }
    };

    fetchUsers();

    return () => {
      cancelled = true;
    };
  }, [filter]);

  return <div>{/* render logic */}</div>;
};
```

## Best Practices

### DO ✅

- Use functional components with hooks (modern React)
- Type all props and state with TypeScript interfaces
- Clean up effects with return functions (prevent memory leaks)
- Use dependency arrays correctly in useEffect, useMemo, useCallback
- Implement proper error boundaries for error handling
- Use React.memo for expensive components that re-render often
- Keep components small and focused (single responsibility)
- Extract reusable logic into custom hooks
- Use proper key props in lists (stable, unique identifiers)
- Implement loading and error states for async operations
- Add accessibility attributes (aria-labels, roles, semantic HTML)
- Use controlled components for forms
- Validate props with TypeScript instead of PropTypes
- Use context for global state, props for local state
- Test components with React Testing Library (user behavior, not implementation)

### DON'T ❌

- Mutate state directly (always use setState functions)
- Forget cleanup in useEffect (causes memory leaks)
- Use index as key in dynamic lists
- Put too much logic in components (extract to hooks or utils)
- Ignore dependency array warnings in useEffect
- Over-optimize with useMemo/useCallback (only when needed)
- Use inline functions in JSX for memoized components
- Fetch data in render (use useEffect instead)
- Store derived state (calculate from existing state)
- Use class components for new code
- Ignore TypeScript errors with 'any' type
- Create deeply nested component trees (flatten when possible)
- Use context for frequently changing values (causes re-renders)
- Test implementation details (test user-facing behavior)
- Forget to handle loading and error states

## Troubleshooting

### "Cannot read property of undefined"
**Cause:** Accessing nested properties without checking if parent exists  
**Solution:** Use optional chaining: `user?.profile?.name` or check before accessing

### "Too many re-renders"
**Cause:** Setting state in render or useEffect without dependencies  
**Solution:** Move state updates to event handlers or add proper dependencies to useEffect

### "Memory leak warning"
**Cause:** Setting state after component unmounts  
**Solution:** Use cleanup function in useEffect with cancelled flag

```typescript
useEffect(() => {
  let cancelled = false;
  
  fetchData().then(data => {
    if (!cancelled) {
      setData(data);
    }
  });
  
  return () => {
    cancelled = true;
  };
}, []);
```

### "Hook called conditionally"
**Cause:** Calling hooks inside conditions, loops, or nested functions  
**Solution:** Always call hooks at the top level of component

```typescript
// ❌ Wrong
if (condition) {
  useState(0);
}

// ✅ Correct
const [count, setCount] = useState(0);
if (condition) {
  // use count here
}
```

### "Infinite loop in useEffect"
**Cause:** Missing or incorrect dependencies in useEffect  
**Solution:** Add all used variables to dependency array or use useCallback for functions

```typescript
// ❌ Wrong - missing dependency
useEffect(() => {
  fetchData(userId);
}, []);

// ✅ Correct
useEffect(() => {
  fetchData(userId);
}, [userId]);
```

### "Component not re-rendering"
**Cause:** Mutating state directly instead of creating new reference  
**Solution:** Always create new objects/arrays when updating state

```typescript
// ❌ Wrong
const handleAdd = () => {
  items.push(newItem);
  setItems(items);
};

// ✅ Correct
const handleAdd = () => {
  setItems([...items, newItem]);
};
```

## Project Structure

```
src/
├── components/          # Reusable UI components
│   ├── Button/
│   │   ├── Button.tsx
│   │   ├── Button.test.tsx
│   │   └── Button.module.css
│   └── Input/
├── hooks/              # Custom hooks
│   ├── useFetch.ts
│   ├── useForm.ts
│   └── useLocalStorage.ts
├── context/            # Context providers
│   ├── AppContext.tsx
│   └── ThemeContext.tsx
├── pages/              # Page components (if using routing)
│   ├── Home.tsx
│   └── Profile.tsx
├── utils/              # Utility functions
│   ├── api.ts
│   └── validation.ts
├── types/              # TypeScript types
│   └── index.ts
└── App.tsx
```

## Configuration

**No configuration required** - This is a documentation-only power providing React development guidance and patterns.

**Recommended Setup:**
- TypeScript for type safety
- ESLint with react-hooks plugin
- React Testing Library + Vitest for testing
- Prettier for code formatting

---

**Package:** Documentation-only power  
**Source:** React best practices and patterns  
**License:** MIT
