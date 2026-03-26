# Frontend Consumption — WebSocket Client Patterns

Client-side patterns for consuming Socket.IO events in React + TypeScript applications.

---

## 1. Connection Setup

```typescript
// lib/socket.ts
import { io, type Socket } from 'socket.io-client';

let socket: Socket | null = null;

export function getSocket(token: string): Socket {
  if (socket?.connected) return socket;

  socket = io(process.env.NEXT_PUBLIC_API_URL + '/events', {
    auth: { token },
    reconnection: true,
    reconnectionAttempts: 5,
    reconnectionDelay: 1000,
    reconnectionDelayMax: 10_000,
    timeout: 20_000,
    transports: ['websocket'],
  });

  return socket;
}

export function disconnectSocket(): void {
  socket?.disconnect();
  socket = null;
}
```

---

## 2. React Provider

```typescript
// providers/socket-provider.tsx
'use client';

import { createContext, useContext, useEffect, useRef, useState } from 'react';
import { type Socket } from 'socket.io-client';
import { getSocket, disconnectSocket } from '@/lib/socket';

interface SocketContextValue {
  socket: Socket | null;
  isConnected: boolean;
}

const SocketContext = createContext<SocketContextValue>({ socket: null, isConnected: false });

export function SocketProvider({
  token,
  children,
}: {
  token: string;
  children: React.ReactNode;
}) {
  const [isConnected, setIsConnected] = useState(false);
  const socketRef = useRef<Socket | null>(null);

  useEffect(() => {
    const s = getSocket(token);
    socketRef.current = s;

    s.on('connect', () => setIsConnected(true));
    s.on('disconnect', () => setIsConnected(false));
    s.on('connect_error', async (err) => {
      if (err.message === 'UNAUTHORIZED' || err.message.includes('jwt')) {
        // Refresh token and reconnect
        const newToken = await refreshAccessToken();
        s.auth = { token: newToken };
        s.connect();
      }
    });

    return () => {
      disconnectSocket();
    };
  }, [token]);

  return (
    <SocketContext.Provider value={{ socket: socketRef.current, isConnected }}>
      {children}
    </SocketContext.Provider>
  );
}

export function useSocketContext(): SocketContextValue {
  return useContext(SocketContext);
}
```

---

## 3. Domain-Scoped Event Hook

Each domain has a dedicated hook. The hook receives the socket context and query client as arguments — it never calls hooks internally to obtain them (keeps it testable and free of unstable dependencies).

```typescript
// hooks/socket-events/use-order-socket-events.ts
'use client';

import type { QueryClient } from '@tanstack/react-query';
import { useEffect } from 'react';

import type { SocketContextValue } from '@/providers/socket-provider';

// Define the event payload type
export type OrderCreatedEvent = {
  id: string;
  customerId: string;
  total: number;
  status: string;
  createdAt: string;
  organizationId: string;
};

export type OrderStatusChangedEvent = {
  id: string;
  status: string;
  previousStatus: string;
  organizationId: string;
};

export function useOrderSocketEvents(
  ctx: SocketContextValue,
  queryClient: QueryClient,
): void {
  const { socket, isConnected } = ctx;

  useEffect(() => {
    // Guard — do not register without an active connection
    if (!socket || !isConnected) return;

    const handleOrderCreated = (event: OrderCreatedEvent) => {
      try {
        // Invalidate the orders list — re-fetch when the user navigates to it
        queryClient.invalidateQueries({ queryKey: ['orders'], refetchType: 'none' });

        // If currently viewing the orders page, update immediately
        queryClient.invalidateQueries({ queryKey: ['orders-list'] });
      } catch (error) {
        if (process.env.NODE_ENV !== 'production') {
          console.error('[useOrderSocketEvents] Error handling order:created', error);
        }
      }
    };

    const handleOrderStatusChanged = (event: OrderStatusChangedEvent) => {
      try {
        // Update the specific order in cache if it exists
        queryClient.setQueryData<OrderCreatedEvent>(
          ['order', event.id],
          (old) => old ? { ...old, status: event.status } : old,
        );
        // Mark the list as stale
        queryClient.invalidateQueries({ queryKey: ['orders'], refetchType: 'none' });
      } catch (error) {
        if (process.env.NODE_ENV !== 'production') {
          console.error('[useOrderSocketEvents] Error handling order:status-changed', error);
        }
      }
    };

    socket.on('order:created', handleOrderCreated);
    socket.on('order:status-changed', handleOrderStatusChanged);

    // Cleanup — always pass the handler reference, not just the event name
    return () => {
      socket.off('order:created', handleOrderCreated);
      socket.off('order:status-changed', handleOrderStatusChanged);
    };
  }, [socket, isConnected, queryClient]);
}
```

---

## 4. Orchestrator Hook

Compose domain hooks in a single orchestrator. Components never call individual domain hooks directly.

```typescript
// hooks/socket-events/index.ts
'use client';

import { useQueryClient } from '@tanstack/react-query';
import { useSocketContext } from '@/providers/socket-provider';
import { useOrderSocketEvents } from './use-order-socket-events';
import { usePaymentSocketEvents } from './use-payment-socket-events';
import { useNotificationSocketEvents } from './use-notification-socket-events';

export function useSocketEvents(): void {
  const ctx = useSocketContext();
  const queryClient = useQueryClient();

  useOrderSocketEvents(ctx, queryClient);
  usePaymentSocketEvents(ctx, queryClient);
  useNotificationSocketEvents(ctx, queryClient);
}
```

