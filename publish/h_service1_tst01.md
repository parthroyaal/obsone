
```python
from flask import Flask, render_template_string, json
from flask_sock import Sock
import pandas as pd
import time
import random
import threading

###################################
###################################
app = Flask(__name__)
sock = Sock(app)

# Global state
current_price = None
INITIAL_TICKS = []
CANDLES = []
LIVE_TICKS = []
CONNECTIONS = set()
LOCK = threading.Lock()

###################################
###################################
def generate_mock_ticks(num_rows, start_index=0):
    global current_price
    base_time = int(time.time()) - 172800  # 2 days ago

    if current_price is None:
        current_price = 100

    ticks = []
    for i in range(num_rows):
        delta = random.uniform(-0.5, 0.5)
        current_price += delta
        timestamp = base_time + start_index + i
        ticks.append({
            "timestamp": timestamp,
            "value": round(current_price, 2)
        })
    return ticks

def create_candle_from_tick(tick):
    return {
        'time': tick['timestamp'] - (tick['timestamp'] % 60),
        'open': tick['value'],
        'high': tick['value'],
        'low': tick['value'],
        'close': tick['value']
    }

def create_initial_candles(ticks):
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
    global INITIAL_TICKS, CANDLES
    INITIAL_TICKS = generate_mock_ticks(172800)  # 2 days of ticks
    CANDLES = create_initial_candles(INITIAL_TICKS)

initialize_history()

###################################
###################################
# Thread responsible for generating new ticks
def tick_generator_thread():
    while True:
        time.sleep(1)
        new_tick = generate_mock_ticks(1, len(INITIAL_TICKS) + len(LIVE_TICKS))[0]
        with LOCK:
            LIVE_TICKS.append(new_tick)

###################################
###################################
# Thread responsible for processing ticks and updating candles
def candle_update_thread():
    while True:
        time.sleep(1)  # Adjust interval as needed
        with LOCK:
            if LIVE_TICKS:
                for tick in LIVE_TICKS:
                    updated_candle = update_candle_with_tick(tick)
                    message = json.dumps({'candle': updated_candle})
                    # Broadcast the updated candle to all connected websocket clients
                    for ws in CONNECTIONS.copy():
                        try:
                            ws.send(message)
                        except:
                            CONNECTIONS.remove(ws)
                LIVE_TICKS.clear()

###################################
###################################
@sock.route('/realtime')
def realtime(ws):
    with LOCK:
        CONNECTIONS.add(ws)
    try:
        while True:
            ws.receive()  # Keep connection open
    finally:
        with LOCK:
            CONNECTIONS.remove(ws)

###################################
###################################
@app.route('/historic')
def historic():
    # Serve complete candle history: initial candles + updates processed during live ticks
    return json.dumps(CANDLES)

###################################
###################################
# Minimal HTML for testing - unchanged for brevity, but uses candle updates on websocket
index_html = """
<!DOCTYPE html>
<html>
<head>
    <title>Real-Time Candlestick Chart</title>
    <script src="https://cdn.jsdelivr.net/npm/lightweight-charts@3.8.0/dist/lightweight-charts.standalone.production.js"></script>
</head>
<body>
    <div id="chart"></div>
    <script>
        const chart = LightweightCharts.createChart(document.getElementById('chart'), {
            width: window.innerWidth,
            height: window.innerHeight,
            timeScale: { timeVisible: true, secondsVisible: false }
        });
        const candleSeries = chart.addCandlestickSeries();

        fetch('/historic')
            .then(response => response.json())
            .then(data => {
                candleSeries.setData(data);
            });

        const ws = new WebSocket(`ws://${window.location.host}/realtime`);
        ws.onmessage = function(event) {
            const msg = JSON.parse(event.data);
            // Only update using the latest aggregated candle information
            const candle = msg.candle;
            candleSeries.update(candle);
        };
    </script>
</body>
</html>
"""
@app.route('/')
def index():
    return render_template_string(index_html)

###################################
###################################
if __name__ == '__main__':
    # Start the tick generator & candle update threads separately
    threading.Thread(target=tick_generator_thread, daemon=True).start()
    threading.Thread(target=candle_update_thread, daemon=True).start()
    app.run(debug=True, port=80)
    
```