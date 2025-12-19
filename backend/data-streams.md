# Agent: data-streams

## Identity
Real-time data processing specialist for event streams, market data normalization, rolling aggregations, and live calculations.

## Stack
- Supabase Realtime (Postgres changes)
- TypeScript streaming patterns
- Web Workers (heavy calculations)
- IndexedDB (local buffering)

## Responsibilities
- Event stream processing
- Time-series windowing
- Rolling aggregations (VWAP, OHLC, SMA)
- Market data normalization
- Live P&L and Greeks calculations
- Data buffering and replay

## Patterns

### Event Stream Architecture
```typescript
interface StreamEvent<T> {
  id: string;
  timestamp: number;
  type: string;
  payload: T;
  sequence: number;
}

interface StreamProcessor<TIn, TOut> {
  process(event: StreamEvent<TIn>): TOut | null;
  flush(): TOut[];
  reset(): void;
}

// Event bus for stream coordination
class EventBus {
  private handlers = new Map<string, Set<(event: StreamEvent<any>) => void>>();
  private sequence = 0;

  emit<T>(type: string, payload: T): void {
    const event: StreamEvent<T> = {
      id: crypto.randomUUID(),
      timestamp: Date.now(),
      type,
      payload,
      sequence: ++this.sequence,
    };

    this.handlers.get(type)?.forEach((handler) => handler(event));
    this.handlers.get("*")?.forEach((handler) => handler(event));
  }

  on<T>(type: string, handler: (event: StreamEvent<T>) => void): () => void {
    if (!this.handlers.has(type)) {
      this.handlers.set(type, new Set());
    }
    this.handlers.get(type)!.add(handler);
    return () => this.handlers.get(type)?.delete(handler);
  }
}

const eventBus = new EventBus();
```

### Time-Series Windowing

#### Tumbling Window (Fixed, Non-Overlapping)
```typescript
class TumblingWindow<T> {
  private buffer: T[] = [];
  private windowStart: number;

  constructor(
    private windowSizeMs: number,
    private onFlush: (items: T[], windowStart: number, windowEnd: number) => void
  ) {
    this.windowStart = this.alignToWindow(Date.now());
    this.startTimer();
  }

  private alignToWindow(timestamp: number): number {
    return Math.floor(timestamp / this.windowSizeMs) * this.windowSizeMs;
  }

  private startTimer(): void {
    const nextFlush = this.windowStart + this.windowSizeMs - Date.now();
    setTimeout(() => {
      this.flush();
      this.startTimer();
    }, Math.max(0, nextFlush));
  }

  add(item: T): void {
    this.buffer.push(item);
  }

  private flush(): void {
    const windowEnd = this.windowStart + this.windowSizeMs;
    if (this.buffer.length > 0) {
      this.onFlush([...this.buffer], this.windowStart, windowEnd);
    }
    this.buffer = [];
    this.windowStart = windowEnd;
  }
}

// Usage: 1-minute OHLC candles
const ohlcWindow = new TumblingWindow<Trade>(60_000, (trades, start, end) => {
  const candle = calculateOHLC(trades);
  eventBus.emit("candle:1m", { ...candle, timestamp: start });
});
```

#### Sliding Window (Overlapping)
```typescript
class SlidingWindow<T> {
  private buffer: Array<{ item: T; timestamp: number }> = [];

  constructor(
    private windowSizeMs: number,
    private slideIntervalMs: number,
    private onSlide: (items: T[]) => void
  ) {
    setInterval(() => this.slide(), slideIntervalMs);
  }

  add(item: T): void {
    this.buffer.push({ item, timestamp: Date.now() });
    this.evictOld();
  }

  private evictOld(): void {
    const cutoff = Date.now() - this.windowSizeMs;
    this.buffer = this.buffer.filter((b) => b.timestamp >= cutoff);
  }

  private slide(): void {
    this.evictOld();
    this.onSlide(this.buffer.map((b) => b.item));
  }

  getItems(): T[] {
    this.evictOld();
    return this.buffer.map((b) => b.item);
  }
}

// Usage: 5-minute rolling average, updated every 10 seconds
const rollingAvg = new SlidingWindow<number>(5 * 60_000, 10_000, (prices) => {
  const avg = prices.reduce((a, b) => a + b, 0) / prices.length;
  eventBus.emit("price:sma5m", avg);
});
```

