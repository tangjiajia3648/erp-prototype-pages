<template>
  <el-row :gutter="16" class="shared-contract-field-grid">
    <template v-if="section === 'settlement'">
      <template v-if="readonly">
        <el-col :span="24"><el-form-item label="货款结算"><div class="readonly-contract-value settlement-clause">{{ settlementDescription }}</div></el-form-item></el-col>
      </template>
      <template v-else>
        <el-col :span="24"><el-form-item label="货款结算"><div class="editable-settlement-clause">
          <el-select v-model="settlementTrigger" class="clause-trigger"><el-option label="甲方收到货物并验收合格通过" value="甲方收到货物并验收合格通过" /><el-option label="货物全部入库" value="货物全部入库" /><el-option label="双方完成对账" value="双方完成对账" /></el-select>
          <span>后</span><el-input-number v-model="billTime" class="clause-number" :min="0" :max="9999" :controls="false" /><span>个工作日内，甲方通过</span>
          <el-select v-model="settlementPaymentMethod" class="clause-method"><el-option label="转账" value="转账" /><el-option label="银行承兑" value="银行承兑" /><el-option label="其他" value="其他" /></el-select>
          <span>方式支付合同货款</span><el-input-number v-model="settlementPercent" class="clause-percent" :min="0" :max="100" :controls="false" /><span>%；支付金额</span><strong>{{ settlementAmount }}</strong>
        </div></el-form-item></el-col>
      </template>
    </template>
    <template v-else>
      <template v-if="readonly">
        <el-col :span="8"><el-form-item label="签约地点"><div class="readonly-contract-value">{{ displayValue(signingLocation) }}</div></el-form-item></el-col>
        <el-col :span="16"><el-form-item label="产品交付"><div class="readonly-contract-value settlement-clause">{{ deliveryDescription }}</div></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="技术服务"><div class="readonly-contract-value settlement-clause">{{ technicalServiceDescription }}</div></el-form-item></el-col>
        <el-col :span="8"><el-form-item label="收货地址"><div class="readonly-contract-value">{{ displayValue(address) }}</div></el-form-item></el-col>
        <el-col :span="8"><el-form-item label="联系人"><div class="readonly-contract-value">{{ displayValue(contact) }}</div></el-form-item></el-col>
        <el-col :span="8"><el-form-item label="联系电话"><div class="readonly-contract-value">{{ displayValue(phone) }}</div></el-form-item></el-col>
      </template>
      <template v-else>
        <el-col :span="8"><el-form-item label="签约地点"><el-input v-model="signingLocation" /></el-form-item></el-col>
        <el-col :span="16"><el-form-item label="产品交付"><div class="editable-complex-clause"><span>合同签订后</span><el-input-number v-model="deliveryDays" class="clause-number" :min="0" :max="9999" :controls="false" /><span>日内送到甲方指定地点，运输方式</span><el-input v-model="transportMethod" class="clause-text" /></div></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="技术服务"><div class="editable-complex-clause"><span>是否需要提供技术服务与培训</span><el-radio-group v-model="technicalServiceRequired"><el-radio label="需要">需要</el-radio><el-radio label="不需要">不需要</el-radio></el-radio-group></div></el-form-item></el-col>
        <el-col :span="8"><el-form-item label="收货地址"><div class="purchase-address-picker"><el-input v-model="address" /><el-button type="primary" link @click="$emit('select-warehouse')">从仓库选择</el-button></div></el-form-item></el-col>
        <el-col :span="8"><el-form-item label="联系人"><el-input v-model="contact" /></el-form-item></el-col>
        <el-col :span="8"><el-form-item label="联系电话"><el-input v-model="phone" /></el-form-item></el-col>
      </template>
    </template>
  </el-row>
</template>

<script setup>
import { computed } from "vue";
const props = defineProps({ model: { type: Object, required: true }, section: { type: String, default: "delivery" }, readonly: { type: Boolean, default: false }, amount: { type: String, default: "¥0.00" } });
defineEmits(["select-warehouse"]);
const mappedField = (key, fallback = "") => computed({ get: () => props.model[key] ?? fallback, set: (value) => { props.model[key] = value; } });
const signingLocation = mappedField("signingLocation", "成都");
const settlementTrigger = mappedField("settlementTrigger", "甲方收到货物并验收合格通过");
const billTime = mappedField("billTime", 30);
const settlementPaymentMethod = mappedField("settlementPaymentMethod", "转账");
const settlementPercent = mappedField("settlementPercent", 100);
const deliveryDays = mappedField("deliveryDays", 7);
const transportMethod = mappedField("transportMethod", "物流运输");
const technicalServiceRequired = mappedField("technicalServiceRequired", "不需要");
const address = mappedField("address", "成都市高新区天府大道");
const contact = mappedField("contact", "王经理");
const phone = mappedField("phone", "13800000000");
const settlementAmount = computed(() => props.model.settlementAmount || props.amount);
const settlementDescription = computed(() => `${settlementTrigger.value}后${billTime.value}个工作日内，甲方通过${settlementPaymentMethod.value}方式支付合同货款${settlementPercent.value}%，支付金额${settlementAmount.value}`);
const deliveryDescription = computed(() => `合同签订后${deliveryDays.value}日内送到甲方指定地点，运输方式：${transportMethod.value}`);
const technicalServiceDescription = computed(() => `是否需要提供技术服务与培训：${technicalServiceRequired.value}`);
const displayValue = (value) => value?.value ?? value ?? "—";
</script>

<style scoped>
.shared-contract-field-grid :deep(.el-input-number), .shared-contract-field-grid :deep(.el-select) { width: 100%; }
.field-unit { margin-left: 8px; color: #7f8c98; white-space: nowrap; }
.purchase-address-picker { width: 100%; display: flex; align-items: center; gap: 8px; }
.readonly-contract-value { min-height: 28px; color: #24364b; font-size: 14px; line-height: 1.7; overflow-wrap: anywhere; }
.settlement-clause { padding: 6px 10px; border-left: 3px solid #8fc7ff; background: #f7faff; }
.editable-settlement-clause { width: 100%; display: flex; align-items: center; flex-wrap: wrap; gap: 8px; color: #425466; line-height: 32px; }
.editable-settlement-clause .clause-trigger { width: 250px; }
.editable-settlement-clause .clause-number { width: 88px; }
.editable-settlement-clause .clause-method { width: 118px; }
.editable-settlement-clause .clause-percent { width: 82px; }
.editable-complex-clause { width: 100%; display: flex; align-items: center; flex-wrap: wrap; gap: 8px; color: #425466; line-height: 32px; }
.editable-complex-clause .clause-number { width: 88px; }
.editable-complex-clause .clause-text { width: 180px; }
</style>
