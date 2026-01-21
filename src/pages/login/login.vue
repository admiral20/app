<route lang="json5" type="page">
{
  style: {
    navigationStyle: 'custom',
    navigationBarTitleText: '',
    disableScroll: true, // 微信禁止页面滚动
    'app-plus': {
      bounce: 'none', // 禁用 iOS 弹性效果
    },
  },
}
</route>

<template>
  <PageLayout :navbarShow="false">
    <view class="page-container">
      <!-- <view class="text-center">
        <image :src="compLogo" mode="aspectFit" class="logo"></image>
        <view class="title text-shadow">{{ compTitle || 'JEECG BOOT' }}</view>
        <view class="enter-area">
          <view v-if="loginWay == 1" class="account-login-area">
            <view class="box account">
              <wd-icon name="user" size="15px"></wd-icon>
              <wd-text text="账号："></wd-text>
              <wd-input placeholder="请输入账号" v-model.trim="userName"></wd-input>
            </view>
            <view class="box password">
              <wd-icon name="lock-on" size="15px"></wd-icon>
              <wd-text text="密码："></wd-text>
              <input
                class="uni-input"
                placeholder="请输入密码"
                :password="showPassword"
                v-model.trim="password"
              />
              <wd-icon
                v-if="showPassword"
                name="eye-close"
                size="18px"
                @click="handleChangePassword"
              ></wd-icon>
              <wd-icon v-else name="view" size="18px" @click="handleChangePassword"></wd-icon>
            </view>
          </view>
          <view v-else class="phone-login-area">
            <view class="account-login-area">
              <view class="box account">
                <wd-icon name="mobile" size="15px"></wd-icon>
                <wd-text text="手机号：" color="#000"></wd-text>
                <wd-input placeholder="请输入手机号" v-model="phoneNo"></wd-input>
              </view>
              <view class="box password">
                <wd-icon name="lock-on" size="15px"></wd-icon>
                <wd-text text="验证码：" color="#000"></wd-text>
                <wd-input placeholder="请输入验证码" v-model="smsCode"></wd-input>
                <wd-button
                  :round="false"
                  size="small"
                  custom-class="sendSMSBtn"
                  plain
                  hairline
                  :disabled="isSendSMSEnable"
                  @click="handleSMSSend"
                >
                  {{ getSendBtnText }}
                </wd-button>
              </view>
            </view>
          </view>
        </view>
        <view class="btn-area text-center">
          <wd-button  custom-class="login align-top" :loading="loading" @click="hanldeLogin">
            {{ loading ? '登录...' : '登录' }}
          </wd-button>
          <wd-button v-if="loginWay == 2" plain hairline @click="toggleLoginWay(1)">
            账户登录
          </wd-button>
          <wd-button v-else custom-class="align-top" plain hairline @click="toggleLoginWay(2)">
            短信登录
          </wd-button>
        </view>
      </view>
      <wd-notify /> -->
      <!-- 静态本地 DB 展示区域（无网络） -->
      <view class="static-db">
        <view class="section-title">本地静态 DB 数据</view>
        <view v-if="dbLoading" class="hint">加载中...</view>
        <view v-else>
          <!-- 调试操作按钮 -->
          <view class="actions">
            <button @click="handleCopyDb">复制到 _doc</button>
            <button @click="handleOpenDb">打开 DB</button>
            <button @click="handleCloseDb">关闭 DB</button>
            <button @click="handleQueryItems">查询 items</button>
            <button @click="handleQueryUsers">查询 users</button>
            <button @click="handleListTables">列出表</button>
            <button @click="handleCountTables">计数</button>
            <button @click="handleRefreshDb">刷新</button>
          </view>

          <!-- 状态信息 -->
          <view class="status">
            <view class="hint">建议将 static.db 放在项目根目录 static/static.db（打包后路径固定为 _www/static/static.db）</view>
            <view>环境: {{ envPlusAvailable ? 'APP-PLUS 可用' : '不可用' }}</view>
            <view>源路径(选中): {{ selectedSrcPath || '未确定' }}</view>
            <view>目标路径: {{ DB_DST_PATH }}</view>
            <view>dbReady: {{ dbReady }}</view>
            <view>dbOpened: {{ dbOpened }}</view>
            <view>dbFileSize(bytes): {{ dbFileSize }}</view>
            <view>items条数: {{ itemsCount }}</view>
            <view>users条数: {{ usersCount }}</view>
            <view v-if="lastError" class="hint">错误: {{ lastError }}</view>
          </view>

          <view class="block">
            <view class="section-title">items 表</view>
            <view v-if="dbItems.length === 0" class="hint">暂无数据</view>
            <block v-else>
              <view v-for="it in dbItems" :key="it.id" class="row">
                <text>{{ it.id }}. {{ it.title }} - {{ it.description }}</text>
                <view class="hint">时间：{{ it.createdAt }}</view>
              </view>
            </block>
          </view>

          <view class="block">
            <view class="section-title">users 表</view>
            <view v-if="dbUsers.length === 0" class="hint">暂无数据</view>
            <block v-else>
              <view v-for="u in dbUsers" :key="u.id" class="row">
                <text>{{ u.id }}. {{ u.name }}（{{ u.role }}）</text>
              </view>
            </block>
          </view>

          <!-- 日志输出 -->
          <view class="logs">
            <view class="section-title">调试日志</view>
            <view v-if="debugLogs.length === 0" class="hint">暂无日志</view>
            <block v-else>
              <view v-for="(l, idx) in debugLogs" :key="idx" class="log-row">{{ l }}</view>
            </block>
          </view>
        </view>
      </view>
    </view>
  </PageLayout>
