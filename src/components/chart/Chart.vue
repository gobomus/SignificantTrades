<template>
	<div id="chart">
		<div class="chart__container" ref="chartContainer" v-bind:class="{fetching: fetching}" v-bind:style="{ height: chartHeight }">
      <div class="chart__scale-handler" ref="chartScaleHandler" v-on:mousedown="startScale" v-on:dblclick.stop.prevent="resetScale"></div>
      <div class="chart__height-handler" ref="chartHeightHandler" v-on:mousedown="startResize" v-on:dblclick.stop.prevent="resetHeight"></div>

      <div class="chart__canvas"></div>
		</div>
	</div>
</template>

<script>
import { mapState } from 'vuex';

import socket from "../../services/socket";
import chartOptions from "./options.json";

import Highcharts from 'highcharts/highstock';
import Indicators from 'highcharts/indicators/indicators';
import EMA from 'highcharts/indicators/ema';
import CMF from 'highcharts/indicators/cmf';

Indicators(Highcharts);
EMA(Highcharts);
CMF(Highcharts);

export default {
  data() {
    return {
      fetching: false,
			chart: null,
      tick: null,
      cursor: null,
      queuedTrades: [],
      queuedTicks: [],

      _timeframe: null,
    };
  },
  computed: {
    ...mapState([
      'timeframe',
      'dark',
      'actives',
      'exchanges',
      'isSnaped',
      'chartHeight',
      'chartRange',
      'chartLiquidations',
      'chartCandlestick',
      'chartPadding'
		])
  },
  created() {
    Highcharts.wrap(Highcharts.Chart.prototype, 'pan', this.doPan(this));

    socket.$on('trades.queued', this.onTrades);

    socket.$on('clean', this.onClean);

    this.onStoreMutation = this.$store.subscribe((mutation, state) => {
      switch (mutation.type) {
        case 'setTimeframe':
          this.setTimeframe(mutation.payload, true, this._timeframe !== this.timeframe);
          this._timeframe = parseInt(this.timeframe);
          break;
        case 'toggleSnap':
          mutation.payload && this.snapRight(true);
          break;
        case 'setExchangeMatch':
          this.updateChartHeight();
          break;
        case 'reloadExchangeState':
        case 'toggleLiquidations':
        case 'setChartPadding':
          this.setTimeframe(this.timeframe);
          break;
        case 'toggleDark':
        case 'toggleCandlestick':
          this.createChart();
          this.setTimeframe(this.timeframe);
        break;
        case 'toggleVolumeAverage':
          this.chart.series[5].update({visible: mutation.payload});
          this.chart.series[6].update({visible: mutation.payload});
        break;
        case 'setVolumeAverageLength':
          this.chart.series[5].update({params: {period: mutation.payload || 14}});
          this.chart.series[6].update({params: {period: mutation.payload || 14}});
        break;
      }
    });

    this._timeframe = parseInt(this.timeframe);
  },
	mounted() {
    console.log(`[chart.mounted]`);

    this.$refs.chartContainer.style.height = this.getChartSize().height;

    window.setTimeout(() => this.createChart());

    this._doResize = this.doResize.bind(this);

    window.addEventListener('resize', this._doResize, false);

    this._doDrag = this.doDrag.bind(this);

    window.addEventListener('mousemove', this._doDrag, false);

    this._stopDrag = this.stopDrag.bind(this);

    window.addEventListener('mouseup', this._stopDrag, false);

    // this.$el.addEventListener('wheel', this.doZoom);

    // this.buildExchangesSeries();

    this.setTimeframe(this.timeframe, true);
	},
  beforeDestroy() {
    this.destroyChart();

    socket.$off('trades.queued', this.onTrades);

    socket.$off('clean', this.onClean);

    window.removeEventListener('resize', this._doResize);
    window.removeEventListener('mousemove', this._doDrag);
    window.removeEventListener('mouseup', this._stopDrag);

    // this.$el.removeEventListener('wheel', this.doZoom);

    this.onStoreMutation();
  },
  methods: {
    createChart() {
      this.destroyChart();

      const options = this.getChartOptions();

      this.chart = Highcharts.stockChart(this.$el.querySelector('.chart__canvas'), options);

      this.updateChartHeight();

      if (this.chartRange) {
        this.setRange(this.chartRange);
      }

      if (this.queuedTicks.length) {
        // console.log(`[chart.createChart] tick queued historicals`, this.queuedTicks.length);

        this.tickHistoricals(this.queuedTicks.splice(0, this.queuedTicks.length));

        this.chart.redraw();
      }

      if (this.queuedTrades.length) {
        // console.log(`[chart.createChart] tick queued trades`, this.queuedTrades.length);

        this.tickTrades(this.queuedTrades.splice(0, this.queuedTrades.length));

        this.chart.redraw();
      }
    },
    destroyChart() {
      if (!this.chart) {
        return;
      }

      this.chart.destroy();
      delete this.chart;
    },
    setTimeframe(timeframe, snap = false, clear = false) {
      console.log(`[chart.setTimeframe]`, timeframe);

      const count = ((this.chart ? this.chart.chartWidth : this.$refs.chartContainer.offsetWidth) * (1 - this.chartPadding) - 20 * .1) / 10;
      const range = timeframe * 2 * count;

      const now = +new Date();

      socket.fetchRangeIfNeeded(range, clear).then(response => {
        if (response) {
          console.log(`[chart.setTimeframe] done fetching (${response.results.length} new ${response.format}(s))`)
        } else {
          console.log(`[chart.setTimeframe] did not fetch anything new`);
        }

        this.setRange(range, snap);

        if (this.chart) {
          console.log(`[chart.setTimeframe] prepare and clean chart for printing`);

          for (let serie of this.chart.series) {
            serie.setData([], false);
          }

          this.tickData = null;

          this.chart.redraw();
        }

        if (!socket.ticks.length && !socket.trades.length) {
          // console.log(`[chart.setTimeframe] nothing to print`);
          return;
        }

        this.cursor = Math.floor(
          (Math.min(socket.ticks[0] ? socket.ticks[0].timestamp : Infinity, socket.trades[0] ? socket.trades[0][1] : Infinity)  / this.timeframe)
        ) * this.timeframe;

        // console.log(`[chart.setTimeframe] Send data to printers\n\tticks:`, socket.ticks, `\n\ttrades:`, socket.trades);

        this.tickHistoricals(socket.ticks);

        this.tickTrades(socket.trades);

        this.setRange(range / 2, snap);
      })
    },
    onTrades(trades) {
      if (this.chart && this.queuedTicks.length) {
        const queuedTicks = this.queuedTicks.splice(0, this.queuedTicks.length);

        if (!this.chart.series[0].xData.length) {
          // console.log(`[chart.onTrades] print ${queuedTicks.length} queued historicals ticks`);
          this.tickHistoricals(queuedTicks);
        } else {
          // console.log(`[chart.onTrades] remove ${queuedTicks.length} queued historical ticks as live trades have already been printed`);
        }
      }
      
      // console.log('[onTrades]');

      this.tickTrades(trades, true);
    },
    tickHistoricals(ticks) {
      if (!this.chart) {
        this.queuedTicks = this.queuedTicks.concat(ticks);

        return;
      } else if (this.queuedTicks.length) {
        ticks = ticks.concat(this.queuedTicks.splice(0, this.queuedTicks.length));
      }

      ticks = ticks.filter(tick => 
        this.actives.indexOf(tick.exchange) !== -1 
        && this.exchanges[tick.exchange] && this.exchanges[tick.exchange].ohlc !== false
      );
      
      if (!ticks.length) {
        return;
      }

      this.createTick(ticks[0].timestamp)

      const formatedTicks = [];

      ticks.forEach((tick, index) => {
        this.tickData.buys += tick.buys;
        this.tickData.sells += tick.sells;
        this.tickData.liquidations += tick.liquidations || 0;
        
        this.tickData.exchanges[tick.exchange] = {
          open: tick.open,
          high: tick.high,
          low: tick.low,
          close: tick.close,
        }
        
        if (!ticks[index + 1] || this.tickData.timestamp !== ticks[index + 1].timestamp) {
          const formatedTick = this.formatTickData(this.tickData);
          this.addTickToSeries(formatedTick);

          formatedTicks.push(formatedTick);
          
          if (ticks[index + 1]) {
            this.createTick(ticks[index + 1].timestamp);
          }
        }
      });

      // console.log('[chart.tickHistoricals] Finished drawing', formatedTicks.length, 'candles', formatedTicks);

      // if (formatedTicks.length) {
      //   console.log('[chart.tickHistoricals] Last printed candle', new Date(formatedTicks[formatedTicks.length - 1].ohlc[0]).toLocaleString(), formatedTicks[formatedTicks.length - 1]);
      // }

      this.tickData.added = true;

      this.chart.redraw();
    },
    tickTrades(trades, live = false) {
      const ticks = [];
    
      if (!trades || !trades.length) {
        trades = [];
      }

      // chart doesn't allow edit on invisible ticks when it is panned
      // we just process them when will get there
      if (this.isPanned()) {

        this.queuedTrades = this.queuedTrades.concat(trades);

        return;

      } else if (this.queuedTrades.length) {

        // right here
        trades = this.queuedTrades.splice(0, this.queuedTrades.length).concat(trades);

      }

      // first we trim trades
      // - equal or higer than current tick
      // - only from actives exchanges (enabled, matched and visible exchange)
      trades = trades
        .filter(a =>
            a[1] >= this.cursor // only process trades >= current tick time
            && this.actives.indexOf(a[0]) !== -1 // only process trades from enabled exchanges (client side)
        );
    
      if (!trades.length) {
        return;
      }

      // console.log('[chart.tickTrades]', trades.length, live ? 'LIVE' : '');

      // define range rounded to the current timeframe
      const from = Math.floor(trades[0][1] / this.timeframe) * this.timeframe;
      const to = Math.ceil(+new Date() / this.timeframe) * this.timeframe;

      // loop through ticks in range
      for (let t = from; t < to; t += this.timeframe) {
        let i;

        if (!this.tickData || this.cursor < t) {
          this.createTick(t);
        }

        for (i = 0; i < trades.length; i++) {
          if (trades[i][1] >= t + this.timeframe) {
            break;
          }

          if (this.tickData.open === null) {
            this.tickData.open = this.tickData.high = this.tickData.low = this.tickData.close = +trades[i][2];
          }

          if (trades[i][5]) {
            switch (+trades[i][5]) {
              case 1:
                if (this.chartLiquidations) {
                  this.tickData.liquidations += trades[i][3] * trades[i][2];

                  /* this.tickData.markers.push({
                    x: trades[i][1],
                    label: `${app.getAttribute('data-symbol')}${this.$root.formatAmount(trades[i][2] * trades[i][3], 1)} liquidated <b>${trades[i][4] == 1 ? 'SHORT' : 'LONG'}</b>`,
                    symbol: 'rip'
                  }); */
                }
                break;
            }

            continue;
          }

          if (this.exchanges[trades[i][0]].ohlc !== false) {
            if (!this.tickData.exchanges[trades[i][0]]) {
              this.tickData.exchanges[trades[i][0]] = {
                open: +trades[i][2],
                high: +trades[i][2],
                low: +trades[i][2],
                close: +trades[i][2],
                size: 0
              };
            }

            this.tickData.exchanges[trades[i][0]].count++;

            this.tickData.exchanges[trades[i][0]].high = Math.max(this.tickData.exchanges[trades[i][0]].high, +trades[i][2]);
            this.tickData.exchanges[trades[i][0]].low = Math.min(this.tickData.exchanges[trades[i][0]].low, +trades[i][2]);
            this.tickData.exchanges[trades[i][0]].close = +trades[i][2];
            this.tickData.exchanges[trades[i][0]].size += +trades[i][3];

            this.tickData[(trades[i][4] > 0 ? 'buys' : 'sells') + 'Count']++;
            this.tickData[trades[i][4] > 0 ? 'buys' : 'sells'] += trades[i][3] * trades[i][2];
          }
        }

        if (i) {
          trades.splice(0, i);
        }

        ticks.push(this.formatTickData(this.tickData));
      }

      if (ticks.length && ticks[0].added) {
        // console.log('[chart.tickTrades] Updating candle', new Date(ticks[0].ohlc[0]).toLocaleString(), ticks[0]);
        this.updateCurrentTick(ticks[0], live);

        ticks.splice(0, 1);
      }

      for (let i = 0; i < ticks.length; i++) {
        // console.log('[chart.tickTrades] Printing new candle', new Date(ticks[i].ohlc[0]).toLocaleString(), ticks[i]);
        this.addTickToSeries(ticks[i], live, i === ticks.length - 1);
      }

      this.chart.redraw();

      this.tickData.added = true;
      this.tickData.markers.splice(0, this.tickData.markers.length);

      window.chart = this.chart;
    },
    getCurrentTickPoints() {
      return {
        ohlc: this.chart.series[0].points[this.chart.series[0].points.length - 1],
        buys: this.chart.series[1].points[this.chart.series[1].points.length - 1],
        sells: this.chart.series[2].points[this.chart.series[2].points.length - 1],
        liquidations: this.chart.series[3].points[this.chart.series[3].points.length - 1],
      };
    },
    createTick(timestamp = null) {
      if (timestamp) {
        this.cursor = timestamp;
      } else if (this.cursor) {
        this.cursor += this.timeframe;
      } else {
        this.cursor = Math.floor(+new Date() / this.timeframe) * this.timeframe;
      }

      if (this.tickData) {
        this.tickData.timestamp = this.cursor;
        this.tickData.markers = [];

        for (let exchange in this.tickData.exchanges) {
          this.tickData.exchanges[exchange].count = 0;
          this.tickData.exchanges[exchange].open = this.tickData.exchanges[exchange].close;
          this.tickData.exchanges[exchange].high = this.tickData.exchanges[exchange].close;
          this.tickData.exchanges[exchange].low = this.tickData.exchanges[exchange].close;
        }

        this.tickData.open = parseFloat(this.tickData.close);
        this.tickData.high = null;
        this.tickData.low = null;
        this.tickData.close = 0;
        this.tickData.liquidations = 0;
        this.tickData.buys = 0;
        this.tickData.sells = 0;
        this.tickData.buysCount = 0;
        this.tickData.sellsCount = 0;
        this.tickData.added = false;
      } else {
        this.tickData = {
          timestamp: this.cursor,
          markers: [],
          exchanges: {},
          open: null,
          high: null,
          low: null,
          close: null,
          liquidations: 0,
          buys: 0,
          sells: 0,
          buysCount: 0,
          sellsCount: 0,
          added: false,
        }
      }
    },
    addTickToSeries(tick, live = false, snap = false) {
      this.chart.series[0].addPoint(tick.ohlc, false);
      this.chart.series[1].addPoint(tick.buys, false);
      this.chart.series[2].addPoint(tick.sells, false);

      if (this.chartLiquidations) {
        this.chart.series[3].addPoint(tick.liquidations, false);
      }

      if (tick.markers && tick.markers.length) {
        this.processTickMarkers(tick);
      }

      if (snap && this.isSnaped) {
        this.snapRight(live);
      }
    },
    updateCurrentTick(tick, live = false) {
      // console.log(`s
      //   \txData: ${this.chart.series[0].xData.length} 
      //     (${this.chart.series[0].xData.length ? new Date(this.chart.series[0].xData[this.chart.series[0].xData.length - 1]).toLocaleString() : 'N/A'}) 
      //   points: ${this.chart.series[0].points.length} 
      //     (${this.chart.series[0].points.length ? new Date(this.chart.series[0].points[this.chart.series[0].points.length - 1].x).toLocaleString() : 'N/A'})`);
      
      // if (!live) {
      //   for (let i = 0; i < Math.max(this.chart.series[0].xData.length, this.chart.series[0].points.length); i++) {
      //     console.log(
      //       this.chart.series[0].xData[i] ? new Date(this.chart.series[0].xData[i]).toLocaleString() : 'N/A',
      //       this.chart.series[0].points[i] ? new Date(this.chart.series[0].points[i].x).toLocaleString() : 'N/A',
      //     )
      //   }
      // }

      const tickPoints = this.getCurrentTickPoints();

      if (this.chartLiquidations) {
        tickPoints.liquidations.update(tick.liquidations[1], false);
      }

      tickPoints.buys.update(tick.buys[1], false);
      tickPoints.sells.update(tick.sells[1], false);
      tickPoints.ohlc.update(tick.ohlc, false);

      if (tick.markers.length) {
        this.processTickMarkers(tick);
      }
    },
    processTickMarkers(tick) {
      if (!tick.markers.length) {
        return;
      }

      tick.markers.forEach(marker =>
        this.createMarker(marker.x || tick.ohlc[0], marker.y || tick.ohlc[4], marker.label, marker.symbol)
      )
    },
    createMarker(x, y, label, symbol) {
      const point = {
        x: x,
        y: y,
        marker: {
          radius: 8,
          lineWidth: 4,
          symbol: symbol,
          fillColor: 'white'
        },
        name: label
      };

      this.chart.series[4].addPoint(point);
    },

    formatTickData(tickData) {
      return {
        buys: [tickData.timestamp, tickData.buys],
        sells: [tickData.timestamp, tickData.sells],
        liquidations: [tickData.timestamp, tickData.liquidations],
        ohlc: this.getExchangesAveragedOHLC(tickData.exchanges, tickData),
        markers: tickData.markers,
        added: tickData.added
      }
    },
    getExchangesAveragedOHLC(exchanges, tickData) {
      let totalWeight = 0;
      let setOpen = false;

      if (tickData.open === null) {
        setOpen = true;
        tickData.open = 0;
      }

      let high = 0;
      let low = 0;

      tickData.close = 0;

      for (let exchange in exchanges) {
        let exchangeWeight = 1;

        if (setOpen) {
          tickData.open += exchanges[exchange].open * exchangeWeight;
        }

        high += exchanges[exchange].high * exchangeWeight;
        low += exchanges[exchange].low * exchangeWeight;
        tickData.close += ((exchanges[exchange].close + exchanges[exchange].high + exchanges[exchange].low) / 3) * exchangeWeight;

        totalWeight += exchangeWeight;
      }

      if (setOpen) {
        tickData.open /= totalWeight;
      }

      high /= totalWeight;
      low /= totalWeight;
      tickData.close /= totalWeight;

      tickData.high = Math.max(tickData.high === null ? 0 : tickData.high, setOpen ? 0 : tickData.open, high);
      tickData.low = Math.min(tickData.low === null ? Infinity : tickData.low, setOpen ? Infinity : tickData.open, low);

      return [
        tickData.timestamp,
        tickData.open,
        tickData.high,
        tickData.low,
        tickData.close
      ];
    },
    startResize(event) {
      if (event.which === 3) {
        return;
      }

      this.resizing = event.pageY;
    },

    resetHeight(event) {
      delete this.resizing;

      this.$store.commit('setChartHeight', null);

      this.updateChartHeight();
    },

    startScale(event) {
      if (event.which === 3) {
        return;
      }

      this.scaling = event.pageY;
    },

    updateChartHeight(height = null) {
      const size = this.getChartSize();

      if (window.innerWidth >= 768) {
        height = this.$el.clientHeight;
      }

      this.chart.setSize(
        size.width,
        height || size.height,
        false
      );
    },

    resetScale(event) {
      delete this.scaling;

      this.updateChartScale(null);
    },

    updateChartScale(scale) {
      let min = this.chart.yAxis[0].min;
      let max = this.chart.yAxis[0].max;

      if (scale === null) {
        min = max = null;
      } else if (scale) {
        min -= scale;
        max += scale;
      }

      this.chart.yAxis[0].update({
        min: min,
        max: max
      })
    },

    getChartSize() {
      const w = this.$refs.chartContainer.offsetWidth;
      const h = document.documentElement.clientHeight;

      return {
        width: w,
        height: this.chartHeight > 0 ? this.chartHeight : +Math.min(w / 2, Math.max(300, h / 3)).toFixed()
      };
    },
    createAndAppendEmptyTick() {
      this.createTick();

      const tick = this.formatTickData(this.tickData);

      this.addTickToSeries(tick);

      this.tickData.added = true;

      this.chart.redraw();
    },

    doResize(event) {
      clearTimeout(this._resizeTimeout);

      this._resizeTimeout = setTimeout(this.updateChartHeight.bind(this), 250);
    },

    doZoom(event, two = false) {
      this.timestamp = +new Date();

      if (!this.chart.series[0].xData.length) {
        return;
      }

      event.preventDefault();
      let axisMin = this.chart.xAxis[0].min;
      let axisMax = this.chart.xAxis[0].max;

      if (event.deltaX || event.deltaZ || !event.deltaY) {
        return;
      }

      const multiplier = (event.deltaY > 0 ? 1 : -1);

      let range = (axisMax - axisMin) * .05;

      this.chart.xAxis[0].setExtremes(axisMin - range * multiplier, axisMax + range * multiplier);

      axisMin = this.chart.yAxis[0].min;
      axisMax = this.chart.yAxis[0].max;

      range = (axisMax - axisMin) * .02;

      this.chart.yAxis[0].setExtremes(axisMin - range * multiplier, axisMax + range * multiplier);

    },

    doPan(self) {
      return function(proceed, event, arg, c) {
        if (!self.chart) {
          return;
        }
        
        const isPanned = self.isPanned();

        if (!isPanned !== self.isSnaped) {
          self.$store.commit('toggleSnap', !isPanned);
        }

        return proceed.call(self.chart, event, arg);
      };
    },

    doDrag(event) {
      if (!isNaN(this.resizing)) {
        this.updateChartHeight(this.chart.chartHeight + (event.pageY - this.resizing));

        this.resizing = event.pageY;
      } else if (!isNaN(this.scaling)) {
        this.updateChartScale((event.pageY - this.scaling) / 10, true);

        this.scaling = event.pageY;
      }
    },

    stopDrag(event) {
      if (this.resizing) {
        this.$store.commit('setChartHeight', this.chart.chartHeight);

        delete this.resizing;
      }

      if (this.scaling) {        
        delete this.scaling;
      }
    },

    snapRight(redraw = false) {
      const now = +new Date();
      const margin = this.chartRange * this.chartPadding;

      this.chart.xAxis[0].setExtremes(now - this.chartRange + margin, now + margin, redraw);
    },

    buildExchangesSeries() {
      for (let i = 4; i < this.chart.series.length; i++) {
        this.chart.series[i].remove();
      }

      for (let exchange of this.actives) {
        this.chart.addAxis({
          id: exchange + '-axis',
          visible: false,
          height: '33%',
        }, false);

        this.chart.addSeries({
          type: 'spline',
          name: exchange,
          yAxis: exchange + '-axis',
          data: []
        }, false);
      }

      setTimeout(() => this.setTimeframe(this.timeframe), 100);
    }

    /*stopTickInterval() {

      clearTimeout(this._tickTimeout);
      clearInterval(this._tickInterval);
    },*/
    /*tickInterval() {
      const now = +new Date();
      const timeToFirstTick = Math.ceil(now / this.timeframe) * this.timeframe;

      this.createAndAppendEmptyTick();

      this._tickTimeout = setTimeout(() => {
        this._tickInterval = setInterval(() => {
          this.createTick();

          const tick = this.formatTickData(this.tickData);

          this.chart.series[0].addPoint(tick.ohlc, false);
          this.chart.series[1].addPoint(tick.buys, false);
          this.chart.series[2].addPoint(tick.sells, false);

          this.tickData.added = true;

          this.chart.redraw();
        }, this.timeframe);
      }, timeToFirstTick - now);
    }*/,
    isPanned() {
      if (!this.chart || !this.chart.series.length) {
        return true;
      }
      
      return this.tickData && this.chart.series[0].points.length && this.chart.series[0].points[this.chart.series[0].points.length - 1].x < this.tickData.timestamp
    },
    setRange(range) {
      this.$store.commit('setChartRange', range);

      if (this.chart) {
        console.log(`[chart.setRange]`, this.chartRange, this.chartPadding ? `with ${(this.chartPadding * 100).toFixed(2)})% margin` : '');

        const margin = this.chartRange * this.chartPadding;
        const now = +new Date();

        this.chart.xAxis[0].update({
            overscroll: margin
        });

        this.chart.xAxis[0].setExtremes(now - this.chartRange + margin, now + margin, true);
      }
    },
    getChartOptions() {
      const options = JSON.parse(JSON.stringify(chartOptions));
      const theme = options.themes[this.dark ? 'dark' : 'bright'];
      
      // time axis
      options.xAxis.labels.color = theme.labels;
      options.xAxis.crosshair.color = theme.crosshair;

      // price axis
      options.yAxis[0].labels.color = theme.labels;
      options.yAxis[0].gridLineColor = theme.gridline;
      options.yAxis[0].crosshair.color = theme.crosshair;

      // candlesticks
      options.series[0].upColor = theme.up;
      options.series[0].upLineColor = theme.upLine;
      options.series[0].color = theme.down;
      options.series[0].lineColor = theme.downLine;

      // buys
      options.series[1].color = theme.buysLine;
      options.series[1].fillColor = {
        linearGradient: {
          x1: 0, y1: 0, x2: 0, y2: 1
        },
        stops: [[0, theme.buysGradient[0]], [1, theme.buysGradient[1]]]
      };

      // sells
      options.series[2].color = theme.sellsLine;
      options.series[2].fillColor = {
        linearGradient: {
          x1: 0, y1: 0, x2: 0, y2: 1
        },
        stops: [[0, theme.sellsGradient[0]], [1, theme.sellsGradient[1]]]
      };

      // liquidations bars
      options.series[3].color = theme.liquidations;

      // price MA
      options.series[4].lineColor = theme.priceMA;

      // sells EMA
      options.series[5].lineColor = theme.sellsMA;

      // buys EMA
      options.series[6].lineColor = theme.buysMA;

      options.series[0].type = this.chartCandlestick ? 'candlestick' : 'spline';
      options.series[0].lineColor = this.chartCandlestick ? options.series[0].downLine : 'white';
      options.series[0].lineWidth = this.chartCandlestick ? 1 : 2;

      return options;
    },
    onClean(min) {
      this.setTimeframe(this.timeframe)
    }
  }
};
</script>

