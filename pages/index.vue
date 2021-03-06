<!--
   Authors: Wesley Ford (ntohq) 
            Sergix
   Date: 03/07/2020
   Updated: 03/31/2021
   By: Wesley Ford (ntohq)
 -->

<template>
  <section class="section">
    <div>
      <div class="tile is-ancestor">
        <div class="tile is-parent is-vertical">
          <b-field
            class="tile is-child"
            style="max-height: 50px"
            label="Ticker"
          >
            <b-input
              v-model="tickerInput"
              type="text"
              style="max-width: 200px"
              @input="debounceInput"
            ></b-input>
          </b-field>
          <ul class="tile is-child">
            <li
              v-for="ticker in searchResults"
              :key="ticker.symbol"
              @click="
                tickerInput = ticker.symbol
                companyName = ticker.name
                fetchSearchData().then(
                  fetchStockData().then(
                    getAnalystReport().then(fetchNewsData())
                  )
                )
              "
            >
              <b-tooltip :label="ticker.name" position="is-right" animated>
                {{ ticker.symbol }}
              </b-tooltip>
            </li>
          </ul>
          <!-- </b-tab-item>
            <b-tab-item label="By Sector"></b-tab-item>
          </b-tabs> -->
        </div>
        <div style="margin-bottom: 15px; flex: 3">
          <TestChart
            :cc-type="chartTypes[currentStyle]"
            :chart-name="chartLabel"
            :tiker-name="tickerInput"
            :company-name="companyName"
            :labels="labels"
            :input-data="data"
          ></TestChart>
          <div>
            <b-dropdown
              v-model="currentStyle"
              :scrollable="true"
              aria-role="list"
              mobile-modal="true"
            >
              <button slot="trigger" class="button is-primary" type="button">
                <template>
                  <span>{{ currentStyle }}</span>
                </template>
                <b-icon icon="caret-down"></b-icon>
              </button>
              <b-dropdown-item
                v-for="(value, name) in chartTypes"
                :key="name"
                :value="name"
                aria-role="listitem"
              >
                <span>{{ name }}</span>
              </b-dropdown-item>
            </b-dropdown>
          </div>
        </div>
        <div class="tile is-parent">
          <div class="tile is-child is-vertical">
            <b-button
              style="margin-bottom: 2em"
              icon-left="link"
              type="is-primary"
              @click="quickLink()"
              >Quick Link</b-button
            >
            <b-field label="Period">
              <b-select v-model="period">
                <option value="max">Max</option>
                <option value="10y">Ten years</option>
                <option value="5y">Five years</option>
                <option value="2y">Two years</option>
                <option value="1y">One year</option>
                <option value="6mo">Six months</option>
                <option value="3mo">Three months</option>
                <option value="1mo">One month</option>
                <option value="1wk">One week</option>
                <option value="5d">Five days</option>
                <option value="1d">One day</option>
              </b-select>
            </b-field>
            <b-field label="Interval">
              <b-select v-model="interval">
                <option value="3mo">Three months</option>
                <option value="1mo">One month</option>
                <option value="1wk">One week</option>
                <option value="5d">Five days</option>
                <option value="1d">One day</option>
                <option value="1h">One hour</option>
                <option value="30m">Thirty minutes</option>
                <option value="15m">Fifteen minutes</option>
                <option value="5m">Five minutes</option>
                <option value="2m">Two minutes</option>
                <option value="1m">One minutes</option>
              </b-select>
            </b-field>
            <b-button style="margin-top: 1em" @click="fetchStockData"
              >Fetch Data</b-button
            >
          </div>
        </div>
      </div>
      <section style="margin-top: 5vh">
        <b-collapse animation="slide">
          <template #trigger="props">
            <div
              class="card-header"
              role="button"
              style="justify-content: right; margin-bottom: 10px"
            >
              <a class="card-header-icon">
                <b-icon :icon="props.open ? 'caret-up' : 'caret-down'">
                </b-icon>
              </a>
            </div>
          </template>
          <b-tabs position="is-centered">
            <b-tab-item label="Stats" icon="chart-pie">
              <div :style="{ 'margin-bottom': '75px', display: displayReport }">
                <analystSummary
                  :summary="analystReport.summary"
                  :sector="analystReport.sector"
                  :employees="analystReport.employees"
                  :name="analystReport.name"
                >
                  <analystStats :content="report"></analystStats>
                </analystSummary>
              </div>
            </b-tab-item>
            <b-tab-item label="News" icon="newspaper" pack="far">
              <h1 class="title is-6"><u>Stock News</u></h1>
              <aside class="section cards" style="padding-top: 0">
                <section class="box tile" style="padding: 20px">
                  <div style="max-height: 100vh; overflow-y: scroll">
                    <div
                      v-for="article in articles"
                      :key="article.title"
                      class="card"
                    >
                      <div class="card-image">
                        <figure class="image">
                          <img :src="article.urlImage" />
                        </figure>
                      </div>
                      <div class="card-content">
                        <div class="media">
                          <div class="media-content" style="overflow: hidden">
                            <p class="title is-4">{{ article.title }}</p>
                          </div>
                        </div>
                      </div>
                      <div class="card-footer">
                        <a :href="article.url" class="card-footer-item"
                          >Open article</a
                        >
                      </div>
                    </div>
                  </div>
                </section>
              </aside>
            </b-tab-item>
            <b-tab-item label="ML" icon="project-diagram">
              <h1 class="title is-6"><u>Machine Learning Stats</u></h1>
            </b-tab-item>
          </b-tabs>
        </b-collapse>
      </section>
    </div>
  </section>
