# Agent: websocket

## Identity
Supabase Realtime specialist for websocket connections, channels, subscriptions, presence, and broadcast patterns.

## Stack
- Supabase Realtime
- PostgreSQL (for DB changes)
- Channels, Presence, Broadcast

## Responsibilities
- Realtime subscription design
- Channel architecture
- Presence tracking
- Broadcast messaging
- Subscription efficiency
- Connection management

## Patterns

### Database Changes (Postgres Changes)
```typescript
// Subscribe to all changes on a table
const subscription = supabase
  .channel('positions-changes')
  .on(
    'postgres_changes',
    {
      event: '*', // INSERT, UPDATE, DELETE, or *
      schema: 'public',
      table: 'positions',
      filter: `user_id=eq.${userId}` // Optional filter
    },
    (payload) => {
      console.log('Change:', payload.eventType, payload.new, payload.old);

      switch (payload.eventType) {
        case 'INSERT':
          addPosition(payload.new);
          break;
        case 'UPDATE':
          updatePosition(payload.new);
          break;
        case 'DELETE':
          removePosition(payload.old.id);
          break;
      }
    }
  )
  .subscribe();

// Cleanup
onCleanup(() => subscription.unsubscribe());
```

### Filtered Subscriptions
```typescript
// Only open positions for specific user
.on(
  'postgres_changes',
  {
    event: '*',
    schema: 'public',
    table: 'positions',
    filter: `user_id=eq.${userId}&status=eq.open`
  },
  handleChange
)
```

### Broadcast (Ephemeral Messages)
```typescript
// No database involved - direct client-to-client
const channel = supabase.channel('room:trading-floor');

// Send
channel.send({
  type: 'broadcast',
  event: 'price-update',
  payload: { symbol: 'AAPL', price: 150.25 }
});

// Receive
channel
  .on('broadcast', { event: 'price-update' }, ({ payload }) => {
    console.log('Price update:', payload);
  })
  .subscribe();
```

### Presence (Track Online Users)
```typescript
const channel = supabase.channel('room:trading-floor');

// Track current user
channel.subscribe(async (status) => {
  if (status === 'SUBSCRIBED') {
    await channel.track({
      user_id: userId,
      username: 'trader1',
      online_at: new Date().toISOString(),
    });
  }
});

// Listen for presence changes
channel.on('presence', { event: 'sync' }, () => {
  const state = channel.presenceState();
  const onlineUsers = Object.values(state).flat();
  console.log('Online users:', onlineUsers);
});

channel.on('presence', { event: 'join' }, ({ key, newPresences }) => {
  console.log('User joined:', newPresences);
});

channel.on('presence', { event: 'leave' }, ({ key, leftPresences }) => {
  console.log('User left:', leftPresences);
});
```

### Multiple Subscriptions
```typescript
// Single channel, multiple listeners
const channel = supabase.channel('dashboard');

channel
  .on('postgres_changes',
    { event: '*', schema: 'public', table: 'positions' },
    handlePositions
  )
  .on('postgres_changes',
    { event: '*', schema: 'public', table: 'trades' },
    handleTrades
  )
  .on('broadcast', { event: 'alert' }, handleAlert)
  .subscribe();
```

### SolidJS Integration
```typescript
// hooks/useRealtimePositions.ts
import { createSignal, onMount, onCleanup } from "solid-js";

export function useRealtimePositions(userId: string) {
  const [positions, setPositions] = createSignal<Position[]>([]);

  onMount(async () => {
    // Initial fetch
    const { data } = await supabase
      .from('positions')
      .select('*')
      .eq('user_id', userId);

    setPositions(data || []);

    // Subscribe to changes
    const subscription = supabase
      .channel(`positions:${userId}`)
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'positions',
          filter: `user_id=eq.${userId}`
        },
        (payload) => {
          switch (payload.eventType) {
            case 'INSERT':
              setPositions(prev => [...prev, payload.new as Position]);
              break;
            case 'UPDATE':
              setPositions(prev =>
                prev.map(p => p.id === payload.new.id ? payload.new as Position : p)
              );
              break;
            case 'DELETE':
              setPositions(prev => prev.filter(p => p.id !== payload.old.id));
              break;
          }
        }
      )
      .subscribe();

    onCleanup(() => subscription.unsubscribe());
  });

  return positions;
}
```