</template>

<script lang="ts" setup>
import { useNotify, useToast } from 'wot-design-uni'
import { ref, onMounted, computed } from 'vue'
import { useUserStore } from '@/store/user'
import { http } from '@/utils/http'
import {
  ACCESS_TOKEN,
  USER_NAME,
  USER_INFO,
  APP_ROUTE,
  APP_CONFIG,
  HOME_CONFIG_EXPIRED_TIME,
  HOME_PAGE,
} from '@/common/constants'

import { cache, getFileAccessHttpUrl } from '@/common/uitls'
import { useRouter } from '@/plugin/uni-mini-router'
import { useParamsStore } from '@/store/page-params'

defineOptions({
  name: 'login',
  options: {
    // apply-shared‌：当前页面样式会影响到子组件样式.(小程序)
    // shared‌：当前页面样式影响到子组件，子组件样式也会影响到当前页面.(小程序)
    styleIsolation: 'shared',
  },
})
const router = useRouter()
const defLogo = 'https://static.jeecg.com/files/app_logo.png'
const shape = ref()
const loading = ref(false)
const userName = ref()
const password = ref()
const phoneNo = ref('')
const smsCode = ref('')
const showPassword = ref(false) //是否显示明文
const loginWay = ref(1) //1: 账密，2：验证码
const smsCountDown = ref(0)
let smsCountInterval = null
const toggleDelay = ref(false)
const version = ref('')
const compLogo = ref(defLogo)
const compTitle = ref('Jeecg Uniapp')
const paramsStore = useParamsStore()
paramsStore.reset()
// 是否开启本地路由配置
let isLocalConfig = getApp().globalData.isLocalConfig;
if (import.meta.env.MODE === 'development') {
  userName.value = 'admin'
  password.value = '123456'
}

if (import.meta.env.MODE === 'production') {
  userName.value = 'jeecg'
  password.value = 'jeecg#123456'
}

