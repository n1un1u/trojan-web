<template>
  <div :style="mainStyle">
    <el-form :inline="true" style="margin-top: 15px">
        <el-form-item  size="small">
            <el-button-group>
                <el-button type="primary" icon="el-icon-refresh" @click="update()">{{ $t('update') }}</el-button>
                <el-button type="primary" icon="el-icon-video-play" @click="start()">{{ $t('start') }}</el-button>
                <el-button type="primary" icon="el-icon-video-pause" @click="stop()">{{ $t('stop') }}</el-button>
                <el-button type="primary" icon="el-icon-refresh-right" @click="restart()">{{ $t('restart') }}</el-button>
            </el-button-group>
        </el-form-item>
        <el-form-item  size="small" :label="$t('loglevel')">
            <el-select size="mini" v-model="loglevel" :placeholder="$t('choice')" filterable @change="setLoglevel()" style="width: 120px;">
            <el-option
                v-for="item in loglevelOptions"
                :key="item.label"
                :label="item.label"
                :value="item.value">
            </el-option>
            </el-select>
        </el-form-item>
        <el-form-item  size="small" :label="$t('line')">
            <el-select size="mini" v-model="line" :placeholder="$t('choice')" filterable @change="getLog()" style="width: 120px;">
            <el-option
                v-for="item in logLineOptions"
                :key="item.label"
                :label="item.label"
                :value="item.value">
            </el-option>
            </el-select>
        </el-form-item>
        <el-form-item  size="small" :label="$t('latest')">
            <el-switch v-model="isFollow"></el-switch>
        </el-form-item>
    </el-form>
    <textarea id="logshow" readonly="readonly" class="el-textarea__inner"></textarea>
  </div>
</template>

<script>
import store from '@/store/index.js'
import { mapState } from 'vuex'
import { start, stop, restart, update, getLoglevel, setLoglevel } from '@/api/trojan'
export default {
    data() {
        return {
            queue: [],
            timer: null,
            isFollow: true,
            ws: null,
            scrollHeight: 0,
            mainStyle: {
                height: 0
            },
            loglevelOptions: [
                {
                    label: 'ALL',
                    value: 0
                },
                {
                    label: 'INFO',
                    value: 1
                },
                {
                    label: 'WARN',
                    value: 2
                },
                {
                    label: 'ERROR',
                    value: 3
                },
                {
                    label: 'FATAL',
                    value: 4
                },
                {
                    label: 'OFF',
                    value: 5
                }
            ],
            logLineOptions: [
                {
                    label: '100',
                    value: 100
                },
                {
                    label: '300',
                    value: 300
                },
                {
                    label: '1000',
                    value: 1000
                },
                {
                    label: '5000',
                    value: 5000
                },
                {
                    label: this.$t('all'),
                    value: -1
                }
            ]
        }
    },
    mounted() {
        this.$store.commit('SET_NOERROR', true)
        this.mainStyle.height = (document.body.clientHeight - 85) + 'px'
        const textarea = document.getElementById('logshow')
        // 监听这个dom的scroll事件
        textarea.addEventListener('scroll', () => {
            if (textarea.scrollHeight > textarea.clientHeight && textarea.scrollTop < this.scrollHeight) {
                this.isFollow = false
            } else {
                this.isFollow = true
            }
        }, true)
        this.getLog()
        this.getLoglevel()
    },
    computed: {
        ...mapState(['line', 'loglevel']),
        line: {
            get() {
                return this.$store.state.line
            },
            set(val) {
                this.$store.commit('SET_LINE', val)
            }
        },
        loglevel: {
            get() {
                return this.$store.state.loglevel
            },
            set(val) {
                this.$store.commit('SET_LOGLEVEL', val)
            }
        }
    },
    destroyed() {
        this.$store.commit('SET_NOERROR', false)
        clearInterval(this.timer)
        this.timer = null
        this.ws.close()
    },
    methods: {
        async setLoglevel() {
            try {
                const formData = new FormData()
                formData.set('level', this.loglevel)
                const result = await setLoglevel(formData)
                if (result.Msg === 'success') {
                    this.$message({
                        message: this.$t('trojan.loglevelSuccess'),
                        type: 'success'
                    })
                }
            } catch (e) {
                this.getLog()
            }
        },
        async getLoglevel() {
            const result = await getLoglevel()
            if (result.Msg === 'success') {
                this.loglevel = result.Data.loglevel
            } else {
                this.$message.error(result.Msg)
            }
        },
        async start() {
            try {
                const result = await start()
                if (result.Msg === 'success') {
                    this.$message({
                        message: this.$t('trojan.startSuccess'),
                        type: 'success'
                    })
                }
            } catch (e) {
                this.getLog()
            }
        },
        async stop() {
            const result = await stop()
            if (result.Msg === 'success') {
                this.$message({
                    message: this.$t('trojan.stopSuccess'),
                    type: 'success'
                })
            }
        },
        async restart() {
            try {
                const result = await restart()
                if (result.Msg === 'success') {
                    this.$message({
                        message: this.$t('trojan.restartSuccess'),
                        type: 'success'
                    })
                }
            } catch (e) {
                this.getLog()
            }
        },
        async update() {
            try {
                const result = await update()
                if (result.Msg === 'success') {
                    this.$message({
                        message: this.$t('trojan.updateSuccess'),
                        type: 'success'
                    })
                }
            } catch (e) {
                this.getLog()
            }
        },
        getLog() {
            this.isFollow = true
            const self = this
            if (this.ws != null) {
                this.ws.close()
                clearInterval(this.timer)
                this.timer = null
                this.queue = []
            }
            const textarea = document.getElementById('logshow')
            textarea.innerText = ''
            const prefix = process.env.NODE_ENV === 'production' ? '/' : '/ws'
            const protocol = document.location.protocol === 'http:' ? 'ws' : 'wss'
            this.ws = new WebSocket(`${protocol}://${location.host}${prefix}/trojan/log?line=${this.line}&token=${store.state.UserToken}`)
            this.ws.onopen = function () {
                console.log('ws connected!')
            }
            this.ws.onmessage = function(e) {
                self.queue.push(e.data)
            }
            this.ws.onerror = function(e) {
                console.log('ws error: ' + e)
            }
            this.ws.onclose = function(e) {
                console.log('ws closed')
            }
            let firstRun = true
            this.timer = setInterval(() => {
                let message = ''
                while (this.queue.length > 0) {
                    message = message + this.queue.shift()
                }
                textarea.append(message)
                if (firstRun) {
                    textarea.scrollTop = textarea.scrollHeight
                    this.scrollHeight = textarea.scrollTop
                    firstRun = !firstRun
                }
                if (this.isFollow) {
                    textarea.scrollTop = textarea.scrollHeight
                    this.scrollHeight = textarea.scrollTop
                }
            }, 1000)
        }
    }
}
</script>

<style lang="scss" scoped>
#logshow {
    width: 100%;
    height: 92%;
    padding: 10px;
    background-color: black;
    color:white;
    font-size: 13px;
}
</style>
