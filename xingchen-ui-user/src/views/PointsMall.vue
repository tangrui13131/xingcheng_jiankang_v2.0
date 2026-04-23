<template>
  <div class="user-layout">
    <div class="user-sidebar">
      <div class="logo-container">
        <img :src="logoSrc" class="user-logo" alt="logo" />
        <div class="logo-text">星辰 · 用户端</div>
      </div>
      <div class="menu">
        <div class="menu-item" @click="router.push('/')">首页</div>
        <div class="menu-item" @click="router.push('/health-data')">健康看板</div>
        <div class="menu-item active">积分商城</div>
        <div class="menu-item" @click="router.push('/profile')">个人中心</div>
        <div class="menu-item" @click="handleLogout">退出登录</div>
      </div>
    </div>
    
    <div class="user-content">
      <div class="user-header">
        <div class="welcome">积分商城</div>
      </div>
      
      <div class="points-mall-container">
        <!-- 顶部积分看板 -->
        <el-card class="dashboard-card" shadow="hover">
          <div class="dashboard-content">
            <div class="points-info">
              <span class="label">我的可用健康积分</span>
              <span class="value">{{ currentPoints }}</span>
            </div>
            <div class="tips">
              <el-icon class="tips-icon"><InfoFilled /></el-icon>
              每日完成健康打卡可获得10积分
            </div>
          </div>
        </el-card>

        <!-- 商品列表区 -->
        <el-row :gutter="20" class="product-list">
          <el-col
            v-for="product in productList"
            :key="product.id"
            :xs="24" :sm="12" :md="8" :lg="6"
            class="product-item"
          >
            <el-card shadow="hover" class="product-card" :body-style="{ padding: '0px' }">
              <!-- 商品图片 -->
              <div class="product-image">
                <el-image
                  :src="product.image"
                  :alt="product.name"
                  fit="cover"
                  class="product-img"
                  lazy
                >
                  <template #error>
                    <div class="image-fallback" :style="{ backgroundColor: product.bgColor }">
                      <el-icon class="fallback-icon"><Present /></el-icon>
                      <span class="fallback-text">暂无图片</span>
                    </div>
                  </template>
                  <template #placeholder>
                    <div class="image-loading">
                      <el-icon class="is-loading"><Loading /></el-icon>
                    </div>
                  </template>
                </el-image>
              </div>
              
              <div class="product-info">
                <h3 class="product-name">{{ product.name }}</h3>
                <div class="product-price">
                  <span class="price-value">{{ product.price }}</span>
                  <span class="price-unit">分</span>
                </div>
                <div class="product-action">
                  <el-button 
                    type="primary" 
                    class="exchange-btn" 
                    @click="handleExchange(product)"
                  >
                    立即兑换
                  </el-button>
                </div>
              </div>
            </el-card>
          </el-col>
        </el-row>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useRouter } from 'vue-router'
import { ElMessage, ElMessageBox } from 'element-plus'
import { InfoFilled, Present, Loading } from '@element-plus/icons-vue'
import request from '@/api/request'
import logoSrc from '@/assets/logo.png'
import { healthData, deductPoints } from '@/utils/mockHealthData'

const router = useRouter()

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

// 积分直接绑定全局数据中心（响应式，打卡后自动更新）
const currentPoints = computed(() => healthData.userPoints)

// 商品列表
// image 字段填写图片路径：本地图片放 public/images/products/ 目录，引用路径为 /images/products/xxx.png
// 也可以使用在线图片 URL
const productList = ref([
  {
    id: 1,
    name: '瑞幸咖啡兑换券',
    price: 150,
    bgColor: '#fdf6ec',
    image: '/images/products/coffee.png'
  },
  {
    id: 2,
    name: '午休颈枕',
    price: 300,
    bgColor: '#ecf5ff',
    image: '/images/products/pillow.png'
  },
  {
    id: 3,
    name: '半天带薪健康假',
    price: 1000,
    bgColor: '#f0f9eb',
    image: '/images/products/holiday.png'
  },
  {
    id: 4,
    name: '智能运动手环',
    price: 2500,
    bgColor: '#fef0f0',
    image: '/images/products/band.png'
  },
  {
    id: 5,
    name: '定制健康水杯',
    price: 200,
    bgColor: '#f4f4f5',
    image: '/images/products/cup.png'
  },
  {
    id: 6,
    name: '眼部按摩仪',
    price: 800,
    bgColor: '#faecd8',
    image: '/images/products/massager.png'
  }
])