const isSendSMSEnable = computed(() => {
  return smsCountDown.value <= 0 && phoneNo.value.length < 4
})
const toast = useToast()
const userStore = useUserStore()
const toggleLoginWay = (val) => {
  loginWay.value = val
}
const handleChangePassword = () => {
  showPassword.value = !showPassword.value
}
const handleSMSSend = () => {
  let smsParams = { mobile: '', smsmode: '0' }
  smsParams.mobile = phoneNo.value
  let checkPhone = new RegExp(/^[1]([3-9])[0-9]{9}$/)
  if (!smsParams.mobile || smsParams.mobile.length == 0) {
    toast.warning('请输入手机号')
    return false
  }
  if (!checkPhone.test(smsParams.mobile)) {
    toast.warning('请输入正确的手机号')
    return false
  }
  if (smsCountDown.value) {
    return
  }
  http.post('/sys/sms', smsParams).then((res: any) => {
    if (res.success) {
      smsCountDown.value = 60
      startSMSTimer()
    } else {
      smsCountDown.value = 0
      toast.warning(res.message)
    }
  })
}
const startSMSTimer = () => {
  smsCountInterval = setInterval(() => {
    smsCountDown.value--
    if (smsCountDown.value <= 0) {
      clearInterval(smsCountInterval)
    }
  }, 1e3)
}
const getSendBtnText = computed(() => {
  if (smsCountDown.value > 0) {
    return smsCountDown.value + '秒后发送'
  } else {
    return '发送验证码'
  }
})
const hanldeLogin = () => {
  loginWay.value === 1 ? accountLogin() : phoneLogin()
}
const accountLogin = () => {
  if (userName.value.length === 0) {
    toast.warning('请输入账号')
    return
  }
  if (password.value.length === 0) {
    toast.warning('请输入密码')
    return
  }
  loading.value = true
  http
    .post('/sys/mLogin', { username: userName.value, password: password.value })
    .then((res: any) => {
      if (res.success) {
        const { result } = res
        const userInfo = result.userInfo
        userStore.setUserInfo({
          ...userInfo,
          token: result.token,
          userid: userInfo.id,
          username: userInfo.username,
          realname: userInfo.realname,
          avatar: userInfo.avatar,
          tenantId: userInfo.loginTenantId,
          localStorageTime: +new Date(),
        })
        appConfig()
        departConfig()
        router.pushTab({ path: HOME_PAGE })
      } else {
        toast.warning(res.message)
      }
    })
    .finally(() => {
      loading.value = false
    })
}

const phoneLogin = () => {
  let checkPhone = new RegExp(/^[1]([3-9])[0-9]{9}$/)

  if (!phoneNo.value || phoneNo.value.length == 0) {
    toast.warning('请输入手机号')
    return
  }
  if (!checkPhone.test(phoneNo.value)) {
    toast.warning('请输入正确的手机号')
    return false
  }
  if (!smsCode.value || smsCode.value.length == 0) {
    toast.warning('请输入短信验证码')
    return
  }
  let loginParams = {
    mobile: phoneNo.value,
    captcha: smsCode.value,
  }
  http
    .post('/sys/phoneLogin', { mobile: phoneNo.value, captcha: smsCode.value })
    .then((res: any) => {
      if (res.success) {
        const { result } = res
        const userInfo = result.userInfo
        userStore.setUserInfo({
          token: result.token,
          userid: userInfo.id,
          username: userInfo.username,
          realname: userInfo.realname,
          avatar: userInfo.avatar,
          tenantId: userInfo.loginTenantId,
          localStorageTime: +new Date(),
        })
        //获取app配置
        appConfig()
        departConfig()
      } else {
        toast.warning(res.message)
      }
    })
    .catch((err) => {
      let msg = err.message || '请求出现错误，请稍后再试'
      toast.warning(msg)
    })
}
//部門配置
const departConfig = () => {
  const appQueryUser = () => {
    http
      .get('/sys/user/appQueryUser', {
        username:userStore.userInfo.username,
      })
      .then((res: any) => {
        if (res.success && res.result.length) {
          let result = res.result[0];
          userStore.editUserInfo({orgCodeTxt: result.orgCodeTxt})
        }
      })
  }
  appQueryUser()
}
const appConfig = () => {
  if (isLocalConfig) {
    toast.success('登录成功!')
    router.pushTab({ path: HOME_PAGE })
  } else {
    http
      .get('/eoa/sysAppConfig/queryAppConfigRoute')
      .then((res: any) => {
        if (res.success) {
          cache(APP_ROUTE, res.result.route, HOME_CONFIG_EXPIRED_TIME)
          cache(APP_CONFIG, res.result.config, HOME_CONFIG_EXPIRED_TIME)
        }
        toast.success('登录成功!')
        router.pushTab({ path: HOME_PAGE })
      })
      .catch((err) => {
        toast.success('登录成功!')
        router.pushTab({ path: HOME_PAGE })
      })
  }
}