Mount the orchestrator once at the top of the layout tree via a headless component:

```typescript
// components/socket-events-initializer.tsx
'use client';

import { useSocketEvents } from '@/hooks/socket-events';

export function SocketEventsInitializer(): null {
  useSocketEvents();
  return null;
}
```

---

## 5. Reconnect Resync

Events emitted while the client was disconnected are permanently lost. On reconnect, always resync from the REST API.

```typescript
// In useMiscSocketEvents (or a dedicated reconnect handler)
useEffect(() => {
  if (!socket) return;

  const handleReconnect = () => {
    // 1. Invalidate all critical queries (refetchType: 'none' avoids a flood of requests)
    queryClient.invalidateQueries({ queryKey: ['orders'], refetchType: 'none' });
    queryClient.invalidateQueries({ queryKey: ['notifications'], refetchType: 'none' });

    // 2. Clear local deduplication sets (stale IDs are irrelevant after a gap)
    processedIdsRef.current.clear();
  };

  // Note: socket.io.on (manager), not socket.on (socket instance)
  socket.io.on('reconnect', handleReconnect);

  return () => {
    socket.io.off('reconnect', handleReconnect);
  };
}, [socket, queryClient]);
```

---

## 6. Event Deduplication

The server may emit the same event more than once under retry or reconnect conditions. Use a bounded `Set` to deduplicate by a stable ID.

```typescript
const processedIdsRef = useRef<Set<string>>(new Set());
const MAX_PROCESSED = 100;

const handleOrderCreated = (event: OrderCreatedEvent) => {
  if (processedIdsRef.current.has(event.id)) return;

  processedIdsRef.current.add(event.id);

  // Evict oldest entry when the Set exceeds the limit (FIFO)
  if (processedIdsRef.current.size > MAX_PROCESSED) {
    const first = processedIdsRef.current.values().next().value;
    if (first) processedIdsRef.current.delete(first);
  }

  // Process event...
};
```

Use `useRef` — mutation does not trigger re-renders. Do not deduplicate ephemeral, high-frequency events (typing indicators, presence updates) — their frequency is intentional and the last value wins.

---

## 7. Type-Safe Event Map

Define a typed map of all server-to-client events to get autocomplete and catch contract mismatches at compile time:

```typescript
// types/socket-events.ts
export interface ServerToClientEvents {
  'order:created': (data: OrderCreatedEvent) => void;
  'order:status-changed': (data: OrderStatusChangedEvent) => void;
  'payment:received': (data: PaymentReceivedEvent) => void;
  'notification:created': (data: NotificationCreatedEvent) => void;
  'channel:status-changed': (data: ChannelStatusChangedEvent) => void;
}

export interface ClientToServerEvents {
  'order:cancel': (data: CancelOrderCommand, ack: (result: WsAck) => void) => void;
  'presence:heartbeat': (ack: (result: WsAck) => void) => void;
}

// Typed socket
import { type Socket } from 'socket.io-client';
export type AppSocket = Socket<ServerToClientEvents, ClientToServerEvents>;
```

---

## 8. Sending Commands with Acknowledgement

```typescript
function cancelOrder(orderId: string): Promise<WsAck> {
  return new Promise((resolve, reject) => {
    const timeout = setTimeout(() => reject(new Error('Command timeout')), 10_000);

    socket.emit('order:cancel', { orderId }, (ack: WsAck) => {
      clearTimeout(timeout);
      resolve(ack);
    });
  });
}

// Usage
const result = await cancelOrder('order-123');
if (!result.ok) {
  toast.error(result.error?.message ?? 'Failed to cancel order');
}
```

---

## 9. Connection State in UI

```typescript
// hooks/use-connection-status.ts
import { useSocketContext } from '@/providers/socket-provider';

export function useConnectionStatus(): 'connected' | 'connecting' | 'disconnected' {
  const { socket, isConnected } = useSocketContext();

  if (isConnected) return 'connected';
  if (socket?.active) return 'connecting'; // reconnecting
  return 'disconnected';
}
```

---

## 10. Anti-Patterns

| Anti-pattern | Problem | Solution |
|---|---|---|
| Calling `useStore()` inside a socket event handler | Violates Rules of Hooks, captures stale closure | Use `store.getState()` inside the handler |
| `socket.off('event')` without handler reference | Removes all listeners for that event | Always `socket.off('event', handlerRef)` |
| HTTP fetch inside a socket event handler | Waterfall, unnecessary loading state | Use `queryClient.invalidateQueries()` |
| Assuming state is correct after reconnect | Events during disconnect are lost | Invalidate and re-fetch on reconnect |
| Registering listener outside `useEffect` | No cleanup → multiple subscriptions on re-render | Always inside `useEffect` with cleanup return |
| `socket.on` without guard `if (!socket \|\| !isConnected)` | Null reference error | Guard at the top of `useEffect` |
| Storing raw WS payload shape in app state | Breaks when backend contract changes | Convert to internal type before storing |
| Unbounded deduplication Set | Memory leak in long sessions | Set with FIFO eviction at max size |