// 点击立即兑换
const handleExchange = (product) => {
  ElMessageBox.confirm(
    `确认消耗 ${product.price} 积分兑换该商品吗？`,
    '兑换确认',
    {
      confirmButtonText: '确定',
      cancelButtonText: '取消',
      type: 'warning',
    }
  ).then(() => {
    // 通过全局数据中心扣除积分（自动持久化到 localStorage）
    if (deductPoints(product.price)) {
      ElMessage({
        type: 'success',
        message: '兑换申请已提交，请等待HR审核！',
      })
    } else {
      ElMessage({
        type: 'error',
        message: '积分不足，无法兑换该商品！',
      })
    }
  }).catch(() => {
    // 取消操作
  })
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

.points-mall-container {
  padding: 40px;
  overflow-y: auto;
  background-color: var(--bg-0);
  min-height: calc(100vh - 80px);
}

.dashboard-card {
  margin-bottom: 24px;
  border-radius: 8px;
}

.dashboard-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  flex-wrap: wrap;
  padding: 10px 0;
}

.points-info {
  display: flex;
  align-items: baseline;
}

.points-info .label {
  font-size: 16px;
  color: #606266;
  margin-right: 12px;
}

.points-info .value {
  font-size: 36px;
  font-weight: bold;
  color: #409eff; /* 浅色系主色调 */
}

.tips {
  display: flex;
  align-items: center;
  color: #909399;
  font-size: 14px;
  margin-top: 10px;
}

.tips-icon {
  margin-right: 6px;
  font-size: 16px;
  color: #e6a23c;
}

.product-list {
  margin-top: 10px;
}

.product-item {
  margin-bottom: 20px;
}

.product-card {
  border-radius: 8px;
  overflow: hidden;
  transition: all 0.3s;
  height: 100%;
}

.product-card:hover {
  transform: translateY(-5px);
  box-shadow: 0 12px 20px rgba(0, 0, 0, 0.1);
}

.product-image {
  height: 180px;
  overflow: hidden;
  background-color: #f5f7fa;

  .product-img {
    width: 100%;
    height: 100%;
    display: block;
  }

  .image-fallback {
    width: 100%;
    height: 100%;
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
    gap: 8px;

    .fallback-icon {
      font-size: 48px;
      color: rgba(0, 0, 0, 0.12);
    }

    .fallback-text {
      font-size: 12px;
      color: rgba(0, 0, 0, 0.25);
    }
  }

  .image-loading {
    width: 100%;
    height: 100%;
    display: flex;
    justify-content: center;
    align-items: center;
    font-size: 24px;
    color: #c0c4cc;
  }
}

.product-info {
  padding: 16px;
}

.product-name {
  margin: 0 0 12px 0;
  font-size: 16px;
  color: #303133;
  white-space: nowrap;
  overflow: hidden;
  text-overflow: ellipsis;
}

.product-price {
  margin-bottom: 16px;
}

.price-value {
  font-size: 24px;
  font-weight: bold;
  color: #f56c6c; /* 醒目的红色 */
}

.price-unit {
  font-size: 14px;
  color: #909399;
  margin-left: 4px;
  font-weight: normal;
}

.product-action {
  text-align: center;
}

.exchange-btn {
  width: 100%;
  border-radius: 20px;
}

/* 响应式调整 */
@media (max-width: 768px) {
  .dashboard-content {
    flex-direction: column;
    align-items: flex-start;
  }
  .tips {
    margin-top: 16px;
  }
}
</style>