const loadConfig = () => {
  http.get('/eoa/sysAppConfig/queryAppConfig').then((res: any) => {
    if (res.success) {
      let info = res.result
      if (info) {
        compLogo.value = getFileAccessHttpUrl(info.appLogo) || defLogo
        compTitle.value = info.appTitle || 'JEECG-BOOT'
      } else {
        compLogo.value = defLogo
      }
    }
  })
}
const checkToken = () => {
  const { userInfo, clearUserInfo } = userStore
  if (userInfo.token) {
    if (+new Date() - userInfo.localStorageTime > 2 * 3600000) {
      // 超过2小时过期
      clearUserInfo()
    } else {
      router.pushTab({ path: HOME_PAGE })
    }
  } else {
    clearUserInfo()
  }
}
const checkAccount = () => {}
// #ifdef APP-PLUS || H5
checkToken()
checkAccount()
// #endif
// @ts-ignore
if (isLocalConfig === false) {
  loadConfig()
}

/**
 * 静态本地 DB 展示（无网络）
 * 在 app-plus 环境下通过 plus.sqlite 直接读取打包内的 SQLite 文件。
 * 当在 H5 或读取失败时，使用内置示例数据。
 */
type DbItem = { id: number; title: string; description?: string; createdAt: string }
type DbUser = { id: number; name: string; role: string }
const DB_NAME = 'localStaticDb'
// 仅 APP 使用：将数据库放在项目根 static/static.db（与图片同级）
// 打包后固定路径为 _www/static/static.db，这里只保留该路径以减少歧义
const DB_SRC_CANDIDATES = ['_www/static/static.db']
const DB_DST_DIR = '_doc/'
const DB_DST_FILE = 'static.db'
const DB_DST_PATH = `${DB_DST_DIR}${DB_DST_FILE}`

const dbItems = ref<DbItem[]>([])
const dbUsers = ref<DbUser[]>([])
const dbLoading = ref(false)
const dbReady = ref(false)
const dbOpened = ref(false)
const itemsCount = ref(0)
const usersCount = ref(0)
const lastError = ref<string | null>(null)
const envPlusAvailable = ref(false)
const debugLogs = ref<string[]>([])
const selectedSrcPath = ref<string | null>(null)
const dbFileSize = ref<number | null>(null)

function pushLog(msg: any) {
  const s = typeof msg === 'string' ? msg : JSON.stringify(msg)
  const line = `[${new Date().toLocaleString()}] ${s}`
  debugLogs.value.unshift(line)
  if (debugLogs.value.length > 200) debugLogs.value.pop()
  // 同步打印到控制台
  console.log(line)
}

// 不考虑 H5：不提供内置数据回退

