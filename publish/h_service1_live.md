

```python

# https://chatgpt.com/c/67e3c255-f900-800a-b977-64f9473d62a6


from flask import Flask, render_template_string, json
from flask_sock import Sock

import json, time, logging, os
from datetime import datetime
from pytz import timezone
import pandas as pd

import threading

###################################################### Flask Setup #######################################################
app = Flask(__name__)
sock = Sock(app)

###################################################### Global Data ########################################################
# Global list to store all 5-second candles (historic + current)
CANDLES = []

# Global set and lock for managing WebSocket connections
from threading import Lock
LOCK = Lock()
CONNECTIONS = set()

# Global variable for managing the Fyers WebSocket thread
ws_client = None  
ws_thread = None

data_dir = '/var/lib/data'
if not os.path.exists(data_dir):
    print(f"Data directory {data_dir} does not exist.")
else:
    print(f"Data directory {data_dir} exists.")

###################################################### Logging Configuration ###############################################
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logging.info(f"Initial CANDLES: {CANDLES}")

###################################################### Candle Update Functions ####################################################
def broadcast_updated_candle(candle):
    """
    Broadcasts the updated candle to all active WebSocket connections.
    """
    message = json.dumps({'candle': candle})
    with LOCK:
        for ws in list(CONNECTIONS):
            try:
                ws.send(message)
            except Exception as e:
                logging.error("Error sending WS message: %s", e)
                CONNECTIONS.remove(ws)


def update_candle_with_tick(tick):
    """
    Processes an incoming tick to update the 5-second candle.
    If the tick falls within the current 5-second bucket, the candle is updated;
    otherwise, a new candle is created.
    After updating, the updated candle is broadcast to WS clients.
    """
    global CANDLES
    # Convert tick's exchange feed time to an integer timestamp
    tick_dt = datetime.utcfromtimestamp(tick["exch_feed_time"]).replace(microsecond=0)
    tick_timestamp = int(tick_dt.timestamp())
    # Determine the 5-second bucket
    candle_time = tick_timestamp - (tick_timestamp % 5)
    
    if CANDLES and CANDLES[-1]['time'] == candle_time:
        # Update the existing candle
        last_candle = CANDLES[-1]
        last_candle['high'] = max(last_candle['high'], tick["ltp"])
        last_candle['low'] = min(last_candle['low'], tick["ltp"])
        last_candle['close'] = tick["ltp"]
        updated_candle = last_candle
    else:
        # Create a new candle for a new 5-second bucket
        new_candle = {
            'time': candle_time,
            'open': tick["ltp"],
            'high': tick["ltp"],
            'low': tick["ltp"],
            'close': tick["ltp"]
        }
        CANDLES.append(new_candle)
        updated_candle = new_candle

    # Broadcast updated candle to all connected WebSocket clients
    broadcast_updated_candle(updated_candle)




###################################################### WebSocket Client Setup (Fyers) #######################################
def client_connect():
    """
    Connects to the Fyers WebSocket and processes ticks as they arrive.
    Each tick is immediately processed to update the candle and then broadcast.
    """
    import requests, base64, struct, hmac
    from fyers_apiv3 import fyersModel
    from urllib.parse import urlparse, parse_qs 
    pin = '8894'
    
    class FyesApp:
        def __init__(self) -> None:
            self.__username = 'XP12325'
            self.__totp_key = 'Q2HC7F57FHMHPRT2VRLPRWA4ORWPK34E'
            self.__pin = '8894'
            self.__client_id = "1BE74FZNXA-100"
            self.__secret_key = "P485IP4X2O"
            self.__redirect_uri = 'http://127.0.0.1:8081'
            self.__access_token = None

        def enable_app(self):
            appSession = fyersModel.SessionModel(
                client_id=self.__client_id,
                redirect_uri=self.__redirect_uri,
                response_type='code',
                state='state',
                secret_key=self.__secret_key,
                grant_type='authorization_code'
            )
            return appSession.generate_authcode()

        def __totp(self, key, time_step=30, digits=6, digest="sha1"):
            key = base64.b32decode(key.upper() + "=" * ((8 - len(key)) % 8))
            counter = struct.pack(">Q", int(time.time() / time_step))
            mac = hmac.new(key, counter, digest).digest()
            offset = mac[-1] & 0x0F
            binary = struct.unpack(">L", mac[offset: offset + 4])[0] & 0x7FFFFFFF
            return str(binary)[-digits:].zfill(digits)

        def get_token(self, refresh=False):
            try:
                if self.__access_token is None and refresh:
                    logging.error("Access token is None and refresh is True")
                    return

                headers = {
                    "Accept": "application/json",
                    "Accept-Language": "en-US,en;q=0.9",
                    "User-Agent": "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/112.0.0.0 Safari/537.36",
                }
                s = requests.Session()
                s.headers.update(headers)

                # Step 1: Send login OTP
                data1 = f'{{"fy_id":"{base64.b64encode(f"{self.__username}".encode()).decode()}","app_id":"2"}}'
                r1 = s.post("https://api-t2.fyers.in/vagator/v2/send_login_otp_v2", data=data1)
                logging.info(f"Step 1 Response: {r1.status_code} - {r1.text}")
                if r1.status_code != 200:
                    raise Exception(f"Failed to send OTP: {r1.text}")

                # Step 2: Verify OTP
                request_key = r1.json()["request_key"]
                totp_code = self.__totp(self.__totp_key)
                data2 = f'{{"request_key":"{request_key}","otp":{totp_code}}}'
                logging.info(f"TOTP Generated: {totp_code}")
                r2 = s.post("https://api-t2.fyers.in/vagator/v2/verify_otp", data=data2)
                logging.info(f"Step 2 Response: {r2.status_code} - {r2.text}")
                if r2.status_code != 200:
                    raise Exception(f"Failed to verify OTP: {r2.text}")

                request_key = r2.json()["request_key"]
                data3 = f'{{"request_key":"{request_key}","identity_type":"pin","identifier":"{base64.b64encode(f"{pin}".encode()).decode()}"}}'
                r3 = s.post("https://api-t2.fyers.in/vagator/v2/verify_pin_v2", data=data3)
                assert r3.status_code == 200, f"Error in r3:\n {r3.json()}"

                headers = {"authorization": f"Bearer {r3.json()['data']['access_token']}",
                           "content-type": "application/json; charset=UTF-8"}
                data4 = f'{{"fyers_id":"{self.__username}","app_id":"{self.__client_id[:-4]}","redirect_uri":"{self.__redirect_uri}","appType":"100","code_challenge":"","state":"abcdefg","scope":"","nonce":"","response_type":"code","create_cookie":true}}'
                r4 = s.post("https://api.fyers.in/api/v2/token", headers=headers, data=data4)
                assert r4.status_code == 308, f"Error in r4:\n {r4.json()}"

                parsed = urlparse(r4.json()["Url"])
                auth_code = parse_qs(parsed.query)["auth_code"][0]

                session = fyersModel.SessionModel(
                    client_id=self.__client_id,
                    secret_key=self.__secret_key,
                    redirect_uri=self.__redirect_uri,
                    response_type="code",
                    grant_type="authorization_code"
                )
                session.set_token(auth_code)
                response = session.generate_token()
                self.__access_token = response["access_token"]
                return self.__access_token
            except Exception as e:
                logging.error(f"Error in get_token: {str(e)}")
                raise

    # Get access token
    app_obj = FyesApp()
    access_token = app_obj.get_token()
    print(f'Access TOKEN: {access_token}')

    # Optionally, you can use historical data for initial candles.
    # In this example, we assume that historical data (if needed) is processed separately
    # and its candles are appended to the global CANDLES list.
    from fyers_apiv3 import fyersModel
    client_id = "1BE74FZNXA-100"

    def get_hist():
        global CANDLES

        fyers = fyersModel.FyersModel(client_id=client_id, is_async=False, token=access_token, log_path="./")
        
        data = {
            "symbol": "NSE:NIFTY50-INDEX",
            "resolution": "5S",
            "date_format": "1",
            "range_from": "2025-03-26",
            "range_to": "2025-03-27",
            "cont_flag": "1"
        }

        res = fyers.history(data=data)

        if "candles" in res:
            # Each candle is a list of 6 elements (time, open, high, low, close, volume)
            # We only want the first five, so we slice each candle accordingly.
            candles_sliced = [candle[:5] for candle in res['candles']]
            df = pd.DataFrame(candles_sliced, columns=['time', 'open', 'high', 'low', 'close'])

            # Convert time to integer timestamps if needed.
            df["time"] = df["time"].astype(int)

            # Convert historical timestamps from IST to UTC by subtracting 5.5 hours (19800 seconds)
            df["time"] = df["time"] - 19800

            CANDLES.extend(df.to_dict(orient="records"))

            logging.info(f"Historical candles appended. Total count: {len(CANDLES)}")
            # logging.info(f"CANDLES: {CANDLES}")
        else:
            logging.error("Failed to fetch historical data.")



    from fyers_apiv3.FyersWebsocket import data_ws

    def onmessage(message):
        """
        Callback function for handling incoming messages from Fyers WebSocket.
        Processes market data ticks to update the candle and broadcast the update.
        """
        logging.info(f"Raw message received: {message}")
        try:
            tick = json.loads(message) if isinstance(message, str) else message
            logging.info(f"Parsed JSON message: {tick}")
        except json.JSONDecodeError as e:
            logging.error(f"Failed to parse JSON message: {e}")
            return

        # Process only market data messages
        if "ltp" in tick and "exch_feed_time" in tick:
            update_candle_with_tick(tick)
            
        elif "type" in tick and tick["type"] in ["cn", "ful", "sub"]:
            logging.info(f"Received system message: {tick['type']} - {tick.get('message', '')}")
        else:
            logging.warning(f"Unexpected message format: {tick}")

    def onerror(message):
        logging.error("WebSocket Error: %s", message)

    def onclose(message):
        logging.info("WebSocket Connection Closed: %s", message)

    def onopen():
        # Subscribe to market data using the FyersDataSocket instance.
        data_type = "SymbolUpdate"
        symbols = ['NSE:NIFTY50-INDEX']
        ws_client.subscribe(symbols=symbols, data_type=data_type)
        ws_client.keep_running()

    global ws_client
    ws_client = data_ws.FyersDataSocket(
        access_token=access_token,  # Use your valid token here.
        log_path="",
        litemode=False,
        write_to_file=False,
        reconnect=True,
        on_connect=onopen,
        on_close=onclose,
        on_error=onerror,
        on_message=onmessage
    )
    # use get_hist() to append to global CANDLES
    get_hist()
    # Establish the connection (this call blocks until the connection is closed)
    ws_client.connect()

###################################################### Flask WebSocket Server #######################################################
@sock.route("/ws")
def ws_endpoint(ws):
    """
    Registers WebSocket connections.
    Clients connecting to /ws will receive updated candle data broadcast
    from the onmessage callback.
    """
    with LOCK:
        CONNECTIONS.add(ws)
    try:
        # Keep the connection open
        while True:
            ws.receive()  # This call blocks until a message is received (or connection is closed)
    except Exception as e:
        logging.error("WS endpoint error: %s", e)
    finally:
        with LOCK:
            CONNECTIONS.remove(ws)

###################################################### Historic Data Endpoint #######################################################
@app.route("/historic")
def historic():
    """
    Returns all candles (historic and the latest updated)
    so that the frontend chart always has the complete candle history.
    """
    return json.dumps(CANDLES)

###################################################### Frontend Setup #######################################################
@app.route("/")
def index():
    """
    Serves the frontend chart with timeframe selection.
    The page fetches /historic for the initial 5-second candle history and then uses
    a WS connection to receive real-time updated candle data. It includes buttons for
    multiple timeframes, and client-side code resamples the 5-second candles accordingly.
    """
    html = """
    <!DOCTYPE html>
    <html lang="en">
    <head>
      <meta charset="UTF-8">
      <meta name="viewport" content="width=device-width, initial-scale=1.0">
      <title>Live NIFTY Tick Chart (IST)</title>
      <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
      <style>
        #buttons { margin-bottom: 10px; }
        .tf-button { padding: 5px 10px; margin-right: 5px; cursor: pointer; }
        .active { background-color: #4CAF50; color: white; }
      </style>
    </head>
    <body>
      <h1>Live NIFTY Tick Chart (IST)</h1>
      <div id="buttons">
        <!-- Buttons for second-based timeframes -->
        <button class="tf-button active" data-timeframe="5s">5 Second</button>
        <button class="tf-button" data-timeframe="10s">10 Second</button>
        <button class="tf-button" data-timeframe="15s">15 Second</button>
        <button class="tf-button" data-timeframe="30s">30 Second</button>
        <button class="tf-button" data-timeframe="45s">45 Second</button>

        <!-- Buttons for minute-based timeframes -->
        <button class="tf-button" data-timeframe="1">1 Minute</button>
        <button class="tf-button" data-timeframe="3">3 Minute</button>
        <button class="tf-button" data-timeframe="5">5 Minute</button>
        <button class="tf-button" data-timeframe="10">10 Minute</button>
        <button class="tf-button" data-timeframe="15">15 Minute</button>
        <button class="tf-button" data-timeframe="30">30 Minute</button>
        <button class="tf-button" data-timeframe="60">1 Hour</button>
        
        <!-- Buttons for extended timeframes -->
        <button class="tf-button" data-timeframe="2h">2 Hour</button>
        <button class="tf-button" data-timeframe="4h">4 Hour</button>
        <button class="tf-button" data-timeframe="1d">1 Day</button>
        <button class="tf-button" data-timeframe="1w">1 Week</button>
        <button class="tf-button" data-timeframe="1month">1 Month</button>
      </div>
      <div id="chart"></div>
      <script>
        // Create the chart
        const chart = LightweightCharts.createChart(document.getElementById('chart'), {
          width: window.innerWidth,
          height: window.innerHeight,
          timeScale: {
            timeVisible: true,
            secondsVisible: true,
            tickMarkFormatter: (time) => {
               const utcDate = new Date(time * 1000); // Convert UNIX time to Date (UTC)
               const istDate = new Date(utcDate.getTime() + (5.5 * 3600 * 1000)); // Convert to IST
               return istDate.toLocaleTimeString('en-IN');
            }
          },
          localization: {
            timeFormatter: (time) => {
              const utcDate = new Date(time * 1000);
              const istDate = new Date(utcDate.getTime() + (5.5 * 3600 * 1000)); 
              return istDate.toLocaleDateString('en-IN') + ' ' + istDate.toLocaleTimeString('en-IN');
            }
          }
        });
        
        const candleSeries = chart.addCandlestickSeries();
        
        // Store the base 5-second candles (from historic endpoint and realtime updates)
        let fiveSecondCandles = [];
        // Default selected timeframe is 5s (raw 5-second candles)
        let currentTimeframe = "5s";
        
        // Helper function: converts a timeframe string to seconds.
        // Supports "s" (seconds), "h" (hours), "d" (days), "w" (weeks), "month" (months) and minutes (no suffix)
        function getTimeframeSeconds(tf) {
          if (typeof tf !== "string") {
            return Number(tf) * 60;
          }
          tf = tf.trim();
          if (tf.toLowerCase().endsWith("s")) {
            return Number(tf.slice(0, -1));
          } else if (tf.toLowerCase().endsWith("h")) {
            return Number(tf.slice(0, -1)) * 3600;
          } else if (tf.toLowerCase().endsWith("d")) {
            return Number(tf.slice(0, -1)) * 86400;
          } else if (tf.toLowerCase().endsWith("w")) {
            return Number(tf.slice(0, -1)) * 604800;
          } else if (tf.toLowerCase().endsWith("month")) {
            return Number(tf.slice(0, -5)) * 2592000;
          } else {
            // assume minutes
            return Number(tf) * 60;
          }
        }
        
        // Resample the 5-second candles into a higher timeframe based on the specified timeframe in seconds.
        function resampleCandles(candles, timeframeSeconds) {
          const resampled = [];
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
        
        // Update chart with either the raw 5-second candles or resampled data.
        function updateChart() {
          const tfSec = getTimeframeSeconds(currentTimeframe);
          let aggregated;
          if (tfSec === 5) {
            aggregated = fiveSecondCandles;
          } else {
            aggregated = resampleCandles(fiveSecondCandles, tfSec);
          }
          candleSeries.setData(aggregated);
        }
        
        // Setup WebSocket for real-time updated candle data
        const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws");
        ws.onmessage = function(event) {
          const data = JSON.parse(event.data);
          const candle = data.candle;
          // Update fiveSecondCandles: if the candle's time matches the last candle, update it; otherwise, append.
          if (fiveSecondCandles.length && fiveSecondCandles[fiveSecondCandles.length - 1].time === candle.time) {
            fiveSecondCandles[fiveSecondCandles.length - 1] = candle;
          } else {
            fiveSecondCandles.push(candle);
          }
          updateChart();
        };
        
        // On page load, fetch the historic 5-second candles.
        fetch('/historic')
          .then(response => response.json())
          .then(data => {
            fiveSecondCandles = data;
            updateChart();
          });
        
        // Add event listeners for timeframe buttons.
        document.querySelectorAll('.tf-button').forEach(button => {
          button.addEventListener('click', () => {
            // Update active state: remove "active" from all and add it on the clicked button.
            document.querySelectorAll('.tf-button').forEach(btn => btn.classList.remove('active'));
            button.classList.add('active');
        
            // Set the new timeframe from the data attribute.
            currentTimeframe = button.getAttribute('data-timeframe');
        
            // Update the chart based on the newly selected timeframe.
            updateChart();
          });
        });
      </script>
    </body>
    </html>
    """
    return render_template_string(html)

###################################################### Main Flow #######################################################
def main():
    """
    Starts the Fyers WebSocket client thread.
    Ticks are processed as they arrive to update the candle history.
    """
    global ws_thread
    ws_thread = threading.Thread(target=client_connect, daemon=True)
    ws_thread.start()
    logging.info("Fyers WebSocket thread started.")

def stop_main():
    """
    Stops the Fyers WebSocket client connection gracefully.
    """
    global ws_client
    logging.info("Stopping Fyers WebSocket connection gracefully...")
    if ws_client:
        try:
            ws_client.close_connection()
            logging.info("WebSocket connection closed.")
        except Exception as e:
            logging.error("Error closing WebSocket connection: %s", e)

# Start the WS client thread
main()

###################################################### Start Flask App #######################################################
port = int(os.getenv('PORT', 80))
print('Listening on port %s' % port)
app.run(debug=False, host="0.0.0.0", port=port)

# wscat -c ws://127.0.0.1/ws
```


http://127.0.0.1/historic





# updates

* the from and to to : current date -1 , from: such to & from diffence 200 cnalde in hourly tf 35 days working 5 days and not coming in holiday stack https://groww.in/p/nse-holidays
