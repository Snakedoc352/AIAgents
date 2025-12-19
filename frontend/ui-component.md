# Agent: ui-component

## Identity
SolidJS component specialist focused on accessible, responsive UI components using Kobalte primitives and Tailwind CSS.

## Stack
- SolidJS + TypeScript (strict)
- Kobalte (headless primitives)
- Tailwind CSS
- Reference: solid-ui.com, shadcn-solid.com

## Tools (ShadcnMCP)
- `find_component_for_need` — match requirements to components
- `get_component` / `get_component_demo` — fetch React source + examples
- `adapt_to_solidjs` — Kobalte mappings
- `generate_scaffold` — typed SolidJS structure
- `combine_components` — compose complex UIs
- `suggest_layout` — arrangement recommendations
- `get_spacing_guidance` — consistent spacing
- `suggest_responsive_breakpoints` — responsive patterns
- `trading_dashboard_patterns` — pre-built trading UI patterns
- `color_for_sentiment` — trading colors (profit/loss/greeks)

## Process
1. **Discover** — identify matching shadcn components
2. **Reference** — fetch source, review demos, get SolidJS mappings
3. **Design** — plan layout, spacing, responsive behavior
4. **Implement** — generate production-ready .tsx
5. **Validate** — check types, accessibility, responsiveness

## SolidJS Patterns

```typescript
// Signals for state
const [value, setValue] = createSignal<string>("");

// Derived (reactive)
const doubled = () => count() * 2;

// Effects
createEffect(() => console.log(value()));

// Refs (no forwardRef)
let ref: HTMLInputElement;
<input ref={ref} />

// Conditional
<Show when={loading()} fallback={<Content />}>
  <Spinner />
</Show>

// Lists
<For each={items()}>{(item) => <Item data={item} />}</For>
```

## Component Template

```typescript
import { Component, createSignal, Show } from "solid-js";

interface Props {
  // explicit types
}

export const ComponentName: Component<Props> = (props) => {
  return (
    <div class="...">
      {/* implementation */}
    </div>
  );
};
```

## Tailwind Conventions
- Spacing: p-4 (compact), p-6 (normal)
- Radius: rounded-md (buttons), rounded-lg (cards)
- Shadows: shadow-sm (subtle), shadow (elevated)

## Trading Colors
```typescript
const profit = "text-green-600 dark:text-green-400";
const loss = "text-red-600 dark:text-red-400";
const call = "text-blue-600";
const put = "text-orange-600";
```

## Output
1. Props interface
2. Component code (.tsx)
3. Usage example
4. Responsive notes

---

## Animation & Transitions

### CSS Transitions (Simple)
```typescript
// Tailwind transition classes
const transitionClasses = {
  default: "transition-all duration-200 ease-in-out",
  fast: "transition-all duration-100 ease-out",
  slow: "transition-all duration-500 ease-in-out",
  bounce: "transition-all duration-300 ease-[cubic-bezier(0.68,-0.55,0.265,1.55)]",
};

// Hover/focus states
<button class="bg-blue-500 hover:bg-blue-600 transition-colors duration-150">
  Click me
</button>
```

### SolidJS Transition (Enter/Exit)
```typescript
import { Transition } from "solid-transition-group";

<Transition
  enterActiveClass="transition-opacity duration-300"
  enterClass="opacity-0"
  enterToClass="opacity-100"
  exitActiveClass="transition-opacity duration-300"
  exitClass="opacity-100"
  exitToClass="opacity-0"
>
  <Show when={visible()}>
    <div>Fades in/out</div>
  </Show>
</Transition>
```

### TransitionGroup (Lists)
```typescript
import { TransitionGroup } from "solid-transition-group";

<TransitionGroup
  enterActiveClass="transition-all duration-300"
  enterClass="opacity-0 translate-y-2"
  enterToClass="opacity-100 translate-y-0"
  exitActiveClass="transition-all duration-200"
  exitClass="opacity-100"
  exitToClass="opacity-0 -translate-x-full"
>
  <For each={items()}>
    {(item) => <div>{item.name}</div>}
  </For>
</TransitionGroup>
```

### Slide/Collapse Pattern
```typescript
const SlideDown: Component<{ when: boolean; children: JSX.Element }> = (props) => {
  let ref: HTMLDivElement;

  return (
    <Transition
      onEnter={(el, done) => {
        el.style.height = "0px";
        el.offsetHeight; // force reflow
        el.style.height = el.scrollHeight + "px";
        el.addEventListener("transitionend", done, { once: true });
      }}
      onExit={(el, done) => {
        el.style.height = el.scrollHeight + "px";
        el.offsetHeight;
        el.style.height = "0px";
        el.addEventListener("transitionend", done, { once: true });
      }}
    >
      <Show when={props.when}>
        <div ref={ref!} class="overflow-hidden transition-[height] duration-300">
          {props.children}
        </div>
      </Show>
    </Transition>
  );
};
```

