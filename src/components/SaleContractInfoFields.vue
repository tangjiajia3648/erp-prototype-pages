<template>
  <el-row :gutter="16" class="shared-contract-field-grid">
    <template v-if="section === 'settlement'">
      <template v-if="readonly">
        <el-col :span="24"><el-form-item label="货款结算"><div class="readonly-contract-value settlement-clause">{{ settlementDescription }}</div></el-form-item></el-col>
      </template>
      <template v-else>
        <el-col :span="24"><el-form-item label="货款结算"><div class="editable-settlement-clause">
          <el-select v-model="settlementTrigger" class="clause-trigger"><el-option label="乙方发货" value="乙方发货" /><el-option label="甲方验收合格" value="甲方验收合格" /><el-option label="双方完成对账" value="双方完成对账" /></el-select>
          <span>后</span><el-input-number v-model="billTime" class="clause-number" :min="0" :max="9999" :controls="false" /><span>日内，甲方通过</span>
          <el-select v-model="settlementPaymentMethod" class="clause-method"><el-option label="转账" value="转账" /><el-option label="银行承兑" value="银行承兑" /><el-option label="其他" value="其他" /></el-select>
          <span>方式支付合同货款</span><el-input-number v-model="settlementPercent" class="clause-percent" :min="0" :max="100" :controls="false" /><span>%；支付金额</span><strong>{{ settlementAmount }}</strong>
        </div></el-form-item></el-col>
      </template>
    </template>
    <template v-else>
      <template v-if="readonly">
        <el-col :span="8"><el-form-item label="签约地点"><div class="readonly-contract-value">{{ displayValue(signingLocation) }}</div></el-form-item></el-col>
        <el-col :span="16"><el-form-item label="交付"><div class="readonly-contract-value settlement-clause">{{ deliveryDescription }}</div></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="技术支持及服务"><div class="readonly-contract-value settlement-clause">{{ technicalServiceDescription }}</div></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="发票"><div class="readonly-contract-value settlement-clause">{{ invoiceDescription }}</div></el-form-item></el-col>
      </template>
      <template v-else>
        <el-col :span="8"><el-form-item label="签约地点"><el-input v-model="signingLocation" /></el-form-item></el-col>
        <el-col :span="16"><el-form-item label="交付"><div class="editable-complex-clause"><el-select v-model="deliveryTrigger" class="delivery-trigger"><el-option label="合同生效" value="合同生效" /><el-option label="收到全款" value="收到全款" /><el-option label="收到订单" value="收到订单" /></el-select><span>后</span><el-input-number v-model="deliveryDays" class="clause-number" :min="0" :max="9999" :controls="false" /><span>日内完成交付</span></div></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="其他交付要求"><el-input v-model="deliveryRequirement" type="textarea" :rows="2" maxlength="300" show-word-limit resize="none" /></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="技术支持及服务"><div class="editable-complex-clause"><span>乙方</span><el-select v-model="installationRequired" class="clause-method"><el-option label="需要" value="需要" /><el-option label="不需要" value="不需要" /></el-select><span>提供安装调试，安装调试由</span><el-select v-model="installationProvider" class="clause-method"><el-option label="乙方" value="乙方" /><el-option label="厂商" value="厂商" /><el-option label="第三方" value="第三方" /></el-select><span>提供；质保与售后由</span><el-select v-model="warrantyProvider" class="clause-method"><el-option label="厂商" value="厂商" /><el-option label="乙方" value="乙方" /><el-option label="第三方" value="第三方" /></el-select><span>提供，质保时间与条件</span><el-input v-model="warrantyTerms" class="clause-wide-text" /></div></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="其他技术服务要求"><el-input v-model="technicalServiceRemark" type="textarea" :rows="2" maxlength="300" show-word-limit resize="none" /></el-form-item></el-col>
        <el-col :span="24"><el-form-item label="发票"><div class="editable-complex-clause"><el-select v-model="invoiceTrigger" class="invoice-trigger"><el-option label="甲方支付全款" value="甲方支付全款" /><el-option label="乙方完成交付" value="乙方完成交付" /><el-option label="双方完成对账" value="双方完成对账" /></el-select><span>后</span><el-input-number v-model="invoiceDays" class="clause-number" :min="0" :max="9999" :controls="false" /><span>日内，乙方向甲方开具</span><el-select v-model="invoiceType" class="invoice-type"><el-option label="增值税专用发票" value="专票" /><el-option label="增值税普通发票" value="普票" /><el-option label="未开票" value="未开票" /></el-select></div></el-form-item></el-col>
      </template>
      <el-col :span="24" class="contract-address-table-cell"><el-form-item label="收货地址" required><div class="contract-address-editor">
      <div v-if="!readonly" class="contract-address-toolbar"><el-button type="primary" plain @click="$emit('add-address')">选择客户地址</el-button><span>支持选择多条客户收货地址，至少保留1条</span></div>
      <el-table :data="model.shippingAddressList" border size="small" class="contract-address-table">
        <el-table-column label="联系人" min-width="120"><template #default="{ row }"><span v-if="readonly">{{ displayValue(row.contact) }}</span><el-input v-else v-model="row.contact" /></template></el-table-column>
        <el-table-column label="联系电话" min-width="150"><template #default="{ row }"><span v-if="readonly">{{ displayValue(row.phone) }}</span><el-input v-else v-model="row.phone" /></template></el-table-column>
        <el-table-column label="省市区" min-width="220"><template #default="{ row }"><span v-if="readonly">{{ displayValue(row.provinceCityDistrict) }}</span><el-input v-else v-model="row.provinceCityDistrict" /></template></el-table-column>
        <el-table-column label="详细地址" min-width="260"><template #default="{ row }"><span v-if="readonly">{{ displayValue(row.receiveAddress) }}</span><el-input v-else v-model="row.receiveAddress" /></template></el-table-column>
        <el-table-column v-if="!readonly" label="操作" width="76" fixed="right"><template #default="{ $index }"><el-button link type="danger" @click="$emit('remove-address', $index)">删除</el-button></template></el-table-column>
      </el-table>
      </div></el-form-item></el-col>
    </template>
  </el-row>
