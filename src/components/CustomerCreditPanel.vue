<template>
  <div class="customer-credit-reuse-note">
    <span>与原内容保持一致</span>
  </div>
</template>

<script setup>
defineProps({ snapshot: Boolean });
defineEmits(["inspect"]);
</script>

<style scoped>
.customer-credit-reuse-note {
  display: flex;
  align-items: center;
  min-height: 52px;
  padding: 14px 16px;
  border: 1px solid #e4ebf3;
  border-radius: 6px;
  background: #f8fafc;
  color: #637386;
  font-size: 13px;
  line-height: 22px;
}
</style>
