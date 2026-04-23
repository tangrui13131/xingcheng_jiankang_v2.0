<template>
  <div class="user-layout">
    <div class="user-sidebar">
      <div class="logo-container">
        <img :src="logoSrc" class="user-logo" alt="logo" />
        <div class="logo-text">星辰 · 用户端</div>
      </div>
      <div class="menu">
        <div class="menu-item active">首页</div>
        <div class="menu-item" @click="router.push('/health-data')">健康看板</div>
        <div class="menu-item" @click="router.push('/points-mall')">积分商城</div>
        <div class="menu-item" @click="router.push('/profile')">个人中心</div>
        <div class="menu-item" @click="handleLogout">退出登录</div>
      </div>
    </div>
    <div class="user-content">
      <div class="user-header">
        <div class="welcome">你好，{{ username }}！祝你今天愉快。</div>
      </div>
      <div class="dashboard">
        <el-row :gutter="20">
          <el-col :span="8">
            <el-card class="stats-card">
              <template #header>我的数据</template>
              <div class="stats-num">128</div>
              <div class="stats-desc">本月记录总数</div>
            </el-card>
          </el-col>
          <el-col :span="8">
            <el-card class="stats-card accent">
              <template #header>待办事项</template>
              <div class="stats-num">5</div>
              <div class="stats-desc">需要你关注的事项</div>
            </el-card>
          </el-col>
          <el-col :span="8">
            <el-card class="stats-card success">
              <template #header>已完成</template>
              <div class="stats-num">96%</div>
              <div class="stats-desc">本月目标进度</div>
            </el-card>
          </el-col>
        </el-row>

        <el-row :gutter="20" class="mt-20">
          <el-col :span="12">
            <!-- 游戏化健康打卡组件 -->
            <DailyHealthCheckIn />
          </el-col>
          <el-col :span="12">
            <el-card class="content-card no-margin">
              <template #header>
                <div class="card-header">
                  <span>最新公告</span>
                  <el-button type="primary" link>查看更多</el-button>
                </div>
              </template>
              <div class="news-list">
                <div class="news-item">
                  <span class="news-title">系统升级公告</span>
                  <span class="news-date">2024-03-20</span>
                </div>
                <div class="news-item">
                  <span class="news-title">温馨提示：保持良好心情哦</span>
                  <span class="news-date">2024-03-19</span>
                </div>
              </div>
            </el-card>

            <!-- 屏幕使用时长与久坐提醒 -->
            <el-card shadow="hover" class="screen-card mt-16">
              <template #header>
                <div class="card-header">
                  <span>👀 屏幕关注眼监控</span>
                  <el-button type="info" link size="small" class="demo-btn" @click="screenSeconds = 3595">[演示测试]</el-button>
                </div>
              </template>
              <div class="screen-dashboard">
                <el-progress
                  type="dashboard"
                  :percentage="progressPercentage"
                  :stroke-width="10"
                  :color="progressColor"
                >
                  <template #default>
                    <div class="screen-time-display">
                      <span class="time-value">{{ formattedTime }}</span>
                      <span class="time-label">连续用屏</span>
                    </div>
                  </template>
                </el-progress>
                <p class="screen-tip">每 60 分钟建议休息一次，保护颈椎与视力</p>
              </div>
            </el-card>
          </el-col>
        </el-row>

        <!-- AI私人健康顾问模块 -->
        <el-card class="ai-advisor-card mt-20">
          <template #header>
            <div class="card-header">
              <span>🤖AI私人健康顾问//我的独特周报</span>
            </div>
          </template>

          <!-- 初始状态：唤醒按钮 -->
          <div v-if="!aiGenerated" class="ai-advisor-init">
            <el-button
              type="primary"
              size="large"
              plain
              :loading="aiLoading"
              @click="handleGenerateAdvice"
            >
              ✨点击生成本周专属健康建议
            </el-button>
            <p class="ai-hint">基于你最近7天的打卡数据生成</p>
          </div>

          <!-- 结果展示区（默认隐藏） -->
          <div v-else class="ai-advisor-result">
            <div class="ai-result-content" v-html="aiAdviceDisplayHtml"></div>
            <div class="ai-result-footer">
              <el-button type="primary" link :disabled="aiTyping" @click="handleRegenerate">
                🔄 重新生成
              </el-button>
            </div>
          </div>
        </el-card>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted, watch } from 'vue'
import { useRouter } from 'vue-router'
import request from '@/api/request'
import { ElMessage, ElMessageBox } from 'element-plus'
import logoSrc from '@/assets/logo.png'
import DailyHealthCheckIn from '@/components/DailyHealthCheckIn.vue'

const router = useRouter()
const username = ref('用户')

