<template>
  <div id="config">
    <el-row>
      <el-col :xs="23" :sm="23" :md="23" :lg="11" :xl="11">
        <el-card class="box-card">
          <template #header>
            <div class="card-header">
              <span>AList运行状态</span>
              <div>
                <el-button type="primary" v-if="store.aListStatus===0" @click="handleAList('start')">启动</el-button>
                <el-button type="warning" v-if="store.aListStatus!==0" @click="handleAList('restart')">重启</el-button>
                <el-button type="danger" v-if="store.aListStatus!==0" @click="handleAList('stop')">停止</el-button>
              </div>
            </div>
          </template>
          <el-switch
            v-model="aListStarted"
            inline-prompt
            :disabled="true"
            :active-text="store.aListStatus>=2?'运行中':'启动中'"
            inactive-text="停止中"
          />
          <span class="hint" v-if="aListStartTime">启动时间：{{ formatTime(aListStartTime) }}</span>
          <span class="hint warning" v-if="aListRestart">AList需要重启</span>
          <el-progress
            :percentage="percentage"
            :stroke-width="15"
            status="success"
            striped
            striped-flow
            :duration="duration"
            v-if="intervalId"
          />
        </el-card>

        <el-card class="box-card" v-if="store.aListStatus">
          <el-form :model="login" label-width="120px">
            <el-form-item prop="token" label="强制登录AList">
              <el-switch
                v-model="login.enabled"
                inline-prompt
                active-text="开启"
                inactive-text="关闭"
              />
            </el-form-item>
            <el-form-item prop="username" label="用户名">
              <el-input v-model="login.username"/>
            </el-form-item>
            <el-form-item prop="password" label="密码">
              <el-input v-model="login.password" type="password" show-password/>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" @click="updateLogin">保存</el-button>
            </el-form-item>
          </el-form>
        </el-card>

        <el-card class="box-card">
          <el-form :model="form" label-width="120px">
            <el-form-item prop="enabledToken" label="安全订阅">
              <el-switch
                v-model="form.enabledToken"
                inline-prompt
                active-text="开启"
                inactive-text="关闭"
              />
              <span class="hint">建议外网开启，多个安全Token，逗号分割。</span>
            </el-form-item>
            <el-form-item prop="token" label="安全Token">
              <el-input v-model="form.token" type="password" show-password/>
            </el-form-item>
            <el-form-item>
              <el-button type="primary" @click="updateToken">更新</el-button>
            </el-form-item>
          </el-form>
        </el-card>
      </el-col>

      <el-col :xs="23" :sm="23" :md="23" :lg="11" :xl="11">
        <el-card class="box-card">
          <el-form label-width="120px">
            <el-form-item label="计划时间">
              <el-time-picker v-model="scheduleTime"/>
              <el-button type="primary" @click="updateScheduleTime">更新</el-button>
              <span class="hint">自动签到的时间</span>
            </el-form-item>
          </el-form>
        </el-card>

        <el-card class="box-card" v-if="dockerVersion||appVersion||aListVersion">
          <template #header>
            <div class="card-header">
              <span>应用数据</span>
              <div>
                <el-button type="primary" plain @click="dialogVisible=true">高级设置</el-button>
              </div>
            </div>
          </template>
          <div v-if="appVersion">应用版本：{{ appVersion }}</div>
          <div v-if="appRemoteVersion&&appRemoteVersion>appVersion">
            <el-tooltip
              class="box-item"
              effect="dark"
              :content="tooltip"
              placement="top"
            >
              最新版本：{{ appRemoteVersion }}，请重新运行安装脚本，升级应用。
            </el-tooltip>
            <div class="changelog" v-if="changelog">更新日志： {{ changelog }}</div>
          </div>
          <div v-if="aListVersion">AList版本：{{ aListVersion }}</div>
          <div v-if="aListRemoteVersion&&aListRemoteVersion>aListVersion">
            <el-tooltip
              class="box-item"
              effect="dark"
              :content="tooltip"
              placement="top"
            >
              最新版本：{{ aListRemoteVersion }}，请重新运行安装脚本，升级应用。
            </el-tooltip>
          </div>
        </el-card>

        <el-card class="box-card" v-if="indexVersion">
          <template #header>
            <div class="card-header">索引数据</div>
          </template>
          <div>本地版本：{{ indexVersion }}</div>
          <div v-if="indexRemoteVersion&&indexRemoteVersion!=indexVersion">
            最新版本：{{ indexRemoteVersion }}，后台更新中。
          </div>
        </el-card>

        <el-card class="box-card" v-if="movieVersion&&!store.standalone">
          <template #header>
            <div class="card-header">豆瓣电影数据</div>
          </template>
          <el-form label-width="110px">
            <el-form-item label="海报墙混合模式">
              <el-switch
                v-model="mixSiteSource"
                inline-prompt
                active-text="开启"
                inactive-text="关闭"
                @change="updateMixed"
              />
            </el-form-item>
          </el-form>
          <div>本地版本：<a href="/#/meta">{{ movieVersion }}</a></div>
          <div v-if="movieRemoteVersion&&movieRemoteVersion>movieVersion">
            最新版本：{{
              movieRemoteVersion
            }}，后台更新中。
          </div>
        </el-card>
        <el-card class="box-card" v-if="!store.xiaoya">
          <template #header>
            <div class="card-header">海报墙</div>
          </template>
          <el-form label-width="110px">
            <el-form-item label="海报墙混合模式">
              <el-switch
                v-model="mixSiteSource"
                inline-prompt
                active-text="开启"
                inactive-text="关闭"
                @change="updateMixed"
              />
            </el-form-item>
          </el-form>
        </el-card>
      </el-col>
    </el-row>

    <el-dialog id="adv" v-model="dialogVisible" title="高级设置" width="60%">
      <el-form label-width="180px">
        <el-form-item label="开放Token认证URL">
          <el-select v-model="openTokenUrl" class="m-2" placeholder="Select">
            <el-option-group
              v-for="group in options"
              :key="group.label"
              :label="group.label"
            >
              <el-option
                v-for="item in group.options"
                :key="item.value"
                :label="item.label"
                :value="item.value"
              />
            </el-option-group>
          </el-select>
          <el-input v-model="openTokenUrl"/>
          <el-button type="primary" @click="updateOpenTokenUrl">更新</el-button>
        </el-form-item>
        <el-form-item label="Client ID" v-if="openTokenUrl=='https://openapi.alipan.com/oauth/access_token'">
          <el-input v-model="apiClientId" type="text"/>
        </el-form-item>
        <el-form-item label="Client Secret" v-if="openTokenUrl=='https://openapi.alipan.com/oauth/access_token'">
          <el-input v-model="apiClientSecret" type="password" show-password/>
          <el-button type="primary" @click="updateOpenTokenUrl">更新</el-button>
        </el-form-item>
        <el-form-item label="TMDB API Key">
          <el-input v-model="tmdbApiKey" type="password" show-password/>
          <el-button type="primary" @click="updateTmdbApiKey">更新</el-button>
        </el-form-item>
        <el-form-item label="User Agent">
          <el-input v-model="userAgent"/>
          <el-button type="primary" @click="setUserAgent">更新</el-button>
          <el-button type="primary" @click="currentUserAgent">当前UA</el-button>
          <el-button type="primary" @click="randomUserAgent">随机UA</el-button>
        </el-form-item>
        <el-form-item label="AList管理密码" v-if="!store.xiaoya">
          <el-input v-model="atvPass" style="width: 160px" type="password" show-password/>
          <el-button type="primary" class="hint" @click="resetAListPassword">重置</el-button>
        </el-form-item>
        <el-form-item label="AList TvBox API Key">
          <el-input v-model="apiKey" style="width: 300px" type="password" readonly show-password/>
          <el-button type="primary" class="hint" @click="resetApiKey">重置</el-button>
        </el-form-item>
        <el-form-item label="Cookie地址">
          <a :href="currentUrl + '/ali/token/' + aliSecret" target="_blank">
            阿里 Token
          </a><span class="hint"></span>
          <a :href="currentUrl + '/ali/open/' + aliSecret" target="_blank">
            阿里 Open Token
          </a><span class="hint"></span>
          <a :href="currentUrl + '/quark/cookie/' + aliSecret" target="_blank">
            夸克 Cookie
          </a><span class="hint"></span>
          <a :href="currentUrl + '/uc/cookie/' + aliSecret" target="_blank">
            UC Cookie
          </a><span class="hint"></span>
          <a :href="currentUrl + '/115/cookie/' + aliSecret" target="_blank">
            115 Cookie
          </a><span class="hint"></span>
          <a :href="currentUrl + '/baidu/cookie/' + aliSecret" target="_blank">
            百度 Cookie
          </a><span class="hint"></span>
          <a :href="currentUrl + '/bili/cookie/' + aliSecret" target="_blank">
            B站 Cookie
          </a>
        </el-form-item>
        <div class="el-row">
          <el-form-item label="订阅替换阿里token地址">
            <el-switch
              v-model="replaceAliToken"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateReplaceAliToken"
            />
          </el-form-item>
          <el-form-item label="订阅域名支持HTTPS">
            <el-switch
              v-model="enableHttps"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateEnableHttps"
            />
          </el-form-item>
        </div>
        <div class="el-row">
          <el-form-item label="开启调试日志">
            <el-switch
              v-model="debugLog"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateDebugLog"
            />
          </el-form-item>
          <el-form-item label="开启AList调试模式">
            <el-switch
              v-model="aListDebug"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateAListDebug"
            />
          </el-form-item>
        </div>
        <div class="el-row">
          <el-form-item label="网盘分享延迟校验">
            <el-switch
              v-model="aliLazyLoad"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateAliLazyLoad"
            />
          </el-form-item>
          <el-form-item label="自动清理失效资源">
            <el-switch
              v-model="cleanInvalidShares"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateCleanInvalidShares"
            />
          </el-form-item>
          <el-form-item label="网盘帐号负载均衡">
            <el-switch
              v-model="driverRoundRobin"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateDriverRoundRobin"
            />
          </el-form-item>
        </div>
        <div class="el-row">
          <el-form-item label="开启阿里快传115">
            <el-switch
              v-model="aliTo115"
              inline-prompt
              active-text="开启"
              inactive-text="关闭"
              @change="updateAliTo115"
            />
            <span class="hint">帐号页面添加115网盘、配置删除码</span>
          </el-form-item>
        </div>
        <div class="el-row">
          <el-form-item label="网盘文件删除延时">
            <el-input-number v-model="deleteDelayTime" min="0"></el-input-number>
            &nbsp;&nbsp;秒
            <span class="hint">0表示不删除</span>
            <span class="hint"></span>
            <el-button type="primary" @click="updateDeleteDelayTime">更新</el-button>
          </el-form-item>
          <el-form-item label="临时分享过期时间">
            <el-input-number v-model="tempShareExpiration" min="1"></el-input-number>
            &nbsp;&nbsp;小时
            <span class="hint"></span>
            <el-button type="primary" @click="updateTempShareExpiration">更新</el-button>
          </el-form-item>
        </div>
        <el-form-item>
          <el-button @click="resetAListToken">重置AList认证Token</el-button>
          <el-button @click="exportDatabase">导出数据库</el-button>
        </el-form-item>
      </el-form>
      <template #footer>
      <span class="dialog-footer">
        <el-button @click="dialogVisible=false">关闭</el-button>
      </span>
      </template>
    </el-dialog>

  </div>
