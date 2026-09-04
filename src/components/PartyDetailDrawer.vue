<template>
  <el-drawer :model-value="modelValue" :title="kind === 'customer' ? '客户详情' : '供应商详情'" size="min(960px, 94vw)" append-to-body @update:model-value="$emit('update:modelValue', $event)">
    <div class="party-detail">
      <el-alert title="原型演示数据；空白业务数据统一显示“—”" type="info" :closable="false" />
      <h3>基础信息</h3>
      <div class="party-grid">
        <div v-for="field in baseFields" :key="field.label"><label>{{ field.label }}</label><span>{{ field.value || '—' }}</span></div>
      </div>
      <h3>往来金额</h3>
      <div class="party-grid">
        <div v-for="label in amountLabels" :key="label"><label>{{ label }}</label><span class="amount">0.00</span></div>
      </div>
      <template v-if="kind === 'customer'">
        <h3>资信数据</h3>
        <div class="party-grid">
          <div v-for="label in creditLabels" :key="label"><label>{{ label }}</label><span>—</span></div>
        </div>
        <h3>月度提货与超期情况</h3>
        <div class="party-table-wrap">
          <el-table :data="creditRows" border>
            <el-table-column prop="label" label="指标 / 月份" width="240" fixed />
            <el-table-column v-for="month in 6" :key="month" :label="'月份' + month" width="110"><template #default>—</template></el-table-column>
          </el-table>
        </div>
      </template>
    </div>
  </el-drawer>
</template>

<script setup>
import { computed } from "vue";
const props = defineProps({ modelValue: Boolean, name: String, kind: { type: String, default: "supplier" } });
defineEmits(["update:modelValue"]);
const baseFields = computed(() => [
  { label: "名称", value: props.name },
  { label: "工商名称", value: props.name },
  { label: "统一社会信用代码" }, { label: "法定代表人" },
  { label: "省市区" }, { label: "详细地址" },
  { label: "业务员" }, { label: "创建时间" }, { label: "标签" },
  ...(props.kind === "customer" ? [{ label: "华为省代CBG Code" }, { label: "华为KA客户编码" }, { label: "销售在途（数量 / 金额）" }] : [{ label: "采购在途（数量 / 金额）" }]),
]);
const amountLabels = computed(() => props.kind === "customer"
  ? ["预收金额", "应收金额", "应付发票", "客户价保", "客户已用折扣", "客户待结算价保"]
  : ["预付金额", "应付金额", "应收发票", "供应商价保", "供应商待结算价保", "可用折扣", "应收价保", "应收奖励"]);
const creditLabels = ["公司名称", "信用等级", "注册资本（万）", "实缴资本（万）", "成立日期", "法定代表人", "上年度参保人数", "关联往来单位", "授信额度", "本次申请额度", "当前额度占用", "当前超期应收", "当前最长超期天数", "关联企业额度占用", "关联企业超期应收", "应收余额", "预收余额", "应付", "预付", "当前可用额度", "现款占用", "账期占用"];
const creditRows = ["月提货次数", "月提货金额（万）", "月账期提货金额（万）", "月账期提货占比", "超期订单平均超期天数", "最长超期天数/金额（万）", "超期次数/总提货次数", "超期金额（万）/总提货金额（万）"].map(label => ({ label }));
</script>

<style scoped>
.party-detail { min-width: 0; }
h3 { margin: 22px 0 14px; border-left: 3px solid #1687f8; padding-left: 10px; font-size: 15px; }
.party-grid { display: grid; grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 18px 24px; }
.party-grid > div { min-width: 0; }
label { display: block; margin-bottom: 7px; color: #7b8896; font-size: 12px; }
span { display: block; color: #344252; line-height: 22px; overflow-wrap: anywhere; white-space: pre-wrap; }
.amount { font-variant-numeric: tabular-nums; }
.party-table-wrap { max-width: 100%; overflow: auto; }
@media (max-width: 760px) { .party-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); } }
@media (max-width: 480px) { .party-grid { grid-template-columns: minmax(0, 1fr); } }
</style>
