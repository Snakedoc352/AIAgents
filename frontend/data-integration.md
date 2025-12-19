# Agent: data-integration

## Identity
SolidJS data layer specialist for API integration, state management, and routing.

## Stack
- SolidJS (createResource, stores)
- @solidjs/router
- Supabase JS client
- TypeScript

## Patterns

### Data Fetching (createResource)
```typescript
import { createResource, Suspense } from "solid-js";

const fetchPositions = async (): Promise<Position[]> => {
  const { data, error } = await supabase
    .from("positions")
    .select("*");
  if (error) throw error;
  return data;
};

const [positions, { refetch }] = createResource(fetchPositions);

// Usage
<Suspense fallback={<Loading />}>
  <For each={positions()}>{(p) => <Row data={p} />}</For>
</Suspense>
```

### With Parameters
```typescript
const [selectedId, setSelectedId] = createSignal<string>();

const fetchPosition = async (id: string) => {
  const { data } = await supabase
    .from("positions")
    .select("*")
    .eq("id", id)
    .single();
  return data;
};

const [position] = createResource(selectedId, fetchPosition);
```

### Global State (Stores)
```typescript
import { createStore } from "solid-js/store";

interface AppState {
  user: User | null;
  positions: Position[];
  filters: FilterState;
}

const [state, setState] = createStore<AppState>({
  user: null,
  positions: [],
  filters: { status: "open" },
});

// Update nested
setState("filters", "status", "closed");

// Update array
setState("positions", (prev) => [...prev, newPosition]);
```

### Routing
```typescript
import { Router, Route, A } from "@solidjs/router";

<Router>
  <Route path="/" component={Dashboard} />
  <Route path="/positions" component={Positions} />
  <Route path="/positions/:id" component={PositionDetail} />
  <Route path="*" component={NotFound} />
</Router>

// Link
<A href="/positions" class="...">Positions</A>

// Params
import { useParams } from "@solidjs/router";
const params = useParams<{ id: string }>();
```

### Supabase Realtime
```typescript
import { onMount, onCleanup } from "solid-js";

onMount(() => {
  const subscription = supabase
    .channel("positions")
    .on("postgres_changes", 
      { event: "*", schema: "public", table: "positions" },
      (payload) => {
        refetch(); // or update store directly
      }
    )
    .subscribe();

  onCleanup(() => subscription.unsubscribe());
});
```

## Output
1. Data fetching logic
2. Store structure (if needed)
3. Route definitions (if needed)
4. Type definitions

---

## State Persistence & Hydration

### localStorage Persistence
```typescript
import { createSignal, createEffect } from "solid-js";

// Generic persistent signal
function createPersistedSignal<T>(key: string, initialValue: T) {
  const stored = localStorage.getItem(key);
  const initial = stored ? JSON.parse(stored) : initialValue;

  const [value, setValue] = createSignal<T>(initial);

  createEffect(() => {
    localStorage.setItem(key, JSON.stringify(value()));
  });

  return [value, setValue] as const;
}

// Usage
const [filters, setFilters] = createPersistedSignal("position-filters", {
  status: "open",
  symbol: "",
  sortBy: "created_at",
});

// Clear persisted state
const clearFilters = () => {
  localStorage.removeItem("position-filters");
  setFilters({ status: "open", symbol: "", sortBy: "created_at" });
};
```

### Store Persistence
```typescript
import { createStore, reconcile } from "solid-js/store";

function createPersistedStore<T extends object>(key: string, initialValue: T) {
  const stored = localStorage.getItem(key);
  const initial = stored ? JSON.parse(stored) : initialValue;

  const [state, setState] = createStore<T>(initial);

  // Persist on changes
  createEffect(() => {
    localStorage.setItem(key, JSON.stringify(state));
  });

  // Hydrate from storage
  const hydrate = () => {
    const stored = localStorage.getItem(key);
    if (stored) {
      setState(reconcile(JSON.parse(stored)));
    }
  };

  // Reset to initial
  const reset = () => {
    localStorage.removeItem(key);
    setState(reconcile(initialValue));
  };

  return [state, setState, { hydrate, reset }] as const;
}

// Usage
const [settings, setSettings, { reset }] = createPersistedStore("user-settings", {
  theme: "system",
  compactView: false,
  showPL: true,
});
```