async function ensureDbReady(): Promise<boolean> {
  // @ts-ignore
  if (typeof plus === 'undefined') return false
  const resolveURL = (url: string) => new Promise<any>((res, rej) => {
    // @ts-ignore
    plus.io.resolveLocalFileSystemURL(url, (entry: any) => res(entry), (err: any) => rej(err))
  })
  // 若目标不存在，复制到 _doc
  try {
    await resolveURL(DB_DST_PATH)
    pushLog('目标 DB 已存在于 _doc')
  } catch {
    try {
      // 逐个候选源路径尝试
      let srcEntry: any = null
      for (const p of DB_SRC_CANDIDATES) {
        try {
          srcEntry = await resolveURL(p)
          selectedSrcPath.value = p
          pushLog('发现源 DB: ' + p)
          break
        } catch (e) {
          // 尝试下一个
        }
      }
      if (!srcEntry) {
        throw new Error('未找到任何候选源 DB: ' + DB_SRC_CANDIDATES.join(', '))
      }
      const dstDirEntry = await resolveURL(DB_DST_DIR)
      pushLog('开始复制 DB 到 _doc')
      await new Promise<void>((res, rej) => {
        srcEntry.copyTo(dstDirEntry, DB_DST_FILE, () => res(), (err: any) => rej(err))
      })
      pushLog('复制完成: ' + DB_DST_PATH)
    } catch (e) {
      console.warn('复制静态 DB 到 _doc 失败：', e)
      lastError.value = '复制失败: ' + (e && e.message ? e.message : String(e))
      pushLog(lastError.value)
      return false
    }
  }
  // 打开 _doc/static.db
  try {
    // @ts-ignore
    const opened = plus.sqlite.isOpenDatabase({ name: DB_NAME, path: DB_DST_PATH })
    if (!opened) {
      // @ts-ignore
      plus.sqlite.openDatabase({ name: DB_NAME, path: DB_DST_PATH })
      pushLog('已打开 DB: ' + DB_DST_PATH)
    }
    // 获取目标文件大小
    try {
      await new Promise<void>((res, rej) => {
        // @ts-ignore
        plus.io.resolveLocalFileSystemURL(DB_DST_PATH, (entry: any) => {
          entry.getMetadata((meta: any) => {
            dbFileSize.value = meta.size || null
            pushLog('DB 文件大小: ' + (dbFileSize.value ?? '未知'))
            res()
          }, (err: any) => rej(err))
        }, (err: any) => rej(err))
      })
    } catch (e) {
      pushLog('读取 DB 文件大小失败: ' + (e && e.message ? e.message : String(e)))
    }
    dbReady.value = true
    dbOpened.value = true
    return true
  } catch (e) {
    console.warn('打开静态 DB 失败：', e)
    lastError.value = '打开失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
    return false
  }
}

function selectItems(): DbItem[] | null {
  // @ts-ignore
  if (typeof plus === 'undefined' || !plus.sqlite) return null
  if (!dbOpened.value) handleOpenDb()
  // 已在 ensureDbReady 中打开
  try {
    // @ts-ignore
  const rows = plus.sqlite.selectSql({ name: DB_NAME, sql: 'SELECT * FROM items' }) as unknown as Array<Record<string, any>>
  pushLog(`items 查询返回 ${rows?.length || 0} 条`)
  if (rows && rows.length) pushLog('items 第一条: ' + JSON.stringify(rows[0]))
    return (rows || []).map((r) => ({ id: Number(r.id), title: String(r.title), description: String(r.description ?? ''), createdAt: String(r.createdAt ?? '') }))
  } catch (e) {
    console.warn('读取 items 失败：', e)
    lastError.value = '读取 items 失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
    return null
  }
}

function selectUsers(): DbUser[] | null {
  // @ts-ignore
  if (typeof plus === 'undefined' || !plus.sqlite) return null
  if (!dbOpened.value) handleOpenDb()
  // 已在 ensureDbReady 中打开
  try {
    // @ts-ignore
  const rows = plus.sqlite.selectSql({ name: DB_NAME, sql: 'SELECT * FROM users' }) as unknown as Array<Record<string, any>>
  pushLog(`users 查询返回 ${rows?.length || 0} 条`)
  if (rows && rows.length) pushLog('users 第一条: ' + JSON.stringify(rows[0]))
    return (rows || []).map((r) => ({ id: Number(r.id), name: String(r.name), role: String(r.role) }))
  } catch (e) {
    console.warn('读取 users 失败：', e)
    lastError.value = '读取 users 失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
    return null
  }
}

async function loadStaticDb() {
  dbLoading.value = true
  // 轻微延迟，模拟加载体验
  setTimeout(async () => {
    const ok = await ensureDbReady()
    if (ok) {
      const items = selectItems()
      const users = selectUsers()
      if (items) dbItems.value = items
      if (users) dbUsers.value = users
      itemsCount.value = dbItems.value.length
      usersCount.value = dbUsers.value.length
      pushLog(`刷新完成，items=${itemsCount.value}, users=${usersCount.value}`)
    }
    dbLoading.value = false
  }, 200)
}

