<script setup>
import { ref, computed, onMounted } from 'vue';

// ⚠️ ใส่ URL ใหม่ V2 ของคุณที่นี่
const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbyM6gzH-tWCAm0FQ-upNTyb1siDnQnMiJ_WL5tpegSkyTIhwi55bZzoNL-m2aE1XPIXAA/exec';

// --- State ---
const currentPage = ref('home'); // 'home' หรือ 'categories'
const isLoading = ref(false);
const isSaving = ref(false);

const transactions = ref([]);
const categories = ref([]); // เก็บหมวดหมู่ทั้งหมดที่ดึงมาจาก Sheet

// Filter Date
const currentDate = new Date();
const selectedMonth = ref(currentDate.getMonth());
const selectedYear = ref(currentDate.getFullYear());
const months = ["ม.ค.", "ก.พ.", "มี.ค.", "เม.ย.", "พ.ค.", "มิ.ย.", "ก.ค.", "ส.ค.", "ก.ย.", "ต.ค.", "พ.ย.", "ธ.ค."];

// Form บันทึกเงิน
const form = ref({
  date: new Date().toISOString().split('T')[0],
  type: 'รายจ่าย',
  category: '', // จะเปลี่ยนเป็น Dropdown
  name: '',
  amount: '',
  note: '',
  action: 'saveTransaction' // บอก Backend ว่านี่คือการบันทึกเงิน
});

// Form เพิ่มหมวดหมู่ใหม่
const catForm = ref({
  type: 'รายจ่าย',
  name: '',
  action: 'saveCategory' // บอก Backend ว่านี่คือการเพิ่มหมวดหมู่
});

// --- Actions ---

// 1. ดึงข้อมูลทั้งหมด (รายการ + หมวดหมู่)
const fetchAllData = async () => {
  isLoading.value = true;
  try {
    const res = await fetch(GOOGLE_SCRIPT_URL);
    const data = await res.json();
    transactions.value = data.transactions || [];
    categories.value = data.categories || [];
    
    // ตั้งค่า Default หมวดหมู่แรกให้ dropdown ถ้ายังไม่เลือก
    updateCategoryDefault();
    
  } catch (error) {
    console.error("Error:", error);
  } finally {
    isLoading.value = false;
  }
};

// 2. บันทึกรายการเงิน
const submitTransaction = async () => {
  if(!form.value.category) return alert("กรุณาเลือกหมวดหมู่");
  
  isSaving.value = true;
  try {
    await fetch(GOOGLE_SCRIPT_URL, {
      method: 'POST',
      mode: 'no-cors',
      headers: { 'Content-Type': 'text/plain' },
      body: JSON.stringify(form.value)
    });
    // รีเฟรชข้อมูล
    await fetchAllData();
    // เคลียร์ค่า
    form.value.name = '';
    form.value.amount = '';
  } catch (error) {
    alert("บันทึกผิดพลาด");
  } finally {
    isSaving.value = false;
  }
};

// 3. บันทึกหมวดหมู่ใหม่
const submitCategory = async () => {
  if(!catForm.value.name) return;
  
  isSaving.value = true;
  try {
    await fetch(GOOGLE_SCRIPT_URL, {
      method: 'POST',
      mode: 'no-cors',
      headers: { 'Content-Type': 'text/plain' },
      body: JSON.stringify(catForm.value)
    });
    // รีเฟรชข้อมูล (เพื่อให้หมวดหมู่ใหม่โผล่มาใน Dropdown ทันที)
    await fetchAllData();
    // เคลียร์ค่า
    catForm.value.name = '';
    alert("✅ เพิ่มหมวดหมู่เรียบร้อย");
    currentPage.value = 'home'; // กลับหน้าหลัก
  } catch (error) {
    alert("เกิดข้อผิดพลาด");
  } finally {
    isSaving.value = false;
  }
};

// --- Logic ---

// เลือกหมวดหมู่ที่จะโชว์ใน Dropdown ตาม Type ที่เลือก
const availableCategories = computed(() => {
  return categories.value.filter(c => c.type === form.value.type);
});

// เมื่อเปลี่ยน Type ให้ reset หมวดหมู่เป็นอันแรกของ Type นั้น
const updateCategoryDefault = () => {
  const available = categories.value.filter(c => c.type === form.value.type);
  if (available.length > 0) {
    form.value.category = available[0].name;
  } else {
    form.value.category = '';
  }
};

