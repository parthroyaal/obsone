

```python
# Mutliple tf version of write1.py
from flask import Flask, render_template_string, json
from flask_sock import Sock
import time
import random
import threading

###################################
# Initialize Flask, Sock and Globals
###################################
app = Flask(__name__)
sock = Sock(app)

# Global state for pricing and candle data
current_price = None
INITIAL_TICKS = []      # Historic raw tick data (1-minute resolution)
CANDLES = []            # Historic aggregated 1-minute candles
LIVE_TICKS = []         # Unprocessed realtime tick updates
CONNECTIONS = set()     # WebSocket connections
LOCK = threading.Lock()

###################################
# Helper functions for tick and candle generation
###################################

def generate_mock_ticks(num_rows, start_index=0):
    """
    Generate num_rows of tick data starting from a given start index.
    Uses the current_price as a base and applies random fluctuations.
    """
    global current_price
    base_time = int(time.time()) - 172800  # 2 days ago

    if current_price is None:
        current_price = 100

    ticks = []
    for i in range(num_rows):
        delta = random.uniform(-0.5, 0.5)  # Price fluctuation for each tick.
        current_price += delta
        timestamp = base_time + start_index + i
        ticks.append({
            "timestamp": timestamp,
            "value": round(current_price, 2)
        })
    return ticks

def create_candle_from_tick(tick):
    """
    Create a new candle from a tick; uses the tick's timestamp for the candle's time slot.
    """
    return {
        'time': tick['timestamp'] - (tick['timestamp'] % 60),
        'open': tick['value'],
        'high': tick['value'],
        'low': tick['value'],
        'close': tick['value']
    }

def create_initial_candles(ticks):
    """
    Create initial 1-minute candles from a list of ticks.
    """
    candles = []
    for tick in ticks:
        if not candles:
            candles.append(create_candle_from_tick(tick))
            continue

        last_candle = candles[-1]
        candle_time = tick['timestamp'] - (tick['timestamp'] % 60)
        if last_candle['time'] == candle_time:
            last_candle['high'] = max(last_candle['high'], tick['value'])
            last_candle['low'] = min(last_candle['low'], tick['value'])
            last_candle['close'] = tick['value']
        else:
            candles.append(create_candle_from_tick(tick))
    return candles

def update_candle_with_tick(tick):
    """
    Updates latest candle with tick data if the tick's timestamp falls within its interval.
    Otherwise, creates a new candle.
    """
    global CANDLES
    candle_time = tick['timestamp'] - (tick['timestamp'] % 60)
    if CANDLES:
        last_candle = CANDLES[-1]
        if last_candle['time'] == candle_time:
            last_candle['high'] = max(last_candle['high'], tick['value'])
            last_candle['low'] = min(last_candle['low'], tick['value'])
            last_candle['close'] = tick['value']
            return last_candle

    new_candle = create_candle_from_tick(tick)
    CANDLES.append(new_candle)
    return new_candle

def initialize_history():
    """
    Populates INITIAL_TICKS and creates the initial set of 1-minute CANDLES.
    """
    global INITIAL_TICKS, CANDLES
    INITIAL_TICKS = generate_mock_ticks(172800)  # 2 days worth of ticks.
    CANDLES = create_initial_candles(INITIAL_TICKS)

initialize_history()

###################################
# Background Threads:
# 1. Tick Generator Thread: Generates new ticks every second.
# 2. Candle Update Thread: Processes new ticks in LIVE_TICKS, updates candles, and broadcasts updates.
###################################

def tick_generator_thread():
    while True:
        time.sleep(1)
        new_tick = generate_mock_ticks(1, len(INITIAL_TICKS) + len(LIVE_TICKS))[0]
        with LOCK:
            LIVE_TICKS.append(new_tick)

def candle_update_thread():
    while True:
        time.sleep(1)  # Adjust interval as needed.
        with LOCK:
            if LIVE_TICKS:
                for tick in LIVE_TICKS:
                    updated_candle = update_candle_with_tick(tick)
                    message = json.dumps({'candle': updated_candle})
                    # Broadcast the updated candle to all connected websocket clients.
                    for ws in CONNECTIONS.copy():
                        try:
                            ws.send(message)
                        except Exception as e:
                            CONNECTIONS.remove(ws)
                LIVE_TICKS.clear()

###################################
# WebSocket Endpoint
###################################
@sock.route('/realtime')
def realtime(ws):
    with LOCK:
        CONNECTIONS.add(ws)
    try:
        while True:
            ws.receive()  # Keeps the connection open.
    finally:
        with LOCK:
            CONNECTIONS.remove(ws)

###################################
# Historic Data Endpoint: Serves complete 1-minute candle history.
###################################
@app.route('/historic')
def historic():
    return json.dumps(CANDLES)

###################################
# HTML Template with Multi-Timeframe Support
# This template includes buttons for selecting different timeframes.
# It uses a client-side resampler function to aggregate the 1-minute candles.
###################################
index_html = """
<!DOCTYPE html>
<html>
<head>
    <title>Real-Time Candlestick Chart with Multiple Timeframes</title>
    <script src="https://cdn.jsdelivr.net/npm/lightweight-charts@3.8.0/dist/lightweight-charts.standalone.production.js"></script>
    <style>
      #buttons {
          margin-bottom: 10px;
      }
      .tf-button {
          padding: 5px 10px;
          margin-right: 5px;
          cursor: pointer;
      }
    </style>
</head>
<body>
    <div id="buttons">
        <button class="tf-button" data-timeframe="1">1 Minute</button>
        <button class="tf-button" data-timeframe="3">3 Minute</button>
        <button class="tf-button" data-timeframe="5">5 Minute</button>
        <button class="tf-button" data-timeframe="10">10 Minute</button>
        <button class="tf-button" data-timeframe="15">15 Minute</button>
        <button class="tf-button" data-timeframe="30">30 Minute</button>
        <button class="tf-button" data-timeframe="60">1 Hour</button>
    </div>
    <div id="chart"></div>
    <script>
        // Create the chart
        const chart = LightweightCharts.createChart(document.getElementById('chart'), {
            width: window.innerWidth,
            height: window.innerHeight,
            timeScale: { timeVisible: true, secondsVisible: false }
        });
        const candleSeries = chart.addCandlestickSeries();

        // Local storage for 1-minute candles from /historic and realtime updates
        let oneMinuteCandles = [];
        // Default selected timeframe (in minutes)
        let currentTimeframe = 1;

        // Resample 1-minute candles into a higher timeframe given in minutes.
        function resampleCandles(candles, timeframeMinutes) {
            const resampled = [];
            const timeframeSeconds = timeframeMinutes * 60;
            const groups = {};

            candles.forEach(candle => {
                // Group interval: floor(candle.time / timeframeSeconds) * timeframeSeconds
                const groupTime = Math.floor(candle.time / timeframeSeconds) * timeframeSeconds;
                if (!groups[groupTime]) {
                    groups[groupTime] = {
                        time: groupTime,
                        open: candle.open,
                        high: candle.high,
                        low: candle.low,
                        close: candle.close
                    };
                } else {
                    groups[groupTime].high = Math.max(groups[groupTime].high, candle.high);
                    groups[groupTime].low = Math.min(groups[groupTime].low, candle.low);
                    groups[groupTime].close = candle.close;
                }
            });
            Object.values(groups).forEach(aggCandle => resampled.push(aggCandle));
            resampled.sort((a, b) => a.time - b.time);
            return resampled;
        }

        // Update chart with either raw 1-minute candles or resampled data
        function updateChart() {
            const aggregated = currentTimeframe === 1 ? oneMinuteCandles : resampleCandles(oneMinuteCandles, currentTimeframe);
            candleSeries.setData(aggregated);
        }
        
        // Fetch historic 1-minute candles
        fetch('/historic')
            .then(response => response.json())
            .then(data => {
                oneMinuteCandles = data;
                updateChart();
            });

        // Setup WebSocket for realtime updates
        const ws = new WebSocket(`ws://${window.location.host}/realtime`);
        ws.onmessage = function(event) {
            const msg = JSON.parse(event.data);
            const candle = msg.candle;
            // Update local 1-minute candle history:
            if(oneMinuteCandles.length && oneMinuteCandles[oneMinuteCandles.length - 1].time === candle.time) {
                oneMinuteCandles[oneMinuteCandles.length - 1] = candle;
            } else {
                oneMinuteCandles.push(candle);
            }
            updateChart();
        };

        // Add event listeners for timeframe buttons
        document.querySelectorAll('.tf-button').forEach(button => {
            button.addEventListener('click', () => {
                currentTimeframe = Number(button.getAttribute('data-timeframe'));
                // Optionally, refetch full history if needed.
                fetch('/historic')
                    .then(response => response.json())
                    .then(data => {
                        oneMinuteCandles = data;
                        updateChart();
                    });
            });
        });
    </script>
</body>
</html>
"""

###################################
# Root Endpoint serving the HTML
###################################
@app.route('/')
def index():
    return render_template_string(index_html)

###################################
# Run the Flask App and Start Background Threads
###################################
if __name__ == '__main__':
    threading.Thread(target=tick_generator_thread, daemon=True).start()
    threading.Thread(target=candle_update_thread, daemon=True).start()
    app.run(debug=True, port=80)
```