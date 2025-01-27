<template lang="pug">
#app.h-full
  nav.fixed.inset-x-0.top-0.z-10(v-if="false")
    .container.mx-auto.p-2.flex.items-center.text-gray-100.text-sm(
      class="lg:p-4"
    )
      .grow
      a.text-gray-800.inline-flex.px-2.py-2(
        href="https://replicate.com/"
        class="hover:text-blue-600"
        target="_new"
      ) Replicate
  .text-3xl.font-medium.text-gray-800.text-center.mt-10.mb-6(
    v-if="false"
    class="lg:text-4xl lg:mt-20 lg:mb-12"
  ) Webcam

  main.container.mx-auto.p-2(
    class="lg:p-4"
  )
    webcam(@data="onData")
    select(v-model="narrator")
      option(
        v-for="(item, index) in narrators"
        :key="`narrator-${index}`"
        :value="item.value"
      ) {{ item.text }}

    .text-3xl.font-medium.text-gray-800.text-center {{ state_text[state] }}
</template>

<script>
import rwp from 'replicate-webhook-proxy'

const makeid = (length) => {
  let result = ''
  const characters =
    'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789'
  const charactersLength = characters.length
  let counter = 0
  while (counter < length) {
    result += characters.charAt(Math.floor(Math.random() * charactersLength))
    counter += 1
  }
  return result
}

const TICK_INTERVAL = 1000
const ws_id = makeid(10)
const client = rwp(ws_id)

export default {
  name: 'App',
  data: () => ({
    interval: null,

    /**
     * state:
     *  0 = ready
     *  1 = submitted image to create text (waiting)
     *  2 = submitted text to create audio (waiting)
     *  3 = audio is playing (wait until audio done)
     */
    state: 0,
    state_text: {
      0: '👀 watching',
      1: '👀 watching',
      2: '🧠 thinking...',
      3: '🎤 speaking'
    },
    // Used to countdown before resetting state when audio is playing
    countdown: 3,

    image: null,

    narrator: 'david',
    narrators: [
      { text: 'David Attenborough', value: 'david' },
      { text: 'Morgan Freeman', value: 'morgan' }
    ]
  }),
  methods: {
    // Tick is responsible for firing new observations if we're stuck/ready
    tick() {
      console.log('--- log: tick, state = ', this.state)
      if (this.state === 0) {
        this.countdown -= 1
        if (this.countdown <= 0) this.submit()
      }
      if (this.state === 3) {
        this.countdown -= 1
        if (this.countdown <= 0) {
          this.state = 0
          this.countdown = 3
        }
      }
    },
    onData(image) {
      this.image = image
    },
    async submit() {
      try {
        if (!this.image) return

        this.state = 1
        await fetch('/api/text', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json'
          },
          body: JSON.stringify({
            narrator: this.narrator,
            image: this.image,
            ws_id
          })
        })
      } catch (e) {
        console.log('--- log: failed to call API:', e)
        this.state = 0
      }
    }
  },
  mounted() {
    this.interval = setInterval(() => this.tick(), TICK_INTERVAL)
    client.on('message', async (event) => {
      try {
        const { type, narrator } = event.data.query

        // We got a text response, send it further to synthesize an audio response
        if (type === 'text') {
          this.state = 2

          const text = event.data.body.output.join('')
          console.log(
            `--- log: got text response with narrator = ${narrator}: `,
            text
          )

          await fetch('/api/audio', {
            method: 'POST',
            headers: {
              'Content-Type': 'application/json'
            },
            body: JSON.stringify({
              narrator,
              text: text,
              ws_id
            })
          })
        }

        // We got the final audio, play it
        if (type === 'audio') {
          this.state = 3
          this.countdown = 30

          const { output } = event.data.body
          console.log(
            `--- log: got audio response with narrator = ${narrator}: `,
            output
          )

          const audio = new Audio(output)
          audio.play()
        }
      } catch (e) {
        console.log('--- log: failed to parse webhook:', e)
        this.state = 0
      }
    })
  },
  beforeUnmount() {
    if (this.interval) clearInterval(this.interval)
  }
}
</script>

<style lang="stylus" scoped>
#app
  nav
    background-color rgba(240, 240, 240, .8)
    -webkit-backdrop-filter saturate(180%) blur(20px)
    backdrop-filter saturate(180%) blur(20px)

    .router-link-active
      border-bottom 3px solid #92400e
      color #92400e

  select
    width 100%
    margin 10px 0
    padding 15px
    background #444
    color #fff
    display block
    border-radius 10px
</style>
