<template>
  <div class="goods-linked-tabs">
    <div class="goods-linked-tabbar" role="tablist" aria-label="商品明细查看">
      <button
        type="button"
        :class="{ active: activeView === 'main' }"
        @click="activeView = 'main'"
      >
        {{ mainLabel }}
      </button>
      <button
        v-if="itemProduct"
        type="button"
        :class="{ active: activeView === 'item' }"
        :title="`单品详情：${itemProduct.skuName || itemProduct.spuName || '当前商品'}`"
        @click="activeView = 'item'"
      >
        <span>单品详情 · {{ shortName(itemProduct) }}</span>
        <i title="关闭单品详情" @click.stop="closeView('item')">×</i>
      </button>
      <button
        v-if="inventoryProduct"
        type="button"
        :class="{ active: activeView === 'inventory' }"
        :title="`库存详情：${inventoryProduct.skuName || inventoryProduct.spuName || '当前商品'}`"
        @click="activeView = 'inventory'"
      >
        <span>库存详情 · {{ shortName(inventoryProduct) }}</span>
        <i title="关闭库存详情" @click.stop="closeView('inventory')">×</i>
      </button>
    </div>

    <div v-show="activeView === 'main'" class="goods-linked-main">
      <slot />
    </div>

    <section v-if="activeView === 'inventory' && inventoryProduct" class="goods-linked-detail">
      <header>
        <div>
          <h4>库存详情</h4>
          <p>库存按商品、库存主体及仓库维度展示</p>
        </div>
        <el-button link type="primary" @click="activeView = 'main'">返回{{ mainLabel }}</el-button>
      </header>
      <div class="goods-context-grid">
        <div><span>SPU名称</span><strong :title="inventoryProduct.spuName">{{ inventoryProduct.spuName || '—' }}</strong></div>
        <div><span>SKU名称</span><strong :title="inventoryProduct.skuName">{{ inventoryProduct.skuName || '—' }}</strong></div>
        <div><span>库存数量</span><strong>{{ inventoryQuantity(inventoryProduct) }}</strong></div>
        <div><span>配置说明</span><strong>{{ inventoryProduct.specModel || '—' }}</strong></div>
      </div>
      <el-table :data="inventoryRows" border size="small">
        <el-table-column type="index" label="序号" width="65" />
        <el-table-column prop="warehouse" label="仓库名称" min-width="240" />
        <el-table-column prop="quantity" label="数量" min-width="130" align="right">
          <template #default="{ row }"><el-link type="primary">{{ row.quantity }}</el-link></template>
        </el-table-column>
      </el-table>
      <div class="goods-detail-total">共 {{ inventoryRows.length }} 条</div>
    </section>

    <section v-if="activeView === 'item' && itemProduct" class="goods-linked-detail">
      <header>
        <div>
          <h4>单品详情</h4>
          <p>单品按采购、入库及序列号维度展示</p>
        </div>
        <el-button link type="primary" @click="activeView = 'main'">返回{{ mainLabel }}</el-button>
      </header>
      <div class="goods-context-grid compact">
        <div><span>SPU名称</span><strong :title="itemProduct.spuName">{{ itemProduct.spuName || '—' }}</strong></div>
        <div><span>SKU名称</span><strong :title="itemProduct.skuName">{{ itemProduct.skuName || '—' }}</strong></div>
        <div><span>单品数量</span><strong>{{ itemQuantity(itemProduct) }}</strong></div>
        <div><span>配置说明</span><strong>{{ itemProduct.specModel || '—' }}</strong></div>
      </div>
      <el-table :data="itemRows" border size="small">
        <el-table-column type="selection" width="42" />
        <el-table-column type="index" label="序号" width="55" />
        <el-table-column label="操作" width="65" align="center"><template #default>⋮</template></el-table-column>
        <el-table-column prop="purchaseDate" label="采购日期" width="115" />
        <el-table-column prop="buyer" label="采购人" width="95" />
        <el-table-column prop="owner" label="采购责任人" width="110" />
        <el-table-column prop="inboundDate" label="入库日期" width="115" />
        <el-table-column prop="warehouse" label="仓库名称" min-width="150" />
        <el-table-column prop="serialNumber" label="序列号" min-width="215"><template #default="{ row }"><el-link type="primary">{{ row.serialNumber }}</el-link></template></el-table-column>
        <el-table-column label="更新序列号" width="100"><template #default><el-link type="primary">编辑</el-link></template></el-table-column>
        <el-table-column prop="reconciliationStatus" label="采购对账状态" width="125"><template #default="{ row }"><el-link type="primary">{{ row.reconciliationStatus }}</el-link></template></el-table-column>
        <el-table-column prop="unboxStatus" label="开箱状态" width="100" />
        <el-table-column prop="unboxRemark" label="开箱说明" min-width="120" />
        <el-table-column prop="invoiceType" label="发票" width="100" />
        <el-table-column prop="purchaseRemark" label="采购说明" min-width="130" />
      </el-table>
      <div class="goods-detail-total">共 {{ itemQuantity(itemProduct) }} 条</div>
    </section>
  </div>
