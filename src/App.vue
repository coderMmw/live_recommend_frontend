<template>
  <div class="app-container">
    <!-- Notification System -->
    <TransitionGroup name="toast">
      <div v-for="msg in messages" :key="msg.id" :class="['toast', msg.type]">
        <span class="toast-icon">{{ msg.type === 'success' ? '✅' : '❌' }}</span>
        {{ msg.text }}
      </div>
    </TransitionGroup>

    <header class="header">
      <div class="header-content">
        <h1 class="logo-text">24节气·AI食疗</h1>
        <div v-if="user" class="user-panel">
          <span class="user-name">👋 你好, {{ user.username }}</span>
          <button @click="logout" class="btn-text">退出</button>
        </div>
        <div v-else class="user-panel">
          <button @click="openAuth" class="btn-primary-sm">登录 / 注册</button>
        </div>
      </div>
    </header>

    <main class="content">
      <div class="main-layout">
        <!-- Sidebar: Category Selection -->
        <aside :class="['sidebar', { collapsed: sidebarCollapsed }]">
          <div class="sidebar-toggle" @click="sidebarCollapsed = !sidebarCollapsed">
            {{ sidebarCollapsed ? '➡️' : '⬅️' }}
          </div>
          
          <div class="sidebar-inner" v-show="!sidebarCollapsed">
            <div class="tree-tabs">
              <button 
                :class="['tree-tab', { active: currentTreeType === 'recipe' }]" 
                @click="currentTreeType = 'recipe'"
              >菜谱分类</button>
              <button 
                :class="['tree-tab', { active: currentTreeType === 'knowledge' }]" 
                @click="currentTreeType = 'knowledge'"
              >烹饪知识</button>
            </div>
            
            <div class="category-scroll">
              <div 
                v-for="cat in categoryTrees[currentTreeType]" 
                :key="cat.code" 
                class="category-group"
              >
                <div class="parent-cat" @click="toggleGroup(cat.code)">
                  <span>{{ cat.name }}</span>
                  <span class="group-arrow">{{ expandedGroups.includes(cat.code) ? '▼' : '▶' }}</span>
                </div>
                <div 
                  :class="['children-cats', { 'one-row': !expandedGroups.includes(cat.code) }]"
                >
                  <span 
                    v-for="sub in cat.children" 
                    :key="sub.code"
                    :class="['cat-chip', { active: selectedCategory?.code === sub.code }]"
                    @click="selectCategory(sub)"
                  >
                    {{ sub.name }}
                  </span>
                </div>
              </div>
            </div>
          </div>
          
          <!-- Collapsed state view (just icons/first column) -->
          <div class="sidebar-collapsed-icons" v-show="sidebarCollapsed">
            <div 
              v-for="cat in categoryTrees[currentTreeType]" 
              :key="cat.code" 
              class="collapsed-icon-btn"
              @click="sidebarCollapsed = false; toggleGroup(cat.code, true)"
              :title="cat.name"
            >
              {{ cat.name.substring(0, 1) }}
            </div>
          </div>
        </aside>

        <!-- Main Content Area -->
        <div class="main-view">
          <!-- 日期+天气信息栏 -->
          <section class="info-bar">
            <!-- 左侧：精简万年历 -->
            <div class="mini-calendar">
              <div class="mini-cal-day">{{ calendarInfo.day }}</div>
              <div class="mini-cal-info">
                <div class="mini-cal-date">{{ calendarInfo.year }}/{{ calendarInfo.month }} {{ calendarInfo.weekday }}</div>
                <div class="mini-cal-lunar">{{ calendarInfo.lunar }} · {{ calendarInfo.lunarYear }}</div>
              </div>
              <div class="mini-cal-term" v-if="calendarInfo.nextTerm">
                距<strong>{{ calendarInfo.nextTerm }}</strong>{{ calendarInfo.nextTermDays }}天
              </div>
            </div>

            <!-- 中间：宜忌 -->
            <div class="mini-yiji">
              <div class="yiji-row">
                <span class="yi-tag">宜</span>
                <span class="yiji-text">{{ calendarInfo.yi.slice(0,4).join(' · ') }}</span>
              </div>
              <div class="yiji-row">
                <span class="ji-tag">忌</span>
                <span class="yiji-text">{{ calendarInfo.ji.slice(0,4).join(' · ') }}</span>
              </div>
            </div>

            <!-- 右侧：天气 -->
            <div class="weather-box">
              <div class="weather-icon">{{ weather.icon }}</div>
              <div class="weather-info">
                <div class="weather-temp">{{ weather.temp }}°C</div>
                <div class="weather-desc">{{ weather.desc }}</div>
                <div class="weather-controls">
                  <select v-model="selectedCity" @change="changeCity" class="city-select">
                    <option value="ip">自动定位</option>
                    <option value="Beijing">北京</option>
                    <option value="Shanghai">上海</option>
                    <option value="Guangzhou">广州</option>
                    <option value="Shenzhen">深圳</option>
                    <option value="Hangzhou">杭州</option>
                    <option value="Nanjing">南京</option>
                    <option value="Wuhan">武汉</option>
                    <option value="Chengdu">成都</option>
                    <option value="Xi'an">西安</option>
                    <option value="Chongqing">重庆</option>
                  </select>
                  <div class="weather-city">{{ weather.city }}</div>
                </div>
              </div>
            </div>
          </section>

          <!-- Solar Term Info Card -->
          <section v-if="solarTerm" class="main-card term-highlight">
            <div class="term-badge">今日节气</div>
            <div class="term-title-row">
              <div class="season-circle">{{ solarTerm.termSeason }}</div>
              <h2 class="term-name">{{ solarTerm.name }}</h2>
            </div>
            
            <div class="term-details">
              <div class="detail-box">
                <div class="box-label">养生要点</div>
                <div class="box-value">{{ solarTerm.healthTip }}</div>
              </div>
              <div class="detail-box">
                <div class="box-label">饮食建议</div>
                <div class="box-value">{{ solarTerm.dietarySuggestion }}</div>
              </div>
              <div class="detail-box ingredients-box">
                <div class="box-label">时令食材</div>
                <div class="box-value highlight-text">{{ solarTerm.recommendedIngredients }}</div>
              </div>
            </div>
          </section>

          <!-- Search Section -->
          <section class="search-area">
            <div class="search-container">
              <input 
                v-model="searchKeyword" 
                type="text" 
                class="search-input" 
                placeholder="🔍 搜索菜名或功效..."
              />
            </div>
          </section>

          <!-- Recommendation Section -->
          <section class="recommendation-area">
            <div class="section-title">
              <h3 v-if="searchKeyword">🔍 搜索结果 "{{ searchKeyword }}"</h3>
              <h3 v-else-if="selectedCategory">📂 分类: {{ selectedCategory.name }}</h3>
              <h3 v-else>🍽️ {{ solarTerm?.name }} · 专属食疗方案</h3>
              <span v-if="!user && !searchKeyword && !selectedCategory" class="tag-outline">需登录解锁</span>
            </div>

            <!-- 搜索或分类结果 -->
            <div v-if="searchKeyword || selectedCategory" class="search-results">
              <div v-if="filteredDishes.length" class="dish-grid">
                <div 
                  v-for="item in filteredDishes" 
                  :key="item.id" 
                  :class="['dish-card', { 'knowledge-card': item.isKnowledge }]"
                >
                  <div class="dish-header-tags">
                    <span class="type-tag">{{ item.dishType || '推荐' }}</span>
                    <span class="dish-tag">{{ item.categoryName || item.category }}</span>
                  </div>
                  <h4>{{ item.name }}</h4>
                  
                  <template v-if="item.isKnowledge">
                    <p class="dish-desc knowledge-summary">{{ item.summary || item.nutritionalEfficacy }}</p>
                    <div v-if="item.keyPoints" class="card-key-points">
                      <span v-for="(p, i) in item.keyPoints.split('\n').slice(0,2)" :key="i" class="mini-point">
                        • {{ p.length > 20 ? p.substring(0,20)+'...' : p }}
                      </span>
                    </div>
                  </template>
                  <template v-else>
                    <p class="dish-desc">{{ item.nutritionalEfficacy }}</p>
                  </template>
                  
                  <button @click="showDetail(item)" class="btn-outline-sm">
                    {{ item.isKnowledge ? '阅读全文' : '查看详情' }}
                  </button>
                </div>
              </div>
              <div v-else-if="(searchKeyword || selectedCategory) && !loadingSearch" class="empty-state">
                <p>未找到相关{{ currentTreeType === 'knowledge' ? '知识' : '菜谱' }}</p>
              </div>
              <div v-else class="loading-state">
                <p>正在获取数据...</p>
              </div>
            </div>

            <!-- 个人推荐 (非搜索/分类模式) -->
            <div v-else-if="user">
              <div v-if="recommendations.length" class="dish-grid">
                <div v-for="dish in recommendations" :key="dish.id" class="dish-card">
                  <div class="dish-header-tags">
                    <span class="type-tag">{{ dish.dishType || '推荐' }}</span>
                    <span class="dish-tag">{{ dish.categoryName || dish.category }}</span>
                  </div>
                  <h4>{{ dish.name }}</h4>
                  <p class="dish-desc">{{ dish.nutritionalEfficacy }}</p>
                  <button @click="showDetail(dish)" class="btn-outline-sm">查看详情</button>
                </div>
              </div>
              <div v-else-if="!loading" class="empty-state">
                <p>正在为您计算最匹配的食疗方案...</p>
              </div>
            </div>

            <!-- Guest View (非搜索/分类模式) -->
            <div v-else class="locked-state">
              <div class="locked-mask">
                <div class="fake-grid">
                  <div v-for="i in 3" :key="i" class="fake-card"></div>
                </div>
              </div>
              <div class="locked-content">
                <div class="lock-icon">🔒</div>
                <p>已为您匹配 <strong>10+</strong> 道 {{ solarTerm?.name }} 节气养生菜谱</p>
                <button @click="openAuth" class="btn-primary">立即登录解锁</button>
              </div>
            </div>
          </section>
        </div>
      </div>

      <div v-if="loading && !solarTerm" class="loading-full">加载中...</div>
    </main>

    <!-- Auth Modal -->
    <div v-if="showAuth" class="modal-overlay" @click="showAuth = false">
      <div class="modal-content auth-card" @click.stop>
        <div class="auth-header">
          <h2>{{ isLogin ? '欢迎回来' : '开启健康生活' }}</h2>
          <p>{{ isLogin ? '请登录您的账号' : '创建一个新账号' }}</p>
        </div>
        
        <div class="auth-body">
          <div class="input-group">
            <input v-model="authForm.username" type="text" placeholder="用户名" />
          </div>
          <div class="input-group password-group">
            <input v-model="authForm.password" :type="showPassword ? 'text' : 'password'" placeholder="密码" />
            <span class="password-toggle" @click="showPassword = !showPassword">
              {{ showPassword ? '👁️' : '🙈' }}
            </span>
          </div>
          <div v-if="!isLogin" class="input-group password-group">
            <input v-model="authForm.confirmPassword" :type="showPassword ? 'text' : 'password'" placeholder="确认密码" />
          </div>
          
          <button @click="handleAuth" class="btn-primary-block" :disabled="authLoading">
            {{ authLoading ? '处理中...' : (isLogin ? '登录' : '注册并登录') }}
          </button>
          
          <div class="auth-footer">
            <span @click="toggleAuthMode">{{ isLogin ? '没有账号？立即注册' : '已有账号？返回登录' }}</span>
          </div>
        </div>
      </div>
    </div>

    <!-- Detail Modal -->
    <div v-if="selectedDish" class="modal-overlay" @click="selectedDish = null">
      <div class="modal-content detail-card" @click.stop>
        <div class="detail-header">
          <h2>{{ selectedDish.name || selectedDish.title }}</h2>
          <button class="close-btn" @click="selectedDish = null">&times;</button>
        </div>
        <div class="detail-scroll">
          <template v-if="selectedDish.isKnowledge">
            <div v-if="selectedDish.summary" class="detail-section summary-box">
              <p class="summary-text"><strong>摘要：</strong>{{ selectedDish.summary }}</p>
            </div>

            <div v-if="selectedDish.keyPoints" class="detail-section key-points-box">
              <h5>【核心要点】</h5>
              <ul class="points-list">
                <li v-for="(point, idx) in selectedDish.keyPoints.split('\n')" :key="idx">{{ point }}</li>
              </ul>
            </div>

            <div class="detail-section">
              <h5>【详细正文】</h5>
              <div class="steps-box knowledge-content">{{ selectedDish.content }}</div>
            </div>
          </template>
          <template v-else>
            <div class="detail-section">
              <h5>【主料】</h5>
              <p>{{ selectedDish.mainIngredients }}</p>
            </div>
            <div class="detail-section">
              <h5>【做法步骤】</h5>
              <div class="steps-box">{{ selectedDish.production }}</div>
            </div>
            <div v-if="selectedDish.healthTips" class="detail-section tips">
              <h5>【温馨贴士】</h5>
              <p>{{ selectedDish.healthTips }}</p>
            </div>
          </template>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, computed, watch } from 'vue'

