<template>
  <div class="d-flex h-100 w-100 flex-column">
    <header class="p-4 mb-auto">
      <img
        src="static/ride-receipts.svg"
        alt="Ride Receipts"
        width="253"
      >
    </header>
    <main
      class="mt-5"
      @keyup.enter="submitForm"
    >
      <transition
        name="fade"
        mode="out-in"
      >
        <section
          v-if="form === 'LOGIN_FORM'"
          key="loginForm"
          class="p-3 text-center"
        >
          <div class="row">
            <div class="col-md-10 mx-auto">
              <p class="sign-in-text mb-5">
                Sign in to your Gmail account to automatically download your Uber receipts. <i
                  id="privacy"
                  class="far fa-2x fa-question-circle"
                />
              </p>
              <p
                v-if="!awaiting_code"
                class="text-center"
              >
                <button
                  v-if="!loading"
                  type="button"
                  class="btn btn-lg btn-started"
                  @click="signInWithgoogle('google')"
                >
                  Sign In to Gmail
                </button>
              </p>
              <b-popover
                ref="popover"
                target="privacy"
                triggers="click focus"
                placement="bottom"
              >
                <template slot="title">
                  Privacy
                </template>
                Ride Receipts is an automation app that has no database; therefore, it does not store your login credentials, personal information or any other data. Once you log in, we’ll fetch your Uber or Lyft receipts and auto-generate PDFs for you.
                <br>
                <p class="text-right">
                  <a
                    class="js-external-link"
                    href="https://ridereceipts.io/privacy"
                  >Learn more</a>
                </p>
              </b-popover>
              <div
                v-if="awaiting_code"
                class="col-md-10 mx-auto"
              >
                <div class="form-group">
                  <b-form-input
                    id="input-live"
                    v-model="approval_code"
                    placeholder="Enter approval code from browser"
                    trim
                  />
                </div>
                <p class="text-center">
                  <button
                    type="button"
                    class="btn btn-lg btn-started"
                    @click="fetchGoogleToken('google')"
                  >
                    Submit
                  </button>
                </p>
              </div>
            </div>
          </div>
        </section>
        <section
          v-if="form === 'FILTER_OPTION'"
          key="filteroption"
        >
          <div class="row">
            <div class="col-8 mx-auto">
              <p class="sign-in-text text-center mb-5">
                Which receipts would you like to <br> download?
              </p>
            </div>
            <div class="col-7 mx-auto">
              <div class="form-group">
                <div class="row mx-auto">
                  <div class="col">
                    <div class="form-check">
                      <label class="form-check-label">
                        <input
                          v-model="filter_option"
                          class="form-check-input"
                          type="radio"
                          name="filter"
                          value="previousyear"
                        >
                        Previous year
                      </label>
                    </div>
                    <div class="form-check">
                      <label class="form-check-label">
                        <input
                          v-model="filter_option"
                          class="form-check-input"
                          type="radio"
                          name="filter"
                          value="currentyear"
                        >
                        Current year
                      </label>
                    </div>
                  </div>
                  <div class="col">
                    <div class="form-check">
                      <label class="form-check-label">
                        <input
                          v-model="filter_option"
                          class="form-check-input"
                          type="radio"
                          name="filter"
                          value="lastthreemonths"
                        >
                        Last 3 months
                      </label>
                    </div>
                    <div class="form-check">
                      <label class="form-check-label">
                        <input
                          v-model="filter_option"
                          class="form-check-input"
                          type="radio"
                          name="filter"
                          value="lastmonth"
                        >
                        Last month
                      </label>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        </section>
        <section
          v-if="form === 'INVOICE_COUNT'"
          key="invoicecounts"
        >
          <div class="row">
            <div class="col-8 mx-auto">
              <p class="sign-in-text mb-5 text-center">
                {{ downloadingMessage }}
              </p>
              <br>
              <div class="progress">
                <div
                  class="progress-bar"
                  role="progressbar"
                  :style="{ width: progress + '%' }"
                  :aria-valuenow="progress"
                  aria-valuemin="0"
                  aria-valuemax="100"
                />
              </div>
            </div>
          </div>
        </section>
        <section
          v-if="form === 'DOWNLOADED'"
          key="downloaded"
        >
          <div class="row">
            <div class="col-8 mx-auto">
              <p
                v-if="invoiceCount > 0"
                class="sign-in-text mb-4 text-center"
              >
                Success! All receipts have been downloaded for you.
              </p>
              <p
                v-if="invoiceCount === 0"
                class="sign-in-text mb-4 text-center"
              >
                {{ downloadingMessage }}
              </p>
              <div
                v-if="invoiceCount > 0"
                class="card-deck mb-5"
              >
                <div class="card">
                  <div class="card-body d-flex flex-row">
                    <img
                      src="static/rideshare-car.svg"
                      width="86"
                      class="mr-4"
                    >
                    <p class="card-text">
                      Number of trips<br>
                      <span
                        v-if="invoiceCount > 1"
                        class="trip-count"
                      >{{ invoiceCount }} trips</span>
                      <span
                        v-if="invoiceCount === 1"
                        class="trip-count"
                      >{{ invoiceCount }} trip</span>
                    </p>
                  </div>
                </div>
                <div class="card">
                  <div class="card-body">
                    <carousel
                      :navigation-enabled="navigation"
                      :pagination-enabled="pagination"
                      :per-page="perPage"
                    >
                      <slide
                        v-for="rate in rates"
                        :key="rate.currency"
                        class="d-flex flex-row"
                      >
                        <img
                          src="static/piggy-bank.svg"
                          width="86"
                          class="mr-4"
                        >
                        <p class="card-text">
                          Total spend<br>
                          <span class="trip-count">{{ rate.currency }} {{ Number.parseFloat(rate.amount.reduce((a, b) => a + b, 0) * 100 / 100).toFixed(2) }}</span>
                        </p>
                      </slide>
                    </carousel>
                  </div>
                </div>
              </div>
              <p
                v-if="invoiceCount > 0"
                class="text-center"
              >
                <button
                  type="button"
                  class="btn btn-lg btn-started"
                  @click.stop.prevent="openInvoiceFolder()"
                >
                  View Receipts
                </button>
              </p>
              <p
                v-if="invoiceCount > 0"
                class="text-center"
              >
                Run again: <a
                  href="#"
                  class="mr-1 font-weight-bold"
                  @click="startAgain"
                >Uber</a> <router-link
                  :to="{ name: 'lyft'}"
                  class="font-weight-bold"
                  tag="a"
                >
                  Lyft
                </router-link>
              </p>
              <p
                v-if="invoiceCount === 0"
                class="text-center"
              >
                <router-link
                  :to="{ name: 'main-page' }"
                  class="btn btn-lg btn-started"
                  tag="button"
                >
                  Start again
                </router-link>
              </p>
            </div>
          </div>
        </section>
        <section
          v-if="form === 'CHROME_NOT_FOUND'"
          key="chomenotfound"
        >
          <div class="row">
            <div class="col-10 mx-auto">
              <p class="sign-in-text text-center">
                It seems like you don't have Chromium installed. Please download it from <a
                  class="js-external-link"
                  href="https://download-chromium.appspot.com/"
                >here</a>.
              </p>
            </div>
          </div>
        </section>
      </transition>
    </main>
    <footer
      v-if="form === 'DOWNLOADED'"
      class="mt-auto"
    />
    <footer
      v-if="form === 'DOWNLOADED'"
      class="mt-auto p-4"
    >
      <div class="row">
        <div class="col-md-10 mx-auto">
          <p class="text-center">
            <a
              href="https://ridereceipts.io"
              class="upgrade-link js-external-link"
            >Upgrade to Ride Receipts PRO and get an itemized Excel doc of all your trips. <svg
              xmlns="http://www.w3.org/2000/svg"
              width="24"
              height="24"
              viewBox="0 0 24 24"
              fill="none"
              stroke="currentColor"
              stroke-width="2"
              stroke-linecap="round"
              stroke-linejoin="round"
              class="feather feather-arrow-right"
            ><line
              x1="5"
              y1="12"
              x2="19"
              y2="12"
            /><polyline points="12 5 19 12 12 19" /></svg></a>
          </p>
        </div>
      </div>
    </footer>
    <footer
      v-if="form !== 'DOWNLOADED'"
      class="mt-auto p-4"
    >
      <div class="row">
        <div class="col">
          <router-link
            v-if="!hideBackButton"
            :to="{ name: 'main-page' }"
            tag="button"
            class="btn btn-outline-primary btn--submit back-btn float-left mt-3"
          >
            <img
              class="arrow"
              src="static/back-arrow.svg"
            >Back
          </router-link>
        </div>
        <div class="col">
          <button
            v-if="!hideButton"
            type="button"
            class="btn btn-outline-primary btn--submit float-right mt-3"
            @click="submitForm"
          >
            Next<img
              class="arrow"
              src="static/next-arrow.svg"
            >
          </button>
        </div>
      </div>
    </footer>
  </div>