<style lang='scss'>
@import '../../assets/sass/variables';

.chart__range {
  display: flex;
  justify-content: space-between;
  position: absolute;
  width: 100%;
  height: 0px;
  font-size: 14px;
  opacity: 0.4;
  font-family: monospace;
  font-weight: 300;
  letter-spacing: -0.5px;

  > div {
    padding: 10px;
  }
}

.chart__container {
  position: relative;
  width: calc(100% + 1px);

  .highcharts-container {
    width: 100% !important;
  }

  .chart__selection {
    position: absolute;
    top: 0;
    bottom: 0;
    background-color: rgba(black, 0.1);
    z-index: 1;
    pointer-events: none;
  }

  &.fetching {
    opacity: 0.5;
  }

  .highcharts-credits {
    visibility: hidden;
  }
}

.chart__scale-handler,
.chart__height-handler {
  position: absolute;
  bottom: 0;
  z-index: 2;
}

.chart__scale-handler {
  right: 0;
  top: 0;
  width: 7%;
  cursor: ns-resize;
}

.chart__height-handler {
  left: 0;
  right: 0;
  height: 8px;
  margin-top: -4px;
  cursor: row-resize;

  @media screen and (min-width: 768px) {
    display: none;
  }
}

.chart_timeframe {
  position: absolute;
  margin: 10px;
  z-index: 1;
}

.highcharts-tooltip-box tspan {
  font-weight: 400 !important;
}

.highcharts-yaxis-grid path:first-child {
  display: none;
}
</style>
