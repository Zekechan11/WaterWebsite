<script setup>
import { useLayout } from "@/layout/composables/layout";
import axios from "axios";
import QrcodeVue from "qrcode.vue";
import { computed, onMounted, ref, watchEffect } from "vue";
import { WATER_API } from "../../config";
import { attempt } from "../../service/attempt";

const { getPrimary, getSurface, isDarkTheme } = useLayout();

const user_data = JSON.parse(localStorage.getItem("user_data"));

const visible = ref(false);
const qrCodeModal = ref(false);
const ingredient = ref("");
const latestOrder = ref(null);
const userData = ref(null);
const days = ref({});
const pricePerGallon = ref(0.00);
const gallons = ref(0);
const agentName = ref("");
const dashboardData = ref("");
const customers2 = ref([]);

const customerArea = user_data.area;
const fullName = computed(() => {
  return `${user_data.firstname} ${user_data.lastname}`;
});

const fetchLatestOrder = async () => {
  try {
    const response = await axios.get(`${WATER_API}/api/get_order`, {
      params: {
        customer_id: user_data.uid,
        status: "Pending",
      },
    });

    const orderData = Array.isArray(response.data)
      ? response.data[0]
      : response.data;

    if (orderData && orderData.CustomerID === parseInt(user_data.uid)) {
      updateUserData(orderData);
    } else {
      resetOrderData();
    }
  } catch (error) {
    console.error("Error fetching latest order:", error);
    resetOrderData();
  }
};

const updateUserData = (orderData) => {
  latestOrder.value = orderData;
  userData.value = {
    orderId: orderData.ID, // Make sure to map from the ID field
    customerId: orderData.CustomerID,
    customerFirstName: orderData.CustomerFirstName,
    customerLastName: orderData.CustomerLastName,
    gallons: orderData.Num_gallons_order,
    date: orderData.Date,
    dateCreated: orderData.Date_created,
    totalPrice: orderData.Total_price,
    status: orderData.Status,
    customerArea: customerArea,
  };
};

const resetOrderData = () => {
  latestOrder.value = null;
  userData.value = null;
};


const getDashboardData = async () => {

  const [agentResponse, agentError] = await attempt(
    axios.get(`${WATER_API}/v2/api/agent/assigned/${user_data.area_id}`)
  );
  if(agentError) {
    console.error("Field at ", agentError);
  } else {
    agentName.value = agentResponse.data.data;
  }

  const [dayResponse, dayError] = await attempt(
    axios.get(`${WATER_API}/api/get_schedule`)
  );
  if(dayError) {
    console.error("Field at ", dayError);
  } else {
    days.value = dayResponse.data.days;
  }

  const [countResponse, countError] = await attempt(
    axios.get(`${WATER_API}/v2/api/dashboard/${user_data.uid}`)
  );
  if(countError) {
    console.error("Field at ", countError);
  } else {
    dashboardData.value = countResponse.data;
  }

  const [transactionResponse, transactionError] = await attempt(
    axios.get(`${WATER_API}/v2/api/orders/${user_data.uid}`)
  );
  if(transactionError) {
    console.error("Field at ", transactionError);
  } else {
    customers2.value = transactionResponse.data;
  }

  const [priceResponse, priceError] = await attempt(
    axios.get(`${WATER_API}/api/price/${user_data.type}`)
  )
  if(priceError) {
    console.error("Field at ", priceError);
  } else {
    pricePerGallon.value = priceResponse.data;
  }
}

const placeOrder = async () => {
  try {
    const response = await axios.post(`${WATER_API}/api/save_order`, {
      customer_id: user_data.uid,
      num_gallons_order: parseInt(gallons.value),
      date: ingredient.value,
      area_id: parseInt(user_data.area_id),
      type: user_data.type,
    });

    console.log("Order saved:", response.data);

    if (response.data) {
      const newOrderData = {
        ID: response.data.order_id, // Map from the API response
        CustomerID: parseInt(user_data.uid),
        CustomerFirstName: user_data.firstname,
        CustomerLastName: user_data.lastname,
        Num_gallons_order: parseInt(gallons.value),
        Date: ingredient.value,
        Date_created: new Date().toISOString(),
        Total_price: response.data.total_price,
        Status: "Pending",
      };

      updateUserData(newOrderData);
      console.log("QR Code Data:", JSON.stringify(newOrderData));
    }

    visible.value = false;
    qrCodeModal.value = true;
    gallons.value = "";
  } catch (error) {
    console.error("Error saving order:", error);
  }
};

// Watch for changes in the order data
watchEffect(() => {
  if (latestOrder.value && !userData.value) {
    updateUserData(latestOrder.value);
  }
});

onMounted(() => {
  getDashboardData();
  fetchLatestOrder();
});

const qrCodeSize = computed(() => {
  const screenWidth = window.innerWidth;
  if (screenWidth < 768) return 250;
  if (screenWidth < 1024) return 300;
  return 350;
});


function formatCurrency(value) {
  return value.toLocaleString("en-US", { style: "currency", currency: "PHP" });
}

const validateGallons = () => {
  if (gallons.value > 80) {
    gallons.value = 80;
  } else if (gallons.value < 0) {
    gallons.value = 0;
  }
};

const totalPayment = computed(() => {
  return gallons.value * pricePerGallon.value;
});
</script>