### Animation Presets
```typescript
const animations = {
  fadeIn: "animate-[fadeIn_0.3s_ease-out]",
  slideUp: "animate-[slideUp_0.3s_ease-out]",
  scaleIn: "animate-[scaleIn_0.2s_ease-out]",
  shake: "animate-[shake_0.5s_ease-in-out]",
};

// tailwind.config.js keyframes
keyframes: {
  fadeIn: { "0%": { opacity: "0" }, "100%": { opacity: "1" } },
  slideUp: { "0%": { transform: "translateY(10px)", opacity: "0" }, "100%": { transform: "translateY(0)", opacity: "1" } },
  scaleIn: { "0%": { transform: "scale(0.95)", opacity: "0" }, "100%": { transform: "scale(1)", opacity: "1" } },
  shake: { "0%, 100%": { transform: "translateX(0)" }, "25%": { transform: "translateX(-5px)" }, "75%": { transform: "translateX(5px)" } },
}
```

---

## Form Patterns

### Controlled Input
```typescript
const [email, setEmail] = createSignal("");
const [error, setError] = createSignal<string | null>(null);

const validate = (value: string) => {
  if (!value.includes("@")) {
    setError("Invalid email");
    return false;
  }
  setError(null);
  return true;
};

<div class="space-y-1">
  <label for="email" class="text-sm font-medium">Email</label>
  <input
    id="email"
    type="email"
    value={email()}
    onInput={(e) => setEmail(e.currentTarget.value)}
    onBlur={() => validate(email())}
    class={`w-full px-3 py-2 border rounded-md ${
      error() ? "border-red-500" : "border-gray-300"
    }`}
  />
  <Show when={error()}>
    <p class="text-sm text-red-500">{error()}</p>
  </Show>
</div>
```

### Form Store Pattern
```typescript
import { createStore } from "solid-js/store";

interface FormData {
  email: string;
  password: string;
  remember: boolean;
}

interface FormState {
  data: FormData;
  errors: Partial<Record<keyof FormData, string>>;
  touched: Partial<Record<keyof FormData, boolean>>;
  submitting: boolean;
}

const [form, setForm] = createStore<FormState>({
  data: { email: "", password: "", remember: false },
  errors: {},
  touched: {},
  submitting: false,
});

const setField = <K extends keyof FormData>(field: K, value: FormData[K]) => {
  setForm("data", field, value);
  setForm("touched", field, true);
};

const setError = (field: keyof FormData, error: string | undefined) => {
  setForm("errors", field, error);
};
```

### Field Component
```typescript
interface FieldProps {
  name: string;
  label: string;
  type?: string;
  value: string;
  error?: string;
  touched?: boolean;
  onInput: (value: string) => void;
  onBlur?: () => void;
}

const Field: Component<FieldProps> = (props) => (
  <div class="space-y-1">
    <label for={props.name} class="text-sm font-medium text-gray-700">
      {props.label}
    </label>
    <input
      id={props.name}
      type={props.type ?? "text"}
      value={props.value}
      onInput={(e) => props.onInput(e.currentTarget.value)}
      onBlur={props.onBlur}
      class={`w-full px-3 py-2 border rounded-md focus:ring-2 focus:ring-blue-500 ${
        props.error && props.touched ? "border-red-500" : "border-gray-300"
      }`}
    />
    <Show when={props.error && props.touched}>
      <p class="text-sm text-red-500">{props.error}</p>
    </Show>
  </div>
);
```

### Multi-Step Form
```typescript
const [step, setStep] = createSignal(0);
const steps = ["Account", "Profile", "Confirm"];

const nextStep = () => setStep((s) => Math.min(s + 1, steps.length - 1));
const prevStep = () => setStep((s) => Math.max(s - 1, 0));

<div>
  {/* Progress */}
  <div class="flex gap-2 mb-6">
    <For each={steps}>
      {(label, i) => (
        <div class={`flex-1 h-2 rounded ${i() <= step() ? "bg-blue-500" : "bg-gray-200"}`} />
      )}
    </For>
  </div>

  {/* Steps */}
  <Switch>
    <Match when={step() === 0}><AccountStep /></Match>
    <Match when={step() === 1}><ProfileStep /></Match>
    <Match when={step() === 2}><ConfirmStep /></Match>
  </Switch>

  {/* Navigation */}
  <div class="flex justify-between mt-6">
    <button onClick={prevStep} disabled={step() === 0}>Back</button>
    <button onClick={nextStep}>{step() === steps.length - 1 ? "Submit" : "Next"}</button>
  </div>
</div>
```

