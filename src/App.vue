<template>
  <div class="app-container">
    <header class="header">
      <h1>24节气AI食疗推荐</h1>
      <p class="date">{{ currentDate }}</p>
      <div v-if="user" class="user-info">
        <span class="user-tag">👤 {{ user.username }}</span>
        <button @click="logout" class="logout-btn">退出登录</button>
      </div>
      <div v-else class="user-info">
        <button @click="showAuth = true" class="login-trigger">登录 / 注册</button>
      </div>
    </header>

    <main class="content">
      <!-- Section 1: Solar Term Info (Always Visible) -->
      <section v-if="solarTerm" class="term-card">
        <div class="term-header">
          <span class="season-tag">{{ solarTerm.termSeason }}季</span>
          <h2>今日节气：{{ solarTerm.name }}</h2>
        </div>
        <div class="term-body">
          <div class="info-group">
            <h3>🌟 养生要点</h3>
            <p>{{ solarTerm.healthTip }}</p>
          </div>
          <div class="info-group">
            <h3>🍲 饮食建议</h3>
            <p>{{ solarTerm.dietarySuggestion }}</p>
          </div>
          <div class="info-group">
            <h3>🥦 时令食材</h3>
            <p class="ingredients">{{ solarTerm.recommendedIngredients }}</p>
          </div>
        </div>
      </section>

      <!-- Section 2: Recommended Dishes (Login Required for the List) -->
      <section v-if="solarTerm" class="recommendation-section">
        <div class="section-header">
          <h3>🍽️ {{ solarTerm.name }}·时令美食推荐</h3>
        </div>

        <!-- If Logged In: Show the Grid -->
        <div v-if="user">
          <div v-if="recommendations.length" class="dish-grid">
            <div v-for="dish in recommendations" :key="dish.id" class="dish-card">
              <h4>{{ dish.name }}</h4>
              <p class="dish-cat">{{ dish.category }}</p>
              <div class="dish-efficacy">
                <strong>功效:</strong> {{ dish.nutritionalEfficacy }}
              </div>
              <button @click="showDetail(dish)" class="detail-btn">查看做法</button>
            </div>
          </div>
          <div v-else-if="!loading" class="empty-msg">暂无匹配推荐</div>
        </div>

        <!-- If Not Logged In: Show a simple notice instead of the grid -->
        <div v-else class="login-prompt-box">
          <p>🔒 登录后解锁为您准备的 <strong>{{ solarTerm.name }}</strong> 专属食疗方案</p>
          <button @click="showAuth = true" class="auth-btn-small">立即登录查看</button>
        </div>
      </section>

      <div v-if="loading" class="loading">正在加载时令数据...</div>
      <div v-if="error" class="error">{{ error }}</div>
    </main>

    <!-- Auth Modal (Login/Register) -->
    <div v-if="showAuth" class="modal-overlay" @click="showAuth = false">
      <div class="modal-content auth-modal" @click.stop>
        <div class="modal-header">
          <h2>{{ isLogin ? '用户登录' : '新用户注册' }}</h2>
          <button class="close-icon" @click="showAuth = false">&times;</button>
        </div>
        <div class="auth-form">
          <div class="form-group">
            <label>用户名</label>
            <input v-model="authForm.username" type="text" placeholder="请输入用户名" />
          </div>
          <div class="form-group">
            <label>密码</label>
            <input v-model="authForm.password" type="password" placeholder="请输入密码" />
          </div>
          <div v-if="!isLogin" class="form-group">
            <label>电子邮箱</label>
            <input v-model="authForm.email" type="email" placeholder="请输入邮箱 (可选)" />
          </div>
          <div v-if="authError" class="error-msg">{{ authError }}</div>
          <button @click="handleAuth" class="auth-btn">{{ isLogin ? '立即登录' : '提交注册' }}</button>
          <p class="switch-mode" @click="isLogin = !isLogin">
            {{ isLogin ? '没有账号？点击注册' : '已有账号？点击返回登录' }}
          </p>
        </div>
      </div>
    </div>

    <!-- Dish Detail Modal -->
    <div v-if="selectedDish" class="modal-overlay" @click="selectedDish = null">
      <div class="modal-content detail-modal" @click.stop>
        <div class="modal-header">
          <h2>{{ selectedDish.name }}</h2>
          <button class="close-icon" @click="selectedDish = null">&times;</button>
        </div>
        <div class="detail-body">
          <div class="detail-item">
            <span class="label">【主料】</span>
            <p>{{ selectedDish.mainIngredients }}</p>
          </div>
          <div class="detail-item">
            <span class="label">【制作方法】</span>
            <div class="production">{{ selectedDish.production }}</div>
          </div>
          <div class="detail-item" v-if="selectedDish.healthTips">
            <span class="label">【健康贴士】</span>
            <p>{{ selectedDish.healthTips }}</p>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted } from 'vue'

const user = ref(JSON.parse(localStorage.getItem('user') || 'null'))
const showAuth = ref(false)
const isLogin = ref(true)
const authForm = ref({ username: '', password: '', email: '' })
const authError = ref('')

const solarTerm = ref(null)
const recommendations = ref([])
const loading = ref(true)
const error = ref(null)
const selectedDish = ref(null)
const currentDate = new Date().toLocaleDateString('zh-CN', { year: 'numeric', month: 'long', day: 'numeric', weekday: 'long' })

const fetchCurrentTerm = async () => {
  try {
    const res = await fetch('http://localhost:8080/api/solar-term/current')
    const result = await res.json()
    if (result.code === 200) {
      solarTerm.value = result.data
      recommendations.value = result.recommendations
    } else {
      error.value = result.msg
    }
  } catch (err) {
    error.value = '无法连接到后端服务器'
  } finally {
    loading.value = false
  }
}