#### Session Window (Activity-Based)
```typescript
class SessionWindow<T> {
  private buffer: T[] = [];
  private lastActivity: number = 0;
  private timeout: NodeJS.Timeout | null = null;

  constructor(
    private gapMs: number,
    private onClose: (items: T[]) => void
  ) {}

  add(item: T): void {
    this.buffer.push(item);
    this.lastActivity = Date.now();

    if (this.timeout) clearTimeout(this.timeout);
    this.timeout = setTimeout(() => this.close(), this.gapMs);
  }

  private close(): void {
    if (this.buffer.length > 0) {
      this.onClose([...this.buffer]);
      this.buffer = [];
    }
  }
}

// Usage: Group trades into sessions (30s inactivity gap)
const tradeSession = new SessionWindow<Trade>(30_000, (trades) => {
  console.log(`Session closed: ${trades.length} trades`);
});
```

---

## Rolling Aggregations

### VWAP (Volume-Weighted Average Price)
```typescript
interface VWAPState {
  cumulativeTPV: number; // Total Price × Volume
  cumulativeVolume: number;
  vwap: number;
}

class VWAPCalculator {
  private state: VWAPState = {
    cumulativeTPV: 0,
    cumulativeVolume: 0,
    vwap: 0,
  };

  addTrade(price: number, volume: number): number {
    this.state.cumulativeTPV += price * volume;
    this.state.cumulativeVolume += volume;
    this.state.vwap = this.state.cumulativeTPV / this.state.cumulativeVolume;
    return this.state.vwap;
  }

  reset(): void {
    this.state = { cumulativeTPV: 0, cumulativeVolume: 0, vwap: 0 };
  }

  get value(): number {
    return this.state.vwap;
  }
}

// Usage with stream
const vwap = new VWAPCalculator();
eventBus.on<Trade>("trade", (event) => {
  const newVwap = vwap.addTrade(event.payload.price, event.payload.volume);
  eventBus.emit("vwap:update", newVwap);
});
```

### OHLC (Open-High-Low-Close)
```typescript
interface OHLC {
  open: number;
  high: number;
  low: number;
  close: number;
  volume: number;
  trades: number;
  timestamp: number;
}

class OHLCBuilder {
  private candle: OHLC | null = null;

  addTrade(price: number, volume: number, timestamp: number): OHLC {
    if (!this.candle) {
      this.candle = {
        open: price,
        high: price,
        low: price,
        close: price,
        volume,
        trades: 1,
        timestamp,
      };
    } else {
      this.candle.high = Math.max(this.candle.high, price);
      this.candle.low = Math.min(this.candle.low, price);
      this.candle.close = price;
      this.candle.volume += volume;
      this.candle.trades += 1;
    }
    return { ...this.candle };
  }

  finalize(): OHLC | null {
    const result = this.candle;
    this.candle = null;
    return result;
  }
}

// Multi-timeframe OHLC
const timeframes = [60_000, 300_000, 900_000, 3600_000]; // 1m, 5m, 15m, 1h
const ohlcBuilders = new Map<number, { builder: OHLCBuilder; window: TumblingWindow<Trade> }>();

timeframes.forEach((tf) => {
  const builder = new OHLCBuilder();
  const window = new TumblingWindow<Trade>(tf, (trades, start) => {
    const candle = builder.finalize();
    if (candle) {
      eventBus.emit(`candle:${tf}`, { ...candle, timestamp: start });
    }
  });
  ohlcBuilders.set(tf, { builder, window });
});
```

