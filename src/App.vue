<script setup>
import { ref, computed, onMounted } from 'vue';
import { Chart as ChartJS, ArcElement, Tooltip, Legend, BarElement, CategoryScale, LinearScale, Title } from 'chart.js';
import { Pie, Bar } from 'vue-chartjs';

ChartJS.register(ArcElement, Tooltip, Legend, BarElement, CategoryScale, LinearScale, Title);

// ⚠️ ใส่ URL ของคุณที่นี่
const GOOGLE_SCRIPT_URL = 'https://script.google.com/macros/s/AKfycbyM6gzH-tWCAm0FQ-upNTyb1siDnQnMiJ_WL5tpegSkyTIhwi55bZzoNL-m2aE1XPIXAA/exec';

// --- State ---
const currentPage = ref('home');
const isLoading = ref(false);
const isSaving = ref(false);
const transactions = ref([]);
const categories = ref([]);

const summaryYear = ref('all');
const summaryMonth = ref('all');
const months = ["มกราคม", "กุมภาพันธ์", "มีนาคม", "เมษายน", "พฤษภาคม", "มิถุนายน", "กรกฎาคม", "สิงหาคม", "กันยายน", "ตุลาคม", "พฤศจิกายน", "ธันวาคม"];

const currentDate = new Date();
const homeMonth = ref(currentDate.getMonth());
const homeYear = ref(currentDate.getFullYear());

// --- Forms ---
const form = ref({
  date: new Date().toISOString().split('T')[0],
  type: 'รายจ่าย',
  category: '',
  name: '',
  amount: '',
  note: '', // ✅ เพิ่มตัวแปร note ตรงนี้
  action: 'saveTransaction'
});

const catForm = ref({
  type: 'รายจ่าย',
  name: '',
  action: 'saveCategory'
});

// --- API Actions ---
const fetchAllData = async () => {
  isLoading.value = true;
  try {
    const res = await fetch(GOOGLE_SCRIPT_URL);
    const data = await res.json();
    transactions.value = data.transactions || [];
    categories.value = data.categories || [];
    updateCategoryDefault();
  } catch (error) {
    console.error("Error:", error);
  } finally {
    isLoading.value = false;
  }
};

const submitTransaction = async () => {
  if(!form.value.category) return alert("กรุณาเลือกหมวดหมู่");
  isSaving.value = true;
  try {
    await fetch(GOOGLE_SCRIPT_URL, { method: 'POST', mode: 'no-cors', headers: { 'Content-Type': 'text/plain' }, body: JSON.stringify(form.value) });
    await fetchAllData();
    // ✅ เคลียร์ค่า note ด้วย
    form.value.name = ''; 
    form.value.amount = ''; 
    form.value.note = ''; 
    alert("บันทึกสำเร็จ");
  } catch (error) { alert("บันทึกผิดพลาด"); } finally { isSaving.value = false; }
};

const submitCategory = async () => {
  if(!catForm.value.name) return;
  isSaving.value = true;
  try {
    await fetch(GOOGLE_SCRIPT_URL, { method: 'POST', mode: 'no-cors', headers: { 'Content-Type': 'text/plain' }, body: JSON.stringify(catForm.value) });
    await fetchAllData();
    catForm.value.name = '';
    alert("เพิ่มหมวดหมู่เรียบร้อย");
  } catch (error) { alert("เกิดข้อผิดพลาด"); } finally { isSaving.value = false; }
};

// --- Computed Logic ---
const homeTransactions = computed(() => {
  return transactions.value.filter(t => {
    const d = new Date(t.date);
    return d.getMonth() === homeMonth.value && d.getFullYear() === homeYear.value;
  });
});
const availableCategories = computed(() => categories.value.filter(c => c.type === form.value.type));
const updateCategoryDefault = () => {
  const available = categories.value.filter(c => c.type === form.value.type);
  form.value.category = available.length > 0 ? available[0].name : '';
};
const homeSummary = computed(() => {
  return {
    income: homeTransactions.value.filter(t => t.type === 'รายรับ').reduce((s, t) => s + Number(t.amount), 0),
    expense: homeTransactions.value.filter(t => t.type === 'รายจ่าย').reduce((s, t) => s + Number(t.amount), 0),
    savings: homeTransactions.value.filter(t => t.type === 'เงินออม').reduce((s, t) => s + Number(t.amount), 0)
  }
});