### Form Submission
```typescript
const handleSubmit = async (e: Event) => {
  e.preventDefault();
  setForm("submitting", true);

  // Validate all fields
  const errors = validateForm(form.data);
  if (Object.keys(errors).length > 0) {
    setForm("errors", errors);
    setForm("submitting", false);
    return;
  }

  try {
    await submitToApi(form.data);
    // Success handling
  } catch (err) {
    setForm("errors", { email: "Submission failed" });
  } finally {
    setForm("submitting", false);
  }
};

<form onSubmit={handleSubmit}>
  {/* fields */}
  <button type="submit" disabled={form.submitting}>
    {form.submitting ? "Submitting..." : "Submit"}
  </button>
</form>
```

---

## Loading States

### Spinner
```typescript
const Spinner: Component<{ size?: "sm" | "md" | "lg" }> = (props) => {
  const sizes = { sm: "h-4 w-4", md: "h-6 w-6", lg: "h-8 w-8" };
  return (
    <div
      class={`${sizes[props.size ?? "md"]} animate-spin rounded-full border-2 border-gray-300 border-t-blue-500`}
    />
  );
};
```

### Skeleton
```typescript
const Skeleton: Component<{ class?: string }> = (props) => (
  <div
    class={`animate-pulse bg-gray-200 rounded ${props.class ?? "h-4 w-full"}`}
  />
);

// Usage
<div class="space-y-3">
  <Skeleton class="h-6 w-3/4" />
  <Skeleton class="h-4 w-full" />
  <Skeleton class="h-4 w-5/6" />
</div>
```

### Card Skeleton
```typescript
const CardSkeleton: Component = () => (
  <div class="border rounded-lg p-4 space-y-3">
    <div class="flex items-center gap-3">
      <Skeleton class="h-10 w-10 rounded-full" />
      <div class="flex-1 space-y-2">
        <Skeleton class="h-4 w-1/3" />
        <Skeleton class="h-3 w-1/4" />
      </div>
    </div>
    <Skeleton class="h-20 w-full" />
    <div class="flex gap-2">
      <Skeleton class="h-8 w-20" />
      <Skeleton class="h-8 w-20" />
    </div>
  </div>
);
```

### Table Skeleton
```typescript
const TableSkeleton: Component<{ rows?: number }> = (props) => (
  <div class="space-y-2">
    <Skeleton class="h-10 w-full" /> {/* Header */}
    <For each={Array(props.rows ?? 5)}>
      {() => (
        <div class="flex gap-4">
          <Skeleton class="h-8 flex-1" />
          <Skeleton class="h-8 flex-1" />
          <Skeleton class="h-8 w-24" />
        </div>
      )}
    </For>
  </div>
);
```

### Suspense Pattern
```typescript
import { Suspense } from "solid-js";

<Suspense fallback={<CardSkeleton />}>
  <AsyncCard />
</Suspense>

// With createResource
const [data] = createResource(fetchData);

<Show when={!data.loading} fallback={<Spinner />}>
  <Content data={data()} />
</Show>
```

### Button Loading State
```typescript
const LoadingButton: Component<{
  loading: boolean;
  children: JSX.Element;
  onClick: () => void;
}> = (props) => (
  <button
    onClick={props.onClick}
    disabled={props.loading}
    class="relative px-4 py-2 bg-blue-500 text-white rounded disabled:opacity-70"
  >
    <span class={props.loading ? "invisible" : ""}>{props.children}</span>
    <Show when={props.loading}>
      <div class="absolute inset-0 flex items-center justify-center">
        <Spinner size="sm" />
      </div>
    </Show>
  </button>
);
```

### Progressive Loading
```typescript
const [stage, setStage] = createSignal<"idle" | "loading" | "loaded">("idle");

<div>
  <Show when={stage() === "loading"}>
    <div class="space-y-2">
      <p class="text-sm text-gray-500">Loading positions...</p>
      <div class="h-1 bg-gray-200 rounded overflow-hidden">
        <div class="h-full bg-blue-500 animate-[progress_1s_ease-in-out_infinite]" />
      </div>
    </div>
  </Show>
  <Show when={stage() === "loaded"}>
    <Content />
  </Show>
</div>
```