<template>
  <div class="flex flex-col space-y-8 p-2 md:flex-row md:space-x-8 md:space-y-2">
    <div class="w-full space-y-8 md:w-2/3">
      <div class="flex items-center justify-between rounded-lg bg-blue-400 p-6 shadow-md">
        <div>
          <h1 class="text-2xl font-semibold">{{ fullName }}!</h1>
          <p class="mt-2 text-gray-800 font-semibold">{{ customerArea }}</p>
          <div class="flex space-x-4">
            <Button label="Order Now" @click="visible = true" />
            <Button v-if="userData && userData.status === 'Pending'" label="View QR Code" severity="secondary"
              @click="qrCodeModal = true" />
          </div>
        </div>
      </div>

      <!-- Order Dialog -->
      <Dialog v-model:visible="visible" modal header="Order Now" :style="{ width: '25rem' }">
        <div
          v-for="day in days"
          :key="day"
          class="flex items-center gap-4 mb-4"
        >
          <div class="flex items-center gap-2">
            <RadioButton
              v-model="ingredient"
              :inputId="'ingredient' + day"
              name="day"
              :value="day.date"
            />
            <label for="ingredient1">{{day.day}} ({{ day.date }})</label>
          </div>
        </div>
        <div class="flex flex-col gap-2 mb-8">
          <div class="flex items-center gap-4">
            <label for="gallons" class="font-semibold w-25">Order Gallons:</label>
            <InputText 
              id="gallons" 
              v-model="gallons" 
              class="flex-auto" 
              autocomplete="off" 
              @input="validateGallons"
              type="number"
              min="0"
              max="80"
            />
          </div>

          <div class="text-sm text-red-500 font-semibold">
            Note: Only 80 gallons are available for order.
          </div>

          <div class="text-sm text-gray-600">
            Price per gallon: ₱{{ pricePerGallon.toFixed(2) }}
          </div>

          <div class="text-lg font-bold text-blue-600">
            Total Payment: ₱{{ totalPayment.toFixed(2) }}
          </div>
        </div>
        <div class="flex justify-end gap-2">
          <Button type="button" label="Cancel" severity="secondary" @click="visible = false"></Button>
          <Button type="button" label="Order" @click="placeOrder"></Button>
        </div>
      </Dialog>


      <!-- QR Code Modal -->
      <Dialog v-model:visible="qrCodeModal" modal header="Your Order QR Code" :style="{ width: '25rem' }">
        <div v-if="userData" class="flex justify-center">
          <qrcode-vue :value="JSON.stringify(userData)" :size="qrCodeSize" level="H" />
        </div>
        <div v-else class="text-center">
          <p class="text-red-500">No QR code available.</p>
        </div>
      </Dialog>

      <!-- DataTable Card for Customers -->
      <div class="card shadow-md">
        <DataTable :value="customers2" scrollable scrollHeight="400px" class="mt-6">
          <Column field="num_gallons_order" header="Purchase Gallons" style="min-width: 200px"></Column>
          <Column field="returned_gallons" header="Returned Gallons" style="min-width: 200px"></Column>
          <Column field="payment" header="Amount Paid" :body="formatCurrency" style="min-width: 200px"></Column>
          <Column field="date" header="Date" style="min-width: 200px" alignFrozen="right"></Column>
        </DataTable>
      </div>
    </div>

    <div class="w-full space-y-8 md:w-1/3">
      <div class="space-y-4">
        <div class="flex items-center justify-between p-4 rounded-lg bg-teal-100 shadow-md">
          <div class="flex items-center space-x-3">
            <div class="bg-teal-300 p-5 rounded-full">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-12 w-12 text-white" viewBox="0 0 24 24" fill="none"
                stroke="currentColor">
                <!-- Head -->
                <circle cx="12" cy="8" r="4" stroke-linecap="round" stroke-linejoin="round" stroke-width="2"></circle>
                <!-- Body -->
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M6 20v-4a4 4 0 014-4h4a4 4 0 014 4v4"></path>
              </svg>
            </div>
            <div class="font-semibold">
              <h2 class="text-2xl font-semibold">{{ agentName || 'Agent is Dead💀' }}</h2>
              <p class="text-sm text-gray-600">Assigned Agent</p>
            </div>
          </div>
        </div>

        <div class="flex items-center justify-between p-4 rounded-lg bg-indigo-100 shadow-md">
          <div class="flex items-center space-x-3">
            <div class="bg-indigo-300 p-5 rounded-full">
              <svg xmlns="http://www.w3.org/2000/svg" class="h-12 w-12 text-white" viewBox="0 0 24 24" fill="none"
                stroke="currentColor">
                <!-- Bottle Body -->
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M9 6c-1 0-2 .8-2 2v10c0 1.2 1 2 2 2h6c1 0 2-.8 2-2V8c0-1.2-1-2-2-2H9z" />
                <!-- Bottle Neck -->
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="2"
                  d="M10 2h4c.6 0 1 .4 1 1v3H9V3c0-.6.4-1 1-1z" />
                <!-- Water Lines -->
                <path stroke-linecap="round" stroke-linejoin="round" stroke-width="1.5" d="M10 11h4M10 14h4M10 17h4" />
              </svg>
            </div>
            <div class="font-semibold">
              <h2 class="text-2xl font-semibold">{{ dashboardData.col || 0 }}</h2>
              <p class="text-sm text-gray-600">Containers on Hold</p>
            </div>
          </div>
        </div>

        <div class="flex items-center justify-between p-4 rounded-lg bg-pink-100 shadow-md">
          <div class="flex items-center space-x-3">
            <div class="bg-pink-300 p-5 rounded-full text-white text-4xl flex items-center justify-center"
              style="height: 75px; width: 75px">
              ₱
            </div>
            <div class="font-semibold">
              <h2 class="text-2xl font-semibold">{{ formatCurrency(dashboardData.loan || 0) }}</h2>
              <p class="text-sm text-gray-600">Total Amount Payable</p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.qr-code-container {
  margin-left: 100px;
  text-align: left;
  max-width: 100%;
}
</style>