### Session Storage (Tab-Specific)
```typescript
function createSessionSignal<T>(key: string, initialValue: T) {
  const stored = sessionStorage.getItem(key);
  const initial = stored ? JSON.parse(stored) : initialValue;

  const [value, setValue] = createSignal<T>(initial);

  createEffect(() => {
    sessionStorage.setItem(key, JSON.stringify(value()));
  });

  return [value, setValue] as const;
}

// Usage - form draft that survives refresh but not new tab
const [formDraft, setFormDraft] = createSessionSignal("position-draft", {
  symbol: "",
  quantity: 0,
  notes: "",
});
```

### URL State Synchronization
```typescript
import { useSearchParams } from "@solidjs/router";

// Sync filters with URL
const PositionFilters: Component = () => {
  const [searchParams, setSearchParams] = useSearchParams();

  const filters = () => ({
    status: searchParams.status ?? "all",
    symbol: searchParams.symbol ?? "",
    page: parseInt(searchParams.page ?? "1"),
  });

  const setFilter = <K extends keyof ReturnType<typeof filters>>(
    key: K,
    value: ReturnType<typeof filters>[K]
  ) => {
    setSearchParams({ [key]: String(value) });
  };

  return (
    <div class="flex gap-4">
      <select
        value={filters().status}
        onChange={(e) => setFilter("status", e.currentTarget.value)}
      >
        <option value="all">All</option>
        <option value="open">Open</option>
        <option value="closed">Closed</option>
      </select>

      <input
        type="text"
        value={filters().symbol}
        onInput={(e) => setFilter("symbol", e.currentTarget.value)}
        placeholder="Filter by symbol"
      />
    </div>
  );
};

// URL becomes: /positions?status=open&symbol=AAPL&page=1
```

### Complex URL State
```typescript
import { createSignal, createEffect, on } from "solid-js";
import { useSearchParams } from "@solidjs/router";

interface TableState {
  page: number;
  pageSize: number;
  sortBy: string;
  sortOrder: "asc" | "desc";
  filters: Record<string, string>;
}

function useURLTableState(defaults: TableState) {
  const [searchParams, setSearchParams] = useSearchParams();

  const parseState = (): TableState => ({
    page: parseInt(searchParams.page ?? String(defaults.page)),
    pageSize: parseInt(searchParams.pageSize ?? String(defaults.pageSize)),
    sortBy: searchParams.sortBy ?? defaults.sortBy,
    sortOrder: (searchParams.sortOrder as "asc" | "desc") ?? defaults.sortOrder,
    filters: searchParams.filters ? JSON.parse(searchParams.filters) : defaults.filters,
  });

  const [state, setState] = createSignal(parseState());

  // Sync URL -> State when URL changes externally
  createEffect(() => {
    setState(parseState());
  });

  // Sync State -> URL
  const updateState = (updates: Partial<TableState>) => {
    const newState = { ...state(), ...updates };
    setState(newState);

    setSearchParams({
      page: String(newState.page),
      pageSize: String(newState.pageSize),
      sortBy: newState.sortBy,
      sortOrder: newState.sortOrder,
      filters: JSON.stringify(newState.filters),
    });
  };

  return [state, updateState] as const;
}
```

