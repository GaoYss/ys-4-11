<template>
  <div class="split-layout">
    <section class="panel">
      <div class="panel-head">
        <h2>票据列表</h2>
        <button @click="load">刷新</button>
      </div>
      <div class="filter-bar">
        <label>票据编号
          <input v-model="filter.receipt_no" placeholder="输入票据编号搜索" @keyup.enter="load" />
        </label>
        <label>账期
          <input v-model="filter.period" placeholder="如 2026-06" @keyup.enter="load" />
        </label>
        <button class="small" @click="load">搜索</button>
        <button class="small secondary" @click="resetFilter">重置</button>
      </div>
      <DataTable :columns="columns" :rows="receipts" @click="noop">
        <template #cell-amount="{ row }">¥{{ Number(row.amount).toFixed(2) }}</template>
        <template #cell-method="{ row }">{{ row.method_display || row.method }}</template>
        <template #actions="{ row }">
          <button @click="selected = row">预览</button>
        </template>
      </DataTable>
    </section>

    <section class="receipt-panel">
      <article v-if="selected" class="receipt">
        <header>
          <h2>物业费电子票据</h2>
          <span class="receipt-no">{{ selected.receipt_no }}</span>
        </header>
        <dl>
          <div><dt>账期</dt><dd>{{ selected.period }}</dd></div>
          <div><dt>费用类型</dt><dd>{{ selected.fee_name }}</dd></div>
          <div><dt>房屋</dt><dd>{{ selected.room_label }}</dd></div>
          <div><dt>业主/付款人</dt><dd>{{ selected.payer || selected.owner_name }}</dd></div>
          <div><dt>支付方式</dt><dd>{{ selected.method_display || selected.method }}</dd></div>
          <div><dt>收款人</dt><dd>{{ selected.payee }}</dd></div>
          <div class="amount-row"><dt>票据金额</dt><dd class="amount">¥{{ Number(selected.amount).toFixed(2) }}</dd></div>
          <div><dt>支付时间</dt><dd>{{ selected.paid_at }}</dd></div>
          <div><dt>关联账单</dt><dd>{{ selected.bill_no }}</dd></div>
        </dl>
        <footer>
          <span>收款单位：{{ selected.payee }}</span>
          <button @click="printReceipt">打印票据</button>
        </footer>
      </article>
      <div v-else class="placeholder">选择一条缴费记录预览票据</div>
    </section>
  </div>
</template>

<script setup>
import { onMounted, reactive, ref } from "vue";
import { propertyApi } from "../api/property";
import DataTable from "../components/DataTable.vue";

const receipts = ref([]);
const selected = ref(null);
const filter = reactive({ receipt_no: "", period: "" });

const columns = [
  { key: "receipt_no", label: "票据编号" },
  { key: "room_label", label: "房屋" },
  { key: "period", label: "账期" },
  { key: "fee_name", label: "费用类型" },
  { key: "payee", label: "收款人" },
  { key: "amount", label: "金额" },
  { key: "method", label: "支付方式" },
  { key: "paid_at", label: "支付时间" }
];

async function load() {
  const params = {};
  if (filter.receipt_no) params.receipt_no = filter.receipt_no;
  if (filter.period) params.period = filter.period;
  receipts.value = await propertyApi.listReceipts(params);
  selected.value = selected.value || receipts.value[0] || null;
}

function resetFilter() {
  filter.receipt_no = "";
  filter.period = "";
  load();
}

function printReceipt() {
  window.print();
}

function noop() {}

onMounted(load);
</script>
