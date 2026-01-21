<template>
  <PageLayout :navbarShow="false">
    <view class="page-container">
      <!-- 静态本地 DB 展示区域 -->
      <view class="static-db">
        <view class="section-title">本地静态 DB 数据</view>
        <view v-if="dbLoading" class="hint">加载中...</view>
        <view v-else>
          <!-- 调试操作按钮 -->
          <view class="actions">
            <!-- sql.js 方案中，"打开"即加载文件到内存，"复制"则是物理文件操作 -->
            <button @click="handleCopyDb">复制到 _doc</button>
            <button @click="handleOpenDb">加载 DB 到内存</button>
            <button @click="handleCloseDb">关闭/释放内存</button>
            <button @click="handleQueryItems">查询 items</button>
            <button @click="handleQueryUsers">查询 users</button>
            <button @click="handleListTables">列出表</button>
            <button @click="handleCountTables">计数</button>
            <button @click="handleRefreshDb">刷新</button>
          </view>

          <!-- 状态信息 -->
          <view class="status">
            <view class="hint">DB 文件源: static/static.db</view>
            <view>引擎: sql.js (Wasm/JS)</view>
            <view>当前读取路径: {{ currentDbPath || '未加载' }}</view>
            <view>目标路径(_doc): {{ DB_DST_PATH }}</view>
            <view>dbReady: {{ dbReady }}</view>
            <view>dbOpened: {{ dbOpened }}</view>
            <view>dbSize(bytes): {{ dbFileSize }}</view>
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
import { ref, onMounted } from 'vue'
import initSqlJs from 'sql.js' // 引入 sql.js

// 假设这些引入在你的项目中依然有效，保留原样
import { useUserStore } from '@/store/user'
// ... 其他原有的 import
import { useRouter } from '@/plugin/uni-mini-router'

// 定义类型
type DbItem = { id: number; title: string; description?: string; createdAt: string }
type DbUser = { id: number; name: string; role: string }

// 路由和 Store 初始化 (保留原有逻辑)
const router = useRouter()
const userStore = useUserStore()
const toast = useToast()
// ... 其余原有的登录逻辑变量 (loading, userName, password 等) 保持不变
// 注意：为了节省篇幅，这里省略了未修改的登录相关变量和方法，实际使用请保留它们
// ...

/**
 * sql.js 核心变量
 */
let SQL: any = null // sql.js 的构造函数类
let db: any = null  // 当前数据库实例

// 配置路径
const DB_DST_DIR = '_doc/'
const DB_DST_FILE = 'static.db'
const DB_DST_PATH = `${DB_DST_DIR}${DB_DST_FILE}`

// 状态变量
const dbItems = ref<DbItem[]>([])
const dbUsers = ref<DbUser[]>([])
const dbLoading = ref(false)
const dbReady = ref(false) // sql.js 引擎是否准备好
const dbOpened = ref(false) // 数据库文件是否加载到内存
const itemsCount = ref(0)
const usersCount = ref(0)
const lastError = ref<string | null>(null)
const debugLogs = ref<string[]>([])
const currentDbPath = ref<string | null>(null) // 实际读取的文件路径
const dbFileSize = ref<number | null>(null)

function pushLog(msg: any) {
  const s = typeof msg === 'string' ? msg : JSON.stringify(msg)
  const line = `[${new Date().toLocaleString()}] ${s}`
  debugLogs.value.unshift(line)
  if (debugLogs.value.length > 200) debugLogs.value.pop()
  console.log(line)
}

/**
 * 核心：读取文件为 ArrayBuffer
 * 兼容 H5 (uni.request) 和 App (plus.io)
 */
async function readFileAsBuffer(path: string): Promise<ArrayBuffer> {
  // #ifdef APP-PLUS
  return new Promise((resolve, reject) => {
    plus.io.resolveLocalFileSystemURL(path, (entry: any) => {
      entry.file((file: any) => {
        const reader = new plus.io.FileReader()
        reader.onload = (e: any) => {
          resolve(e.target.result)
        }
        reader.onerror = (err: any) => reject(err)
        reader.readAsArrayBuffer(file)
      }, reject)
    }, reject)
  })
  // #endif

  // #ifdef H5
  // H5 环境下，_doc 路径无法直接访问，只能读取 static 目录
  // 如果传入的是 _doc 开头，这里会失败，需要逻辑处理
  // 为了演示，我们假设 H5 下只读取 static 目录的文件
  return new Promise((resolve, reject) => {
    uni.request({
      url: path,
      method: 'GET',
      responseType: 'arraybuffer',
      success: (res: any) => {
        resolve(res.data)
      },
      fail: (err) => {
        reject(err)
      }
    })
  })
  // #endif
  
  // #ifndef APP-PLUS || H5
  return Promise.reject(new Error('当前平台不支持'))
  // #endif
}

