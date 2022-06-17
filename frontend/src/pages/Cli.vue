<template>
  <q-page class="column items-center bg-black">
    <div
      v-if="!connected"
      class="column flex-center q-my-xl"
    >
      <q-spinner
        color="primary"
        size="3em"
        class="q-mb-md"
      ></q-spinner>
      <p class="text-white">Waiting for Flipper...</p>
    </div>
    <div v-if="connected && !flags.rpcActive" class="full-width" style="height: calc(100vh - 50px)">
      <div id="terminal-container" class="fit bg-black q-pl-sm"></div>
      <q-btn
        @click="flags.sharePopup = true"
        outline
        color="white"
        class="absolute-top-right q-ma-sm z-top"
      >
        {{ flags.serverActive ? 'Session live' : 'Share session' }}
        <q-badge
          v-if="flags.serverActive"
          :label="connections > 0 ? connections.toString() : ''"
          rounded
          color="green"
          class="q-ml-md"
        />
      </q-btn>
    </div>
    <q-dialog v-model="flags.sharePopup">
      <q-card>
        <q-card-section class="row items-center q-pb-none">
          <div class="text-h6 q-mr-xl">CLI session sharing</div>
          <q-space />
          <q-btn icon="close" flat round dense v-close-popup />
        </q-card-section>
        <q-card-section>
          <template v-if="flags.serverActive">
            <p>
              Address:
              <pre class="bg-grey-4 q-pa-xs rounded-borders">{{ address }}</pre>
              <a :href="'/remote-cli#' + address">Share this link with your friend</a>
            </p>

            <p>Peers connected: {{ connections }}</p>
          </template>
        </q-card-section>

        <q-card-actions align="right">
          <q-btn
            flat
            :loading="flags.serverToggling"
            :color="flags.serverActive ? 'negative' : 'black'"
            @click="flags.serverActive ? stopServer() : startServer()"
          >
            {{ flags.serverActive ? 'Stop server' : 'Start server' }}
          </q-btn>
        </q-card-actions>
      </q-card>
    </q-dialog>
  </q-page>
</template>

<script>
import { defineComponent, ref } from 'vue'
import { Terminal } from 'xterm'
import 'xterm/css/xterm.css'
import { FitAddon } from 'xterm-addon-fit'

export default defineComponent({
  name: 'PageCli',

  props: {
    flipper: Object,
    connected: Boolean,
    rpcActive: Boolean
  },

  setup () {
    return {
      flags: ref({
        rpcActive: false,
        rpcToggling: false,
        serverActive: false,
        serverToggling: false,
        sharePopup: false
      }),
      terminal: ref(undefined),
      readInterval: undefined,
      input: ref(''),
      unbind: ref(undefined),
      server: ref(null),
      connections: ref(0),
      address: ref('')
    }
  },

  methods: {
    init () {
      this.terminal = new Terminal({
        scrollback: 10_000
      })
      const fitAddon = new FitAddon()
      this.terminal.loadAddon(fitAddon)
      this.terminal.open(document.getElementById('terminal-container'))
      document.querySelector('.xterm').setAttribute('style', 'height:' + getComputedStyle(document.querySelector('.xterm')).height)
      this.terminal.focus()
      fitAddon.fit()

      this.write('\x01')
      this.read()

      this.terminal.onData(async data => {
        this.write(data)
      })
    },

    write (data) {
      this.flipper.write('cli', data)
    },

    read () {
      this.flipper.read('cli')
    },

    async stopRpc () {
      this.flags.rpcToggling = true
      await this.flipper.commands.stopRpcSession()
      this.flags.rpcActive = false
      this.flags.rpcToggling = false
      this.$emit('setRpcStatus', false)
    },

    startServer () {
      this.flags.serverToggling = true
      this.server = new window.Bugout({ seed: localStorage.getItem('bugout-seed') })
      localStorage.setItem('bugout-seed', this.server.seed)
      this.address = this.server.address()

      this.server.on('connections', c => {
        this.connections = c
      })

      // this.server.on('message', function (address, msg) { console.log('message:', address, msg) })

      this.flags.serverToggling = false
      this.flags.serverActive = true
    },

    stopServer () {
      this.flags.serverToggling = true
      this.server.close()
      this.server = null
      this.connections = 0
      this.address = ''
      this.flags.serverToggling = false
      this.flags.serverActive = false
      this.flags.sharePopup = false
    },

    async start () {
      this.flags.rpcActive = this.rpcActive
      if (this.rpcActive) {
        await this.stopRpc()
      }
      setTimeout(this.init, 500)

      let isUnicode = false,
        unicodeBytesLeft = 0,
        unicodeBuffer = []

      this.unbind = this.flipper.emitter.on('cli output', data => {
        if (data.byteLength === 1) {
          const byte = data[0]
          if (!isUnicode && byte >> 7 === 1) {
            isUnicode = true
            data = undefined
            unicodeBuffer.push(byte)
            for (let i = 6; i >= 4; i--) {
              if ((byte >> i) % 2 === 1) {
                unicodeBytesLeft++
              } else {
                break
              }
            }
          } else {
            if (unicodeBytesLeft > 0 && byte >> 6 === 2) {
              unicodeBuffer.push(byte)
              unicodeBytesLeft--
              if (unicodeBytesLeft === 0) {
                data = new Uint8Array(unicodeBuffer)
                isUnicode = false
                unicodeBuffer = []
              } else {
                data = undefined
              }
            } else {
              isUnicode = false
              unicodeBytesLeft = 0
              unicodeBuffer = []
            }
          }
        }
        if (data) {
          const text = new TextDecoder().decode(data).replaceAll('\x7F', '')
          if (this.flags.serverActive) {
            this.server.send({ data: text })
          }
          this.terminal.write(text)
        }
      })
    }
  },

  mounted () {
    if (this.connected) {
      setTimeout(this.start, 500)
    }
  },

  async beforeUnmount () {
    this.stopServer()
    this.unbind()
  }
})
</script>
