<template>
  <LeafletSingleComponent
    :geo="geo"
    :relay="relay"
    :result="result"
  />

  <div id="wrapper">

    <HeaderComponent :relays="relays" />

    <div id="relay-wrapper">

      <row container :gutter="12">
        <column :xs="12" :md="12" :lg="12" class="title-card">
          <span v-tooltip:top.tooltip="'Click to copy'" style="display:block">
            <h2 @click="copy(relayUrl())">{{ relayUrl() }}</h2>
          </span>
        </column>
      </row>
      

      <row container :gutter="12">
        <column :xs="12" :md="12" :lg="12" class="title-card">
          <span  class="badges">
            <span><img :src="badgeCheck('connect')" /></span>
            <span><img :src="badgeCheck('read')" /></span>
            <span><img :src="badgeCheck('write')" /></span>
          </span>

          <span v-if="result.info?.supported_nips" class="badges">
              <span v-for="(nip) in result.info.supported_nips" :key="`${relay}_${nip}`">
                  <a :href="nipLink(nip)" target="_blank"><img :src="badgeLink(nip)" /></a>
              </span>
            </span>
        </column>
      </row>

      <row container :gutter="12" v-if="!result?.check?.connect">
        <column :xs="12" :md="12" :lg="12" class="title-card">
          This relay appears to be offline.
        </column>
      </row>

      <row container :gutter="12" v-if="result?.check?.connect">
        <column :xs="12" :md="12" :lg="12" class="title-card">
            <table v-if="result.info">
              <tr>
                <th colspan="2"><h4>Info</h4></th>
              </tr>
              <tbody v-if="result.info">
                <tr v-for="(value, key) in Object.entries(result.info).filter(value => value[0] != 'id' && value[0] != 'supported_nips')" :key="`${value}_${key}`">
                    <td>{{ value[0] }}</td>
                    <td v-if="value[0]!='contact' && value[0]!='pubkey' && value[0]!='software' && value[0]!='version'">{{ value[1] }} </td>
                    <td v-if="value[0]=='contact'"><SafeMail :email="value[1]" /></td>
                    <td v-if="value[0]=='pubkey' || value[0]=='version'"><code>{{ value[1] }}</code></td>
                    <td v-if="value[0]=='software'"><a :href="value[1]">{{ value[1] }}</a></td>
                </tr>
              </tbody>
              <tr v-if="Object.entries(result.info).length == 0 && result.check.connect">
                Relay does not have NIP-11 support, or the administrator has not configured the relay to return information.
              </tr>
            </table>


            <table v-if="result.identities">
              <tr>
                <th colspan="2"><h4>Identities</h4></th>
              </tr>
              <tbody v-if="result.identities">
                <tr v-for="(value, key) in Object.entries(result?.identities)" :key="`${value}_${key}`">
                  <td>{{ value[0] }}</td>
                  <td><code>{{ value[1] }}</code></td>
                </tr>
                <tr v-if="Object.entries(result.identities).length==0">
                  Relay does not provide NIP-05 support and has not registered an administrator key.
                </tr>
              </tbody>
            </table>

          </column>
          <column :xs="12" :md="6" :lg="6" class="title-card">
            <table v-if="geo[relay]">
              <tr>
                <th colspan="2"><h4>GEO {{geo?.countryCode ? getFlag() : ''}}</h4></th>
              </tr>
              <tbody v-if="geo[relay]">
                <tr v-for="(value, key) in Object.entries(geo[relay])" :key="`${value}_${key}`">
                  <td>{{ value[0] }}</td>
                  <td>{{ value[1] }} </td>
                </tr>
              </tbody>
            </table>
          </column>
          <column :xs="12" :md="6" :lg="6" class="title-card">
            <table v-if="geo[relay]">
              <tr>
                <th colspan="2"><h4>DNS</h4></th>
              </tr>
              <tbody v-if="geo[relay]">
                <tr v-for="(value, key) in Object.entries(geo[relay].dns)" :key="`${value}_${key}`">
                <td>{{ value[0] }}</td>
                <td>{{ value[1] }} </td>
              </tr>
              </tbody>
            </table>
          </column> 
      </row>
      

     <!--  <RefreshComponent 
          :relay="relay"
        /> -->
    </div>
    

  </div>