</template>
<script>
import oauth from '../services/oauth'
import dayjs from 'dayjs'
import cheerio from 'cheerio'
import axios from 'axios'
import path from 'path'
import _ from 'lodash'
import fs from 'fs'
import Fuse from 'fuse.js'
const log = require('electron-log')
const countryJsonData = JSON.parse(fs.readFileSync(path.join(__static, '/countrydata.json'), 'utf-8'))
const fuseOption = {
  shouldSort: true,
  matchAllTokens: true,
  includeScore: true,
  threshold: 0.6,
  location: 0,
  distance: 100,
  maxPatternLength: 32,
  minMatchCharLength: 1,
  keys: [
    'name',
    'currencies',
    'altSpellings',
    'nativeName'
  ]
}

var fuse = new Fuse(countryJsonData, fuseOption)

export default {
  data () {
    return {
      form: 'LOGIN_FORM',
      awaiting_code: false,
      approval_code: null,
      filter_option: null,
      loading: false,
      downloadingMessage: null,
      totalAmount: [],
      pagination: false,
      perPage: 1,
      rates: [],
      invoiceCount: 0,
      navigation: true,
      progress: ''
    }
  },
  computed: {
    hideBackButton () {
      if (this.form === 'INVOICE_COUNT') {
        return true
      }
      if (this.form === 'FILTER_OPTION') {
        return true
      }
      if (this.form === 'ERROR') {
        return true
      }
      if (this.form === 'DOWNLOADED') {
        return true
      }
      return false
    },
    hideButton () {
      if (this.form === 'LOGIN_FORM') {
        return true
      }
      if (this.form === 'INVOICE_COUNT') {
        return true
      }
      if (this.form === 'DOWNLOADED') {
        return true
      }
      if (this.form === 'ERROR') {
        return true
      }
      return false
    }
  },
  methods: {
    startAgain () {
      this.form = 'LOGIN_FORM'
      this.filter_option = null
      this.loading = false
      this.downloadingMessage = null
      this.totalAmount = []
      this.pagination = false
      this.perPage = 1
      this.rates = []
      this.invoiceCount = 0
      this.navigation = true
      this.progress = ''
      this.awaiting_code = false
      this.approval_code = null
    },
    downloadMessage (count) {
      if (count > 76) {
        this.downloadingMessage = `Wow this could take a while! Let Ride Receipts do its thing and we'll let you know once your ${count} trips are in order.`
      } else if (count > 56 && count <= 76) {
        this.downloadingMessage = `Whoa ${count} trips! Put your feet up and relax. This will take a while, my friend.`
      } else if (count > 46 && count <= 56) {
        this.downloadingMessage = `You have ${count} trips. Give the little robot some time to download and organize them for you.`
      } else if (count > 36 && count <= 46) {
        this.downloadingMessage = `Whoa someone's been busy! You have ${count} trips. Downloading now.`
      } else if (count > 26 && count <= 36) {
        this.downloadingMessage = `You have ${count} trips. Downloading and organizing them for you now. Sweet deal, huh?`
      } else if (count > 11 && count <= 26) {
        this.downloadingMessage = `You have ${count} trips. Pour yourself a drink and relax. We got this.`
      } else if (count > 6 && count <= 11) {
        this.downloadingMessage = `You have ${count} trips! This should download fairly quickly.`
      } else if (count > 1 && count <= 6) {
        this.downloadingMessage = `Running this app for just ${count} trips?\nThat's okay, we won't judge ;)`
      } else if (count === 1) {
        this.downloadingMessage = `Running this app for just ${count} trip?\nThat's okay, we won't judge ;)`
      } else {
        this.form = 'DOWNLOADED'
        this.downloadingMessage = 'You have 0 trips within the time frame you selected.'
      }
    },
    async fetchGoogleToken () {
      const token = await oauth.fetchToken('google', this.approval_code)
      if (token) {
        const profile = await oauth.fetchGoogleProfile('google', token.access_token)
        this.loading = false
        localStorage.setItem('provider', 'google')
        localStorage.setItem('token_data', JSON.stringify(token))
        localStorage.setItem('user_data', JSON.stringify(profile))
        this.form = 'FILTER_OPTION'
      }
    },
    signInWithgoogle (provider) {
      const authUrl = oauth.buildAuthUrl(provider)
      this.$electron.shell.openExternal(authUrl)
      this.awaiting_code = true
    },
    async submitForm () {
      let startDate, endDate

      const user = JSON.parse(localStorage.getItem('user_data'))
      let messages
      const self = this

      if (this.filter_option === 'currentyear') {
        startDate = dayjs().startOf('year').unix()
        endDate = dayjs().endOf('year').unix()
      } else if (this.filter_option === 'previousyear') {
        startDate = dayjs().subtract(1, 'years').startOf('year').unix()
        endDate = dayjs().subtract(1, 'years').endOf('year').unix()
      } else if (this.filter_option === 'lastmonth') {
        startDate = dayjs().subtract(1, 'month').startOf('month').unix()
        endDate = dayjs().startOf('month').unix()
      } else if (this.filter_option === 'lastthreemonths') {
        startDate = dayjs().subtract(3, 'month').add(1, 'day').startOf('month').unix()
        endDate = dayjs().startOf('month').unix()
      } else {
        return
      }

      const emails = []
      let nextToken = null

      do {
        let apiUrl
        if (nextToken) {
          apiUrl = `https://www.googleapis.com/gmail/v1/users/me/messages?pageToken=${nextToken}&q='{from:"Uber Receipts" from:"Reçu Uber" from:"Uber 收据" from:"Uber Fahrtinfo" from:"Recibos da Uber" from:"Recibos Uber" from:Recibos} after:${startDate} before:${endDate}'`
        } else {
          apiUrl = `https://www.googleapis.com/gmail/v1/users/me/messages?q='{from:"Uber Receipts" from:"Reçu Uber" from:"Uber 收据" from:"Uber Fahrtinfo" from: "Recibos da Uber" from: "Recibos Uber" from:Recibos} after:${startDate} before:${endDate}'`
        }
        const list = await axios.get(encodeURI(apiUrl), {
          headers: {
            Authorization: `Bearer ${JSON.parse(localStorage.getItem('token_data')).access_token}`
          }
        })

        if (typeof list.data.messages !== 'undefined' && list.data.messages.length > 0) {
          for (let i = 0; i < list.data.messages.length; i++) {
            emails.push(list.data.messages[i])
          }
        }

        if (typeof list.data.nextPageToken !== 'undefined') {
          nextToken = list.data.nextPageToken
        } else {
          nextToken = null
        }
      } while (nextToken !== null)

      if (emails.length === 0) {
        this.invoiceCount = 0
        this.downloadMessage(0)
        self.form = 'DOWNLOADED'
      } else {
        this.downloadMessage(emails.length)
        messages = emails
        this.invoiceCount = messages.length

        if (messages.length > 0) {
          this.form = 'INVOICE_COUNT'
        }

        if (typeof messages !== 'undefined') {
          for (let i = 0; i < messages.length; i++) {
            const data = await axios.get(`https://www.googleapis.com/gmail/v1/users/me/messages/${messages[i].id}`, {
              headers: {
                Authorization: `Bearer ${JSON.parse(localStorage.getItem('token_data')).access_token}`
              }
            })
            const processed = self.processEmails(data.data, user)
            if (processed) {
              const number = i + 1
              self.progress = _.ceil(_.divide(number, messages.length) * 100)
            }
            if (self.progress === 100) {
              self.form = 'DOWNLOADED'
              const notification = new Notification('Ride Receipts', {
                body: 'Success! All receipts have been downloaded for you.'
              })
              notification.onclick = () => {
                console.log('Notification clicked')
              }
            }
          }
        }
      }
    },
    async processEmails (data, user) {
      var html
      var currency
      var totalRate
      var address
      var amount
      var check
      var uberEats

      if (data.payload.parts) {
        const payload = _.find(data.payload.parts, { mimeType: 'multipart/alternative' })
        if (payload) {
          const htmlData = _.find(payload.parts, { mimeType: 'text/html' })
          if (htmlData) {
            html = Buffer.from(htmlData.body.data, 'base64')
          }
        } else if (data.payload.parts) {
          html = Buffer.from(data.payload.parts[0].body.data, 'base64')
        } else {
          html = Buffer.from(data.payload.body.data, 'base64')
        }
      }
      const from = _.find(data.payload.headers, { name: 'From' }).value
      const countryFromEmail = from.match(/<(\w+.\w+@\w+.\w+)>/g)[0].replace(/[<>]/g, '').replace('uber.', '').replace('@uber.com', '')
      const subject = _.find(data.payload.headers, { name: 'Subject' }).value
      const cancelled = subject.includes('canceled')
      if (cancelled) {
        log.info('Trip is cancelled')
      }
      const date = new Date(parseInt(data.internalDate))
      const dom = cheerio.load(html.toString(), {
        normalizeWhitespace: true
      })

      try {
        const newUberEats = dom('body > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table:nth-child(2) > tbody > tr > td > table > tbody > tr > td > div > div > table > tbody > tr > td:nth-child(2) > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td:nth-child(2) > table > tbody > tr > td > table > tbody > tr > td > table.t7of12.layout > tbody > tr > td > table > tbody > tr:nth-child(1) > td').text()
        const oldUberEats = dom('#bgwrapper > tbody > tr > td > table > tbody > tr > td > table:nth-child(4) > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table:nth-child(1) > tbody > tr > td:nth-child(1) > table:nth-child(3) > tbody > tr > td').text()

        if (dom('.eventinfo').length > 0) {
          if (_.trim(dom('.topPrice.tal.black').text()).match(/[+-]?\d+(\.\d+)?/) !== null) {
            amount = _.trim(dom('.topPrice.tal.black').text())
          } else {
            amount = _.trim(dom('td.chargedFare').text())
          }
          dom('.rideTime').remove()
          var firstAddress = _.words(_.trim(dom('.firstAddress').text()))
          address = firstAddress.indexOf('accepted') < 0 ? _.trim(dom('.firstAddress').text()) : _.trim(dom('#cancelledABlock > tbody > tr:nth-child(2) > td > table > tbody > tr > td.tripInfoDescription.gray.vam.tal').text())
        } else {
          amount = _.trim(dom('body > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table:nth-child(1) > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td:nth-child(2) > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td:nth-child(2) > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td:nth-child(2) > div > span').text())
          if (newUberEats.includes('ordering')) {
            address = _.trim(dom('body > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table:nth-child(4) > tbody > tr:nth-child(2) > td > table > tbody > tr > td > table.t11of12 > tbody > tr > td > table > tbody > tr:nth-child(3) > td > table:nth-child(1) > tbody > tr > td > table:nth-child(2) > tbody > tr > td > table > tbody > tr:nth-child(2) > td').text())
          } else {
            address = _.trim(dom('body > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table > tbody > tr > td > table:nth-child(5) > tbody > tr:nth-child(1) > td > table > tbody > tr > td > table.t11of12 > tbody > tr > td > table > tbody > tr > td > table.t5of12 > tbody > tr > td > table > tbody > tr > td > table:nth-child(1) > tbody > tr > td.Uber18_text_p2.black > table > tbody > tr:nth-child(2) > td').text())
          }
        }
        if (countryFromEmail) {
          var geoData = _.minBy(fuse.search(countryFromEmail), 'score')
        } else {
          geoData = _.minBy(fuse.search(amount.replace(/\d+([,.]\d+)?/, '').replace(/[^\w+\s]/, '')), 'score')
        }
        if (amount) {
          currency = geoData.item.currencies[0].code
          totalRate = parseFloat(amount.match(/[+-]?\d+(\.\d+)?/)[0])
          check = _.findIndex(this.rates, ['currency', currency])
          if (check < 0) {
            this.rates.push({
              currency: currency,
              amount: [totalRate]
            })
          } else {
            this.rates[check].amount.push(totalRate)
          }
        }

        if (newUberEats.includes('ordering') || oldUberEats.includes('Uber Eats')) {
          uberEats = 'UberEats'
        } else {
          uberEats = 'Uber'
        }
      } catch (e) {
        log.info(address)
        log.info(e)
        console.log(e)
      }

      if (this.rates.length === 1) {
        this.navigation = false
      } else {
        this.navigation = true
      }

      this.$electron.ipcRenderer.send('downloadPDF', {
        email: user.email,
        date: date,
        year: dayjs(date).format('YYYY'),
        cancelled: cancelled,
        invoiceDate: dayjs(date).format('MMMM-DD-YYYY_hh-mm-a'),
        html: `data:text/html;charset=UTF-8,${encodeURIComponent(html)}`,
        rideType: uberEats
      })
      setTimeout(() => { console.log('Resting.....') }, 2000)
    },
    openInvoiceFolder () {
      const documentDir = this.$electronstore.get('invoicePath')
      this.$electron.shell.openItem(documentDir)
    }
  }
}
</script>