### Simple Moving Average (SMA)
```typescript
class SMACalculator {
  private values: number[] = [];

  constructor(private period: number) {}

  add(value: number): number | null {
    this.values.push(value);
    if (this.values.length > this.period) {
      this.values.shift();
    }
    if (this.values.length < this.period) {
      return null; // Not enough data
    }
    return this.values.reduce((a, b) => a + b, 0) / this.period;
  }

  get value(): number | null {
    if (this.values.length < this.period) return null;
    return this.values.reduce((a, b) => a + b, 0) / this.period;
  }
}

// Exponential Moving Average (EMA)
class EMACalculator {
  private ema: number | null = null;
  private multiplier: number;

  constructor(private period: number) {
    this.multiplier = 2 / (period + 1);
  }

  add(value: number): number {
    if (this.ema === null) {
      this.ema = value;
    } else {
      this.ema = (value - this.ema) * this.multiplier + this.ema;
    }
    return this.ema;
  }

  get value(): number | null {
    return this.ema;
  }
}
```

### Bollinger Bands
```typescript
interface BollingerBands {
  middle: number;
  upper: number;
  lower: number;
  bandwidth: number;
}

class BollingerCalculator {
  private values: number[] = [];

  constructor(
    private period: number = 20,
    private stdDevMultiplier: number = 2
  ) {}

  add(value: number): BollingerBands | null {
    this.values.push(value);
    if (this.values.length > this.period) {
      this.values.shift();
    }
    if (this.values.length < this.period) {
      return null;
    }

    const sma = this.values.reduce((a, b) => a + b, 0) / this.period;
    const variance = this.values.reduce((sum, v) => sum + Math.pow(v - sma, 2), 0) / this.period;
    const stdDev = Math.sqrt(variance);

    return {
      middle: sma,
      upper: sma + stdDev * this.stdDevMultiplier,
      lower: sma - stdDev * this.stdDevMultiplier,
      bandwidth: (2 * stdDev * this.stdDevMultiplier) / sma,
    };
  }
}
```

---

## Market Data Normalization

### Quote Normalization
```typescript
interface RawQuote {
  symbol: string;
  bid?: number;
  ask?: number;
  bidSize?: number;
  askSize?: number;
  last?: number;
  lastSize?: number;
  timestamp: string | number;
  source: string;
}

interface NormalizedQuote {
  symbol: string;
  bid: number;
  ask: number;
  bidSize: number;
  askSize: number;
  mid: number;
  spread: number;
  spreadPct: number;
  last: number | null;
  lastSize: number | null;
  timestamp: number;
  source: string;
  receivedAt: number;
}

function normalizeQuote(raw: RawQuote): NormalizedQuote {
  const bid = raw.bid ?? 0;
  const ask = raw.ask ?? 0;
  const mid = (bid + ask) / 2;

  return {
    symbol: raw.symbol.toUpperCase(),
    bid,
    ask,
    bidSize: raw.bidSize ?? 0,
    askSize: raw.askSize ?? 0,
    mid,
    spread: ask - bid,
    spreadPct: mid > 0 ? ((ask - bid) / mid) * 100 : 0,
    last: raw.last ?? null,
    lastSize: raw.lastSize ?? null,
    timestamp: typeof raw.timestamp === "string" ? new Date(raw.timestamp).getTime() : raw.timestamp,
    source: raw.source,
    receivedAt: Date.now(),
  };
}
```