</template>

<script setup>
import { computed } from "vue";
const props = defineProps({ model: { type: Object, required: true }, source: { type: String, default: "contract" }, section: { type: String, default: "delivery" }, readonly: { type: Boolean, default: false }, amount: { type: String, default: "¥0.00" } });
defineEmits(["add-address", "remove-address"]);
const mappedField = (contractKey, orderKey = contractKey, fallback = "") => computed({ get: () => props.model[props.source === "order" ? orderKey : contractKey] ?? fallback, set: (value) => { props.model[props.source === "order" ? orderKey : contractKey] = value; } });
const signingLocation = mappedField("signingLocation", "signatureLocation", "成都");
const settlementTrigger = mappedField("settlementTrigger", "settlementTrigger", "乙方发货");
const billTime = mappedField("billTime", "billTime", 30);
const settlementPaymentMethod = mappedField("settlementPaymentMethod", "settlementPaymentMethod", "转账");
const settlementPercent = mappedField("settlementPercent", "settlementPercent", 100);
const invoiceTrigger = mappedField("invoiceTrigger", "invoiceTrigger", "甲方支付全款");
const invoiceDays = mappedField("invoiceDays", "invoiceDays", 7);
const invoiceType = mappedField("invoiceType", "invoiceType", "专票");
const deliveryTrigger = mappedField("deliveryTrigger", "deliveryTrigger", "合同生效");
const deliveryDays = mappedField("deliveryDays", "deliveryDays", 7);
const deliveryRequirement = mappedField("deliveryRequirement", "deliveryRequirement", "");
const installationRequired = mappedField("installationRequired", "installationRequired", "不需要");
const installationProvider = mappedField("installationProvider", "installationProvider", "乙方");
const warrantyProvider = mappedField("warrantyProvider", "warrantyProvider", "厂商");
const warrantyTerms = mappedField("warrantyTerms", "warrantyTerms", "按厂商规定执行");
const technicalServiceRemark = mappedField("technicalServiceRemark", "technicalServiceRemark", "");
const settlementAmount = computed(() => props.model.settlementAmount || props.amount);
const invoiceTypeLabel = computed(() => ({ 专票: "增值税专用发票", 普票: "增值税普通发票", 未开票: "未开票" }[invoiceType.value] || invoiceType.value));
const settlementDescription = computed(() => `${settlementTrigger.value}后${billTime.value}日内，甲方通过${settlementPaymentMethod.value}方式支付合同货款${settlementPercent.value}%，支付金额${settlementAmount.value}`);
const invoiceDescription = computed(() => `${invoiceTrigger.value}后${invoiceDays.value}日内，乙方向甲方开具${invoiceTypeLabel.value}`);
const deliveryDescription = computed(() => `${deliveryTrigger.value}后${deliveryDays.value}日内完成交付；其他交付要求：${deliveryRequirement.value || "无"}`);
const technicalServiceDescription = computed(() => `乙方${installationRequired.value}提供安装调试，安装调试由${installationProvider.value}提供；质保与售后由${warrantyProvider.value}提供，质保时间与条件：${warrantyTerms.value || "无"}；其他技术服务要求：${technicalServiceRemark.value || "无"}`);
const displayValue = (value) => (value?.value ?? value) || "—";
</script>

<style scoped>
.shared-contract-field-grid :deep(.el-input-number), .shared-contract-field-grid :deep(.el-select) { width: 100%; }
.field-unit { margin-left: 8px; color: #7f8c98; white-space: nowrap; }
.contract-address-editor { width: 100%; }
.contract-address-toolbar { display: flex; align-items: center; gap: 12px; margin-bottom: 10px; color: #7f8c98; font-size: 12px; }
.readonly-contract-value { min-height: 28px; color: #24364b; font-size: 14px; line-height: 1.7; overflow-wrap: anywhere; }
.settlement-clause { padding: 6px 10px; border-left: 3px solid #8fc7ff; background: #f7faff; }
.editable-settlement-clause { width: 100%; display: flex; align-items: center; flex-wrap: wrap; gap: 8px; color: #425466; line-height: 32px; }
.editable-settlement-clause .clause-trigger { width: 180px; }
.editable-settlement-clause .clause-number { width: 88px; }
.editable-settlement-clause .clause-method { width: 118px; }
.editable-settlement-clause .clause-percent { width: 82px; }
.editable-complex-clause { width: 100%; display: flex; align-items: center; flex-wrap: wrap; gap: 8px; color: #425466; line-height: 32px; }
.editable-complex-clause .delivery-trigger { width: 150px; }
.editable-complex-clause .invoice-trigger { width: 180px; }
.editable-complex-clause .invoice-type { width: 180px; }
.editable-complex-clause .clause-number { width: 88px; }
.editable-complex-clause .clause-method { width: 112px; }
.editable-complex-clause .clause-wide-text { width: 240px; }
</style>
