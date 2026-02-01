<script setup>
import { ref, computed, onMounted } from 'vue';

// ⚠️ ใส่ URL ล่าสุดของคุณที่นี่
const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbwGxHMcdYbDTMBEfFOuAjYk0FjKV5zjG0AMc0NkvXTFZVRhAlyorFGnpGAIEeez7ohMGQ/exec';

// --- State ---
const transactions = ref([]); // เก็บข้อมูลรายการทั้งหมด
const isLoading = ref(false);
const isSaving = ref(false);

const form = ref({
  date: new Date().toISOString().split('T')[0],
  type: 'รายจ่าย',
  category: '',
  name: '',
  amount: '',
  note: ''
});

// --- Actions ---

// 1. ฟังก์ชันดึงข้อมูล (Fetch Data)
const fetchData = async () => {
  isLoading.value = true;
  try {
    const res = await fetch(GOOGLE_SCRIPT_URL);
    const data = await res.json();
    transactions.value = data;
  } catch (error) {
    console.error("ดึงข้อมูลไม่สำเร็จ:", error);
  } finally {
    isLoading.value = false;
  }
};

// 2. ฟังก์ชันบันทึกข้อมูล
const submitForm = async () => {
  isSaving.value = true;
  try {
    await fetch(GOOGLE_SCRIPT_URL, {
      method: 'POST',
      mode: 'no-cors',
      headers: { 'Content-Type': 'text/plain' },
      body: JSON.stringify(form.value)
    });
    
    // บันทึกเสร็จ ให้ดึงข้อมูลใหม่ทันที
    await fetchData(); 
    
    // เคลียร์ค่า
    form.value.name = '';
    form.value.amount = '';
    
  } catch (error) {
    alert("บันทึกผิดพลาด");
  } finally {
    isSaving.value = false;
  }
};

// --- Computed (คำนวณยอดอัตโนมัติ) ---

// กรองรายการตามเดือนปัจจุบัน (Optional: เดี๋ยวทำตัวเลือก Filter ทีหลัง ตอนนี้เอาเดือนปัจจุบันก่อน)
const currentMonthTransactions = computed(() => {
  return transactions.value; // ตอนนี้เอาทั้งหมดก่อน
});

const totalIncome = computed(() => 
  transactions.value
    .filter(t => t.type === 'รายรับ' || t.type === 'เงินเดือน' || t.type === 'โบนัส')
    .reduce((sum, t) => sum + Number(t.amount), 0)
);

const totalExpense = computed(() => 
  transactions.value
    .filter(t => t.type === 'รายจ่าย')
    .reduce((sum, t) => sum + Number(t.amount), 0)
);

const totalSavings = computed(() => 
  transactions.value
    .filter(t => t.type === 'เงินออม')
    .reduce((sum, t) => sum + Number(t.amount), 0)
);

const balance = computed(() => (totalIncome.value - totalExpense.value - totalSavings.value));

// เรียกใช้เมื่อเปิดเว็บ
onMounted(() => {
  fetchData();
});

// Helper แปลงวันที่ให้อ่านง่าย
const formatDate = (dateString) => {
  const d = new Date(dateString);
  return d.toLocaleDateString('th-TH', { day: 'numeric', month: 'short', year: '2-digit' });
};
</script>

