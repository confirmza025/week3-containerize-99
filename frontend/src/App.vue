<script setup>
import { ref, computed, onMounted } from 'vue'

const products       = ref([])
const loading        = ref(true)
const search         = ref('')
const platformFilter = ref('')
const showModal      = ref(false)
const editingId      = ref(null)
const confirmDelete  = ref(null)

const form = ref({ name: '', platform: '', price: '', stock: 0, description: '' })
let debounceTimer = null

const filtered = computed(() => {
  const s = search.value.toLowerCase()
  return products.value.filter(p => {
    const matchS = !s || p.name.toLowerCase().includes(s) || (p.description || '').toLowerCase().includes(s)
    const matchP = !platformFilter.value || p.platform === platformFilter.value
    return matchS && matchP
  })
})

const platforms = computed(() => [...new Set(products.value.map(p => p.platform))].sort())

const stats = computed(() => ({
  total:      products.value.length,
  lowStock:   products.value.filter(p => p.stock < 10).length,
  totalKeys:  products.value.reduce((s, p) => s + p.stock, 0),
  totalValue: products.value.reduce((s, p) => s + parseFloat(p.price) * p.stock, 0)
}))

function debounceFetch() {
  clearTimeout(debounceTimer)
  debounceTimer = setTimeout(fetchProducts, 280)
}

async function fetchProducts() {
  loading.value = true
  const q = new URLSearchParams()
  if (search.value)         q.set('search',   search.value)
  if (platformFilter.value) q.set('platform', platformFilter.value)
  try {
    const res = await fetch(`/api/products?${q}`)
    products.value = await res.json()
  } catch {
    products.value = []
  } finally {
    loading.value = false
  }
}

function openAdd() {
  editingId.value = null
  form.value = { name: '', platform: '', price: '', stock: 0, description: '' }
  showModal.value = true
}

function openEdit(p) {
  editingId.value = p.id
  form.value = { name: p.name, platform: p.platform, price: p.price, stock: p.stock, description: p.description || '' }
  showModal.value = true
}

async function saveProduct() {
  const body = {
    name: form.value.name, platform: form.value.platform,
    price: parseFloat(form.value.price), stock: parseInt(form.value.stock),
    description: form.value.description
  }
  const url    = editingId.value ? `/api/products/${editingId.value}` : '/api/products'
  const method = editingId.value ? 'PUT' : 'POST'
  await fetch(url, { method, headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body) })
  showModal.value = false
  fetchProducts()
}

async function deleteProduct(id) {
  await fetch(`/api/products/${id}`, { method: 'DELETE' })
  confirmDelete.value = null
  fetchProducts()
}

function stockClass(s) {
  if (s <= 0) return 'out'
  if (s < 10) return 'low'
  if (s < 30) return 'mid'
  return 'high'
}

function platformMeta(plat) {
  const map = {
    'Steam':       { icon: '🎮', cls: 'p-steam' },
    'Epic Games':  { icon: '⚡', cls: 'p-epic' },
    'PlayStation': { icon: '🎯', cls: 'p-ps' },
    'Xbox':        { icon: '🟢', cls: 'p-xbox' },
    'Nintendo':    { icon: '🔴', cls: 'p-nintendo' },
    'Blizzard':    { icon: '❄️', cls: 'p-blizzard' },
    'EA':          { icon: '🔷', cls: 'p-ea' },
    'Ubisoft':     { icon: '🔶', cls: 'p-ubisoft' },
  }
  return map[plat] || { icon: '🕹️', cls: 'p-other' }
}

onMounted(fetchProducts)
</script>