### Hydration from Server/API
```typescript
// Initial data from server (SSR or initial load)
interface HydrationData {
  user: User | null;
  positions: Position[];
  settings: UserSettings;
}

// Global hydration store
const [hydrationData, setHydrationData] = createStore<HydrationData>({
  user: null,
  positions: [],
  settings: defaultSettings,
});

// Hydrate on app start
const hydrateApp = async () => {
  // Check for SSR data
  const ssrData = (window as any).__INITIAL_DATA__;
  if (ssrData) {
    setHydrationData(reconcile(ssrData));
    return;
  }

  // Otherwise fetch from API
  const [user, positions, settings] = await Promise.all([
    fetchUser(),
    fetchPositions(),
    fetchSettings(),
  ]);

  setHydrationData({ user, positions, settings });
};

// Usage in App
const App: Component = () => {
  const [ready, setReady] = createSignal(false);

  onMount(async () => {
    await hydrateApp();
    setReady(true);
  });

  return (
    <Show when={ready()} fallback={<SplashScreen />}>
      <Router>
        <Routes />
      </Router>
    </Show>
  );
};
```

### Cross-Tab Synchronization
```typescript
function createSyncedSignal<T>(key: string, initialValue: T) {
  const [value, setValue] = createSignal<T>(
    JSON.parse(localStorage.getItem(key) ?? JSON.stringify(initialValue))
  );

  // Listen for changes from other tabs
  const handleStorage = (e: StorageEvent) => {
    if (e.key === key && e.newValue) {
      setValue(JSON.parse(e.newValue));
    }
  };

  onMount(() => {
    window.addEventListener("storage", handleStorage);
    onCleanup(() => window.removeEventListener("storage", handleStorage));
  });

  // Persist and notify other tabs
  const setAndSync = (newValue: T | ((prev: T) => T)) => {
    const resolved = typeof newValue === "function"
      ? (newValue as Function)(value())
      : newValue;
    setValue(() => resolved);
    localStorage.setItem(key, JSON.stringify(resolved));
  };

  return [value, setAndSync] as const;
}

// Usage - theme syncs across tabs
const [theme, setTheme] = createSyncedSignal("theme", "light");
```

---

## Optimistic Updates & Concurrency

### Basic Optimistic Update
```typescript
const [positions, setPositions] = createStore<Position[]>([]);

const updatePosition = async (id: string, updates: Partial<Position>) => {
  // Store previous state for rollback
  const previousPositions = [...positions];
  const index = positions.findIndex((p) => p.id === id);

  // Optimistically update UI
  setPositions(index, (prev) => ({ ...prev, ...updates }));

  try {
    // Sync with server
    const { error } = await supabase
      .from("positions")
      .update(updates)
      .eq("id", id);

    if (error) throw error;
  } catch (error) {
    // Rollback on failure
    setPositions(reconcile(previousPositions));
    addToast({ type: "error", message: "Failed to update position" });
  }
};
```

### Optimistic Create with Temporary ID
```typescript
const addPosition = async (data: Omit<Position, "id" | "created_at">) => {
  // Create temporary optimistic entry
  const tempId = `temp-${Date.now()}`;
  const optimisticPosition: Position = {
    ...data,
    id: tempId,
    created_at: new Date().toISOString(),
    _pending: true, // Mark as pending
  };

  // Add to UI immediately
  setPositions((prev) => [optimisticPosition, ...prev]);

  try {
    const { data: created, error } = await supabase
      .from("positions")
      .insert(data)
      .select()
      .single();

    if (error) throw error;

    // Replace temp with real data
    setPositions((prev) =>
      prev.map((p) => p.id === tempId ? created : p)
    );
  } catch (error) {
    // Remove optimistic entry
    setPositions((prev) => prev.filter((p) => p.id !== tempId));
    addToast({ type: "error", message: "Failed to create position" });
  }
};
```