// AI健康顾问相关状态
const aiLoading = ref(false)   // 按钮 loading（模拟思考中）
const aiGenerated = ref(false) // 是否切换到结果展示区
const aiTyping = ref(false)    // 打字机是否正在输出
const aiAdviceText = ref('')   // 已输出到界面的文本（逐字累加）
let aiTimer = null             // 打字机定时器句柄，用于清除

// Mock 数据：完整 AI 回复文本（\n 为真实换行符）
const AI_MOCK_TEXT = `根据你最近 7 天的打卡数据，你的睡眠质量整体一般，睡眠平均时间为 5.8 小时，其中有 3 天睡眠不足 5 小时；颈椎状态评分平均为 3.2 分（满分 5 分），有明显的疲劳；每日饮水量平均为 1200ml，低于推荐的 1500ml 标准。

建议：
1. 每天努力在 23 点前入睡，保证 7-8 小时睡眠
2. 每工作 1 小时，起身活动 5 分钟，做颈椎拉伸
3. 每天分多次饮水，每次 200ml 左右，目标达到 1500ml
4. 可以在办公桌上放一个水杯，提醒自己喝水`

/**
 * 将原始文本转成 HTML（\n → <br>），并在打字机输出期间追加闪烁光标
 */
const aiAdviceDisplayHtml = computed(() => {
  // 转义 HTML 特殊字符，防止 XSS
  const escaped = aiAdviceText.value
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/\n/g, '<br>')
  // 打字机输出期间追加光标
  return escaped + (aiTyping.value ? '<span class="typing-cursor">|</span>' : '')
})

/** 启动打字机：每 30ms 输出一个字符 */
const startTypewriter = (fullText) => {
  let index = 0
  aiAdviceText.value = ''
  aiTyping.value = true

  aiTimer = setInterval(() => {
    if (index < fullText.length) {
      aiAdviceText.value += fullText[index]
      index++
    } else {
      // 全部输出完毕，停止打字机
      clearInterval(aiTimer)
      aiTimer = null
      aiTyping.value = false
    }
  }, 30)
}

/** 生成AI健康建议 */
const handleGenerateAdvice = async () => {
  aiLoading.value = true
  try {
    // TODO: 对接后端 /ai/chat/send 接口，传入用户最近7天打卡数据
    // const res = await request.post('/ai/chat/send', { message: '请根据以下打卡数据生成健康建议...' })
    // const fullText = res.data

    // 模拟 AI 思考延迟 1.5s
    await new Promise(resolve => setTimeout(resolve, 1500))
    const fullText = AI_MOCK_TEXT

    // 切换到结果区，启动打字机
    aiGenerated.value = true
    startTypewriter(fullText)
  } catch (error) {
    console.error(error)
    ElMessage.error('生成失败，请稍后重试')
  } finally {
    aiLoading.value = false
  }
}

/** 重新生成：清除打字机 → 重置状态 → 重新触发 */
const handleRegenerate = () => {
  if (aiTimer) {
    clearInterval(aiTimer)
    aiTimer = null
  }
  aiGenerated.value = false
  aiAdviceText.value = ''
  aiTyping.value = false
  handleGenerateAdvice()
}

// 组件卸载时清除定时器，防止内存泄漏
onUnmounted(() => {
  if (aiTimer) clearInterval(aiTimer)
  if (screenTimer) clearInterval(screenTimer)
})

// ─── 屏幕使用时长与久坐提醒 ──────────────────────────────────
const screenSeconds = ref(0)
let screenTimer = null

/** 格式化为 HH:mm:ss */
const formattedTime = computed(() => {
  const h = Math.floor(screenSeconds.value / 3600)
  const m = Math.floor((screenSeconds.value % 3600) / 60)
  const s = screenSeconds.value % 60
  return `${String(h).padStart(2, '0')}:${String(m).padStart(2, '0')}:${String(s).padStart(2, '0')}`
})

/** 环形进度：60 分钟 (3600s) = 100% */
const progressPercentage = computed(() => {
  return Math.min(Math.round((screenSeconds.value / 3600) * 100), 100)
})

/** 进度条颜色渐变：绿 → 橙 → 红 */
const progressColor = computed(() => {
  if (progressPercentage.value < 50) return '#67C23A'
  if (progressPercentage.value < 80) return '#E6A23C'
  return '#F56C6C'
})

/** 监听秒数，满 60 分钟弹窗提醒 */
watch(screenSeconds, (val) => {
  if (val === 3600) {
    ElMessageBox.alert(
      '您已经连续面对屏幕 1 小时了！为了您的颈椎和视力，请站起来倒杯水，眺望远方 20 秒。',
      '🔔 护眼与久坐提醒',
      {
        confirmButtonText: '我这就去休息',
        type: 'warning',
        center: true
      }
    )
  }
})

onMounted(async () => {
  // 启动屏幕使用计时器（每秒 +1）
  screenTimer = setInterval(() => {
    screenSeconds.value++
  }, 1000)

  try {
    const res = await request.get('/getInfo')
    username.value = res.user.nickName || res.user.userName
  } catch (error) {
    console.error(error)
  }
})