const dashboardTransactions = computed(() => {
  return transactions.value.filter(t => {
    const d = new Date(t.date);
    const matchYear = summaryYear.value === 'all' || d.getFullYear() === summaryYear.value;
    const matchMonth = summaryMonth.value === 'all' || d.getMonth() === summaryMonth.value;
    return matchYear && matchMonth;
  });
});

const dashSummary = computed(() => {
  return {
    income: dashboardTransactions.value.filter(t => t.type === 'รายรับ').reduce((s, t) => s + Number(t.amount), 0),
    expense: dashboardTransactions.value.filter(t => t.type === 'รายจ่าย').reduce((s, t) => s + Number(t.amount), 0),
    savings: dashboardTransactions.value.filter(t => t.type === 'เงินออม').reduce((s, t) => s + Number(t.amount), 0)
  }
});

const mission2026Data = computed(() => {
  const missionList = transactions.value.filter(t => t.type === 'เงินออม' && t.category === 'Mission 2026');
  return missionList.filter(t => {
    const d = new Date(t.date);
    const matchYear = summaryYear.value === 'all' || d.getFullYear() === summaryYear.value;
    const matchMonth = summaryMonth.value === 'all' || d.getMonth() === summaryMonth.value;
    return matchYear && matchMonth;
  });
});
const missionTotal = computed(() => mission2026Data.value.reduce((s, t) => s + Number(t.amount), 0));

const getChartDataByType = (type) => {
  const filtered = dashboardTransactions.value.filter(t => t.type === type);
  const grouped = filtered.reduce((acc, curr) => {
    acc[curr.category] = (acc[curr.category] || 0) + Number(curr.amount);
    return acc;
  }, {});
  return {
    labels: Object.keys(grouped),
    datasets: [{ backgroundColor: ['#41B883', '#E46651', '#00D8FF', '#DD1B16', '#FFC0CB', '#FFD700', '#8A2BE2'], data: Object.values(grouped) }]
  };
};

const barData = computed(() => ({
  labels: ['รายรับ', 'รายจ่าย', 'เงินออม'],
  datasets: [{ label: 'จำนวนเงิน (บาท)', backgroundColor: ['#22c55e', '#ef4444', '#3b82f6'], data: [dashSummary.value.income, dashSummary.value.expense, dashSummary.value.savings] }]
}));
const chartOptions = { responsive: true, maintainAspectRatio: false };

const formatDate = (d) => new Date(d).toLocaleDateString('th-TH', { day: 'numeric', month: 'short', year: '2-digit' });
const formatMoney = (n) => Number(n).toLocaleString();

onMounted(() => { fetchAllData(); });
</script>