### Optimistic Delete
```typescript
const deletePosition = async (id: string) => {
  // Store for potential rollback
  const deletedPosition = positions.find((p) => p.id === id);
  const deletedIndex = positions.findIndex((p) => p.id === id);

  // Optimistically remove from UI
  setPositions((prev) => prev.filter((p) => p.id !== id));

  try {
    const { error } = await supabase
      .from("positions")
      .delete()
      .eq("id", id);

    if (error) throw error;

    addToast({ type: "success", message: "Position deleted" });
  } catch (error) {
    // Rollback - reinsert at original position
    setPositions((prev) => {
      const restored = [...prev];
      restored.splice(deletedIndex, 0, deletedPosition!);
      return restored;
    });
    addToast({ type: "error", message: "Failed to delete position" });
  }
};
```

### Optimistic Queue (Multiple Operations)
```typescript
interface OptimisticOperation {
  id: string;
  type: "create" | "update" | "delete";
  data: any;
  rollback: () => void;
  timestamp: number;
}

const [pendingOps, setPendingOps] = createStore<OptimisticOperation[]>([]);

const executeWithOptimism = async (
  operation: Omit<OptimisticOperation, "id" | "timestamp">,
  serverAction: () => Promise<void>
) => {
  const opId = crypto.randomUUID();

  // Track the operation
  setPendingOps((prev) => [...prev, {
    ...operation,
    id: opId,
    timestamp: Date.now(),
  }]);

  try {
    await serverAction();
    // Success - remove from pending
    setPendingOps((prev) => prev.filter((op) => op.id !== opId));
  } catch (error) {
    // Failure - rollback and remove
    operation.rollback();
    setPendingOps((prev) => prev.filter((op) => op.id !== opId));
    throw error;
  }
};

// Show pending indicator
const PendingIndicator: Component = () => (
  <Show when={pendingOps.length > 0}>
    <div class="fixed bottom-4 left-4 bg-yellow-100 px-3 py-2 rounded shadow">
      Saving {pendingOps.length} change(s)...
    </div>
  </Show>
);
```

### Version-Based Concurrency Control
```typescript
interface VersionedEntity {
  id: string;
  version: number;
  // ... other fields
}

const updateWithVersionCheck = async (
  id: string,
  updates: Partial<Position>,
  expectedVersion: number
) => {
  const { data, error } = await supabase
    .from("positions")
    .update({ ...updates, version: expectedVersion + 1 })
    .eq("id", id)
    .eq("version", expectedVersion) // Only update if version matches
    .select()
    .single();

  if (error?.code === "PGRST116") {
    // No rows updated - version conflict
    throw new ConflictError("Position was modified by another user");
  }

  if (error) throw error;
  return data;
};

// Handle conflicts
const handleUpdate = async (id: string, updates: Partial<Position>) => {
  const position = positions.find((p) => p.id === id)!;

  try {
    const updated = await updateWithVersionCheck(id, updates, position.version);
    setPositions((p) => p.id === id, updated);
  } catch (error) {
    if (error instanceof ConflictError) {
      // Refetch and show conflict dialog
      const { data: serverData } = await supabase
        .from("positions")
        .select("*")
        .eq("id", id)
        .single();

      showConflictDialog({
        local: { ...position, ...updates },
        server: serverData,
        onResolve: (resolved) => {
          // Retry with resolved data and new version
          handleUpdate(id, resolved);
        },
      });
    } else {
      throw error;
    }
  }
};
```