### Options Chain Normalization
```typescript
interface RawOption {
  symbol: string;
  underlying: string;
  expiration: string;
  strike: number | string;
  type: "C" | "P" | "call" | "put";
  bid?: number;
  ask?: number;
  last?: number;
  volume?: number;
  openInterest?: number;
  iv?: number;
  delta?: number;
  gamma?: number;
  theta?: number;
  vega?: number;
}

interface NormalizedOption {
  symbol: string;
  underlying: string;
  expiration: Date;
  daysToExpiry: number;
  strike: number;
  type: "call" | "put";
  bid: number;
  ask: number;
  mid: number;
  last: number | null;
  volume: number;
  openInterest: number;
  greeks: {
    iv: number | null;
    delta: number | null;
    gamma: number | null;
    theta: number | null;
    vega: number | null;
  };
  intrinsicValue: number;
  extrinsicValue: number;
  moneyness: "ITM" | "ATM" | "OTM";
}

function normalizeOption(raw: RawOption, underlyingPrice: number): NormalizedOption {
  const expiration = new Date(raw.expiration);
  const strike = typeof raw.strike === "string" ? parseFloat(raw.strike) : raw.strike;
  const type = raw.type === "C" || raw.type === "call" ? "call" : "put";
  const bid = raw.bid ?? 0;
  const ask = raw.ask ?? 0;
  const mid = (bid + ask) / 2;

  // Calculate intrinsic value
  const intrinsicValue = type === "call"
    ? Math.max(0, underlyingPrice - strike)
    : Math.max(0, strike - underlyingPrice);

  const extrinsicValue = Math.max(0, mid - intrinsicValue);

  // Determine moneyness
  const atmThreshold = strike * 0.02; // 2% threshold
  let moneyness: "ITM" | "ATM" | "OTM";
  if (Math.abs(underlyingPrice - strike) < atmThreshold) {
    moneyness = "ATM";
  } else if (type === "call") {
    moneyness = underlyingPrice > strike ? "ITM" : "OTM";
  } else {
    moneyness = underlyingPrice < strike ? "ITM" : "OTM";
  }

  return {
    symbol: raw.symbol,
    underlying: raw.underlying.toUpperCase(),
    expiration,
    daysToExpiry: Math.ceil((expiration.getTime() - Date.now()) / (1000 * 60 * 60 * 24)),
    strike,
    type,
    bid,
    ask,
    mid,
    last: raw.last ?? null,
    volume: raw.volume ?? 0,
    openInterest: raw.openInterest ?? 0,
    greeks: {
      iv: raw.iv ?? null,
      delta: raw.delta ?? null,
      gamma: raw.gamma ?? null,
      theta: raw.theta ?? null,
      vega: raw.vega ?? null,
    },
    intrinsicValue,
    extrinsicValue,
    moneyness,
  };
}
```

---

## Live P&L Calculations

### Position P&L
```typescript
interface Position {
  id: string;
  symbol: string;
  quantity: number;
  entryPrice: number;
  side: "long" | "short";
  type: "stock" | "option";
  multiplier: number; // 100 for options, 1 for stocks
}

interface PositionPL {
  positionId: string;
  symbol: string;
  unrealizedPL: number;
  unrealizedPLPercent: number;
  marketValue: number;
  costBasis: number;
  currentPrice: number;
  dayPL: number;
  dayPLPercent: number;
}

class PLCalculator {
  private positions = new Map<string, Position>();
  private prices = new Map<string, { current: number; previousClose: number }>();

  updatePosition(position: Position): void {
    this.positions.set(position.id, position);
  }

  updatePrice(symbol: string, current: number, previousClose: number): void {
    this.prices.set(symbol, { current, previousClose });
    this.recalculate(symbol);
  }

  private recalculate(symbol: string): void {
    for (const [id, position] of this.positions) {
      if (position.symbol === symbol) {
        const pl = this.calculatePL(position);
        eventBus.emit("pl:update", pl);
      }
    }
  }

  calculatePL(position: Position): PositionPL {
    const priceData = this.prices.get(position.symbol);
    const currentPrice = priceData?.current ?? position.entryPrice;
    const previousClose = priceData?.previousClose ?? position.entryPrice;

    const costBasis = position.entryPrice * position.quantity * position.multiplier;
    const marketValue = currentPrice * position.quantity * position.multiplier;

    const direction = position.side === "long" ? 1 : -1;
    const unrealizedPL = (marketValue - costBasis) * direction;
    const unrealizedPLPercent = costBasis !== 0 ? (unrealizedPL / costBasis) * 100 : 0;

    const previousValue = previousClose * position.quantity * position.multiplier;
    const dayPL = (marketValue - previousValue) * direction;
    const dayPLPercent = previousValue !== 0 ? (dayPL / previousValue) * 100 : 0;

    return {
      positionId: position.id,
      symbol: position.symbol,
      unrealizedPL,
      unrealizedPLPercent,
      marketValue,
      costBasis,
      currentPrice,
      dayPL,
      dayPLPercent,
    };
  }

  getPortfolioPL(): {
    totalUnrealizedPL: number;
    totalDayPL: number;
    totalMarketValue: number;
    totalCostBasis: number;
  } {
    let totalUnrealizedPL = 0;
    let totalDayPL = 0;
    let totalMarketValue = 0;
    let totalCostBasis = 0;

    for (const position of this.positions.values()) {
      const pl = this.calculatePL(position);
      totalUnrealizedPL += pl.unrealizedPL;
      totalDayPL += pl.dayPL;
      totalMarketValue += pl.marketValue;
      totalCostBasis += pl.costBasis;
    }

    return { totalUnrealizedPL, totalDayPL, totalMarketValue, totalCostBasis };
  }
}
```

