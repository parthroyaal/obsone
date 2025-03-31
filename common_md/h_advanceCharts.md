# Fin

[h_fyersReplay](h_fyersReplay.md)

[h_specchat](h_specchat.md)

# Chart init sequence

## 1.  add div for  chart cointainer value  and lib src

````html
<div id="chart"> </div>
<script type="text/javascript" src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js"></script>

````

## 2. have global window obj for global scope  rather using varible for datafeed object  == hooks dict/obj or also called function signatures dict  /obj

````html

<!DOCTYPE html>
<html lang="en" dir="ltr">
  <head>
    <meta charset="utf-8">
    <title>mar13.1</title>
  </head>
  <body>
    <script type="text/javascript" src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js"></script>

    <div id="tv_chart_container"></div>

    <!-- 1. Define Datafeed FIRST -->
    <script>
      // Properly attach to window object
      window.Datafeed = {
        onReady: (callback) => {
          console.log('[onReady]: Method call');
        },
        searchSymbols: (userInput, exchange, symbolType, onResultReadyCallback) => {
          console.log('[searchSymbols]: Method call');
        },
        resolveSymbol: (symbolName, onSymbolResolvedCallback, onResolveErrorCallback, extension) => {
          console.log('[resolveSymbol]: Method call', symbolName);
        },
        getBars: (symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) => {
          console.log('[getBars]: Method call', symbolInfo);
        },
        subscribeBars: (symbolInfo, resolution, onRealtimeCallback, subscriberUID, onResetCacheNeededCallback) => {
          console.log('[subscribeBars]: Method call with subscriberUID:', subscriberUID);
        },
        unsubscribeBars: (subscriberUID) => {
          console.log('[unsubscribeBars]: Method call with subscriberUID:', subscriberUID);
        },
      };
    </script>

    <!-- 2. Then create widget -->
    <script>
      new TradingView.widget({
        container_id: "tv_chart_container",
        datafeed: window.Datafeed, // Now available
        library_path: 'https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js',
        symbol: "BTC/USD",
        interval: "5S",
        timezone: "Asia/Kolkata",
        theme: "dark",
        locale: "en",
        fullscreen: "true",   
        enabled_features: ["dont_show_boolean_study_arguments", "seconds_resolution"],
      });
    </script>

    <h1>mar13.1</h1>
  </body>
</html>
````

### For seconds to work in resolve symbol hook it should have options for

````html
<script>
resolveSymbol: function(symbolName, onSymbolResolvedCallback, onResolveErrorCallback) {
  setTimeout(function() {
    onSymbolResolvedCallback({
      name: symbolName,
      ticker: symbolName,
      session: "24x7",  // Must be a non-empty string.
      timezone: "Etc/UTC",
      minmov: 1,
      pricescale: 100,
      has_intraday: true,
      has_seconds: true, 
    });
  }, 0);
},
</script>
````

### For GetBars to work with data available not the range requested

## 3.  # Define the widget

````html
    <script>
      new TradingView.widget({
        container_id: "tv_chart_container",
        datafeed: window.Datafeed,
        library_path: 'https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js',
        symbol: "BTC/USD",
        interval: "5S",
        timezone: "Asia/Kolkata",
        theme: "dark",
        locale: "en",
        enabled_features: ["seconds_resolution"],
        disabled_features: ["use_localstorage_for_settings"]
      });
    </script>
````

## 3.1 Datafeed imp1