onMounted(() => {
  // @ts-ignore
  envPlusAvailable.value = typeof plus !== 'undefined'
  pushLog('plus 可用: ' + envPlusAvailable.value)
  loadStaticDb()
})

// 调试操作 - 复制
async function handleCopyDb() {
  // @ts-ignore
  if (typeof plus === 'undefined') return
  lastError.value = null
  dbReady.value = false
  try {
    const resolveURL = (url: string) => new Promise<any>((res, rej) => {
      // @ts-ignore
      plus.io.resolveLocalFileSystemURL(url, (entry: any) => res(entry), (err: any) => rej(err))
    })
    let srcEntry: any = null
    selectedSrcPath.value = null
    for (const p of DB_SRC_CANDIDATES) {
      try {
        srcEntry = await resolveURL(p)
        selectedSrcPath.value = p
        pushLog('手动复制发现源 DB: ' + p)
        break
      } catch (e) {
        // continue
      }
    }
    if (!srcEntry) throw new Error('手动复制未找到源 DB')
    const dstDirEntry = await resolveURL(DB_DST_DIR)
    // 若目标文件已存在，先删除，以免 copyTo 因目标存在报错
    try {
      const dstFileEntry = await resolveURL(DB_DST_PATH)
      await new Promise<void>((res, rej) => {
        dstFileEntry.remove(() => res(), (err: any) => rej(err))
      })
      pushLog('已删除旧目标文件: ' + DB_DST_PATH)
    } catch (e) {
      pushLog('目标文件不存在，无需删除: ' + DB_DST_PATH)
    }
    await new Promise<void>((res, rej) => {
      srcEntry.copyTo(dstDirEntry, DB_DST_FILE, () => res(), (err: any) => rej(err))
    })
    pushLog('手动复制完成: ' + DB_DST_PATH)
  } catch (e) {
    lastError.value = '手动复制失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
  }
}

// 调试操作 - 打开
function handleOpenDb() {
  // @ts-ignore
  if (typeof plus === 'undefined' || !plus.sqlite) return
  try {
    // @ts-ignore
    const opened = plus.sqlite.isOpenDatabase({ name: DB_NAME, path: DB_DST_PATH })
    if (!opened) {
      // @ts-ignore
      plus.sqlite.openDatabase({ name: DB_NAME, path: DB_DST_PATH })
    }
    dbOpened.value = true
    dbReady.value = true
    pushLog('手动打开 DB: ' + DB_DST_PATH)
  } catch (e) {
    lastError.value = '手动打开失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
  }
}

// 调试操作 - 关闭
function handleCloseDb() {
  // @ts-ignore
  if (typeof plus === 'undefined' || !plus.sqlite) return
  try {
    // @ts-ignore
    plus.sqlite.closeDatabase({ name: DB_NAME })
    dbOpened.value = false
  dbReady.value = false
    pushLog('已关闭 DB')
  } catch (e) {
    lastError.value = '关闭失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
  }
}

// 调试操作 - 查询 items
function handleQueryItems() {
  const items = selectItems()
  if (items) {
    dbItems.value = items
    itemsCount.value = items.length
    pushLog(`手动查询 items 成功(${itemsCount.value})`)
  }
}

// 调试操作 - 查询 users
function handleQueryUsers() {
  const users = selectUsers()
  if (users) {
    dbUsers.value = users
    usersCount.value = users.length
    pushLog(`手动查询 users 成功(${usersCount.value})`)
  }
}

// 调试操作 - 刷新
async function handleRefreshDb() {
  await loadStaticDb()
}

// 调试操作 - 列出表
function handleListTables() {
  // @ts-ignore
  if (typeof plus === 'undefined' || !plus.sqlite) return
  if (!dbOpened.value) handleOpenDb()
  try {
    // @ts-ignore
    const rows = plus.sqlite.selectSql({ name: DB_NAME, sql: "SELECT name, type FROM sqlite_master WHERE type='table' ORDER BY name" }) as unknown as Array<Record<string, any>>
    pushLog('sqlite_master 表列表: ' + JSON.stringify(rows))
  } catch (e) {
    lastError.value = '列出表失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
  }
}