<template>
  <div class="min-h-screen bg-gray-100 p-4 md:p-8">
    <div class="max-w-4xl mx-auto grid grid-cols-1 md:grid-cols-2 gap-8">
      
      <div class="bg-white rounded-2xl shadow-sm p-6 h-fit">
        <h2 class="text-xl font-bold text-gray-800 mb-4">📝 บันทึกรายการใหม่</h2>
        <form @submit.prevent="submitForm" class="space-y-4">
          <div>
            <label class="text-xs text-gray-500">วันที่</label>
            <input type="date" v-model="form.date" required class="w-full p-2 border rounded-lg bg-gray-50">
          </div>
          
          <div>
            <label class="text-xs text-gray-500">ประเภท</label>
            <div class="grid grid-cols-3 gap-2">
              <button type="button" @click="form.type='รายจ่าย'" :class="form.type==='รายจ่าย'?'bg-red-500 text-white':'bg-gray-100'" class="p-2 rounded-lg text-sm font-medium transition-colors">รายจ่าย</button>
              <button type="button" @click="form.type='รายรับ'" :class="form.type==='รายรับ'?'bg-green-500 text-white':'bg-gray-100'" class="p-2 rounded-lg text-sm font-medium transition-colors">รายรับ</button>
              <button type="button" @click="form.type='เงินออม'" :class="form.type==='เงินออม'?'bg-blue-500 text-white':'bg-gray-100'" class="p-2 rounded-lg text-sm font-medium transition-colors">เงินออม</button>
            </div>
          </div>

          <div>
            <label class="text-xs text-gray-500">รายละเอียด</label>
            <div class="flex gap-2">
              <input type="text" v-model="form.category" placeholder="หมวดหมู่ (เช่น อาหาร)" required class="w-1/2 p-2 border rounded-lg">
              <input type="text" v-model="form.name" placeholder="ชื่อรายการ" class="w-1/2 p-2 border rounded-lg">
            </div>
          </div>

          <div>
            <label class="text-xs text-gray-500">จำนวนเงิน</label>
            <input type="number" v-model="form.amount" placeholder="0.00" required class="w-full p-3 border rounded-lg text-xl font-bold text-right text-blue-600">
          </div>

          <button type="submit" :disabled="isSaving" class="w-full bg-black text-white py-3 rounded-xl font-bold hover:bg-gray-800 transition-all disabled:bg-gray-400">
            {{ isSaving ? '⏳ กำลังบันทึก...' : 'บันทึกรายการ' }}
          </button>
        </form>
      </div>

      <div class="space-y-6">
        
        <div class="grid grid-cols-2 gap-4">
          <div class="bg-green-100 p-4 rounded-2xl">
            <p class="text-xs text-green-600 font-bold uppercase">รายรับรวม</p>
            <p class="text-2xl font-bold text-green-800">+{{ totalIncome.toLocaleString() }}</p>
          </div>
          <div class="bg-red-100 p-4 rounded-2xl">
            <p class="text-xs text-red-600 font-bold uppercase">รายจ่ายรวม</p>
            <p class="text-2xl font-bold text-red-800">-{{ totalExpense.toLocaleString() }}</p>
          </div>
          <div class="bg-blue-100 p-4 rounded-2xl">
            <p class="text-xs text-blue-600 font-bold uppercase">เงินออม</p>
            <p class="text-2xl font-bold text-blue-800">{{ totalSavings.toLocaleString() }}</p>
          </div>
          <div class="bg-white border border-gray-200 p-4 rounded-2xl">
            <p class="text-xs text-gray-500 font-bold uppercase">คงเหลือสุทธิ</p>
            <p class="text-2xl font-bold" :class="balance >= 0 ? 'text-gray-800' : 'text-red-600'">
              {{ balance.toLocaleString() }}
            </p>
          </div>
        </div>

        <div class="bg-white rounded-2xl shadow-sm p-6 overflow-hidden">
          <div class="flex justify-between items-center mb-4">
            <h3 class="font-bold text-gray-800">รายการล่าสุด</h3>
            <button @click="fetchData" class="text-xs text-blue-500 hover:underline">รีเฟรช</button>
          </div>
          
          <div v-if="isLoading" class="text-center py-10 text-gray-400">กำลังโหลดข้อมูล...</div>
          
          <div v-else class="space-y-3 max-h-[400px] overflow-y-auto pr-2">
            <div v-for="(item, index) in transactions" :key="index" class="flex justify-between items-center p-3 hover:bg-gray-50 rounded-lg border-b border-gray-100 last:border-0">
              <div class="flex items-center gap-3">
                <div class="w-10 h-10 rounded-full flex items-center justify-center text-lg"
                  :class="{
                    'bg-red-100': item.type === 'รายจ่าย',
                    'bg-green-100': item.type === 'รายรับ',
                    'bg-blue-100': item.type === 'เงินออม'
                  }">
                  {{ item.type === 'รายจ่าย' ? '💸' : (item.type === 'รายรับ' ? '💰' : '🐷') }}
                </div>
                <div>
                  <p class="font-bold text-sm text-gray-800">{{ item.name || item.category }}</p>
                  <p class="text-xs text-gray-400">{{ formatDate(item.date) }} • {{ item.category }}</p>
                </div>
              </div>
              <span class="font-bold" 
                :class="{
                  'text-red-500': item.type === 'รายจ่าย',
                  'text-green-500': item.type === 'รายรับ',
                  'text-blue-500': item.type === 'เงินออม'
                }">
                {{ item.type === 'รายจ่าย' ? '-' : '+' }}{{ Number(item.amount).toLocaleString() }}
              </span>
            </div>
            
            <div v-if="transactions.length === 0" class="text-center text-gray-400 py-4">
              ยังไม่มีรายการ
            </div>
          </div>
        </div>

      </div>
    </div>
  </div>
</template>