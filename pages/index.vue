<template>
  <div ref="page" class="page__container">
    <div class="relative">
      <canvas ref="canvas"></canvas>
      <div class="absolute top-0">
        <audio ref="audio_player" src="/chicago.ogg" />
      </div>
      <div class="buttons__container">
        <button ref="key_0" v-touch:start="() => { keyDown(0) }" v-touch:end="() => { keyUp(0) }">D</button>
        <button ref="key_1" v-touch:start="() => { keyDown(1) }" v-touch:end="() => { keyUp(1) }">F</button>
        <button ref="key_2" v-touch:start="() => { keyDown(2) }" v-touch:end="() => { keyUp(2) }">J</button>
        <button ref="key_3" v-touch:start="() => { keyDown(3) }" v-touch:end="() => { keyUp(3) }">K</button>
      </div>
      <div class="row" style="position: absolute; bottom: 0; width: 100%; height: 100px; display: flex; align-items: center; text-align: center; font-size: 2em;">
        <div class="col "><p>{{this.score_key_0}}</p></div>
        <div class="col "><p>{{this.score_key_1}}</p></div>
        <div class="col "><p>{{this.score_key_2}}</p></div>
        <div class="col "><p>{{this.score_key_3}}</p></div>
      </div>
    </div>
    <div style="display:none">
      <ShareNetwork
          id='twitter-d'
          network="twitter"
          url="https://www.piano-king.com"
          title="I just scored 97% on the piano tiles game of Sofiane Pamart. Play on:"
          quote="The hot reload is so fast it's near instant. - Evan You"
          hashtags="vuejs,vite"
        >
          Share on Twitter
      </ShareNetwork>
    </div>
    <div style="display:none">
      <ShareNetwork
          id="facebook-d"
          network="facebook"
          url="https://www.piano-king.com"
          title="I just scored 97% on the piano tiles game of Sofiane Pamart. Play on:"
          quote="The hot reload is so fast it's near instant. - Evan You"
          hashtags="vuejs,vite"
        >
          Share on Facebook
      </ShareNetwork>
    </div>
      <div id="perfect" class='gradient-pink' style="position: absolute; top: 30px; right: 30px; font-size: 40px; text-align: right; color: #E6ABBD;">
        <p class='gradient-pink'>Score: {{ score }}
        <br>Max Perfects: {{ maxPerfects }}
        <br>Perfects: {{ perfects }}
        <br>Missed: {{ missed }}</p>
      </div>
  </div>
</template>

<script >
import Swal from "sweetalert2";
import sheet from "~/assets/json/chicago.json";
import VueSocialSharing from 'vue-social-sharing';
import Vue from 'vue';
import html2canvas from "html2canvas";

let timerInterval;
document.click_social_media = function(css_selector) {
  document.getElementById(css_selector).click()
}

document.downloadscreenshot = function() {
  html2canvas(document.getElementsByClassName("swal2-popup swal2-modal swal2-show")[0], { useCORS: true, logging: true, scrollX: 0, scrollY: 0 }).then(function(canvas) {
    var a = document.createElement('a');
    a.href = canvas.toDataURL("image/jpeg").replace("image/jpeg", "image/octet-stream");
    a.download = 'king_tiles.jpg';
    a.click();
  });
}

Vue.use(VueSocialSharing);