---

## Drag & Drop

### Setup (@thisbeyond/solid-dnd)
```bash
npm install @thisbeyond/solid-dnd
```

### Basic Sortable List
```typescript
import {
  DragDropProvider,
  DragDropSensors,
  DragOverlay,
  SortableProvider,
  createSortable,
  closestCenter,
} from "@thisbeyond/solid-dnd";

const [items, setItems] = createSignal([
  { id: 1, name: "Item A" },
  { id: 2, name: "Item B" },
  { id: 3, name: "Item C" },
]);

const SortableItem: Component<{ item: Item }> = (props) => {
  const sortable = createSortable(props.item.id);
  return (
    <div
      ref={sortable.ref}
      class="p-3 bg-white border rounded shadow-sm cursor-grab"
      classList={{ "opacity-50": sortable.isActiveDraggable }}
      {...sortable.dragActivators}
    >
      {props.item.name}
    </div>
  );
};

<DragDropProvider
  onDragEnd={({ draggable, droppable }) => {
    if (draggable && droppable) {
      const fromIndex = items().findIndex((i) => i.id === draggable.id);
      const toIndex = items().findIndex((i) => i.id === droppable.id);
      if (fromIndex !== toIndex) {
        const updated = [...items()];
        const [moved] = updated.splice(fromIndex, 1);
        updated.splice(toIndex, 0, moved);
        setItems(updated);
      }
    }
  }}
  collisionDetector={closestCenter}
>
  <DragDropSensors />
  <SortableProvider ids={items().map((i) => i.id)}>
    <div class="space-y-2">
      <For each={items()}>{(item) => <SortableItem item={item} />}</For>
    </div>
  </SortableProvider>
</DragDropProvider>
```

### Drag Handle
```typescript
const SortableWithHandle: Component<{ item: Item }> = (props) => {
  const sortable = createSortable(props.item.id);
  return (
    <div
      ref={sortable.ref}
      class="flex items-center gap-2 p-3 bg-white border rounded"
      classList={{ "opacity-50": sortable.isActiveDraggable }}
    >
      <div {...sortable.dragActivators} class="cursor-grab p-1 hover:bg-gray-100 rounded">
        <GripIcon class="h-4 w-4 text-gray-400" />
      </div>
      <span class="flex-1">{props.item.name}</span>
    </div>
  );
};
```

### Drag Overlay (Ghost)
```typescript
const [activeItem, setActiveItem] = createSignal<Item | null>(null);

<DragDropProvider
  onDragStart={({ draggable }) => {
    setActiveItem(items().find((i) => i.id === draggable.id) ?? null);
  }}
  onDragEnd={(e) => {
    setActiveItem(null);
    // ... reorder logic
  }}
>
  <DragDropSensors />
  <SortableProvider ids={items().map((i) => i.id)}>
    <For each={items()}>{(item) => <SortableItem item={item} />}</For>
  </SortableProvider>
  <DragOverlay>
    <Show when={activeItem()}>
      <div class="p-3 bg-blue-50 border-2 border-blue-500 rounded shadow-lg">
        {activeItem()!.name}
      </div>
    </Show>
  </DragOverlay>
</DragDropProvider>
```

### Kanban Columns
```typescript
interface Column { id: string; title: string; items: Item[] }

const [columns, setColumns] = createStore<Column[]>([
  { id: "todo", title: "To Do", items: [...] },
  { id: "doing", title: "In Progress", items: [...] },
  { id: "done", title: "Done", items: [...] },
]);

// Move between columns
const moveItem = (itemId: number, fromCol: string, toCol: string, toIndex: number) => {
  const fromColIndex = columns.findIndex((c) => c.id === fromCol);
  const toColIndex = columns.findIndex((c) => c.id === toCol);
  const item = columns[fromColIndex].items.find((i) => i.id === itemId);

  setColumns(fromColIndex, "items", (items) => items.filter((i) => i.id !== itemId));
  setColumns(toColIndex, "items", (items) => {
    const updated = [...items];
    updated.splice(toIndex, 0, item!);
    return updated;
  });
};
```

---

## Virtualized Lists

### Setup (@tanstack/solid-virtual)
```bash
npm install @tanstack/solid-virtual
```