## Database Setup
```sql
-- Enable realtime for a table
alter publication supabase_realtime add table positions;

-- Or in Supabase dashboard: Database > Replication
```

## Performance Considerations
```
1. Filter subscriptions — Don't subscribe to all rows
2. Use single channel — Multiple listeners on one channel
3. Debounce UI updates — Batch rapid changes
4. Cleanup subscriptions — Always unsubscribe on unmount
5. Connection pooling — Supabase handles this, but be aware
```

## Process
1. **Identify realtime needs** — What data needs live updates?
2. **Choose pattern** — Postgres changes, broadcast, or presence?
3. **Design channel structure** — Naming, filtering
4. **Implement subscription** — With proper cleanup
5. **Handle edge cases** — Reconnection, initial state

## Output
1. Channel/subscription code
2. Database replication setup (if needed)
3. SolidJS hook (if frontend integration)
4. Cleanup/connection management

---

## Reconnection Strategy

### Connection State Machine
```typescript
type ConnectionState = "disconnected" | "connecting" | "connected" | "reconnecting";

const [connectionState, setConnectionState] = createSignal<ConnectionState>("disconnected");
const [retryCount, setRetryCount] = createSignal(0);

const MAX_RETRIES = 10;
const BASE_DELAY = 1000;
const MAX_DELAY = 30000;
```

### Exponential Backoff
```typescript
const getBackoffDelay = (attempt: number): number => {
  const delay = Math.min(BASE_DELAY * Math.pow(2, attempt), MAX_DELAY);
  // Add jitter to prevent thundering herd
  const jitter = delay * 0.2 * Math.random();
  return delay + jitter;
};

const scheduleReconnect = () => {
  if (retryCount() >= MAX_RETRIES) {
    setConnectionState("disconnected");
    console.error("Max reconnection attempts reached");
    return;
  }

  setConnectionState("reconnecting");
  const delay = getBackoffDelay(retryCount());

  setTimeout(() => {
    setRetryCount((c) => c + 1);
    connect();
  }, delay);
};
```

### Supabase Reconnection Handler
```typescript
const createReconnectingChannel = (channelName: string, config: ChannelConfig) => {
  let channel: RealtimeChannel | null = null;

  const connect = () => {
    setConnectionState("connecting");

    channel = supabase
      .channel(channelName)
      .on("postgres_changes", config, handleChange)
      .subscribe((status, err) => {
        switch (status) {
          case "SUBSCRIBED":
            setConnectionState("connected");
            setRetryCount(0);
            break;
          case "CLOSED":
            scheduleReconnect();
            break;
          case "CHANNEL_ERROR":
            console.error("Channel error:", err);
            scheduleReconnect();
            break;
          case "TIMED_OUT":
            scheduleReconnect();
            break;
        }
      });
  };

  const disconnect = () => {
    if (channel) {
      supabase.removeChannel(channel);
      channel = null;
    }
    setConnectionState("disconnected");
  };

  return { connect, disconnect, state: connectionState };
};
```

### Network-Aware Reconnection
```typescript
const useNetworkAwareConnection = () => {
  const [online, setOnline] = createSignal(navigator.onLine);

  onMount(() => {
    const handleOnline = () => {
      setOnline(true);
      // Immediately attempt reconnect when coming back online
      if (connectionState() === "disconnected") {
        setRetryCount(0);
        connect();
      }
    };

    const handleOffline = () => {
      setOnline(false);
      // Don't waste attempts when offline
    };

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    onCleanup(() => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    });
  });

  return online;
};
```