export default {
  data: () => ({
    canvas_height: undefined,
    canvas_width: undefined,
    ctx: undefined,
    number_max: sheet.length,
    score: 100,
    perfects: 0,
    maxPerfects: 0,
    score_key_0: "",
    score_key_1: "",
    score_key_2: "",
    score_key_3: "",
    nextTileToType: 0,
    missed: 0,
    score: 0,

    curr_timestamp: undefined,
    prev_timestamp: undefined,
    start_timestamp: undefined,

    is_playing: false,
    tile_list: [],
    scroll_speed: 3000, // 3000ms for a tile to cross the entire screen
  }),

  computed :{
    percentage: function() {
       return ( this.score / ( this.number_max * 4 ) * 100 ).toFixed(2)
    }
  },

  mounted() {
    // Fit canvas to page dimensions
    const page = this.$refs.page;
    const canvas = this.$refs.canvas;
    canvas.height = page.clientHeight;
    canvas.width = page.clientWidth;
    this.canvas_height = canvas.height;
    this.canvas_width = canvas.width;
    this.ctx = canvas.getContext("2d");

    this.StartGame()
    // Add keyboard event listener
    document.addEventListener("keydown", (event) => {
      let key_id = event.key == "d" ? 0 : event.key == "f" ? 1 : event.key == "j" ? 2 : event.key == "k" ? 3 : null
      if (key_id === null) return
      this.keyDown(key_id)
    });
    document.addEventListener("keyup", (event) => {
      let key_id = event.key == "d" ? 0 : event.key == "f" ? 1 : event.key == "j" ? 2 : event.key == "k" ? 3 : null;
      if (key_id === null) return
      this.keyUp(key_id)
    });

    // Pre-draw keys line separators
    this.ctx.beginPath();
    this.ctx.strokeStyle = "#0007";
    this.ctx.moveTo(this.canvas_width * 0.25, 0);
    this.ctx.lineTo(this.canvas_width * 0.25, this.canvas_height);
    this.ctx.moveTo(this.canvas_width * 0.5, 0);
    this.ctx.lineTo(this.canvas_width * 0.5, this.canvas_height);
    this.ctx.moveTo(this.canvas_width * 0.75, 0);
    this.ctx.lineTo(this.canvas_width * 0.75, this.canvas_height);
    this.ctx.stroke();
  },

  methods: {
    draw(timestamp) {
      if (!this.start_timestamp) this.start_timestamp = timestamp;
      this.curr_timestamp = timestamp - this.start_timestamp;
      // Clear canvas
      this.ctx.clearRect(0, 0, this.canvas_width, this.canvas_height);
      // Keys line separators
      this.ctx.beginPath();
      this.ctx.strokeStyle = "#0007";
      this.ctx.moveTo(this.canvas_width * 0.25, 0);
      this.ctx.lineTo(this.canvas_width * 0.25, this.canvas_height);
      this.ctx.moveTo(this.canvas_width * 0.5, 0);
      this.ctx.lineTo(this.canvas_width * 0.5, this.canvas_height);
      this.ctx.moveTo(this.canvas_width * 0.75, 0);
      this.ctx.lineTo(this.canvas_width * 0.75, this.canvas_height);
      this.ctx.stroke();

      // Calculate speed
      let px_per_ms = this.canvas_height / this.scroll_speed;

      // Tiles drawing
      sheet.forEach((tile, i) => {
        // 480: distance from bottom to button
        let start_position = -tile.time + (this.scroll_speed - 480);
        let x = this.canvas_width * 0.25 * tile.key + this.canvas_width * 0.125;
        // -30: arbitrary delay offset (-30 for mobile / +30 for desktop ??)
        let y = (start_position + this.curr_timestamp) * px_per_ms - 30;
        this.ctx.save();
        this.ctx.beginPath();
        this.ctx.strokeStyle = "black";
        this.ctx.arc(x, y, 22, 0, 2 * Math.PI);
        this.ctx.fill();
        this.ctx.restore();

      
        if (i === this.nextTileToType && y > this.canvas_height *.9 + 500 * px_per_ms) {
          this.missed++
          this.nextTileToType++
          this.displayScore("MISSED", tile.key)
        }
      });

      this.prev_timestamp = timestamp;
      requestAnimationFrame(this.draw);
    },
    EndGame() {
      Swal.fire({
        title: `<div class = 'result' ><p>You are ${this.percentage}% as skilled as the king!</p><img src="${require('/assets/img/crown.png')}<p>Buy your first NFT on <a href='https://piano-king.com'>piano-king.com</a>!</p>"></div>`,
        background: `#E6ABBD`,
        footer: `<p>Share on social media: </p><img src="${require('/assets/img/twitter.svg')}" onclick="click_social_media(\'twitter-d\')"><img src="${require('../assets/img/tiktok.svg')}" onclick="click_social_media(\'facebook-d\')"> or download : <img src="${require('../assets/img/download.svg')}" onclick="downloadscreenshot()">`,
        imageUrl: require('/assets/img/FEqSaeQWYAcu0ln.jpeg'),
        showConfirmButton: false,
        color: "#E6ABBD",
        imageWidth: 400,
        imageHeight: 400,
        imageAlt: 'Custom image',
      })
    },
    StartGame() {
      Swal.fire({
        background: `transparent`,
        confirmButtonColor: 'transparent',
        confirmButtonText: 'Play against the Piano King!',
        customClass: 'start-game gradient-pink',
        //showConfirmButton: false,
      }).then((result) => {
        Swal.fire({
          html:'<p style="font-size: 288px; font-weight: 900;"><strong></strong></p>',
          timer: 4000,
          background: `transparent`,
          confirmButtonColor: 'none',
          timerProgressBar: false,
          showConfirmButton: false,
          customClass: 'gradient-pink-big',
          didOpen: () => {
            timerInterval = setInterval(() => {
              Swal.getHtmlContainer().querySelector('strong')
                .textContent = (Swal.getTimerLeft() / 1000)
                  .toFixed(0)
            }, 100)
          },
          willClose: () => {
            clearInterval(timerInterval)
          }
        }).then((result) => {  
          this.play()
        })     
      })
    },
    play() {
      if (this.is_playing) return;

      this.$refs.audio_player.play();
      requestAnimationFrame(this.draw);
      setTimeout(() => this.EndGame(), 61062.415);
      this.is_playing = true;
    },

    keyDown(key_id) {
      this.$refs[`key_${key_id}`].classList.add("active");
      this.score = this.score + this.GetPoints(key_id, this.curr_timestamp - 20);
      // TODO: check tile
    },
    keyUp(key_id) {
      this.$refs[`key_${key_id}`].classList.remove("active");
    },
    displayScore(text, id) {
      if (text === "PERFECT") this.perfects++
      else this.perfects = 0
      this.maxPerfects = Math.max(this.perfects, this.maxPerfects)

      if (id == 0) this.score_key_0 = text;
      else if (id == 1) this.score_key_1 = text;
      else if (id == 2) this.score_key_2 = text;
      else if (id == 3) this.score_key_3 = text;
      setTimeout(() => { this.score_key_0 = "" }, 300);
      setTimeout(() => { this.score_key_1 = "" }, 300);
      setTimeout(() => { this.score_key_2 = "" }, 300);
      setTimeout(() => { this.score_key_3 = "" }, 300);

    },
    GetPoints(key_id, timestamp){
      console.log('enter', timestamp)
      for (var i = 0; i < sheet.length; i++) {
        if (sheet[i].key == key_id) {
          if ( sheet[i].time + 300 > timestamp && timestamp > sheet[i].time - 300) {
            this.displayScore("PERFECT", sheet[i].key);
            this.nextTileToType++
            return 4;
          }
          else if ( sheet[i].time + 350 > timestamp && timestamp > sheet[i].time - 350 ) {
            this.displayScore("GOOD", sheet[i].key);
            this.nextTileToType++
            return 3;
          }
          else if ( sheet[i].time + 400 > timestamp && timestamp > sheet[i].time - 400 ) {
            this.displayScore("BAD", sheet[i].key);
            console.log('BAD', sheet[i].key, sheet[i].time, timestamp);
            this.nextTileToType++
            return 2;
          }
          else if ( sheet[i].time + 450 > timestamp && timestamp > sheet[i].time - 450 ){
            this.displayScore("POOR", sheet[i].key);
            console.log('POOR', sheet[i].key, sheet[i].time, timestamp)
            this.nextTileToType++
            return 1;
          }
        }
      }
      this.missed++
      this.displayScore("MISSED", key_id)
      return 0;
    },
    getValues: function(object) {
      //use a polyfill in case Object.values is not supported by current browser
      return Object.values ? Object.values(object) : Object.keys(object).map(key => object[key]);
    }
  },
};
</script>