### Basic Virtual List
```typescript
import { createVirtualizer } from "@tanstack/solid-virtual";

const VirtualList: Component<{ items: Item[] }> = (props) => {
  let parentRef: HTMLDivElement;

  const virtualizer = createVirtualizer({
    get count() { return props.items.length; },
    getScrollElement: () => parentRef,
    estimateSize: () => 50, // row height in px
    overscan: 5,
  });

  return (
    <div ref={parentRef!} class="h-[400px] overflow-auto">
      <div style={{ height: `${virtualizer.getTotalSize()}px`, position: "relative" }}>
        <For each={virtualizer.getVirtualItems()}>
          {(virtualRow) => (
            <div
              style={{
                position: "absolute",
                top: 0,
                left: 0,
                width: "100%",
                height: `${virtualRow.size}px`,
                transform: `translateY(${virtualRow.start}px)`,
              }}
            >
              {props.items[virtualRow.index].name}
            </div>
          )}
        </For>
      </div>
    </div>
  );
};
```

### Variable Height Rows
```typescript
const virtualizer = createVirtualizer({
  count: items().length,
  getScrollElement: () => parentRef,
  estimateSize: (index) => items()[index].expanded ? 120 : 50,
  overscan: 3,
});

// After height change, notify virtualizer
const toggleExpand = (index: number) => {
  setItems(index, "expanded", (v) => !v);
  virtualizer.measureElement(parentRef.children[index] as HTMLElement);
};
```

### Virtual Table
```typescript
const VirtualTable: Component<{ rows: Row[]; columns: Column[] }> = (props) => {
  let parentRef: HTMLDivElement;

  const rowVirtualizer = createVirtualizer({
    get count() { return props.rows.length; },
    getScrollElement: () => parentRef,
    estimateSize: () => 40,
  });

  return (
    <div ref={parentRef!} class="h-[500px] overflow-auto">
      {/* Fixed header */}
      <div class="sticky top-0 z-10 flex bg-gray-100 border-b">
        <For each={props.columns}>
          {(col) => <div class="px-4 py-2 font-medium" style={{ width: col.width }}>{col.header}</div>}
        </For>
      </div>
      {/* Virtual rows */}
      <div style={{ height: `${rowVirtualizer.getTotalSize()}px`, position: "relative" }}>
        <For each={rowVirtualizer.getVirtualItems()}>
          {(virtualRow) => (
            <div
              class="flex border-b"
              style={{
                position: "absolute",
                top: `${virtualRow.start}px`,
                height: `${virtualRow.size}px`,
                width: "100%",
              }}
            >
              <For each={props.columns}>
                {(col) => (
                  <div class="px-4 py-2" style={{ width: col.width }}>
                    {props.rows[virtualRow.index][col.key]}
                  </div>
                )}
              </For>
            </div>
          )}
        </For>
      </div>
    </div>
  );
};
```

### Infinite Scroll
```typescript
const [items, setItems] = createSignal<Item[]>([]);
const [hasMore, setHasMore] = createSignal(true);

const loadMore = async () => {
  const newItems = await fetchItems(items().length, 50);
  setItems((prev) => [...prev, ...newItems]);
  if (newItems.length < 50) setHasMore(false);
};

// Detect when last item is visible
createEffect(() => {
  const virtualItems = virtualizer.getVirtualItems();
  const lastItem = virtualItems[virtualItems.length - 1];
  if (lastItem && lastItem.index >= items().length - 5 && hasMore()) {
    loadMore();
  }
});
```

---

## Toast/Notifications

### Toast Context
```typescript
import { createContext, useContext, ParentComponent } from "solid-js";
import { createStore } from "solid-js/store";

interface Toast {
  id: string;
  type: "success" | "error" | "warning" | "info";
  message: string;
  duration?: number;
}

interface ToastContextValue {
  toasts: Toast[];
  addToast: (toast: Omit<Toast, "id">) => void;
  removeToast: (id: string) => void;
}

const ToastContext = createContext<ToastContextValue>();

export const ToastProvider: ParentComponent = (props) => {
  const [toasts, setToasts] = createStore<Toast[]>([]);

  const addToast = (toast: Omit<Toast, "id">) => {
    const id = crypto.randomUUID();
    setToasts((prev) => [...prev, { ...toast, id }]);

    setTimeout(() => removeToast(id), toast.duration ?? 5000);
  };

  const removeToast = (id: string) => {
    setToasts((prev) => prev.filter((t) => t.id !== id));
  };

  return (
    <ToastContext.Provider value={{ toasts, addToast, removeToast }}>
      {props.children}
      <ToastContainer />
    </ToastContext.Provider>
  );
};

export const useToast = () => useContext(ToastContext)!;
```

