<script lang="ts">
  import logo from '$lib/assets/logo.svg';
  import { onMount, onDestroy } from 'svelte';
  import { tweened } from 'svelte/motion';
  import { cubicOut } from 'svelte/easing';

  // Constants
  const CHART = {
    WIDTH: 600,
    HEIGHT: 200,
    PADDING: 10,
    CANDLE_COUNT: 24,
    CANDLE_WIDTH: 8,
  } as const;

  const PRICE_BOUNDS = {
    MIN: 0.0001,
    MAX: 10,
  } as const;

  const ANIMATION = {
    DURATION: 550,
    PRICE_DURATION: 450,
    EASING: cubicOut,
  } as const;

  // Derived constants
  const spacing = (CHART.WIDTH - 2 * CHART.PADDING) / CHART.CANDLE_COUNT;

  // Market state
  let price = 0.06;
  let prices = initializePrices();
  let volumes = initializeVolumes();
  // Price pressure (positive => upward bias, negative => downward bias)
  let pressureTicks = 0;
  let pressurePerTick = 0;
  // Track recent price direction for candle generation
  let recentPriceDirection: 'up' | 'down' | 'neutral' = 'neutral';
  let lastPriceChange = 0;

  // Chart bounds
  let minPrice = 0;
  let maxPrice = 0;

  // EMAs
  let ema5 = price;
  let ema10 = price;
  let ema20 = price;
  let ema200 = price;

  // Animation stores
  const priceTween = tweened(price, {
    duration: ANIMATION.PRICE_DURATION,
    easing: ANIMATION.EASING,
  });
  let priceSlide = false;
  let priceSlideTimer: ReturnType<typeof setTimeout> | null = null;

  // Candle animation stores
  let bodyTopStores: any[] = [];
  let bodyHStores: any[] = [];
  let wickTopStores: any[] = [];
  let wickBottomStores: any[] = [];
  let bodyTopValues: number[] = Array(CHART.CANDLE_COUNT).fill(0);
  let bodyHValues: number[] = Array(CHART.CANDLE_COUNT).fill(0);
  let wickTopValues: number[] = Array(CHART.CANDLE_COUNT).fill(0);
  let wickBottomValues: number[] = Array(CHART.CANDLE_COUNT).fill(0);

  // Trading state
  let balance = 200;
  let holdings = 0;
  let orderQty = 1;
  let tradeMsg = '';
  let toastVisible = false;
  let trades: Array<{
    id: number;
    type: 'Buy' | 'Sell';
    qty: number;
    price: number;
    time: string;
  }> = [];
  let balanceFlash = '';
  let balanceFlashTimer: any = null;
  let tradeTimer: any = null;

  // Initialization helpers
  function initializePrices(): number[] {
    return Array.from({ length: CHART.CANDLE_COUNT }, () => {
      const variance = (Math.random() - 0.5) * 0.01;
      const value = price + variance;
      return clampPrice(value);
    });
  }

  function initializeVolumes(): number[] {
    return Array.from(
      { length: CHART.CANDLE_COUNT },
      () => Math.floor(Math.random() * 300) + 20
    );
  }

  function clampPrice(value: number): number {
    return +Math.min(
      PRICE_BOUNDS.MAX,
      Math.max(PRICE_BOUNDS.MIN, value)
    ).toFixed(4);
  }

  // Price update logic
  function updatePrice() {
    const prev = prices[prices.length - 1];
    const normalStep = 0.003;
    let delta: number;

    if (pressureTicks > 0) {
      // pressurePerTick may be positive (push up) or negative (push down)
      const bias = pressurePerTick * (pressureTicks / (pressureTicks + 1));
      const jitter = (Math.random() - 0.5) * normalStep;
      delta = bias + jitter;
      pressureTicks = Math.max(0, pressureTicks - 1);
    } else {
      delta = (Math.random() - 0.5) * 2 * normalStep;
    }

    // If user has holdings, prevent any upward movement
    if (holdings > 0 && delta > 0) {
      delta = -Math.abs(delta);
    }

    let candidate = prev + delta;
    if (candidate < PRICE_BOUNDS.MIN)
      candidate = PRICE_BOUNDS.MIN + Math.random() * 0.0005;
    if (candidate > PRICE_BOUNDS.MAX)
      candidate = PRICE_BOUNDS.MAX - Math.random() * 0.0005;

    // Final check: if user has holdings, ensure price never goes above previous
    if (holdings > 0 && candidate > prev) {
      candidate = prev;
    }

    const next = +candidate.toFixed(4);
    prices = [...prices.slice(1), next];
    volumes = [...volumes.slice(1), Math.floor(Math.random() * 200) + 6];

    // Track price direction for candle generation
    lastPriceChange = next - prev;
    if (lastPriceChange > 0.001) {
      recentPriceDirection = 'up';
    } else if (lastPriceChange < -0.001) {
      recentPriceDirection = 'down';
    } else {
      recentPriceDirection = 'neutral';
    }

    updateEMAs(next);
    price = next;
  }

  function updateEMAs(newPrice: number) {
    const alphas = {
      ema5: 2 / 6,
      ema10: 2 / 11,
      ema20: 2 / 21,
      ema200: 2 / 201,
    };

    ema5 = +(alphas.ema5 * newPrice + (1 - alphas.ema5) * ema5).toFixed(6);
    ema10 = +(alphas.ema10 * newPrice + (1 - alphas.ema10) * ema10).toFixed(6);
    ema20 = +(alphas.ema20 * newPrice + (1 - alphas.ema20) * ema20).toFixed(6);
    ema200 = +(alphas.ema200 * newPrice + (1 - alphas.ema200) * ema200).toFixed(
      6
    );
  }

  // Chart calculations
  function generateCandles(prices: number[]) {
    return prices.map((close, i) => {
      const prevPrice = i > 0 ? prices[i - 1] : close;
      const priceChange = close - prevPrice;
      const priceChangePercent =
        prevPrice > 0 ? Math.abs(priceChange / prevPrice) : 0;
      const isUp = priceChange >= 0;

      // Base range - increased for more volatility
      const baseRange = Math.max(prevPrice * 0.15, 0.005);

      // For strong moves, create more dramatic candles
      let open: number;
      let high: number;
      let low: number;

      if (isUp && priceChangePercent > 0.02) {
        // Strong upward move - create big green candles
        const volatility = Math.random();
        if (volatility < 0.3) {
          // Long green candle (30% chance)
          open = +(prevPrice + (Math.random() - 0.8) * baseRange * 0.3).toFixed(
            4
          );
          const bodySize = close - open;
          high = +(
            close +
            Math.random() * bodySize * 0.5 +
            baseRange * 0.2
          ).toFixed(4);
          low = +(open - Math.random() * baseRange * 0.1).toFixed(4);
        } else if (volatility < 0.5) {
          // Spiky upward candle with long upper wick (20% chance)
          open = +(prevPrice + (Math.random() - 0.5) * baseRange * 0.4).toFixed(
            4
          );
          const mid = (open + close) / 2;
          high = +(
            mid +
            Math.random() * baseRange * 0.8 +
            priceChange * 0.8
          ).toFixed(4);
          low = +(
            Math.min(open, close) -
            Math.random() * baseRange * 0.2
          ).toFixed(4);
        } else {
          // Normal but strong upward move (50% chance)
          open = +(prevPrice + (Math.random() - 0.6) * baseRange * 0.5).toFixed(
            4
          );
          high = +(
            Math.max(open, close) +
            Math.random() * baseRange * 0.4 +
            priceChange * 0.3
          ).toFixed(4);
          low = +(
            Math.min(open, close) -
            Math.random() * baseRange * 0.3
          ).toFixed(4);
        }
      } else if (!isUp && priceChangePercent > 0.02) {
        // Strong downward move
        const volatility = Math.random();
        open = +(prevPrice + (Math.random() - 0.2) * baseRange * 0.5).toFixed(
          4
        );
        high = +(
          Math.max(open, close) +
          Math.random() * baseRange * 0.3
        ).toFixed(4);
        low = +(
          Math.min(open, close) -
          Math.random() * baseRange * 0.4 -
          Math.abs(priceChange) * 0.3
        ).toFixed(4);
      } else {
        // Normal volatility
        const volatility = Math.random();
        if (volatility < 0.15 && isUp) {
          // Occasionally create a big green candle (15% chance when up)
          open = +(prevPrice + (Math.random() - 0.7) * baseRange * 0.4).toFixed(
            4
          );
          const bodySize = Math.max(close - open, baseRange * 0.3);
          high = +(
            close +
            Math.random() * bodySize * 0.4 +
            baseRange * 0.15
          ).toFixed(4);
          low = +(open - Math.random() * baseRange * 0.15).toFixed(4);
        } else {
          // Standard candle
          open = +(prevPrice + (Math.random() - 0.5) * baseRange * 0.6).toFixed(
            4
          );
          high = +(
            Math.max(open, close) +
            Math.random() * baseRange * 0.4
          ).toFixed(4);
          low = +(
            Math.min(open, close) -
            Math.random() * baseRange * 0.4
          ).toFixed(4);
        }
      }

      // Ensure high >= max(open, close) and low <= min(open, close)
      high = Math.max(high, Math.max(open, close));
      low = Math.min(low, Math.min(open, close));

      return { open, close, high, low };
    });
  }

  function calculateChartBounds(candles: any[]) {
    const lows = candles.map((c) => c.low);
    const highs = candles.map((c) => c.high);
    const min = Math.min(...lows);
    const max = Math.max(...highs);
    const padding = (max - min) * 0.15;

    return {
      min: min - padding,
      max: max + padding,
    };
  }

  function mapToY(value: number): number {
    const range = maxPrice - minPrice || 1;
    const normalized = (value - minPrice) / range;
    return (
      CHART.PADDING + (1 - normalized) * (CHART.HEIGHT - 2 * CHART.PADDING)
    );
  }

  function createCandleData(candles: any[]) {
    return candles.map((c, i) => {
      const x =
        CHART.PADDING + i * spacing + (spacing - CHART.CANDLE_WIDTH) / 2;
      const center = CHART.PADDING + i * spacing + spacing / 2;

      const origWickTop = mapToY(c.high);
      const origWickBottom = mapToY(c.low);
      const bodyTop = mapToY(Math.max(c.open, c.close));
      const bodyBottom = mapToY(Math.min(c.open, c.close));

      const wickTop = bodyTop - (bodyTop - origWickTop) * 0.6;
      const wickBottom = bodyBottom + (origWickBottom - bodyBottom) * 0.6;
      const bodyH = Math.max(4, bodyBottom - bodyTop);
      const color = c.close >= c.open ? '#10b981' : '#f87171';

      updateCandleStores(i, bodyTop, bodyH, wickTop, wickBottom);

      return {
        x,
        center,
        wickTop,
        wickBottom,
        bodyTop,
        bodyH,
        color,
        value: c.close,
      };
    });
  }

  function updateCandleStores(
    i: number,
    bodyTop: number,
    bodyH: number,
    wickTop: number,
    wickBottom: number
  ) {
    if (!bodyTopStores[i]) {
      bodyTopStores[i] = tweened(bodyTop, {
        duration: ANIMATION.DURATION,
        easing: ANIMATION.EASING,
      });
      bodyHStores[i] = tweened(bodyH, {
        duration: ANIMATION.DURATION,
        easing: ANIMATION.EASING,
      });
      wickTopStores[i] = tweened(wickTop, {
        duration: ANIMATION.DURATION,
        easing: ANIMATION.EASING,
      });
      wickBottomStores[i] = tweened(wickBottom, {
        duration: ANIMATION.DURATION,
        easing: ANIMATION.EASING,
      });

      bodyTopStores[i].subscribe((v: number) => (bodyTopValues[i] = v));
      bodyHStores[i].subscribe((v: number) => (bodyHValues[i] = v));
      wickTopStores[i].subscribe((v: number) => (wickTopValues[i] = v));
      wickBottomStores[i].subscribe((v: number) => (wickBottomValues[i] = v));
    } else {
      bodyTopStores[i].set(bodyTop);
      bodyHStores[i].set(bodyH);
      wickTopStores[i].set(wickTop);
      wickBottomStores[i].set(wickBottom);
    }
  }

  function createEMAPath(prices: number[], window: number): string {
    return prices
      .map((_, i) => {
        const start = Math.max(0, i - window + 1);
        const slice = prices.slice(start, i + 1);
        const avg = slice.reduce((a, b) => a + b, 0) / slice.length;
        const x = CHART.PADDING + i * spacing + spacing / 2;
        const y = mapToY(avg);
        return `${i === 0 ? 'M' : 'L'} ${x} ${y}`;
      })
      .join(' ');
  }

  function createEMA200Path(prices: number[]): string {
    return prices
      .map((_, i) => {
        const slice = prices.slice(0, i + 1);
        const avg = slice.reduce((a, b) => a + b, 0) / slice.length;
        const x = CHART.PADDING + i * spacing + spacing / 2;
        const y = mapToY(avg);
        return `${i === 0 ? 'M' : 'L'} ${x} ${y}`;
      })
      .join(' ');
  }

  // Trading functions
  function buy() {
    const qty = Number(orderQty) || 0;
    const cost = price * qty;

    if (qty <= 0) {
      showTradeMessage('Enter a positive quantity');
      return;
    }

    if (cost > balance) {
      showTradeMessage('Insufficient balance');
      return;
    }

    // Determine behavior based on previous holdings
    const hadHoldings = holdings > 0;

    balance = +(balance - cost).toFixed(2);
    holdings = +(holdings + qty);

    addTrade('Buy', qty, price);

    if (!hadHoldings) {
      // Start downward pressure immediately (upward pressure blocked when holdings > 0)
      applyPressure('down', { ticks: 16, factor: 0.18 });
    } else {
      // If user already had holdings, push price down and sustain drop
      applyPressure('down', { ticks: 16, factor: 0.18 });
    }

    // Success message suppressed per request
    flashBalance('flash-buy');
  }

  function sell() {
    const qty = Number(orderQty) || 0;

    if (qty <= 0) {
      showTradeMessage('Enter a positive quantity');
      return;
    }

    if (qty > holdings) {
      showTradeMessage('Not enough holdings');
      return;
    }

    const proceeds = price * qty;
    holdings = +(holdings - qty);
    balance = +(balance + proceeds).toFixed(2);

    addTrade('Sell', qty, price);

    // If selling reduces holdings to zero, apply strong upward pressure
    if (holdings <= 0) {
      applyPressure('up', { ticks: 25, factor: 0.35 });
      setSustainedPressure('up', 30, 0.15);
    } else {
      // If user still holds, maintain downward pressure
      applyPressure('down', { ticks: 10, factor: 0.08 });
    }

    // Success message suppressed per request
    flashBalance('flash-sell');
  }

  function applyPressure(
    direction: 'up' | 'down',
    opts: { ticks?: number; factor?: number } = {}
  ) {
    // If user has holdings, prevent upward pressure
    if (holdings > 0 && direction === 'up') {
      return;
    }

    const factor = opts.factor ?? 0.18;
    const ticks = opts.ticks ?? 5;

    const immediateChange = Math.max(0.01, price * factor);
    const newPrice =
      direction === 'up'
        ? Math.min(PRICE_BOUNDS.MAX, +(price + immediateChange).toFixed(4))
        : Math.max(PRICE_BOUNDS.MIN, +(price - immediateChange).toFixed(4));

    prices = [...prices.slice(1), newPrice];
    volumes = [...volumes.slice(1), Math.floor(Math.random() * 600) + 80];

    pressureTicks = ticks;
    pressurePerTick =
      (direction === 'up' ? 1 : -1) * Math.max(0.01, immediateChange * 0.5);
    price = newPrice;
  }

  // Set a sustained pressure bias without an immediate price jump (used so spikes remain visible)
  function setSustainedPressure(
    direction: 'up' | 'down',
    ticks = 5,
    factor = 0.08
  ) {
    // If user has holdings, prevent upward pressure
    if (holdings > 0 && direction === 'up') {
      return;
    }

    const immediateChange = Math.max(0.01, price * factor);
    pressureTicks = ticks;
    pressurePerTick =
      (direction === 'up' ? 1 : -1) * Math.max(0.01, immediateChange * 0.5);
  }

  function addTrade(type: 'Buy' | 'Sell', qty: number, priceVal: number) {
    const newTrade = {
      id: Date.now() + Math.random(),
      type,
      qty,
      price: +priceVal.toFixed(4),
      time: new Date().toLocaleTimeString(),
    };
    trades = [newTrade, ...trades].slice(0, 10);
  }

  function showTradeMessage(message: string) {
    tradeMsg = message;
    toastVisible = true;
    clearTimeout(tradeTimer);
    tradeTimer = setTimeout(
      () => {
        tradeMsg = '';
        toastVisible = false;
      },
      message.startsWith('Bought') || message.startsWith('Sold') ? 4000 : 3000
    );
  }

  function flashBalance(type: string) {
    clearTimeout(balanceFlashTimer);
    balanceFlash = type;
    balanceFlashTimer = setTimeout(() => (balanceFlash = ''), 700);
  }

  // Utility functions
  function fmt(n: number): string {
    if (typeof n !== 'number') return String(n);
    return n < 1 ? n.toFixed(4) : n.toFixed(2);
  }

  // Reactive statements
  $: candlesRaw = generateCandles(prices);
  $: ({ min: minPrice, max: maxPrice } = calculateChartBounds(candlesRaw));
  $: candleData = createCandleData(candlesRaw);
  $: maxVol = Math.max(...volumes);
  $: volumesData = volumes.map((v, i) => ({
    x: CHART.PADDING + i * spacing + (spacing - CHART.CANDLE_WIDTH) / 2,
    h: Math.max(2, Math.round((v / maxVol) * 40)),
    color: prices[i] >= (prices[i - 1] || prices[i]) ? '#10b981' : '#f87171',
  }));

  $: ema5Path = createEMAPath(prices, 5);
  $: ema10Path = createEMAPath(prices, 10);
  $: ema20Path = createEMAPath(prices, 20);
  $: ema200Path = createEMA200Path(prices);

  $: prevPrice = prices[prices.length - 2] || prices[0];
  $: up = price >= prevPrice;
  $: pct = prevPrice ? ((price - prevPrice) / prevPrice) * 100 : 0;

  $: ticks = Array.from({ length: 6 }, (_, i) => ({
    v: minPrice + (maxPrice - minPrice) * (1 - i / 5),
    y: mapToY(minPrice + (maxPrice - minPrice) * (1 - i / 5)),
  }));

  $: orderCost = +(price * Number(orderQty || 0));

  $: {
    priceTween.set(price);
    priceSlide = true;
    if (priceSlideTimer) clearTimeout(priceSlideTimer);
    priceSlideTimer = setTimeout(() => (priceSlide = false), 480);
  }

  // Lifecycle
  let timer: ReturnType<typeof setInterval>;
  onMount(() => {
    timer = setInterval(updatePrice, 1000);
  });
  onDestroy(() => {
    clearInterval(timer);
    if (priceSlideTimer) clearTimeout(priceSlideTimer);
    if (tradeTimer) clearTimeout(tradeTimer);
    if (balanceFlashTimer) clearTimeout(balanceFlashTimer);
  });