<template>
  <div class="min-h-screen bg-gray-50 font-sans pb-10">
    <div class="max-w-2xl mx-auto p-4">
      
      <nav class="flex justify-between items-center mb-6 bg-white p-4 rounded-2xl shadow-sm sticky top-2 z-50">
        <h1 class="text-xl font-black text-gray-800 tracking-tighter">🚀 MONEY V3</h1>
        <div class="flex gap-2">
           <button @click="currentPage='home'" :class="currentPage==='home'?'bg-black text-white':'bg-gray-100 text-gray-600'" class="px-3 py-2 rounded-lg text-xs font-bold transition-all">หน้าหลัก</button>
           <button @click="currentPage='dashboard'" :class="currentPage==='dashboard'?'bg-black text-white':'bg-gray-100 text-gray-600'" class="px-3 py-2 rounded-lg text-xs font-bold transition-all">สรุปยอด</button>
           <button @click="currentPage='categories'" :class="currentPage==='categories'?'bg-black text-white':'bg-gray-100 text-gray-600'" class="px-3 py-2 rounded-lg text-xs font-bold transition-all">ตั้งค่า</button>
        </div>
      </nav>

      <div v-if="currentPage === 'home'" class="space-y-6">
        <div class="flex gap-2 justify-end">
           <select v-model="homeMonth" class="bg-white border rounded-lg px-2 py-1 text-sm font-bold"><option v-for="(m, i) in months" :key="i" :value="i">{{ m }}</option></select>
           <select v-model="homeYear" class="bg-white border rounded-lg px-2 py-1 text-sm font-bold"><option v-for="y in 5" :key="y" :value="2024 + y">{{ 2024 + y }}</option></select>
        </div>

        <div class="grid grid-cols-3 gap-2">
          <div class="bg-green-500 text-white p-4 rounded-2xl text-center shadow-lg shadow-green-200"><p class="text-xs opacity-80">รายรับ</p><p class="text-lg font-bold">{{ formatMoney(homeSummary.income) }}</p></div>
          <div class="bg-red-500 text-white p-4 rounded-2xl text-center shadow-lg shadow-red-200"><p class="text-xs opacity-80">รายจ่าย</p><p class="text-lg font-bold">{{ formatMoney(homeSummary.expense) }}</p></div>
          <div class="bg-blue-500 text-white p-4 rounded-2xl text-center shadow-lg shadow-blue-200"><p class="text-xs opacity-80">เงินออม</p><p class="text-lg font-bold">{{ formatMoney(homeSummary.savings) }}</p></div>
        </div>

        <div class="bg-white rounded-2xl shadow-sm p-5 border border-gray-100">
           <div class="flex gap-2 mb-4 bg-gray-100 p-1 rounded-xl">
              <button v-for="t in ['รายจ่าย','รายรับ','เงินออม']" :key="t" @click="form.type=t;updateCategoryDefault()" :class="form.type===t?'bg-white shadow text-black':'text-gray-400'" class="flex-1 py-2 rounded-lg text-sm font-bold transition-all">{{ t }}</button>
           </div>
           <form @submit.prevent="submitTransaction" class="space-y-3">
              <div class="flex gap-2">
                 <input type="date" v-model="form.date" required class="w-1/3 p-2 bg-gray-50 border rounded-xl text-sm font-bold text-gray-600">
                 <select v-model="form.category" required class="w-2/3 p-2 bg-white border rounded-xl text-sm font-bold appearance-none"><option value="" disabled>เลือกหมวดหมู่...</option><option v-for="(c,i) in availableCategories" :key="i" :value="c.name">{{ c.name }}</option></select>
              </div>
              <input type="number" v-model="form.amount" placeholder="0.00" required inputmode="decimal" class="w-full p-4 text-3xl font-black text-right text-gray-800 placeholder-gray-200 border-b-2 border-gray-100 outline-none focus:border-black transition-all">
              
              <input type="text" v-model="form.name" placeholder="ชื่อรายการ (เช่น ข้าวมันไก่)" class="w-full text-right text-sm text-gray-500 outline-none">
              <input type="text" v-model="form.note" placeholder="📝 หมายเหตุ (Optional)" class="w-full text-right text-xs text-gray-400 outline-none border-t pt-3 mb-6">

              <button type="submit" :disabled="isSaving" class="w-full bg-black text-white py-3 rounded-xl font-bold active:scale-95 transition-transform">{{ isSaving ? '...' : 'บันทึกรายการ' }}</button>
           </form>
        </div>

        <div class="space-y-3">
           <h3 class="font-bold text-gray-400 text-xs uppercase ml-2">รายการล่าสุด</h3>
           <div v-for="(item, i) in homeTransactions" :key="i" class="bg-white p-3 rounded-xl flex justify-between items-center border border-gray-100 shadow-sm">
              <div class="flex items-center gap-3">
                 <div class="w-10 h-10 rounded-full flex items-center justify-center text-lg" :class="{'bg-red-50':item.type==='รายจ่าย','bg-green-50':item.type==='รายรับ','bg-blue-50':item.type==='เงินออม'}">{{ item.type==='รายจ่าย'?'💸':(item.type==='รายรับ'?'💰':'🐷') }}</div>
                 <div>
                   <p class="font-bold text-gray-800 text-sm">{{ item.category }}</p>
                   <p class="text-xs text-gray-400">
                     {{ formatDate(item.date) }} 
                     <span v-if="item.name">- {{ item.name }}</span>
                   </p>
                   <p v-if="item.note" class="text-xs text-gray-500 mt-1 bg-gray-50 px-2 py-0.5 rounded inline-block">📝 {{ item.note }}</p>
                 </div>
              </div>
              <span class="font-bold" :class="{'text-red-500':item.type==='รายจ่าย','text-green-500':item.type==='รายรับ','text-blue-500':item.type==='เงินออม'}">{{ item.type==='รายจ่าย'?'-':'+' }}{{ formatMoney(item.amount) }}</span>
           </div>
        </div>
      </div>

      <div v-if="currentPage === 'dashboard'" class="space-y-6 animate-fade-in">
        <div class="bg-white p-4 rounded-2xl shadow-sm border border-gray-100 flex flex-col md:flex-row gap-4 items-center justify-between">
           <h2 class="font-bold text-gray-700">📊 ตัวกรองข้อมูล</h2>
           <div class="flex gap-2">
              <select v-model="summaryMonth" class="bg-gray-100 rounded-lg px-3 py-2 text-sm font-bold outline-none"><option value="all">ทุกเดือน</option><option v-for="(m, i) in months" :key="i" :value="i">{{ m }}</option></select>
              <select v-model="summaryYear" class="bg-gray-100 rounded-lg px-3 py-2 text-sm font-bold outline-none"><option value="all">ทุกปี</option><option v-for="y in 5" :key="y" :value="2024 + y">{{ 2024 + y }}</option></select>
           </div>
        </div>

        <div class="bg-white p-5 rounded-2xl shadow-sm border border-gray-100">
           <h3 class="font-bold text-gray-800 mb-4 text-center">ภาพรวม การเงิน</h3>
           <div class="h-64"><Bar :data="barData" :options="chartOptions" /></div>
           <div class="grid grid-cols-3 gap-2 mt-4 text-center">
              <div><p class="text-xs text-gray-400">สุทธิ</p><p class="font-bold text-xl" :class="(dashSummary.income - dashSummary.expense - dashSummary.savings) >= 0 ? 'text-green-600' : 'text-red-600'">{{ formatMoney(dashSummary.income - dashSummary.expense - dashSummary.savings) }}</p></div>
              <div><p class="text-xs text-gray-400">เงินเข้า</p><p class="font-bold text-green-600">+{{ formatMoney(dashSummary.income) }}</p></div>
              <div><p class="text-xs text-gray-400">เงินออก</p><p class="font-bold text-red-600">-{{ formatMoney(dashSummary.expense) }}</p></div>
           </div>
        </div>

        <div class="bg-gradient-to-r from-indigo-500 via-purple-500 to-pink-500 rounded-2xl p-6 text-white shadow-xl relative overflow-hidden">
           <div class="relative z-10">
              <div class="flex justify-between items-start">
                 <div><h3 class="font-black text-2xl italic tracking-wider">🚀 MISSION 2026</h3><p class="text-white/80 text-xs">เป้าหมายเงินออมแห่งอนาคต</p></div>
                 <div class="text-right"><p class="text-xs text-white/80">ยอดสะสมรวม</p><p class="text-3xl font-black">{{ formatMoney(missionTotal) }} <span class="text-sm font-normal">บาท</span></p></div>
              </div>
              <div class="mt-6 bg-white/10 rounded-xl p-4 backdrop-blur-sm max-h-40 overflow-y-auto">
                 <h4 class="text-xs font-bold text-white/70 mb-2 uppercase">ประวัติการเก็บออม</h4>
                 <div v-if="mission2026Data.length === 0" class="text-center text-white/50 text-sm py-2">ยังไม่มีรายการ (อย่าลืมสร้างหมวดหมู่ 'Mission 2026')</div>
                 <div v-else class="space-y-2">
                    <div v-for="(t, i) in mission2026Data" :key="i" class="flex justify-between items-center text-sm border-b border-white/10 pb-1 last:border-0">
                       <div class="flex flex-col">
                         <span class="font-medium text-white/90">{{ formatDate(t.date) }}</span>
                         <span v-if="t.note" class="text-[10px] text-white/60">📝 {{ t.note }}</span>
                       </div>
                       <span class="font-bold text-white">+{{ formatMoney(t.amount) }}</span>
                    </div>
                 </div>
              </div>
           </div>
           <div class="absolute -bottom-10 -right-10 w-40 h-40 bg-white/20 rounded-full blur-2xl"></div>
        </div>

        <div class="grid md:grid-cols-2 gap-4">
           <div class="bg-white p-4 rounded-2xl shadow-sm border border-gray-100"><h3 class="font-bold text-gray-700 mb-2 text-center text-sm">สัดส่วนรายจ่าย</h3><div class="h-48"><Pie :data="getChartDataByType('รายจ่าย')" :options="chartOptions" /></div></div>
           <div class="bg-white p-4 rounded-2xl shadow-sm border border-gray-100"><h3 class="font-bold text-gray-700 mb-2 text-center text-sm">ที่มาของรายได้</h3><div class="h-48"><Pie :data="getChartDataByType('รายรับ')" :options="chartOptions" /></div></div>
           <div class="bg-white p-4 rounded-2xl shadow-sm border border-gray-100 md:col-span-2"><h3 class="font-bold text-gray-700 mb-2 text-center text-sm">พอร์ตเงินออม</h3><div class="h-48"><Pie :data="getChartDataByType('เงินออม')" :options="chartOptions" /></div></div>
        </div>
      </div>

      <div v-if="currentPage === 'categories'" class="bg-white rounded-2xl shadow-sm p-6 space-y-6">
         <h2 class="font-bold text-gray-800">⚙️ จัดการหมวดหมู่</h2>
         <div class="bg-gray-50 p-4 rounded-xl border">
            <h3 class="font-bold text-sm text-gray-600 mb-3">เพิ่มหมวดหมู่ใหม่</h3>
            <form @submit.prevent="submitCategory" class="space-y-3">
               <div class="grid grid-cols-3 gap-2"><button type="button" v-for="t in ['รายจ่าย','รายรับ','เงินออม']" :key="t" @click="catForm.type=t" :class="catForm.type===t?'bg-black text-white':'bg-white border text-gray-400'" class="py-2 rounded-lg text-xs font-bold transition-all">{{ t }}</button></div>
               <input type="text" v-model="catForm.name" placeholder="ชื่อหมวดหมู่" required class="w-full p-2 border rounded-lg text-sm">
               <button type="submit" :disabled="isSaving" class="w-full bg-green-500 text-white py-2 rounded-lg font-bold text-sm hover:bg-green-600">{{ isSaving ? '...' : '+ เพิ่มหมวดหมู่' }}</button>
            </form>
         </div>
         <div>
            <h3 class="font-bold text-sm text-gray-600 mb-3">รายการที่มีอยู่</h3>
            <div class="space-y-2 max-h-60 overflow-y-auto pr-2">
               <div v-for="(c,i) in categories" :key="i" class="flex justify-between p-2 bg-gray-50 rounded-lg text-sm border hover:border-black transition-colors">
                  <span class="font-medium text-gray-700">{{ c.name }}</span>
                  <span class="text-xs px-2 py-1 rounded bg-white border font-bold" :class="{'text-red-500':c.type==='รายจ่าย','text-green-500':c.type==='รายรับ','text-blue-500':c.type==='เงินออม'}">{{ c.type }}</span>
               </div>
            </div>
         </div>
      </div>

    </div>
  </div>
</template>