</template>

<script setup>
import { computed, ref } from "vue";

defineProps({
  mainLabel: { type: String, default: "商品明细" },
});

const activeView = ref("main");
const inventoryProduct = ref(null);
const itemProduct = ref(null);

const inventoryRows = computed(() => {
  const product = inventoryProduct.value;
  if (!product) return [];
  return [{ warehouse: "九州科瑞大河库", quantity: inventoryQuantity(product) }];
});

const itemRows = computed(() => {
  const product = itemProduct.value;
  if (!product) return [];
  const total = Math.max(1, Math.min(itemQuantity(product), 4));
  return Array.from({ length: total }, (_, index) => ({
    purchaseDate: index < 3 ? "2026-02-09" : "2026-04-29",
    buyer: "代莉",
    owner: "代莉",
    inboundDate: index < 3 ? "2026-02-09" : "2026-04-29",
    warehouse: "九州科瑞大河库",
    serialNumber: `69897eabe4b01${String(index + 1).padStart(3, "0")}R${index + 1}`,
    reconciliationStatus: index < 3 ? "已对账未结算" : "—",
    unboxStatus: "—",
    unboxRemark: "—",
    invoiceType: "专用发票",
    purchaseRemark: "—",
  }));
});

function shortName(product) {
  const name = product?.skuName || product?.spuName || "当前商品";
  if (name.length <= 13) return name;
  const suffix = name.includes("/") ? name.split("/").filter(Boolean).at(-1) : "";
  return suffix && suffix.length <= 9
    ? `${name.slice(0, 6)}…${suffix}`
    : `${name.slice(0, 10)}…`;
}

function inventoryQuantity(product) {
  return Number(product?.stock ?? product?.inventoryQuantity ?? product?.quantity ?? 0);
}

function itemQuantity(product) {
  return Number(product?.purchaseQuantity ?? product?.budgetQuantity ?? product?.quantity ?? 0);
}

function openInventory(product) {
  inventoryProduct.value = { ...product };
  activeView.value = "inventory";
}

function openItem(product) {
  itemProduct.value = { ...product };
  activeView.value = "item";
}

function closeView(view) {
  if (view === "inventory") inventoryProduct.value = null;
  if (view === "item") itemProduct.value = null;
  if (activeView.value === view) activeView.value = "main";
}

defineExpose({ openInventory, openItem });
</script>

<style scoped>
.goods-linked-tabs { min-width: 0; }
.goods-linked-tabbar { display: flex; align-items: flex-end; gap: 2px; min-height: 38px; margin: -4px 0 12px; border-bottom: 1px solid #dfe6ee; overflow-x: auto; }
.goods-linked-tabbar button { display: inline-flex; align-items: center; gap: 8px; flex: 0 0 auto; max-width: 240px; height: 38px; padding: 0 14px; border: 0; border-bottom: 2px solid transparent; background: transparent; color: #637181; cursor: pointer; }
.goods-linked-tabbar button span { max-width: 204px; overflow: hidden; text-overflow: ellipsis; white-space: nowrap; }
.goods-linked-tabbar button.active { border-bottom-color: #1684e8; color: #1684e8; font-weight: 600; }
.goods-linked-tabbar i { color: #9aa6b2; font-style: normal; font-size: 17px; line-height: 1; }
.goods-linked-tabbar i:hover { color: #e15555; }
.goods-linked-detail { min-width: 0; }
.goods-linked-detail > header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; }
.goods-linked-detail h4 { margin: 0; color: #263442; font-size: 15px; }
.goods-linked-detail p { margin: 4px 0 0; color: #94a0ad; font-size: 12px; }
.goods-context-grid { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 18px; margin-bottom: 12px; padding: 12px 14px; background: #f7f9fc; }
.goods-context-grid > div { min-width: 0; }
.goods-context-grid span { display: block; margin-bottom: 6px; color: #7b8896; font-size: 12px; }
.goods-context-grid strong { display: block; overflow: hidden; color: #374454; font-size: 13px; text-overflow: ellipsis; white-space: nowrap; }
.goods-detail-total { padding-top: 8px; color: #7b8896; font-size: 12px; text-align: right; }
@media (max-width: 1100px) { .goods-context-grid { grid-template-columns: repeat(2, minmax(0, 1fr)); } }
</style>