<template>
  <div class="kv-root">

    <header class="kv-header">
      <div class="kv-logo">
        <span class="kv-glyph">⌗</span>
        <div>
          <div class="kv-name">KeyVault</div>
          <div class="kv-sub">Game Key Management</div>
        </div>
      </div>
      <button class="kv-btn-add" @click="openAdd">+ เพิ่มสินค้า</button>
    </header>

    <main class="kv-main">

      <div class="kv-stats">
        <div class="kv-stat s-purple">
          <div class="kv-stat-deco">KEYS</div>
          <div class="kv-stat-num">{{ stats.total }}</div>
          <div class="kv-stat-label">รหัสเกมทั้งหมด</div>
        </div>
        <div class="kv-stat s-amber">
          <div class="kv-stat-deco">ALERT</div>
          <div class="kv-stat-num">{{ stats.lowStock }}</div>
          <div class="kv-stat-label">สต็อกใกล้หมด</div>
        </div>
        <div class="kv-stat s-gray">
          <div class="kv-stat-deco">TOTAL</div>
          <div class="kv-stat-num">{{ stats.totalKeys.toLocaleString() }}</div>
          <div class="kv-stat-label">คีย์คงเหลือรวม</div>
        </div>
        <div class="kv-stat s-green">
          <div class="kv-stat-deco">VALUE</div>
          <div class="kv-stat-num kv-stat-val">฿{{ Math.round(stats.totalValue).toLocaleString() }}</div>
          <div class="kv-stat-label">มูลค่าสต็อก</div>
        </div>
      </div>

      <div class="kv-alert" v-if="stats.lowStock > 0">
        <span class="kv-pip"></span>
        สินค้า <strong>{{ stats.lowStock }} รายการ</strong> มีคีย์เหลือน้อยกว่า 10 — เติมสต็อกด่วน
      </div>

      <div class="kv-toolbar">
        <div class="kv-search-wrap">
          <span class="kv-search-icon">⌕</span>
          <input v-model="search" @input="debounceFetch" class="kv-input" placeholder="ค้นหาชื่อเกม..." />
        </div>
        <select v-model="platformFilter" @change="fetchProducts" class="kv-select">
          <option value="">ทุกแพลตฟอร์ม</option>
          <option v-for="p in platforms" :key="p" :value="p">{{ p }}</option>
        </select>
        <span class="kv-count" v-if="!loading">{{ filtered.length }} / {{ products.length }}</span>
      </div>

      <div class="kv-state" v-if="loading">
        <div class="kv-spinner"></div>
        <div class="kv-state-txt">กำลังโหลด...</div>
      </div>

      <div class="kv-state" v-else-if="filtered.length === 0">
        <div class="kv-state-glyph">⌗</div>
        <div class="kv-state-title">ไม่พบสินค้า</div>
        <div class="kv-state-desc">{{ search || platformFilter ? 'ลองเปลี่ยน filter' : 'กด "+ เพิ่มสินค้า" เพื่อเริ่มต้น' }}</div>
      </div>

      <div class="kv-grid" v-else>
        <div
          v-for="p in filtered" :key="p.id"
          class="kv-card"
          :class="{ 'kv-card-low': p.stock > 0 && p.stock < 10, 'kv-card-out': p.stock <= 0 }"
        >
          <div class="kv-card-top">
            <span class="kv-plat" :class="platformMeta(p.platform).cls">
              {{ platformMeta(p.platform).icon }} {{ p.platform }}
            </span>
            <span class="kv-cid">#{{ String(p.id).padStart(4,'0') }}</span>
          </div>
          <div class="kv-card-body">
            <div class="kv-pname">{{ p.name }}</div>
            <div class="kv-pdesc" v-if="p.description">{{ p.description }}</div>
            <div class="kv-keymask">
              <span class="kv-kseg">XXXXX</span>–<span class="kv-kseg">XXXXX</span>–<span class="kv-kseg">XXXXX</span>
            </div>
            <div class="kv-price-row">
              <span class="kv-price">฿{{ parseFloat(p.price).toLocaleString('th-TH',{minimumFractionDigits:2}) }}</span>
              <span class="kv-perkey">/ คีย์</span>
            </div>
            <div class="kv-stock">
              <div class="kv-stock-row">
                <span class="kv-slabel">คีย์คงเหลือ</span>
                <span class="kv-snum" :class="stockClass(p.stock)">
                  {{ p.stock.toLocaleString() }}
                  <span v-if="p.stock<=0"> · หมดแล้ว</span>
                  <span v-else-if="p.stock<10"> · ใกล้หมด</span>
                </span>
              </div>
              <div class="kv-bar">
                <div class="kv-fill" :class="stockClass(p.stock)"
                  :style="{width: Math.min((p.stock/80)*100,100)+'%'}"></div>
              </div>
            </div>
          </div>
          <div class="kv-card-foot">
            <button class="kv-bedit" @click="openEdit(p)">✏ แก้ไข</button>
            <button class="kv-bdel"  @click="confirmDelete=p">✕ ลบ</button>
          </div>
        </div>
      </div>

    </main>

    <!-- MODAL ADD/EDIT -->
    <div class="kv-overlay" v-if="showModal" @click.self="showModal=false">
      <div class="kv-modal">
        <div class="kv-modal-hd">
          <span class="kv-glyph" style="font-size:1.2rem">⌗</span>
          <span class="kv-modal-title">{{ editingId ? 'แก้ไขสินค้า' : 'เพิ่มสินค้าใหม่' }}</span>
        </div>
        <form @submit.prevent="saveProduct">
          <div class="kv-fg">
            <label class="kv-fl">ชื่อเกม <span class="kv-req">*</span></label>
            <input class="kv-fi" v-model="form.name" placeholder="เช่น Elden Ring" required />
          </div>
          <div class="kv-frow">
            <div class="kv-fg">
              <label class="kv-fl">แพลตฟอร์ม <span class="kv-req">*</span></label>
              <input class="kv-fi" v-model="form.platform" list="kv-plats" placeholder="Steam…" required />
              <datalist id="kv-plats">
                <option v-for="pl in platforms" :value="pl" :key="pl" />
              </datalist>
            </div>
            <div class="kv-fg">
              <label class="kv-fl">ราคา (฿) <span class="kv-req">*</span></label>
              <input class="kv-fi" v-model="form.price" type="number" min="0" step="1" placeholder="0" required />
            </div>
          </div>
          <div class="kv-fg">
            <label class="kv-fl">จำนวนคีย์ <span class="kv-req">*</span></label>
            <input class="kv-fi" v-model="form.stock" type="number" min="0" placeholder="0" required />
          </div>
          <div class="kv-fg">
            <label class="kv-fl">คำอธิบาย</label>
            <textarea class="kv-fi" v-model="form.description" rows="3" placeholder="รายละเอียดเกม" style="resize:vertical"></textarea>
          </div>
          <div class="kv-mfoot">
            <button type="button" class="kv-bcancel" @click="showModal=false">ยกเลิก</button>
            <button type="submit" class="kv-bsave">{{ editingId ? 'บันทึก' : 'เพิ่มสินค้า' }}</button>
          </div>
        </form>
      </div>
    </div>

    <!-- DELETE CONFIRM -->
    <div class="kv-overlay" v-if="confirmDelete" @click.self="confirmDelete=null">
      <div class="kv-modal kv-confirm">
        <div class="kv-cx">✕</div>
        <div class="kv-ctitle">ลบสินค้า?</div>
        <div class="kv-cdesc">
          <strong>{{ confirmDelete?.name }}</strong><br>
          <span class="kv-cwarn">การกระทำนี้ย้อนกลับไม่ได้</span>
        </div>
        <div class="kv-cact">
          <button class="kv-bcancel" @click="confirmDelete=null">ยกเลิก</button>
          <button class="kv-bdanger" @click="deleteProduct(confirmDelete.id)">ลบเลย</button>
        </div>
      </div>
    </div>

  </div>