const user = ref(JSON.parse(localStorage.getItem('user') || 'null'))
const showAuth = ref(false)
const isLogin = ref(true)
const authLoading = ref(false)
const authForm = ref({ username: '', password: '', confirmPassword: '', email: '' })
const selectedDish = ref(null)
const showPassword = ref(false)
const searchKeyword = ref('')
const filteredDishes = ref([]) // 存储搜索/分类结果
const loadingSearch = ref(false)

// 分类树状态
const categoryTrees = ref({ recipe: [], knowledge: [] })
const selectedCategory = ref(null)
const currentTreeType = ref('recipe')
const sidebarCollapsed = ref(true)
const expandedGroups = ref([]) // 记录展开的父分类code

// 切换标签页时清空选中状态
watch(currentTreeType, () => {
  selectedCategory.value = null
  filteredDishes.value = []
  searchKeyword.value = ''
})

const toggleGroup = (code, forceExpand = false) => {
  if (forceExpand) {
    if (!expandedGroups.value.includes(code)) expandedGroups.value.push(code)
    return
  }
  const index = expandedGroups.value.indexOf(code)
  if (index > -1) {
    expandedGroups.value.splice(index, 1)
  } else {
    expandedGroups.value.push(code)
  }
}

// 获取分类树
const fetchCategoryTrees = async () => {
  try {
    const res = await fetch('http://localhost:8080/dicCategory/tree/all')
    const result = await res.json()
    if (result.code === 200) {
      categoryTrees.value = result.data
    }
  } catch (err) {
    console.error('获取分类树失败:', err)
  }
}

