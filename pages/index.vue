<template>
    <div class="grid-container" @mouseup="handleClick({ keyCode: null }, true)">
        <div class="stat-header">
            <div class="game-count">
                Visits: {{ storage.visits ? storage.visits : 0 }}<br>
                Games: {{ storage.games ? storage.games : 0 }}
            </div>
            <div>
                <div>
                    <b>Highscore</b>
                </div>
                <div style="text-align: right;">
                    {{ ((storage['highscore-name'] || '') || '_ _ _ _ _ ') + ':' }}
                    {{ storage['highscore'] || 0 }}
                </div>
            </div>
        </div>
        <div class="game-header">
            <a-button :type="state === 0 ? 'primary' : 'default'" class="reset-button" @click="state === 0 ? start() : restart()">
                {{ state !== 2 ? (state === 0 ? 'Start' : 'Reset') : 'New Game' }}
            </a-button>
        </div>
        <div class="grid">
            {{ score }}
            <canvas
                id="myCanvas"
                width="360px"
                height="360px"
                style="position: absolute;"
            >
                Your browser does not support the canvas element.
            </canvas>
            <input
                ref="controlInput"
                class="control-input"
                onblur="this.focus()"
                autofocus
                inputmode="none"
                @keydown="handleClick($event)"
                @keyup="handleClick($event, true)"
            >
        </div>
        <div class="buttons">
            <div class="row">
                <a-button
                    v-for="btn in buttons"
                    :key="btn.keyCode"
                    :disabled="state === 0"
                    class="button"
                    :class="target[index] === btn.keyCode && state === 1 ? 'target' : (state === 0 ? 'game-paused' : 'game-on')"
                    :style="{
                        backgroundColor: btn.color,
                        'box-shadow': getBoxShadow(btn.color, !active.includes(btn.keyCode)).join(', ')
                    }"
                    @mousedown="handleClick(btn)"
                />
            </div>
        </div>
    </div>
</template>

<script>
const namespace = 'zappy.ninja'

