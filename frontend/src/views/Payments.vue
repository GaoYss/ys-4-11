<template>
  <div class="page-stack">
    <section class="panel">
      <div class="panel-head">
        <h2>待缴账单</h2>
        <button @click="load">刷新</button>
      </div>
      <DataTable :columns="billColumns" :rows="unpaidBills">
        <template #cell-status="{ row }"><StatusBadge :status="row.status" /></template>
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
        <template #actions="{ row }">
          <button @click="openPayDialog(row)">缴费</button>
        </template>
      </DataTable>
    </section>

    <section class="panel">
      <div class="panel-head">
        <h2>缴费记录</h2>
      </div>
      <DataTable :columns="paymentColumns" :rows="payments">
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
        <template #cell-method="{ row }">{{ row.method_display || row.method }}</template>
        <template #actions="{ row }">
          <button @click="viewReceipt(row)">查看票据</button>
        </template>
      </DataTable>
    </section>

    <div v-if="showPayDialog" class="modal-mask" @click.self="showPayDialog = false">
      <div class="modal">
        <header>
          <h3>确认缴费</h3>
          <button class="close" @click="showPayDialog = false">&times;</button>
        </header>
        <div class="modal-body form-grid">
          <label>账单编号
            <input :value="currentBill?.bill_no" disabled />
          </label>
          <label>房屋
            <input :value="currentBill?.room_label" disabled />
          </label>
          <label>账期
            <input :value="currentBill?.period" disabled />
          </label>
          <label>费用类型
            <input :value="currentBill?.fee_name" disabled />
          </label>
          <label>金额
            <input :value="'¥' + Number(currentBill?.amount || 0).toFixed(2)" disabled />
          </label>
          <label>支付方式
            <select v-model="payForm.method">
              <option value="wechat">微信</option>
              <option value="alipay">支付宝</option>
              <option value="bank">银行卡</option>
              <option value="cash">现金</option>
            </select>
          </label>
          <label>付款人
            <input v-model="payForm.payer" :placeholder="currentBill?.owner_name" />
          </label>
          <label>收款人
            <input v-model="payForm.payee" placeholder="小区物业服务中心" />
          </label>
        </div>
        <footer>
          <button class="secondary" @click="showPayDialog = false">取消</button>
          <button @click="confirmPay">确认支付</button>
        </footer>
      </div>
    </div>

    <div v-if="showReceiptDialog" class="modal-mask" @click.self="showReceiptDialog = false">
      <div class="modal receipt-modal">
        <header>
          <h3>票据详情</h3>
          <button class="close" @click="showReceiptDialog = false">&times;</button>
        </header>
        <div class="modal-body">
          <article v-if="currentReceipt" class="receipt">
            <header>
              <h2>物业费电子票据</h2>
              <span>{{ currentReceipt.receipt_no }}</span>
            </header>
            <dl>
              <div><dt>账期</dt><dd>{{ currentReceipt.period }}</dd></div>
              <div><dt>费用类型</dt><dd>{{ currentReceipt.fee_name }}</dd></div>
              <div><dt>房屋</dt><dd>{{ currentReceipt.room_label }}</dd></div>
              <div><dt>业主/付款人</dt><dd>{{ currentReceipt.payer || currentReceipt.owner_name }}</dd></div>
              <div><dt>支付方式</dt><dd>{{ currentReceipt.method_display || currentReceipt.method }}</dd></div>
              <div><dt>收款人</dt><dd>{{ currentReceipt.payee }}</dd></div>
              <div><dt>金额</dt><dd class="amount">¥{{ Number(currentReceipt.amount).toFixed(2) }}</dd></div>
              <div><dt>支付时间</dt><dd>{{ currentReceipt.paid_at }}</dd></div>
            </dl>
            <footer>
              <span>收款单位：{{ currentReceipt.payee }}</span>
            </footer>
          </article>
        </div>
        <footer>
          <button @click="printReceipt">打印票据</button>
          <button class="secondary" @click="showReceiptDialog = false">关闭</button>
        </footer>
      </div>
    </div>
  </div>
</template>

<script setup>
import { computed, onMounted, reactive, ref } from "vue";
import { propertyApi } from "../api/property";
import DataTable from "../components/DataTable.vue";
import StatusBadge from "../components/StatusBadge.vue";

const bills = ref([]);
const payments = ref([]);
const unpaidBills = computed(() => bills.value.filter((bill) => ["unpaid", "overdue"].includes(bill.status)));

const showPayDialog = ref(false);
const showReceiptDialog = ref(false);
const currentBill = ref(null);
const currentReceipt = ref(null);

const payForm = reactive({ method: "wechat", payer: "", payee: "小区物业服务中心" });

const billColumns = [
  { key: "bill_no", label: "账单编号" },
  { key: "room_label", label: "房屋" },
  { key: "owner_name", label: "业主" },
  { key: "fee_name", label: "费用类型" },
  { key: "period", label: "账期" },
  { key: "amount", label: "金额" },
  { key: "due_date", label: "截止日期" },
  { key: "status", label: "状态" }
];
const paymentColumns = [
  { key: "receipt_no", label: "票据编号" },
  { key: "bill_no", label: "账单编号" },
  { key: "room_label", label: "房屋" },
  { key: "period", label: "账期" },
  { key: "fee_name", label: "费用类型" },
  { key: "owner_name", label: "付款人" },
  { key: "payee", label: "收款人" },
  { key: "amount", label: "金额" },
  { key: "method", label: "支付方式" },
  { key: "paid_at", label: "支付时间" }
];

async function load() {
  [bills.value, payments.value] = await Promise.all([propertyApi.listBills(), propertyApi.listPayments()]);
}

function openPayDialog(row) {
  currentBill.value = row;
  payForm.method = "wechat";
  payForm.payer = row.owner_name || "";
  payForm.payee = "小区物业服务中心";
  showPayDialog.value = true;
}

async function confirmPay() {
  if (!currentBill.value) return;
  await propertyApi.payBill(currentBill.value.id, {
    method: payForm.method,
    payer: payForm.payer || currentBill.value.owner_name,
    payee: payForm.payee || "小区物业服务中心"
  });
  showPayDialog.value = false;
  await load();
}

function viewReceipt(row) {
  currentReceipt.value = row;
  showReceiptDialog.value = true;
}

function printReceipt() {
  window.print();
}

onMounted(load);
</script>