// 选择分类
const selectCategory = async (cat) => {
  if (selectedCategory.value?.code === cat.code) {
    selectedCategory.value = null
    filteredDishes.value = []
    return
  }
  
  selectedCategory.value = cat
  searchKeyword.value = '' // 清除搜索词
  loadingSearch.value = true
  try {
    const isKnowledge = currentTreeType.value === 'knowledge'
    const url = isKnowledge 
      ? `http://localhost:8080/api/knowledge/category/${cat.code}`
      : `http://localhost:8080/api/dish/byCategory/${cat.code}`
    
    const res = await fetch(url)
    const result = await res.json()
    if (result.code === 200) {
      if (isKnowledge) {
        // 将知识条目映射为类似菜谱的结构以便展示
        filteredDishes.value = result.data.map(k => ({
          ...k,
          id: 'k' + k.id, // 区分ID
          name: k.title,
          nutritionalEfficacy: k.content.substring(0, 100) + '...',
          isKnowledge: true, // 标记为知识
          dishType: '烹饪百科',
          categoryName: cat.name
        }))
      } else {
        filteredDishes.value = result.data
      }
    }
  } catch (err) {
    console.error('获取分类内容失败:', err)
  } finally {
    loadingSearch.value = false
  }
}

// 监听搜索词变化，执行后端搜索
let searchTimeout = null
watch(searchKeyword, (newVal) => {
  if (searchTimeout) clearTimeout(searchTimeout)
  
  const keyword = newVal.trim()
  if (!keyword) {
    if (!selectedCategory.value) filteredDishes.value = []
    return
  }

  selectedCategory.value = null // 清除选中分类
  loadingSearch.value = true
  filteredDishes.value = [] 
  searchTimeout = setTimeout(async () => {
    try {
      const res = await fetch(`http://localhost:8080/api/dish/search?keyword=${encodeURIComponent(keyword)}`)
      const result = await res.json()
      if (result.code === 200) {
        filteredDishes.value = result.data
      }
    } catch (err) {
      console.error('搜索失败:', err)
    } finally {
      loadingSearch.value = false
    }
  }, 300) 
})