</template>

<script>

import { defineComponent} from 'vue'
import { useStorage } from "vue3-storage";

import LeafletSingleComponent from '../components/LeafletSingleComponent.vue'
import HeaderComponent from '../components/HeaderComponent.vue'
// import NavComponent from '../components/NavComponent.vue'
// import RefreshComponent from '../components/RefreshComponent.vue'

import { Row, Column } from 'vue-grid-responsive';
import SafeMail from "@2alheure/vue-safe-mail";

import RelaysLib from '../lib/relays-lib.js'

import { version } from '../../package.json'
import { relays } from '../../relays.yaml'
import { geo } from '../../cache/geo.yaml'

const localMethods = {
    relayUrl() {
      // We will see what `params` is shortly
      return `wss://${this.$route.params.relayUrl}`
    },

    badgeLink(nip){
      return `https://img.shields.io/static/v1?style=for-the-badge&label=NIP&message=${this.nipSignature(nip)}&color=black`
    },

    badgeCheck(which){
      return `https://img.shields.io/static/v1?style=for-the-badge&label=&message=${which}&color=${this.result?.check?.[which] ? 'green' : 'red'}`
    },

    nipSignature(key){
      return key.toString().length == 1 ? `0${key}` : key
    },

    nipFormatted(key){
      return `NIP-${this.nipSignature(key)}`
    },

    nipLink(key){
      return `https://github.com/nostr-protocol/nips/blob/master/${this.nipSignature(key)}.md`
    },

    async copy(text) {
      try {
        await navigator.clipboard.writeText(text);
      } catch($e) {
        //console.log('Cannot copy');
      }
    },
}

export default defineComponent({
  title: "nostr.watch registry & network status",
  name: 'SingleRelay',
  
  components: {
    Row,
    Column,
    LeafletSingleComponent,
    // NavComponent,
    SafeMail,
    HeaderComponent,
    // RefreshComponent,
  },

  data() {
    return {
      relays,
      result: {},
      messages: {},
      nips: {},
      alerts: {},
      timeouts: {},
      intervals: {},
      lastPing: Date.now(),
      nextPing: Date.now() + (60*1000),
      count: 0,
      geo,
      relay: "",
      version: version,
      storage: null,
      lastUpdate: null,
      cacheExpiration: 10*60*1000 //10 minutes
    }
  },

  async mounted() {
    this.relay = this.relayUrl()

    this.storage = useStorage()
    this.lastUpdate = this.storage.getStorageSync('lastUpdate')
    this.preferences = this.storage.getStorageSync('preferences')
    this.result = this.storage.getStorageSync(this.relay)
    
    if(this.isExpired())
      this.check(this.relay)
  },

  computed: {},

  // updated() {
  //    Object.keys(this.timeouts).forEach(timeout => clearTimeout(this.timeouts[timeout]))
  //    Object.keys(this.intervals).forEach(interval => clearInterval(this.intervals[interval]))
  // },

  methods: Object.assign(localMethods, RelaysLib),

})
</script>
<style scoped>
ul, ul li { padding:0; margin:0; list-style:none; }
td { padding:5px 10px; }
th h4 { text-align:center; padding:5px 10px; margin:0 0 6px; background:#f0f0f0; }
table {margin:20px 10px 20px; border: 2px solid #f5f5f5; padding:20px}
tr td:first-child { text-align:right }
tr td:last-child { text-align:left }
.indicator { display: table-cell; width:33% ; font-weight:bold; text-align: center !important; color: white; text-transform: uppercase; font-size:0.8em}
body, .grid-column { padding:0; margin:0; }
.badges { display:block; margin: 10px 0 11px}
.badges > span {margin-right:5px}
#wrapper {max-width:800px}


#relay-wrapper { margin: 50px 0 20px; padding: 20px 0}
h2 {cursor:pointer;font-size:40pt; margin: 0px 0 15px; padding:0 0 10px; border-bottom:3px solid #e9e9e9}
</style>
