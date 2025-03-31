

```python
from flask import Flask, render_template_string
from flask_sock import Sock
import json
from datetime import datetime
from collections import deque
import pandas as pd
import threading
import os
import logging
import time
from pytz import timezone

###################################################### Flask Setup #######################################################
app = Flask(__name__)
sock = Sock(app)

###################################################### Global Data ########################################################
# Global deque to store tick data for processing


DEQUE_MAXLEN = 50
tick_data = deque(maxlen=DEQUE_MAXLEN)

# Global variables for managing the Fyers WebSocket thread and graceful shutdown.
ws_client = None  # Will hold the FyersDataSocket instance
ws_thread = None
# (We do not use a stop event here because we rely on the scheduler to call stop_main().)

###################################################### Logging Configuration ###############################################
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s',
    datefmt='%Y-%m-%d %H:%M:%S'
)
logging.info(f"Initial tick_data: {list(tick_data)}")


data_dir = '/var/lib/data'

# check if data_dir exists
if not os.path.exists(data_dir):
    print(f"Data directory {data_dir} does not exist.")
else:
    print(f"Data directory {data_dir} exists.")


###################################################### WebSocket Client Setup (Fyers) #######################################
def ws_client_connect():
    """
    Connects to Fyers WebSocket and populates the tick_data deque.
    This function starts the connection and then runs indefinitely until externally closed.
    """
    append_counter = 0  # Local counter

    import requests, time, base64, struct, hmac
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
    print(f'AcessTOKEN: {access_token}')

    client_id = "1BE74FZNXA-100"
    from fyers_apiv3.FyersWebsocket import data_ws

 

    def onmessage(message):
        """
        Callback function for handling incoming messages from Fyers WebSocket.
        """
        logging.info(f"Raw message received: {message}")
        
        if isinstance(message, str):
            try:
                tick = json.loads(message)
                logging.info(f"Parsed JSON message: {tick}")
            except json.JSONDecodeError as e:
                logging.error(f"Failed to parse JSON message: {e}")
                return
        else:
            tick = message

        # Check if this is a market data message (has 'ltp' and 'exch_feed_time')
        if "ltp" in tick and "exch_feed_time" in tick:
            price = tick["ltp"]
            # Convert exchange feed time to datetime
# tick_time = datetime.utcfromtimestamp(tick["exch_feed_time"]) 
# &KnowLibinOutt
# have to convert to int value not float miliscond 
# as  datetime.utcfromtimestamp(timestamp) correctly preserves the milliseconds / will preserve the fractional seconds (milliseconds).


            # tick_time = datetime.utcfromtimestamp(round(tick["exch_feed_time"]))
#If exch_feed_time = 1700000000.8, to make something shorter, especially by cutting off the top or end
# First approach → datetime.utcfromtimestamp(round(1700000001)) → 2023-11-14 20:13:21
# Second approach → datetime.utcfromtimestamp(1700000000).replace(microsecond=0) → 2023-11-14 20:13:20
            tick_time = datetime.utcfromtimestamp(tick["exch_feed_time"]).replace(microsecond=0)

            
            tick_data.append({'timestamp': tick_time, 'price': price})
# tick_data.append({'timestamp': tick_time.isoformat(), 'price': price})

            logging.info(f"Added tick data: price={price}, time={tick_time}")
            logging.info(f"Current deque size: {len(tick_data)}")
            
            if len(tick_data) == tick_data.maxlen:
                logging.info("Deque reached maximum capacity. Flushing to CSV...")
                ist = timezone('Asia/Kolkata')
                current_date = datetime.now(ist).strftime('%b%d').lower()
                csv_filename = f'{current_date}.csv'
                df = pd.DataFrame(list(tick_data))
                df.to_csv(os.path.join(data_dir, csv_filename), mode='a', header=False, index=False)
                tick_data.clear()
                logging.info("CSV file updated and deque cleared")
        elif "type" in tick and tick["type"] in ["cn", "ful", "sub"]:
            # These are connection/subscription messages, not market data
            logging.info(f"Received system message: {tick['type']} - {tick.get('message', '')}")
        else:
            logging.warning(f"Unexpected message format: {tick}")
            
    def onerror(message):
        print("Error:", message)

    def onclose(message):
        print("Connection closed:", message)

    def onopen():
        # Subscribe to data using the FyersDataSocket instance.
        # Here, we use the same instance (ws_client) to subscribe.
        data_type = "SymbolUpdate"
        symbols = ['NSE:NIFTY50-INDEX']
        ws_client.subscribe(symbols=symbols, data_type=data_type)
        ws_client.keep_running()

    global ws_client
    ws_client = data_ws.FyersDataSocket(
        access_token=access_token,  # Note: Use your valid token here.
        log_path="",
        litemode=False,
        write_to_file=False,
        reconnect=True,
        on_connect=onopen,
        on_close=onclose,
        on_error=onerror,
        on_message=onmessage
    )

    # Establish the connection (this call blocks until the connection is closed)
    ws_client.connect()

###################################################### Flask WebSocket Server #######################################################
@sock.route("/ws")
def push_latest_tick(ws):
    """Pushes only the latest tick data to WebSocket clients."""
    while True:
        if tick_data:
            latest_tick = tick_data[-1]
            ws.send(json.dumps(latest_tick, default=str))
        time.sleep(0.3)


#####
from flask import send_from_directory

@app.route("/download-csv")
def download_csv():
    """Provides a list of downloadable CSV files."""
    try:
        # List all CSV files in the data directory
        files = [f for f in os.listdir(data_dir) if f.endswith('.csv')]
        html = "<h1>Download CSV Files</h1>"
        html += "<ul>"
        for file in files:
            html += f'<li><a href="/download-file/{file}">{file}</a></li>'
        html += "</ul>"
        return html
    except Exception as e:
        logging.error(f"Error listing CSV files: {e}")
        return "Error in listing files."

@app.route("/download-file/<filename>")
def download_file(filename):
    """Sends a file from the data directory to the client."""
    try:
        return send_from_directory(data_dir, filename, as_attachment=True)
    except Exception as e:
        logging.error(f"Error sending file {filename}: {e}")
        return "Error in sending file."
###################################################### Frontend Setup #######################################################

@app.route("/")
def index():
    """Serves the frontend chart."""
    html = """
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Live BTC/USDT Tick Data</title>
        <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
    </head>
    <body>
        <h1>Live NIFTY Tick Chart (IST)</h1>
        <div id="chart"></div>
<script>
   const chart = LightweightCharts.createChart(document.getElementById('chart'), {
    width: window.innerWidth,
    height: window.innerHeight,
    priceScale: { borderColor: '#cccccc' },
    timeScale: { 
        borderColor: '#cccccc', 
        timeVisible: true, 
        secondsVisible: true,
        tickMarkFormatter: (time) => {
            const utcDate = new Date(time * 1000); // Convert UNIX time to Date object (UTC)
            const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); // Convert to IST
            return istDate.toLocaleDateString('en-IN') + ' ' + istDate.toLocaleTimeString('en-IN');
        }
    },
    localization: {
        timeFormatter: (time) => {
            const utcDate = new Date(time * 1000);
            const istDate = new Date(utcDate.getTime() + (5.5 * 60 * 60 * 1000)); 
            return istDate.toLocaleDateString('en-IN') + ' ' + istDate.toLocaleTimeString('en-IN');
        }
    }
});

   const candleSeries = chart.addCandlestickSeries();
const ws = new WebSocket((location.protocol === "https:" ? "wss://" : "ws://") + location.host + "/ws");
let lastCandle = null;

ws.onmessage = function(event) {
    const tick = JSON.parse(event.data);
    const tickTime = Math.floor(new Date(tick.timestamp).getTime() / 1000);
    const candleTime = tickTime - (tickTime % 5); // Round to nearest 5-second interval
    
    if (!lastCandle || candleTime > lastCandle.time) {
        // Create a new 5-second candle
        lastCandle = {
            time: candleTime,
            open: tick.price,
            high: tick.price,
            low: tick.price,
            close: tick.price
        };
        candleSeries.update(lastCandle);
    } else {
        // Update the current 5-second candle
        lastCandle.high = Math.max(lastCandle.high, tick.price);
        lastCandle.low = Math.min(lastCandle.low, tick.price);
        lastCandle.close = tick.price;
        candleSeries.update(lastCandle);
    }
};
</script>
    </body>
    </html>
    """
    return render_template_string(html)




###############################################################



# # https://chatgpt.com/c/67d911f0-a050-800a-9d51-3e868505e667
# @app.route("/historical-data")
# def get_historical_data():
#     """Returns raw tick data from CSV as JSON."""
#     try:
#         csv_path = os.path.join(data_dir, csv_filename)
#         if not os.path.exists(csv_path):
#             logging.warning(f"Historical data file not found: {csv_filename}")
#             return json.dumps({"error": "No historical data available yet", "data": []})
#         else:
#             logging.info(f"Reading historical data: {csv_filename}")

#         # Read CSV file with two columns: timestamp and price
#         df = pd.read_csv(csv_path, names=['timestamp', 'price'])

#         # Define a helper function to parse timestamps robustly.
#         def parse_timestamp(ts):
#             ts_str = str(ts).strip()
#             # If the string is all digits, assume it's a Unix timestamp in seconds.
#             if ts_str.isdigit():
#                 try:
#                     return pd.to_datetime(int(ts_str), unit='s', utc=True)
#                 except Exception as e:
#                     logging.error(f"Failed to parse numeric timestamp {ts_str}: {e}")
#                     return pd.NaT
#             else:
#                 # Otherwise, assume it's an ISO formatted datetime string.
#                 try:
#                     return pd.to_datetime(ts_str, utc=True)
#                 except Exception as e:
#                     logging.error(f"Failed to parse datetime string {ts_str}: {e}")
#                     return pd.NaT

#         # Apply the robust parser to the timestamp column.
#         df['timestamp'] = df['timestamp'].apply(parse_timestamp)

#         # Drop rows that couldn't be parsed.
#         df.dropna(subset=['timestamp'], inplace=True)

#         # Convert the parsed timestamps to Unix timestamps in seconds.
#         # (Using int64 conversion requires that all timestamps are valid)
#         df['timestamp'] = df['timestamp'].astype('int64') // 10**9

#         # Convert DataFrame to list of dictionaries.
#         ticks = df.to_dict('records')
#         return json.dumps({"data": ticks})
    
#     except Exception as e:
#         logging.error(f"Error processing historical data: {e}")
#         return json.dumps({"error": str(e), "data": []})

ist = timezone('Asia/Kolkata')
current_date = datetime.now(ist).strftime('%b%d').lower()
# csv_filename = f'{current_date}.csv'
csv_filename = 'mar27.csv'


@app.route("/historical-data")
def get_historical_data():
    """Returns raw tick data from CSV as JSON."""
    try:
        # Read CSV file
        csv_path = os.path.join(data_dir, csv_filename)
        
        # Check if file exists
        if not os.path.exists(csv_path):
            logging.warning(f"Historical data file not found:): {csv_filename}")
            return json.dumps({"error": "No historical data available yet", "data": []})
        else:
            logging.info(f"Reading historical data: {csv_filename}")
        # Read CSV file
   
        

        df = pd.read_csv(csv_path, names=['timestamp', 'price'])
        

        # # Convert timestamp to datetime and then to Unix timestamp (seconds)
        df['timestamp'] = pd.to_datetime(df['timestamp'], utc=True)

        df['timestamp'] = df['timestamp'].astype(int) // 10**9  # Convert to seconds

        # Convert to list of dictionaries
        ticks = df.to_dict('records')
        
        return json.dumps({"data": ticks})
    
    except Exception as e:
        logging.error(f"Error processing historical data: {e}")
        return json.dumps({"error": str(e), "data": []})
    


@app.route("/history")
def historical_chart():
    """Serves the historical chart from CSV data with timeframe selection."""
    html = """
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <title>Historical NIFTY Data</title>
  <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>
  <style>
    .controls { margin: 10px; }
    button { margin: 5px; padding: 5px 10px; }
    .timeframe-group { 
      margin: 5px 0;
      padding: 5px;
      border: 1px solid #ccc;
      border-radius: 4px;
    }
    .group-label {
      font-weight: bold;
      margin-right: 10px;
    }
  </style>
</head>
<body>
  <h1>Historical NIFTY Chart (1m)</h1>
  <div class="controls">
    <div class="timeframe-group">
      <span class="group-label">Seconds:</span>
      <button onclick="changeTimeframe(5/60)">5s</button>
      <button onclick="changeTimeframe(10/60)">10s</button>
      <button onclick="changeTimeframe(15/60)">15s</button>
      <button onclick="changeTimeframe(30/60)">30s</button>
      <button onclick="changeTimeframe(45/60)">45s</button>
    </div>
    <div class="timeframe-group">
      <span class="group-label">Minutes:</span>
      <button onclick="changeTimeframe(1)">1m</button>
      <button onclick="changeTimeframe(3)">3m</button>
      <button onclick="changeTimeframe(5)">5m</button>
      <button onclick="changeTimeframe(15)">15m</button>
      <button onclick="changeTimeframe(30)">30m</button>
      <button onclick="changeTimeframe(60)">1h</button>
    </div>
  </div>
  <div id="chart"></div>
  <script>
    let rawData = [];
    let currentTimeframe = 1; // Default: 1 minute

    // Create the chart with time scale formatting that assumes time is in seconds.
    const chart = LightweightCharts.createChart(document.getElementById('chart'), {
      width: window.innerWidth,
      height: window.innerHeight,
      priceScale: { borderColor: '#cccccc' },
      timeScale: { 
        borderColor: '#cccccc', 
        timeVisible: true, 
        secondsVisible: true,
        tickMarkFormatter: (time) => {
          // Multiply by 1000 because 'time' is in seconds.
          const date = new Date(time * 1000);
          return date.toLocaleDateString('en-IN') + ' ' + date.toLocaleTimeString('en-IN');
        }
      },
      localization: {
        timeFormatter: (time) => {
          const date = new Date(time * 1000);
          return date.toLocaleDateString('en-IN') + ' ' + date.toLocaleTimeString('en-IN');
        }
      }
    });

    const candleSeries = chart.addCandlestickSeries();

    // Process raw tick data into candles.
    function processTicksToCandles(ticks, minutesPerCandle) {
      const candles = new Map();
      // Calculate the grouping interval in seconds.
      const secondsPerCandle = minutesPerCandle * 60;
      
      ticks.forEach(tick => {
        // Group ticks by flooring the Unix timestamp (in seconds)
        const candleTime = Math.floor(tick.timestamp / secondsPerCandle) * secondsPerCandle;
        if (!candles.has(candleTime)) {
          candles.set(candleTime, {
            time: candleTime,  // time must be a Unix timestamp in seconds
            open: tick.price,
            high: tick.price,
            low: tick.price,
            close: tick.price
          });
        } else {
          const candle = candles.get(candleTime);
          candle.high = Math.max(candle.high, tick.price);
          candle.low = Math.min(candle.low, tick.price);
          // Update close price to the latest tick price.
          candle.close = tick.price;
        }
      });
      
      // Return candles sorted in ascending order by time.
      return Array.from(candles.values()).sort((a, b) => a.time - b.time);
    }

    // Update the chart data based on the selected timeframe.
    function changeTimeframe(minutes) {
      currentTimeframe = minutes;
      const candles = processTicksToCandles(rawData, minutes);
      candleSeries.setData(candles);
      
      // Update the chart title with the current timeframe.
      const timeframeText = minutes < 1 ? `${Math.round(minutes * 60)}s` : `${minutes}m`;
      document.querySelector('h1').textContent = `Historical NIFTY Chart (${timeframeText})`;
    }

    // Fetch historical tick data from your server.
    fetch('/historical-data')
      .then(response => response.json())
      .then(jsonData => {
        rawData = jsonData.data;  // Expecting an array of objects with 'timestamp' (in seconds) and 'price'
        changeTimeframe(currentTimeframe);
      })
      .catch(error => {
        console.error('Error fetching historical data:', error);
      });
  </script>
</body>
</html>


    """
    return render_template_string(html)




###################################################### Main Flow #######################################################
def main():
    """Starts the WebSocket client thread."""
    # create_table_if_not_exists()
    # Start ws_client_connect in a separate thread.
    global ws_thread
    ws_thread = threading.Thread(target=ws_client_connect, daemon=True)
    ws_thread.start()
    logging.info("Fyers WebSocket thread started.")

def stop_main():
    """Stops the WebSocket client connection gracefully."""
    global ws_client
    logging.info("Stopping Fyers WebSocket connection gracefully...")
    if ws_client:
        try:
            ws_client.close_connection()
            logging.info("WebSocket connection closed.")
        except Exception as e:
            logging.error("Error closing WebSocket connection: %s", e)


# main()

###################################################### Scheduler Setup #######################################################
from apscheduler.schedulers.background import BackgroundScheduler
scheduler = BackgroundScheduler(daemon=True)

# Schedule main() to start the WebSocket client (adjust the time as needed)
scheduler.add_job(
    main,
    'cron',
    day_of_week='mon-fri',
    hour=9,
    minute=14,
    timezone='Asia/Kolkata'
)

# Schedule stop_main() to close the connection at a specified time
scheduler.add_job(
    stop_main,
    'cron',
    day_of_week='mon-fri',
    hour=15,
    minute=31,
    timezone='Asia/Kolkata',
    id='stop_main'
)

scheduler.start()

###################################################### Start Flask App #######################################################
port = int(os.getenv('PORT', 80))
print('Listening on port %s' % (port))
app.run(debug=False, host="0.0.0.0", port=port)
```