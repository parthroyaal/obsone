
````html<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Document</title>
  <!-- Optional favicon -->
  <link rel="icon" href="favicon.ico" type="image/x-icon">
</head>
<body>
  <!-- Chart container -->
  <div id="chart_container"></div>

  <!-- TradingView Charting Library -->
  <script type="text/javascript" src="https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js"></script>

  <!-- Global Data storage -->
  <script>
    // Data arrays for full CSV split into 4 quadrants.
    const allBars = {
      leftTop: [],
      rightTop: [],
      leftBottom: [],
      rightBottom: []
    };
  </script>

  <!-- Utility function -->
  <script>
    function unixToMillis(unixTime) {
      return unixTime * 1000;
    }
  </script>

  <!-- CSV loading and preprocessing -->
  <script>
    async function readCSV(fileOrUrl, isUrl = false) {
      let text;
      try {
        if (isUrl) {
          // Fetch data from URL (static file in same folder)
          const response = await fetch(fileOrUrl);
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`);
          }
          text = await response.text();
        } else {
          // Read from a file object if needed
          text = await fileOrUrl.text();
        }

        const rows = text.split('\n');
        const headers = rows[0].split(',');

        // Get column indices from headers
        const timeIndex = headers.indexOf('time');
        const openIndex = headers.indexOf('open');
        const highIndex = headers.indexOf('high');
        const lowIndex = headers.indexOf('low');
        const closeIndex = headers.indexOf('close');
        const volumeIndex = headers.indexOf('volume');

        let bars = [];
        for (let i = 1; i < rows.length; i++) {
          const row = rows[i].split(',');
          if (row.length === headers.length) {
            bars.push({
              time: unixToMillis(parseFloat(row[timeIndex])),
              open: parseFloat(row[openIndex]),
              high: parseFloat(row[highIndex]),
              low: parseFloat(row[lowIndex]),
              close: parseFloat(row[closeIndex]),
              volume: parseInt(row[volumeIndex])
            });
          }
        }
        return bars;
      } catch (error) {
        console.error('Error reading CSV:', error);
        throw error;
      }
    }
  </script>

  <!-- Multiple timeframe data arrays -->
  <script>
    const dataArrays = {
      '5S': [],
      '10S': [],
      '15S': [],
      '30S': [],
      '1': [],
      '3': [],
      '5': [],
      '15': [],
      '30': [],
      '60': [],
      '120': [],
      '240': [],
      '1D': [],
      '1W': [],
      '1M': []
    };
    let historicalData = [];
  </script>

  <!-- Streaming and resampling logic -->
  <script>
    let lastUpdateTime = 0;
    let priceUpdateInterval;
    let isPaused = false;  // Global pause variable

    // Stub function to update prices (modify as needed)
    function updatePrices() {
      console.log("Updating prices...");
    }

    // Stub function to update price widget (modify as needed)
    function updatePriceWidget(newData) {
      console.log("Price widget updated:", newData);
    }

    // Stream data using an async iterator simulation
    function streamData(data, interval = 1000) {
      let index = 0;
      return {
        async next() {
          if (index < data.length) {
            await new Promise(resolve => setTimeout(resolve, interval));
            return { value: data[index++], done: false };
          } else {
            return { done: true };
          }
        },
        [Symbol.asyncIterator]() {
          return this;
        }
      };
    }

    // Convert a timeframe value (e.g., "5S", "15", "1D") into milliseconds
    function getTimeframeMs(timeframe) {
      const unit = timeframe.slice(-1);
      const value = parseInt(timeframe);
      switch (unit) {
        case 'S':
          return value * 1000;
        case 'D':
          return value * 24 * 60 * 60 * 1000;
        case 'W':
          return value * 7 * 24 * 60 * 60 * 1000;
        case 'M':
          return value * 30 * 24 * 60 * 60 * 1000;
        default:
          return value * 60 * 1000;
      }
    }

    // Resample data into bars for the given timeframe
    function resampleData(data, timeframe) {
      const resampledData = [];
      let currentBar = null;
      const timeframeMs = getTimeframeMs(timeframe);

      for (const row of data) {
        const barTime = Math.floor(row.time / timeframeMs) * timeframeMs;
        if (!currentBar || currentBar.time !== barTime) {
          if (currentBar) {
            resampledData.push(currentBar);
          }
          currentBar = {
            time: barTime,
            open: row.open,
            high: row.high,
            low: row.low,
            close: row.close,
            volume: row.volume
          };
        } else {
          currentBar.high = Math.max(currentBar.high, row.high);
          currentBar.low = Math.min(currentBar.low, row.low);
          currentBar.close = row.close;
          currentBar.volume += row.volume;
        }
      }
      if (currentBar) {
        resampledData.push(currentBar);
      }
      return resampledData;
    }

    // Start streaming the CSV data (simulate real-time data)
    async function startStreaming(allData) {
      const combinedData = [
        ...allData.leftTop,
        ...allData.rightTop,
        ...allData.leftBottom,
        ...allData.rightBottom
      ].sort((a, b) => a.time - b.time);

      const dataStream = streamData(combinedData);
      for await (const newData of dataStream) {
        // Pause handling if needed
        if (isPaused) {
          await new Promise(resolve => {
            const resumeListener = () => {
              if (!isPaused) {
                resolve();
                document.removeEventListener('keydown', resumeListener);
              }
            };
            document.addEventListener('keydown', resumeListener);
          });
        }

        console.log("New data received:", new Date(newData.time), newData);
        // Update the 5S array (for simplicity) and overall historicalData
        dataArrays['5S'].push(newData);
        historicalData.push(newData);

        // Update data for each timeframe from updated historicalData
        for (let timeframe in dataArrays) {
          dataArrays[timeframe] = resampleData(historicalData, timeframe);
        }

        notifySubscribers(newData);
      }
      console.log("Streaming completed");
    }
  </script>

  <!-- Datafeed and TradingView integration -->
  <script>
    const subscribers = new Map();

    function notifySubscribers(newData) {
      for (let [subscriberUID, handler] of subscribers) {
        const { resolution } = handler;
        if (resolution === '5S') {
          handler.callback(newData);
        } else {
          const resampledData = resampleData(historicalData, resolution);
          if (resampledData.length > 0) {
            handler.callback(resampledData[resampledData.length - 1]);
          }
        }
      }
      updatePriceWidget(newData);
    }

    const configurationData = {
      supported_resolutions: ['5S', '10S', '15S', '30S', '1', '3', '5', '15', '30', '60', '120', '240', '1D', '1W', '1M'],
      exchanges: [{
        value: 'Kraken',
        name: 'Kraken',
        desc: 'Kraken bitcoin exchange'
      }],
      symbols_types: [{
        name: 'crypto',
        value: 'crypto'
      }]
    };

    const Datafeed = {
      onReady: (callback) => {
        console.log('[onReady]: Method call');
        setTimeout(() => callback(configurationData));
      },

      searchSymbols: (userInput, exchange, symbolType, onResultReadyCallback) => {
        console.log('[searchSymbols]: Method call');
        onResultReadyCallback([]);
      },

      resolveSymbol: (symbolName, onSymbolResolvedCallback, onResolveErrorCallback) => {
        console.log('[resolveSymbol]: Method call', symbolName);
        const symbolInfo = {
          name: symbolName,
          description: symbolName,
          type: 'crypto',
          session: '24x7',
          minmov: 1,
          pricescale: 100,
          has_intraday: true,
          has_seconds: true,
          intraday_multipliers: ['1', '60'],
          seconds_multipliers: ["5", "10", "15", "30"],
          supported_resolutions: configurationData.supported_resolutions,
          volume_precision: 2,
          data_status: 'streaming',
          visible: true,
        };
        onSymbolResolvedCallback(symbolInfo);
      },

      getBars: (symbolInfo, resolution, periodParams, onHistoryCallback, onErrorCallback) => {
        const { from, to, firstDataRequest } = periodParams;
        console.log('[getBars]: Method call', symbolInfo, resolution, new Date(from * 1000), new Date(to * 1000));

        if (firstDataRequest) {
          // Utilize the pre-sampled historical data
          const bars = dataArrays[resolution];
          console.log(`[getBars]: returned entire ${resolution} array (${bars.length} bar(s))`, bars);
          onHistoryCallback(bars, { noData: bars.length === 0 });
        } else {
          const bars = dataArrays[resolution].filter(bar => {
            const barTime = bar.time / 1000;
            return barTime >= from && barTime < to;
          });
          console.log(`[getBars]: returned ${bars.length} bar(s)`, bars);
          onHistoryCallback(bars, { noData: bars.length === 0 });
        }
      },

      subscribeBars: (symbolInfo, resolution, onRealtimeCallback, subscriberUID, onResetCacheNeededCallback) => {
        console.log('[subscribeBars]: Method call with subscriberUID:', subscriberUID);
        subscribers.set(subscriberUID, { symbolInfo, resolution, callback: onRealtimeCallback });
      },

      unsubscribeBars: (subscriberUID) => {
        console.log('[unsubscribeBars]: Method call with subscriberUID:', subscriberUID);
        subscribers.delete(subscriberUID);
      },
    };
  </script>

  <!-- Chart construction and automatic CSV loading with initial data filtering -->
  <script>
    let currentInterval = '5S';

    document.addEventListener('DOMContentLoaded', async function() {
      try {
        // Load the static CSV file from the same folder.
        allBars.leftTop = await readCSV('NIFTYBANK.csv', true);
        if (allBars.leftTop.length > 0) {
          // Define the initial candle time.
          // All data before this time is used for the initial chart (historicalData),
          // and data at/after this time is used as streamingData.
          const initialCandleTimeStr = "2024-11-18 09:30:00";
          const initialTimeMs = new Date(initialCandleTimeStr).getTime();
          
          // Filter the CSV data into historicalData and streaming data.
          historicalData = allBars.leftTop.filter(bar => bar.time < initialTimeMs);
          allBars.leftTop = allBars.leftTop.filter(bar => bar.time >= initialTimeMs);
          
          // Pre-sample data across all timeframes using historicalData.
          for (let timeframe in dataArrays) {
            dataArrays[timeframe] = resampleData(historicalData, timeframe);
          }

          // Create the chart using the initial historical data.
          createChart(currentInterval);
          // Start streaming the data (streamingData will be added to historicalData).
          startStreaming(allBars);

          // Start periodic price updates.
          priceUpdateInterval = setInterval(updatePrices, 1000);
        } else {
          throw new Error('No data found in NIFTYBANK.csv');
        }
      } catch (error) {
        console.error('Error loading data:', error);
        alert('Error loading data: ' + error.message);
      }
    });

    function createChart(interval) {
      removeExistingChart();
      if (historicalData.length > 0) {
        // Use "NIFTYBANK" as the symbol name for display purposes.
        createChartInstance("chart_container", historicalData, "NIFTYBANK", interval);
      }
    }

    function createChartInstance(containerId, data, symbolName, interval) {
      const widget = new TradingView.widget({
        symbol: symbolName,
        interval: interval,
        fullscreen: false,
        timezone: "Asia/Kolkata",
        width: window.innerWidth,
        height: window.innerHeight,
        container: containerId,
        datafeed: Datafeed,
        enabled_features: ["dont_show_boolean_study_arguments", "seconds_resolution"],
        library_path: 'https://cdn.jsdelivr.net/gh/parth-royale/cdn@main/charting_library/charting_library.standalone.js',
        theme: 'Light',
        time_scale: { visible: true }
      });

      widget.onChartReady(() => {
        // Add MACD study
        widget.chart().createStudy('MACD', false, false, {
          fast_length: 12,
          slow_length: 26,
          signal_length: 9,
        });

        // Add multiple moving averages
        const studies = [
          { name: "Moving Average", inputs: { length: 12 }, id: "MA12", options: { "plot.linewidth": 3, "plot.color": "yellow" } },
          { name: "Moving Average", inputs: { length: 24 }, id: "MA24", options: { "plot.linewidth": 3, "plot.color": "red" } },
          { name: "Moving Average", inputs: { length: 50 }, id: "MA50", options: { "plot.linewidth": 3, "plot.color": "black" } },
          { name: "Moving Average", inputs: { length: 200 }, id: "MA200", options: { "plot.linewidth": 4, "plot.color": "blue" } }
        ];

        studies.forEach(study => {
          widget.chart().createStudy(study.name, false, false, study.inputs, study.options, { id: study.id });
        });
      });

      return widget;
    }

    function removeExistingChart() {
      const container = document.getElementById('chart_container');
      while (container.firstChild) {
        container.removeChild(container.firstChild);
      }
    }
  </script>
</body>

````