### Greeks Aggregation
```typescript
interface PositionGreeks {
  delta: number;
  gamma: number;
  theta: number;
  vega: number;
  rho: number;
}

class GreeksAggregator {
  private positionGreeks = new Map<string, PositionGreeks>();

  updatePositionGreeks(positionId: string, greeks: PositionGreeks, quantity: number): void {
    this.positionGreeks.set(positionId, {
      delta: greeks.delta * quantity,
      gamma: greeks.gamma * quantity,
      theta: greeks.theta * quantity,
      vega: greeks.vega * quantity,
      rho: greeks.rho * quantity,
    });
    this.emitPortfolioGreeks();
  }

  private emitPortfolioGreeks(): void {
    const portfolio: PositionGreeks = {
      delta: 0,
      gamma: 0,
      theta: 0,
      vega: 0,
      rho: 0,
    };

    for (const greeks of this.positionGreeks.values()) {
      portfolio.delta += greeks.delta;
      portfolio.gamma += greeks.gamma;
      portfolio.theta += greeks.theta;
      portfolio.vega += greeks.vega;
      portfolio.rho += greeks.rho;
    }

    eventBus.emit("greeks:portfolio", portfolio);
  }

  getNetDelta(): number {
    return Array.from(this.positionGreeks.values()).reduce((sum, g) => sum + g.delta, 0);
  }
}
```

---

## Data Buffering & Replay

### Ring Buffer
```typescript
class RingBuffer<T> {
  private buffer: T[];
  private head = 0;
  private tail = 0;
  private count = 0;

  constructor(private capacity: number) {
    this.buffer = new Array(capacity);
  }

  push(item: T): T | null {
    const evicted = this.count === this.capacity ? this.buffer[this.tail] : null;

    this.buffer[this.head] = item;
    this.head = (this.head + 1) % this.capacity;

    if (this.count === this.capacity) {
      this.tail = (this.tail + 1) % this.capacity;
    } else {
      this.count++;
    }

    return evicted;
  }

  getAll(): T[] {
    const result: T[] = [];
    for (let i = 0; i < this.count; i++) {
      result.push(this.buffer[(this.tail + i) % this.capacity]);
    }
    return result;
  }

  getLast(n: number): T[] {
    const count = Math.min(n, this.count);
    const result: T[] = [];
    for (let i = this.count - count; i < this.count; i++) {
      result.push(this.buffer[(this.tail + i) % this.capacity]);
    }
    return result;
  }

  get size(): number {
    return this.count;
  }
}

// Usage: Keep last 1000 trades for replay
const tradeBuffer = new RingBuffer<Trade>(1000);
eventBus.on<Trade>("trade", (event) => {
  tradeBuffer.push(event.payload);
});
```

