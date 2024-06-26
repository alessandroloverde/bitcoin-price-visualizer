<template>
<div id="app" :class="modeValue">
  <div class="appContainer">
    <Loading
      :active="!loaded"
      :can-cancel="false"
      :is-full-page="true"
    />
    <section class="controls">
      <header>
        <h1><span class="icon">&#8383;</span>Bitcoin price visualizer </h1>
        <fieldset>
          <label for="mode-selector">Mode selector</label>
          <select name="mode-selector" id="modeSelector" v-model="modeValue" @change="updateChart()">
            <option value="darkTheme">Fear of the dark</option>
            <option value="lightTheme">Fear of the light</option>
          </select>
        </fieldset>
      </header>

      <form @submit.prevent="updateChart()">
        <fieldset>
          <label for="startDate">Start date</label>
          <Datepicker
            name="startDate"
            :value="startingDate"
            :disabled-dates="disabledStartDates"
            @selected="updateDate($event, 'startDate')"
          />
        </fieldset>
        <fieldset>
          <label for="endDate">End date</label>
          <Datepicker
            name="endDate"
            :value="endingDate"
            :disabled-dates="disabledEndDates"
            @selected="updateDate($event, 'endDate')"
          />
        </fieldset>
        <button type="submit">Render chart</button>
      </form>
    </section>
    <line-chart 
        v-if="loaded" 
        :chart-data="chartData" 
        :solidColor="solidColor" 
        :transparentColor="transparentColor"
    />
  </div>
</div>
</template>

<script>
import Loading from 'vue-loading-overlay'
import LineChart from './components/Chart.vue'
import Datepicker from 'vuejs-datepicker'
import moment from 'moment'
import 'vue-loading-overlay/dist/vue-loading.css'

export default {
  name: 'App',
  components: {
    LineChart, Datepicker, Loading
  },
  data () {
    return {
      loaded: false,
      modeValue: 'darkTheme',
      solidColor: 'red',
      transparentColor: 'red',
      startingDate: moment(new Date()).subtract(20, 'days').format('YYYY-MM-DD'),
      endingDate: moment(new Date()).format('YYYY-MM-DD'),
      style: getComputedStyle(document.getElementById('app')),
      prices: null,
      chartData: {
        labels: [],
        datasets: [{
          data: [],
          startDate: null,
          endDate: null
        }]
      },
      disabledStartDates: {
        from: new Date()
      }
    }
  },
  computed: {
    startSource: function () {
      return this.chartData.datasets[0].startDate
    },
    endSource: function () {
      return this.chartData.datasets[0].endDate
    },
    disabledEndDates: function () {
      return {
        from: new Date(),
        to: new Date(this.startSource)
      }
    }
  },
  methods: {
    updateDate (date, type) {
      type === 'startDate'
      ? this.startingDate = moment(date).format('YYYY-MM-DD') 
      : this.endingDate = moment(date).format('YYYY-MM-DD')
    },
    async updateChart () {
      this.loaded = false

      await this.fetchData()
    },
    async fetchData () {
      try {
        const startTimestamp = Math.floor(new Date(this.startingDate).getTime() / 1000);
        const endTimestamp = Math.floor(new Date(this.endingDate).getTime() / 1000);
        
        /* const response = await fetch(`https://api.coingecko.com/api/v3/coins/bitcoin/market_chart?vs_currency=usd&days=8&interval=daily`) */
        const response = await fetch(`https://api.coingecko.com/api/v3/coins/bitcoin/market_chart/range?vs_currency=usd&from=${startTimestamp}&to=${endTimestamp}`)
        const fetchData = await response.json()

        this.prices = fetchData.prices.reduce((acc, [timestamp, price]) => {
          const date = new Date(timestamp).toISOString().split('T')[0];

          acc[date] = price.toFixed(0);

          return acc;
        }, {})

        const style = getComputedStyle(document.getElementById('app'))
        // const style = getComputedStyle(document.body);

        this.solidColor = style.getPropertyValue('--accentColour')
        this.transparentColor = this.solidColor.substring(0, this.solidColor.length - 2) + '0)'

        this.chartData = await {
          labels: Object.keys(this.prices),
          datasets: [{
            data: Object.values(this.prices),
            borderColor: style.getPropertyValue('--accentColour'),
            borderWidth: 3,
            startDate: this.startingDate,
            endDate: this.endingDate
          }]
        }

        this.loaded = true

      } catch (error) {
        this.loaded = true
        console.log('error', error)
      }
    }
  },
  async mounted () {
    this.loaded = true
    this.chartData.datasets[0].startDate = this.startingDate
    this.chartData.datasets[0].endDate = this.endingDate
    
    await this.fetchData()
  }
}
</script>

<style lang="scss">
  @mixin buttonAspect {
    padding: $buttonsPadding;
    color: var(--txtColour-2);
    font-family: 'Arimo', sans-serif;
    font-weight: 800;
    font-size: 1rem;
    border-radius: $borderRadius;
    border: 2px solid transparent;
  }
  * {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
  }
  html { height: 100%; }
  body {
    position: relative;
    min-height: 100vh;
    font-family: 'Arimo', sans-serif;
    -moz-osx-font-smoothing: grayscale;
    -webkit-font-smoothing: antialiased;
  }
   #app {
    position: absolute;
    width: 100%;
    height: 100%;
    //background: var(--bkgColour-lighter);
    background: linear-gradient(to bottom right, var(--bkgColour-lighter), var(--bkgColour-light)) no-repeat;
  }
  h1 {
    font-size: 3rem;
    text-transform: capitalize;
    display: flex;
    align-items: center;
  }

  @media screen and (max-width: 979px) {
    .appContainer {
      margin: 0;
      border-radius: 0;
      padding: 2rem;
    }
    .controls {
      & > header {
        flex-direction: column;

        & fieldset {
          display: flex;
          align-items: center;
          text-align: center;
          margin-top: 1.5rem;
        }
        h1 {
          font-size: 2rem;

          .icon {
            flex-basis: 2.5em;
            height: auto;
          }
        }
      }
      form {
        flex-direction: column;
        align-items: center;

        fieldset { margin: 0.5rem auto; }
        button[type="submit"] {
          margin: 1rem auto;
        }
      }
    }
  }
</style>