### Visibility-Based Reconnection
```typescript
// Reconnect when tab becomes visible after being hidden
createEffect(() => {
  const handleVisibility = () => {
    if (document.visibilityState === "visible") {
      if (connectionState() !== "connected") {
        setRetryCount(0);
        connect();
      }
    }
  };

  document.addEventListener("visibilitychange", handleVisibility);
  onCleanup(() => document.removeEventListener("visibilitychange", handleVisibility));
});
```

---

## Offline Message Queue

### Message Queue Store
```typescript
interface QueuedMessage {
  id: string;
  channel: string;
  event: string;
  payload: unknown;
  timestamp: number;
  retries: number;
}

const [messageQueue, setMessageQueue] = createStore<QueuedMessage[]>([]);

const enqueue = (channel: string, event: string, payload: unknown) => {
  const message: QueuedMessage = {
    id: crypto.randomUUID(),
    channel,
    event,
    payload,
    timestamp: Date.now(),
    retries: 0,
  };
  setMessageQueue((q) => [...q, message]);
  persistQueue();
};

const dequeue = (id: string) => {
  setMessageQueue((q) => q.filter((m) => m.id !== id));
  persistQueue();
};
```

### Persistent Queue (IndexedDB)
```typescript
const QUEUE_KEY = "ws_message_queue";

const persistQueue = () => {
  localStorage.setItem(QUEUE_KEY, JSON.stringify(messageQueue));
};

const loadQueue = (): QueuedMessage[] => {
  const stored = localStorage.getItem(QUEUE_KEY);
  return stored ? JSON.parse(stored) : [];
};

// Initialize on mount
onMount(() => {
  const stored = loadQueue();
  setMessageQueue(stored);
});
```

### Queue Processor
```typescript
const processQueue = async () => {
  if (connectionState() !== "connected" || messageQueue.length === 0) return;

  const batch = messageQueue.slice(0, 10); // Process in batches

  for (const message of batch) {
    try {
      await sendMessage(message.channel, message.event, message.payload);
      dequeue(message.id);
    } catch (err) {
      // Increment retry count
      setMessageQueue(
        (m) => m.id === message.id,
        "retries",
        (r) => r + 1
      );

      // Remove if too many retries
      if (message.retries >= 3) {
        console.error("Message failed after 3 retries:", message);
        dequeue(message.id);
      }
    }
  }
};

// Process queue when connection is established
createEffect(() => {
  if (connectionState() === "connected") {
    processQueue();
  }
});
```

### Send with Queue Fallback
```typescript
const sendWithQueue = async (channel: string, event: string, payload: unknown) => {
  if (connectionState() === "connected") {
    try {
      await sendMessage(channel, event, payload);
      return { queued: false };
    } catch (err) {
      enqueue(channel, event, payload);
      return { queued: true };
    }
  } else {
    enqueue(channel, event, payload);
    return { queued: true };
  }
};

// Usage
const { queued } = await sendWithQueue("room:trading", "order", { symbol: "AAPL", qty: 100 });
if (queued) {
  showToast({ type: "info", message: "Order queued, will send when online" });
}
```

### Queue Status UI
```typescript
const QueueStatus: Component = () => {
  const pendingCount = () => messageQueue.length;

  return (
    <Show when={pendingCount() > 0}>
      <div class="flex items-center gap-2 text-sm text-yellow-600">
        <CloudOffIcon class="h-4 w-4" />
        <span>{pendingCount()} message(s) pending</span>
      </div>
    </Show>
  );
};
```

---

## Connection Health