````html
<script>

      window.Datafeed = {
        onReady: (callback) => {
          console.log('[onReady]: Method call');
          callback(); // ← MUST CALL THIS TO CONTINUE INITIALIZATION
        },

        resolveSymbol: (symbolName, onSymbolResolvedCallback, onResolveErrorCallback) => {
          console.log('[resolveSymbol]: Method call', symbolName);
          onSymbolResolvedCallback({
            name: symbolName,
            ticker: symbolName,
            description: "BTC/USD",
            type: "crypto",
            session: "24x7",
            timezone: "Asia/Kolkata",
            minmov: 1,
            pricescale: 100,
            has_intraday: true,
            has_seconds: true,
            supported_resolutions: ['5S', '15S', '1D'],
            volume_precision: 8,
            data_status: "streaming"
          });
        },

<script>
  window.Datafeed = { 
  getBars: function(symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) {
    const bars = [];
    
    // Convert all in-memory bars to TV format (time in seconds)
    inMemoryBars.forEach(bar => {
      const barTimeSeconds = Math.floor(isoToMs(bar.time) / 1000);
      bars.push({
        time: barTimeSeconds, // MUST be UNIX seconds
        open: bar.open,
        high: bar.high,
        low: bar.low,
        close: bar.close,
        volume: bar.volume
      });
    });

    // Sort bars chronologically (oldest first)
    bars.sort((a, b) => a.time - b.time);

    // Always return ALL bars regardless of requested range
    if (bars.length > 0) {
      console.log(`Returning ${bars.length} bars`);
      onHistoryCallback(bars, {
        noData: false,
        // Add these metadata fields to prevent infinite requests
        nextTime: null, // No more historical data
        latestTime: bars[bars.length-1].time
      });
    } else {
      onHistoryCallback([], {noData: true});
    }
  },

  subscribeBars: function(symbolInfo, resolution, onRealtimeCallback, subscriberUID) {
    this._realtimeSubscriber = this._realtimeSubscriber || {};
    const barDurationMs = 5000; // 5-second interval
    
    this._realtimeSubscriber[subscriberUID] = setInterval(() => {
      const now = Date.now();
      const currentBarTime = Math.floor(now / 1000); // Current time in seconds
      
      // Get or create new bar
      let lastBar = inMemoryBars[inMemoryBars.length-1];
      const lastBarTime = isoToMs(lastBar?.time) || 0;

      if (!lastBar || (now - lastBarTime) >= barDurationMs) {
        // Create new bar
        lastBar = {
          time: new Date(now).toISOString(),
          open: lastBar?.close || 100,
          high: lastBar?.close || 100,
          low: lastBar?.close || 100,
          close: (lastBar?.close || 100) + (Math.random() * 2 - 1),
          volume: Math.floor(Math.random() * 1000) + 1000
        };
        inMemoryBars.push(lastBar);
      } else {
        // Update existing bar
        lastBar.high = Math.max(lastBar.high, lastBar.close);
        lastBar.low = Math.min(lastBar.low, lastBar.close);
        lastBar.volume += Math.floor(Math.random() * 100);
      }

      // Send update in SECONDS
      onRealtimeCallback({
        time: Math.floor(isoToMs(lastBar.time) / 1000),
        open: lastBar.open,
        high: lastBar.high,
        low: lastBar.low,
        close: lastBar.close,
        volume: lastBar.volume
      });
    }, 5000);
  },


    unsubscribeBars: (subscriberUID) => {
        console.log('[unsubscribeBars]: Method call');
        clearInterval(this.intervalId);
    }
    };
    </script>

````

## 3.2 Datafeed imp 2

````html
    <script>
      window.Datafeed = {
        onReady: (callback) => {
          console.log('[onReady]: Method call');
          callback(); // ← MUST CALL THIS TO CONTINUE INITIALIZATION
        },

        resolveSymbol: (symbolName, onSymbolResolvedCallback, onResolveErrorCallback) => {
          console.log('[resolveSymbol]: Method call', symbolName);
          onSymbolResolvedCallback({
            name: symbolName,
            ticker: symbolName,
            description: "BTC/USD",
            type: "crypto",
            session: "24x7",
            timezone: "Asia/Kolkata",
            minmov: 1,
            pricescale: 100,
            has_intraday: true,
            has_seconds: true,
            supported_resolutions: ['5S', '15S', '1D'],
            volume_precision: 8,
            data_status: "streaming"
          });
        },

        getBars: (symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) => {
          console.log('[getBars]: Method call', periodParams);
          const bars = [
            {
              time: Date.parse('2023-03-01T00:00:00')/1000, // ← MUST USE UNIX TIMESTAMP
              open: 100,
              high: 105,
              low: 95,
              close: 100,
              volume: 1000
            },
            {
              time: Date.parse('2023-03-01T00:00:05')/1000,
              open: 105,
              high: 110,
              low: 100,
              close: 105,
              volume: 1500
            },
            {
              time: Date.parse('2023-03-01T00:00:10')/1000,
              open: 110,
              high: 115,
              low: 105,
              close: 112,
              volume: 2000
            });
          }, 5000);
        },

        unsubscribeBars: (subscriberUID) => {
          console.log('[unsubscribeBars]: Method call');
          clearInterval(this.intervalId);
        }
      };
    </script>