/**
 * 初始化 sql.js 引擎
 */
async function initEngine() {
  if (SQL) return true
  try {
    pushLog('正在初始化 sql.js 引擎...')
    // 指定 wasm 文件位置。
    // H5: 放在 static 目录
    // APP: 最好把 sql-wasm.wasm 放在 static 目录，或者内嵌
    SQL = await initSqlJs({
      locateFile: (file: string) => {
        // 这里的路径指向你的 static 目录下的 wasm 文件
        // 如果是 App 端，使用相对路径指向 static 即可
        return '/static/sql-wasm.wasm' 
      }
    })
    pushLog('sql.js 引擎初始化成功')
    return true
  } catch (e) {
    pushLog('引擎初始化失败: ' + JSON.stringify(e))
    lastError.value = '引擎初始化失败: ' + String(e)
    return false
  }
}

/**
 * 确保数据库就绪 (物理文件层面)
 * 尝试复制 static.db 到 _doc，优先使用 _doc 的文件（如果需要写操作的话，但 sql.js 主要是内存读）
 * 这里保持原逻辑：先找 _doc，没有则从 _www 复制
 */
async function ensureDbFileReady(): Promise<string> {
  const checkFile = async (path: string) => {
    // #ifdef APP-PLUS
    try {
      await new Promise((resolve, reject) => {
        plus.io.resolveLocalFileSystemURL(path, resolve, reject)
      })
      return true
    } catch (e) {
      return false
    }
    // #endif
    // H5 简单判断，直接返回 false 模拟不存在，或者用 fetch HEAD 请求
    return false 
  }

  // 1. 检查 _doc 下是否存在
  const existsInDoc = await checkFile(DB_DST_PATH)
  
  // APP 环境逻辑
  // #ifdef APP-PLUS
  if (existsInDoc) {
    pushLog(`使用文档目录缓存: ${DB_DST_PATH}`)
    return DB_DST_PATH
  }

  // 2. 不存在，从 _www/static/static.db 复制
  const srcPath = '_www/static/static.db'
  const srcExists = await checkFile(srcPath)
  if (!srcExists) {
    throw new Error('源文件不存在: ' + srcPath + '，请确保 static.db 在 static 目录下')
  }

  pushLog(`开始复制 ${srcPath} -> ${DB_DST_PATH}`)
  
  // 复制逻辑
  const srcEntry = await new Promise((res, rej) => plus.io.resolveLocalFileSystemURL(srcPath, res, rej)) as any
  const dstDir = await new Promise((res, rej) => plus.io.resolveLocalFileSystemURL(DB_DST_DIR, res, rej)) as any
  
  await new Promise<void>((res, rej) => {
    srcEntry.copyTo(dstDir, DB_DST_FILE, () => res(), (err: any) => rej(err))
  })
  
  pushLog('复制完成')
  return DB_DST_PATH
  // #endif

  // H5 逻辑
  // #ifdef H5
  // H5 只能读取 static 目录的文件，无法写入 _doc
  const h5Path = '/static/static.db'
  pushLog(`H5 环境使用 static 资源: ${h5Path}`)
  return h5Path
  // #endif
}

/**
 * 辅助函数：执行 SQL 并返回对象数组
 */
function execToObject(sqlStr: string) {
  if (!db || !SQL) return []
  try {
    const stmt = db.prepare(sqlStr)
    const result = []
    while(stmt.step()) {
      const row = stmt.getAsObject()
      result.push(row)
    }
    stmt.free()
    return result
  } catch (e) {
    pushLog(`SQL 执行错误: ${sqlStr} | ${e}`)
    return []
  }
}

/**
 * 加载数据库
 */