### Heartbeat Monitor
```typescript
const HEARTBEAT_INTERVAL = 30000; // 30 seconds
const HEARTBEAT_TIMEOUT = 10000;  // 10 seconds to respond

const [lastHeartbeat, setLastHeartbeat] = createSignal(Date.now());
const [latency, setLatency] = createSignal<number | null>(null);

let heartbeatTimer: number | null = null;
let timeoutTimer: number | null = null;

const startHeartbeat = () => {
  heartbeatTimer = setInterval(() => {
    const pingTime = Date.now();

    // Send ping
    channel.send({ type: "broadcast", event: "ping", payload: { time: pingTime } });

    // Set timeout for pong
    timeoutTimer = setTimeout(() => {
      console.warn("Heartbeat timeout - connection may be stale");
      scheduleReconnect();
    }, HEARTBEAT_TIMEOUT);
  }, HEARTBEAT_INTERVAL);
};

// Listen for pong
channel.on("broadcast", { event: "pong" }, ({ payload }) => {
  if (timeoutTimer) clearTimeout(timeoutTimer);
  setLastHeartbeat(Date.now());
  setLatency(Date.now() - payload.time);
});

const stopHeartbeat = () => {
  if (heartbeatTimer) clearInterval(heartbeatTimer);
  if (timeoutTimer) clearTimeout(timeoutTimer);
};
```

### Stale Connection Detection
```typescript
const STALE_THRESHOLD = 60000; // 1 minute without activity

const isConnectionStale = () => {
  return Date.now() - lastHeartbeat() > STALE_THRESHOLD;
};

// Periodic stale check
createEffect(() => {
  const interval = setInterval(() => {
    if (connectionState() === "connected" && isConnectionStale()) {
      console.warn("Connection appears stale, reconnecting...");
      disconnect();
      connect();
    }
  }, 15000);

  onCleanup(() => clearInterval(interval));
});
```

### Health Status Component
```typescript
const ConnectionHealth: Component = () => {
  const healthStatus = () => {
    const state = connectionState();
    if (state !== "connected") return { status: "offline", color: "red" };
    if (isConnectionStale()) return { status: "stale", color: "yellow" };
    const lat = latency();
    if (lat && lat > 500) return { status: "slow", color: "yellow" };
    return { status: "healthy", color: "green" };
  };

  return (
    <div class="flex items-center gap-2">
      <div
        class="h-2 w-2 rounded-full"
        classList={{
          "bg-green-500": healthStatus().color === "green",
          "bg-yellow-500": healthStatus().color === "yellow",
          "bg-red-500": healthStatus().color === "red",
        }}
      />
      <span class="text-xs text-gray-500">
        {healthStatus().status}
        <Show when={latency()}> ({latency()}ms)</Show>
      </span>
    </div>
  );
};
```

### Connection Metrics
```typescript
interface ConnectionMetrics {
  connectedAt: number | null;
  disconnectCount: number;
  totalMessages: number;
  avgLatency: number;
}

const [metrics, setMetrics] = createStore<ConnectionMetrics>({
  connectedAt: null,
  disconnectCount: 0,
  totalMessages: 0,
  avgLatency: 0,
});

// Track uptime
const uptime = () => {
  if (!metrics.connectedAt) return 0;
  return Date.now() - metrics.connectedAt;
};

// Update on events
const onConnected = () => setMetrics("connectedAt", Date.now());
const onDisconnected = () => setMetrics("disconnectCount", (c) => c + 1);
const onMessage = () => setMetrics("totalMessages", (c) => c + 1);
```

---

## Channel Multiplexing

### Channel Manager
```typescript
interface ManagedChannel {
  name: string;
  channel: RealtimeChannel;
  subscribers: number;
  priority: "high" | "normal" | "low";
}

const [channels, setChannels] = createStore<Record<string, ManagedChannel>>({});

const getOrCreateChannel = (name: string, priority: "high" | "normal" | "low" = "normal") => {
  if (channels[name]) {
    setChannels(name, "subscribers", (s) => s + 1);
    return channels[name].channel;
  }

  const channel = supabase.channel(name);
  setChannels(name, {
    name,
    channel,
    subscribers: 1,
    priority,
  });

  return channel;
};

const releaseChannel = (name: string) => {
  if (!channels[name]) return;

  setChannels(name, "subscribers", (s) => s - 1);

  // Cleanup if no subscribers
  if (channels[name].subscribers <= 0) {
    supabase.removeChannel(channels[name].channel);
    setChannels((c) => {
      const { [name]: _, ...rest } = c;
      return rest;
    });
  }
};
```