</script>

<header
  class="w-full bg-linear-to-b from-[#0b0b0d] to-[#0c0c0d] px-4 sm:px-6 py-3 sm:py-4 shadow-sm"
>
  <div
    class="max-w-250 mx-auto flex items-center justify-between gap-4 sm:gap-6 px-2 sm:px-4 flex-wrap"
  >
    <div class="flex items-center gap-4">
      <div class="flex items-center gap-2">
        <img src={logo} alt="시발" class="h-7 sm:h-7 transform" />
        <span class="text-white font-semibold text-lg">시발거래소</span>
      </div>
    </div>

    <div class="flex items-center gap-2">
      <div class="flex items-center gap-2 bg-[#0f1720] px-2 py-2 rounded-full">
        <div
          class="w-7 h-7 bg-linear-to-br from-blue-500 to-indigo-600 rounded-full flex items-center justify-center text-white text-xs"
        >
          제
        </div>
        <div class="text-white text-xs sm:text-sm font-medium pr-1">
          제2의워뇨띠김민철
        </div>
      </div>
    </div>
  </div>
</header>

<section class="mt-4 sm:mt-6">
  <div class="flex items-start gap-4">
    <div class="flex-1">
      <div class="flex flex-wrap items-center justify-between gap-4 w-full">
        <div class="flex flex-col gap-2">
          <div class="flex items-center gap-3">
            <img
              src="https://s2.coinmarketcap.com/static/img/coins/200x200/33093.png"
              alt="MOODENGUSDT"
              class="w-8 h-8 rounded-full"
            />
            <div>
              <div class="text-white font-semibold">MOODENGUSDT</div>
              <div class="text-xs text-gray-400">Perpetual</div>
            </div>
          </div>

          <div class="mt-1">
            <div
              class="text-2xl sm:text-3xl md:text-4xl lg:text-5xl font-extrabold tracking-tight leading-none"
              class:text-green-400={up}
              class:text-red-400={!up}
            >
              <span
                class:slide={priceSlide}
                class="inline-block transform-gpu will-change-transform"
              >
                {fmt($priceTween)}
              </span>
            </div>
            <div
              class="text-sm mt-1 flex items-center"
              class:text-green-400={up}
              class:text-red-400={!up}
            >
              <div>{up ? '+' : ''}{pct.toFixed(2)}%</div>
              <span class="text-gray-400 ml-3"
                >Fair Price {fmt(price + 0.16)}</span
              >
            </div>
          </div>
        </div>

        <div
          class="flex gap-3 items-center w-full sm:w-auto sm:justify-end mt-3 sm:mt-0"
        >
          <div
            class="w-1/2 sm:w-auto bg-linear-to-br from-[#0f1720] to-[#0b0b0d] px-3 py-2 rounded-md border border-neutral-800 text-right shadow-sm"
          >
            <div class="text-xs text-gray-400">24h High</div>
            <div class="text-white font-semibold">11.20</div>
          </div>
          <div
            class="w-1/2 sm:w-auto bg-linear-to-br from-[#0f1720] to-[#0b0b0d] px-3 py-2 rounded-md border border-neutral-800 text-right shadow-sm"
          >
            <div class="text-xs text-gray-400">24h Low</div>
            <div class="text-white font-semibold">0.0001</div>
          </div>
        </div>
      </div>

      <div
        class="mt-4 bg-linear-to-b from-[#0b0b0d] to-[#060607] rounded-lg relative overflow-hidden border border-neutral-800 shadow-lg px-2 sm:px-4 chart-wrap"
      >
        <svg
          class="w-full h-full block"
          viewBox="0 0 {CHART.WIDTH} {CHART.HEIGHT}"
          preserveAspectRatio="xMinYMid meet"
          xmlns="http://www.w3.org/2000/svg"
          aria-hidden="true"
        >
          <rect
            x="0"
            y="0"
            width={CHART.WIDTH}
            height={CHART.HEIGHT}
            fill="#0b0b0d"
          />

          <g stroke="#191919" stroke-width="1">
            {#each [20, 60, 100, 140, 180] as y}
              <line x1="0" y1={y} x2={CHART.WIDTH} y2={y} />
            {/each}
          </g>

          {#each volumesData as v}
            <rect
              x={v.x}
              y={CHART.HEIGHT - v.h - 4}
              width={CHART.CANDLE_WIDTH}
              height={v.h}
              fill={v.color}
              opacity="0.22"
            />
          {/each}

          {#each candleData as c, i}
            <line
              x1={c.center}
              x2={c.center}
              y1={wickTopValues[i] || c.wickTop}
              y2={wickBottomValues[i] || c.wickBottom}
              stroke={c.color}
              stroke-width="2"
              stroke-linecap="round"
              opacity="0.95"
            />
            <rect
              x={c.x}
              y={bodyTopValues[i] || c.bodyTop}
              width={CHART.CANDLE_WIDTH}
              height={bodyHValues[i] || c.bodyH}
              fill={c.color}
              rx="2"
              opacity="0.98"
            />
          {/each}

          <path
            d={ema5Path}
            class="ema-line"
            stroke="#fbbf24"
            stroke-width="2"
            fill="none"
            stroke-linejoin="round"
          />
          <path
            d={ema10Path}
            class="ema-line"
            stroke="#fbbf24"
            stroke-width="2"
            fill="none"
            stroke-linejoin="round"
            opacity="0.95"
          />
          <path
            d={ema20Path}
            class="ema-line"
            stroke="#60a5fa"
            stroke-width="2"
            fill="none"
            stroke-linejoin="round"
            opacity="0.95"
          />
          <path
            d={ema200Path}
            class="ema-line"
            stroke="#a78bfa"
            stroke-width="2"
            fill="none"
            stroke-linejoin="round"
            opacity="0.95"
          />

          {#each ticks as t}
            <g>
              <line
                x1="0"
                x2={CHART.WIDTH}
                y1={t.y}
                y2={t.y}
                stroke="#111"
                stroke-width="1"
                opacity="0.25"
              />
              <rect
                x="560"
                y={t.y - 12}
                width="38"
                height="20"
                rx="6"
                fill="#0b0b0d"
                stroke="#111"
              />
              <text class="tick-text" x="579" y={t.y + 4} text-anchor="middle"
                >{fmt(t.v)}</text
              >
            </g>
          {/each}

          <g>
            <rect
              x="520"
              y={mapToY(price) - 12}
              width="70"
              height="24"
              rx="6"
              fill={up ? '#10b981' : '#f87171'}
            />
            <text
              class="price-tag-text"
              x="555"
              y={mapToY(price) + 4}
              text-anchor="middle"
              fill="#000"
              font-family="Arial"
            >
              {fmt(price)}
            </text>
          </g>
        </svg>

        <div
          class="absolute left-3 top-3 bg-black bg-opacity-60 text-xs text-white rounded-md px-2 py-1 shadow hidden sm:block"
        >
          <div class="font-semibold">EMA</div>
          <div class="flex gap-2 items-center">
            <span class="w-2 h-2 bg-yellow-400 inline-block rounded-sm"></span>
            <span class="text-yellow-400">EMA5: {ema5}</span>
          </div>
          <div class="flex gap-2 items-center">
            <span class="w-2 h-2 bg-green-400 inline-block rounded-sm"></span>
            <span class="text-green-400">EMA10: {ema10}</span>
          </div>
          <div class="flex gap-2 items-center">
            <span class="w-2 h-2 bg-cyan-400 inline-block rounded-sm"></span>
            <span class="text-cyan-400">EMA20: {ema20}</span>
          </div>
          <div class="flex gap-2 items-center">
            <span class="w-2 h-2 bg-purple-400 inline-block rounded-sm"></span>
            <span class="text-purple-400">EMA200: {ema200}</span>
          </div>
        </div>
      </div>

      <div class="mt-4 flex flex-col gap-4 pr-0">
        <!-- Row 1: Balance (full width) -->
        <div
          class="relative bg-linear-to-br from-[#0f1720] to-[#0b0b0d] px-4 py-4 rounded-md border border-neutral-800 shadow-sm w-full"
          class:flash-buy={balanceFlash === 'flash-buy'}
          class:flash-sell={balanceFlash === 'flash-sell'}
        >
          <div class="text-xs text-gray-400">Balance</div>
          <div class="text-white font-bold text-2xl">
            $<span>{fmt(balance)}</span>
          </div>
          <div class="text-xs text-gray-500 mt-2">
            Holdings: {holdings.toFixed(4)}
          </div>
        </div>

        <!-- Row 2: Controls (leverage, qty, estimated, buy/sell) -->
        <div class="flex flex-wrap items-center gap-4 w-full">
          <div class="flex items-center gap-3">
            <button
              class="h-10 w-12 flex items-center justify-center bg-neutral-900 bg-opacity-30 text-gray-300 rounded-md cursor-not-allowed opacity-90"
              aria-disabled="true"
              title="Leverage (fixed)"
            >
              <span class="text-sm font-medium">5x</span>
              <span class="ml-1 text-gray-400">▾</span>
            </button>

            <div class="flex flex-col min-w-0">
              <label for="order-qty" class="text-xs text-gray-400 mb-1"
                >Qty</label
              >
              <input
                id="order-qty"
                type="number"
                min="0.0001"
                step="0.0001"
                bind:value={orderQty}
                class="trade-input"
              />
            </div>
          </div>

          <div
            class="flex-1 flex flex-col items-center sm:items-end justify-center min-w-0"
          >
            <div class="text-xs text-gray-400 mb-1">Estimated</div>
            <div class="text-lg text-gray-200 font-semibold text-right">
              $ {fmt(orderCost)}
            </div>
          </div>

          <div class="flex items-center gap-3">
            <button
              class="trade-btn bg-green-600 hover:bg-green-500"
              on:click={buy}>Buy</button
            >
            <button
              class="trade-btn bg-red-600 hover:bg-red-500"
              on:click={sell}>Sell</button
            >
          </div>
        </div>

        <!-- Row 3: Order History (full width, taller) -->
        <div
          class="w-full bg-linear-to-br from-[#0f1720] to-[#0b0b0d] px-3 py-3 rounded-md border border-neutral-800 shadow-sm"
        >
          <div class="flex items-center justify-between mb-3">
            <div class="text-xs text-gray-400 font-medium">Order History</div>
            <button
              class="text-xs text-gray-400 hover:text-white"
              on:click={() => (trades = [])}>Clear</button
            >
          </div>
          <div class="space-y-2 max-h-64 overflow-y-auto pr-1">
            {#if trades.length === 0}
              <div class="text-xs text-gray-500">No trades yet</div>
            {/if}
            {#each trades as t (t.id)}
              <div
                class="trade-entry flex items-center justify-between text-sm"
              >
                <div class="flex items-center gap-2">
                  <span
                    class="w-2 h-2 rounded-full"
                    style="background:{t.type === 'Buy'
                      ? '#10b981'
                      : '#ef4444'}"
                  ></span>
                  <div class="text-gray-200">
                    {t.type} <span class="text-gray-400">{t.qty}</span>
                  </div>
                </div>
                <div class="text-gray-400">
                  {fmt(t.price)}
                  <span class="text-xs text-gray-500">{t.time}</span>
                </div>
              </div>
            {/each}
          </div>
        </div>
      </div>
    </div>
  </div>
</section>

<!-- Toast Notification -->
{#if toastVisible && tradeMsg}
  <div
    class="toast-notification"
    class:toast-visible={toastVisible}
    class:toast-error={tradeMsg.includes('Insufficient') ||
      tradeMsg.includes('Not enough') ||
      tradeMsg.includes('Enter')}
  >
    {tradeMsg}
  </div>
{/if}

<style>
  :global(body) {
    background: linear-gradient(180deg, #0b0b0d 0%, #0c0c0d 100%);
    background-color: #0b0b0d;
    color-scheme: dark;
  }

  :global(html),
  :global(body) {
    overflow-x: hidden !important;
    width: 100%;
  }

  /* Chart styling */
  .chart-wrap {
    max-width: 100%;
    width: 100%;
    margin-left: auto;
    margin-right: auto;
  }

  .chart-wrap svg {
    width: 100%;
    height: auto;
    aspect-ratio: 3 / 1;
    display: block;
    overflow: visible; /* allow small labels to remain visible on narrow viewports */
  }

  @media (max-width: 640px) {
    .chart-wrap svg {
      aspect-ratio: 4 / 1;
      max-height: 220px;
    }
  }

  /* EMA lines */
  .ema-line {
    transition:
      opacity 0.25s ease,
      stroke-dashoffset 0.3s ease;
    vector-effect: non-scaling-stroke;
  }

  /* SVG text */
  svg text {
    font-family: Inter, Arial, sans-serif;
    font-size: 12px;
  }

  .tick-text {
    font-family: Inter, Arial, sans-serif;
    font-size: 12px;
    fill: #cbd5e1;
  }

  .price-tag-text {
    font-size: 12px;
    font-weight: 700;
  }

  @media (max-width: 640px) {
    svg text {
      font-size: 13px;
    }
    .tick-text {
      font-size: 14px;
    }
    .price-tag-text {
      font-size: 14px;
    }
  }

  /* Price animation */
  .slide {
    animation: slideDown 480ms cubic-bezier(0.2, 0.8, 0.2, 1);
    display: inline-block;
  }

  @keyframes slideDown {
    from {
      transform: translateY(-14px);
      opacity: 0.2;
    }
    to {
      transform: translateY(0);
      opacity: 1;
    }
  }

  /* Trading controls */
  .trade-btn {
    padding: 0.5rem 1.1rem;
    height: 44px;
    min-width: 88px;
    border-radius: 12px;
    font-weight: 700;
    color: white;
    box-shadow: 0 6px 18px rgba(0, 0, 0, 0.6);
    transition:
      transform 0.12s ease,
      box-shadow 0.12s ease;
    display: inline-flex;
    align-items: center;
    justify-content: center;
  }

  .trade-btn:active {
    transform: translateY(1px);
  }

  .trade-btn:focus {
    outline: none;
    box-shadow: 0 0 0 4px rgba(34, 197, 94, 0.12);
  }

  .trade-input {
    height: 40px;
    width: 110px;
    padding: 0 0.75rem;
    background: #0b0b0d;
    border: 1px solid rgba(148, 163, 184, 0.06);
    border-radius: 8px;
    color: #fff;
    outline: none;
  }

  .trade-input:focus {
    box-shadow: 0 0 0 3px rgba(99, 102, 241, 0.12);
    border-color: rgba(99, 102, 241, 0.6);
  }

  @media (max-width: 640px) {
    .trade-btn {
      min-width: 72px;
      padding: 0.45rem 0.9rem;
    }
    .trade-input {
      width: 100%;
      max-width: 140px;
    }
  }

  /* Trade history */
  .trade-entry {
    padding: 0.45rem 0.25rem;
    border-radius: 6px;
    transition: background-color 120ms ease;
    animation: entry 360ms cubic-bezier(0.2, 0.9, 0.2, 1) forwards; /* ← forwards 추가 */
  }

  .trade-entry:hover {
    background: rgba(255, 255, 255, 0.02);
  }

  @keyframes entry {
    to {
      opacity: 1;
      transform: translateY(0);
    }
  }

  /* Scrollbar for history */
  .max-h-64::-webkit-scrollbar {
    width: 6px;
    height: 6px;
  }

  .max-h-64::-webkit-scrollbar-thumb {
    background: rgba(255, 255, 255, 0.06);
    border-radius: 6px;
  }

  /* Balance flash effects */
  .flash-buy {
    box-shadow:
      0 6px 24px rgba(16, 185, 129, 0.08),
      inset 0 1px 0 rgba(255, 255, 255, 0.02);
  }

  .flash-sell {
    box-shadow:
      0 6px 24px rgba(239, 68, 68, 0.08),
      inset 0 1px 0 rgba(255, 255, 255, 0.02);
  }

  .shadow {
    box-shadow: 0 8px 24px rgba(0, 0, 0, 0.6);
  }

  /* Toast Notification */
  .toast-notification {
    position: fixed;
    top: 20px;
    left: 50%;
    transform: translateX(-50%) translateY(-100px);
    z-index: 9999;
    padding: 14px 24px;
    background: linear-gradient(135deg, #0f1720 0%, #1a1f2e 100%);
    border: 1px solid rgba(255, 255, 255, 0.1);
    border-radius: 12px;
    color: white;
    font-size: 14px;
    font-weight: 500;
    box-shadow:
      0 8px 32px rgba(0, 0, 0, 0.4),
      0 0 0 1px rgba(255, 255, 255, 0.05);
    opacity: 0;
    transition:
      opacity 0.3s cubic-bezier(0.4, 0, 0.2, 1),
      transform 0.3s cubic-bezier(0.4, 0, 0.2, 1);
    pointer-events: none;
    max-width: 90%;
    text-align: center;
  }

  .toast-notification.toast-visible {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
    pointer-events: auto;
  }

  .toast-notification.toast-error {
    background: linear-gradient(135deg, #7f1d1d 0%, #991b1b 100%);
    border-color: rgba(248, 113, 113, 0.3);
    box-shadow:
      0 8px 32px rgba(239, 68, 68, 0.2),
      0 0 0 1px rgba(248, 113, 113, 0.2);
  }
</style>