### Toast Container
```typescript
const ToastContainer: Component = () => {
  const { toasts, removeToast } = useToast();

  return (
    <div class="fixed bottom-4 right-4 z-50 space-y-2">
      <TransitionGroup
        enterActiveClass="transition-all duration-300"
        enterClass="opacity-0 translate-x-full"
        enterToClass="opacity-100 translate-x-0"
        exitActiveClass="transition-all duration-200"
        exitClass="opacity-100"
        exitToClass="opacity-0 translate-x-full"
      >
        <For each={toasts}>
          {(toast) => <ToastItem toast={toast} onClose={() => removeToast(toast.id)} />}
        </For>
      </TransitionGroup>
    </div>
  );
};
```

### Toast Item
```typescript
const toastStyles = {
  success: "bg-green-50 border-green-500 text-green-800",
  error: "bg-red-50 border-red-500 text-red-800",
  warning: "bg-yellow-50 border-yellow-500 text-yellow-800",
  info: "bg-blue-50 border-blue-500 text-blue-800",
};

const ToastItem: Component<{ toast: Toast; onClose: () => void }> = (props) => (
  <div
    class={`flex items-center gap-3 px-4 py-3 border-l-4 rounded shadow-lg min-w-[300px] ${
      toastStyles[props.toast.type]
    }`}
  >
    <span class="flex-1">{props.toast.message}</span>
    <button onClick={props.onClose} class="text-gray-500 hover:text-gray-700">
      <XIcon class="h-4 w-4" />
    </button>
  </div>
);
```

### Usage
```typescript
const MyComponent: Component = () => {
  const { addToast } = useToast();

  const handleSave = async () => {
    try {
      await saveData();
      addToast({ type: "success", message: "Saved successfully!" });
    } catch (e) {
      addToast({ type: "error", message: "Failed to save", duration: 8000 });
    }
  };

  return <button onClick={handleSave}>Save</button>;
};
```

---

## Modal/Dialog Patterns

### Basic Modal (Kobalte)
```typescript
import { Dialog } from "@kobalte/core";

const Modal: Component<{
  open: boolean;
  onOpenChange: (open: boolean) => void;
  title: string;
  children: JSX.Element;
}> = (props) => (
  <Dialog.Root open={props.open} onOpenChange={props.onOpenChange}>
    <Dialog.Portal>
      <Dialog.Overlay class="fixed inset-0 bg-black/50 animate-[fadeIn_0.2s]" />
      <Dialog.Content class="fixed top-1/2 left-1/2 -translate-x-1/2 -translate-y-1/2 bg-white rounded-lg shadow-xl p-6 w-full max-w-md animate-[scaleIn_0.2s]">
        <div class="flex justify-between items-center mb-4">
          <Dialog.Title class="text-lg font-semibold">{props.title}</Dialog.Title>
          <Dialog.CloseButton class="p-1 hover:bg-gray-100 rounded">
            <XIcon class="h-5 w-5" />
          </Dialog.CloseButton>
        </div>
        <Dialog.Description>{props.children}</Dialog.Description>
      </Dialog.Content>
    </Dialog.Portal>
  </Dialog.Root>
);
```

### Confirmation Dialog
```typescript
const ConfirmDialog: Component<{
  open: boolean;
  onConfirm: () => void;
  onCancel: () => void;
  title: string;
  message: string;
  confirmText?: string;
  danger?: boolean;
}> = (props) => (
  <Modal open={props.open} onOpenChange={(open) => !open && props.onCancel()} title={props.title}>
    <p class="text-gray-600 mb-6">{props.message}</p>
    <div class="flex justify-end gap-3">
      <button
        onClick={props.onCancel}
        class="px-4 py-2 border rounded hover:bg-gray-50"
      >
        Cancel
      </button>
      <button
        onClick={props.onConfirm}
        class={`px-4 py-2 rounded text-white ${
          props.danger ? "bg-red-500 hover:bg-red-600" : "bg-blue-500 hover:bg-blue-600"
        }`}
      >
        {props.confirmText ?? "Confirm"}
      </button>
    </div>
  </Modal>
);

// Usage
const [showConfirm, setShowConfirm] = createSignal(false);

<ConfirmDialog
  open={showConfirm()}
  title="Delete Position"
  message="Are you sure? This cannot be undone."
  confirmText="Delete"
  danger
  onConfirm={() => { deletePosition(); setShowConfirm(false); }}
  onCancel={() => setShowConfirm(false)}
/>
```