### Shared Subscription Hook
```typescript
const useSharedChannel = <T>(
  channelName: string,
  table: string,
  filter?: string
) => {
  const [data, setData] = createSignal<T[]>([]);

  onMount(() => {
    const channel = getOrCreateChannel(channelName);

    channel
      .on(
        "postgres_changes",
        { event: "*", schema: "public", table, filter },
        (payload) => {
          // Handle change
          switch (payload.eventType) {
            case "INSERT":
              setData((prev) => [...prev, payload.new as T]);
              break;
            case "UPDATE":
              setData((prev) =>
                prev.map((item: any) =>
                  item.id === payload.new.id ? payload.new : item
                )
              );
              break;
            case "DELETE":
              setData((prev) =>
                prev.filter((item: any) => item.id !== payload.old.id)
              );
              break;
          }
        }
      )
      .subscribe();

    onCleanup(() => releaseChannel(channelName));
  });

  return data;
};
```

### Priority-Based Channel Management
```typescript
const MAX_CHANNELS = 10;

const enforceChannelLimit = () => {
  const channelList = Object.values(channels);
  if (channelList.length <= MAX_CHANNELS) return;

  // Sort by priority and subscriber count
  const sorted = channelList.sort((a, b) => {
    const priorityOrder = { high: 0, normal: 1, low: 2 };
    if (priorityOrder[a.priority] !== priorityOrder[b.priority]) {
      return priorityOrder[a.priority] - priorityOrder[b.priority];
    }
    return b.subscribers - a.subscribers;
  });

  // Remove lowest priority channels
  const toRemove = sorted.slice(MAX_CHANNELS);
  toRemove.forEach((c) => {
    supabase.removeChannel(c.channel);
    setChannels((channels) => {
      const { [c.name]: _, ...rest } = channels;
      return rest;
    });
  });
};
```

### Lazy Channel Subscription
```typescript
const useLazyChannel = (channelName: string) => {
  const [subscribed, setSubscribed] = createSignal(false);
  let channel: RealtimeChannel | null = null;

  const subscribe = () => {
    if (subscribed()) return;
    channel = getOrCreateChannel(channelName);
    channel.subscribe();
    setSubscribed(true);
  };

  const unsubscribe = () => {
    if (!subscribed()) return;
    releaseChannel(channelName);
    setSubscribed(false);
  };

  onCleanup(unsubscribe);

  return { subscribe, unsubscribe, subscribed };
};

// Usage - only subscribe when component is visible
const LazyDataComponent: Component = () => {
  const { subscribe, subscribed } = useLazyChannel("expensive-data");
  let ref: HTMLDivElement;

  onMount(() => {
    const observer = new IntersectionObserver(([entry]) => {
      if (entry.isIntersecting) subscribe();
    });
    observer.observe(ref);
    onCleanup(() => observer.disconnect());
  });

  return <div ref={ref!}>{subscribed() ? <DataContent /> : <Placeholder />}</div>;
};
```

---

## Error Recovery Patterns