````

## Misc

````html
    <script>
        const Datafeed = {
            getBars: function(symbol, resolution, from, to) {
                const url = "/data?symbol=" + symbol + "&resolution=" + resolution + "&from=" + from + "&to=" + to;
                return fetch(url).then(function(response) {
                    return response.json();
                });
            }
        };
    </script>

<script>
           getBars: (symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) => {
          console.log('[getBars]: Method call', symbolInfo);
          const bars = [
            {
              time: '2023-03-01T00:00:00.000Z',
              open: 100,
              high: 105,
              low: 95,
              close: 100
            },
            {
              time: '2023-03-01T00:00:05.000Z',
              open: 105,
              high: 110,
              low: 100,
              close: 105
            },
            {
              time: '2023-03-01T00:00:10.000Z',
              open: 110,
              high: 115,
              low: 105,
              close: 110
            }
          ];
          onHistoryCallback(bars.map(function(bar => (
            {
              time: new Date(bar.time).toISOString(),
              ...bar
            }
          )));
        },
        </script>
````

## impMain

````html
<!DOCTYPE html>
<html>
<head>
    <title>TradingView Chart with 5S Resolution and Real-Time Updates</title>
</head>
<body>
    <div id="tv_chart_container"></div>

    <!-- Include the charting library -->
    <!-- <script type="text/javascript" src="/charting_library/charting_library.standalone.js"></script> -->

    <script type="text/javascript" src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js"></script>
    <!-- Utility Functions and Dynamic In-Memory Data -->
    <script>
      // Convert an ISO date string to a timestamp (milliseconds)
      function isoToMs(isoString) {
          return new Date(isoString).getTime();
      }

      // Generate initial in-memory bar data for the last 10 minutes (one bar every 5 seconds)
      var inMemoryBars = [];
      var now = Date.now();
      var startTime = now - (10 * 60 * 1000);  // 10 minutes ago
      for (var time = startTime; time < now; time += 5000) {
          inMemoryBars.push({
              time: new Date(time).toISOString(),
              open: 100,
              high: 105,
              low: 95,
              close: 100 + Math.random() * 10,
              volume: Math.floor(Math.random() * 1000) + 1000
          });
      }
    </script>
    
    <!-- Datafeed Implementation -->
    <script>
      window.Datafeed = {
        onReady: function(callback) {
          setTimeout(function() {
            callback({
              // Include "5S" for 5‑second resolution support.
              supported_resolutions: ["5S", "1", "5", "15", "30", "60", "D"]
            });
          }, 0);
        },
        resolveSymbol: function(symbolName, onSymbolResolvedCallback, onResolveErrorCallback) {
          setTimeout(function() {
            onSymbolResolvedCallback({
              name: symbolName,
              ticker: symbolName,
              session: "24x7",  // Must be a non-empty string.
              timezone: "Etc/UTC",
              minmov: 1,
              pricescale: 100,
              has_intraday: true,
              has_seconds: true,  // Indicathttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryhttps://cdn.jsdelivr.net/gh/parth-royale/cdn@tree/main/charting_libraryes that seconds-level data is supported.
              supported_resolutions: ["5S", "1", "5", "15", "30", "60", "D"]
            });
          }, 0);
        },
        getBars: function(symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) {
          var bars = [];
          // Convert "from" and "to" parameters (in seconds) to milliseconds.
          var periodFromMs = periodParams.from * 1000;
          var periodToMs = periodParams.to * 1000;
          
          inMemoryBars.forEach(function(bar) {
              var barTime = isoToMs(bar.time);
              if (barTime >= periodFromMs && barTime <= periodToMs) {
                  bars.push({
                      time: barTime,
                      open: bar.open,
                      high: bar.high,
                      low: bar.low,
                      close: bar.close,
                      volume: bar.volume
                  });
              }
          });
          
          if (bars.length) {
              onHistoryCallback(bars, { noData: false });
          } else {
              onHistoryCallback([], { noData: true });
          }
        },
        subscribeBars: function(symbolInfo, resolution, onRealtimeCallback, subscriberUID, onResetCacheNeededCallback) {
          // Update the chart every second, even though each bar represents 5 seconds.
          // When a new 5-second period starts, a new bar is pushed; otherwise, the current bar is updated.
          var intervalMs = 1000;  // update every 1 second
          var barDurationMs = 5000; // 5-second bar interval

          // Save the timer ID using the subscriberUID for later unsubscription.
          this._realtimeSubscriber = this._realtimeSubscriber || {};
          this._realtimeSubscriber[subscriberUID] = setInterval(function() {
              var now = Date.now();
              // Calculate the start of the current 5-second period.
              var currentBarStartTime = Math.floor(now / barDurationMs) * barDurationMs;
              
              // Get the latest bar.
              var lastBar = inMemoryBars[inMemoryBars.length - 1];
              var lastBarTime = isoToMs(lastBar.time);
              
              if (currentBarStartTime > lastBarTime) {
                  // A new bar period has started.
                  var newBar = {
                      time: new Date(currentBarStartTime).toISOString(),
                      open: lastBar.close,
                      high: lastBar.close,
                      low: lastBar.close,
                      close: lastBar.close + (Math.random() * 2 - 1), // some slight random variation
                      volume: Math.floor(Math.random() * 100) + 100
                  };
                  inMemoryBars.push(newBar);
                  onRealtimeCallback({
                      time: currentBarStartTime,
                      open: newBar.open,
                      high: newBar.high,
                      low: newBar.low,
                      close: newBar.close,
                      volume: newBar.volume
                  });
              } else {
                  // Update the existing bar.
                  var newPrice = lastBar.close + (Math.random() * 2 - 1);
                  lastBar.high = Math.max(lastBar.high, newPrice);
                  lastBar.low = Math.min(lastBar.low, newPrice);
                  lastBar.close = newPrice;
                  lastBar.volume += Math.floor(Math.random() * 10);
                  onRealtimeCallback({
                      time: currentBarStartTime,
                      open: lastBar.open,
                      high: lastBar.high,
                      low: lastBar.low,
                      close: lastBar.close,
                      volume: lastBar.volume
                  });
              }
          }, intervalMs);
        },
        unsubscribeBars: function(subscriberUID) {
          if (this._realtimeSubscriber && this._realtimeSubscriber[subscriberUID]) {
              clearInterval(this._realtimeSubscriber[subscriberUID]);
              delete this._realtimeSubscriber[subscriberUID];
          }
        }
      };
    </script>
    
    <!-- TradingView Widget Initialization -->
    <script>
      new TradingView.widget({
        // Use "container" (not the deprecated "container_id")
        container: "tv_chart_container",
        datafeed: window.Datafeed,
        library_path: 'https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js',
        symbol: "BTC/USD",
        interval: "5S",  // Default to 5-second intervals.
        timezone: "Asia/Kolkata",
        theme: "dark",
        locale: "en",
        fullscreen: "true",   
        enabled_features: ["dont_show_boolean_study_arguments", "seconds_resolution"],     
        // Note: Studies have been omitted; built-in studies might not support sub‑minute data.
      });
    </script>
</body>
</html>
````
