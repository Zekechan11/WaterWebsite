<script setup>
import axios from "axios"; // Make sure to import axios
import { ref, onMounted, computed } from "vue";
import { WATER_API } from "../../config";
import { formatCurrency } from "../../service/formatcurrency";

const orders = ref([]);
const visible = ref(false);
const dashData = ref([]);

const userData = JSON.parse(localStorage.getItem("user_data"));

const fetchOrders = async () => {
  try {
    const response = await axios.get(`${WATER_API}/api/get_order?area_id=${userData.area_id}&status=Pending`);
    orders.value = response.data;
  } catch (error) {
    console.error('Error fetching orders:', error);
  }
};

const fetchDashData = async () => {
  try {
    const response = await axios.get(`${WATER_API}/v2/api/agent/dashboard/${userData.area_id}`);
    dashData.value = response.data;
  } catch (error) {
    console.error('Error fetching orders:', error);
  }
};

onMounted(() => {
  fetchDashData();
  fetchOrders();
});

const orderCount = computed(() => orders.value.length);

</script>

<template>
  <div class="space">
    <h1 class="text-center font-semibold text-gray-500 mb-6 text-4xl">
      {{ userData.area }}
    </h1>
  </div>
  <div class="grid grid-cols-12 gap-8">
    <div class="col-span-12 lg:col-span-6 xl:col-span-4">
      <div class="card mb-0 shadow-md">
        <div class="flex justify-between mb-4">
          <div>
            <span class="block text-muted-color font-medium mb-4">FGS </span>
            <div class="text-surface-900 dark:text-surface-0 font-medium text-xl">
              {{ dashData.fgs || 0 }}
            </div>
          </div>
          <div class="flex items-center justify-center bg-orange-100 dark:bg-orange-400/10 rounded-border"
            style="width: 5rem; height: 5rem">
            <i class="pi pi-users text-orange-500 !text-4xl"></i>
          </div>
        </div>
      </div>
    </div>
    <div class="col-span-12 lg:col-span-6 xl:col-span-4">
      <div class="card mb-0 shadow-md">
        <div class="flex justify-between mb-4">
          <div>
            <span class="block text-muted-color font-medium mb-4">GALLONS DELIVERED</span>
            <div class="text-surface-900 dark:text-surface-0 font-medium text-xl">
              {{ dashData.gallons_delivered || 0 }}
            </div>
          </div>
          <div class="flex items-center justify-center bg-cyan-100 dark:bg-cyan-400/10 rounded-border"
            style="width: 5rem; height: 5rem">
            <i class="pi pi-users text-cyan-500 !text-4xl"></i>
          </div>
        </div>
      </div>
    </div>
    <div class="col-span-12 lg:col-span-6 xl:col-span-4">
      <div class="card mb-0 shadow-md">
        <div class="flex justify-between mb-4">
          <div>
            <span class="block text-muted-color font-medium mb-4">GALLONS RETURNED</span>
            <div class="text-surface-900 dark:text-surface-0 font-medium text-xl">
              {{ dashData.gallons_returned || 0 }}
            </div>
          </div>
          <div class="flex items-center justify-center bg-purple-100 dark:bg-purple-400/10 rounded-border"
            style="width: 5rem; height: 5rem">
            <i class="pi pi-paypal text-purple-500 !text-4xl"></i>
          </div>
        </div>
      </div>
    </div>
    <div class="col-span-12 lg:col-span-6 xl:col-span-4">
      <div class="card mb-0 shadow-md">
        <div class="flex justify-between mb-4">
          <div>
            <span class="block text-muted-color font-medium mb-4">AMOUNT COLLECTED</span>
            <div class="text-surface-900 dark:text-surface-0 font-medium text-xl">
              {{ formatCurrency(dashData.collected_ammount || 0) }}
            </div>
          </div>
          <div class="flex items-center justify-center bg-purple-100 dark:bg-purple-400/10 rounded-border"
            style="width: 5rem; height: 5rem">
            <i class="pi pi-paypal text-purple-500 !text-4xl"></i>
          </div>
        </div>
      </div>
    </div>
    <div class="col-span-12 lg:col-span-6 xl:col-span-4 cursor-pointer" @click="visible = true">
      <div class="card mb-0 shadow-md">
        <div class="flex justify-between mb-4">
          <div>
            <span class="block text-muted-color font-medium mb-4">CUSTOMER'S ORDER</span>
            <div class="text-surface-900 dark:text-surface-0 font-medium text-xl">
              {{ orderCount }}
            </div>
          </div>
          <div class="flex items-center justify-center bg-purple-100 dark:bg-purple-400/10 rounded-border"
            style="width: 5rem; height: 5rem">
            <i class="pi pi-users text-purple-500 !text-4xl"></i>
          </div>
        </div>
      </div>
    </div>

    <Dialog v-model:visible="visible" modal header="Customer's Order" :style="{ width: '50rem' }"
      :breakpoints="{ '1199px': '75vw', '575px': '90vw' }">
      <DataTable :value="orders" showGridlines tableStyle="min-width: 40rem">
        <Column field="customer_fullname" header="Fullname"></Column>
        <Column field="num_gallons_order" header="Quantity"></Column>
        <Column field="total_containers_on_loan" header="COL"></Column>
        <Column field="payable_amount" header="Payables">
          <template #body="slotProps">
            <span class="text-red-600">{{ formatCurrency(slotProps.data.payable_amount) }}</span>
          </template>
        </Column>
        <Column field="total_price" header="Total Payment">
          <template #body="slotProps">
            <span class="text-green-600">{{ formatCurrency(slotProps.data.total_price) }}</span>
          </template>
        </Column>
        <Column field="date" header="Date"></Column>
       
      </DataTable>
    </Dialog>

  </div>
</template>