### Error Classification
```typescript
type ErrorType = "network" | "auth" | "rate_limit" | "server" | "unknown";

const classifyError = (error: unknown): ErrorType => {
  if (!navigator.onLine) return "network";

  const message = error instanceof Error ? error.message : String(error);

  if (message.includes("401") || message.includes("403")) return "auth";
  if (message.includes("429")) return "rate_limit";
  if (message.includes("5")) return "server";

  return "unknown";
};

const getRecoveryStrategy = (errorType: ErrorType) => {
  switch (errorType) {
    case "network":
      return { action: "wait_online", delay: 0 };
    case "auth":
      return { action: "refresh_token", delay: 0 };
    case "rate_limit":
      return { action: "backoff", delay: 60000 };
    case "server":
      return { action: "retry", delay: 5000 };
    default:
      return { action: "retry", delay: 2000 };
  }
};
```

### Graceful Degradation
```typescript
const [degradedMode, setDegradedMode] = createSignal(false);

const enterDegradedMode = () => {
  setDegradedMode(true);
  // Switch to polling fallback
  startPollingFallback();
};

const exitDegradedMode = () => {
  setDegradedMode(false);
  stopPollingFallback();
};

// Polling fallback when WS fails
let pollInterval: number | null = null;

const startPollingFallback = () => {
  pollInterval = setInterval(async () => {
    const data = await fetchLatestData();
    updateLocalState(data);
  }, 5000);
};

const stopPollingFallback = () => {
  if (pollInterval) clearInterval(pollInterval);
};
```

### State Sync After Reconnection
```typescript
const syncStateAfterReconnect = async () => {
  // Get last known timestamp
  const lastSync = localStorage.getItem("last_sync_timestamp");

  // Fetch changes since disconnect
  const { data: changes } = await supabase
    .from("positions")
    .select("*")
    .gt("updated_at", lastSync)
    .order("updated_at", { ascending: true });

  // Apply changes in order
  changes?.forEach((change) => {
    applyChange(change);
  });

  // Update sync timestamp
  localStorage.setItem("last_sync_timestamp", new Date().toISOString());
};

// Call after successful reconnection
const onReconnected = async () => {
  setConnectionState("connected");
  await syncStateAfterReconnect();
  processQueue(); // Send queued messages
};
```

### Error Boundary for Realtime
```typescript
const RealtimeErrorBoundary: ParentComponent = (props) => {
  const [error, setError] = createSignal<Error | null>(null);

  const handleError = (err: Error) => {
    setError(err);
    console.error("Realtime error:", err);

    // Attempt recovery based on error type
    const errorType = classifyError(err);
    const strategy = getRecoveryStrategy(errorType);

    if (strategy.action === "refresh_token") {
      refreshAuthToken().then(() => {
        setError(null);
        reconnect();
      });
    } else if (strategy.action === "wait_online") {
      // Will auto-reconnect when online event fires
    } else {
      setTimeout(() => {
        setError(null);
        reconnect();
      }, strategy.delay);
    }
  };

  return (
    <Show
      when={!error()}
      fallback={
        <div class="p-4 bg-yellow-50 border border-yellow-200 rounded">
          <p class="text-yellow-800">Live updates temporarily unavailable</p>
          <p class="text-sm text-yellow-600">Data will refresh automatically</p>
        </div>
      }
    >
      {props.children}
    </Show>
  );
};
```

### Conflict Resolution
```typescript
// Handle conflicts when local changes conflict with server
const resolveConflict = (local: Position, server: Position): Position => {
  // Last-write-wins strategy
  if (new Date(server.updated_at) > new Date(local.updated_at)) {
    return server;
  }

  // Or merge strategy for specific fields
  return {
    ...server,
    // Preserve local unsaved changes
    notes: local.notes,
    tags: [...new Set([...local.tags, ...server.tags])],
  };
};

const handleConflictingUpdate = (serverData: Position) => {
  const localData = getLocalPosition(serverData.id);

  if (localData?.pendingChanges) {
    const resolved = resolveConflict(localData, serverData);
    updateLocalPosition(resolved);

    // Optionally notify user
    if (resolved !== serverData) {
      showToast({ type: "info", message: "Merged remote changes with your edits" });
    }
  } else {
    updateLocalPosition(serverData);
  }
};
```