</template>

<script setup lang="ts">
import {computed, onMounted, ref} from "vue";
import {ElMessage} from "element-plus";
import axios from "axios";
import {onUnmounted} from "@vue/runtime-core";
import {store} from "@/services/store";

let intervalId = 0
const currentUrl = window.location.origin
const percentage = ref<number>(0)
const duration = computed(() => Math.floor(percentage.value / 10))

const increase = () => {
  percentage.value += 1
  if (percentage.value > 100) {
    percentage.value = 100
  }
}

const options = [
  {
    label: '作者自建',
    options: [
      {label: 'har01d', value: 'https://ali.har01d.org/access_token'},
      {label: 'ycyup', value: 'https://ycyup.cn/alipan/access_token'},
    ]
  },
  {
    label: 'TV Token',
    options: [
      {label: '本地代理', value: window.location.origin + '/ali/access_token'},
    ]
  },
  {
    label: '阿里',
    options: [
      {label: '阿里官方', value: 'https://openapi.alipan.com/oauth/access_token'}
    ]
  }
]
const tooltip = ref('sudo bash -c "$(curl -fsSL https://d.har01d.cn/alist-tvbox.sh)"')
const aListStarted = ref(false)
const aListRestart = ref(false)
const mixSiteSource = ref(false)
const replaceAliToken = ref(false)
const debugLog = ref(false)
const aListDebug = ref(false)
const aliTo115 = ref(false)
const driverRoundRobin = ref(false)
const aliLazyLoad = ref(false)
const cleanInvalidShares = ref(false)
const enableHttps = ref(false)
const autoCheckin = ref(false)
const dialogVisible = ref(false)
const changelog = ref('')
const aListVersion = ref('')
const aListRemoteVersion = ref('')
const appVersion = ref(0)
const appRemoteVersion = ref(0)
const dockerVersion = ref('')
const indexVersion = ref('')
const indexRemoteVersion = ref('')
const movieVersion = ref(0)
const movieRemoteVersion = ref(0)
const cachedMovieVersion = ref(0)
const deleteDelayTime = ref(900)
const tempShareExpiration = ref(24)
const aListStartTime = ref('')
const openTokenUrl = ref('')
const dockerAddress = ref('')
const aliSecret = ref('')
const tmdbApiKey = ref('')
const userAgent = ref('')
const atvPass = ref('')
const apiKey = ref('')
const apiClientId = ref('')
const apiClientSecret = ref('')
const scheduleTime = ref(new Date(2023, 6, 20, 8, 0))
const login = ref({
  username: '',
  password: '',
  enabled: false
})