export default {
    layout: 'default',
    data () {
        return {
            size: 15,
            speed: 1,
            state: 0,
            score: 0,
            active: [],
            target: [],
            last: 90,
            index: null,
            buffer: [],
            storage: {},
            buttons: [
                {
                    keyCode: 90,
                    color: '#0EC518'
                },
                {
                    keyCode: 88,
                    color: '#fa9c00'
                },
                {
                    keyCode: 67,
                    color: '#db1c11'
                },
                {
                    keyCode: 86,
                    color: '#ffdb1f'
                }
            ]
        }
    },
    computed: {
        pixels () {
            return 360 / this.size
        }
    },
    watch: {
        state (value) {
            if (value === 1) {
                this.count('games')
            }
        }
    },
    mounted () {
        this.focusInput()
        const c = document.getElementById('myCanvas')
        const ctx = c.getContext('2d')
        this.vueCanvas = ctx
    },
    created () {
        this.$nextTick(() => {
            this.count('games', 'get')
            this.count('visits')
            this.updateCanvas()
            this.getHighscore()
        })
    },
    methods: {
        focusInput () {
            this.$refs.controlInput.focus()
        },
        start () {
            this.state = 1
            this.index = null
            this.buffer = []
            this.target = []
            this.active = []
            this.speed = 1
            this.score = 0
            setTimeout(() => this.game(), 1000)
        },
        game () {
            this.interval = setInterval(() => {
                if (this.state !== 1) {
                    clearInterval(this.interval)
                    return
                }
                let candidates
                if (this.target.length === 0) {
                    candidates = this.buttons.map(btn => btn.keyCode).filter(code => code !== this.last)
                } else {
                    candidates = this.buttons.map(btn => btn.keyCode).filter(code => code !== this.target[this.target.length - 1])
                }
                this.target.push(candidates[Math.floor(Math.random() * candidates.length)])
                if (this.index === null) {
                    this.index = 0
                } else {
                    this.index++
                }
                if (this.index > 10) {
                    this.state = 0
                    this.saveScore(this.score)
                } else {
                    const newSpeed = this.speed + (this.score / (100 * ((this.score || 1) / 2)))
                    if (newSpeed !== this.speed) {
                        clearInterval(this.interval)
                        this.speed = newSpeed
                        this.game()
                    }
                }
            }, 1500 / this.speed)
        },
        restart () {
            clearInterval(this.interval)
            this.state = 0
            this.index = null
            this.buffer = []
            this.target = []
            this.active = []
            this.speed = 1
            this.updateCanvas()
        },
        handleClick ({ keyCode }, up) {
            if (this.state !== 1) {
                return
            }
            this.$refs.controlInput.value = ''
            if (keyCode && this.buffer[this.buffer.length - 1] !== keyCode) {
                this.buffer.push(keyCode)
                if (this.buffer.length > 10) {
                    this.buffer.shift()
                }
            }
            if (up) {
                this.active.shift()
            } else if (this.active[this.active.length - 1] !== keyCode && keyCode !== null) {
                this.active.push(keyCode)
                if (keyCode && this.target[0] === keyCode) {
                    this.score++
                    this.last = keyCode
                    this.target.shift()
                    this.index--
                } else {
                    this.state = 0
                    this.saveScore(this.score)
                }
            }
        },
        updateCanvas () {
            this.removePart([0, 0], this.pixels * this.size, this.pixels * this.size)
        },
        removePart ([y, x], w = this.pixels, h = this.pixels) {
            this.vueCanvas.clearRect(x * this.pixels, y * this.pixels, w, h)
        },
        getBoxShadow (color, active) {
            return active
                ? [
                    `0px 5px 5px rgba(${this.lightenDarkenColor(color, -40)}, 0.5)`,
                    `0px -2px 0px 3px rgb(${this.lightenDarkenColor(color, -10)}) inset`,
                    '0px 8px rgba(255, 255, 255, 0.25) inset'
                ]
                : [
                    `0px 2px 0px 2px rgb(${this.lightenDarkenColor(color, -30)}) inset`
                ]
        },
        hexToRgb (hex) {
            const shorthandRegex = /^#?([a-f\d])([a-f\d])([a-f\d])$/i
            hex = hex.replace(shorthandRegex, function (m, r, g, b) {
                return r + r + g + g + b + b
            })

            const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(hex)
            return result
                ? Object.values({
                    r: parseInt(result[1], 16),
                    g: parseInt(result[2], 16),
                    b: parseInt(result[3], 16)
                }).join(', ')
                : null
        },
        lightenDarkenColor (color, amount) {
            return this.hexToRgb('#' + color.replace(/^#/, '').replace(/../g, color => ('0' + Math.min(255, Math.max(0, parseInt(color, 16) + amount)).toString(16)).substr(-2)))
        },
        create (key, value = 0) {
            const xhr = new XMLHttpRequest()
            xhr.open('GET', `https://api.countapi.xyz/create?namespace=${namespace}&key=${key}&value=${value}&enable_reset=1`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                console.log(target.response)
            }
            xhr.send()
        },
        saveName (prefix, name = '') {
            const xhr = new XMLHttpRequest()
            const key = `${prefix}-name`
            let hexValue = Buffer.from((name || '').slice(0, 1)).toString('hex')
            let intValue = BigInt('0x' + hexValue)
            let length = 1
            let result
            while (true) {
                length++
                hexValue = Buffer.from((name || '').slice(0, length)).toString('hex')
                intValue = BigInt('0x' + hexValue)
                if (intValue <= 99999999999999 && length < 10) {
                    result = intValue.toString()
                } else {
                    break
                }
            }
            xhr.open('GET', `https://api.countapi.xyz/set/${namespace}/${key}?value=${result}`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                const hexValue = (value || '').toString(16)
                this.$set(this.storage, key, Buffer.from(hexValue, 'hex').toString())
            }
            xhr.send()
        },
        getName (prefix) {
            const xhr = new XMLHttpRequest()
            const key = `${prefix}-name`
            xhr.open('GET', `https://api.countapi.xyz/get/${namespace}/${key}`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                let hexValue = ''
                let result = null
                if (value) {
                    hexValue = BigInt(value).toString(16)
                    result = Buffer.from(hexValue, 'hex').toString()
                } else if (value === 0) {
                    result = 0
                }
                this.$set(this.storage, key, result)
                if (!this.storage[key] && this.storage[key] !== 0) {
                    this.create(key)
                }
            }
            xhr.send()
        },
        getHighscore (cb) {
            const xhr = new XMLHttpRequest()
            const key = 'highscore'
            xhr.open('GET', `https://api.countapi.xyz/get/${namespace}/${key}`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                this.$set(this.storage, key, value)
                if (!this.storage[key] && this.storage[key] !== 0) {
                    this.create(key)
                }
                if (cb) {
                    cb()
                }
            }
            xhr.send()
            this.getName(key)
        },
        saveScore (score) {
            this.getHighscore(() => {
                const key = 'highscore'
                const currentValue = this.storage[key] || 0
                let name
                if (score >= currentValue) {
                    name = prompt('Please enter your name', '')
                }
                if (score >= currentValue) {
                    const xhr = new XMLHttpRequest()
                    xhr.open('GET', `https://api.countapi.xyz/set/${namespace}/${key}?value=${score}`)
                    xhr.responseType = 'json'
                    xhr.onload = ({ target }) => {
                        const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                        this.$set(this.storage, key, value)
                    }
                    xhr.send()
                    this.saveName(key, name)
                }
            })
        },
        count (key, endpoint = 'hit') {
            const xhr = new XMLHttpRequest()
            xhr.open('GET', `https://api.countapi.xyz/${endpoint}/${namespace}/${key}`)
            xhr.responseType = 'json'
            xhr.onload = ({ target }) => {
                const value = Object.hasOwnProperty.call(target.response, 'value') ? target.response.value : 0
                this.$set(this.storage, key, value)
            }
            xhr.send()
        }
    }
}
</script>

<style lang="scss">
    .grid-container {
        position: relative;
        display: flex;
        justify-content: center;
        align-items: center;
        flex-direction: column;
        min-height: 100vh;
        background: #2b2b2b;
    }

    .stat-header {
        width: 360px;
        color: #88adc2;
        justify-content: space-between;
        align-items: center;
        display: flex;
        flex-direction: row;
        margin-bottom: 10px;
    }

    .game-header {
        width: 360px;
        height: 60px;
        justify-content: space-between;
        align-items: center;
        display: flex;
        flex-direction: row;
        margin-bottom: 5px;
    }

    .grid {
        position: relative;
        font-size: 220px;
        text-align: center;
        width: 360px;
        height: 360px;
        display: flex;
        flex-direction: column;
        border-radius: 12px;
        border: 1px solid #525252;
        color: #15321d;
        background: #eaeaea;
        background: linear-gradient(45deg, #bad9d2, #79a577 100%);
    }

    .buttons {
        margin-top: 15px;
        position: relative;
        width: 360px;
        display: flex;
        flex-direction: column;
    }

    .row {
        width: 100%;
        flex-grow: 1;
        display: flex;
    }

    .game-paused {
        filter: brightness(100%);
    }

    .game-on {
        filter: brightness(50%);
    }

    .target {
        filter: brightness(100%);
    }

    .enabled-buttons {
        width: 100%;
        height: 100px;
        position: absolute;
        opacity: 0;
        top: 0;
        left: 0;
        z-index: -100;
        transition: all 0.5s;
    }

    .button {
        position: relative;
        background-color: #d7d7d7;
        border: none;
        flex-grow: 1;
        border-radius: 50%;
        height: 80px;
        margin: 5px;
        background-image: -webkit-linear-gradient(top, #f4f1ee, #fff);
        background-image: linear-gradient(top, #f4f1ee, #fff);
        float: left;
        width: 70px;
        -webkit-transition: all .2s linear;
        transition: all .2s linear;
    }

    .button:active {
        background-image: -webkit-linear-gradient(top, #efedec, #f7f4f4);
        background-image: linear-gradient(top, #efedec, #f7f4f4);
        box-shadow: 0 3px 5px 0 rgba(0,0,0,.4), inset 0px -3px 1px 1px rgba(204,198,197,.5);
    }

    .reset-button {
        width: 100%;
        height: 40px;
        font-size: 18px;
        border-radius: 12px;
        margin-bottom: 10px;
    }

    .control-input {
        position: absolute;
        opacity: 0;
        top: 0;
        left: 0;
        width: 360px;
    }
</style>