const handleLogout = async () => {
  try {
    await request.post('/logout')
    localStorage.removeItem('token')
    ElMessage.success('已安全退出')
    router.push('/login')
  } catch (error) {
    console.error(error)
  }
}
</script>

<style scoped lang="scss">
.user-layout {
  display: flex;
  height: 100vh;
}

.user-sidebar {
  width: 240px;
  background-color: var(--bg-1);
  border-right: 1px solid var(--line);
  padding: 40px 20px;
  display: flex;
  flex-direction: column;
  
  .logo-container {
    display: flex;
    flex-direction: column;
    align-items: center;
    margin-bottom: 60px;
    
    .user-logo {
      width: 64px;
      height: 64px;
      margin-bottom: 12px;
      object-fit: contain;
    }
    
    .logo-text {
      color: var(--primary);
      font-size: 20px;
      font-weight: bold;
      text-align: center;
    }
  }
  
  .menu-item {
    padding: 16px 24px;
    margin-bottom: 12px;
    border-radius: 12px;
    cursor: pointer;
    color: var(--text-2);
    font-weight: 500;
    transition: all 0.3s;
    
    &:hover {
      background-color: var(--primary-light);
      color: var(--primary);
    }
    
    &.active {
      background-color: var(--primary);
      color: #fff;
    }
  }
}

.user-content {
  flex: 1;
  background-color: var(--bg-0);
  display: flex;
  flex-direction: column;
}

.user-header {
  height: 80px;
  padding: 0 40px;
  display: flex;
  align-items: center;
  background-color: var(--bg-1);
  border-bottom: 1px solid var(--line);
  
  .welcome {
    font-size: 18px;
    color: var(--text-1);
    font-weight: 600;
  }
}

.dashboard {
  padding: 40px;
  overflow-y: auto;
}

.stats-card {
  text-align: center;
  margin-bottom: 24px;
  
  .stats-num {
    font-size: 36px;
    font-weight: bold;
    color: var(--primary);
    margin: 12px 0;
  }
  
  .stats-desc {
    color: var(--text-2);
    font-size: 14px;
  }
  
  &.accent .stats-num { color: var(--accent); }
  &.success .stats-num { color: var(--success); }
}

.content-card {
  margin-top: 20px;
  
  &.no-margin {
    margin-top: 0;
  }
  
  .card-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    font-weight: 600;
  }
}

.mt-20 {
  margin-top: 20px;
}

.ai-advisor-card {
  border-radius: 12px;

  .card-header {
    font-weight: 600;
    font-size: 16px;
  }
}

.ai-advisor-init {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 40px 20px 32px;
  gap: 16px;

  .el-button {
    font-size: 16px;
    padding: 14px 36px;
    border-radius: 24px;
  }
}

.ai-hint {
  margin: 0;
  font-size: 13px;
  color: #909399;
}

.ai-advisor-result {
  padding: 4px 0 0;
}

.ai-result-content {
  background-color: #f5f7fa;
  border-radius: 10px;
  padding: 24px 28px;
  color: #303133;
  font-size: 15px;
  line-height: 1.8;
  min-height: 80px;
}

/* 打字机光标：竖线闪烁动画 */
:deep(.typing-cursor) {
  display: inline-block;
  width: 2px;
  height: 1em;
  background-color: #409eff;
  margin-left: 2px;
  vertical-align: text-bottom;
  animation: cursor-blink 0.8s step-end infinite;
}

@keyframes cursor-blink {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0; }
}

.ai-result-footer {
  display: flex;
  justify-content: flex-end;
  margin-top: 12px;
}

.news-list {
  .news-item {
    display: flex;
    justify-content: space-between;
    padding: 16px 0;
    border-bottom: 1px solid var(--line);
    
    &:last-child {
      border-bottom: none;
    }
    
    .news-title {
      color: var(--text-1);
    }
    
    .news-date {
      color: var(--text-2);
      font-size: 14px;
    }
  }
}

/* ── 屏幕监控卡片 ── */
.mt-16 {
  margin-top: 16px;
}

.screen-card {
  border-radius: 12px;

  .demo-btn {
    font-size: 11px;
    color: #c0c4cc;
    padding: 0;
  }
}

.screen-dashboard {
  display: flex;
  flex-direction: column;
  align-items: center;
  padding: 12px 0 4px;
}

.screen-time-display {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;

  .time-value {
    font-size: 22px;
    font-weight: 700;
    color: #303133;
    font-variant-numeric: tabular-nums;
    letter-spacing: 1px;
  }

  .time-label {
    font-size: 12px;
    color: #909399;
  }
}

.screen-tip {
  margin: 12px 0 0;
  font-size: 12px;
  color: #909399;
  text-align: center;
}
</style>