const form = ref({
  token: '',
  enabledToken: true
})

const formatTime = (value: string | number) => {
  return new Date(value).toLocaleString('zh-cn')
}

const updateToken = () => {
  axios.post('/api/token', form.value).then(({data}) => {
    form.value = data
    if (data.enabledToken) {
      ElMessage.success('成功开启安全订阅')
    } else {
      ElMessage.info('成功关闭安全订阅')
    }
  })
}

const resetAListToken = () => {
  axios.post('/api/alist/reset_token', {}).then(() => {
    ElMessage.success('AList认证Token重置成功')
  })
}

const updateOpenTokenUrl = () => {
  axios.post('/api/open-token-url', {
    url: openTokenUrl.value,
    clientId: apiClientId.value,
    clientSecret: apiClientSecret.value
  }).then(() => {
    ElMessage.success('更新成功，重启生效')
  })
}

const updateTmdbApiKey = () => {
  axios.post('/api/settings', {name: 'tmdb_api_key', value: tmdbApiKey.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const currentUserAgent = () => {
  updateUserAgent('current')
}

const randomUserAgent = () => {
  updateUserAgent('')
}

const setUserAgent = () => {
  updateUserAgent(userAgent.value)
}

const updateUserAgent = (value: string) => {
  axios.post('/api/settings', {name: 'user_agent', value: value}).then(({data}) => {
    userAgent.value = data.value
    ElMessage.success('更新成功')
  })
}

const resetApiKey = () => {
  axios.post('/api/settings/apikey').then(({data}) => {
    apiKey.value = data
    ElMessage.success('重置成功')
  })
}

const resetAListPassword = () => {
  axios.post('/api/alist/password').then(({data}) => {
    atvPass.value = data
    ElMessage.success('重置成功，重启生效')
  })
}

const updateDeleteDelayTime = () => {
  axios.post('/api/settings', {name: 'delete_delay_time', value: deleteDelayTime.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateTempShareExpiration = () => {
  axios.post('/api/settings', {name: 'temp_share_expiration', value: tempShareExpiration.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateMixed = () => {
  axios.post('/api/settings', {name: 'mix_site_source', value: mixSiteSource.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateReplaceAliToken = () => {
  axios.post('/api/settings', {name: 'replace_ali_token', value: replaceAliToken.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateEnableHttps = () => {
  axios.post('/api/settings', {name: 'enable_https', value: enableHttps.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateDebugLog = () => {
  axios.post('/api/settings', {name: 'debug_log', value: debugLog.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateAListDebug = () => {
  axios.post('/api/settings', {name: 'alist_debug', value: aListDebug.value}).then(() => {
    ElMessage.success('更新成功，重启生效')
  })
}

const updateAliTo115 = () => {
  axios.post('/api/settings', {name: 'ali_to_115', value: aliTo115.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateDriverRoundRobin = () => {
  axios.post('/api/settings', {name: 'driver_round_robin', value: driverRoundRobin.value}).then(() => {
    ElMessage.success('更新成功')
  })
}

const updateAliLazyLoad = () => {
  axios.post('/api/settings', {name: 'ali_lazy_load', value: aliLazyLoad.value}).then(() => {
    ElMessage.success('更新成功，重启生效')
  })
}

const updateCleanInvalidShares = () => {
  axios.post('/api/settings', {name: 'clean_invalid_shares', value: cleanInvalidShares.value}).then(() => {
    ElMessage.success('更新成功，重启生效')
  })
}

const updateLogin = () => {
  axios.post('/api/alist/login', login.value).then(({data}) => {
    ElMessage.success('保存成功')
    login.value = data
  })
}

const exportDatabase = () => {
  window.location.href = '/api/settings/export' + '?t=' + new Date().getTime() + '&X-ACCESS-TOKEN=' + localStorage.getItem("token");
}

const updateScheduleTime = () => {
  axios.post('/api/schedule', scheduleTime.value).then(() => {
    ElMessage.success('更新成功')
  })
}

const handleAList = (op: string) => {
  axios.post('/api/alist/' + op).then(() => {
    ElMessage.success('操作成功')
    setTimeout(() => getAListStatus(), 3000)
  })
}

const getAListStatus = () => {
  axios.get('/api/alist/status').then(({data}) => {
    increase()
    store.aListStatus = data
    aListStarted.value = data != 0
    if (data !== 1) {
      clearInterval(intervalId)
      intervalId = 0
    } else if (!intervalId) {
      percentage.value = 0
      intervalId = setInterval(getAListStatus, 1000)
    }
  })
}

onMounted(() => {
  axios.get('/api/settings').then(({data}) => {
    form.value.token = data.token
    form.value.enabledToken = data.enabled_token === 'true'
    scheduleTime.value = data.schedule_time || new Date(2023, 6, 20, 9, 0)
    aListStartTime.value = data.alist_start_time
    deleteDelayTime.value = +data.delete_delay_time || 900
    tempShareExpiration.value = +data.temp_share_expiration || 24
    movieVersion.value = data.movie_version
    indexVersion.value = data.index_version
    dockerVersion.value = data.docker_version
    appVersion.value = data.app_version
    aListVersion.value = data.alist_version
    openTokenUrl.value = data.open_token_url
    dockerAddress.value = data.docker_address
    aliSecret.value = data.ali_secret
    tmdbApiKey.value = data.tmdb_api_key
    userAgent.value = data.user_agent
    autoCheckin.value = data.auto_checkin === 'true'
    aListRestart.value = data.alist_restart_required === 'true'
    replaceAliToken.value = data.replace_ali_token === 'true'
    enableHttps.value = data.enable_https === 'true'
    debugLog.value = data.debug_log === 'true'
    aListDebug.value = data.alist_debug === 'true'
    aliTo115.value = data.ali_to_115 === 'true'
    driverRoundRobin.value = data.driver_round_robin === 'true'
    cleanInvalidShares.value = data.clean_invalid_shares === 'true'
    aliLazyLoad.value = data.ali_lazy_load !== 'false'
    mixSiteSource.value = data.mix_site_source !== 'false'
    atvPass.value = data.atv_password
    apiKey.value = data.api_key
    apiClientId.value = data.open_api_client_id || ''
    apiClientSecret.value = data.open_api_client_secret || ''
    login.value.username = data.alist_username
    login.value.password = data.alist_password
    login.value.enabled = data.alist_login === 'true'
    if (store.standalone) {
      tooltip.value = 'bash -c "$(curl -fsSL http://d.har01d.cn/install-service.sh)"'
    } else {
      tooltip.value = 'sudo bash -c "$(curl -fsSL http://d.har01d.cn/alist-tvbox.sh)" -s update'
    }
  })
  axios.get('/api/alist/status').then(({data}) => {
    store.aListStatus = data
    aListStarted.value = data != 0
    if (data === 1) {
      percentage.value = 0
      intervalId = setInterval(getAListStatus, 1000)
    }
  })
  axios.get('/api/versions').then(({data}) => {
    movieRemoteVersion.value = data.movie
    cachedMovieVersion.value = data.cachedMovie
    indexRemoteVersion.value = data.index
    appRemoteVersion.value = data.app
    aListRemoteVersion.value = data.alist
    changelog.value = data.changelog
  })
})

onUnmounted(() => {
  clearInterval(intervalId)
})
</script>

<style>
.main {
  max-width: 1080px;
}

.el-col {
  margin-left: 24px;
}

.bottom {
  margin-bottom: 0px;
}

.warning {
  color: #e6a23c;
}

.append {
  margin-left: 6px;
  margin-right: 6px;
}

.hint {
  margin-left: 16px;
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.box-card {
  margin-bottom: 12px;
}

.changelog {
  color: #67c23a;
}

@media only screen and (max-width: 1500px) {
  #adv {
    width: 80%;
  }
}

@media only screen and (max-width: 960px) {
  #adv {
    width: 96%;
  }
}
</style>
