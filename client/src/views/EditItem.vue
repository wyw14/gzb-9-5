<template>
  <div v-if="loading" style="text-align:center;padding:60px;color:white;">
    加载中...
  </div>

  <div v-else-if="!item" class="empty-state card">
    <h2>物品不存在</h2>
    <router-link to="/my-items">
      <button class="btn btn-primary">返回我的物品</button>
    </router-link>
  </div>

  <div v-else class="card" style="max-width:700px;margin:0 auto;">
    <div style="display:flex;justify-content:space-between;align-items:center;margin-bottom:16px;">
      <div>
        <h1 style="margin-bottom:8px;">{{ isEditable ? '编辑盲盒' : '盲盒详情' }}</h1>
        <p style="color:#666;margin-bottom:0;">
          {{ isEditable ? '修改你的盲盒信息' : '该盲盒已完成交换，仅可查看' }}
        </p>
      </div>
      <span :class="item.status === 'available' ? 'badge badge-available' : 'badge badge-exchanged'">
        {{ item.status === 'available' ? '可交换' : '已交换' }}
      </span>
    </div>

    <form v-if="isEditable" @submit.prevent="handleSubmit">
      <div class="form-group">
        <label>物品类别 *</label>
        <select v-model="form.category" required>
          <option value="">请选择类别</option>
          <option value="book">书籍类</option>
          <option value="figure">手办类</option>
          <option value="toy">玩具类</option>
          <option value="game">游戏类</option>
          <option value="digital">数码类</option>
          <option value="other">其他</option>
        </select>
      </div>

      <div class="form-group">
        <label>神秘标签 *（用逗号分隔）</label>
        <input v-model="tagsInput" placeholder="例如：全新,限量,珍藏版" required />
        <small style="color:#999;">其他用户只能看到这些标签来猜测物品内容</small>
      </div>

      <div class="form-group">
        <label>真实物品名称 *（仅交换成功后对方可见）</label>
        <input v-model="form.realName" placeholder="例如：《三体》全集精装版" required />
      </div>

      <div class="form-group">
        <label>物品详细描述（仅交换成功后对方可见）</label>
        <textarea v-model="form.description" rows="4" placeholder="描述物品的成色、特点等信息"></textarea>
      </div>

      <div class="form-group">
        <label>物品图片</label>
        <input type="file" accept="image/*" @change="handleFileChange" />
        <div style="margin-top:12px;display:flex;gap:16px;align-items:center;">
          <div>
            <p style="color:#666;font-size:13px;margin-bottom:8px;">当前图片：</p>
            <img :src="appendAuth(item.image)" style="max-width:150px;border-radius:8px;"/>
          </div>
          <div v-if="previewUrl">
            <p style="color:#666;font-size:13px;margin-bottom:8px;">新图片预览（将替换）：</p>
            <img :src="previewUrl" style="max-width:150px;border-radius:8px;"/>
          </div>
        </div>
        <small style="color:#999;">不上传则保留原图片；若替换将重新生成模糊图</small>
      </div>

      <div class="form-group">
        <label>联系方式 *（交换成功后对方可见）</label>
        <input v-model="form.contact" placeholder="例如：微信 xxx 或 QQ xxx" required />
      </div>

      <div style="display:flex;gap:12px;">
        <button type="submit" class="btn btn-primary" style="flex:1;" :disabled="submitting">
          {{ submitting ? '保存中...' : '保存修改' }}
        </button>
        <router-link to="/my-items" style="flex:1;">
          <button type="button" class="btn btn-secondary" style="width:100%;">取消</button>
        </router-link>
      </div>
    </form>

    <div v-else>
      <div class="form-group">
        <label>物品类别</label>
        <p style="padding:10px;background:#f5f7fa;border-radius:8px;margin:0;">{{ getCategoryName(item.category) }}</p>
      </div>

      <div class="form-group">
        <label>神秘标签</label>
        <div>
          <span v-for="tag in item.mysteryTags" :key="tag" class="tag">{{ tag }}</span>
        </div>
      </div>

      <div class="form-group">
        <label>真实物品名称</label>
        <p style="padding:10px;background:#f5f7fa;border-radius:8px;margin:0;">{{ item.realName }}</p>
      </div>

      <div class="form-group" v-if="item.description">
        <label>物品详细描述</label>
        <p style="padding:10px;background:#f5f7fa;border-radius:8px;margin:0;line-height:1.6;">{{ item.description }}</p>
      </div>

      <div class="form-group">
        <label>物品图片</label>
        <div>
          <img :src="appendAuth(item.image)" style="max-width:250px;border-radius:8px;"/>
        </div>
      </div>

      <div class="form-group">
        <label>联系方式</label>
        <p style="padding:10px;background:#f5f7fa;border-radius:8px;margin:0;">{{ item.contact }}</p>
      </div>

      <div style="display:flex;gap:12px;">
        <router-link to="/my-items" style="flex:1;">
          <button type="button" class="btn btn-secondary" style="width:100%;">返回我的物品</button>
        </router-link>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, onMounted } from 'vue'
import { useRoute, useRouter } from 'vue-router'
import { getItemDetail, updateItem, appendAuth } from '../api/index.js'
import { userStore } from '../store/user.js'

const route = useRoute()
const router = useRouter()

const item = ref(null)
const loading = ref(true)
const submitting = ref(false)
const imageFile = ref(null)
const previewUrl = ref('')

const categories = {
  book: '书籍类',
  figure: '手办类',
  toy: '玩具类',
  game: '游戏类',
  digital: '数码类',
  other: '其他'
}

const form = ref({
  category: '',
  realName: '',
  description: '',
  contact: ''
})

const tagsInput = ref('')

const isEditable = computed(function() {
  return item.value && item.value.status === 'available' && item.value.ownerId === userStore.user.id
})

function getCategoryName(key) {
  return categories[key] || key
}

function handleFileChange(e) {
  const file = e.target.files[0]
  if (file) {
    imageFile.value = file
    previewUrl.value = URL.createObjectURL(file)
  }
}

const mysteryTags = computed(function() {
  return tagsInput.value
    .split(/[,，]/)
    .map(function(t) { return t.trim() })
    .filter(function(t) { return t.length > 0 })
})

async function loadItem() {
  loading.value = true
  try {
    const data = await getItemDetail(route.params.id, userStore.user.id)
    item.value = data
    if (data.ownerId !== userStore.user.id) {
      router.push('/my-items')
      return
    }
    form.value.category = data.category
    form.value.realName = data.realName
    form.value.description = data.description || ''
    form.value.contact = data.contact
    tagsInput.value = (data.mysteryTags || []).join(', ')
  } catch (e) {
    console.error(e)
    alert('加载失败')
  } finally {
    loading.value = false
  }
}

async function handleSubmit() {
  if (mysteryTags.value.length === 0) {
    alert('请至少填写一个神秘标签')
    return
  }

  submitting.value = true
  try {
    const formData = new FormData()
    formData.append('userId', userStore.user.id)
    formData.append('category', form.value.category)
    formData.append('mysteryTags', JSON.stringify(mysteryTags.value))
    formData.append('realName', form.value.realName)
    formData.append('description', form.value.description)
    formData.append('contact', form.value.contact)
    if (imageFile.value) {
      formData.append('image', imageFile.value)
    }

    await updateItem(route.params.id, formData)
    alert('保存成功！')
    router.push('/my-items')
  } catch (e) {
    alert('保存失败：' + (e.response && e.response.data ? e.response.data.error : e.message))
  } finally {
    submitting.value = false
  }
}

onMounted(loadItem)
</script>