### IndexedDB Persistence
```typescript
class StreamPersistence {
  private db: IDBDatabase | null = null;
  private dbName = "stream-data";

  async init(): Promise<void> {
    return new Promise((resolve, reject) => {
      const request = indexedDB.open(this.dbName, 1);

      request.onerror = () => reject(request.error);
      request.onsuccess = () => {
        this.db = request.result;
        resolve();
      };

      request.onupgradeneeded = (event) => {
        const db = (event.target as IDBOpenDBRequest).result;

        // Trades store
        const tradesStore = db.createObjectStore("trades", { keyPath: "id" });
        tradesStore.createIndex("symbol", "symbol");
        tradesStore.createIndex("timestamp", "timestamp");

        // Candles store
        const candlesStore = db.createObjectStore("candles", { keyPath: ["symbol", "timeframe", "timestamp"] });
        candlesStore.createIndex("symbol_timeframe", ["symbol", "timeframe"]);
      };
    });
  }

  async saveTrade(trade: Trade): Promise<void> {
    return new Promise((resolve, reject) => {
      const tx = this.db!.transaction("trades", "readwrite");
      tx.objectStore("trades").add(trade);
      tx.oncomplete = () => resolve();
      tx.onerror = () => reject(tx.error);
    });
  }

  async getTradesSince(symbol: string, since: number): Promise<Trade[]> {
    return new Promise((resolve, reject) => {
      const tx = this.db!.transaction("trades", "readonly");
      const store = tx.objectStore("trades");
      const index = store.index("symbol");
      const range = IDBKeyRange.only(symbol);

      const trades: Trade[] = [];
      const request = index.openCursor(range);

      request.onsuccess = () => {
        const cursor = request.result;
        if (cursor) {
          if (cursor.value.timestamp >= since) {
            trades.push(cursor.value);
          }
          cursor.continue();
        } else {
          resolve(trades);
        }
      };
      request.onerror = () => reject(request.error);
    });
  }

  async pruneOldData(maxAgeMs: number): Promise<number> {
    const cutoff = Date.now() - maxAgeMs;
    let deleted = 0;

    return new Promise((resolve, reject) => {
      const tx = this.db!.transaction("trades", "readwrite");
      const store = tx.objectStore("trades");
      const index = store.index("timestamp");
      const range = IDBKeyRange.upperBound(cutoff);

      const request = index.openCursor(range);
      request.onsuccess = () => {
        const cursor = request.result;
        if (cursor) {
          cursor.delete();
          deleted++;
          cursor.continue();
        }
      };

      tx.oncomplete = () => resolve(deleted);
      tx.onerror = () => reject(tx.error);
    });
  }
}
```

---

## Web Worker Processing