import { Solar, Lunar } from 'lunar-javascript'

const solarTerm = ref(null)
const recommendations = ref([])
const loading = ref(true)
const error = ref(null)

// 天气数据
const weather = ref({
  temp: '--',
  desc: '加载中...',
  city: '定位中',
  icon: '🌤️',
  humidity: '--',
  windDirection: '',
  windScale: ''
})

const getWeatherIcon = (code) => {
  const codeNum = parseInt(code)
  if (codeNum === 0 || codeNum === 1) return '☀️' 
  if (codeNum === 2 || codeNum === 3) return '🌤️' 
  if (codeNum >= 4 && codeNum <= 9) return '☁️'  
  if (codeNum >= 10 && codeNum <= 12) return '🌧️'
  if (codeNum >= 13 && codeNum <= 17) return '🌧️'
  if (codeNum >= 18 && codeNum <= 20) return '🌨️'
  if (codeNum >= 21 && codeNum <= 25) return '❄️'
  if (codeNum >= 26 && codeNum <= 29) return '🌫️'
  if (codeNum >= 30 && codeNum <= 36) return '💨'
  if (codeNum === 37 || codeNum === 38) return '🌡️'
  return '🌤️'
}

const cityMap = {
  'ip': 'ip',
  '北京': 'Beijing',
  '上海': 'Shanghai',
  '广州': 'Guangzhou',
  '深圳': 'Shenzhen',
  '杭州': 'Hangzhou',
  '南京': 'Nanjing',
  '武汉': 'Wuhan',
  '成都': 'Chengdu',
  '西安': 'Xi\'an',
  '重庆': 'Chongqing'
}

const selectedCity = ref('上海')

