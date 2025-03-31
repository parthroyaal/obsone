
````html
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Technical Indicators - Candlestick Chart</title>
  <style>
    /* Background and text colors */
    body { font-family: Arial, sans-serif; background-color: #1a1a1a; color: white; }
    /* Container and Panels */
    .container { padding: 10px; max-width: 100%; margin: 0 auto; }
    h1 { font-size: 24px; padding: 10px; }
    .panel { background-color: #2a2a2a; padding: 15px; border-radius: 5px; width: 100%; min-width: 250px; }
    #chart { width: 100%; height: 400px; background-color: #000; margin-bottom: 15px; }
    /* Responsive Design */
    @media (min-width: 768px) {
      .controls { flex-direction: row; flex-wrap: wrap; }
      .panel { flex: 1; min-width: 300px; }
      h1 { font-size: 28px; }
      #chart { height: 500px; }
    }
  </style>
</head>
<body>
  <h1>Technical Indicators Demo</h1>
  <div class="container">
    <div id="chart"></div>
    <!-- Panels Section -->
    <div class="panels">
      <div class="panel" id="sma-panel">
        <h2>SMA Indicators</h2>
        <p>
          Displays Simple Moving Averages for various periods (e.g., SMA12, SMA24, SMA50, SMA200).
        </p>
      </div>
      <div class="panel" id="rsi-panel">
        <h2>RSI Indicator</h2>
        <p>
          Relative Strength Index (RSI) helps determine overbought or oversold conditions.
        </p>
      </div>
      <div class="panel" id="macd-panel">
        <h2>MACD Indicator</h2>
        <p>
          The Moving Average Convergence Divergence (MACD) reveals changes in trend direction.
        </p>
      </div>
    </div>
  </div>


  <script src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/lightweight-charts.standalone.production.js"></script>


  
<!-- #################################################################
 


-->




</body>
</html>'''
````