<style lang="postcss">
.page__container {
  @apply h-full max-w-2xl;
  @apply mx-auto;
  @apply flex flex-col justify-center;
}
canvas {
  box-shadow: 0 15px 70px rgb( 0 0 0 / 10% );
  background: rgba( 230, 171, 189, 0.69 ); {/*rgba( 255, 255, 255, 0.69 );*/}
}

.buttons__container {
  @apply absolute;
  bottom: 10%;
  @apply w-full;
  @apply flex justify-around;
}
button {
  @apply h-14 w-14;
  @apply rounded-full;
  @apply transition-all;
  background: linear-gradient( 145deg, #E6E6E6, #FFF );
  box-shadow: 0px 10px #560E36;

  @media (min-width: 1024px) {
    @apply h-20 w-20;
    background: linear-gradient( 145deg, #DDD, #FFF );
    box-shadow: 0px 10px #560E36;
    font-size: 3em;
    font-weight: 900;
  }
}
button.active {
  background: linear-gradient( 145deg, #CACACA, #F0F0F0 );
  box-shadow: 0px 0 #D1859F;
  transform: translateY(5px);

  @media (min-width: 1024px) {
    box-shadow: 0px 0 #D1859F;
  }
}
.row
{
  display: flex; /* equal height of the children */
}
.col
{
  flex: 1; /* additionally, equal width */
  padding: 1em;
}
</style>
<style>
.swal2-footer, .result
{
  align-items: center;
}
.swal2-footer > *
{
  padding: 10px;
}
.swal2-container *
{
  color: white;
}
.swal2-footer > img
{
  cursor: pointer;
}
.start-game button
{
  width: 100%;
  font-size: 40px !important;
  font-weight: 900;
  height: 100%;
}
.gradient-pink *
{
  text-shadow: 1px 1px rgba(209,133,159, 0.80), 2px 2px rgba(209,133,159, 0.60), 3px 3px rgba(209,133,159, 0.40), 4px 4px rgba(209,133,159, 0.20), 5px 5px rgba(209,133,159, 0.00);
}
.gradient-pink-big
{
  text-shadow: 1px 1px rgba(209,133,159, 1.00), 2px 2px rgba(209,133,159, 1.00), 3px 3px rgba(209,133,159, 1.00), 4px 4px rgba(209,133,159, 1.00), 5px 5px rgba(209,133,159, 1.00), 6px 6px rgba(209,133,159, 0.99), 7px 7px rgba(209,133,159, 0.98), 8px 8px rgba(209,133,159, 0.97), 9px 9px rgba(209,133,159, 0.96), 10px 10px rgba(209,133,159, 0.94), 11px 11px rgba(209,133,159, 0.91), 12px 12px rgba(209,133,159, 0.87), 13px 13px rgba(209,133,159, 0.82), 14px 14px rgba(209,133,159, 0.76), 15px 15px rgba(209,133,159, 0.68), 16px 16px rgba(209,133,159, 0.59), 17px 17px rgba(209,133,159, 0.48), 18px 18px rgba(209,133,159, 0.34), 19px 19px rgba(209,133,159, 0.19), 20px 20px rgba(209,133,159, 0.00)
}
.swal2-popup.swal2-modal.swal2-show
{
  margin: 3px;
}
</style>