const fetchWeather = async (city = null) => {
  try {
    const cityName = city || selectedCity.value
    const location = cityMap[cityName] || cityName
    const res = await fetch(`http://localhost:8080/api/weather/now?location=${encodeURIComponent(location)}`)
    const result = await res.json()
    
    if (result.code === 200 && result.data) {
      const data = result.data
      weather.value = {
        temp: data.temp,
        desc: data.text,
        city: data.city,
        icon: getWeatherIcon(data.weatherCode),
        humidity: data.humidity,
        windDirection: data.windDirection,
        windScale: data.windScale
      }
    } else {
      weather.value = { temp: '--', desc: '获取失败', city: '--', icon: '❓' }
    }
  } catch (err) {
    weather.value = { temp: '--', desc: '获取失败', city: '--', icon: '❓' }
  }
}

const changeCity = async () => {
  await fetchWeather()
}

const calendarInfo = ref({
  year: '',
  month: '',
  day: '',
  weekday: '',
  lunar: '',
  lunarYear: '',
  yi: [],
  ji: [],
  nextTerm: '',
  nextTermDays: 0
})

const getLunarDate = () => {
  const solar = Solar.fromDate(new Date())
  const lunar = solar.getLunar()
  const dayYi = lunar.getDayYi()
  const dayJi = lunar.getDayJi()
  const jieQi = lunar.getNextJieQi()
  const nextTermName = jieQi ? jieQi.getName() : ''
  const nextTermSolar = jieQi ? jieQi.getSolar() : null
  
  let daysToNextTerm = 0
  if (nextTermSolar) {
    const today = new Date()
    const nextDate = new Date(nextTermSolar.getYear(), nextTermSolar.getMonth() - 1, nextTermSolar.getDay())
    daysToNextTerm = Math.ceil((nextDate - today) / (1000 * 60 * 60 * 24))
  }
  
  calendarInfo.value = {
    year: solar.getYear(),
    month: solar.getMonth(),
    day: solar.getDay(),
    weekday: '星期' + lunar.getWeekInChinese(),
    lunar: lunar.getMonthInChinese() + '月' + lunar.getDayInChinese(),
    lunarYear: lunar.getYearInGanZhi() + '年 【' + lunar.getYearShengXiao() + '年】',
    yi: dayYi.slice(0, 6),
    ji: dayJi.slice(0, 6),
    nextTerm: nextTermName,
    nextTermDays: daysToNextTerm
  }
}

const messages = ref([])
let msgId = 0

const showMessage = (text, type = 'success') => {
  const id = msgId++
  messages.value.push({ id, text, type })
  setTimeout(() => {
    messages.value = messages.value.filter(m => m.id !== id)
  }, 3000)
}

const fetchCurrentTerm = async () => {
  try {
    const res = await fetch('http://localhost:8080/api/solar-term/current')
    const result = await res.json()
    if (result.code === 200) {
      solarTerm.value = result.data
      recommendations.value = result.recommendations
    }
  } catch (err) {
    showMessage('无法连接到后端服务器', 'error')
  } finally {
    loading.value = false
  }
}