// 调试操作 - 计数
function handleCountTables() {
  // @ts-ignore
  if (typeof plus === 'undefined' || !plus.sqlite) return
  if (!dbOpened.value) handleOpenDb()
  try {
    // @ts-ignore
    const itemsCnt = plus.sqlite.selectSql({ name: DB_NAME, sql: 'SELECT COUNT(*) AS c FROM items' }) as unknown as Array<Record<string, any>>
    const usersCnt = plus.sqlite.selectSql({ name: DB_NAME, sql: 'SELECT COUNT(*) AS c FROM users' }) as unknown as Array<Record<string, any>>
    pushLog('items COUNT: ' + JSON.stringify(itemsCnt))
    pushLog('users COUNT: ' + JSON.stringify(usersCnt))
  } catch (e) {
    lastError.value = '计数失败: ' + (e && e.message ? e.message : String(e))
    pushLog(lastError.value)
  }
}
</script>

<style lang="scss" scoped>
//

.page-container {
  padding: 0 20upx;
  padding-top: 100upx;
  position: relative;
  font-size: 15px;
  color: var(--UI-FG-0);
  .logo {
    width: 200upx;
    height: 150px;
  }
  .title {
    font-size: 58upx;
  }
  .enter-area {
    padding-top: 75px;
    width: 87%;
    margin: 0 auto 60upx;
    .box {
      display: flex;
      align-items: center;
      min-height: 100upx;
      color: #000;
      border: 1px solid #eee;
      background-color: #fff;
      padding: 0 20upx;
      margin-bottom: 30upx;
      box-shadow: 0 0 5px rgba(0, 0, 0, 0.1);
      :deep(.wd-text) {
        margin: 0 10upx;
      }
      :deep(.wd-input),
      :deep(.uni-input) {
        flex: 1;
        &::after {
          height: 0;
        }
      }
      .uni-input {
        text-align: left;
        font-size: var(--wot-input-fs, var(--wot-cell-title-fs, 14px));
        height: var(--wot-input-inner-height, 34px);
        color: var(--wot-input-color, #262626);
        .uni-input-placeholder {
          color: var(--wot-input-placeholder-color, #bfbfbf);
        }
      }
    }
    :deep(.sendSMSBtn) {
      margin-left: 20upx;
    }
    :deep(.wd-icon-view),
    :deep(.wd-icon-eye-close) {
      color: #555;
    }
  }
  .btn-area {
    :deep(.login) {
      margin-right: 30px;
    }
    :deep(.wd-button) {
      --wot-button-medium-height: 41px;
      --wot-button-medium-fs: 16px;
      --wot-button-plain-bg-color: transparent;
      min-width: 100px;
      box-shadow: 3px 3px 4px rgba(0, 102, 204, 0.2);
    }
  }
}
.static-db {
  margin-top: 24px;
  padding: 12px;
  background: #fff;
  border: 1px solid #eee;
  border-radius: 8px;
  box-shadow: 0 0 5px rgba(0, 0, 0, 0.05);
  .actions {
    display: flex;
    flex-direction: row;
    flex-wrap: nowrap;
    gap: 8px;
    overflow-x: auto;
    -webkit-overflow-scrolling: touch;
    padding: 6px 0 8px;
    // 让按钮在一排显示，超出可横向滚动
    button,
    .uni-button {
      flex: 0 0 auto;
      font-size: 12px;
      padding: 6px 8px;
      height: 28px;
      line-height: 28px;
    }
  }
  .section-title {
    font-size: 10px;
    font-weight: 600;
    margin-bottom: 8px;
  }
  .hint {
    color: #999;
    font-size: 12px;
    margin-bottom: 8px;
  }
  .block {
    margin-top: 12px;
    .row { padding: 8px 0; border-bottom: 1px solid #f2f2f2; }
    .row:last-child { border-bottom: 0; }
  }
}
</style>