### Conflict Resolution UI
```typescript
interface ConflictDialogProps {
  local: Position;
  server: Position;
  onResolve: (resolved: Partial<Position>) => void;
  onCancel: () => void;
}

const ConflictDialog: Component<ConflictDialogProps> = (props) => {
  const [resolved, setResolved] = createStore({ ...props.local });

  const changedFields = () => {
    const fields: (keyof Position)[] = [];
    for (const key of Object.keys(props.local) as (keyof Position)[]) {
      if (props.local[key] !== props.server[key]) {
        fields.push(key);
      }
    }
    return fields;
  };

  return (
    <Modal open onOpenChange={props.onCancel} title="Conflict Detected">
      <p class="text-gray-600 mb-4">
        This record was modified by another user. Choose which values to keep:
      </p>

      <div class="space-y-4">
        <For each={changedFields()}>
          {(field) => (
            <div class="border rounded p-3">
              <p class="font-medium mb-2">{field}</p>
              <div class="grid grid-cols-2 gap-2">
                <button
                  onClick={() => setResolved(field, props.local[field])}
                  class={`p-2 border rounded ${
                    resolved[field] === props.local[field] ? "border-blue-500 bg-blue-50" : ""
                  }`}
                >
                  <span class="text-xs text-gray-500">Your change</span>
                  <p>{String(props.local[field])}</p>
                </button>
                <button
                  onClick={() => setResolved(field, props.server[field])}
                  class={`p-2 border rounded ${
                    resolved[field] === props.server[field] ? "border-blue-500 bg-blue-50" : ""
                  }`}
                >
                  <span class="text-xs text-gray-500">Server value</span>
                  <p>{String(props.server[field])}</p>
                </button>
              </div>
            </div>
          )}
        </For>
      </div>

      <div class="flex justify-end gap-3 mt-6">
        <button onClick={props.onCancel} class="px-4 py-2 border rounded">
          Cancel
        </button>
        <button
          onClick={() => props.onResolve(resolved)}
          class="px-4 py-2 bg-blue-500 text-white rounded"
        >
          Save Resolved
        </button>
      </div>
    </Modal>
  );
};
```

### Retry with Exponential Backoff
```typescript
const retryWithBackoff = async <T>(
  fn: () => Promise<T>,
  maxRetries = 3,
  baseDelay = 1000
): Promise<T> => {
  let lastError: Error;

  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      lastError = error as Error;

      // Don't retry on conflict errors
      if (error instanceof ConflictError) throw error;

      // Wait with exponential backoff + jitter
      const delay = baseDelay * Math.pow(2, attempt) + Math.random() * 1000;
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }

  throw lastError!;
};

// Usage
const savePosition = async (data: Partial<Position>) => {
  return retryWithBackoff(
    () => supabase.from("positions").update(data).eq("id", data.id),
    3,
    1000
  );
};
```

### Offline Queue
```typescript
interface QueuedMutation {
  id: string;
  action: "create" | "update" | "delete";
  table: string;
  data: any;
  createdAt: number;
}

const QUEUE_KEY = "offline-mutation-queue";

const getMutationQueue = (): QueuedMutation[] => {
  return JSON.parse(localStorage.getItem(QUEUE_KEY) ?? "[]");
};

const addToQueue = (mutation: Omit<QueuedMutation, "id" | "createdAt">) => {
  const queue = getMutationQueue();
  queue.push({
    ...mutation,
    id: crypto.randomUUID(),
    createdAt: Date.now(),
  });
  localStorage.setItem(QUEUE_KEY, JSON.stringify(queue));
};

const processQueue = async () => {
  const queue = getMutationQueue();
  const failed: QueuedMutation[] = [];

  for (const mutation of queue) {
    try {
      switch (mutation.action) {
        case "create":
          await supabase.from(mutation.table).insert(mutation.data);
          break;
        case "update":
          await supabase.from(mutation.table)
            .update(mutation.data)
            .eq("id", mutation.data.id);
          break;
        case "delete":
          await supabase.from(mutation.table)
            .delete()
            .eq("id", mutation.data.id);
          break;
      }
    } catch (error) {
      failed.push(mutation);
    }
  }

  // Keep only failed mutations
  localStorage.setItem(QUEUE_KEY, JSON.stringify(failed));

  if (failed.length > 0) {
    addToast({
      type: "warning",
      message: `${failed.length} changes failed to sync`
    });
  }
};

// Process queue when coming online
window.addEventListener("online", processQueue);

// Check queue on app start
onMount(() => {
  if (navigator.onLine && getMutationQueue().length > 0) {
    processQueue();
  }
});
```
