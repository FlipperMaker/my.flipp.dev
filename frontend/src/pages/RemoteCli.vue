<template>
  <q-page class="column items-center bg-black">
    <div class="full-width" style="height: calc(100vh - 50px)">
      <div id="terminal-container" class="fit bg-black q-pl-sm"></div>
    </div>
  </q-page>
</template>

<script>
import { defineComponent, ref } from 'vue'
import { Terminal } from 'xterm'
import 'xterm/css/xterm.css'
import { FitAddon } from 'xterm-addon-fit'

export default defineComponent({
  name: 'PageRemoteCli',

  setup () {
    return {
      terminal: ref(undefined),
      readInterval: undefined,
      input: ref(''),
      unbind: ref(undefined)
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

      const address = window.location.hash.substr(1)
      if (!address) {
        console.error('No host address found in URL')
      }
      const b = new window.Bugout(window.location.hash.substr(1))
      console.log('My address is ' + b.address())
      console.log('Connecting to the server...\n(this can take a minute)')

      b.on('server', () => {
        console.log('Connected to the server.')
      })

      b.on('message', (address, msg) => {
        this.terminal.write(msg.data)
      })
    }
  },

  mounted () {
    this.init()
  }
})
</script>