</template>

<style scoped>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}

.kv-root{
  background:#0d0d14 !important;
  min-height:100vh;
  color:#e2e2f0;
  font-family:'Inter',system-ui,sans-serif;
}

/* HEADER */
.kv-header{
  position:sticky;top:0;z-index:100;
  background:#0d0d14;
  border-bottom:1px solid #1f1f30;
  height:60px;padding:0 1.5rem;
  display:flex;align-items:center;gap:.75rem;
}
.kv-logo{display:flex;align-items:center;gap:.65rem}
.kv-glyph{font-family:'Courier New',monospace;font-size:1.6rem;font-weight:900;color:#a78bfa;line-height:1}
.kv-name{font-weight:700;font-size:1.05rem;color:#e2e2f0;letter-spacing:.04em}
.kv-sub{font-size:.66rem;color:#4a4a6a;letter-spacing:.1em;text-transform:uppercase}
.kv-btn-add{
  margin-left:auto;
  background:#7c3aed;color:#fff;
  border:none;border-radius:6px;
  padding:.48rem 1.1rem;font-size:.84rem;font-weight:600;
  cursor:pointer;letter-spacing:.02em;
  transition:background .15s,transform .1s;
}
.kv-btn-add:hover{background:#9f67fa;transform:translateY(-1px)}

/* MAIN */
.kv-main{max-width:1240px;margin:0 auto;padding:1.75rem 1.5rem}

/* STATS */
.kv-stats{
  display:grid;
  grid-template-columns:repeat(auto-fit,minmax(190px,1fr));
  gap:.75rem;margin-bottom:1.25rem;
}
.kv-stat{
  background:#111120;
  border:1px solid #1f1f30;
  border-radius:10px;padding:1.1rem 1.25rem;
  position:relative;overflow:hidden;
}
.kv-stat-deco{
  position:absolute;right:.85rem;top:.7rem;
  font-size:.6rem;letter-spacing:.2em;font-weight:700;
  font-family:'Courier New',monospace;opacity:.14;
}
.kv-stat-num{font-size:2rem;font-weight:800;font-family:'Courier New',monospace;line-height:1}
.kv-stat-val{font-size:1.45rem}
.kv-stat-label{font-size:.72rem;margin-top:.35rem;letter-spacing:.05em;text-transform:uppercase;color:#fbbf24}

.s-purple .kv-stat-num,.s-purple .kv-stat-deco{color:#a78bfa}
.s-purple{border-color:#2d1f5e}
.s-amber .kv-stat-num,.s-amber .kv-stat-deco{color:#f59e0b}
.s-amber{border-color:#3d2800}
.s-gray .kv-stat-num{color:#c0c0d8}
.s-green .kv-stat-num,.s-green .kv-stat-deco{color:#34d399}
.s-green{border-color:#0d3322}
.kv-stat-label{color:#55556a}

/* ALERT */
.kv-alert{
  display:flex;align-items:center;gap:.75rem;
  background:#1e1200;border:1px solid #4a3000;
  border-radius:8px;padding:.7rem 1.1rem;
  font-size:.85rem;color:#f59e0b;margin-bottom:1.25rem;
}
.kv-pip{
  width:7px;height:7px;border-radius:50%;
  background:#f59e0b;flex-shrink:0;
  animation:kv-pulse 1.5s ease-in-out infinite;
}
@keyframes kv-pulse{0%,100%{opacity:1;transform:scale(1)}50%{opacity:.3;transform:scale(.65)}}

/* TOOLBAR */
.kv-toolbar{display:flex;gap:.6rem;margin-bottom:1.5rem;flex-wrap:wrap;align-items:center}
.kv-search-wrap{flex:1;min-width:200px;position:relative}
.kv-search-icon{
  position:absolute;left:.85rem;top:50%;transform:translateY(-50%);
  color:#3a3a50;font-size:1.05rem;pointer-events:none;
}
.kv-input{
  width:100%;padding:.58rem 1rem .58rem 2.2rem;
  background:#111120;color:#e2e2f0;
  border:1px solid #1f1f30;border-radius:7px;
  font-size:.88rem;outline:none;transition:border-color .15s;
}
.kv-input:focus{border-color:#7c3aed}
.kv-input::placeholder{color:#3a3a55}
.kv-select{
  padding:.58rem .9rem;
  background:#111120;color:#e2e2f0;
  border:1px solid #1f1f30;border-radius:7px;
  font-size:.86rem;outline:none;cursor:pointer;
}
.kv-select:focus{border-color:#7c3aed}
.kv-count{font-size:.76rem;color:#3a3a55;font-family:'Courier New',monospace;white-space:nowrap}

/* STATE */
.kv-state{text-align:center;padding:5rem 1rem}
.kv-spinner{
  width:34px;height:34px;margin:0 auto .75rem;
  border:2px solid #1f1f30;border-top-color:#a78bfa;
  border-radius:50%;animation:kv-spin .7s linear infinite;
}
@keyframes kv-spin{to{transform:rotate(360deg)}}
.kv-state-txt{color:#3a3a55;font-size:.88rem}
.kv-state-glyph{font-size:2.2rem;color:#2a2a40;font-family:'Courier New',monospace;margin-bottom:.5rem}
.kv-state-title{font-size:1rem;font-weight:600;color:#4a4a65;margin-bottom:.3rem}
.kv-state-desc{font-size:.82rem;color:#2a2a40}

/* GRID */
.kv-grid{
  display:grid;
  grid-template-columns:repeat(auto-fill,minmax(280px,1fr));
  gap:1rem;
}
.kv-card{
  background:#111120;
  border:1px solid #1f1f30;
  border-radius:12px;overflow:hidden;
  display:flex;flex-direction:column;
  transition:border-color .18s,transform .15s;
}
.kv-card:hover{border-color:#2e2e50;transform:translateY(-2px)}
.kv-card-low{border-color:#3d2800 !important}
.kv-card-out{opacity:.45}

.kv-card-top{
  display:flex;align-items:center;justify-content:space-between;
  padding:.55rem .85rem;
  background:#0d0d14;
  border-bottom:1px solid #1a1a28;
}
.kv-plat{
  font-size:.7rem;font-weight:700;
  padding:.2rem .65rem;border-radius:20px;
  letter-spacing:.03em;
}
.p-steam    {background:#1c2e4a;color:#5dade2}
.p-epic     {background:#1a1a2e;color:#a78bfa}
.p-ps       {background:#1a203e;color:#60a5fa}
.p-xbox     {background:#0d2b0d;color:#4ade80}
.p-nintendo {background:#3b0a0a;color:#f87171}
.p-blizzard {background:#0d1e3d;color:#93c5fd}
.p-ea       {background:#2a1500;color:#fb923c}
.p-ubisoft  {background:#1e1a00;color:#fbbf24}
.p-other    {background:#1a1a28;color:#fbbf24}
.kv-cid{font-family:'Courier New',monospace;font-size:.66rem;color:#2e2e45}

.kv-card-body{padding:.9rem;flex:1}
.kv-pname{font-size:.95rem;font-weight:700;color:#e2e2f0;line-height:1.35;margin-bottom:.3rem}
.kv-pdesc{font-size:.77rem;color:#5a5a75;line-height:1.55;margin-bottom:.7rem}
.kv-keymask{
  font-family:'Courier New',monospace;font-size:.73rem;
  color:#2e2e45;letter-spacing:.1em;margin-bottom:.7rem;
  display:flex;align-items:center;gap:.15rem;
}
.kv-kseg{
  background:#0d0d14;border:1px solid #1a1a28;
  border-radius:4px;padding:.18rem .45rem;
}
.kv-price-row{display:flex;align-items:baseline;gap:.35rem;margin-bottom:.85rem}
.kv-price{font-size:1.2rem;font-weight:800;color:#a78bfa;font-family:'Courier New',monospace}
.kv-perkey{font-size:.72rem;color:#3a3a50}

.kv-stock-row{display:flex;justify-content:space-between;font-size:.77rem;margin-bottom:.28rem}
.kv-slabel{color:#2e2e45;text-transform:uppercase;letter-spacing:.07em;font-size:.66rem}
.kv-snum{font-family:'Courier New',monospace;font-weight:700}
.kv-snum.out{color:#2e2e45}
.kv-snum.low{color:#f59e0b}
.kv-snum.mid{color:#a78bfa}
.kv-snum.high{color:#34d399}
.kv-bar{height:4px;background:#0d0d14;border-radius:2px;overflow:hidden}
.kv-fill{height:100%;border-radius:2px;transition:width .4s ease;min-width:3px}
.kv-fill.out{background:#1f1f30;width:2% !important}
.kv-fill.low{background:#f59e0b}
.kv-fill.mid{background:#7c3aed}
.kv-fill.high{background:#34d399}

.kv-card-foot{
  display:flex;gap:.45rem;padding:.6rem .85rem;
  border-top:1px solid #1a1a28;background:#0d0d14;
}
.kv-bedit,.kv-bdel{
  flex:1;padding:.4rem;border-radius:6px;
  font-size:.78rem;font-weight:600;cursor:pointer;
  border:1px solid #1f1f30;background:transparent;
  color:#4a4a65;letter-spacing:.02em;transition:all .15s;
}
.kv-bedit:hover{background:#2d1f5e;border-color:#7c3aed;color:#a78bfa}
.kv-bdel:hover{background:#2a0d0d;border-color:#f87171;color:#f87171}

/* OVERLAY */
.kv-overlay{
  position:fixed;inset:0;
  background:rgba(0,0,0,.75);
  backdrop-filter:blur(6px);
  display:flex;align-items:center;justify-content:center;
  z-index:500;padding:1rem;
}
.kv-modal{
  background:#111120;
  border:1px solid #2a2a3e;
  border-radius:14px;
  width:100%;max-width:480px;
  max-height:90vh;overflow-y:auto;
  padding:1.75rem;
}
.kv-modal-hd{display:flex;align-items:center;gap:.5rem;margin-bottom:1.35rem}
.kv-modal-title{font-size:1.05rem;font-weight:700;color:#e2e2f0}
.kv-fg{margin-bottom:.85rem}
.kv-fl{display:block;font-size:.75rem;font-weight:600;color:#4a4a65;margin-bottom:.32rem;letter-spacing:.05em;text-transform:uppercase}
.kv-req{color:#a78bfa}
.kv-fi{
  width:100%;padding:.56rem .85rem;
  background:#0d0d14;color:#e2e2f0;
  border:1px solid #1f1f30;border-radius:7px;
  font-size:.9rem;outline:none;transition:border-color .15s;
}
.kv-fi:focus{border-color:#7c3aed}
.kv-fi::placeholder{color:#2e2e45}
.kv-frow{display:grid;grid-template-columns:1fr 1fr;gap:.65rem}
.kv-mfoot{display:flex;gap:.6rem;justify-content:flex-end;padding-top:.85rem;border-top:1px solid #1a1a28;margin-top:.85rem}
.kv-bcancel{background:transparent;color:#4a4a65;border:1px solid #1f1f30;border-radius:6px;padding:.48rem 1.1rem;font-size:.84rem;font-weight:600;cursor:pointer}
.kv-bcancel:hover{background:#1a1a28;color:#e2e2f0}
.kv-bsave{background:#7c3aed;color:#fff;border:none;border-radius:6px;padding:.48rem 1.3rem;font-size:.84rem;font-weight:700;cursor:pointer}
.kv-bsave:hover{background:#9f67fa}

/* CONFIRM */
.kv-confirm{text-align:center;max-width:360px;padding:2rem}
.kv-cx{
  width:50px;height:50px;border-radius:50%;
  background:#2a0d0d;color:#f87171;
  font-size:1.2rem;font-weight:700;
  display:flex;align-items:center;justify-content:center;
  margin:0 auto .85rem;
}
.kv-ctitle{font-size:1.05rem;font-weight:700;margin-bottom:.5rem;color:#e2e2f0}
.kv-cdesc{font-size:.85rem;color:#5a5a75;margin-bottom:1.4rem;line-height:1.6}
.kv-cwarn{color:#f87171;font-size:.8rem}
.kv-cact{display:flex;gap:.65rem;justify-content:center}
.kv-bdanger{background:#c0392b;color:#fff;border:none;border-radius:6px;padding:.48rem 1.3rem;font-size:.84rem;font-weight:700;cursor:pointer}
.kv-bdanger:hover{background:#e74c3c}

@media(max-width:640px){
  .kv-frow{grid-template-columns:1fr}
  .kv-stats{grid-template-columns:1fr 1fr}
  .kv-main{padding:1.25rem 1rem}
}
</style>