### Heavy Calculations in Worker
```typescript
// worker.ts
self.onmessage = (event: MessageEvent) => {
  const { type, payload } = event.data;

  switch (type) {
    case "calculate-greeks":
      const greeks = calculateBlackScholes(payload);
      self.postMessage({ type: "greeks-result", payload: greeks });
      break;

    case "calculate-vwap":
      const vwap = calculateVWAP(payload.trades);
      self.postMessage({ type: "vwap-result", payload: vwap });
      break;

    case "build-candles":
      const candles = buildCandles(payload.trades, payload.timeframe);
      self.postMessage({ type: "candles-result", payload: candles });
      break;
  }
};

function calculateBlackScholes(params: {
  spotPrice: number;
  strikePrice: number;
  timeToExpiry: number;
  riskFreeRate: number;
  volatility: number;
  optionType: "call" | "put";
}) {
  const { spotPrice, strikePrice, timeToExpiry, riskFreeRate, volatility, optionType } = params;

  const d1 =
    (Math.log(spotPrice / strikePrice) +
      (riskFreeRate + (volatility * volatility) / 2) * timeToExpiry) /
    (volatility * Math.sqrt(timeToExpiry));
  const d2 = d1 - volatility * Math.sqrt(timeToExpiry);

  const nd1 = normalCDF(d1);
  const nd2 = normalCDF(d2);
  const nnd1 = normalCDF(-d1);
  const nnd2 = normalCDF(-d2);
  const npd1 = normalPDF(d1);

  const expRT = Math.exp(-riskFreeRate * timeToExpiry);
  const sqrtT = Math.sqrt(timeToExpiry);

  let delta: number, gamma: number, theta: number, vega: number;

  if (optionType === "call") {
    delta = nd1;
    theta =
      (-spotPrice * npd1 * volatility) / (2 * sqrtT) -
      riskFreeRate * strikePrice * expRT * nd2;
  } else {
    delta = nd1 - 1;
    theta =
      (-spotPrice * npd1 * volatility) / (2 * sqrtT) +
      riskFreeRate * strikePrice * expRT * nnd2;
  }

  gamma = npd1 / (spotPrice * volatility * sqrtT);
  vega = spotPrice * sqrtT * npd1 / 100;

  return { delta, gamma, theta: theta / 365, vega };
}

function normalCDF(x: number): number {
  const a1 = 0.254829592;
  const a2 = -0.284496736;
  const a3 = 1.421413741;
  const a4 = -1.453152027;
  const a5 = 1.061405429;
  const p = 0.3275911;

  const sign = x < 0 ? -1 : 1;
  x = Math.abs(x) / Math.sqrt(2);

  const t = 1.0 / (1.0 + p * x);
  const y = 1.0 - ((((a5 * t + a4) * t + a3) * t + a2) * t + a1) * t * Math.exp(-x * x);

  return 0.5 * (1.0 + sign * y);
}

function normalPDF(x: number): number {
  return Math.exp(-0.5 * x * x) / Math.sqrt(2 * Math.PI);
}
```

### Worker Manager
```typescript
class WorkerManager {
  private worker: Worker;
  private pending = new Map<string, { resolve: Function; reject: Function }>();

  constructor() {
    this.worker = new Worker(new URL("./worker.ts", import.meta.url), { type: "module" });
    this.worker.onmessage = this.handleMessage.bind(this);
  }

  private handleMessage(event: MessageEvent): void {
    const { type, payload, requestId } = event.data;
    const pending = this.pending.get(requestId);
    if (pending) {
      pending.resolve(payload);
      this.pending.delete(requestId);
    }
  }

  async calculate<T>(type: string, payload: any): Promise<T> {
    const requestId = crypto.randomUUID();

    return new Promise((resolve, reject) => {
      this.pending.set(requestId, { resolve, reject });
      this.worker.postMessage({ type, payload, requestId });

      // Timeout after 10 seconds
      setTimeout(() => {
        if (this.pending.has(requestId)) {
          this.pending.delete(requestId);
          reject(new Error("Worker timeout"));
        }
      }, 10000);
    });
  }

  terminate(): void {
    this.worker.terminate();
  }
}

// Usage
const workerManager = new WorkerManager();
const greeks = await workerManager.calculate("calculate-greeks", {
  spotPrice: 150,
  strikePrice: 155,
  timeToExpiry: 30 / 365,
  riskFreeRate: 0.05,
  volatility: 0.25,
  optionType: "call",
});
```

---

## Process
1. **Identify data sources** — What streams need processing?
2. **Define normalization** — Standardize formats
3. **Choose windowing strategy** — Tumbling, sliding, or session
4. **Implement aggregations** — VWAP, OHLC, moving averages
5. **Set up buffering** — Ring buffer, IndexedDB
6. **Offload heavy calculations** — Web Workers for Greeks

## Output
1. Stream processor implementations
2. Aggregation calculators
3. Normalization functions
4. Worker scripts for heavy calculations
5. Persistence layer setup
