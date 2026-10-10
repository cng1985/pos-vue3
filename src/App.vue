<script setup>
import { computed, reactive, ref, watch } from 'vue'

const products = reactive([
  { id: 1, name: '可乐', price: 3.5, category: '饮料' },
  { id: 2, name: '矿泉水', price: 2, category: '饮料' },
  { id: 3, name: '薯片', price: 5, category: '零食' },
  { id: 4, name: '饼干', price: 4.5, category: '零食' },
  { id: 5, name: '牛奶', price: 6, category: '乳品' },
  { id: 6, name: '三明治', price: 12, category: '熟食' }
])

const keyword = ref('')
const cart = ref([])
const received = ref('')
const orderHistory = ref(JSON.parse(localStorage.getItem('pos-orders') || '[]'))

const filteredProducts = computed(() => {
  const key = keyword.value.trim().toLowerCase()
  if (!key) return products
  return products.filter((item) => item.name.toLowerCase().includes(key) || item.category.includes(key))
})

const subtotal = computed(() => cart.value.reduce((sum, item) => sum + item.price * item.qty, 0))
const taxRate = ref(0.06)
const tax = computed(() => subtotal.value * taxRate.value)
const total = computed(() => subtotal.value + tax.value)

const change = computed(() => {
  const paid = Number(received.value || 0)
  return paid - total.value
})

watch(
  orderHistory,
  (list) => {
    localStorage.setItem('pos-orders', JSON.stringify(list))
  },
  { deep: true }
)

function addToCart(product) {
  const exists = cart.value.find((item) => item.id === product.id)
  if (exists) {
    exists.qty += 1
  } else {
    cart.value.push({ ...product, qty: 1 })
  }
}

function increaseQty(item) {
  item.qty += 1
}

function decreaseQty(item) {
  if (item.qty > 1) item.qty -= 1
}

function removeItem(id) {
  cart.value = cart.value.filter((item) => item.id !== id)
}

function clearCart() {
  cart.value = []
  received.value = ''
}

function checkout() {
  if (!cart.value.length) return alert('购物车为空')
  if (change.value < 0) return alert('收款金额不足')

  const order = {
    id: Date.now(),
    createdAt: new Date().toLocaleString(),
    items: cart.value.map((item) => ({
      name: item.name,
      qty: item.qty,
      unitPrice: item.price,
      amount: item.qty * item.price
    })),
    subtotal: subtotal.value,
    tax: tax.value,
    total: total.value,
    paid: Number(received.value),
    change: change.value
  }

  orderHistory.value.unshift(order)
  clearCart()
  alert('结算成功')
}
</script>

<template>
  <main class="container">
    <section class="products panel">
      <h2>商品列表</h2>
      <input v-model="keyword" placeholder="搜索商品/分类" class="search" />
      <div class="grid">
        <button v-for="item in filteredProducts" :key="item.id" class="product" @click="addToCart(item)">
          <span class="name">{{ item.name }}</span>
          <small>{{ item.category }}</small>
          <strong>¥ {{ item.price.toFixed(2) }}</strong>
        </button>
      </div>
    </section>

    <section class="checkout panel">
      <h2>收银台</h2>
      <div v-if="cart.length" class="cart-list">
        <div v-for="item in cart" :key="item.id" class="cart-item">
          <div>
            <p>{{ item.name }}</p>
            <small>¥ {{ item.price.toFixed(2) }}</small>
          </div>
          <div class="qty">
            <button @click="decreaseQty(item)">-</button>
            <span>{{ item.qty }}</span>
            <button @click="increaseQty(item)">+</button>
          </div>
          <div class="line-total">¥ {{ (item.qty * item.price).toFixed(2) }}</div>
          <button class="remove" @click="removeItem(item.id)">删除</button>
        </div>
      </div>
      <p v-else class="empty">请选择商品加入购物车</p>

      <div class="totals">
        <p><span>小计</span><strong>¥ {{ subtotal.toFixed(2) }}</strong></p>
        <p><span>税率</span><strong>{{ (taxRate * 100).toFixed(0) }}%</strong></p>
        <p><span>税额</span><strong>¥ {{ tax.toFixed(2) }}</strong></p>
        <p class="grand"><span>应收</span><strong>¥ {{ total.toFixed(2) }}</strong></p>
      </div>

      <label class="pay-row">
        收款金额
        <input v-model="received" type="number" min="0" step="0.01" placeholder="输入实收金额" />
      </label>
      <p class="change" :class="{ danger: change < 0 }">找零：¥ {{ Number.isFinite(change) ? change.toFixed(2) : '0.00' }}</p>

      <div class="actions">
        <button class="secondary" @click="clearCart">清空</button>
        <button class="primary" @click="checkout">结算</button>
      </div>
    </section>

    <section class="history panel">
      <h2>交易记录</h2>
      <p v-if="!orderHistory.length" class="empty">暂无交易</p>
      <ul v-else>
        <li v-for="order in orderHistory.slice(0, 8)" :key="order.id">
          <header>
            <strong>#{{ order.id }}</strong>
            <small>{{ order.createdAt }}</small>
          </header>
          <p>{{ order.items.length }} 件商品，总计 ¥ {{ order.total.toFixed(2) }}，实收 ¥ {{ order.paid.toFixed(2) }}</p>
        </li>
      </ul>
    </section>
  </main>
</template>