</template>

<script>
import { mapGetters } from 'vuex'
import TestChart from '@/components/charts/ApexChart'
import { buildRequest } from '@/util/request.js'
import { switchcaseF } from '@/util/switchcase.js'
import { errorNotificationFactory } from '@/util/notification.js'
import analystSummary from '@/components/businessSummary'
import analystStats from '@/components/analystStats'

const _ = require('lodash')
const unknownErrorNotification = errorNotificationFactory(
  `Unknown error! Refresh the page and try again.`
)
const serverErrorNotification = errorNotificationFactory(
  'Internal server error!'
)

export default {
  name: 'HomePage',
  components: {
    TestChart,
    analystSummary,
    analystStats
  },
  data() {
    return {
      labels: [],
      data: [],
      tickerInput: '',
      chartLabel: '',
      searchResults: [],
      period: '',
      interval: '',
      articles: [],
      currentStyle: 'Line Chart',
      chartTypes: { 'Line Chart': 'line', 'Candle Stick Chart': 'candlestick' },
      analystReport: {
        name: '',
        summary: '',
        employees: '',
        sector: ''
      },
      report: {},
      displayReport: 'none'
    }
  },
  computed: {
    ...mapGetters(['getSetBroker'])
  },
  methods: {
    debounceInput: _.debounce(function() {
      this.fetchSearchData()
    }, 500),
    async fetchStockData() {
      const query = buildRequest({
        url: `https://analyst.herokuapp.com/stockQuery/?`,
        ticker: this.tickerInput,
        period: this.period,
        interval: this.interval
      })
      const data = await this.$axios.$get(query)
      this.chartLabel = this.tickerInput
      this.labels = Object.keys(data)
      this.data = Object.values(data)
    },

    async fetchSearchData() {
      const query = buildRequest({
        url: `https://analyst.herokuapp.com/searchQuery/?`,
        ticker: this.tickerInput
      })

      try {
        this.searchResults = (await this.$axios.$get(query)).ticker_data
      } catch (error) {
        switchcaseF({
          500: serverErrorNotification
        })(unknownErrorNotification)(error.response.status)
      }
    },

    async fetchNewsData() {
      const query = buildRequest({
        url: `https://analyst.herokuapp.com/newsQuery/?`,
        ticker: this.companyName
      })

      try {
        const articles = await this.$axios.$get(query)
        this.articles = articles.articles
      } catch (error) {
        switchcaseF({
          500: serverErrorNotification
        })(unknownErrorNotification)(error.response.status)
      }
    },

    quickLink() {
      const broker = this.$store.state.brokers.set
      // eslint-disable-next-line
      console.debug(this.$store.state.settings.options.quick_link)
      const quickLink = this.$store.state.settings.options.quick_link
      if (broker.name !== 'none' && quickLink === true) {
        if (this.tickerInput !== undefined || this.tickerInput !== '') {
          const brokerUrl = broker.link.replace('{tiker}', this.tickerInput)
          window.open(brokerUrl, '_blank')
        }
      } else if (quickLink === false || !quickLink) {
        this.$buefy.dialog.alert("Turn on quick link in the setting's tab")
      } else if (broker.name === 'none') {
        this.$buefy.dialog.alert("Please select a broker in the setting's tab")
      }
    },
    async getAnalystReport() {
      const query = buildRequest({
        url: `https://analyst.herokuapp.com/analyst-report/?`,
        ticker: this.tickerInput
      })

      const report = await this.$axios.$get(query)
      this.report = report
      this.analystReport.name = report.name
      this.analystReport.summary = report.BusinessSummary
      this.analystReport.employees = report.employees
        .toString()
        .replace(/\B(?=(\d{3})+(?!\d))/g, ',')
      this.analystReport.sector = report.sector
      if (this.displayReport !== '') {
        this.displayReport = ''
      }
    }
  }
}
</script>

<style lang="css" scoped>
#more {
  display: none;
}

a.dropdown-item.is-active span,
.dropdown .dropdown-menu .has-link a.is-active span,
button.dropdown-item.is-active span {
  color: white;
}

div.tabItem {
  display: flex;
  flex-direction: row;
}

div.tabItem div.field.tile.is-child {
  display: contents;
}
</style>