async function loadStaticDb() {
  dbLoading.value = true
  try {
    // 1. 初始化引擎
    const engineOk = await initEngine()
    if (!engineOk) throw new Error('引擎初始化失败')

    // 2. 准备文件路径 (物理层面)
    const targetPath = await ensureDbFileReady()
    currentDbPath.value = targetPath

    // 3. 读取文件到内存
    pushLog('正在读取二进制文件...')
    const buffer = await readFileAsBuffer(targetPath)
    dbFileSize.value = buffer.byteLength
    pushLog(`文件读取成功, 大小: ${buffer.byteLength}`)

    // 4. 创建数据库实例 (加载到内存)
    db = new SQL.Database(buffer)
    dbOpened.value = true
    dbReady.value = true
    pushLog('数据库已加载到内存')

    // 5. 初始查询
    handleQueryItems()
    handleQueryUsers()

  } catch (e: any) {
    console.error(e)
    lastError.value = String(e)
    pushLog('加载失败: ' + String(e))
    dbReady.value = false
    dbOpened.value = false
  } finally {
    dbLoading.value = false
  }
}

onMounted(() => {
  pushLog('组件挂载，开始加载')
  loadStaticDb()
})

// --- 调试操作函数 ---

async function handleCopyDb() {
  // #ifdef APP-PLUS
  try {
    lastError.value = null
    // 强制删除旧文件重新复制
    const dstDir = await new Promise((res, rej) => plus.io.resolveLocalFileSystemURL(DB_DST_DIR, res, rej)) as any
    try {
      const oldFile = await new Promise((res, rej) => plus.io.resolveLocalFileSystemURL(DB_DST_PATH, res, rej)) as any
      await new Promise((res, rej) => oldFile.remove(() => res(), rej))
      pushLog('旧文件已删除')
    } catch(e) {}
    
    const srcPath = '_www/static/static.db'
    const srcEntry = await new Promise((res, rej) => plus.io.resolveLocalFileSystemURL(srcPath, res, rej)) as any
    await new Promise<void>((res, rej) => {
      srcEntry.copyTo(dstDir, DB_DST_FILE, () => res(), (err: any) => rej(err))
    })
    pushLog('手动复制完成')
    // 复制后重新加载
    await loadStaticDb()
  } catch (e: any) {
    lastError.value = '复制失败: ' + String(e)
    pushLog(lastError.value)
  }
  // #endif
  
  // #ifndef APP-PLUS
  toast.warning('仅 App 端支持文件复制')
  // #endif
}

async function handleOpenDb() {
  if (dbOpened.value) {
    toast.warning('数据库已打开')
    return
  }
  await loadStaticDb()
}

function handleCloseDb() {
  if (db) {
    db.close()
    db = null
    dbOpened.value = false
    dbReady.value = false
    pushLog('数据库连接已关闭，内存已释放')
  }
}

function handleQueryItems() {
  if (!dbOpened.value) return
  const rows = execToObject('SELECT * FROM items') as any[]
  // 类型转换适配
  dbItems.value = rows.map(r => ({
    id: Number(r.id),
    title: String(r.title),
    description: String(r.description ?? ''),
    createdAt: String(r.createdAt ?? '')
  }))
  itemsCount.value = dbItems.value.length
  pushLog(`查询 items 完成: ${itemsCount.value} 条`)
}

function handleQueryUsers() {
  if (!dbOpened.value) return
  const rows = execToObject('SELECT * FROM users') as any[]
  dbUsers.value = rows.map(r => ({
    id: Number(r.id),
    name: String(r.name),
    role: String(r.role)
  }))
  usersCount.value = dbUsers.value.length
  pushLog(`查询 users 完成: ${usersCount.value} 条`)
}

function handleRefreshDb() {
  loadStaticDb()
}

function handleListTables() {
  if (!dbOpened.value) return
  const rows = execToObject("SELECT name, type FROM sqlite_master WHERE type='table' ORDER BY name") as any[]
  pushLog('表列表: ' + JSON.stringify(rows))
}

function handleCountTables() {
  if (!dbOpened.value) return
  const itemsCnt = execToObject('SELECT COUNT(*) AS c FROM items')
  const usersCnt = execToObject('SELECT COUNT(*) AS c FROM users')
  pushLog('items count: ' + JSON.stringify(itemsCnt))
  pushLog('users count: ' + JSON.stringify(usersCnt))
}

</script>