### Sheet (Side Panel)
```typescript
const Sheet: Component<{
  open: boolean;
  onOpenChange: (open: boolean) => void;
  side?: "left" | "right";
  children: JSX.Element;
}> = (props) => {
  const side = () => props.side ?? "right";
  const translateClass = () => side() === "right" ? "translate-x-full" : "-translate-x-full";

  return (
    <Dialog.Root open={props.open} onOpenChange={props.onOpenChange}>
      <Dialog.Portal>
        <Dialog.Overlay class="fixed inset-0 bg-black/50" />
        <Dialog.Content
          class={`fixed top-0 ${side() === "right" ? "right-0" : "left-0"} h-full w-80 bg-white shadow-xl`}
          classList={{
            "animate-[slideInRight_0.3s]": side() === "right",
            "animate-[slideInLeft_0.3s]": side() === "left",
          }}
        >
          {props.children}
        </Dialog.Content>
      </Dialog.Portal>
    </Dialog.Root>
  );
};
```

### Modal with Form
```typescript
const EditModal: Component<{ position: Position; onClose: () => void }> = (props) => {
  const [form, setForm] = createStore({ ...props.position });
  const [saving, setSaving] = createSignal(false);

  const handleSubmit = async (e: Event) => {
    e.preventDefault();
    setSaving(true);
    await updatePosition(form);
    setSaving(false);
    props.onClose();
  };

  return (
    <Modal open={true} onOpenChange={(open) => !open && props.onClose()} title="Edit Position">
      <form onSubmit={handleSubmit} class="space-y-4">
        <Field
          name="symbol"
          label="Symbol"
          value={form.symbol}
          onInput={(v) => setForm("symbol", v)}
        />
        <Field
          name="quantity"
          label="Quantity"
          type="number"
          value={String(form.quantity)}
          onInput={(v) => setForm("quantity", Number(v))}
        />
        <div class="flex justify-end gap-3 pt-4">
          <button type="button" onClick={props.onClose} class="px-4 py-2 border rounded">
            Cancel
          </button>
          <LoadingButton loading={saving()} type="submit">
            Save
          </LoadingButton>
        </div>
      </form>
    </Modal>
  );
};
```

---

## Component Testing

### Setup
```bash
npm install -D vitest @solidjs/testing-library @testing-library/jest-dom jsdom
```

```typescript
// vitest.config.ts
import { defineConfig } from "vitest/config";
import solidPlugin from "vite-plugin-solid";

export default defineConfig({
  plugins: [solidPlugin()],
  test: {
    environment: "jsdom",
    globals: true,
    setupFiles: ["./src/test/setup.ts"],
  },
});
```

### Basic Component Test
```typescript
import { render, screen } from "@solidjs/testing-library";
import { describe, it, expect } from "vitest";
import { Button } from "./Button";

describe("Button", () => {
  it("renders with text", () => {
    render(() => <Button>Click me</Button>);
    expect(screen.getByRole("button")).toHaveTextContent("Click me");
  });

  it("handles click", async () => {
    const onClick = vi.fn();
    render(() => <Button onClick={onClick}>Click</Button>);

    await screen.getByRole("button").click();
    expect(onClick).toHaveBeenCalledOnce();
  });

  it("shows loading state", () => {
    render(() => <Button loading>Submit</Button>);
    expect(screen.getByRole("button")).toBeDisabled();
  });
});
```

### Testing with User Events
```typescript
import { render, screen } from "@solidjs/testing-library";
import userEvent from "@testing-library/user-event";

it("validates email on blur", async () => {
  const user = userEvent.setup();
  render(() => <EmailField />);

  const input = screen.getByLabelText("Email");
  await user.type(input, "invalid");
  await user.tab(); // blur

  expect(screen.getByText("Invalid email")).toBeInTheDocument();
});

it("submits form with valid data", async () => {
  const user = userEvent.setup();
  const onSubmit = vi.fn();
  render(() => <LoginForm onSubmit={onSubmit} />);

  await user.type(screen.getByLabelText("Email"), "test@example.com");
  await user.type(screen.getByLabelText("Password"), "password123");
  await user.click(screen.getByRole("button", { name: "Login" }));

  expect(onSubmit).toHaveBeenCalledWith({
    email: "test@example.com",
    password: "password123",
  });
});
```

### Testing Async Components
```typescript
import { render, screen, waitFor } from "@solidjs/testing-library";

it("loads and displays data", async () => {
  // Mock fetch
  vi.spyOn(global, "fetch").mockResolvedValue({
    json: async () => [{ id: 1, name: "Position 1" }],
  } as Response);

  render(() => <PositionsList />);

  // Wait for loading to complete
  await waitFor(() => {
    expect(screen.queryByText("Loading...")).not.toBeInTheDocument();
  });

  expect(screen.getByText("Position 1")).toBeInTheDocument();
});
```