const handleAuth = async () => {
  if (!authForm.value.username || !authForm.value.password) {
    showMessage('请输入用户名和密码', 'error')
    return
  }

  if (!isLogin.value && authForm.value.password !== authForm.value.confirmPassword) {
    showMessage('两次输入的密码不一致', 'error')
    return
  }

  authLoading.value = true
  const url = isLogin.value ? '/api/users/login' : '/api/users/register'
  try {
    const res = await fetch(`http://localhost:8080${url}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(authForm.value)
    })
    const result = await res.json()
    if (result.code === 200) {
      user.value = result.data
      localStorage.setItem('user', JSON.stringify(result.data))
      showAuth.value = false
      showMessage(isLogin.value ? '登录成功，欢迎回来' : '注册成功，已自动登录')
      if (!recommendations.value.length) fetchCurrentTerm()
    } else {
      showMessage(result.msg || '操作失败', 'error')
    }
  } catch (err) {
    showMessage('连接服务器失败', 'error')
  } finally {
    authLoading.value = false
  }
}

const logout = () => {
  user.value = null
  localStorage.removeItem('user')
  showMessage('已安全退出')
}

const openAuth = () => {
  isLogin.value = true
  showAuth.value = true
}

const toggleAuthMode = () => {
  isLogin.value = !isLogin.value
}

const showDetail = (dish) => {
  selectedDish.value = dish
}

onMounted(() => {
  fetchCurrentTerm()
  fetchWeather()
  getLunarDate()
  fetchCategoryTrees() // 加载分类树
})
</script>

<style scoped>
/* Modern Color Palette */
:root {
  --primary: #27ae60;
  --primary-dark: #219150;
  --secondary: #e67e22;
  --bg: #f5f7f9;
  --text: #2c3e50;
  --text-light: #7f8c8d;
  --white: #ffffff;
}

.app-container {
  min-height: 100vh;
  background-color: #f5f7f9;
  color: #2c3e50;
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
  padding-bottom: 50px;
}

/* Header */
.header {
  background: white;
  padding: 15px 0;
  box-shadow: 0 2px 10px rgba(0,0,0,0.05);
  position: sticky;
  top: 0;
  z-index: 100;
}

.header-content {
  max-width: 1200px;
  margin: 0 auto;
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0 20px;
}

.logo-text {
  font-size: 1.4rem;
  font-weight: 800;
  color: #27ae60;
  margin: 0;
}

.user-panel {
  display: flex;
  align-items: center;
  gap: 15px;
}

.user-name {
  font-size: 0.9rem;
  font-weight: 500;
}

/* Layout */
.content {
  max-width: 1200px;
  margin: 0 auto;
  padding: 20px;
}

.main-layout {
  display: flex;
  gap: 25px;
  align-items: flex-start;
}

/* Sidebar */
.sidebar {
  width: 260px;
  background: white;
  border-radius: 20px;
  padding: 20px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.04);
  position: sticky;
  top: 90px;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  min-height: 500px;
  flex-shrink: 0;
}

.sidebar.collapsed {
  width: 60px;
  padding: 20px 10px;
}

.sidebar-toggle {
  position: absolute;
  right: -12px;
  top: 20px;
  background: #27ae60;
  color: white;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  font-size: 0.7rem;
  box-shadow: 0 2px 8px rgba(39,174, 96, 0.4);
  z-index: 10;
}

.sidebar-collapsed-icons {
  display: flex;
  flex-direction: column;
  gap: 15px;
  align-items: center;
}

.collapsed-icon-btn {
  width: 40px;
  height: 40px;
  background: white;
  color: #27ae60;
  border: 1px solid #e8f5e9;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: bold;
  cursor: pointer;
  transition: all 0.2s;
  box-shadow: 0 2px 5px rgba(0,0,0,0.02);
}

.collapsed-icon-btn:hover {
  background: #27ae60;
  color: white;
  transform: scale(1.05);
}

.main-view {
  flex: 1;
  min-width: 0;
}

/* Category Area In Sidebar */
.tree-tabs {
  display: flex;
  gap: 10px;
  margin-bottom: 20px;
  border-bottom: 1px solid #eee;
  padding-bottom: 10px;
}

.tree-tab {
  background: none;
  border: none;
  font-size: 0.9rem;
  font-weight: 600;
  color: #95a5a6;
  cursor: pointer;
  padding: 5px;
  flex: 1;
  text-align: center;
  white-space: nowrap;
}

.tree-tab.active { color: #27ae60; border-bottom: 2px solid #27ae60; }

.category-scroll {
  display: flex;
  flex-direction: column;
  gap: 18px;
  max-height: 70vh;
  overflow-y: auto;
  padding-right: 5px;
}

/* Custom Scrollbar */
.category-scroll::-webkit-scrollbar { width: 4px; }
.category-scroll::-webkit-scrollbar-track { background: #f1f1f1; }
.category-scroll::-webkit-scrollbar-thumb { background: #ddd; border-radius: 4px; }

.category-group {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.parent-cat {
  font-size: 0.9rem;
  font-weight: 700;
  color: #2c3e50;
  display: flex;
  justify-content: space-between;
  align-items: center;
  cursor: pointer;
  padding: 4px 0;
  transition: color 0.2s;
}

.parent-cat:hover { color: #27ae60; }

.group-arrow {
  font-size: 0.6rem;
  color: #bdc3c7;
  transition: transform 0.3s;
}

.children-cats {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  transition: max-height 0.4s cubic-bezier(0.4, 0, 0.2, 1);
  overflow: hidden;
}

.children-cats.one-row {
  max-height: 32px; /* 约一行高度 */
}

.cat-chip {
  font-size: 0.75rem;
  padding: 5px 12px;
  background: #f8f9fa;
  border: 1px solid #eee;
  border-radius: 15px;
  cursor: pointer;
  transition: all 0.2s;
  white-space: nowrap;
}

.cat-chip:hover { background: #eef2f5; border-color: #27ae60; color: #27ae60; }
.cat-chip.active { background: #27ae60; color: white; border-color: #27ae60; }

/* 精简信息栏样式 */
.info-bar {
  background: linear-gradient(135deg, #1a5c37 0%, #27ae60 100%);
  border-radius: 16px;
  padding: 16px 20px;
  margin-bottom: 20px;
  display: flex;
  align-items: center;
  gap: 20px;
  color: white;
  box-shadow: 0 6px 20px rgba(39, 174, 96, 0.25);
}

.mini-calendar {
  display: flex;
  align-items: center;
  gap: 12px;
  padding-right: 20px;
  border-right: 1px solid rgba(255,255,255,0.25);
}

.mini-cal-day {
  font-size: 2.8rem;
  font-weight: 800;
  line-height: 1;
}

.mini-cal-info {
  display: flex;
  flex-direction: column;
}

.mini-cal-date { font-size: 0.9rem; font-weight: 600; }
.mini-cal-lunar { font-size: 0.8rem; opacity: 0.85; }
.mini-cal-term {
  background: rgba(255,255,255,0.15);
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 0.75rem;
}
.mini-cal-term strong { color: #ffe066; margin: 0 3px; }

.mini-yiji {
  flex: 1;
  display: flex;
  flex-direction: column;
  gap: 6px;
  padding-right: 20px;
  border-right: 1px solid rgba(255,255,255,0.25);
}

.yiji-row { display: flex; align-items: center; gap: 8px; }
.yi-tag { background: #e74c3c; color: white; width: 22px; height: 22px; border-radius: 4px; display: flex; align-items: center; justify-content: center; font-weight: bold; font-size: 0.7rem; }
.ji-tag { background: #3498db; color: white; width: 22px; height: 22px; border-radius: 4px; display: flex; align-items: center; justify-content: center; font-weight: bold; font-size: 0.7rem; }
.yiji-text { font-size: 0.8rem; opacity: 0.9; }

.weather-box { display: flex; align-items: center; gap: 10px; }
.weather-icon { font-size: 2.2rem; }
.weather-temp { font-size: 1.4rem; font-weight: 700; }
.weather-desc { font-size: 0.85rem; opacity: 0.9; }
.city-select { background: rgba(255, 255, 255, 0.2); color: white; border: 1px solid rgba(255, 255, 255, 0.3); border-radius: 6px; padding: 2px 6px; font-size: 0.7rem; outline: none; }
.city-select option { background: #27ae60; color: white; }

/* Cards */
.main-card {
  background: white;
  border-radius: 20px;
  padding: 30px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.04);
  margin-bottom: 30px;
  position: relative;
  overflow: hidden;
}

.term-highlight { border-top: 6px solid #27ae60; }
.term-badge { position: absolute; top: 0; right: 0; background: #27ae60; color: white; padding: 5px 20px; font-size: 0.75rem; border-bottom-left-radius: 15px; }
.term-title-row { display: flex; align-items: center; gap: 20px; margin-bottom: 30px; }
.season-circle { width: 50px; height: 50px; background: #e8f5e9; color: #27ae60; border-radius: 50%; display: flex; align-items: center; justify-content: center; font-weight: bold; font-size: 1.2rem; }
.term-name { font-size: 2.2rem; margin: 0; letter-spacing: 2px; }
.term-details { display: grid; grid-template-columns: repeat(auto-fit, minmax(200px, 1fr)); gap: 25px; }
.detail-box { border-left: 3px solid #eee; padding-left: 15px; }
.box-label { font-size: 0.8rem; color: #95a5a6; margin-bottom: 8px; font-weight: 600; }
.box-value { font-size: 1rem; line-height: 1.6; }
.highlight-text { color: #e67e22; font-weight: bold; font-size: 1.1rem; }

/* Search Area */
.search-area { margin: 20px 0 30px; }
.search-container { max-width: 600px; margin: 0 auto; }
.search-input { width: 100%; padding: 15px 20px; border: 2px solid #e0e0e0; border-radius: 30px; font-size: 1rem; outline: none; transition: all 0.3s ease; }
.search-input:focus { border-color: #27ae60; box-shadow: 0 0 0 3px rgba(39, 174, 96, 0.1); }

/* Recommendation Area */
.section-title { display: flex; align-items: center; justify-content: space-between; margin-bottom: 25px; }
.tag-outline { font-size: 0.75rem; padding: 4px 10px; border: 1px solid #ddd; border-radius: 4px; color: #7f8c8d; }

.dish-grid { display: grid; grid-template-columns: repeat(auto-fill, minmax(300px, 1fr)); gap: 25px; }
.dish-card {
  background: white;
  border-radius: 16px;
  padding: 24px;
  box-shadow: 0 4px 12px rgba(0,0,0,0.03);
  transition: all 0.3s ease;
  border: 1px solid rgba(0,0,0,0.05);
}
.dish-card:hover { transform: translateY(-5px); box-shadow: 0 12px 25px rgba(0,0,0,0.08); }
.dish-header-tags { display: flex; justify-content: space-between; align-items: center; margin-bottom: 12px; }
.type-tag { background: #e8f5e9; color: #27ae60; padding: 2px 8px; border-radius: 4px; font-size: 0.7rem; font-weight: bold; }
.dish-tag { color: #7f8c8d; font-size: 0.75rem; font-weight: 500; }
.dish-desc { font-size: 0.85rem; color: #7f8c8d; margin-bottom: 20px; height: 2.6rem; overflow: hidden; display: -webkit-box; -webkit-line-clamp: 2; -webkit-box-orient: vertical; }

/* Knowledge Card Specifics */
.knowledge-card { border-left: 5px solid #ffb300; }
.knowledge-card h4 { color: #5d4037; }
.knowledge-summary { font-style: italic; color: #546e7a; }
.card-key-points { margin-bottom: 15px; display: flex; flex-direction: column; gap: 4px; }
.mini-point { font-size: 0.75rem; color: #795548; background: #fff8e1; padding: 2px 8px; border-radius: 4px; }

/* Locked State */
.locked-state { position: relative; height: 300px; background: white; border-radius: 20px; overflow: hidden; display: flex; align-items: center; justify-content: center; }
.locked-mask { position: absolute; inset: 0; filter: blur(15px); opacity: 0.2; }
.fake-grid { display: flex; gap: 20px; padding: 20px; }
.fake-card { width: 280px; height: 200px; background: #eee; border-radius: 15px; }
.locked-content { position: relative; text-align: center; z-index: 10; }
.lock-icon { font-size: 3rem; margin-bottom: 15px; }

/* Buttons */
.btn-primary { background: #27ae60; color: white; border: none; padding: 12px 35px; border-radius: 30px; font-weight: 600; cursor: pointer; }
.btn-primary-sm { background: #27ae60; color: white; border: none; padding: 8px 18px; border-radius: 6px; font-size: 0.85rem; font-weight: 600; cursor: pointer; }
.btn-primary-block { width: 100%; background: #27ae60; color: white; border: none; padding: 14px; border-radius: 10px; font-weight: 600; cursor: pointer; margin-top: 10px; }
.btn-outline-sm { background: transparent; border: 1px solid #27ae60; color: #27ae60; padding: 8px 15px; border-radius: 6px; font-size: 0.8rem; width: 100%; cursor: pointer; }
.btn-text { background: none; border: none; color: #e74c3c; font-size: 0.85rem; cursor: pointer; }

/* Toasts */
.toast { position: fixed; top: 20px; right: 20px; padding: 15px 25px; border-radius: 10px; background: white; box-shadow: 0 10px 25px rgba(0,0,0,0.15); display: flex; align-items: center; gap: 12px; z-index: 2000; }
.toast.success { border-left: 5px solid #27ae60; }
.toast.error { border-left: 5px solid #e74c3c; }

/* Modals */
.modal-overlay { 
  position: fixed; 
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(0,0,0,0.7); 
  backdrop-filter: blur(5px); 
  display: flex; 
  align-items: center; 
  justify-content: center; 
  z-index: 9999; 
}
.auth-card { width: 100%; max-width: 400px; background: white; border-radius: 24px; padding: 40px; box-shadow: 0 20px 60px rgba(0,0,0,0.3); }
.input-group { margin-bottom: 15px; }
.input-group input { width: 100%; padding: 14px; background: #f8f9fa; border: 1px solid #eee; border-radius: 10px; font-size: 0.95rem; }
.password-group { position: relative; }
.password-toggle { position: absolute; right: 15px; top: 50%; transform: translateY(-50%); cursor: pointer; }
.detail-card { 
  width: 90%; 
  max-width: 800px; 
  max-height: 85vh;
  background: white; 
  border-radius: 24px; 
  overflow: hidden; 
  display: flex;
  flex-direction: column;
  box-shadow: 0 25px 70px rgba(0,0,0,0.4);
}
.detail-header { padding: 25px 30px; border-bottom: 1px solid #f0f0f0; display: flex; justify-content: space-between; align-items: center; flex-shrink: 0; }
.detail-header h2 { margin: 0; color: #27ae60; font-size: 1.5rem; }
.close-btn { background: #f1f2f6; border: none; font-size: 1.5rem; width: 36px; height: 36px; border-radius: 50%; cursor: pointer; display: flex; align-items: center; justify-content: center; transition: all 0.2s; }
.close-btn:hover { background: #dfe4ea; color: #e74c3c; }
.detail-scroll { padding: 30px; overflow-y: auto; flex: 1; }
.summary-box { background: #e8f5e9; padding: 15px; border-radius: 10px; border-left: 4px solid #27ae60; margin-bottom: 20px; }
.summary-text { font-size: 0.9rem; color: #2e7d32; margin: 0; }
.key-points-box { background: #fff8e1; padding: 15px; border-radius: 10px; border-left: 4px solid #ffb300; margin-bottom: 20px; }
.points-list { margin: 10px 0 0 20px; padding: 0; }
.points-list li { font-size: 0.9rem; color: #5d4037; margin-bottom: 5px; }
.steps-box { background: #f9f9f9; padding: 20px; border-radius: 12px; line-height: 1.8; white-space: pre-wrap; }
.knowledge-content { font-size: 1rem; color: #37474f; text-align: justify; }
</style>