// คำนวณยอดเงิน (เหมือนเดิม)
const filteredTransactions = computed(() => {
  return transactions.value.filter(t => {
    const tDate = new Date(t.date);
    return tDate.getMonth() === selectedMonth.value && 
           tDate.getFullYear() === selectedYear.value;
  });
});
const totalIncome = computed(() => filteredTransactions.value.filter(t => t.type === 'รายรับ').reduce((sum, t) => sum + Number(t.amount), 0));
const totalExpense = computed(() => filteredTransactions.value.filter(t => t.type === 'รายจ่าย').reduce((sum, t) => sum + Number(t.amount), 0));
const totalSavings = computed(() => filteredTransactions.value.filter(t => t.type === 'เงินออม').reduce((sum, t) => sum + Number(t.amount), 0));
const balance = computed(() => (totalIncome.value - totalExpense.value - totalSavings.value));
const formatDate = (dateString) => {
  const d = new Date(dateString);
  return d.toLocaleDateString('th-TH', { day: 'numeric', month: 'short' });
};

onMounted(() => {
  fetchAllData();
});
</script>

<template>
  <div class="min-h-screen bg-gray-100 p-4 font-sans">
    <div class="max-w-md mx-auto">
      
      <div class="flex justify-between items-center mb-6">
        <h1 class="text-2xl font-bold text-gray-800">💰 Money V2</h1>
        
        <button v-if="currentPage === 'home'" @click="currentPage = 'categories'" 
          class="bg-gray-200 text-gray-700 px-3 py-1 rounded-lg text-sm font-medium hover:bg-gray-300">
          ⚙️ จัดการหมวดหมู่
        </button>
        <button v-else @click="currentPage = 'home'" 
          class="bg-gray-200 text-gray-700 px-3 py-1 rounded-lg text-sm font-medium hover:bg-gray-300">
          ⬅️ กลับหน้าหลัก
        </button>
      </div>

      <div v-if="currentPage === 'home'" class="space-y-6">
        
        <div class="flex gap-2 bg-white p-2 rounded-lg shadow-sm justify-center">
          <select v-model="selectedMonth" class="bg-transparent font-bold text-gray-700 outline-none"><option v-for="(m, i) in months" :key="i" :value="i">{{ m }}</option></select>
          <select v-model="selectedYear" class="bg-transparent font-bold text-gray-700 outline-none"><option v-for="y in 5" :key="y" :value="2024 + y">{{ 2024 + y }}</option></select>
        </div>

        <div class="grid grid-cols-3 gap-2">
          <div class="bg-green-100 p-3 rounded-xl text-center"><p class="text-xs text-green-600 font-bold">รับ</p><p class="text-lg font-bold text-green-800">{{ totalIncome.toLocaleString() }}</p></div>
          <div class="bg-red-100 p-3 rounded-xl text-center"><p class="text-xs text-red-600 font-bold">จ่าย</p><p class="text-lg font-bold text-red-800">{{ totalExpense.toLocaleString() }}</p></div>
          <div class="bg-white border p-3 rounded-xl text-center"><p class="text-xs text-gray-500 font-bold">เหลือ</p><p class="text-lg font-bold">{{ balance.toLocaleString() }}</p></div>
        </div>

        <div class="bg-white rounded-2xl shadow-sm p-5">
          <h2 class="font-bold text-gray-700 mb-3">📝 บันทึกใหม่</h2>
          <form @submit.prevent="submitTransaction" class="space-y-3">
            
            <div class="grid grid-cols-3 gap-2 bg-gray-100 p-1 rounded-lg">
              <button type="button" v-for="t in ['รายจ่าย','รายรับ','เงินออม']" :key="t"
                @click="form.type = t; updateCategoryDefault()"
                :class="form.type === t ? 'bg-white shadow text-black' : 'text-gray-500'"
                class="py-2 rounded-md text-sm font-bold transition-all">
                {{ t }}
              </button>
            </div>

            <div class="flex gap-2">
              <input type="date" v-model="form.date" required class="w-1/3 p-2 border rounded-lg bg-gray-50 text-sm">
              
              <div class="w-2/3 relative">
                <select v-model="form.category" required class="w-full p-2 border rounded-lg bg-white appearance-none text-sm">
                  <option disabled value="">เลือกหมวดหมู่</option>
                  <option v-for="(cat, i) in availableCategories" :key="i" :value="cat.name">
                    {{ cat.name }}
                  </option>
                </select>
                <div class="pointer-events-none absolute inset-y-0 right-0 flex items-center px-2 text-gray-700">▼</div>
              </div>
            </div>

            <input type="text" v-model="form.name" placeholder="ชื่อรายการ (ระบุหรือไม่ก็ได้)" class="w-full p-2 border rounded-lg text-sm">
            
            <input type="number" v-model="form.amount" placeholder="0.00" required inputmode="decimal" class="w-full p-3 border rounded-lg text-xl font-bold text-right text-blue-600 bg-gray-50">

            <button type="submit" :disabled="isSaving" class="w-full bg-black text-white py-3 rounded-xl font-bold hover:bg-gray-800 disabled:bg-gray-400">
              {{ isSaving ? '⏳ กำลังบันทึก...' : 'บันทึก' }}
            </button>
          </form>
        </div>

        <div class="bg-white rounded-2xl shadow-sm p-5">
           <h3 class="font-bold text-gray-700 mb-3 text-sm">ประวัติล่าสุด ({{ months[selectedMonth] }})</h3>
           <div class="space-y-3 max-h-[300px] overflow-y-auto">
              <div v-for="(item, index) in filteredTransactions" :key="index" class="flex justify-between items-center p-2 border-b last:border-0">
                <div class="flex items-center gap-3">
                   <div class="w-8 h-8 rounded-full flex items-center justify-center text-sm" :class="item.type==='รายจ่าย'?'bg-red-100':(item.type==='รายรับ'?'bg-green-100':'bg-blue-100')">
                      {{ item.type==='รายจ่าย'?'💸':(item.type==='รายรับ'?'💰':'🐷') }}
                   </div>
                   <div>
                      <p class="font-bold text-sm">{{ item.category }}</p>
                      <p class="text-xs text-gray-400">{{ formatDate(item.date) }} <span v-if="item.name">- {{ item.name }}</span></p>
                   </div>
                </div>
                <span class="font-bold text-sm" :class="item.type==='รายจ่าย'?'text-red-500':'text-green-500'">
                  {{ item.type==='รายจ่าย'?'-':'+' }}{{ Number(item.amount).toLocaleString() }}
                </span>
              </div>
           </div>
        </div>
      </div>

      <div v-else class="bg-white rounded-2xl shadow-sm p-6 space-y-6">
        <h2 class="text-xl font-bold text-gray-800">⚙️ จัดการหมวดหมู่</h2>
        
        <div class="bg-gray-50 p-4 rounded-xl border">
          <h3 class="font-bold text-sm text-gray-600 mb-2">เพิ่มหมวดหมู่ใหม่</h3>
          <form @submit.prevent="submitCategory" class="space-y-3">
             <div class="grid grid-cols-3 gap-2">
                <button type="button" v-for="t in ['รายจ่าย','รายรับ','เงินออม']" :key="t"
                  @click="catForm.type = t"
                  :class="catForm.type === t ? 'bg-blue-600 text-white' : 'bg-white border text-gray-500'"
                  class="py-2 rounded-md text-xs font-bold">
                  {{ t }}
                </button>
             </div>
             <input type="text" v-model="catForm.name" placeholder="ชื่อหมวดหมู่ใหม่ (เช่น ค่าเทอม, ถูกหวย)" required class="w-full p-2 border rounded-lg">
             <button type="submit" :disabled="isSaving" class="w-full bg-blue-600 text-white py-2 rounded-lg font-bold text-sm hover:bg-blue-700">
                {{ isSaving ? '⏳ กำลังบันทึก...' : '+ เพิ่มหมวดหมู่' }}
             </button>
          </form>
        </div>

        <div>
          <h3 class="font-bold text-sm text-gray-600 mb-2">หมวดหมู่ทั้งหมด</h3>
          <div class="space-y-2 max-h-[400px] overflow-y-auto">
             <div v-for="(cat, i) in categories" :key="i" class="flex justify-between p-2 bg-gray-50 rounded-lg border text-sm">
                <span>{{ cat.name }}</span>
                <span class="text-xs px-2 py-1 rounded bg-white border" :class="cat.type==='รายจ่าย'?'text-red-500':'text-green-500'">{{ cat.type }}</span>
             </div>
          </div>
        </div>
      </div>

    </div>
  </div>
</template>