const handleAuth = async () => {
  const url = isLogin.value ? '/api/users/login' : '/api/users/register'
  authError.value = ''
  try {
    const res = await fetch(`http://localhost:8080${url}`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(authForm.value)
    })
    const result = await res.json()
    if (result.code === 200) {
      if (isLogin.value) {
        user.value = result.data
        localStorage.setItem('user', JSON.stringify(result.data))
        showAuth.value = false
      } else {
        alert('注册成功，请登录')
        isLogin.value = true
        authForm.value.password = ''
      }
    } else {
      authError.value = result.msg
    }
  } catch (err) {
    authError.value = '连接服务器失败'
  }
}

const logout = () => {
  user.value = null
  localStorage.removeItem('user')
}

const showDetail = (dish) => {
  selectedDish.value = dish
}

onMounted(() => {
  fetchCurrentTerm()
})
</script>

<style scoped>
.app-container {
  max-width: 1000px;
  margin: 0 auto;
  padding: 20px;
  font-family: 'PingFang SC', 'Microsoft YaHei', sans-serif;
  color: #2c3e50;
  background-color: #f8f9fa;
  min-height: 100vh;
}

.header {
  display: flex;
  flex-direction: column;
  align-items: center;
  margin-bottom: 40px;
  position: relative;
}

.header h1 {
  color: #27ae60;
  font-size: 2.5rem;
  margin-bottom: 10px;
}

.user-info {
  margin-top: 15px;
  display: flex;
  gap: 15px;
  align-items: center;
}

.user-tag {
  font-weight: bold;
  color: #27ae60;
}

.logout-btn, .login-trigger {
  padding: 6px 15px;
  border-radius: 4px;
  cursor: pointer;
  font-size: 0.9rem;
  border: 1px solid transparent;
}

.logout-btn {
  background: #fdf0f0;
  color: #e74c3c;
  border-color: #fadbd8;
}

.login-trigger {
  background: #27ae60;
  color: white;
}

.term-card {
  background: white;
  border-radius: 15px;
  padding: 30px;
  box-shadow: 0 4px 15px rgba(0,0,0,0.05);
  margin-bottom: 40px;
  border-top: 5px solid #27ae60;
}

.term-header {
  display: flex;
  align-items: center;
  gap: 15px;
  margin-bottom: 20px;
}

.season-tag {
  background: #27ae60;
  color: white;
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 0.8rem;
}

.info-group h3 {
  color: #2c3e50;
  margin-bottom: 8px;
  font-size: 1.1rem;
}

.ingredients {
  color: #e67e22;
  font-weight: bold;
  font-size: 1.1rem;
}

.recommendation-section {
  margin-top: 40px;
}

.section-header {
  margin-bottom: 20px;
  border-bottom: 2px solid #27ae60;
  padding-bottom: 10px;
}

.dish-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
  gap: 20px;
}

.dish-card {
  background: white;
  padding: 20px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0,0,0,0.05);
}

.dish-cat { font-size: 0.8rem; color: #95a5a6; margin-bottom: 8px; }

.dish-efficacy {
  font-size: 0.85rem;
  margin-bottom: 15px;
  color: #7f8c8d;
}

.detail-btn {
  width: 100%;
  padding: 10px;
  background: #27ae60;
  color: white;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}

/* Login Prompt Styles */
.login-prompt-box {
  background: white;
  padding: 40px;
  border-radius: 15px;
  text-align: center;
  border: 1px dashed #27ae60;
}

.auth-btn-small {
  background: #27ae60;
  color: white;
  padding: 8px 20px;
  border: none;
  border-radius: 4px;
  font-size: 1rem;
  cursor: pointer;
  margin-top: 15px;
}

/* Modal Styles */
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.5);
  display: flex; justify-content: center; align-items: center;
  z-index: 1000;
}

.modal-content {
  background: white; border-radius: 15px;
  width: 90%; max-width: 500px;
  box-shadow: 0 10px 25px rgba(0,0,0,0.2);
}

.modal-header {
  padding: 20px;
  border-bottom: 1px solid #eee;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.close-icon {
  background: none; border: none; font-size: 1.5rem; color: #95a5a6; cursor: pointer;
}

.auth-form { padding: 30px; }
.form-group { margin-bottom: 20px; text-align: left; }
.form-group label { display: block; margin-bottom: 8px; font-size: 0.9rem; color: #7f8c8d; }
.form-group input {
  width: 100%; padding: 12px; border: 1px solid #ddd; border-radius: 8px;
  box-sizing: border-box;
}

.auth-btn {
  width: 100%; padding: 12px; background: #27ae60; color: white;
  border: none; border-radius: 8px; cursor: pointer; font-size: 1rem;
  margin-top: 10px;
}

.switch-mode { text-align: center; margin-top: 20px; color: #3498db; cursor: pointer; font-size: 0.9rem; }
.error-msg { color: #e74c3c; margin-bottom: 15px; font-size: 0.85rem; text-align: left; }

.detail-body { padding: 25px; max-height: 70vh; overflow-y: auto; }
.detail-item { margin-bottom: 20px; }
.detail-item .label { font-weight: bold; color: #27ae60; display: block; margin-bottom: 8px; }
.production { white-space: pre-wrap; line-height: 1.8; color: #34495e; background: #f9f9f9; padding: 15px; border-radius: 10px; }

.empty-msg { text-align: center; padding: 40px; color: #95a5a6; }
</style>