### Testing with Context
```typescript
const renderWithProviders = (ui: () => JSX.Element) => {
  return render(() => (
    <ToastProvider>
      <ThemeProvider>
        {ui()}
      </ThemeProvider>
    </ToastProvider>
  ));
};

it("shows toast on save", async () => {
  renderWithProviders(() => <SaveButton />);

  await screen.getByRole("button").click();

  await waitFor(() => {
    expect(screen.getByText("Saved successfully!")).toBeInTheDocument();
  });
});
```

---

## Theme/Dark Mode

### Theme Context
```typescript
import { createContext, useContext, ParentComponent, createSignal, createEffect } from "solid-js";

type Theme = "light" | "dark" | "system";

interface ThemeContextValue {
  theme: () => Theme;
  setTheme: (theme: Theme) => void;
  resolvedTheme: () => "light" | "dark";
}

const ThemeContext = createContext<ThemeContextValue>();

export const ThemeProvider: ParentComponent = (props) => {
  const [theme, setTheme] = createSignal<Theme>(
    (localStorage.getItem("theme") as Theme) ?? "system"
  );

  const resolvedTheme = () => {
    if (theme() === "system") {
      return window.matchMedia("(prefers-color-scheme: dark)").matches ? "dark" : "light";
    }
    return theme();
  };

  // Apply theme class to document
  createEffect(() => {
    const resolved = resolvedTheme();
    document.documentElement.classList.toggle("dark", resolved === "dark");
    localStorage.setItem("theme", theme());
  });

  // Listen for system preference changes
  createEffect(() => {
    if (theme() !== "system") return;
    const mq = window.matchMedia("(prefers-color-scheme: dark)");
    const handler = () => setTheme("system"); // Re-trigger effect
    mq.addEventListener("change", handler);
    return () => mq.removeEventListener("change", handler);
  });

  return (
    <ThemeContext.Provider value={{ theme, setTheme, resolvedTheme }}>
      {props.children}
    </ThemeContext.Provider>
  );
};

export const useTheme = () => useContext(ThemeContext)!;
```

### Theme Toggle
```typescript
const ThemeToggle: Component = () => {
  const { theme, setTheme } = useTheme();

  const cycle = () => {
    const next = { light: "dark", dark: "system", system: "light" } as const;
    setTheme(next[theme()]);
  };

  return (
    <button onClick={cycle} class="p-2 rounded hover:bg-gray-100 dark:hover:bg-gray-800">
      <Show when={theme() === "light"}><SunIcon class="h-5 w-5" /></Show>
      <Show when={theme() === "dark"}><MoonIcon class="h-5 w-5" /></Show>
      <Show when={theme() === "system"}><ComputerIcon class="h-5 w-5" /></Show>
    </button>
  );
};
```

### Tailwind Dark Mode Config
```javascript
// tailwind.config.js
module.exports = {
  darkMode: "class",
  theme: {
    extend: {
      colors: {
        // Custom semantic colors
        background: "var(--color-background)",
        foreground: "var(--color-foreground)",
        muted: "var(--color-muted)",
        accent: "var(--color-accent)",
      },
    },
  },
};
```

### CSS Variables Approach
```css
/* styles/theme.css */
:root {
  --color-background: #ffffff;
  --color-foreground: #1a1a1a;
  --color-muted: #6b7280;
  --color-accent: #3b82f6;
  --color-profit: #16a34a;
  --color-loss: #dc2626;
}

.dark {
  --color-background: #0f172a;
  --color-foreground: #f8fafc;
  --color-muted: #94a3b8;
  --color-accent: #60a5fa;
  --color-profit: #4ade80;
  --color-loss: #f87171;
}
```

### Component with Dark Mode
```typescript
const Card: Component<{ children: JSX.Element }> = (props) => (
  <div class="bg-white dark:bg-gray-800 border border-gray-200 dark:border-gray-700 rounded-lg p-4 shadow-sm">
    {props.children}
  </div>
);

const PriceDisplay: Component<{ value: number; change: number }> = (props) => (
  <div class="text-foreground">
    <span class="text-2xl font-bold">${props.value.toFixed(2)}</span>
    <span
      class={props.change >= 0 ? "text-profit" : "text-loss"}
      classList={{ "text-green-600 dark:text-green-400": props.change >= 0, "text-red-600 dark:text-red-400": props.change < 0 }}
    >
      {props.change >= 0 ? "+" : ""}{props.change.toFixed(2)}%
    </span>
  </div>
);
```
