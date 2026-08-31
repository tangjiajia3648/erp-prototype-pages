<template>
  <div :class="['workspace', { 'budget-flow-workspace': prototypeVariant === 'flow-template' }]">
    <aside class="task-rail">
      <div class="rail-title">业务工作台</div>
      <button :class="{ active: activeView === 'list' }" @click="switchView('list')">
        <List /> 分销产品预算
      </button>
      <button :class="{ active: activeView === 'contract' }" @click="openContractPrototype">
        <DocumentChecked /> 合同管理
      </button>
      <button :class="{ active: activeView === 'saleOrders' }" @click="switchView('saleOrders')">
        <Document /> 销售订单
      </button>
      <button :class="{ active: activeView === 'purchaseOrders' }" @click="switchView('purchaseOrders')">
        <Document /> 采购订单
      </button>
    </aside>

    <main class="stage">
      <div class="business-tabs" role="tablist" aria-label="已打开的业务页面">
        <button
          v-for="tab in tabs"
          :key="tab.key"
          :class="{ active: activeTab === tab.key }"
          role="tab"
          @click="activateTab(tab.key)"
        >
          {{ tab.title }}
          <Close v-if="tab.closable" class="tab-close" @click.stop="requestClose(tab.key)" />
        </button>
        <div class="approval-launch"><el-badge :value="pendingItems.length" :hidden="!pendingItems.length"><button class="approval-shortcut" title="我的审批" @click="switchView('approval')"><Checked /><span>我的审批</span></button></el-badge></div>
      </div>

      <section v-if="activeView === 'list' && !currentDocument" class="list-page">
        <header class="page-heading">
          <div><h1>分销产品预算</h1></div>
          <el-button type="primary" :icon="Plus" @click="openDocument('create')">新建预算</el-button>
        </header>
        <div class="filters">
          <el-segmented v-model="listStatus" :options="['全部', '审批中', '审批完成', '草稿']" />
          <el-badge :value="pendingContracts.length" :hidden="!pendingContracts.length"><el-button @click="pendingContractVisible = true">待提交合同</el-button></el-badge>
          <el-input v-model="keyword" clearable placeholder="预算单号 / 供应商 / 责任人" :prefix-icon="Search" />
          <el-button :icon="Filter" @click="advancedSearchVisible = true">高级搜索</el-button>
          <el-button :icon="Setting">自定义列</el-button>
        </div>
        <el-table :data="filteredBudgets" border stripe>
          <el-table-column type="selection" width="50" fixed="left" />
          <el-table-column type="index" label="序号" width="60" fixed="left" />
          <el-table-column v-for="column in listColumns" :key="column.prop" :prop="column.prop" :label="column.label" :min-width="column.width || 120" :align="column.align" :fixed="column.fixed" show-overflow-tooltip>
            <template #default="{ row }">
              <el-link v-if="column.prop === 'code'" type="primary" @click="openDocument('detail', row)">{{ row.code }}</el-link>
              <el-tag v-else-if="column.prop === 'status'" :type="statusType(row.status)" effect="light">{{ row.status }}</el-tag>
              <el-link v-else-if="column.prop === 'modifyRecord' && row.version" type="primary" @click="openModificationRecords(row)">查看修改记录</el-link>
              <span v-else>{{ row[column.prop] ?? '-' }}</span>
            </template>
          </el-table-column>
          <el-table-column label="操作" width="130" fixed="right">
            <template #default="{ row }">
              <el-dropdown trigger="click"><el-button link type="primary">业务操作<ArrowDown /></el-button><template #dropdown><el-dropdown-menu><el-dropdown-item @click="openDocument('detail', row)">查看详情</el-dropdown-item><el-dropdown-item v-if="row.status === '草稿'" @click="openDocument('edit', row)">编辑预算</el-dropdown-item><el-dropdown-item v-if="row.status === '审批完成'" divided @click="openContractDraft(row, 'purchase')">创建采购合同</el-dropdown-item><el-dropdown-item v-if="row.status === '审批完成'" @click="openContractDraft(row, 'sale')">创建销售合同</el-dropdown-item><el-dropdown-item v-if="row.status === '审批完成'" divided @click="openOrderDraft(row, 'purchase')">创建采购订单</el-dropdown-item><el-dropdown-item v-if="row.status === '审批完成'" @click="openOrderDraft(row, 'sale')">创建销售订单</el-dropdown-item><el-dropdown-item v-if="row.version" divided @click="openModificationRecords(row)">预算修改记录</el-dropdown-item></el-dropdown-menu></template></el-dropdown>
            </template>
          </el-table-column>
        </el-table>
      </section>

      <section v-else-if="activeView === 'saleOrders' && !activeOrderPage" class="list-page order-list-page">
        <header class="page-heading"><div><h1>销售订单</h1><p>关注客户订单的发货、开票和收款执行情况</p></div><el-button type="primary" :icon="Plus" @click="openManualOrder('sale')">新增销售订单</el-button></header>
        <div class="filters"><el-segmented v-model="saleOrderStatus" :options="['全部','草稿','审批中','待发货','已完成']" /><el-input v-model="orderKeyword" clearable placeholder="订单编号 / 客户 / 业务员" :prefix-icon="Search" /><el-button :icon="Filter">高级搜索</el-button><el-button :icon="Setting">自定义列</el-button></div>
        <el-table :data="saleOrderRows" border stripe><el-table-column type="selection" width="44" fixed="left" /><el-table-column type="index" label="序号" width="55" fixed="left" /><el-table-column prop="code" label="单据编号" min-width="170" fixed="left"><template #default="{ row }"><el-link type="primary">{{ row.code }}</el-link></template></el-table-column><el-table-column prop="status" label="单据状态" width="100"><template #default="{ row }"><el-tag :type="row.status === '已完成' ? 'success' : row.status === '审批中' ? 'warning' : 'info'">{{ row.status }}</el-tag></template></el-table-column><el-table-column prop="contractSituation" label="合同情况" width="120" /><el-table-column prop="budgetCode" label="关联预算单" min-width="165"><template #default="{ row }"><el-link v-if="row.budgetCode !== '-'" type="primary">{{ row.budgetCode }}</el-link><span v-else>-</span></template></el-table-column><el-table-column prop="customer" label="客户" min-width="190" show-overflow-tooltip /><el-table-column prop="businessType" label="业务类型" width="125" /><el-table-column prop="businessUnit" label="业务单元" width="120" /><el-table-column prop="salesman" label="业务员" width="95" /><el-table-column prop="billTime" label="账期(天)" width="85" align="right" /><el-table-column prop="contractCode" label="合同编号" min-width="165" /><el-table-column prop="department" label="部门" width="100" /><el-table-column prop="created" label="制单时间" width="160" /><el-table-column prop="invoiceType" label="发票类型" width="105" /><el-table-column prop="amount" label="订单金额" width="125" align="right" /><el-table-column prop="outboundAmount" label="已发货金额" width="125" align="right" /><el-table-column prop="invoiceAmount" label="发票金额" width="120" align="right" /><el-table-column prop="paidAmount" label="已收款金额" width="125" align="right" /><el-table-column label="操作" width="110" fixed="right"><template #default><el-button link type="primary">查看详情</el-button></template></el-table-column></el-table>
      </section>

      <section v-else-if="activeView === 'purchaseOrders' && !activeOrderPage" class="list-page order-list-page">
        <header class="page-heading"><div><h1>采购订单</h1><p>关注供应商订单的到货、入库、付款和价保情况</p></div><el-button type="primary" :icon="Plus" @click="openManualOrder('purchase')">新增采购订单</el-button></header>
        <div class="filters"><el-segmented v-model="purchaseOrderStatus" :options="['全部','草稿','审批中','待入库','入库中','完成']" /><el-input v-model="orderKeyword" clearable placeholder="订单编号 / 供应商 / 责任人" :prefix-icon="Search" /><el-button :icon="Filter">高级搜索</el-button><el-button :icon="Setting">自定义列</el-button></div>
        <el-table :data="purchaseOrderRows" border stripe><el-table-column type="selection" width="44" fixed="left" /><el-table-column type="index" label="序号" width="55" fixed="left" /><el-table-column prop="code" label="单据编号" min-width="170" fixed="left"><template #default="{ row }"><el-link type="primary">{{ row.code }}</el-link></template></el-table-column><el-table-column prop="contractSituation" label="合同情况" width="120" /><el-table-column prop="directDelivery" label="是否直发" width="90" /><el-table-column prop="status" label="单据状态" width="100"><template #default="{ row }"><el-tag :type="row.status === '完成' ? 'success' : row.status === '审批中' ? 'warning' : 'info'">{{ row.status }}</el-tag></template></el-table-column><el-table-column prop="budgetCode" label="关联预算单" min-width="165" /><el-table-column prop="businessType" label="业务类型" width="125" /><el-table-column prop="entity" label="合同签署主体" min-width="170" /><el-table-column prop="supplier" label="供应商" min-width="180" show-overflow-tooltip /><el-table-column prop="purchaseOwner" label="采购责任人" width="110" /><el-table-column prop="saleOwner" label="销售责任人" width="110" /><el-table-column prop="billTime" label="账期(天)" width="85" align="right" /><el-table-column prop="contractCode" label="合同编号" min-width="165" /><el-table-column prop="amount" label="订单金额" width="125" align="right" /><el-table-column prop="priceProtection" label="价保" width="110" align="right" /><el-table-column prop="discount" label="折扣" width="110" align="right" /><el-table-column prop="inboundAmount" label="入库金额" width="120" align="right" /><el-table-column prop="invoiceAmount" label="发票金额" width="120" align="right" /><el-table-column prop="paidAmount" label="已付金额" width="120" align="right" /><el-table-column prop="payStatus" label="付款状态" width="100" /><el-table-column label="操作" width="110" fixed="right"><template #default><el-button link type="primary">查看详情</el-button></template></el-table-column></el-table>
      </section>

      <section v-else-if="activeView === 'approval' && !currentDocument" class="approval-center">
        <header class="page-heading"><div><h1>我的审批</h1></div><el-button :icon="Refresh" @click="resetApprovalFilters">重置筛选</el-button></header>
        <el-tabs v-model="approvalTab">
          <el-tab-pane :label="`待办 ${approvalCounts.pending}`" name="pending" />
          <el-tab-pane :label="`已办 ${approvalCounts.done}`" name="done" />
          <el-tab-pane :label="`我提交的 ${approvalCounts.mine}`" name="mine" />
        </el-tabs>
        <div class="approval-filters">
          <el-select v-model="approvalFilters.businessType" clearable placeholder="全部业务类型"><el-option v-for="type in approvalBusinessTypes" :key="type" :label="type" :value="type" /></el-select>
          <el-select v-model="approvalFilters.department" clearable placeholder="全部所属部门"><el-option v-for="dept in approvalDepartments" :key="dept" :label="dept" :value="dept" /></el-select>
          <el-date-picker v-model="approvalFilters.dateRange" type="daterange" value-format="YYYY-MM-DD" start-placeholder="提交开始日期" end-placeholder="提交结束日期" />
          <el-input v-model="approvalFilters.keyword" clearable :prefix-icon="Search" placeholder="单据编号 / 申请人 / 企业名称" />
        </div>
        <el-table :data="filteredApprovalItems" border stripe height="calc(100% - 138px)">
          <el-table-column type="index" label="序号" width="60" />
          <el-table-column prop="businessType" label="业务类型" min-width="140" />
          <el-table-column prop="title" label="审批事项" min-width="160" />
          <el-table-column prop="code" label="单据编号" min-width="180"><template #default="{ row }"><el-link type="primary" @click="handleApprovalOpen(row)">{{ row.code }}</el-link></template></el-table-column>
          <el-table-column prop="supplier" label="供应商/客户" min-width="190" show-overflow-tooltip />
          <el-table-column prop="applicant" label="申请人" width="100" />
          <el-table-column prop="department" label="所属部门" width="130" />
          <el-table-column prop="currentNode" label="当前审批节点" min-width="150" />
          <el-table-column prop="submittedAt" label="提交审批时间" width="170" sortable />
          <el-table-column prop="statusLabel" label="状态" width="110"><template #default="{ row }"><el-tag :type="row.state === 'pending' ? 'warning' : row.state === 'done' ? 'success' : 'info'">{{ row.statusLabel }}</el-tag></template></el-table-column>
          <el-table-column label="操作" width="100" fixed="right"><template #default="{ row }"><el-button link type="primary" @click="handleApprovalOpen(row)">{{ row.state === 'pending' ? '去审批' : '查看' }}</el-button></template></el-table-column>
        </el-table>
      </section>

      <section v-else-if="activeView === 'contract' && !currentDocument && !activeContractPage && !activeContractEditPage && !activeSupplierPage && !activeOrderPage" class="list-page contract-management-page">
        <header class="page-heading"><div><h1>合同管理</h1><p>统一管理采购合同、销售合同及其关联业务</p></div><el-dropdown trigger="click"><el-button type="primary" :icon="Plus">新建合同<ArrowDown /></el-button><template #dropdown><el-dropdown-menu><el-dropdown-item @click="openContractPage('create', 'purchase')">新建采购合同</el-dropdown-item><el-dropdown-item @click="openContractPage('create', 'sale')">新建销售合同</el-dropdown-item></el-dropdown-menu></template></el-dropdown></header>
        <el-tabs v-model="contractManageTab"><el-tab-pane label="单项合同" name="singleContract" /><el-tab-pane label="销售框架协议" name="saleFrameworkAgreement" /><el-tab-pane label="采购框架协议" name="purchaseFrameworkAgreement" /></el-tabs>
        <div class="filters"><el-select v-model="contractTypeFilter" clearable placeholder="合同类型"><el-option label="采购合同" value="purchase" /><el-option label="销售合同" value="sale" /><el-option label="采购销售一体" value="combined" /></el-select><el-select v-model="contractStatusFilter" clearable placeholder="审核状态"><el-option label="待提交" value="待提交" /><el-option label="审批中" value="审批中" /><el-option label="已生效" value="已生效" /></el-select><el-badge :value="1"><el-button>待用印合同</el-button></el-badge><el-input v-model="contractKeyword" clearable placeholder="预算单号 / 合同号 / 客户 / 供应商" :prefix-icon="Search" /><el-button :icon="Filter">高级搜索</el-button><el-button :icon="Refresh" circle /></div>
        <el-table :data="filteredContractRows" border stripe><el-table-column type="selection" width="50" /><el-table-column type="index" label="序号" width="60" /><el-table-column prop="code" label="合同编号" min-width="175"><template #default="{ row }"><el-link type="primary" @click="openContractPage('detail', row.type, row)">{{ row.code }}</el-link></template></el-table-column><el-table-column prop="typeLabel" label="合同类型" width="110" /><el-table-column prop="enterprise" label="供应商/客户" min-width="190" show-overflow-tooltip /><el-table-column prop="budgetCode" label="关联预算单" min-width="170" /><el-table-column prop="orderCode" label="关联订单" min-width="170" /><el-table-column prop="amount" label="合同金额" width="135" align="right" /><el-table-column prop="owner" label="责任人" width="100" /><el-table-column prop="status" label="状态" width="100"><template #default="{ row }"><el-tag :type="row.status === '已生效' ? 'success' : row.status === '审批中' ? 'warning' : row.status === '已作废' ? 'danger' : 'info'">{{ row.status }}</el-tag></template></el-table-column><el-table-column prop="created" label="创建时间" width="165" /><el-table-column label="操作" width="130" fixed="right"><template #default="{ row }"><el-dropdown><el-button link type="primary">业务操作<ArrowDown /></el-button><template #dropdown><el-dropdown-menu><el-dropdown-item @click="openContractPage('detail', row.type, row)">查看详情</el-dropdown-item><el-dropdown-item v-if="row.status === '待提交'" @click="openContractPage('edit', row.type, row)">编辑合同</el-dropdown-item><el-dropdown-item v-if="row.status === '已生效'" @click="openContractArchive(row)">合同归档</el-dropdown-item><el-dropdown-item v-if="!['审批中', '已作废'].includes(row.status)" divided @click="cancelContract(row)">合同作废</el-dropdown-item></el-dropdown-menu></template></el-dropdown></template></el-table-column></el-table>
      </section>

      <section v-else-if="activeView === 'contract' && activeSupplierPage" class="document-page supplier-detail-page"><header class="document-header"><div class="title-block"><div class="eyebrow">供应商管理 · 供应商详情</div><h1>{{ activeSupplierPage.data.name }} <el-tag type="success">启用</el-tag></h1><p>统一社会信用代码：91510100MA6XXXXX · 业务员：miya</p></div></header><div class="document-scroll"><article class="section-card"><SectionTitle number="01" title="基础信息" /><el-descriptions :column="3" border><el-descriptions-item label="供应商名称">{{ activeSupplierPage.data.name }}</el-descriptions-item><el-descriptions-item label="统一社会信用代码">91510100MA6XXXXX</el-descriptions-item><el-descriptions-item label="法定代表人">姜心</el-descriptions-item><el-descriptions-item label="工商名称">{{ activeSupplierPage.data.name }}</el-descriptions-item><el-descriptions-item label="省市区">四川省成都市</el-descriptions-item><el-descriptions-item label="创建时间">2026-08-03 10:03:51</el-descriptions-item></el-descriptions></article><article class="section-card"><SectionTitle number="02" title="往来金额" /><div class="supplier-amount-grid"><div v-for="item in supplierAmounts" :key="item.label"><span>{{ item.label }}</span><el-link type="primary">{{ item.value }}</el-link><small>查看明细</small></div></div></article><article class="section-card"><SectionTitle number="03" title="采购在途" /><el-table :data="supplierTransitRows" border><el-table-column prop="spu" label="采购SPU" min-width="180" /><el-table-column prop="sku" label="采购SKU" min-width="220" /><el-table-column prop="quantity" label="在途数量" width="110" /><el-table-column prop="amount" label="在途金额" width="130" align="right" /><el-table-column prop="arrival" label="预计到货日期" width="140" /></el-table></article></div></section>

      <section v-else-if="activeView === 'contract' && activeContractPage" :class="['document-page', 'contract-detail-page', { 'contract-approval-view': activeContractPage.data.status === '审批中' }]">
        <header :class="['document-header', 'contract-detail-summary-header', { 'contract-approval-header': activeContractPage.data.status === '审批中' }]">
          <div class="title-block"><div class="eyebrow"><span>{{ activeContractPage.data.status === '审批中' ? '我的审批 · 合同审批' : '合同管理 · 合同详情' }}</span><span>{{ activeContractPage.data.budget?.type }}</span><span>{{ contractModeLabel(activeContractPage.data.contractMode) }}</span></div><h1>{{ activeContractPage.data.status === '审批中' ? `${activeContractPage.data.typeLabel}审批` : activeContractPage.data.code }} <el-tag :type="activeContractPage.data.status === '已生效' ? 'success' : activeContractPage.data.status === '已作废' ? 'danger' : 'warning'">{{ activeContractPage.data.status }}</el-tag></h1><p class="document-meta"><span v-if="activeContractPage.data.status === '审批中'">合同编号：{{ activeContractPage.data.code }}</span><span>{{ activeContractPage.data.type === 'purchase' ? '供应商' : '客户' }}：{{ activeContractPage.data.enterprise }}</span><span>{{ activeContractPage.data.type === 'purchase' ? '采购责任人' : '销售责任人' }}：{{ activeContractPage.data.owner }}</span><span>制单时间：{{ activeContractPage.data.created }}</span><span>最后更新：2026-08-24 10:32</span></p><div v-if="activeContractPage.data.status === '审批中'" class="contract-approval-metrics"><div><span>合同总额（含税）</span><strong>{{ activeContractPage.data.amount }}</strong></div><div><span>预算单毛利</span><strong class="positive">¥57,000.00</strong></div><div><span>预算单毛利率</span><strong class="positive">18.18%</strong></div></div></div>
          <div v-if="activeContractPage.data.status !== '审批中'" class="contract-detail-header-metrics"><div><span>合同总额（含税）</span><strong>{{ activeContractPage.data.amount }}</strong></div><div><span>预算单毛利</span><strong>¥57,000.00</strong></div><div><span>预算单毛利率</span><strong>18.18%</strong></div></div>
          <div class="budget-create-header-actions"><template v-if="activeContractPage.data.status === '审批中'"><el-button>驳回</el-button><el-button type="primary">通过</el-button></template><template v-else-if="activeContractPage.data.status !== '已作废'"><el-button @click="openContractArchive(activeContractPage.data)">归档</el-button><el-button type="danger" plain @click="cancelContract(activeContractPage.data)">作废</el-button></template></div>
        </header>
        <div ref="contractDetailScrollArea" class="document-scroll" @scroll="updateContractDetailCurrentModule">
          <article class="section-card contract-readonly-section"><SectionTitle number="01" title="基础信息" /><el-form label-position="top" disabled><el-row :gutter="16"><el-col :span="8"><el-form-item :label="activeContractPage.data.type === 'purchase' ? '供应商' : '客户'"><el-link class="detail-entity-link" type="primary">{{ activeContractPage.data.enterprise }}</el-link></el-form-item></el-col><el-col :span="8"><el-form-item :label="activeContractPage.data.type === 'purchase' ? '采购责任人' : '销售责任人'"><el-input :model-value="activeContractPage.data.owner" /></el-form-item></el-col><el-col :span="8"><el-form-item label="合同签署主体"><el-input :model-value="activeContractPage.data.budget?.entity" /></el-form-item></el-col><el-col :span="8"><el-form-item label="业务单元"><el-input model-value="西南业务部" /></el-form-item></el-col><el-col :span="8"><el-form-item label="业务类型"><el-input :model-value="activeContractPage.data.budget?.type" /></el-form-item></el-col><el-col :span="8"><el-form-item label="合同编号"><el-input :model-value="activeContractPage.data.code" /></el-form-item></el-col></el-row></el-form><el-collapse class="secondary-contract-info"><el-collapse-item title="更多基础信息"><el-descriptions :column="3" border><el-descriptions-item label="创建人">{{ activeContractPage.data.owner }}</el-descriptions-item><el-descriptions-item label="用印状态">{{ activeContractPage.data.status === '已生效' ? '已完成用印' : '待用印' }}</el-descriptions-item><el-descriptions-item label="签署日期">{{ activeContractPage.data.status === '已生效' ? '2026-08-21' : '-' }}</el-descriptions-item><el-descriptions-item label="提交审批时间">2026-08-20 15:10</el-descriptions-item><el-descriptions-item label="审批通过时间">{{ activeContractPage.data.status === '已生效' ? '2026-08-21 10:30' : '-' }}</el-descriptions-item></el-descriptions></el-collapse-item></el-collapse></article>
          <article class="section-card contract-readonly-section"><SectionTitle number="02" title="关联单据信息" /><h4 class="contract-content-subtitle">基础信息</h4><el-form label-position="top" disabled><el-row :gutter="16"><el-col :span="8"><el-form-item label="预算单编号"><div class="readonly-link-field"><el-link type="primary">{{ activeContractPage.data.budgetCode }}</el-link><span>↗</span></div></el-form-item></el-col><el-col :span="8"><el-form-item label="发起方"><el-input model-value="预算单" /></el-form-item></el-col><el-col :span="8"><el-form-item label="预算有效期"><el-input model-value="60 天" /></el-form-item></el-col><el-col :span="8"><el-form-item label="单据状态"><el-input model-value="审批完成" /></el-form-item></el-col><el-col :span="8"><el-form-item label="市场环境说明"><el-input model-value="价格稳定" /></el-form-item></el-col><el-col :span="8"><el-form-item label="预计库存周期"><el-input model-value="45 天" /></el-form-item></el-col><el-col v-if="activeContractPage.data.type === 'purchase' && activeContractPage.data.orderCode !== '-'" :span="8"><el-form-item label="关联采购订单"><div class="readonly-link-field"><el-link type="primary">{{ activeContractPage.data.orderCode }}</el-link><span>↗</span></div></el-form-item></el-col></el-row></el-form><el-collapse class="secondary-contract-info detail-budget-collapse"><el-collapse-item title="预算计划 / 收益测算"><div class="split-panel contract-budget-plan"><div><h4 class="contract-subtitle">预算计划</h4><el-form label-position="top" disabled><el-row :gutter="16"><el-col :span="12"><el-form-item label="采购账期"><el-input model-value="30 天" /></el-form-item></el-col><el-col :span="12"><el-form-item label="预计付款日期"><el-input model-value="2026-09-20" /></el-form-item></el-col><el-col :span="12"><el-form-item label="最长销售周期"><el-input model-value="45 天" /></el-form-item></el-col><el-col :span="12"><el-form-item label="预计项目单毛利回补总额"><el-input model-value="¥0.00" /></el-form-item></el-col><el-col :span="24"><el-form-item label="利润说明"><el-input model-value="价格及毛利测算依据预算单" type="textarea" :rows="2" /></el-form-item></el-col><el-col :span="24"><el-form-item label="业务标签"><el-input model-value="重点业务" /></el-form-item></el-col></el-row></el-form></div><div class="metrics"><h4 class="contract-subtitle">收益测算</h4><div class="metric-grid"><div v-for="metric in metrics" :key="metric.label" :class="{ warning: metric.warning }"><span>{{ metric.label }}</span><strong>{{ metric.value }}</strong><small v-if="metric.note">{{ metric.note }}</small></div></div></div></div></el-collapse-item><el-collapse-item title="预算附件（1）"><div class="source-budget-attachments"><div class="file-row"><Document /><div><strong>产品报价及采购说明.pdf</strong><span>来源预算单附件</span></div><el-link type="primary">预览</el-link></div></div></el-collapse-item></el-collapse></article>
          <article class="section-card contract-readonly-section"><SectionTitle number="03" :title="activeContractPage.data.type === 'purchase' ? '采购清单' : '销售清单'" /><div v-if="activeContractPage.data.status === '审批中'" class="contract-local-attention"><WarningFilled /><span>本次清单存在1项预算差异，请重点核对数量与价格。</span></div><el-table :data="goods" border size="small"><el-table-column prop="spuName" label="SPU名称" min-width="170" fixed="left" show-overflow-tooltip /><el-table-column prop="skuName" label="SKU名称" min-width="220" fixed="left" show-overflow-tooltip /><el-table-column prop="specModel" label="配置说明" min-width="150" /><el-table-column prop="budgetQuantity" label="预算数量" width="100" align="right" /><el-table-column prop="purchaseQuantity" label="本次数量" width="100" align="right" /><el-table-column :prop="activeContractPage.data.type === 'purchase' ? 'purchaseAmount' : 'salePrice'" :label="activeContractPage.data.type === 'purchase' ? '单台采购价' : '单台销售价'" width="120" align="right" /><el-table-column label="合计金额" width="130" align="right"><template #default="{ row }">¥{{ (row.purchaseQuantity * (activeContractPage.data.type === 'purchase' ? row.purchaseAmount : row.salePrice)).toLocaleString() }}</template></el-table-column></el-table></article>
          <article class="section-card contract-detail-info contract-content-group contract-readonly-section"><SectionTitle number="04" title="合同内容" /><h4 class="contract-content-subtitle">合同信息</h4><el-form label-position="top" disabled><el-row :gutter="16"><el-col :span="24" class="contract-field-group-title">合同生成方式</el-col><el-col :span="8"><el-form-item label="合同情况"><el-input :model-value="contractModeLabel(activeContractPage.data.contractMode)" /></el-form-item></el-col><template v-if="activeContractPage.data.contractMode === 'framework'"><el-col :span="8"><el-form-item label="框架协议编号"><el-input :model-value="activeContractPage.data.frameworkCode || 'KJXY-2026-0086'" /></el-form-item></el-col><el-col :span="8"><el-form-item label="框架协议附件"><div class="readonly-link-field"><el-link type="primary">框架协议正文.pdf</el-link><span>↗</span></div></el-form-item></el-col></template><template v-else><el-col :span="24" class="contract-field-group-title">合同金额与结算</el-col><el-col :span="8"><el-form-item label="合同税率情况"><el-input model-value="13%" /></el-form-item></el-col><el-col :span="8"><el-form-item :label="activeContractPage.data.type === 'purchase' ? '付款方式' : '回款方式'"><el-input model-value="货到付款" /></el-form-item></el-col><el-col :span="8"><el-form-item label="合同总金额（含税）"><el-input :model-value="activeContractPage.data.amount" /></el-form-item></el-col><el-col :span="8"><el-form-item label="合同总金额（不含税）"><el-input model-value="¥336,283.19" /></el-form-item></el-col><el-col :span="8"><el-form-item label="合同税额"><el-input model-value="¥43,716.81" /></el-form-item></el-col><el-col :span="8"><el-form-item :label="activeContractPage.data.type === 'purchase' ? '采购账期' : '销售账期'"><el-input model-value="30 天" /></el-form-item></el-col><el-col :span="8"><el-form-item :label="activeContractPage.data.type === 'purchase' ? '预计付款日期' : '预计回款日期'"><el-input model-value="2026-09-20" /></el-form-item></el-col><el-col :span="8"><el-form-item :label="activeContractPage.data.type === 'purchase' ? '货款结算' : '回款约定'"><el-input model-value="到货验收后30个工作日内支付100%货款" /></el-form-item></el-col></template></el-row></el-form><el-collapse v-if="activeContractPage.data.contractMode !== 'framework'" class="secondary-contract-info"><el-collapse-item title="交付、履约与用印"><el-form label-position="top" disabled><el-row :gutter="16"><el-col :span="24" class="contract-field-group-title">交付与履约</el-col><el-col :span="8"><el-form-item label="签约地点"><el-input model-value="成都" /></el-form-item></el-col><el-col :span="8"><el-form-item label="技术服务"><el-input model-value="提供安装调试及技术支持" /></el-form-item></el-col><template v-if="activeContractPage.data.type === 'purchase' && activeContractPage.data.contractMode === 'generated'"><el-col :span="8"><el-form-item label="产品交付"><el-input model-value="7日" /></el-form-item></el-col><el-col :span="8"><el-form-item label="收货地址"><el-input model-value="成都市高新区天府大道" /></el-form-item></el-col><el-col :span="8"><el-form-item label="联系人"><el-input model-value="王经理" /></el-form-item></el-col><el-col :span="8"><el-form-item label="联系电话"><el-input model-value="13800000000" /></el-form-item></el-col></template><template v-if="activeContractPage.data.type === 'sale' && activeContractPage.data.contractMode === 'generated'"><el-col :span="8"><el-form-item label="开票模式"><el-input model-value="先票后款" /></el-form-item></el-col><el-col :span="8"><el-form-item label="开票天数"><el-input model-value="30" /></el-form-item></el-col><el-col :span="8"><el-form-item label="发票类型"><el-input model-value="增值税专用发票" /></el-form-item></el-col><el-col :span="8"><el-form-item label="交付约定"><el-input model-value="合同生效后7个工作日内交付" /></el-form-item></el-col><el-col :span="8"><el-form-item label="交付要求"><el-input model-value="按客户指定地址完成交付" /></el-form-item></el-col><el-col :span="8"><el-form-item label="技术支持及服务"><el-input model-value="需要" /></el-form-item></el-col><el-col :span="8"><el-form-item label="产品售后"><el-input model-value="按原厂售后标准执行" /></el-form-item></el-col><el-col :span="8"><el-form-item label="质保条件"><el-input model-value="原厂质保一年" /></el-form-item></el-col></template><el-col :span="24" class="contract-field-group-title">用印信息</el-col><el-col :span="8"><el-form-item label="用印方式"><el-input model-value="电子用印" /></el-form-item></el-col><el-col :span="8"><el-form-item label="用印情况"><el-input model-value="双方用印" /></el-form-item></el-col><el-col :span="8"><el-form-item label="印章需求"><el-input model-value="合同专用章" /></el-form-item></el-col><el-col :span="8"><el-form-item label="审批后自动用印"><el-input model-value="是" /></el-form-item></el-col><el-col :span="8"><el-form-item label="是否需要邮寄"><el-input model-value="否" /></el-form-item></el-col><el-col :span="24"><el-form-item label="备注"><el-input model-value="以最终用印文件为准" type="textarea" :rows="2" /></el-form-item></el-col></el-row></el-form></el-collapse-item></el-collapse></article><article class="section-card contract-readonly-section contract-file-detail-section"><SectionTitle number="05" :title="activeContractPage.data.contractMode === 'framework' ? '订单证明文件' : '合同文件'" /><div v-if="activeContractPage.data.status === '审批中'" class="contract-local-status"><InfoFilled /><span>审批时优先核对AI识别变更内容和印章配置结果。</span></div><el-table :data="contractFiles" border size="small"><el-table-column label="文件名" min-width="210"><template #default="{ row }"><el-link type="primary" @click="previewContractFile(row)">{{ row.name }}</el-link><el-dropdown trigger="click"><el-button link type="primary" class="file-download-trigger"><ArrowDown /></el-button><template #dropdown><el-dropdown-menu><el-dropdown-item>预览文件</el-dropdown-item><el-dropdown-item>下载文件</el-dropdown-item></el-dropdown-menu></template></el-dropdown></template></el-table-column><el-table-column v-if="activeContractPage.data.contractMode !== 'framework'" prop="companyTemplate" label="是否公司模板" width="125"><template #default="{ row }"><el-tag :type="row.companyTemplate === '是' ? 'success' : row.companyTemplate === '识别中' ? 'warning' : 'info'">{{ row.companyTemplate }}</el-tag></template></el-table-column><el-table-column v-if="activeContractPage.data.contractMode !== 'framework'" prop="sealConfigured" label="是否配置印章" width="125"><template #default="{ row }"><el-tag :type="row.sealConfigured ? 'success' : 'info'">{{ row.sealConfigured ? '是' : '否' }}</el-tag></template></el-table-column><el-table-column v-if="activeContractPage.data.contractMode !== 'framework'" prop="sealCount" label="配置印章数量" width="125" align="center" /><el-table-column prop="history" label="历史附件" width="95" /><el-table-column v-if="activeContractPage.data.contractMode !== 'framework'" prop="aiChange" label="AI识别文件变更内容" min-width="210" show-overflow-tooltip /><el-table-column v-if="activeContractPage.data.contractMode !== 'framework'" prop="counterpartySealed" label="对方已用印" width="105" /></el-table></article>
          <article class="section-card contract-readonly-section contract-file-supplement-section"><SectionTitle number="06" title="签收文件与归档信息" /><h4 class="contract-content-subtitle">签收文件</h4><el-table :data="contractSignFiles" border size="small"><el-table-column prop="name" label="文件名" min-width="220"><template #default="{ row }"><el-link type="primary">{{ row.name }}</el-link></template></el-table-column><el-table-column prop="uploader" label="上传人" width="120" /><el-table-column prop="uploadedAt" label="上传时间" width="170" /></el-table><h4 class="contract-content-subtitle detail-file-subtitle">归档信息</h4><el-table :data="contractArchiveRows" border size="small"><el-table-column prop="type" label="归档类型" min-width="170" /><el-table-column prop="status" label="归档状态" width="120"><template #default="{ row }"><el-tag :type="row.status === '已归档' ? 'success' : 'info'">{{ row.status }}</el-tag></template></el-table-column><el-table-column prop="operator" label="归档人" width="120" /><el-table-column prop="date" label="归档时间" width="170" /></el-table></article>
          <article v-if="activeContractPage.data.type === 'purchase'" class="section-card contract-readonly-section"><SectionTitle number="07" title="后续任务" /><el-tabs model-value="purchaseOrder"><el-tab-pane :label="`创建采购订单（${purchaseOrderTasks.length}）`" name="purchaseOrder"><el-table :data="purchaseOrderTasks" border size="small"><el-table-column prop="status" label="任务状态" width="100" /><el-table-column prop="orderCode" label="订单号（执行后才有）" min-width="175" /><el-table-column prop="spu" label="采购SPU" min-width="170" /><el-table-column prop="sku" label="采购SKU" min-width="220" /><el-table-column prop="price" label="单台采购价" width="115" align="right" /><el-table-column prop="quantity" label="采购数量" width="100" align="right" /><el-table-column prop="warehouse" label="入库仓库" min-width="140" /></el-table></el-tab-pane></el-tabs></article>
          <article class="section-card"><SectionTitle :number="activeContractPage.data.type === 'purchase' ? '08' : '07'" title="审批信息" /><el-empty v-if="activeContractPage.data.status === '待提交'" description="合同提交后展示审批进度与审批评论" :image-size="60" /><template v-else><div class="flat-approval-list"><div v-for="node in flatApprovalNodes.slice(0, 4)" :key="node.order" :class="['flat-node', node.status]"><span class="flat-order">{{ node.order }}</span><div class="flat-node-main"><strong>{{ node.name }}</strong><span>{{ node.approver }}</span></div><el-tag :type="approvalStatusType(node.status)">{{ approvalStatusLabel(node.status) }}</el-tag></div></div><el-descriptions class="approval-extra-info" :column="3" border><el-descriptions-item label="审批备注">合同条款及金额已核对</el-descriptions-item><el-descriptions-item label="取消原因">-</el-descriptions-item><el-descriptions-item label="备注">以最终用印文件为准</el-descriptions-item></el-descriptions></template></article>
          <footer v-if="activeContractPage.data.status === '审批中'" class="bottom-actions approval-bottom-actions"><el-button type="danger" plain>驳回</el-button><el-button type="primary">通过</el-button></footer>
        </div>
        <aside class="contract-side-directory"><section class="flow-navigation-card"><div class="flow-control-toggle"><List /><span>模块导航</span></div><button v-for="(item,index) in contractDetailModules" :key="item.label" :class="['flow-control-item',{active:index===currentContractDetailModule}]" @click="jumpToContractDetailModule(index)"><span>{{ item.order }}</span><b>{{ item.label }}</b><em :class="item.status === '审批中' ? 'warning' : 'done'">{{ item.status }}</em></button></section><div class="flow-status-reminders contract-status-reminders"><div><b>状态提醒</b><span>{{ activeContractPage.data.status === '审批中' ? '2项' : '1项' }}</span></div><button v-if="activeContractPage.data.status === '审批中'">AI识别结果及印章配置待核对</button><button v-if="activeContractPage.data.type === 'purchase'">采购订单任务执行状态待关注</button></div></aside>
      </section>

      <section v-else-if="activeView !== 'contract'" :class="['document-page', { 'budget-create-page': isEditing, 'budget-edit-page': mode === 'edit', 'budget-readonly-page': !isEditing, 'budget-detail-view': mode === 'detail', 'budget-approval-view': mode === 'audit', 'budget-flow-template': prototypeVariant === 'flow-template' }]">
        <header :class="['document-header', { 'approval-document-header': mode === 'audit' }]">
          <template v-if="mode === 'audit'">
            <div class="budget-approval-header-content">
              <div class="eyebrow">我的审批 · 预算审批</div>
              <div class="approval-header-title"><h1>分销产品预算审批</h1><el-tag type="warning" round>审批中</el-tag></div>
              <div class="approval-header-body">
                <div class="approval-summary-identity"><div><el-tag effect="plain">{{ documentData.type }}</el-tag><span class="approval-summary-code">{{ documentData.code }}</span></div><p>供应商：<strong>{{ documentData.supplier }}</strong><i></i>采销责任人：<strong>{{ documentData.owner }}</strong></p></div>
                <div class="approval-summary-metrics"><div><span>预计销售收入</span><strong>{{ documentData.revenue || '¥328,000.00' }}</strong></div><div><span>预算单毛利</span><strong class="positive">¥57,000.00</strong></div><div><span>预算单毛利率</span><strong class="positive">{{ documentData.margin || '18.42%' }}</strong></div></div>
              </div>
            </div>
          </template>
          <template v-else>
            <div class="title-block">
              <div class="eyebrow">{{ modeLabel }} · {{ documentData.type }}</div>
              <h1>{{ documentTitle }} <el-tag v-if="mode === 'detail'" type="success">审批完成</el-tag></h1>
              <p v-if="mode === 'detail'" class="document-meta"><span>预算单号：{{ documentData.code }}</span><span>业务类型：{{ documentData.type }}</span><span>采销责任人：{{ documentData.owner }}</span><span>制单时间：{{ documentData.created }}</span><span>最后更新：2026-08-24 16:08</span></p>
            </div>
            <div v-if="isEditing" class="budget-create-header-actions"><el-button @click="requestClose(activeTab)">取消</el-button><el-button @click="saveDraft">保存草稿</el-button><el-button type="primary" @click="submitDocument">保存并提交</el-button></div>
            <div v-else-if="mode === 'detail'" class="budget-create-header-actions"><el-button :icon="Printer">打印</el-button><el-button type="primary" @click="changeMode('edit')">编辑</el-button><el-dropdown><el-button>更多<ArrowDown /></el-button><template #dropdown><el-dropdown-menu><el-dropdown-item>创建采购合同</el-dropdown-item><el-dropdown-item>创建销售合同</el-dropdown-item></el-dropdown-menu></template></el-dropdown></div>
          </template>
        </header>

        <div class="document-layout">
          <div ref="scrollArea" class="document-scroll" @scroll="updateBudgetCurrentModule">
            <div v-if="mode === 'edit'" class="edit-scope-panel"><strong>本次可修改范围</strong><span>白色输入框为可编辑字段</span><span>灰色字段由系统或关联单据自动带出，不可修改</span></div>

            <article id="basic" class="section-card">
              <SectionTitle :number="isEditing ? '1' : '01'" title="基础信息" />
              <el-form :model="form" label-position="top" :disabled="!isEditing">
                <h3 class="budget-subsection-title">业务归属</h3>
                <el-row :gutter="20" class="business-affiliation-row">
                  <el-col v-if="mode === 'edit'" :span="8"><el-form-item label="预算单编号"><el-input :model-value="documentData.code" disabled /></el-form-item></el-col>
                  <el-col :span="6"><el-form-item label="业务类型" required><el-select v-model="form.type" @change="handleBusinessTypeChange"><el-option v-for="type in businessTypes" :key="type" :label="type" :value="type" /></el-select></el-form-item></el-col>
                  <el-col :span="6"><el-form-item label="合同签署主体" required><el-select v-model="form.entity"><el-option label="四川科瑞供应链管理有限公司" value="四川科瑞供应链管理有限公司" /></el-select></el-form-item></el-col>
                  <el-col v-if="businessUnitField" :span="6"><el-form-item label="业务单元" required><el-select v-model="form.dynamic.businessUnit" placeholder="请选择业务单元"><el-option v-for="option in businessUnitField.options" :key="option" :label="option" :value="option" /></el-select></el-form-item></el-col>
                  <el-col :span="6"><el-form-item :label="isEditing ? '预算有效期（天）' : '预算有效期'" required><el-input-number v-if="isEditing" v-model="form.validDays" :min="1" /><div v-else class="detail-result"><strong>{{ form.validDays || '—' }}</strong><span v-if="form.validDays">天</span></div><div v-if="isEditing" class="field-tip"><InfoFilled />提示：需要在预算审批通过后{{ form.validDays }}天内完成销售、采购订单的创建</div></el-form-item></el-col>
                </el-row>
                <div class="type-fields">
                  <el-row :gutter="20">
                    <el-col v-for="field in basicTypeFields" :key="field.key" :span="8">
                      <el-form-item :label="field.label" :required="field.required">
                        <el-select v-if="field.kind === 'select'" v-model="form.dynamic[field.key]" :placeholder="`请选择${field.label}`"><el-option v-for="option in field.options" :key="option" :label="option" :value="option" /></el-select>
                        <el-input-number v-else-if="field.kind === 'number'" v-model="form.dynamic[field.key]" :min="0" />
                        <el-switch v-else-if="field.kind === 'switch'" v-model="form.dynamic[field.key]" inline-prompt active-text="是" inactive-text="否" />
                        <el-input v-else v-model="form.dynamic[field.key]" :placeholder="`请输入${field.label}`" />
                      </el-form-item>
                    </el-col>
                  </el-row>
                </div>
                <h3 class="budget-subsection-title transaction-title">交易对象</h3>
                <el-row :gutter="20">
                  <el-col :span="8"><el-form-item label="供应商" required><el-select v-if="isEditing" v-model="form.supplier"><el-option label="成都星海科技有限公司" value="成都星海科技有限公司" /></el-select><el-link v-else class="detail-entity-link" type="primary">{{ form.supplier || '—' }}</el-link></el-form-item></el-col>
                  <el-col :span="8"><el-form-item label="采购责任人" required><el-select v-model="form.owner"><el-option label="张晨" value="张晨" /></el-select></el-form-item></el-col>
                </el-row>
              </el-form>
            </article>

            <article id="plan" class="section-card">
              <SectionTitle :number="isEditing ? '2' : '02'" title="预算计划 / 收益测算" />
              <div class="split-panel">
                <div class="plan-form"><h3>预算计划</h3>
                  <el-form :model="form" label-position="top" :disabled="!isEditing">
                    <el-row :gutter="16">
                      <el-col :span="12"><el-form-item label="预计到货日期" required><el-date-picker v-if="isEditing" v-model="form.arrival" type="date" value-format="YYYY-MM-DD" /><div v-else class="detail-result detail-date"><span class="detail-date-mark">日</span><strong>{{ form.arrival || '—' }}</strong></div></el-form-item></el-col>
                      <el-col :span="12"><el-form-item label="预计付款日期" required><el-date-picker v-if="isEditing" v-model="form.payment" type="date" value-format="YYYY-MM-DD" /><div v-else class="detail-result detail-date"><span class="detail-date-mark">日</span><strong>{{ form.payment || '—' }}</strong></div></el-form-item></el-col>
                      <el-col :span="12"><el-form-item :label="isEditing ? '采购账期（天）' : '采购账期'" required><el-input-number v-if="isEditing" v-model="form.purchaseDays" :min="0" /><div v-else class="detail-result detail-term-result"><strong>{{ form.purchaseDays ?? '—' }}</strong><span v-if="form.purchaseDays !== null && form.purchaseDays !== undefined">天</span></div></el-form-item></el-col>
                      <el-col :span="12"><el-form-item :label="isEditing ? '最长销售周期（天）' : '最长销售周期'" required><el-input-number v-if="isEditing" v-model="form.saleDays" :min="1" /><div v-else class="detail-result"><strong>{{ form.saleDays || '—' }}</strong><span v-if="form.saleDays">天</span></div></el-form-item></el-col>
                      <el-col v-if="form.type === '产品导向分销'" :span="12"><el-form-item label="市场环境说明" required><el-select v-model="form.dynamic.marketEnvironment"><el-option v-for="option in ['价格稳定','价格上涨','价格下行']" :key="option" :label="option" :value="option" /></el-select></el-form-item></el-col>
                      <el-col v-if="form.type === '产品导向分销'" :span="12"><el-form-item label="项目单毛利回补总额"><el-input-number v-if="isEditing" v-model="form.dynamic.profitBackfill" :min="0" /><div v-else class="detail-result detail-money"><strong>¥{{ Number(form.dynamic.profitBackfill || 0).toLocaleString('zh-CN', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></div></el-form-item></el-col>
                      <el-col :span="24"><el-form-item label="利润说明"><el-input v-model="form.profitDesc" type="textarea" :rows="3" maxlength="500" show-word-limit placeholder="请输入利润说明（选填）" /></el-form-item></el-col>
                    </el-row>
                  </el-form>
                </div>
                <div class="metrics"><h3>收益测算 <small>根据商品明细自动计算</small></h3>
                  <div class="metric-grid"><div v-for="metric in metrics" :key="metric.label" :class="{ warning: metric.warning }"><span class="metric-label">{{ metric.label }}<el-tooltip :content="metric.rule" placement="top"><QuestionFilled /></el-tooltip></span><strong>{{ metric.value }}</strong></div></div>
                </div>
              </div>
            </article>

            <article id="goods" class="section-card wide-card">
              <SectionTitle :number="isEditing ? '3' : '03'" title="采销明细">
                <template v-if="isEditing">
                  <div class="goods-actions" style="display:inline-flex;flex-direction:row;flex-wrap:nowrap;align-items:center;gap:8px;width:auto;white-space:nowrap">
                    <el-button type="primary" :icon="Plus" @click="openProductDialog">添加商品</el-button>
                    <el-upload style="display:inline-flex;width:auto;flex:none" action="#" :auto-upload="false" :show-file-list="false" accept=".xls,.xlsx" :on-change="handleBatchImport">
                      <el-button :icon="Upload">导入商品</el-button>
                    </el-upload>
                  </div>
                </template>
              </SectionTitle>
              <el-table :data="goods" border>
                <el-table-column v-if="isEditing" type="selection" width="50" />
                <el-table-column type="index" label="序号" width="58" fixed="left" />
                <el-table-column prop="skuCode" label="SKU编码" width="170" fixed="left"><template #default="{ row }"><el-input v-if="isEditing" v-model="row.skuCode" placeholder="请输入SKU编码" /><span v-else>{{ row.skuCode }}</span></template></el-table-column>
                <el-table-column prop="spuName" label="SPU名称" min-width="170" show-overflow-tooltip />
                <el-table-column prop="skuName" label="SKU名称" min-width="230" show-overflow-tooltip />
                <el-table-column prop="businessUnit" label="业务单元" min-width="140" show-overflow-tooltip />
                <el-table-column label="采购数量" width="125"><template #default="{ row }"><el-input-number v-if="isEditing" v-model="row.purchaseQuantity" :min="0" controls-position="right" /><span v-else>{{ row.purchaseQuantity }}</span></template></el-table-column>
                <el-table-column label="单台采购价" width="145"><template #default="{ row }"><el-input-number v-if="isEditing" v-model="row.purchaseAmount" :min="0" :precision="2" controls-position="right" /><span v-else>¥{{ row.purchaseAmount.toLocaleString() }}</span></template></el-table-column>
                <el-table-column label="单台销售价" width="145"><template #default="{ row }"><el-input-number v-if="isEditing" v-model="row.salePrice" :min="0" :precision="2" controls-position="right" /><span v-else>¥{{ row.salePrice.toLocaleString() }}</span></template></el-table-column>
                <el-table-column label="预计销售周期" width="130"><template #default="{ row }"><el-input-number v-if="isEditing" v-model="row.saleCycle" :min="0" controls-position="right" /><span v-else>{{ row.saleCycle }}</span></template></el-table-column>
                <el-table-column v-if="isEditing" label="操作" width="90" fixed="right"><template #default="{ $index }"><el-button type="danger" link @click="removeGoods($index)">删除</el-button></template></el-table-column>
              </el-table>
            </article>

            <article v-if="mode === 'detail'" id="related" class="section-card">
              <SectionTitle number="04" title="关联单据" />
              <el-tabs v-model="relatedTab"><el-tab-pane label="销售订单（2）" name="sale"><RelatedTable type="销售订单" /></el-tab-pane><el-tab-pane label="采购订单（1）" name="purchase"><RelatedTable type="采购订单" /></el-tab-pane><el-tab-pane label="合同（1）" name="contract"><RelatedTable type="合同" /></el-tab-pane></el-tabs>
            </article>

            <article id="attachments" class="section-card">
              <SectionTitle :number="isEditing ? '4' : '05'" title="附件与说明" />
              <div v-if="prototypeVariant === 'flow-template'" class="auxiliary-module-control"><div><b>辅助信息</b><span>附件最多5个，补充说明选填</span></div><el-button link type="primary" @click="attachmentsExpanded = !attachmentsExpanded">{{ attachmentsExpanded ? '收起' : '展开填写' }}<ArrowDown :class="{ rotated: attachmentsExpanded }" /></el-button></div>
              <div v-show="prototypeVariant !== 'flow-template' || attachmentsExpanded" class="attachment-layout">
                <div class="attachment-column">
                  <h3>附件</h3>
                  <el-upload
                    v-if="isEditing"
                    drag
                    action="#"
                    :auto-upload="false"
                    multiple
                    :limit="5"
                    :on-change="handleAttachmentChange"
                    accept=".jpg,.jpeg,.png,.pdf,.xls,.xlsx"
                  >
                    <div class="upload-action"><Plus /><b>上传附件</b></div>
                    <div class="upload-help">支持 .jpg、.png、.pdf、.xls、.xlsx 等，最多5个，单个文件不超过20MB</div>
                  </el-upload>
                  <div v-else class="file-row"><Document /><div><b>渠道报价与销售预测.xlsx</b><span>1.8 MB · 李然上传</span></div><el-button link type="primary">预览</el-button><el-button link>下载</el-button></div>
                </div>
                <div class="note-column">
                  <h3>补充说明</h3>
                  <el-input v-model="form.supplement" type="textarea" :rows="5" maxlength="500" show-word-limit :disabled="!isEditing" placeholder="请输入补充说明（选填）" />
                </div>
              </div>
            </article>

            <article v-if="mode === 'detail' && documentData.version" id="change-records" class="section-card">
              <SectionTitle number="06" title="变更记录" />
              <el-collapse><el-collapse-item :title="`第 ${documentData.version} 次修改 · 审批完成`"><el-descriptions :column="3" border><el-descriptions-item label="修改人">周敏</el-descriptions-item><el-descriptions-item label="修改时间">2026-08-18 09:20</el-descriptions-item><el-descriptions-item label="变更内容">单据信息 2 项、商品信息 1 项</el-descriptions-item></el-descriptions></el-collapse-item></el-collapse>
            </article>

            <article v-if="mode === 'detail'" id="approval-info" class="section-card detail-approval-info">
              <SectionTitle number="07" title="审批信息" />
              <div class="flat-approval-list">
                <div v-for="node in flatApprovalNodes.slice(0, 4)" :key="node.order" :class="['flat-node', node.status]">
                  <span class="flat-order">{{ node.order }}</span>
                  <div class="flat-node-main"><strong>{{ node.name }}</strong><span>{{ node.approver }}</span></div>
                  <div class="flat-node-result"><small v-if="node.time">{{ node.time }}</small><el-tag size="small" :type="approvalStatusType(node.status)">{{ approvalStatusLabel(node.status) }}</el-tag></div>
                </div>
              </div>
              <el-descriptions class="approval-extra-info" :column="3" border>
                <el-descriptions-item label="审批结果"><el-tag type="success">审批完成</el-tag></el-descriptions-item>
                <el-descriptions-item label="审批完成时间">2026-08-21 16:08</el-descriptions-item>
                <el-descriptions-item label="审批意见">预算金额及商品信息已核对</el-descriptions-item>
              </el-descriptions>
            </article>

            <article v-if="mode === 'audit'" id="approval-progress" class="section-card">
              <SectionTitle number="06" title="审批进度" />
              <div class="current-task-card">
                <div class="task-state-icon"><Clock /></div>
                <div class="task-main"><span>当前审批任务</span><strong>财务BP审批</strong><small>审批人：当前用户 · 财务BP</small></div>
                <div class="task-stat"><span>节点进度</span><strong>4 / 7</strong><el-progress :percentage="57" :show-text="false" /></div>
                <el-tag type="warning" effect="dark">等待你审批</el-tag>
              </div>
              <div class="flat-approval-list">
                <div v-for="node in flatApprovalNodes" :key="node.order" :class="['flat-node', node.status]">
                  <div class="flat-order"><CircleCheckFilled v-if="node.status === 'done'" /><Clock v-else-if="node.status === 'active'" /><span v-else>{{ node.order }}</span></div>
                  <div class="flat-node-main"><strong>{{ node.name }}</strong><span>{{ node.approver }}</span></div>
                  <div class="flat-node-result"><small v-if="node.time">{{ node.time }}</small><el-tag size="small" :type="approvalStatusType(node.status)">{{ approvalStatusLabel(node.status) }}</el-tag></div>
                </div>
              </div>
            </article>

            <article v-if="mode === 'audit'" id="approval-comments" class="section-card">
              <SectionTitle number="07" title="审批评论" />
              <div class="approval-timeline">
                <div v-for="record in approvalComments" :key="record.time + record.user" class="timeline-item">
                  <div class="timeline-dot"></div>
                  <div class="comment-avatar">{{ record.user.slice(0, 1) }}</div>
                  <div class="timeline-content"><div><b>{{ record.user }}</b><el-tag size="small" :type="record.tagType">{{ record.action }}</el-tag><span>{{ record.time }}</span></div><p>{{ record.content }}</p></div>
                </div>
              </div>
              <div class="comment-title">本次审批意见</div>
              <el-input v-model="approvalComment" type="textarea" :rows="3" maxlength="300" show-word-limit placeholder="添加审批意见（选填）" />
            </article>

            <footer class="bottom-actions">
              <el-button v-if="isEditing" @click="requestClose(activeTab)">取消</el-button>
              <el-button v-if="isEditing" @click="saveDraft">保存草稿</el-button>
              <el-button v-if="isEditing" type="primary" @click="submitDocument">保存并提交</el-button>
              <el-button v-if="mode === 'audit'" type="danger" plain @click="auditDialog = 'reject'">驳回</el-button>
              <el-button v-if="mode === 'audit'" type="primary" @click="auditDialog = 'approve'">通过</el-button>
            </footer>
          </div>
          <aside :class="['flow-module-control', { collapsed: flowControlCollapsed }]">
            <template v-if="!flowControlCollapsed">
              <section v-if="isEditing" class="flow-revenue-summary"><header><strong>收益测算</strong><span>关键指标</span></header><div v-for="metric in metrics.slice(0, 4)" :key="metric.label"><span>{{ metric.label }}</span><b>{{ metric.value }}</b></div><button @click="jumpFromFlowControl('plan')">查看完整测算 →</button></section>
              <section class="flow-navigation-card"><button class="flow-control-toggle" @click="flowControlCollapsed = !flowControlCollapsed"><List /><span>模块导航</span><ArrowDown class="rotated" /></button><button v-for="item in budgetModules" :key="item.id" :class="['flow-control-item', { active: activeSection === item.id }]" @click="jumpFromFlowControl(item.id)"><span>{{ item.order }}</span><b>{{ item.label }}</b><em :class="budgetModuleStatus(item).className">{{ budgetModuleStatus(item).label }}</em></button></section>
              <section class="flow-status-reminders"><div><strong>状态提醒</strong><span>2项</span></div><template v-if="mode === 'detail'"><button @click="jumpFromFlowControl('plan')">预计月均毛利率低于建议值</button><button @click="jumpFromFlowControl('related')">1 条采购订单执行中</button></template><template v-else-if="mode === 'audit'"><button @click="jumpFromFlowControl('plan')">预计月均毛利率低于建议值</button><button @click="jumpFromFlowControl('goods')">1 项商品数量与价格需复核</button></template><template v-else><button @click="jumpFromFlowControl('goods')">商品库存覆盖天数需关注</button><button @click="jumpFromFlowControl('attachments')">附件尚未上传</button></template></section>
            </template>
            <button v-else class="flow-control-toggle collapsed-toggle" @click="flowControlCollapsed = false"><List /><ArrowDown /></button>
          </aside>
        </div>
      </section>
    </main>

    <el-dialog v-model="showCloseConfirm" title="离开此页面？" width="420px">
      <p>当前内容尚未保存，离开后本次修改将不会保留。</p>
      <template #footer><el-button @click="showCloseConfirm = false">继续编辑</el-button><el-button type="danger" plain @click="confirmClose">放弃修改</el-button><el-button type="primary" @click="saveThenClose">保存草稿并离开</el-button></template>
    </el-dialog>
    <el-dialog v-model="showAuditDialog" :title="auditDialog === 'approve' ? '确认通过审批' : '确认驳回申请'" width="440px">
      <el-alert :type="auditDialog === 'approve' ? 'success' : 'warning'" :closable="false" :title="auditDialog === 'approve' ? '通过后将流转至下一审批节点' : '驳回后单据将退回申请人修改'" />
      <el-input v-model="approvalComment" type="textarea" :rows="4" class="dialog-comment" :placeholder="auditDialog === 'reject' ? '请输入驳回原因（必填）' : '审批意见（选填）'" />
      <template #footer><el-button @click="auditDialog = ''">取消</el-button><el-button :type="auditDialog === 'approve' ? 'primary' : 'danger'" @click="completeAudit">确认{{ auditDialog === 'approve' ? '通过' : '驳回' }}</el-button></template>
    </el-dialog>
    <el-dialog v-model="productDialogVisible" title="选择商品" width="1080px" top="7vh" :close-on-click-modal="false">
      <div class="product-dialog-toolbar">
        <el-input v-model="productFilters.keyword" clearable :prefix-icon="Search" placeholder="SKU编码 / SKU名称 / SPU名称" />
        <el-select v-model="productFilters.businessUnit" clearable placeholder="全部业务单元"><el-option label="西南业务部" value="西南业务部" /><el-option label="华东业务部" value="华东业务部" /></el-select>
        <el-checkbox v-model="productFilters.inStockOnly">仅看有库存</el-checkbox>
      </div>
      <el-table ref="productTableRef" :data="filteredProductOptions" border height="430" row-key="skuCode" @selection-change="productSelection = $event">
        <el-table-column type="selection" width="50" :selectable="isProductSelectable" reserve-selection />
        <el-table-column prop="spuName" label="SPU名称" min-width="170" show-overflow-tooltip />
        <el-table-column prop="skuCode" label="SKU编码" min-width="150" />
        <el-table-column prop="skuName" label="SKU名称" min-width="220" show-overflow-tooltip />
        <el-table-column prop="specModel" label="配置说明" min-width="160" show-overflow-tooltip />
        <el-table-column prop="businessUnit" label="业务单元" width="120" />
        <el-table-column prop="stock" label="当前库存" width="100" align="right" />
        <el-table-column label="状态" width="100"><template #default="{ row }"><el-tag v-if="isProductAdded(row)" type="info">已添加</el-tag><el-tag v-else-if="row.stock > 0" type="success">可选择</el-tag><el-tag v-else type="warning">无库存</el-tag></template></el-table-column>
      </el-table>
      <div class="product-selection-summary">已选择 <b>{{ selectableProductSelection.length }}</b> 项<span>已加入采销明细的商品不可重复选择</span></div>
      <template #footer><el-button @click="productDialogVisible = false">取消</el-button><el-button type="primary" :disabled="!selectableProductSelection.length" @click="confirmProductSelection">确认添加（{{ selectableProductSelection.length }}）</el-button></template>
    </el-dialog>
    <el-dialog v-model="advancedSearchVisible" title="高级搜索" width="720px">
      <el-form :model="advancedFilters" label-position="top"><el-row :gutter="18"><el-col :span="12"><el-form-item label="预算单编号"><el-input v-model="advancedFilters.code" clearable /></el-form-item></el-col><el-col :span="12"><el-form-item label="采购/销售责任人"><el-input v-model="advancedFilters.owner" clearable /></el-form-item></el-col><el-col :span="12"><el-form-item label="供应商/客户"><el-input v-model="advancedFilters.supplier" clearable /></el-form-item></el-col><el-col :span="12"><el-form-item label="业务类型"><el-select v-model="advancedFilters.type" clearable><el-option v-for="type in businessTypes" :key="type" :label="type" :value="type" /></el-select></el-form-item></el-col><el-col :span="12"><el-form-item label="审批状态"><el-select v-model="advancedFilters.status" clearable><el-option v-for="status in ['审批中','审批完成','草稿']" :key="status" :label="status" :value="status" /></el-select></el-form-item></el-col><el-col :span="12"><el-form-item label="创建时间"><el-date-picker v-model="advancedFilters.dateRange" type="daterange" value-format="YYYY-MM-DD" /></el-form-item></el-col></el-row></el-form>
      <template #footer><el-button @click="resetAdvancedSearch">重置</el-button><el-button type="primary" @click="applyAdvancedSearch">查询</el-button></template>
    </el-dialog>
    <el-dialog v-model="modificationVisible" title="预算修改记录" width="900px"><div class="record-budget-code">预算单编号：{{ modificationBudget?.code }}</div><el-table :data="modificationRecords" border><el-table-column prop="version" label="版本" width="80" /><el-table-column prop="field" label="修改字段" width="150" /><el-table-column prop="before" label="修改前" min-width="170" /><el-table-column prop="after" label="修改后" min-width="170" /><el-table-column prop="user" label="修改人" width="100" /><el-table-column prop="time" label="修改时间" width="170" /></el-table></el-dialog>
    <el-dialog v-model="pendingContractVisible" title="待提交合同" width="900px"><el-table :data="pendingContracts" border><el-table-column prop="budgetCode" label="来源预算单" width="180" /><el-table-column prop="contractType" label="合同类型" width="120" /><el-table-column prop="enterprise" label="供应商/客户" min-width="190" /><el-table-column prop="created" label="暂存时间" width="170" /><el-table-column label="操作" width="100"><template #default="{ row }"><el-button link type="primary" @click="continueContract(row)">继续填写</el-button></template></el-table-column></el-table></el-dialog>
    <Teleport to="body"><section v-if="activeContractEditPage" :class="['document-page', 'contract-edit-page', `mode-${contractDraft.contractMode}`, `type-${contractDraft.type}`]">
      <header class="document-header"><div class="title-block"><div class="eyebrow">合同管理 · {{ contractDraft.typeLabel }}</div><h1>{{ contractPageMode === 'edit' ? `编辑${contractDraft.typeLabel}` : `创建${contractDraft.typeLabel}` }}</h1></div><div class="budget-create-header-actions"><el-button @click="requestClose(activeTab)">取消</el-button><el-button @click="savePendingContract">保存为草稿</el-button><el-button type="primary" @click="submitContract">保存并提交</el-button></div></header>
      <div ref="contractScrollArea" class="document-scroll" @scroll="updateContractCurrentModule">
        <div v-if="contractPageMode === 'edit'" class="edit-scope-panel contract-edit-scope"><strong>本次可修改范围</strong><span>采购清单本次数量、合同字段和合同文件可维护</span><span>供应商、业务类型、签署主体等关联字段自动带出</span></div>
        <div class="contract-prototype-content">
        <section id="contract-basic-section" data-contract-module="contract-basic-section" class="contract-block"><h3><b>01</b>基础信息</h3><el-form label-position="top"><el-row :gutter="16">
          <el-col :span="8"><el-form-item label="合同类型"><el-input :model-value="contractDraft.typeLabel" disabled /></el-form-item></el-col><el-col :span="8"><el-form-item :label="contractDraft.type === 'purchase' ? '供应商' : '客户'" required><div class="readonly-link-field"><el-link type="primary" @click="openSupplierDetail(contractDraft.budget?.supplier)">{{ contractDraft.budget?.supplier }}</el-link><ArrowDown /></div></el-form-item></el-col><el-col :span="8"><el-form-item :label="contractDraft.type === 'purchase' ? '采购责任人' : '销售责任人'" required><el-select v-model="contractDraft.responsible"><el-option label="张晨" value="张晨" /><el-option label="李然" value="李然" /></el-select></el-form-item></el-col><el-col :span="8"><el-form-item label="合同签署主体"><el-input :model-value="contractDraft.budget?.entity" disabled /></el-form-item></el-col><el-col :span="8"><el-form-item label="业务单元"><el-input v-model="contractDraft.businessLine" /></el-form-item></el-col><el-col :span="8"><el-form-item label="合同编号"><el-input placeholder="提交后由系统生成" disabled /></el-form-item></el-col>
          <el-col :span="8"><el-form-item label="业务类型"><el-input :model-value="contractDraft.budget?.type" disabled /></el-form-item></el-col>
        </el-row></el-form></section>
        <div id="contract-related-section" data-contract-module="contract-related-section" class="association-context"><div><strong><b>02</b>关联单据信息</strong></div></div>
        <section class="contract-block budget-reference-block"><h4 class="contract-content-subtitle budget-basic-subtitle">预算基础信息</h4><el-form label-position="top"><el-row :gutter="20"><el-col :span="8"><el-form-item label="预算单编号"><div class="readonly-link-field"><el-link type="primary" @click="openBudgetFromContract">{{ contractDraft.budget?.code }}</el-link><span>↗</span></div></el-form-item></el-col><el-col :span="8"><el-form-item label="发起方"><el-input model-value="预算单" disabled /></el-form-item></el-col><el-col :span="8"><el-form-item label="预算有效期（天）"><el-input model-value="60" disabled /></el-form-item></el-col><el-col :span="8"><el-form-item label="单据状态"><el-input :model-value="contractDraft.budget?.status" disabled /></el-form-item></el-col><el-col :span="8"><el-form-item label="市场环境说明"><el-input model-value="价格稳定" disabled /></el-form-item></el-col><el-col v-for="field in contractBudgetBasicFields" :key="field.key" :span="8"><el-form-item :label="field.label"><el-input :model-value="contractBudgetFieldValue(field.key)" disabled /></el-form-item></el-col></el-row></el-form></section>
        <section class="contract-block budget-reference-block related-budget-details">
          <el-collapse>
            <el-collapse-item name="budget-plan">
              <template #title><div class="related-budget-collapse-title"><strong>预算计划 / 收益测算</strong><span>采购账期、付款日期、毛利及利润说明</span></div></template>
          <div class="split-panel contract-budget-plan">
            <div>
              <h4 class="contract-subtitle">预算计划</h4>
              <el-form label-position="top" disabled>
                <el-row :gutter="16">
                  <el-col :span="12"><el-form-item label="采购账期（天）"><el-input :model-value="contractDraft.billTime" /></el-form-item></el-col>
                  <el-col :span="12"><el-form-item label="预计付款日期"><el-input :model-value="contractDraft.paymentDate" /></el-form-item></el-col>
                  <el-col :span="12"><el-form-item label="最长销售周期（天）"><el-input model-value="45" /></el-form-item></el-col>
                  <el-col :span="12"><el-form-item label="预计项目单毛利回补总额"><el-input model-value="¥0.00" /></el-form-item></el-col>
                  <el-col :span="24"><el-form-item label="利润说明"><el-input model-value="价格及毛利测算依据预算单" type="textarea" :rows="2" /></el-form-item></el-col>
                  <el-col :span="24"><el-form-item label="业务标签"><el-input model-value="重点业务" /></el-form-item></el-col>
                </el-row>
              </el-form>
            </div>
            <div class="metrics">
              <h4 class="contract-subtitle">收益测算</h4>
              <div class="metric-grid"><div v-for="metric in metrics" :key="metric.label" :class="{ warning: metric.warning }"><span>{{ metric.label }}</span><strong>{{ metric.value }}</strong><small v-if="metric.note">{{ metric.note }}</small></div></div>
            </div>
          </div>
            </el-collapse-item>
            <el-collapse-item name="budget-attachments">
              <template #title><div class="related-budget-collapse-title"><strong>预算附件（1）</strong><span>来源预算单附件</span></div></template>
              <div class="source-budget-attachments"><div class="file-row"><Document /><div><strong>产品报价及采购说明.pdf</strong><span>来源预算单附件</span></div><el-link type="primary">预览</el-link></div></div>
            </el-collapse-item>
          </el-collapse>
        </section>
        <section id="contract-list-section" data-contract-module="contract-list-section" class="contract-block contract-list-block"><h3><b>03</b>{{ contractDraft.type === 'purchase' ? '采购清单' : '销售清单' }}</h3><div v-if="contractDraft.type === 'purchase'" class="contract-local-attention"><WarningFilled /><span>ThinkBook 16+ 本次数量已使用全部预算可用数量，请确认后提交。</span></div><el-table :data="goods" border size="small"><el-table-column prop="spuName" label="SPU名称" min-width="160" fixed="left" show-overflow-tooltip /><el-table-column prop="skuName" label="SKU名称" min-width="210" fixed="left" show-overflow-tooltip /><el-table-column prop="specModel" label="配置说明" min-width="150" show-overflow-tooltip /><el-table-column prop="budgetQuantity" label="预算数量" width="100" align="right" /><el-table-column v-if="contractDraft.type === 'purchase'" prop="availableQuantity" label="预算可用数量" width="120" align="right" /><el-table-column label="本次数量" width="120"><template #default="{ row }"><el-input-number v-model="row.purchaseQuantity" :min="1" :max="contractDraft.type === 'purchase' ? row.availableQuantity : undefined" controls-position="right" /></template></el-table-column><template v-if="contractDraft.type === 'purchase'"><el-table-column prop="lastPurchasePrice" label="上次采购单价" width="115" align="right" /><el-table-column prop="purchaseAmount" label="单台采购价" width="110" align="right" /></template><template v-else><el-table-column prop="limitPrice" label="销售限价" width="105" align="right" /><el-table-column prop="salePrice" label="单台销售价" width="110" align="right" /></template><el-table-column label="合计金额" width="125" align="right"><template #default="{ row }">¥{{ (row.purchaseQuantity * (contractDraft.type === 'purchase' ? row.purchaseAmount : row.salePrice)).toLocaleString() }}</template></el-table-column><template v-if="contractDraft.type === 'purchase'"><el-table-column prop="stock" label="库存数量" width="95" /><el-table-column prop="salePrice" label="单台销售价（元）" width="130" align="right" /><el-table-column prop="saleCycle" label="预计销售周期（天）" width="140" /><el-table-column prop="safeAmount" label="预提单台价保（元）" width="140" /><el-table-column prop="arriveDate" label="预提价保到账日期" width="140" /></template><template v-else><el-table-column prop="purchaseQuantity" label="外采数量" width="95" /><el-table-column prop="stock" label="当前商品库存量" width="125" /><el-table-column prop="numberStockUsed" label="使用库存商品数量" width="140" /><el-table-column prop="inPriceStockUsed" label="库存商品单台成本价（元）" width="180" /><el-table-column prop="safeAmount" label="预提单台价保（元）" width="140" /><el-table-column prop="arriveDate" label="预提价保到账日期" width="140" /></template></el-table></section>
        <section id="contract-content-section" data-contract-module="contract-content-section" class="contract-block contract-info-block"><h3><b>04</b>合同内容</h3><h4 class="contract-content-subtitle">合同信息</h4><el-form label-position="top"><el-row :gutter="16">
          <el-col :span="24" class="contract-field-group-title">合同生成方式</el-col><el-col :span="24"><el-form-item label="合同情况" required class="contract-mode-form-item"><el-radio-group v-model="contractDraft.contractMode" class="contract-mode-cards"><el-radio label="generated"><strong>系统生成合同</strong><span>填写合同条款后，由系统生成合同文件</span></el-radio><el-radio label="upload"><strong>上传合同</strong><span>填写合同编号并上传已有合同文件</span></el-radio><el-radio label="framework"><strong>适用框架协议</strong><span>选择框架协议，仅上传订单证明文件</span></el-radio></el-radio-group></el-form-item></el-col>
          <el-col v-if="contractDraft.contractMode === 'framework'" :span="8"><el-form-item label="框架协议编号"><el-select v-model="contractDraft.frameworkCode"><el-option label="KJXY-2026-0086" value="KJXY-2026-0086" /><el-option label="KJXY-2026-0092" value="KJXY-2026-0092" /></el-select></el-form-item></el-col>
          <el-col v-if="contractDraft.contractMode === 'framework'" :span="8"><el-form-item label="框架协议附件"><el-link type="primary">框架协议正文-KJXY-2026-0086.pdf</el-link></el-form-item></el-col>
          <el-col :span="24" class="contract-field-group-title">合同金额与结算</el-col><el-col v-if="contractDraft.contractMode !== 'framework'" :span="8"><el-form-item label="合同税率情况"><el-select v-model="contractDraft.taxRate"><el-option v-for="rate in 20" :key="rate" :label="`${rate}%`" :value="`${rate}%`" /></el-select></el-form-item></el-col>
          <el-col :span="8"><el-form-item :label="contractDraft.type === 'purchase' ? '付款方式' : '回款方式'"><el-select v-model="contractDraft.payWay"><el-option label="货到付款" value="货到付款" /><el-option label="款到发货" value="款到发货" /><el-option label="分阶段付款" value="分阶段付款" /></el-select></el-form-item></el-col>
          <template v-if="contractDraft.contractMode !== 'framework'"><el-col :span="8" class="primary-amount-field"><el-form-item label="合同总金额（含税）"><el-input model-value="¥380,000.00" disabled /></el-form-item></el-col><el-col :span="8" class="primary-amount-field"><el-form-item label="合同总金额（不含税）"><el-input model-value="¥336,283.19" disabled /></el-form-item></el-col><el-col :span="8" class="primary-amount-field"><el-form-item label="合同税额"><el-input model-value="¥43,716.81" disabled /></el-form-item></el-col><el-col :span="8"><el-form-item :label="contractDraft.type === 'purchase' ? '货款结算' : '回款约定'"><el-input v-model="contractDraft.settlement" /></el-form-item></el-col><el-col :span="8"><el-form-item :label="contractDraft.type === 'purchase' ? '采购账期（天）' : '销售账期（天）'"><el-input v-model="contractDraft.billTime" /></el-form-item></el-col><el-col :span="8"><el-form-item :label="contractDraft.type === 'purchase' ? '预计付款日期' : '预计回款日期'"><el-date-picker v-model="contractDraft.paymentDate" type="date" value-format="YYYY-MM-DD" /></el-form-item></el-col></template>
          <template v-if="contractDraft.type === 'purchase' && contractDraft.contractMode === 'generated'"><el-col :span="24" class="contract-field-group-title">交付与履约</el-col><el-col :span="8"><el-form-item label="产品交付"><el-input v-model="contractDraft.deliveryDays" /></el-form-item></el-col><el-col :span="8"><el-form-item label="收货地址"><el-input v-model="contractDraft.address" /></el-form-item></el-col><el-col :span="8"><el-form-item label="联系人"><el-input v-model="contractDraft.contact" /></el-form-item></el-col><el-col :span="8"><el-form-item label="联系电话"><el-input v-model="contractDraft.phone" /></el-form-item></el-col><el-col :span="8"><el-form-item label="签约地点"><el-input v-model="contractDraft.signingLocation" /></el-form-item></el-col><el-col :span="8"><el-form-item label="技术服务"><el-input v-model="contractDraft.technicalService" /></el-form-item></el-col></template>
          <template v-if="contractDraft.type === 'sale' && contractDraft.contractMode === 'generated'"><el-col :span="24" class="contract-field-group-title">交付与履约</el-col><el-col :span="8"><el-form-item label="开票模式"><el-select v-model="contractDraft.invoiceMode"><el-option label="先票后款" value="先票后款" /><el-option label="先款后票" value="先款后票" /></el-select></el-form-item></el-col><el-col :span="8"><el-form-item label="开票天数"><el-input-number v-model="contractDraft.invoiceDays" :min="1" :max="365" /></el-form-item></el-col><el-col :span="8"><el-form-item label="发票类型"><el-select v-model="contractDraft.invoiceType"><el-option label="增值税专用发票" value="专票" /><el-option label="增值税普通发票" value="普票" /></el-select></el-form-item></el-col><el-col :span="8"><el-form-item label="交付约定"><el-input v-model="contractDraft.deliveryAgreement" /></el-form-item></el-col><el-col :span="8"><el-form-item label="交付要求"><el-input v-model="contractDraft.deliveryRequirement" /></el-form-item></el-col><el-col :span="8"><el-form-item label="技术支持及服务"><el-select v-model="contractDraft.technicalSupport"><el-option label="需要" value="需要" /><el-option label="不需要" value="不需要" /></el-select></el-form-item></el-col><el-col :span="8"><el-form-item label="产品售后"><el-input v-model="contractDraft.afterSales" /></el-form-item></el-col><el-col :span="8"><el-form-item label="质保条件"><el-input v-model="contractDraft.warranty" /></el-form-item></el-col></template><template v-if="contractDraft.contractMode !== 'framework'"><el-col :span="24" class="contract-field-group-title">用印信息</el-col><el-col :span="8"><el-form-item label="用印方式"><el-select v-model="contractDraft.printingMethod"><el-option label="电子用印" value="电子用印" /><el-option label="线下用印" value="线下用印" /></el-select></el-form-item></el-col><template v-if="contractDraft.contractMode === 'generated'"><el-col :span="8"><el-form-item label="用印情况"><el-select v-model="contractDraft.printingSituation"><el-option label="双方用印" value="双方用印" /><el-option label="我方用印" value="我方用印" /><el-option label="对方用印" value="对方用印" /></el-select></el-form-item></el-col><el-col :span="8"><el-form-item label="印章需求"><el-input v-model="contractDraft.sealRequirements" placeholder="请输入印章需求" /></el-form-item></el-col><el-col :span="8"><el-form-item label="审批后自动用印"><el-switch v-model="contractDraft.autoSeal" /></el-form-item></el-col><el-col :span="8"><el-form-item label="是否需要邮寄"><el-switch v-model="contractDraft.needMail" /></el-form-item></el-col><el-col v-if="contractDraft.needMail" :span="8"><el-form-item label="收件人信息"><el-input v-model="contractDraft.receiveInfo" placeholder="请输入收件人、电话及地址" /></el-form-item></el-col><el-col :span="24"><el-form-item label="备注"><el-input v-model="contractDraft.remark" type="textarea" :rows="2" /></el-form-item></el-col></template></template>
        </el-row></el-form></section>
        <section id="contract-files-section" data-contract-module="contract-files-section" class="contract-block contract-files-block">
          <h3><b>05</b>{{ contractDraft.contractMode === 'framework' ? '订单证明文件' : '合同文件' }}</h3>
          <div v-if="contractDraft.contractMode === 'generated'" class="generate-contract-action"><div><strong>系统生成合同</strong><span>合同生成后自动加入下方文件列表</span></div><el-button type="primary" @click="generateContractFile">生成合同</el-button></div>
          <el-form v-if="contractDraft.contractMode === 'upload'" label-position="top" class="upload-contract-code"><el-form-item label="合同编号" required><div class="contract-code-action"><el-input v-model="contractDraft.contractCode" placeholder="请输入合同编号" /><el-button @click="generatePrototypeContractCode">生成合同编号</el-button></div></el-form-item></el-form>
          <div v-if="contractDraft.contractMode === 'upload'" class="compact-file-upload"><el-upload action="#" :auto-upload="false" :show-file-list="false" :on-change="handleContractFileUpload"><el-button type="primary" :icon="Upload">上传合同</el-button></el-upload><span>支持上传多份合同文件，上传后逐份进行AI识别</span></div>
          <div v-if="contractDraft.contractMode === 'framework'" class="compact-file-upload"><el-upload action="#" :auto-upload="false" :show-file-list="false" :on-change="handleOrderProofUpload"><el-button type="primary" :icon="Upload">上传订单证明文件</el-button></el-upload><span>只上传订单证明文件，框架协议附件从所选协议自动带出</span></div>
          <div v-if="workingContractFiles.some(file => file.companyTemplate === '识别中')" class="contract-local-status"><InfoFilled /><span>AI正在逐份识别合同文件，识别完成后将在对应文件行更新结果。</span></div>
          <el-table :data="workingContractFiles" border size="small" class="contract-file-table" :empty-text="contractDraft.contractMode === 'generated' ? '点击生成合同后，文件将在这里展示' : '暂未上传文件'">
            <el-table-column label="文件名" min-width="250"><template #default="{ row }"><el-link type="primary" @click="previewContractFile(row)">{{ row.name }}</el-link><el-dropdown trigger="click"><el-button link type="primary" class="file-download-trigger"><ArrowDown /></el-button><template #dropdown><el-dropdown-menu><el-dropdown-item>预览文件</el-dropdown-item><el-dropdown-item>下载文件</el-dropdown-item></el-dropdown-menu></template></el-dropdown></template></el-table-column>
            <el-table-column prop="size" label="大小" width="90" />
            <el-table-column v-if="contractDraft.contractMode !== 'framework'" prop="companyTemplate" label="是否公司模板" width="130"><template #default="{ row }"><el-tag :type="row.companyTemplate === '是' ? 'success' : row.companyTemplate === '识别中' ? 'warning' : 'info'">{{ row.companyTemplate }}</el-tag></template></el-table-column>
            <el-table-column v-if="contractDraft.contractMode !== 'framework'" prop="sealConfigured" label="是否配置印章" width="125"><template #default="{ row }"><el-tag :type="row.sealConfigured ? 'success' : 'info'">{{ row.sealConfigured ? '是' : '否' }}</el-tag></template></el-table-column>
            <el-table-column v-if="contractDraft.contractMode !== 'framework'" prop="sealCount" label="配置印章数量" width="125" align="center" />
            <el-table-column label="操作" width="145"><template #default="{ row }"><el-button v-if="contractDraft.contractMode !== 'framework'" link type="primary" @click="configureSeal(row)">配置印章</el-button><el-button link type="danger" @click="removeContractFile(row)">删除</el-button></template></el-table-column>
          </el-table>
        </section>
        <section v-if="contractDraft.type === 'purchase'" id="contract-task-section" data-contract-module="contract-task-section" class="contract-block downstream-task-block"><h3><b>06</b>后续任务</h3><el-tabs v-model="downstreamTaskTab"><el-tab-pane :label="`创建采购订单（${purchaseOrderTasks.length}）`" name="purchaseOrder"><div class="task-tab-toolbar"><span>合同审核通过后按任务逐条创建采购订单，订单号在执行成功后回填</span><el-button type="primary" :icon="Plus">新增采购订单任务</el-button></div><el-table :data="purchaseOrderTasks" border size="small"><el-table-column label="操作" width="120" fixed="left"><template #default><el-button link type="primary">编辑</el-button><el-button link type="danger">删除</el-button></template></el-table-column><el-table-column prop="status" label="任务状态" width="100"><template #default="{ row }"><el-tag :type="row.status === '待执行' ? 'warning' : 'success'">{{ row.status }}</el-tag></template></el-table-column><el-table-column prop="orderCode" label="订单号（执行后才有）" min-width="175" /><el-table-column prop="modifier" label="最后修改人" width="105" /><el-table-column prop="modifiedAt" label="最后修改时间" width="165" /><el-table-column prop="spu" label="采购SPU" min-width="170" show-overflow-tooltip /><el-table-column prop="sku" label="采购SKU" min-width="210" show-overflow-tooltip /><el-table-column prop="price" label="单台采购价" width="115" align="right" /><el-table-column prop="quantity" label="采购数量" width="100" align="right" /><el-table-column prop="warehouse" label="入库仓库" min-width="140" /></el-table></el-tab-pane></el-tabs></section>
        <section v-if="contractDraft.type === 'purchase' && contractRelatedOrders.length" class="contract-block related-order-block"><h3><b>07</b>关联采购订单信息</h3><el-table :data="contractRelatedOrders" border size="small" empty-text="当前合同暂未关联采购订单"><el-table-column prop="code" label="订单编号" min-width="170" /><el-table-column prop="type" label="订单类型" width="110" /><el-table-column prop="createMode" label="订单创建方式" width="130" /><el-table-column prop="enterprise" label="供应商" min-width="180" /><el-table-column prop="amount" label="订单金额" width="130" align="right" /><el-table-column prop="status" label="订单状态" width="100" /><el-table-column prop="created" label="创建时间" width="160" /><el-table-column label="操作" width="90"><template #default><el-button link type="primary">查看订单</el-button></template></el-table-column></el-table></section>
      </div><div class="bottom-actions"><el-button @click="requestClose(activeTab)">取消</el-button><el-button @click="savePendingContract">保存为草稿</el-button><el-button type="primary" @click="submitContract">保存并提交</el-button></div></div>
      <aside class="contract-side-directory"><section class="flow-navigation-card"><div class="flow-control-toggle"><List /><span>模块导航</span></div><button v-for="item in contractModules" :key="item.id" :class="['flow-control-item',{active:item.id===currentContractModule}]" @click="jumpToContractModule(item.id)"><span>{{ item.order }}</span><b>{{ item.label }}</b><em :class="item.id === 'contract-files-section' ? 'warning' : 'done'">{{ item.id === 'contract-files-section' ? '待完善' : '已填写' }}</em></button></section><div class="flow-status-reminders contract-status-reminders"><div><b>状态提醒</b><span>2项</span></div><button>合同文件待上传或识别</button><button>印章尚未配置</button></div></aside>
    </section></Teleport>
    <el-dialog v-model="contractArchiveVisible" title="合同归档" width="860px" :close-on-click-modal="false"><el-tabs v-model="contractArchiveForm.type" @tab-change="handleArchiveTypeChange"><el-tab-pane v-for="item in contractArchiveTypes" :key="item.value" :label="item.label" :name="item.value" /></el-tabs><el-form label-position="top"><el-row :gutter="16"><el-col :span="8"><el-form-item label="当前状态"><el-tag :type="contractArchiveForm.status === '已归档' ? 'success' : 'info'">{{ contractArchiveForm.status }}</el-tag></el-form-item></el-col><el-col :span="8"><el-form-item label="归档状态"><el-select v-model="contractArchiveForm.status"><el-option label="待归档" value="待归档" /><el-option label="已归档" value="已归档" /></el-select></el-form-item></el-col><el-col :span="8"><el-form-item label="经办人"><el-input v-model="contractArchiveForm.operator" /></el-form-item></el-col><el-col :span="12"><el-form-item label="归档日期"><el-date-picker v-model="contractArchiveForm.date" type="date" value-format="YYYY-MM-DD" /></el-form-item></el-col><el-col v-if="isScanArchiveType" :span="12"><el-form-item label="归档附件"><div class="archive-file-action"><el-link v-if="contractArchiveForm.attachment" type="primary">{{ contractArchiveForm.attachment }}</el-link><el-upload action="#" :auto-upload="false" :show-file-list="false" :on-change="handleArchiveFileChange"><el-button>{{ contractArchiveForm.attachment ? '替换扫描件' : '上传扫描件' }}</el-button></el-upload></div></el-form-item></el-col><el-col :span="24"><el-form-item label="备注"><el-input v-model="contractArchiveForm.remark" type="textarea" :rows="3" /></el-form-item></el-col></el-row></el-form><div class="archive-history-title">历史归档记录 <span>共 {{ currentArchiveHistory.length }} 条</span></div><el-table :data="currentArchiveHistory" border size="small" max-height="230" empty-text="暂无历史归档记录"><el-table-column prop="time" label="归档时间" width="165" /><el-table-column prop="operator" label="操作人" width="110" /><el-table-column label="附件" min-width="220"><template #default="{ row }"><el-link v-if="row.attachment && row.attachment !== '-'" type="primary">{{ row.attachment }}</el-link><span v-else>-</span></template></el-table-column><el-table-column prop="remark" label="备注" min-width="190" show-overflow-tooltip /></el-table><template #footer><el-button @click="contractArchiveVisible = false">取消</el-button><el-button type="primary" @click="submitContractArchive">保存归档信息</el-button></template></el-dialog>
    <Teleport to="body"><section v-if="activeOrderPage" class="document-page order-edit-page">
      <header class="document-header"><div class="title-block"><div class="eyebrow">{{ orderDraft.typeLabel }} · {{ activeOrderPage.orderPageMode === 'edit' ? '编辑' : '新建' }}</div><h1>{{ activeOrderPage.orderPageMode === 'edit' ? `编辑${orderDraft.typeLabel}` : `新建${orderDraft.typeLabel}` }}</h1><p>来源预算和合同信息自动带出，按现有正式字段规则维护</p></div><div class="budget-create-header-actions"><el-button @click="requestClose(activeTab)">取消</el-button><el-button @click="saveOrderDraft">保存草稿</el-button><el-button type="primary" @click="submitOrder">保存并提交</el-button></div></header>
      <div ref="orderScrollArea" class="document-scroll order-document-scroll" @scroll="updateOrderCurrentModule">
        <article id="order-basic" class="section-card"><SectionTitle number="01" title="基础信息" /><el-form label-position="top"><el-row :gutter="16"><el-col :span="6"><el-form-item label="业务类型" required><el-input :model-value="orderDraft.budget?.type" disabled /></el-form-item></el-col><el-col :span="6"><el-form-item label="订单来源"><el-input :model-value="orderDraft.saveType === 'manual' ? '单独发起' : orderDraft.saveType === 'contract' ? '合同发起' : '预算单发起'" disabled /></el-form-item></el-col><el-col v-if="orderDraft.type === 'sale'" :span="6"><el-form-item label="暂存类型" required><el-select v-model="orderDraft.tempType"><el-option label="非暂存订单" value="0" /><el-option label="暂存订单" value="1" /><el-option label="暂存释放订单" value="2" /></el-select></el-form-item></el-col><el-col :span="6"><el-form-item label="是否占用资金"><el-switch v-model="orderDraft.capitalOccupied" /></el-form-item></el-col><el-col :span="6"><el-form-item label="合同签署主体"><el-input :model-value="orderDraft.budget?.entity" disabled /></el-form-item></el-col><el-col :span="6"><el-form-item :label="orderDraft.type === 'purchase' ? '供应商' : '客户'" required><el-input :model-value="orderDraft.budget?.supplier" :disabled="orderDraft.saveType !== 'manual'" /></el-form-item></el-col><el-col :span="6"><el-form-item label="业务员" required><el-select v-model="orderDraft.salesman"><el-option label="当前用户" value="当前用户" /><el-option label="张晨" value="张晨" /><el-option label="李然" value="李然" /></el-select></el-form-item></el-col><el-col :span="6"><el-form-item label="部门"><el-input v-model="orderDraft.department" /></el-form-item></el-col><el-col :span="6"><el-form-item label="业务单元"><el-input v-model="orderDraft.businessLine" /></el-form-item></el-col></el-row></el-form></article>
        <article id="order-related" class="section-card"><SectionTitle number="02" title="关联来源与交易条件" /><el-form label-position="top"><el-row :gutter="16"><el-col :span="6"><el-form-item label="关联预算单"><el-input :model-value="orderDraft.saveType === 'manual' ? '不关联预算单' : orderDraft.budget?.code" disabled /></el-form-item></el-col><el-col :span="6"><el-form-item label="关联合同"><el-input v-model="orderDraft.contractCode" :disabled="orderDraft.saveType === 'budget'" :placeholder="orderDraft.saveType === 'manual' ? '可选：关联已有合同' : '请选择已生效合同'" /></el-form-item></el-col><el-col :span="6"><el-form-item label="账期（天）" required><el-input-number v-model="orderDraft.billTime" :min="0" :max="365" /></el-form-item></el-col><el-col :span="6"><el-form-item label="发票类型" required><el-select v-model="orderDraft.invoiceType"><el-option label="增值税专用发票" value="专票" /><el-option label="增值税普通发票" value="普票" /><el-option label="未开票" value="未开票" /></el-select></el-form-item></el-col></el-row></el-form></article>
        <article id="order-goods" class="section-card order-goods-section"><SectionTitle number="03" :title="orderDraft.type === 'purchase' ? '采购明细' : '商品明细'"><el-button type="primary" :icon="Plus">添加商品</el-button><el-button>导入商品</el-button></SectionTitle><el-table :data="goods" border><el-table-column type="index" label="序号" width="55" /><el-table-column prop="skuCode" label="SKU编码" width="155" /><el-table-column prop="spuName" label="SPU名称" min-width="170" /><el-table-column prop="skuName" label="SKU名称" min-width="230" /><el-table-column prop="purchaseQuantity" label="数量" width="95" align="right" /><el-table-column :prop="orderDraft.type === 'purchase' ? 'purchaseAmount' : 'salePrice'" label="销售单价" width="120" align="right" /><el-table-column label="合计金额" width="135" align="right"><template #default="{ row }">¥{{ (row.purchaseQuantity * (orderDraft.type === 'purchase' ? row.purchaseAmount : row.salePrice)).toLocaleString() }}</template></el-table-column><el-table-column v-if="orderDraft.type === 'sale'" label="出库仓库" width="150"><template #default>成都中心仓</template></el-table-column><el-table-column label="备注" min-width="130" /></el-table><div class="order-table-summary"><span>SKU种类：{{ goods.length }}</span><span>合计数量：{{ orderTotalQuantity }}</span><strong>订单金额：¥{{ orderTotalAmount.toLocaleString() }}</strong></div></article>
        <article id="order-delivery" class="section-card"><SectionTitle number="04" :title="orderDraft.type === 'purchase' ? '到货与履约' : '收货与履约'" /><el-form label-position="top"><el-row :gutter="16"><template v-if="orderDraft.type === 'sale'"><el-col :span="8"><el-form-item label="收货地址"><el-input v-model="orderDraft.address" /></el-form-item></el-col><el-col :span="8"><el-form-item label="联系人"><el-input v-model="orderDraft.contact" /></el-form-item></el-col><el-col :span="8"><el-form-item label="联系电话"><el-input v-model="orderDraft.phone" /></el-form-item></el-col><el-col :span="8"><el-form-item label="是否代交货"><el-switch v-model="orderDraft.substitute" /></el-form-item></el-col></template><template v-else><el-col :span="8"><el-form-item label="预计到货日期"><el-date-picker v-model="orderDraft.arrivalDate" type="date" value-format="YYYY-MM-DD" /></el-form-item></el-col><el-col :span="8"><el-form-item label="是否直发订单"><el-switch v-model="orderDraft.directDelivery" /></el-form-item></el-col><el-col :span="8"><el-form-item label="价保"><el-input-number v-model="orderDraft.priceProtection" :min="0" /></el-form-item></el-col></template></el-row></el-form></article>
        <article id="order-attachments" class="section-card order-attachments-section"><SectionTitle number="05" title="附件与说明" /><el-row :gutter="20"><el-col :span="12"><el-form label-position="top"><el-form-item label="备注"><el-input v-model="orderDraft.remark" type="textarea" :rows="2" maxlength="500" show-word-limit /></el-form-item></el-form></el-col><el-col :span="12"><el-upload drag action="#" :auto-upload="false"><Upload /><div>点击或拖拽上传附件</div><small>最多5个文件</small></el-upload></el-col></el-row></article>
        <div class="bottom-actions"><el-button @click="requestClose(activeTab)">取消</el-button><el-button @click="saveOrderDraft">保存草稿</el-button><el-button type="primary" @click="submitOrder">保存并提交</el-button></div>
      </div>
      <aside class="contract-side-directory order-side-directory"><section class="order-overview-card"><div><b>订单概览</b><small>实时计算</small></div><p><span>SKU种类</span><strong>{{ goods.length }}</strong></p><p><span>合计数量</span><strong>{{ orderTotalQuantity }}</strong></p><p><span>订单金额</span><strong>¥{{ orderTotalAmount.toLocaleString() }}</strong></p><p><span>账期</span><strong>{{ orderDraft.billTime }}天</strong></p></section><section class="flow-navigation-card"><div class="flow-control-toggle"><List /><span>模块导航</span></div><button v-for="item in orderModules" :key="item.id" :class="['flow-control-item',{active:item.id===currentOrderModule}]" @click="jumpToOrderModule(item.id)"><span>{{ item.order }}</span><b>{{ item.label }}</b><em :class="item.status === '待完善' ? 'warning' : 'done'">{{ item.status }}</em></button></section><div class="flow-status-reminders contract-status-reminders"><div><b>状态提醒</b><span>2项</span></div><button>收货地址待确认</button><button>附件尚未上传</button></div></aside>
    </section></Teleport>
  </div>
</template>

<script setup>
import { computed, defineComponent, h, reactive, ref, nextTick, onMounted, watch } from 'vue'
import { ElMessage, ElMessageBox } from 'element-plus'
import { List, Checked, Plus, Search, Setting, Close, Printer, DocumentChecked, CircleCheckFilled, WarningFilled, Upload, Document, InfoFilled, QuestionFilled, Refresh, Clock, Filter, ArrowDown } from '@element-plus/icons-vue'

const SectionTitle = defineComponent({ props: ['number', 'title'], setup(props, { slots }) { return () => h('div', { class: 'section-heading' }, [h('span', { class: 'section-number' }, props.number), h('h2', props.title), h('div', { class: 'section-extra' }, slots.default?.())]) } })
const RelatedTable = defineComponent({ props: ['type'], setup(props) { const rows = [{ code: props.type === '合同' ? 'XSHT-202608-018' : props.type === '采购订单' ? 'CGDD-202608-103' : 'XSDD-202608-286', owner: '李然', date: '2026-08-20 14:30', amount: '¥268,000.00', status: '已完成' }]; return () => h(resolveComponent('el-table'), { data: rows, border: true }, () => [h(resolveComponent('el-table-column'), { prop: 'code', label: `${props.type}编号`, minWidth: 180 }), h(resolveComponent('el-table-column'), { prop: 'owner', label: '责任人' }), h(resolveComponent('el-table-column'), { prop: 'date', label: '制单时间', minWidth: 160 }), h(resolveComponent('el-table-column'), { prop: 'amount', label: '金额' }), h(resolveComponent('el-table-column'), { prop: 'status', label: '状态' })]) } })
import { resolveComponent } from 'vue'

const { prototypeVariant = 'default' } = defineProps({ prototypeVariant: { type: String, default: 'default' } })

const budgets = ref([
  { code: 'YSGL-202608-02660', owner: '张晨', supplier: '成都星海科技有限公司', type: '产品导向分销', revenue: '¥328,000.00', margin: '18.42%', status: '审批中', version: 1, modifyRecord: '查看修改记录', created: '2026-08-21 09:42', entity: '四川科瑞供应链管理有限公司' },
  { code: 'YSGL-202608-02658', owner: '周敏', supplier: '四川智联商贸有限公司', type: 'FA囤货', revenue: '¥516,800.00', margin: '21.30%', status: '审批完成', version: 2, modifyRecord: '查看修改记录', created: '2026-08-18 09:42', entity: '四川科瑞供应链管理有限公司' },
  { code: 'YSGL-202608-02651', owner: '李然', supplier: '成都云帆数码有限公司', type: '订单导向分销', revenue: '¥186,000.00', margin: '16.85%', status: '草稿', version: 0, created: '2026-08-17 11:20', entity: '四川科瑞供应链管理有限公司' },
])
const orderKeyword = ref(''), saleOrderStatus = ref('全部'), purchaseOrderStatus = ref('全部')
const saleOrderRows = ref([
  { code: 'XSDD-202608-02816', status: '审批中', contractSituation: '有单项合同', budgetCode: 'YSGL-202608-02660', customer: '成都星海科技有限公司', businessType: '订单导向分销', businessUnit: '西南业务部', salesman: '张晨', billTime: 30, contractCode: 'XSHT-202608-00018', department: '销售部', created: '2026-08-24 10:30', invoiceType: '专票', amount: '¥356,000.00', outboundAmount: '¥0.00', invoiceAmount: '¥0.00', paidAmount: '¥0.00' },
  { code: 'XSDD-202608-02792', status: '待发货', contractSituation: '适用框架协议', budgetCode: 'YSGL-202608-02658', customer: '重庆恒信贸易有限公司', businessType: '产品导向分销', businessUnit: '西南业务部', salesman: '周敏', billTime: 45, contractCode: 'KJXY-2026-0086', department: '渠道部', created: '2026-08-22 15:16', invoiceType: '专票', amount: '¥516,800.00', outboundAmount: '¥186,000.00', invoiceAmount: '¥0.00', paidAmount: '¥120,000.00' },
  { code: 'XSDD-202608-02761', status: '已完成', contractSituation: '-', budgetCode: '-', customer: '成都启明科技有限公司', businessType: 'FA-以销定采', businessUnit: '政企业务部', salesman: '李然', billTime: 0, contractCode: '-', department: '销售部', created: '2026-08-19 09:42', invoiceType: '普票', amount: '¥186,000.00', outboundAmount: '¥186,000.00', invoiceAmount: '¥186,000.00', paidAmount: '¥186,000.00' },
])
const purchaseOrderRows = ref([
  { code: 'CGDD-202608-00103', contractSituation: '系统生成合同', directDelivery: '否', status: '待入库', budgetCode: 'YSGL-202608-02660', businessType: '产品导向分销', entity: '四川科瑞供应链管理有限公司', supplier: '成都星海科技有限公司', purchaseOwner: '张晨', saleOwner: '李然', billTime: 30, contractCode: 'CGHT-202608-00192', amount: '¥299,000.00', priceProtection: '¥3,200.00', discount: '¥0.00', inboundAmount: '¥0.00', invoiceAmount: '¥0.00', paidAmount: '¥0.00', payStatus: '未付款' },
  { code: 'CGDD-202608-00098', contractSituation: '上传合同', directDelivery: '是', status: '入库中', budgetCode: 'YSGL-202608-02658', businessType: 'FA-以销定采', entity: '四川科瑞供应链管理有限公司', supplier: '四川智联商贸有限公司', purchaseOwner: '周敏', saleOwner: '周敏', billTime: 45, contractCode: 'CGHT-202608-00186', amount: '¥426,000.00', priceProtection: '¥0.00', discount: '¥2,000.00', inboundAmount: '¥218,000.00', invoiceAmount: '¥218,000.00', paidAmount: '¥120,000.00', payStatus: '部分付款' },
  { code: 'CGDD-202608-00071', contractSituation: '-', directDelivery: '否', status: '完成', budgetCode: '-', businessType: 'B类', entity: '四川科瑞供应链管理有限公司', supplier: '成都云帆数码有限公司', purchaseOwner: '李然', saleOwner: '张晨', billTime: 0, contractCode: '-', amount: '¥88,000.00', priceProtection: '¥0.00', discount: '¥0.00', inboundAmount: '¥88,000.00', invoiceAmount: '¥88,000.00', paidAmount: '¥88,000.00', payStatus: '已付款' },
])
const tabs = ref([{ key: 'list', title: '分销产品预算', closable: false }])
const activeTab = ref('list'), activeView = ref('list'), mode = ref(''), activeSection = ref('basic')
const keyword = ref(''), listStatus = ref('全部'), approvalTab = ref('pending'), relatedTab = ref('sale'), approvalComment = ref('')
const flatApprovalNodes = [
  { order: '01', name: '业务部门初审', approver: '李明 · 销售主管', status: 'done', time: '2026-08-21 09:56' },
  { order: '02', name: '业务部门复审', approver: '王伟 · 销售总监', status: 'done', time: '2026-08-21 10:06' },
  { order: '03', name: '风控审批', approver: '赵倩 · 风控经理', status: 'done', time: '2026-08-21 10:18' },
  { order: '04', name: '财务BP审批', approver: '当前用户 · 财务BP', status: 'active' },
  { order: '05', name: '合同风险审核', approver: '陈晨 / 刘静', status: 'waiting' },
  { order: '06', name: '财务分管领导', approver: '孙强 · 财务总监', status: 'waiting' },
  { order: '07', name: '总裁审批', approver: '周总 · 总裁办', status: 'waiting' },
]
const approvalComments = [{ user: '赵倩', action: '通过', time: '2026-08-21 10:18', content: '客户信用及资金占用情况正常，同意。', tagType: 'success' }, { user: '王伟', action: '通过', time: '2026-08-21 10:06', content: '业务计划及采销价格已核对，同意。', tagType: 'success' }]
const approvalFilters = reactive({ businessType: '', department: '', dateRange: [], keyword: '' })
const approvalBusinessTypes = ['分销产品预算', '销售订单', '采购订单', '销售合同', '采购合同']
const approvalDepartments = ['销售部', '采购部', '财务部', '供应链部']
const dirty = ref(false), showCloseConfirm = ref(false), closingKey = ref(''), auditDialog = ref(''), scrollArea = ref()
const attachmentsExpanded = ref(false)
const flowControlCollapsed = ref(false)
const productDialogVisible = ref(false), productTableRef = ref(), productSelection = ref([])
const advancedSearchVisible = ref(false), modificationVisible = ref(false), pendingContractVisible = ref(false), contractPreconditionVisible = ref(false), contractCreateVisible = ref(false)
const contractArchiveVisible = ref(false)
const contractArchiveTarget = ref(null)
const contractArchiveTypes = [{ value: 'contractScan', label: '合同扫描件' }, { value: 'contractPaper', label: '合同纸质件' }, { value: 'signScan', label: '签收单扫描件' }, { value: 'signPaper', label: '签收单纸质件' }]
const contractArchiveForm = reactive({ type: 'contractScan', status: '待归档', operator: '张晨', date: '2026-08-25', attachment: '', remark: '' })
const isScanArchiveType = computed(() => ['contractScan', 'signScan'].includes(contractArchiveForm.type))
const contractArchiveHistory = ref({
  contractScan: [{ time: '2026-08-23 09:30', operator: '张晨', attachment: 'CGHT-202608-00192-盖章扫描件.pdf', remark: '双方盖章版本已归档' }, { time: '2026-08-21 16:42', operator: '李然', attachment: 'CGHT-202608-00192-初次扫描件.pdf', remark: '首次上传扫描版本' }],
  contractPaper: [],
  signScan: [{ time: '2026-08-23 10:15', operator: '周敏', attachment: '合同签收单-扫描件.pdf', remark: '签收单扫描件已核对' }],
  signPaper: [],
})
const currentArchiveHistory = computed(() => contractArchiveHistory.value[contractArchiveForm.type] || [])
const advancedFilters = reactive({ code: '', owner: '', supplier: '', type: '', status: '', dateRange: [] })
const modificationBudget = ref(null)
const modificationRecords = ref([])
const pendingContracts = ref([{ budgetCode: 'YSGL-202608-02658', contractType: '采购合同', enterprise: '四川智联商贸有限公司', created: '2026-08-22 15:20' }, { budgetCode: 'YSGL-202608-02658', contractType: '销售合同', enterprise: '重庆恒信贸易有限公司', created: '2026-08-22 16:05' }])
const contractManageTab = ref('singleContract'), contractKeyword = ref(''), contractTypeFilter = ref(''), contractStatusFilter = ref(''), contractPageMode = ref('create')
const contractScrollArea = ref()
const contractDetailScrollArea = ref()
const currentContractModule = ref('contract-basic-section')
const currentContractDetailModule = ref(0)
const contractRows = ref([{ code: 'CGHT-202608-00192', type: 'purchase', typeLabel: '采购合同', contractMode: 'generated', enterprise: '四川智联商贸有限公司', budgetCode: 'YSGL-202608-02658', orderCode: 'CGDD-202608-00103', amount: '¥380,000.00', owner: '张晨', status: '已生效', created: '2026-08-20 14:30' }, { code: 'XSHT-202608-00018', type: 'sale', typeLabel: '销售合同', contractMode: 'upload', enterprise: '重庆恒信贸易有限公司', budgetCode: 'YSGL-202608-02658', orderCode: 'XSDD-202608-00286', amount: '¥428,000.00', owner: '李然', status: '审批中', created: '2026-08-22 10:16' }, { code: 'CGHT-草稿-00007', type: 'purchase', typeLabel: '采购合同', contractMode: 'framework', frameworkCode: 'KJXY-2026-0086', enterprise: '成都云帆数码有限公司', budgetCode: 'YSGL-202608-02651', orderCode: '-', amount: '¥186,000.00', owner: '周敏', status: '待提交', created: '2026-08-23 11:10' }, { code: 'XSHT-202608-00021', type: 'sale', typeLabel: '销售合同', contractMode: 'generated', enterprise: '成都启航科技有限公司', budgetCode: 'YSGL-202608-02660', orderCode: '-', amount: '¥356,000.00', owner: '王芳', status: '待提交', created: '2026-08-24 09:18' }, { code: 'XS-KJXY-2026-0012', type: 'sale', typeLabel: '销售框架协议', contractMode: 'framework', frameworkCode: 'XS-KJXY-2026-0012', enterprise: '重庆恒信贸易有限公司', budgetCode: 'YSGL-202608-02658', orderCode: '-', amount: '¥0.00', owner: '李然', status: '已生效', created: '2026-08-18 15:26' }])
const contractDraft = reactive({ budget: null, type: '', typeLabel: '', passed: false, step: 'base', contractMode: 'generated', contractCode: '', responsible: '张晨', businessLine: '西南业务部', billTime: 30, paymentDate: '2026-09-20', frameworkCode: '', signingLocation: '成都', taxRate: '13%', payWay: '货到付款', settlement: '到货验收后30个工作日内支付100%货款', technicalService: '供应商提供安装调试及技术支持', printingMethod: '电子用印', printingSituation: '双方用印', sealRequirements: '合同专用章', autoSeal: true, deliveryDays: '7日', address: '成都市高新区天府大道', contact: '王经理', phone: '13800000000', invoiceMode: '先票后款', invoiceDays: 7, invoiceType: '专票', deliveryAgreement: '合同生效后7日内交付', deliveryRequirement: '按销售订单约定交付', technicalSupport: '需要', afterSales: '按厂商标准售后', warranty: '整机质保一年', needMail: false, receiveInfo: '', remark: '', createPurchaseOrder: true })
const contractFiles = ref([{ name: '采购合同正文.pdf', size: '2.8 MB', companyTemplate: '是', sealConfigured: true, sealCount: 2, history: 'V2', updateContent: '付款账期由30天调整为45天', aiChange: '识别到付款期限发生变化', counterpartySealed: '是' }, { name: '商品清单.xlsx', size: '860 KB', companyTemplate: '否', sealConfigured: false, sealCount: 0, history: 'V1', updateContent: '调整2项商品数量', aiChange: '商品数量与上一版本存在2处差异', counterpartySealed: '否' }, { name: '补充协议.pdf', size: '1.2 MB', companyTemplate: '识别中', sealConfigured: true, sealCount: 1, history: '-', updateContent: '新增售后责任条款', aiChange: '识别中', counterpartySealed: '否' }])
const contractSignFiles = ref([{ name: '合同签收单.pdf', uploader: '张晨', uploadedAt: '2026-08-22 16:20' }])
const contractArchiveRows = ref([
  { type: '合同扫描件归档', status: '已归档', operator: '张晨', date: '2026-08-23', attachment: 'CGHT-202608-00192-盖章扫描件.pdf', remark: '双方盖章版本已归档' },
  { type: '合同纸质件归档', status: '待归档', operator: '-', date: '', attachment: '', remark: '' },
  { type: '签收单扫描件归档', status: '已归档', operator: '周敏', date: '2026-08-23', attachment: '合同签收单-扫描件.pdf', remark: '签收单扫描件已核对' },
  { type: '签收单纸质件归档', status: '待归档', operator: '-', date: '', attachment: '', remark: '' },
])
const contractWorkingFiles = ref([])
const orderProofFiles = ref([])
const workingContractFiles = computed(() => contractDraft.contractMode === 'framework' ? orderProofFiles.value : contractWorkingFiles.value)
const downstreamTaskTab = ref('purchaseOrder')
const purchaseOrderTasks = ref([{ status: '待执行', orderCode: '-', modifier: '张晨', modifiedAt: '2026-08-24 10:20', spu: 'ThinkPad X1 笔记本', sku: 'ThinkPad X1 Carbon Ultra 7 / 32G / 1TB', price: '¥10,900.00', quantity: 20, warehouse: '成都中心仓' }, { status: '待执行', orderCode: '-', modifier: '李然', modifiedAt: '2026-08-24 10:32', spu: 'ThinkBook 16+ 笔记本', sku: 'ThinkBook 16+ Ultra 5 / 32G / 1TB', price: '¥8,100.00', quantity: 10, warehouse: '成都中心仓' }])
const orderDraft = reactive({ budget: null, type: '', typeLabel: '', saveType: 'budget', contractCode: '', salesman: '张晨', purchaseOwner: '李然', department: '销售部', businessLine: '西南业务部', billTime: 30, invoiceType: '专票', arrivalDate: '2026-09-15', directDelivery: false, priceProtection: 0, discount: 0, tempType: '0', capitalOccupied: false, address: '成都市高新区天府大道', contact: '王经理', phone: '13800000000', substitute: false, remark: '' })
const orderScrollArea = ref()
const currentOrderModule = ref('order-basic')
const budgetChangePanels = ref([])
const budgetChangeBatches = [
  { id: 'change-2', version: 2, status: '审批完成', user: '周敏', time: '2026-08-18 09:20', goodsChangeCount: 1, documentChanges: [{ field: '预计销售收入', before: '¥498,000.00', after: '¥516,800.00', change: '+¥18,800.00', direction: 'increase' }, { field: '最长销售周期', before: '60天', after: '45天', change: '-15天', direction: 'decrease' }], goodsChanges: [{ changeType: '修改', dataState: '原', spuName: 'ThinkPad X1 笔记本', skuName: 'ThinkPad X1 Carbon Ultra 7 / 32G / 1TB', purchaseQuantity: 20, purchasePrice: '¥10,900.00', stockQuantity: 0, stockCost: '¥0.00', salePrice: '¥13,200.00', priceProtectionDate: '-', reward: '¥0.00' }, { changeType: '修改', dataState: '新', spuName: 'ThinkPad X1 笔记本', skuName: 'ThinkPad X1 Carbon Ultra 7 / 32G / 1TB', purchaseQuantity: 20, purchasePrice: '¥10,900.00', stockQuantity: 0, stockCost: '¥0.00', salePrice: '¥13,500.00', priceProtectionDate: '-', reward: '¥0.00' }] },
  { id: 'change-1', version: 1, status: '审批完成', user: '李然', time: '2026-08-12 14:35', goodsChangeCount: 1, documentChanges: [{ field: '预计采购成本', before: '¥286,000.00', after: '¥299,000.00', change: '+¥13,000.00', direction: 'increase' }, { field: '预计毛利', before: '¥52,000.00', after: '¥57,000.00', change: '+¥5,000.00', direction: 'increase' }], goodsChanges: [{ changeType: '修改', dataState: '原', spuName: 'ThinkBook 16+ 笔记本', skuName: 'ThinkBook 16+ Ultra 5 / 32G / 1TB', purchaseQuantity: 8, purchasePrice: '¥8,000.00', stockQuantity: 0, stockCost: '¥0.00', salePrice: '¥9,200.00', priceProtectionDate: '-', reward: '¥0.00' }, { changeType: '修改', dataState: '新', spuName: 'ThinkBook 16+ 笔记本', skuName: 'ThinkBook 16+ Ultra 5 / 32G / 1TB', purchaseQuantity: 10, purchasePrice: '¥8,100.00', stockQuantity: 0, stockCost: '¥0.00', salePrice: '¥9,200.00', priceProtectionDate: '-', reward: '¥0.00' }] },
]
const contractRelatedOrders = ref([])
const contractChecks = ref([])
const productFilters = reactive({ keyword: '', businessUnit: '', inStockOnly: false })
const productOptions = ref([
  { spuName: 'ThinkPad X1 笔记本', skuCode: 'SKU-1001', skuName: 'ThinkPad X1 Carbon i7/16G/512G', specModel: 'i7-1360P / 16G / 512G SSD', businessUnit: '西南业务部', stock: 35, purchaseAmount: 8300, salePrice: 12000, saleCycle: 30 },
  { spuName: '戴尔 U2723 显示器', skuCode: 'SKU-1002', skuName: '戴尔 U2723QE 4K 27寸', specModel: '27寸 / 4K / Type-C', businessUnit: '西南业务部', stock: 62, purchaseAmount: 3100, salePrice: 4500, saleCycle: 25 },
  { spuName: '罗技 MX 键鼠套装', skuCode: 'SKU-1003', skuName: '罗技 MX Keys + MX Master 3', specModel: '无线蓝牙套装', businessUnit: '华东业务部', stock: 128, purchaseAmount: 950, salePrice: 1450, saleCycle: 20 },
  { spuName: '华为 MateBook 笔记本', skuCode: 'SKU-1004', skuName: 'MateBook 14s i5/16G/1T', specModel: 'i5-13500H / 16G / 1T', businessUnit: '华东业务部', stock: 18, purchaseAmount: 5600, salePrice: 7999, saleCycle: 35 },
  { spuName: '爱普生 L15158 打印机', skuCode: 'SKU-1005', skuName: '爱普生 L15158 A3彩色', specModel: 'A3 / 彩色 / 墨仓式', businessUnit: '西南业务部', stock: 0, purchaseAmount: 8800, salePrice: 12800, saleCycle: 45 },
])
const businessTypes = ['产品导向分销', '订单导向分销', 'FA囤货', 'FA以销定采', '项目A', '项目B']
const listColumns = [
  { prop: 'code', label: '预算单编号', width: 150, fixed: 'left' }, { prop: 'owner', label: '采购/销售责任人', width: 140 }, { prop: 'entity', label: '合同签署主体', width: 150 }, { prop: 'supplier', label: '供应商/客户', width: 180 }, { prop: 'type', label: '业务类型', width: 140 }, { prop: 'contractCode', label: '合同编号', width: 150 }, { prop: 'orderCode', label: '关联采/销订单', width: 150 }, { prop: 'contractEnterprise', label: '合同供应商/客户', width: 170 }, { prop: 'revenue', label: '预计销售收入', width: 140, align: 'right' }, { prop: 'purchaseCost', label: '预计采购成本', width: 140, align: 'right' }, { prop: 'margin', label: '预期毛利率', width: 120, align: 'right' }, { prop: 'status', label: '状态', width: 110 }, { prop: 'paymentDate', label: '预计付款日期', width: 130 }, { prop: 'creator', label: '创建人', width: 100 }, { prop: 'department', label: '创建人所属部门', width: 150 }, { prop: 'created', label: '创建时间', width: 170 }, { prop: 'stockCost', label: '使用库存商品成本', width: 150, align: 'right' }, { prop: 'priceProtection', label: '预计价保总额', width: 140, align: 'right' }, { prop: 'grossProfit', label: '预计毛利', width: 120, align: 'right' }, { prop: 'maxSaleCycle', label: '最长销售周期', width: 130 }, { prop: 'saleBillTime', label: '销售账期', width: 110 }, { prop: 'marketAnalysis', label: '市场环境说明', width: 150 }, { prop: 'remark', label: '其他说明', width: 130 }, { prop: 'profitDesc', label: '利润说明', width: 130 }, { prop: 'modified', label: '预算是否有修改', width: 140 }, { prop: 'modifyRecord', label: '预算修改记录', width: 140 },
]
const businessTypeConfigs = {
  产品导向分销: { description: '围绕商品备货与预期销售进行预算', fields: [{ key: 'businessUnit', label: '业务单元', kind: 'select', options: ['西南业务部', '华东业务部'], required: true }, { key: 'marketEnvironment', label: '市场环境说明', kind: 'select', options: ['价格稳定', '价格上涨', '价格下行'], required: true }, { key: 'profitBackfill', label: '项目单毛利回补总额', kind: 'number' }] },
  订单导向分销: { description: '以明确订单需求为来源，突出订单关联关系', fields: [{ key: 'sourceOrder', label: '来源销售订单', kind: 'input', required: true }, { key: 'customer', label: '订单客户', kind: 'input', required: true }, { key: 'projectRebate', label: '是否属于项目后返订单', kind: 'switch' }] },
  FA囤货: { description: 'FA备货场景，重点关注资金占用与库存周期', fields: [{ key: 'capitalOccupied', label: '是否占用资金', kind: 'switch', required: true }, { key: 'stockDays', label: '预计库存周期（天）', kind: 'number', required: true }] },
  FA以销定采: { description: '依据销售需求组织采购，强调销售来源与采购匹配', fields: [{ key: 'sourceOrder', label: '来源销售订单', kind: 'input', required: true }, { key: 'capitalOccupied', label: '是否占用资金', kind: 'switch', required: true }, { key: 'purchaseMatch', label: '采销匹配方式', kind: 'select', options: ['一单一采', '多单合并采购'], required: true }] },
  项目A: { description: '项目型业务预算，展示项目身份与交付要求', fields: [{ key: 'projectCode', label: '项目编号', kind: 'input', required: true }, { key: 'projectName', label: '项目名称', kind: 'input', required: true }, { key: 'deliveryBatch', label: '计划交付批次', kind: 'number' }] },
  项目B: { description: '对应代码中的B类，额外关注资金占用情况', fields: [{ key: 'projectCode', label: '项目编号', kind: 'input', required: true }, { key: 'projectName', label: '项目名称', kind: 'input', required: true }, { key: 'capitalOccupied', label: '是否占用资金', kind: 'switch', required: true }] },
}
const form = reactive({ type: '产品导向分销', entity: '四川科瑞供应链管理有限公司', validDays: 60, supplier: '成都星海科技有限公司', owner: '张晨', arrival: '2026-09-08', payment: '2026-09-15', purchaseDays: 0, saleDays: 45, profitDesc: '价格及毛利测算依据预算单', supplement: '', dynamic: {}, contractCode: 'HT-202608-0018', contractEnterprise: '成都星海科技有限公司', signDate: '2026-08-20', contractEndDate: '2027-08-19', estimatedTotalPrice: '¥3,200.00', stockCost: '¥0.00' })
const currentTypeConfig = computed(() => businessTypeConfigs[form.type] || businessTypeConfigs.产品导向分销)
const businessUnitField = computed(() => currentTypeConfig.value.fields.find(field => field.key === 'businessUnit'))
const basicTypeFields = computed(() => currentTypeConfig.value.fields.filter(field => !['businessUnit', 'marketEnvironment', 'profitBackfill'].includes(field.key)))
const contractBudgetTypeConfig = computed(() => businessTypeConfigs[contractDraft.budget?.type] || businessTypeConfigs.产品导向分销)
const contractBudgetBasicFields = computed(() => contractBudgetTypeConfig.value.fields.filter(field => !['businessUnit', 'marketEnvironment'].includes(field.key)))
const documentData = reactive({ code: '', owner: '张晨', supplier: '成都星海科技有限公司', type: '产品导向分销' })
const goods = ref([{ spuName: 'ThinkPad X1 笔记本', skuCode: 'TP-X1C-U7-32-1T', skuName: 'ThinkPad X1 Carbon Ultra 7 / 32G / 1TB / 黑色', specModel: 'Ultra 7 / 32G / 1TB SSD', businessUnit: '西南业务部', budgetQuantity: 30, availableQuantity: 24, purchaseQuantity: 20, lastPurchasePrice: 10800, purchaseAmount: 10900, limitPrice: 12800, salePrice: 13200, stock: 35, numberStockUsed: 0, inPriceStockUsed: 10600, saleCycle: 45, safeAmount: 100, arriveDate: '2026-09-30' }, { spuName: 'ThinkBook 16+ 笔记本', skuCode: 'TB16-U5-32-1T', skuName: 'ThinkBook 16+ Ultra 5 / 32G / 1TB / 灰色', specModel: 'Ultra 5 / 32G / 1TB SSD', businessUnit: '西南业务部', budgetQuantity: 18, availableQuantity: 12, purchaseQuantity: 10, lastPurchasePrice: 8000, purchaseAmount: 8100, limitPrice: 9000, salePrice: 9200, stock: 18, numberStockUsed: 0, inPriceStockUsed: 7900, saleCycle: 30, safeAmount: 80, arriveDate: '2026-09-30' }])
const metrics = computed(() => { const cost = goods.value.reduce((s, x) => s + x.purchaseQuantity * x.purchaseAmount, 0), revenue = goods.value.reduce((s, x) => s + x.purchaseQuantity * x.salePrice, 0), profit = revenue - cost; return [{ label: '预计销售收入', value: `¥${revenue.toLocaleString()}`, note: '', rule: '预计销售收入 = Σ（采购数量 × 单台销售价）' }, { label: '预计采购成本', value: `¥${cost.toLocaleString()}`, note: '', rule: '预计采购成本 = Σ（采购数量 × 单台采购价）' }, { label: '预提价保、奖励总额', value: '¥3,200', note: '', rule: '预提价保、奖励总额 = Σ（预提单台价保 + 预提单台奖励）× 商品数量' }, { label: '预计毛利', value: `¥${profit.toLocaleString()}`, note: '', rule: '预计毛利 = 预计销售收入 − 预计采购成本 + 项目单毛利回补总额 − 预提价保、奖励总额' }, { label: '预期毛利率', value: `${(profit / revenue * 100).toFixed(2)}%`, note: '', rule: '预期毛利率 = 预计毛利 ÷ 预计销售收入 × 100%' }, { label: '预计月均毛利率', value: '8.73%', note: '', warning: true, rule: '预计月均毛利率 = 预期毛利率 ÷ 最长销售周期 × 30天' }] })
const pendingItems = computed(() => budgets.value.filter(x => x.status === '审批中'))
const approvalItems = ref([
  { state: 'pending', statusLabel: '待审批', businessType: '分销产品预算', title: '预算审批', code: 'YSGL-202608-02660', supplier: '成都星海科技有限公司', applicant: '张晨', owner: '张晨', department: '销售部', currentNode: '财务负责人审批', submittedAt: '2026-08-21 10:24', type: '产品导向分销' },
  { state: 'pending', statusLabel: '待审批', businessType: '销售订单', title: '销售订单审批', code: 'XSDD-202608-02816', supplier: '四川智联商贸有限公司', applicant: '周敏', owner: '周敏', department: '销售部', currentNode: '部门负责人审批', submittedAt: '2026-08-20 16:08', type: '订单导向分销' },
  { state: 'pending', statusLabel: '待审批', businessType: '采购合同', title: '采购合同审批', code: 'CGHT-202608-00192', supplier: '成都云帆数码有限公司', applicant: '李然', owner: '李然', department: '采购部', currentNode: '法务审批', submittedAt: '2026-08-19 14:35', type: 'FA囤货' },
  { state: 'done', statusLabel: '已通过', businessType: '分销产品预算', title: '预算审批', code: 'YSGL-202608-02658', supplier: '四川智联商贸有限公司', applicant: '周敏', owner: '周敏', department: '销售部', currentNode: '审批完成', submittedAt: '2026-08-18 09:42', type: 'FA囤货' },
  { state: 'done', statusLabel: '已驳回', businessType: '采购订单', title: '采购订单审批', code: 'CGDD-202608-00103', supplier: '成都启明科技有限公司', applicant: '陈浩', owner: '陈浩', department: '采购部', currentNode: '已退回申请人', submittedAt: '2026-08-17 11:20', type: '产品导向分销' },
  { state: 'mine', statusLabel: '审批中', businessType: '销售合同', title: '销售合同审批', code: 'XSHT-202608-00018', supplier: '重庆恒信贸易有限公司', applicant: '当前用户', owner: '当前用户', department: '销售部', currentNode: '财务负责人审批', submittedAt: '2026-08-21 09:10', type: '订单导向分销' },
])
const approvalCounts = computed(() => ({ pending: approvalItems.value.filter(x => x.state === 'pending').length, done: approvalItems.value.filter(x => x.state === 'done').length, mine: approvalItems.value.filter(x => x.state === 'mine').length }))
const filteredApprovalItems = computed(() => approvalItems.value.filter(item => {
  const range = approvalFilters.dateRange || []
  const date = item.submittedAt.slice(0, 10)
  return item.state === approvalTab.value && (!approvalFilters.businessType || item.businessType === approvalFilters.businessType) && (!approvalFilters.department || item.department === approvalFilters.department) && (!approvalFilters.keyword || [item.code, item.applicant, item.supplier].some(x => x.includes(approvalFilters.keyword))) && (!range.length || (date >= range[0] && date <= range[1]))
}))
const filteredBudgets = computed(() => budgets.value.filter(x => {
  const range = advancedFilters.dateRange || [], date = (x.created || '').slice(0, 10)
  return (listStatus.value === '全部' || x.status === listStatus.value) && Object.values(x).some(v => String(v).includes(keyword.value)) && (!advancedFilters.code || x.code.includes(advancedFilters.code)) && (!advancedFilters.owner || x.owner.includes(advancedFilters.owner)) && (!advancedFilters.supplier || x.supplier.includes(advancedFilters.supplier)) && (!advancedFilters.type || x.type === advancedFilters.type) && (!advancedFilters.status || x.status === advancedFilters.status) && (!range.length || (date >= range[0] && date <= range[1]))
}))
const filteredContractRows = computed(() => { const tabRows = contractManageTab.value === 'singleContract' ? contractRows.value.filter(row => row.contractMode !== 'framework') : contractManageTab.value === 'saleFrameworkAgreement' ? contractRows.value.filter(row => row.type === 'sale' && row.contractMode === 'framework') : contractRows.value.filter(row => row.type === 'purchase' && row.contractMode === 'framework'); return tabRows.filter(row => (!contractTypeFilter.value || row.type === contractTypeFilter.value) && (!contractStatusFilter.value || row.status === contractStatusFilter.value) && (!contractKeyword.value || [row.code, row.enterprise, row.owner, row.budgetCode, row.orderCode].some(value => String(value).includes(contractKeyword.value)))) })
const contractModules = computed(() => [
  { id: 'contract-basic-section', order: '01', label: '基础信息', note: '已完成' },
  { id: 'contract-related-section', order: '02', label: '关联单据信息', note: '已完成' },
  { id: 'contract-list-section', order: '03', label: contractDraft.type === 'purchase' ? '采购清单' : '销售清单', note: contractDraft.type === 'purchase' ? '1项风险' : '已完成' },
  { id: 'contract-content-section', order: '04', label: '合同内容', note: '已填写' },
  { id: 'contract-files-section', order: '05', label: contractDraft.contractMode === 'framework' ? '订单证明文件' : '合同文件', note: workingContractFiles.value.some(file => file.companyTemplate === '识别中') ? '识别中' : '待完善' },
  ...(contractDraft.type === 'purchase' ? [{ id: 'contract-task-section', order: '06', label: '后续任务', note: `${purchaseOrderTasks.value.length}条` }] : []),
])
const currentContractModuleLabel = computed(() => contractModules.value.find(item => item.id === currentContractModule.value)?.label || '基础信息')
const filteredProductOptions = computed(() => productOptions.value.filter(item => (!productFilters.keyword || [item.spuName, item.skuCode, item.skuName].some(value => value.toLowerCase().includes(productFilters.keyword.toLowerCase()))) && (!productFilters.businessUnit || item.businessUnit === productFilters.businessUnit) && (!productFilters.inStockOnly || item.stock > 0)))
const selectableProductSelection = computed(() => productSelection.value.filter(item => !isProductAdded(item)))
const currentDocument = computed(() => !!tabs.value.find(x => x.key === activeTab.value)?.mode)
const activeContractPage = computed(() => { const tab = tabs.value.find(x => x.key === activeTab.value); return tab?.contractPageMode === 'detail' ? tab : null })
const activeOrderPage = computed(() => { const tab = tabs.value.find(x => x.key === activeTab.value); return tab?.orderPageMode ? tab : null })
const orderTotalQuantity = computed(() => goods.value.reduce((sum, row) => sum + Number(row.purchaseQuantity || 0), 0))
const orderTotalAmount = computed(() => goods.value.reduce((sum, row) => sum + Number(row.purchaseQuantity || 0) * Number(orderDraft.type === 'purchase' ? row.purchaseAmount : row.salePrice), 0))
const orderModules = computed(() => [
  { id: 'order-basic', order: '01', label: '基础信息', status: '已完成' },
  { id: 'order-related', order: '02', label: '关联来源与交易条件', status: orderDraft.contractCode ? '已完成' : '待完善' },
  { id: 'order-goods', order: '03', label: orderDraft.type === 'purchase' ? '采购明细' : '商品明细', status: goods.value.length ? '已完成' : '待完善' },
  { id: 'order-delivery', order: '04', label: orderDraft.type === 'purchase' ? '到货与履约' : '收货与履约', status: orderDraft.type === 'purchase' || (orderDraft.address && orderDraft.contact && orderDraft.phone) ? '已完成' : '待完善' },
  { id: 'order-attachments', order: '05', label: '附件与说明', status: '待完善' },
])
const contractDetailModules = computed(() => {
  const purchase = activeContractPage.value?.data?.type === 'purchase'
  const approvalStatus = activeContractPage.value?.data?.status === '审批中' ? '审批中' : '已完成'
  return [
    { order: '01', label: '基础信息', status: '已完成' },
    { order: '02', label: '关联单据信息', status: '1项' },
    { order: '03', label: purchase ? '采购清单' : '销售清单', status: `${goods.value.length}项` },
    { order: '04', label: '合同内容', status: '已完成' },
    { order: '05', label: activeContractPage.value?.data?.contractMode === 'framework' ? '订单证明文件' : '合同文件', status: `${contractFiles.value.length}个` },
    { order: '06', label: '签收文件与归档信息', status: '已归档' },
    ...(purchase ? [{ order: '07', label: '后续任务', status: `${purchaseOrderTasks.value.length}项` }] : []),
    { order: purchase ? '08' : '07', label: '审批信息', status: approvalStatus },
  ]
})
const currentContractDetailModuleLabel = computed(() => contractDetailModules.value[currentContractDetailModule.value]?.label || '基础信息')
const budgetModules = computed(() => [{ id: 'basic', order: '01', label: '基础信息', kind: '核心' }, { id: 'plan', order: '02', label: '预算计划 / 收益测算', kind: '核心' }, { id: 'goods', order: '03', label: '采销明细', kind: '核心' }, ...(!isEditing.value ? [{ id: 'related', order: '04', label: '关联单据', kind: '辅助' }] : []), { id: 'attachments', order: isEditing.value ? '04' : '05', label: '附件与说明', kind: '辅助' }, ...(mode.value === 'detail' && documentData.version ? [{ id: 'change-records', order: '06', label: '变更记录', kind: '辅助' }] : []), ...(mode.value === 'detail' ? [{ id: 'approval-info', order: '07', label: '审批信息', kind: '辅助' }] : []), ...(mode.value === 'audit' ? [{ id: 'approval-progress', order: '06', label: '审批进度', kind: '辅助' }, { id: 'approval-comments', order: '07', label: '审批评论', kind: '辅助' }] : [])])
const currentBudgetModuleLabel = computed(() => budgetModules.value.find(item => item.id === activeSection.value)?.label || '基础信息')
const activeContractEditPage = computed(() => { const tab = tabs.value.find(x => x.key === activeTab.value); return ['create', 'edit'].includes(tab?.contractPageMode) ? tab : null })
const activeSupplierPage = computed(() => { const tab = tabs.value.find(x => x.key === activeTab.value); return tab?.supplierPage ? tab : null })
const supplierAmounts = [{ label: '预付金额', value: '¥20,041.00' }, { label: '应付金额', value: '¥58,132.00' }, { label: '应收货票', value: '¥1,898.00' }, { label: '供应商价保', value: '¥14,842.00' }, { label: '供应商销售价保', value: '¥208.00' }, { label: '可用抵扣', value: '¥1.00' }, { label: '应收价保', value: '¥224.00' }, { label: '应收奖励', value: '¥0.00' }]
const supplierTransitRows = [{ spu: 'ThinkPad X1 笔记本', sku: 'ThinkPad X1 Carbon Ultra 7 / 32G / 1TB', quantity: 20, amount: '¥218,000.00', arrival: '2026-09-15' }, { spu: 'ThinkBook 16+ 笔记本', sku: 'ThinkBook 16+ Ultra 5 / 32G / 1TB', quantity: 10, amount: '¥81,000.00', arrival: '2026-09-20' }]
const isEditing = computed(() => ['create', 'edit'].includes(mode.value))
const modeLabel = computed(() => ({ create: '新建预算', edit: '编辑预算', detail: '预算详情', audit: '预算审核' }[mode.value]))
const documentTitle = computed(() => mode.value === 'create' ? '新建分销产品预算' : mode.value === 'edit' ? '编辑分销产品预算' : documentData.code)
const showAuditDialog = computed({ get: () => !!auditDialog.value, set: v => { if (!v) auditDialog.value = '' } })
function statusType(status) { return ({ 审批中: 'warning', 审批完成: 'success', 草稿: 'info' }[status]) }
function resetAdvancedSearch() { Object.assign(advancedFilters, { code: '', owner: '', supplier: '', type: '', status: '', dateRange: [] }) }
function applyAdvancedSearch() { advancedSearchVisible.value = false; ElMessage.success('已应用高级搜索条件') }
function openModificationRecords(row) { modificationBudget.value = row; modificationRecords.value = [{ version: 'V2', field: '预计销售收入', before: '¥498,000.00', after: row.revenue, user: '周敏', time: '2026-08-18 09:20' }, { version: 'V2', field: '最长销售周期', before: '60天', after: '45天', user: '周敏', time: '2026-08-18 09:20' }, { version: 'V1', field: '供应商', before: '成都星海科技有限公司', after: row.supplier, user: row.owner, time: '2026-08-17 16:42' }]; modificationVisible.value = true }
function revertBudget(row) { row.status = '草稿'; ElMessage.success('审批已撤销，预算单已回到草稿状态') }
function checkContractPrecondition(row, type) { contractDraft.budget = row; contractDraft.type = type; contractDraft.typeLabel = type === 'purchase' ? '采购合同' : '销售合同'; contractChecks.value = [{ label: '预算单已审批完成', passed: row.status === '审批完成' }, { label: '预算单仍在有效期内', passed: true }, { label: '采销明细完整且金额有效', passed: true }, { label: `${type === 'purchase' ? '供应商' : '客户'}信息已维护`, passed: !!row.supplier }, { label: `不存在重复的${type === 'purchase' ? '采购' : '销售'}合同申请`, passed: true }]; contractDraft.passed = contractChecks.value.every(x => x.passed); contractPreconditionVisible.value = true }
function createContractDraft() { contractPreconditionVisible.value = false; openContractDraft(contractDraft.budget, contractDraft.type) }
function prepareContractDraft(row, type) { contractDraft.budget = row; contractDraft.type = type; contractDraft.typeLabel = type === 'purchase' ? '采购合同' : '销售合同'; contractDraft.step = 'base'; contractDraft.contractMode = 'generated'; contractDraft.contractCode = ''; contractWorkingFiles.value = []; orderProofFiles.value = [] }
function openContractDraft(row, type) { const key = `contract-create-${type}-${row.code}`; if (!tabs.value.some(x => x.key === key)) tabs.value.push({ key, title: `新建${type === 'purchase' ? '采购' : '销售'}合同`, closable: true, contractPageMode: 'create', contractType: type, data: { budget: row } }); activeTab.value = key; activeView.value = 'contract'; contractPageMode.value = 'create'; prepareContractDraft(row, type); contractRelatedOrders.value = [] }
function openContractPrototype() { switchView('contract') }
function openContractPage(pageMode, type, row = null) { const budget = budgets.value.find(x => x.code === row?.budgetCode) || budgets.value[1], suffix = row?.code || Date.now(), key = `contract-${pageMode}-${suffix}`; if (!tabs.value.some(x => x.key === key)) tabs.value.push({ key, title: pageMode === 'create' ? `新建${type === 'purchase' ? '采购' : '销售'}合同` : `${pageMode === 'edit' ? '编辑' : '详情'}-${row.code.slice(-6)}`, closable: true, contractPageMode: pageMode, contractType: type, data: { ...row, budget } }); activeTab.value = key; activeView.value = 'contract'; contractPageMode.value = pageMode; if (pageMode === 'detail') return; prepareContractDraft(budget, type); contractDraft.contractCode = row?.code || ''; contractDraft.contractMode = row?.contractMode || 'generated'; contractDraft.frameworkCode = row?.frameworkCode || ''; if (type === 'purchase' && row?.orderCode && row.orderCode !== '-') contractRelatedOrders.value = [{ code: row.orderCode, type: '采购订单', createMode: '合同创建', enterprise: row.enterprise, amount: row.amount, status: '已完成', created: row.created }]; else contractRelatedOrders.value = [] }
function savePendingContract() { pendingContracts.value.push({ budgetCode: contractDraft.budget.code, contractType: contractDraft.typeLabel, enterprise: contractDraft.budget.supplier, created: '2026-08-23 11:10' }); contractCreateVisible.value = false; ElMessage.success('合同已暂存，可从待提交合同继续填写') }
function submitContract() { contractCreateVisible.value = false; ElMessage.success(`${contractDraft.typeLabel}已提交审批`) }
function continueContract(row) { pendingContractVisible.value = false; openContractPage('edit', row.contractType === '采购合同' ? 'purchase' : 'sale', { ...row, code: `草稿-${row.budgetCode.slice(-5)}`, typeLabel: row.contractType, status: '待提交' }) }
function openOrderDraft(row, type) { const key = `order-create-${type}-${row.code}`; if (!tabs.value.some(x => x.key === key)) tabs.value.push({ key, title: `新建${type === 'purchase' ? '采购' : '销售'}订单`, closable: true, orderPageMode: 'create', orderType: type, data: { budget: row } }); activeTab.value = key; activeView.value = 'contract'; orderDraft.budget = row; orderDraft.type = type; orderDraft.typeLabel = type === 'purchase' ? '采购订单' : '销售订单'; orderDraft.contractCode = type === 'purchase' ? 'CGHT-202608-00192' : 'XSHT-202608-00018'; currentOrderModule.value = 'order-basic' }
function openManualOrder(type) { const source = { code: '', owner: '当前用户', supplier: type === 'purchase' ? '请选择供应商' : '请选择客户', type: '订单导向分销', entity: '四川科瑞供应链管理有限公司' }; const key = `order-manual-${type}-${Date.now()}`; tabs.value.push({ key, title: `新建${type === 'purchase' ? '采购' : '销售'}订单`, closable: true, orderPageMode: 'create', orderType: type, data: { budget: source } }); activeTab.value = key; activeView.value = type === 'purchase' ? 'purchaseOrders' : 'saleOrders'; Object.assign(orderDraft, { budget: source, type, typeLabel: type === 'purchase' ? '采购订单' : '销售订单', saveType: 'manual', contractCode: '', salesman: '当前用户', purchaseOwner: '当前用户', department: type === 'purchase' ? '采购部' : '销售部' }); currentOrderModule.value = 'order-basic' }
function saveOrderDraft() { ElMessage.success(`${orderDraft.typeLabel}已保存为草稿`) }
function submitOrder() { ElMessage.success(`${orderDraft.typeLabel}已提交审批`) }
function resetApprovalFilters() { Object.assign(approvalFilters, { businessType: '', department: '', dateRange: [], keyword: '' }) }
function approvalStatusType(status) { return ({ done: 'success', active: 'warning', waiting: 'info' }[status]) }
function approvalStatusLabel(status) { return ({ done: '已完成', active: '进行中', waiting: '未开始' }[status]) }
function handleApprovalOpen(row) { if (row.businessType === '分销产品预算') openDocument(row.state === 'pending' ? 'audit' : 'detail', row); else ElMessage.info(`${row.businessType}将进入对应业务单据审核页`) }
function switchView(view) { activeView.value = view; activeTab.value = view; if (!tabs.value.some(x => x.key === view)) tabs.value.push({ key: view, title: view === 'approval' ? '我的审批' : view === 'contract' ? '合同管理' : view === 'saleOrders' ? '销售订单' : view === 'purchaseOrders' ? '采购订单' : '分销产品预算', closable: view !== 'list' }) }
function activateTab(key) { activeTab.value = key; const tab = tabs.value.find(x => x.key === key); if (['list', 'approval', 'contract', 'saleOrders', 'purchaseOrders'].includes(key)) activeView.value = key; else if (tab?.supplierPage) activeView.value = 'contract'; else if (tab?.orderPageMode) { activeView.value = tab.orderType === 'purchase' ? 'purchaseOrders' : 'saleOrders'; orderDraft.budget = tab.data?.budget || budgets.value[1]; orderDraft.type = tab.orderType; orderDraft.typeLabel = tab.orderType === 'purchase' ? '采购订单' : '销售订单' } else if (tab?.contractPageMode) { activeView.value = 'contract'; contractPageMode.value = tab.contractPageMode; if (tab.contractPageMode !== 'detail') { prepareContractDraft(tab.data?.budget || budgets.value[1], tab.contractType); contractDraft.contractCode = tab.data?.code || '' } } else { activeView.value = 'list'; mode.value = tab.mode; Object.assign(documentData, tab.data) } }
function openBudgetFromContract() { if (contractDraft.budget) { activeView.value = 'list'; openDocument('detail', contractDraft.budget) } }
function openSupplierDetail(name) { const supplierName = name || '供应商详情', key = `supplier-${supplierName}`; if (!tabs.value.some(tab => tab.key === key)) tabs.value.push({ key, title: `供应商详情-${supplierName.slice(0, 6)}`, closable: true, supplierPage: true, data: { name: supplierName } }); activeTab.value = key; activeView.value = 'contract' }
function jumpToContractModule(id) { const container = contractScrollArea.value, target = container?.querySelector(`#${id}`); if (!target) return; currentContractModule.value = id; container.scrollTo({ top: Math.max(0, target.offsetTop - 58), behavior: 'smooth' }) }
function updateContractCurrentModule() { const container = contractScrollArea.value; if (!container) return; const threshold = container.getBoundingClientRect().top + 110; let active = contractModules.value[0]?.id; for (const item of contractModules.value) { const target = container.querySelector(`#${item.id}`); if (target && target.getBoundingClientRect().top <= threshold) active = item.id; } if (active) currentContractModule.value = active }
function jumpToContractDetailModule(index) { const container = contractDetailScrollArea.value, target = container?.querySelectorAll('.section-card')[index]; if (!target) return; currentContractDetailModule.value = index; container.scrollTo({ top: Math.max(0, target.offsetTop - 58), behavior: 'smooth' }) }
function updateContractDetailCurrentModule() { const container = contractDetailScrollArea.value; if (!container) return; const threshold = container.getBoundingClientRect().top + 110; let active = 0; container.querySelectorAll('.section-card').forEach((target, index) => { if (target.getBoundingClientRect().top <= threshold) active = index }); currentContractDetailModule.value = active }
function jumpToOrderModule(id) { const container = orderScrollArea.value, target = container?.querySelector(`#${id}`); if (!target) return; currentOrderModule.value = id; container.scrollTo({ top: Math.max(0, target.offsetTop - 58), behavior: 'smooth' }) }
function updateOrderCurrentModule() { const container = orderScrollArea.value; if (!container) return; const threshold = container.getBoundingClientRect().top + 110; let active = orderModules.value[0]?.id; for (const item of orderModules.value) { const target = container.querySelector(`#${item.id}`); if (target && target.getBoundingClientRect().top <= threshold) active = item.id } if (active) currentOrderModule.value = active }
function jumpToBudgetModule(id) { const container = scrollArea.value, target = container?.querySelector(`#${id}`); if (!target) return; activeSection.value = id; container.scrollTo({ top: Math.max(0, target.offsetTop - 58), behavior: 'smooth' }) }
function jumpFromFlowControl(id) { if (id === 'attachments') attachmentsExpanded.value = true; nextTick(() => jumpToBudgetModule(id)) }
function updateBudgetCurrentModule() { const container = scrollArea.value; if (!container) return; const threshold = container.getBoundingClientRect().top + 110; let active = budgetModules.value[0]?.id; for (const item of budgetModules.value) { const target = container.querySelector(`#${item.id}`); if (target && target.getBoundingClientRect().top <= threshold) active = item.id } if (active) activeSection.value = active }
function budgetModuleStatus(item) {
  if (mode.value === 'detail') {
    const labels = { related: '3类', attachments: '1个', 'change-records': `${documentData.version || 0}次`, 'approval-info': '已通过' }
    return { label: labels[item.id] || '已完成', className: 'done' }
  }
  if (mode.value === 'audit') {
    const labels = { attachments: '1个', 'approval-progress': '当前', 'approval-comments': '2条' }
    return { label: labels[item.id] || '已核对', className: item.id === 'approval-progress' ? 'warning' : 'done' }
  }
  if (item.id === 'basic') return { label: '待完善', className: 'warning' }
  if (item.id === 'attachments') return { label: '未上传', className: 'empty' }
  return { label: '已完成', className: 'done' }
}
function openDocument(nextMode, row = budgets.value[0]) { const code = nextMode === 'create' ? 'new' : row.code, key = `${nextMode}-${code}`, existing = tabs.value.find(x => x.key === key), actionName = ({ audit: '审核', edit: '编辑', detail: '详情' })[nextMode]; if (!existing) tabs.value.push({ key, title: nextMode === 'create' ? '新建预算' : `${actionName}-${code.slice(-5)}`, closable: true, mode: nextMode, data: { ...row } }); activeTab.value = key; mode.value = nextMode; Object.assign(documentData, row); form.type = nextMode === 'create' ? '产品导向分销' : row.type; form.supplier = row.supplier || form.supplier; form.owner = row.owner || form.owner; form.entity = row.entity || form.entity; form.dynamic = nextMode === 'create' ? {} : { businessUnit: '西南业务部', marketEnvironment: '价格稳定', profitBackfill: 0, faChannel: '核心渠道', capitalOccupied: false, stockDays: 45, sourceOrder: 'XSDD-202608-00286', customer: '重庆恒信贸易有限公司', projectRebate: false, purchaseMatch: '一单一采', projectCode: 'XM-2026-018', projectName: '企业终端设备采购', deliveryBatch: 1 }; dirty.value = nextMode === 'create'; nextTick(() => scrollArea.value?.scrollTo({ top: 0 })) }
function changeMode(nextMode) { mode.value = nextMode; const tab = tabs.value.find(x => x.key === activeTab.value); if (tab) { tab.mode = nextMode; tab.title = `编辑-${documentData.code.slice(-5)}` } dirty.value = true }
function requestClose(key) { const tab = tabs.value.find(x => x.key === key); if (dirty.value && tab?.mode && ['create', 'edit'].includes(tab.mode)) { closingKey.value = key; showCloseConfirm.value = true; return } closeTab(key) }
function closeTab(key) { tabs.value = tabs.value.filter(x => x.key !== key); activeTab.value = activeView.value; dirty.value = false }
function confirmClose() { showCloseConfirm.value = false; closeTab(closingKey.value) }
function saveThenClose() { ElMessage.success('草稿已保存'); showCloseConfirm.value = false; closeTab(closingKey.value) }
function saveDraft() { dirty.value = false; ElMessage.success('草稿已保存') }
function submitDocument() { dirty.value = false; ElMessage.success('预算已提交审批'); closeTab(activeTab.value) }
function completeAudit() { if (auditDialog.value === 'reject' && !approvalComment.value.trim()) return ElMessage.warning('请输入驳回原因'); const result = auditDialog.value; const code = documentData.code; const item = budgets.value.find(x => x.code === code); if (item) item.status = result === 'approve' ? '审批完成' : '草稿'; const task = approvalItems.value.find(x => x.code === code); if (task) { task.state = 'done'; task.statusLabel = result === 'approve' ? '已通过' : '已驳回'; task.currentNode = result === 'approve' ? '审批完成' : '已退回申请人' } auditDialog.value = ''; ElMessage.success(result === 'reject' ? '已驳回申请' : '审批已完成'); closeTab(activeTab.value); switchView('approval') }
function addGoods() { goods.value.push({ spuName: '', skuCode: '', skuName: '', businessUnit: '', purchaseQuantity: 1, purchaseAmount: 0, salePrice: 0, saleCycle: 30 }); dirty.value = true }
function removeGoods(index) { goods.value.splice(index, 1); dirty.value = true }
function isProductAdded(product) { return goods.value.some(item => item.skuCode === product.skuCode) }
function isProductSelectable(product) { return !isProductAdded(product) }
function openProductDialog() { productSelection.value = []; Object.assign(productFilters, { keyword: '', businessUnit: '', inStockOnly: false }); productDialogVisible.value = true }
function confirmProductSelection() { const additions = selectableProductSelection.value.map(product => ({ spuName: product.spuName, skuCode: product.skuCode, skuName: product.skuName, businessUnit: product.businessUnit, purchaseQuantity: 1, purchaseAmount: product.purchaseAmount, salePrice: product.salePrice, saleCycle: product.saleCycle })); goods.value.push(...additions); productDialogVisible.value = false; dirty.value = true; ElMessage.success(`已添加 ${additions.length} 个商品`) }
function handleBusinessTypeChange(type) { form.dynamic = {}; documentData.type = type; dirty.value = true }
function contractBudgetFieldValue(key) { return ({ businessUnit: contractDraft.businessLine, marketEnvironment: '价格稳定', profitBackfill: '¥0.00', sourceOrder: 'XSDD-202608-00286', customer: '重庆恒信贸易有限公司', projectRebate: '否', faChannel: '核心渠道', capitalOccupied: '否', stockDays: '45天', purchaseMatch: '一单一采', projectCode: 'XM-2026-018', projectName: '企业终端设备采购', deliveryBatch: '1批' })[key] || '-' }
function handleBatchImport(file) { ElMessage.success(`已导入 ${file.name}，请核对商品明细`); dirty.value = true }
function handleAttachmentChange(file) { if ((file.raw?.size || file.size) / 1024 / 1024 > 20) { ElMessage.error('单个附件不能超过20MB'); return } dirty.value = true }
function previewContractFile(row) { ElMessage.info(`打开 ${row.name}，可在文件预览中下载`) }
function contractModeLabel(mode) { return ({ generated: '系统生成合同', upload: '上传合同', framework: '适用框架协议' })[mode] || '系统生成合同' }
function generateContractFile() { const name = contractDraft.type === 'purchase' ? '采购合同正文-系统生成.docx' : '销售合同正文-系统生成.docx'; contractWorkingFiles.value = [{ name, size: '2.6 MB', companyTemplate: '是', sealConfigured: false, sealCount: 0 }]; ElMessage.success('合同已生成并加入文件列表') }
function generatePrototypeContractCode() { contractDraft.contractCode = `${contractDraft.type === 'purchase' ? 'CGHT' : 'XSHT'}-202608-00208`; ElMessage.success('合同编号已生成') }
function handleContractFileUpload(file) { contractWorkingFiles.value.push({ name: file.name, size: file.size ? `${(file.size / 1024 / 1024).toFixed(1)} MB` : '-', companyTemplate: '识别中', sealConfigured: false, sealCount: 0 }) }
function handleOrderProofUpload(file) { orderProofFiles.value.push({ name: file.name, size: file.size ? `${(file.size / 1024 / 1024).toFixed(1)} MB` : '-' }) }
function configureSeal(row) { row.sealConfigured = true; row.sealCount = row.sealCount || 1; ElMessage.success(`已打开 ${row.name} 的印章配置`) }
function removeContractFile(row) { const list = contractDraft.contractMode === 'framework' ? orderProofFiles.value : contractWorkingFiles.value; const index = list.indexOf(row); if (index >= 0) list.splice(index, 1); ElMessage.success('文件已删除') }
function archiveTypeLabel(type) { return contractArchiveTypes.find(item => item.value === type)?.label || '' }
function loadArchiveRecord(type) { const label = archiveTypeLabel(type); const record = contractArchiveRows.value.find(item => item.type.startsWith(label)); Object.assign(contractArchiveForm, { type, status: record?.status || '待归档', operator: record?.operator === '-' ? (contractArchiveTarget.value?.owner || '张晨') : (record?.operator || contractArchiveTarget.value?.owner || '张晨'), date: record?.date || '2026-08-25', attachment: record?.attachment || '', remark: record?.remark || '' }) }
function openContractArchive(row) { contractArchiveTarget.value = row; loadArchiveRecord('contractScan'); contractArchiveVisible.value = true }
function handleArchiveTypeChange(type) { loadArchiveRecord(type) }
function handleArchiveFileChange(file) { contractArchiveForm.attachment = file.name }
function submitContractArchive() { const type = archiveTypeLabel(contractArchiveForm.type); const archiveRow = contractArchiveRows.value.find(item => item.type.startsWith(type)); if (archiveRow) Object.assign(archiveRow, { status: contractArchiveForm.status, operator: contractArchiveForm.operator, date: contractArchiveForm.date, attachment: contractArchiveForm.attachment, remark: contractArchiveForm.remark }); if (contractArchiveForm.status === '已归档') contractArchiveHistory.value[contractArchiveForm.type].unshift({ time: `${contractArchiveForm.date} 14:30`, operator: contractArchiveForm.operator, attachment: contractArchiveForm.attachment || '-', remark: contractArchiveForm.remark || '-' }); contractArchiveVisible.value = false; ElMessage.success(`${type}归档信息已保存，并形成一条历史记录`) }
async function cancelContract(row) { try { await ElMessageBox.confirm(`合同作废后不可继续执行后续业务，确定作废合同 ${row.code} 吗？`, '合同作废', { type: 'warning', confirmButtonText: '确认作废', cancelButtonText: '取消' }); row.status = '已作废'; const source = contractRows.value.find(item => item.code === row.code); if (source) source.status = '已作废'; ElMessage.success('合同已作废') } catch { /* 用户取消 */ } }
watch(() => contractDraft.contractMode, (next, previous) => {
  if (!previous || next === previous) return
  contractDraft.contractCode = ''
  if (next === 'framework') {
    contractWorkingFiles.value = []
    contractDraft.frameworkCode = ''
  } else {
    orderProofFiles.value = []
    contractDraft.frameworkCode = ''
    contractWorkingFiles.value = []
  }
  ElMessage.info(`已切换为${contractModeLabel(next)}，不适用的合同内容和文件已清空`)
}, { flush: 'sync' })

onMounted(() => {
  if (prototypeVariant === 'flow-template') {
    openDocument('create')
    attachmentsExpanded.value = false
  }
})
</script>

<style lang="scss" scoped>
.workspace { height: 100%; min-width: 1080px; display: flex; background: #eef1f5; color: #323a45; }
.task-rail { width: 220px; flex: none; padding: 18px 12px; background: #fff; border-right: 1px solid #e5e9ef; .rail-title { padding: 0 12px 14px; font-size: 15px; font-weight: 700; } button { width: 100%; border: 0; background: transparent; border-radius: 5px; padding: 11px 12px; margin-bottom: 4px; color: #667685; display: flex; align-items: center; gap: 9px; cursor: pointer; font-size: 14px; text-align: left; svg { width: 17px; } b { margin-left: auto; min-width: 20px; padding: 2px 6px; border-radius: 10px; color: #fff; background: #f45454; font-size: 11px; } &.active { color: #1687f8; background: #eaf4ff; font-weight: 600; } } .rail-tip { margin: 22px 10px; padding: 12px; color: #8592a0; background: #f7f9fb; border-radius: 6px; font-size: 12px; line-height: 1.65; } }
.stage { min-width: 0; flex: 1; height: 100%; display: flex; flex-direction: column; }
.business-tabs { height: 42px; display: flex; align-items: end; gap: 5px; padding: 0 10px; background: #dfe4ea; button { height: 34px; min-width: 130px; padding: 0 12px; border: 0; border-radius: 5px 5px 0 0; background: #f4f6f8; color: #667685; cursor: pointer; display: flex; gap: 10px; justify-content: space-between; align-items: center; &.active { color: #1687f8; background: #fff; } .tab-close { width: 14px; } } }
.approval-launch { align-self: center; margin-left: auto; margin-right: 4px; .approval-shortcut { min-width: 112px; width: 112px; height: 30px; padding: 0 12px; border: 1px solid #d7dee7; border-radius: 4px; background: #fff; color: #566472; display: inline-flex; align-items: center; justify-content: center; gap: 6px; white-space: nowrap; &:hover { border-color: #1687f8; color: #1687f8; } svg { width: 16px; flex: none; } span { white-space: nowrap; } } :deep(.el-badge__content) { top: 2px; right: 5px; } }
.list-page, .approval-center { margin: 10px; padding: 22px; background: #fff; min-height: 0; flex: 1; overflow: auto; }
.page-heading { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; h1 { margin: 0 0 5px; font-size: 20px; } p { margin: 0; color: #93a0ad; font-size: 13px; } }
.filters { display: flex; gap: 12px; margin-bottom: 16px; .el-input { width: 340px; margin-left: auto; } }
.approval-item { min-height: 78px; display: flex; gap: 14px; align-items: center; padding: 14px 18px; border: 1px solid #e7eaf0; border-radius: 6px; margin-bottom: 10px; cursor: pointer; transition: .2s; &:hover { border-color: #79bbff; box-shadow: 0 3px 12px rgba(24, 144, 255, .09); } .approval-icon { width: 42px; height: 42px; display: grid; place-items: center; border-radius: 6px; color: #1687f8; background: #eaf4ff; svg { width: 21px; } } .approval-copy { display: flex; flex: 1; flex-direction: column; gap: 7px; span { color: #8592a0; font-size: 12px; } } > svg { width: 15px; color: #a6afb8; } }
.approval-filters { display: grid; grid-template-columns: 180px 170px 300px minmax(260px, 1fr); gap: 10px; padding: 12px; margin-bottom: 14px; border: 1px solid #e7eaf0; border-radius: 5px; background: #f7f9fb; .el-select, .el-input, .el-date-editor { width: 100%; } }
.document-page { flex: 1; min-height: 0; display: flex; flex-direction: column; }
.document-header { min-height: 82px; flex: none; padding: 14px 24px; background: #fff; border-bottom: 1px solid #e6e9f2; display: flex; justify-content: space-between; align-items: center; .eyebrow { color: #1687f8; font-size: 12px; margin-bottom: 5px; } h1 { margin: 0; display: flex; align-items: center; gap: 10px; font-size: 20px; } p { margin: 5px 0 0; color: #8592a0; font-size: 12px; } }
.budget-create-header-actions { display: flex; align-items: center; gap: 8px; }
.document-layout { min-height: 0; flex: 1; display: flex; }
.document-scroll { flex: 1; min-width: 0; padding: 16px 18px 50px; overflow: auto; scroll-behavior: smooth; }
.audit-summary { max-width: 1400px; margin: 0 auto 12px; padding: 12px 16px; border: 1px solid #b8e8dc; background: #f1fbf8; border-radius: 6px; display: flex; justify-content: space-between; align-items: center; > div { display: flex; align-items: center; gap: 8px; } svg { width: 18px; color: #13a884; } .risk { color: #b66a00; svg { color: #e99516; } } }
.section-card { max-width: 1400px; margin: 0 auto 14px; padding: 20px 22px; background: #fff; border: 1px solid #e7eaf0; border-radius: 6px; scroll-margin-top: 16px; }
.section-heading { display: flex; align-items: center; gap: 12px; padding-bottom: 16px; margin-bottom: 17px; border-bottom: 1px solid #edf0f3; .section-number { color: #1687f8; font: 700 18px/1 monospace; } h2 { margin: 0; font-size: 15px; } .section-extra { margin-left: auto; display: flex; align-items: center; gap: 8px; :deep(.el-upload) { display: inline-flex; } } }
.budget-create-page { background: #fff; .document-header { min-height: 56px; padding: 9px 18px; } .document-header .eyebrow { display: none; } .document-header h1 { font-size: 18px; } .document-scroll { padding: 8px 16px 24px; background: #fff; } .section-card { box-sizing: border-box; width: 100%; max-width: 1600px; margin-bottom: 8px; padding: 12px 16px; border-color: #cfe0f5; border-radius: 6px; box-shadow: none; } .section-heading { min-height: 28px; margin-bottom: 10px; padding-bottom: 8px; gap: 10px; } .section-heading .section-number { width: 24px; height: 24px; border-radius: 50%; background: #1687f8; color: #fff; display: grid; place-items: center; font: 700 13px/1 sans-serif; } .section-heading h2 { font-size: 15px; } #plan .split-panel { grid-template-columns: minmax(440px, 2fr) minmax(620px, 3fr); gap: 16px; } #plan .plan-form, #plan .metrics { padding: 12px 14px; border: 1px solid #cfe0f5; border-radius: 5px; } #plan .metrics { padding-left: 14px; } #plan .plan-form > h3, #plan .metrics > h3 { color: #1678d2; } #goods { border-color: #f2d5ae; } #goods .section-number { background: #f39828; } #attachments { border-color: #ded2f5; } #attachments .section-number { background: #7759d6; } .attachment-layout { gap: 28px; } .bottom-actions { margin-top: 10px; padding: 10px; border: 0; } :deep(.el-form-item) { margin-bottom: 10px; } :deep(.el-form-item__label) { padding: 0 10px 0 0; line-height: 32px; } :deep(.el-input__wrapper), :deep(.el-select__wrapper), :deep(.el-input-number) { min-height: 32px; } }
.budget-create-page #basic :deep(.el-form-item) { display: grid; grid-template-columns: 112px minmax(0, 1fr); align-items: start; }
.budget-create-page #basic :deep(.el-form-item__content) { min-width: 0; }
.budget-create-page #basic :deep(.el-col-8) { max-width: 25%; flex: 0 0 25%; }
.budget-create-page #plan .plan-form :deep(.el-form-item) { display: block; align-items: start; }
.budget-create-page #plan .plan-form :deep(.el-form-item__content) { min-width: 0; }
.budget-create-page { min-width: 0; .document-header { min-height: 52px; padding: 8px 18px; border-bottom-color: #dfe6ef; } .document-scroll { padding: 8px 16px 18px; } .section-card { max-width: none; margin: 0 0 8px; padding: 10px 14px; border-width: 1px; border-radius: 6px; } .section-heading { min-height: 26px; margin-bottom: 8px; padding-bottom: 7px; } .section-heading .section-number { width: 22px; height: 22px; font-size: 12px; } .section-heading h2 { font-size: 14px; } #basic { padding-bottom: 6px; } #basic .type-fields { margin: 0; padding: 0; } #basic :deep(.el-row) { row-gap: 0; } #basic :deep(.el-col-8) { padding-right: 10px !important; padding-left: 10px !important; } #basic :deep(.el-form-item) { grid-template-columns: 104px minmax(0, 1fr); margin-bottom: 8px; } #basic :deep(.el-form-item__label) { font-size: 12px; } #plan .split-panel { grid-template-columns: minmax(390px, 45fr) minmax(480px, 55fr); gap: 14px; align-items: stretch; } #plan .plan-form, #plan .metrics { padding: 10px 12px; } #plan .plan-form > h3, #plan .metrics > h3 { margin-bottom: 9px; font-size: 13px; } #plan .plan-form :deep(.el-form-item) { grid-template-columns: 140px minmax(0, 1fr); margin-bottom: 7px; } #plan .plan-form :deep(.el-form-item__label) { font-size: 12px; white-space: nowrap; } #plan .metric-grid { grid-template-columns: repeat(3, minmax(0, 1fr)); gap: 0 20px; } #plan .metric-grid > div { min-height: 38px; padding: 5px 0; border-bottom: 1px dashed #e8edf3; border-radius: 0; background: transparent; display: grid; grid-template-columns: minmax(100px, 1fr) auto; align-items: center; } #plan .metric-grid > div.warning { background: transparent; } #plan .metric-grid > div span { font-size: 12px; } #plan .metric-grid > div strong { font-size: 15px; text-align: right; } #goods { padding-right: 10px; padding-left: 10px; } #goods .section-heading { padding-right: 4px; padding-left: 4px; } #goods :deep(.el-table) { font-size: 12px; } #goods :deep(.el-table .el-table__cell) { padding-top: 5px; padding-bottom: 5px; } #goods :deep(.el-input__wrapper), #goods :deep(.el-input-number) { min-height: 30px; } #attachments .attachment-layout { grid-template-columns: minmax(320px, .9fr) minmax(500px, 1.8fr); gap: 30px; } #attachments :deep(.el-upload-dragger) { height: 100px; padding: 18px 14px; } #attachments :deep(.el-textarea__inner) { min-height: 100px !important; } .bottom-actions { margin: 4px 0 0; padding: 8px 0 2px; } :deep(.el-button) { min-height: 32px; } }
.budget-create-page{#basic :deep(.el-form-item){grid-template-columns:132px minmax(0,1fr);align-items:start;min-height:40px;margin-bottom:10px}#basic :deep(.el-form-item__label){box-sizing:border-box;min-height:32px;padding:7px 14px 0 0;line-height:18px;white-space:normal;word-break:break-word}#basic :deep(.el-input),#basic :deep(.el-select),#basic :deep(.el-input-number){width:100%;min-width:0}#basic :deep(.el-input__inner),#basic :deep(.el-select__selected-item){overflow:hidden;text-overflow:ellipsis;white-space:nowrap}#basic .field-tip{grid-column:2;max-width:100%;margin-top:5px;line-height:18px;white-space:normal}#plan .split-panel{grid-template-columns:minmax(560px,58fr) minmax(420px,42fr)}#plan .plan-form :deep(.el-form-item){grid-template-columns:132px minmax(0,1fr);min-height:38px;margin-bottom:9px}#plan .plan-form :deep(.el-form-item__label){padding:7px 14px 0 0;line-height:18px;white-space:normal}#plan .plan-form :deep(.el-input),#plan .plan-form :deep(.el-select),#plan .plan-form :deep(.el-date-editor),#plan .plan-form :deep(.el-input-number){width:100%}#plan .metric-grid{grid-template-columns:repeat(2,minmax(0,1fr));gap:0 18px}}
.budget-create-page{#basic :deep(.el-form-item){display:block;min-height:0;margin-bottom:12px}#basic :deep(.el-form-item__label){display:block;min-height:0;padding:0 0 6px;line-height:20px;white-space:normal}#basic .field-tip{display:block;max-width:100%;margin-top:5px;overflow:hidden;color:#8795a5;line-height:18px;text-overflow:ellipsis;white-space:nowrap}#basic :deep(.el-row){align-items:start}#basic .type-fields{margin-top:0}}
.budget-create-page{#plan .split-panel{grid-template-columns:minmax(620px,65fr) minmax(330px,35fr);gap:22px;align-items:start}#plan .plan-form,#plan .metrics{padding:0;border:0;border-radius:0}#plan .metrics{padding-left:22px;border-left:1px solid #e4eaf1}#plan .plan-form :deep(.el-col-12){max-width:50%;flex:0 0 50%}#plan .plan-form :deep(.el-col-24){max-width:100%;flex:0 0 100%}#plan .plan-form :deep(.el-form-item){display:block;min-height:0;margin-bottom:12px}#plan .plan-form :deep(.el-form-item__label){display:block;padding:0 0 6px;line-height:20px;white-space:normal}#plan .metric-grid{grid-template-columns:1fr;gap:0}#plan .metric-grid>div{grid-template-columns:minmax(130px,1fr) auto;min-height:42px;padding:7px 0}#plan .metric-grid>div strong{font-size:16px}.flow-module-control{width:164px}}
.document-page{.document-header{min-height:72px;padding:12px 22px}.document-header h1{font-size:19px}.document-meta{display:flex;flex-wrap:wrap;gap:6px 0!important;margin-top:8px!important}.document-meta span{position:relative;margin-right:18px;padding-right:18px;color:#6f7d8b}.document-meta span:not(:last-child)::after{position:absolute;right:0;color:#c9d0d8;content:'|'}.document-scroll{padding-top:10px}.section-card{margin-bottom:8px;padding:12px 16px;border:1px solid #e1e7ed;border-radius:5px;box-shadow:none}.section-heading{min-height:28px;margin-bottom:10px;padding-bottom:9px}.section-heading .section-number{width:24px;height:24px;display:grid;place-items:center;border-radius:50%;background:#1687f8;color:#fff;font:700 13px/1 sans-serif}.section-heading h2{font-size:15px}:deep(.el-table){font-size:12px}:deep(.el-table .el-table__cell){padding-top:6px;padding-bottom:6px}:deep(.el-table th.el-table__cell){background:#f4f6f9;color:#526170}.bottom-actions{border:0;box-shadow:none}}
.budget-readonly-page,.contract-detail-page{.section-card{padding-bottom:13px}:deep(.el-form-item){display:grid;grid-template-columns:128px minmax(0,1fr);align-items:center;min-height:34px;margin-bottom:4px}:deep(.el-form-item__label){height:auto;padding:0 12px 0 0;line-height:20px;color:#657485;font-weight:600}:deep(.el-input__wrapper),:deep(.el-select__wrapper){min-height:30px;padding:0;background:transparent!important;box-shadow:none!important}:deep(.el-input.is-disabled .el-input__inner),:deep(.el-textarea.is-disabled .el-textarea__inner){color:#303944;-webkit-text-fill-color:#303944}:deep(.el-input__inner){font-size:13px}:deep(.el-textarea__inner){padding:7px 9px;background:#fafbfd;box-shadow:0 0 0 1px #edf0f3 inset}.split-panel{gap:24px}.metrics{padding-left:20px}.metric-grid>div{min-height:50px;padding:9px 12px;background:#fafbfd}.secondary-contract-info :deep(.el-collapse-item__header),.detail-budget-collapse :deep(.el-collapse-item__header){min-height:40px;background:#f7f9fb}}
.contract-detail-page{.contract-key-summary{margin-bottom:8px;border-color:#e1e7ed;box-shadow:none}.contract-key-summary>div{padding:12px 16px}.contract-key-summary strong{font-size:18px}.contract-content-subtitle{margin-top:4px}.contract-readonly-section :deep(.el-input__wrapper){background:transparent;box-shadow:none}}
.budget-create-page,.contract-edit-page{.section-card,.contract-block{box-shadow:none}.section-heading .section-number,.contract-block h3 b{border-radius:50%}.bottom-actions{position:relative;z-index:2}}
.contract-edit-page{.contract-block h3 b{width:24px;height:24px;display:grid;place-items:center;flex:none;background:#1687f8;color:#fff}.contract-block h3 b::before{font-size:12px;color:#fff}.contract-block h3{min-height:28px;margin-bottom:10px;padding-bottom:9px}.contract-block{margin-bottom:8px;padding:13px 16px 5px;border-color:#e1e7ed}.contract-directory-bar{min-height:44px}.document-header{min-height:64px}}
.document-page{font-size:13px;line-height:1.5}.document-page :deep(.el-form-item__label){font-size:13px}.document-page :deep(.el-input__inner),.document-page :deep(.el-select__selected-item),.document-page :deep(.el-textarea__inner){font-size:13px}.document-page .section-heading>.section-number{width:24px!important;height:24px!important;display:grid!important;place-items:center!important;flex:none;border-radius:50%!important;background:#1687f8!important;color:#fff!important;font:700 13px/1 sans-serif!important}.document-page .section-heading{gap:10px}.document-page .section-heading h2{font-size:15px;line-height:24px}.document-page .section-card+.section-card{margin-top:0}
.contract-detail-page,.contract-edit-page{position:relative}.contract-detail-page>.document-scroll,.contract-edit-page>.document-scroll{box-sizing:border-box;margin-right:172px}.contract-side-directory{position:absolute;z-index:9;top:73px;right:0;bottom:0;width:164px;box-sizing:border-box;padding:10px 8px;border-left:1px solid #dfe5ec;background:#fff;overflow:auto}.contract-edit-page>.contract-side-directory{top:65px}.side-directory-title{min-height:34px;display:flex;align-items:center;padding:0 9px;margin-bottom:5px;border-radius:4px;background:#f3f6f9;color:#566575;font-size:12px;font-weight:600}.contract-side-directory button{width:100%;min-height:40px;display:grid;grid-template-columns:26px minmax(0,1fr);align-items:center;padding:0 8px;margin-top:4px;border:0;border-left:3px solid transparent;background:transparent;color:#667685;text-align:left;cursor:pointer}.contract-side-directory button span{font-size:11px}.contract-side-directory button b{overflow:hidden;font-size:12px;font-weight:500;text-overflow:ellipsis;white-space:nowrap}.contract-side-directory button:hover,.contract-side-directory button.active{border-left-color:#1687f8;background:#edf6ff;color:#1687f8}.contract-operation-bar{max-width:none;margin-right:0;margin-left:0;box-shadow:none}.contract-operation-bar>span,.contract-operation-bar>strong{color:#526170;font-size:13px}
.flow-module-control{width:164px}.flow-module-control .flow-control-item{min-height:40px}.flow-module-control .flow-control-item b{font-size:12px}.budget-create-page .section-card,.budget-readonly-page .section-card,.contract-detail-page .section-card,.contract-edit-page .contract-block{margin-bottom:8px}.budget-create-page :deep(.el-form-item),.contract-edit-page :deep(.el-form-item){margin-bottom:12px}.budget-create-page :deep(.el-form-item__label),.contract-edit-page :deep(.el-form-item__label){padding-bottom:6px;line-height:20px}
.goods-actions { display: flex; flex-flow: row nowrap; align-items: center; gap: 8px; white-space: nowrap; :deep(.el-upload) { display: flex !important; width: auto !important; } :deep(.el-button) { margin: 0; } }
.field-tip { width: 100%; display: flex; align-items: flex-start; gap: 5px; margin-top: 7px; color: #8b97a3; font-size: 12px; line-height: 1.5; svg { width: 14px; flex: none; margin-top: 2px; color: #1687f8; } }
.type-fields { margin-top: 2px; padding-top: 4px; }
.split-panel { display: grid; grid-template-columns: minmax(460px, 1.15fr) minmax(390px, .85fr); gap: 22px; h3 { margin: 0 0 15px; font-size: 14px; small { margin-left: 8px; color: #9aa5af; font-weight: 400; } } .metrics { padding-left: 22px; border-left: 1px solid #edf0f3; } }
.metric-grid { display: grid; grid-template-columns: repeat(2, 1fr); gap: 10px; > div { min-height: 87px; padding: 13px; background: #f7f9fb; border-radius: 5px; display: flex; flex-direction: column; gap: 5px; span, small { color: #8995a1; font-size: 12px; } .metric-label { display: flex; align-items: center; gap: 5px; svg { width: 14px; color: #8b98a5; cursor: help; } } strong { color: #27313a; font-size: 19px; } &.warning { background: #fff8eb; strong, small { color: #c97800; } } } }
.file-row { display: flex; align-items: center; gap: 12px; padding: 12px; border: 1px solid #e7eaf0; border-radius: 5px; > svg { width: 26px; color: #1687f8; } > div { display: flex; flex: 1; flex-direction: column; gap: 4px; span { color: #9aa5af; font-size: 12px; } } }
.comment-history { display: flex; gap: 12px; margin-bottom: 16px; padding: 14px; border: 1px solid #e7eaf0; border-radius: 5px; background: #fafbfd; .comment-avatar { width: 34px; height: 34px; flex: none; display: grid; place-items: center; border-radius: 50%; color: #1687f8; background: #e8f4ff; font-weight: 600; } > div:last-child { flex: 1; b { margin-right: 10px; font-size: 13px; } span { color: #9aa5af; font-size: 12px; } p { margin: 8px 0 0; color: #596775; font-size: 13px; } } }
.current-task-card { display: flex; align-items: center; gap: 14px; padding: 16px 18px; border: 1px solid #f0cf91; border-radius: 6px; background: #fffaf0; .task-state-icon { width: 42px; height: 42px; display: grid; place-items: center; border-radius: 50%; color: #c97800; background: #fff0ce; svg { width: 21px; } } .task-main { flex: 1; display: flex; flex-direction: column; gap: 4px; span, small { color: #87939e; font-size: 12px; } strong { font-size: 15px; } } .task-stat { width: 180px; display: grid; grid-template-columns: 1fr auto; gap: 5px 10px; span { color: #87939e; font-size: 12px; } strong { color: #c97800; } .el-progress { grid-column: 1 / 3; } } }
.flat-approval-list { margin-top: 14px; border: 1px solid #e3e7ec; border-radius: 6px; overflow: hidden; }
.flat-node { min-height: 66px; padding: 11px 15px; display: flex; align-items: center; gap: 13px; border-bottom: 1px solid #edf0f3; &:last-child { border-bottom: 0; } &.active { background: #fff9ed; box-shadow: inset 3px 0 #e99516; } &.waiting { background: #fafbfd; } .flat-order { width: 28px; height: 28px; flex: none; display: grid; place-items: center; border-radius: 50%; color: #8995a1; background: #edf0f3; font-size: 11px; svg { width: 20px; color: #13a884; } } &.active .flat-order { color: #e99516; background: #fff0ce; svg { color: #e99516; } } .flat-node-main { flex: 1; display: flex; flex-direction: column; gap: 5px; strong { font-size: 13px; } span { color: #87939e; font-size: 12px; } } .flat-node-result { display: flex; align-items: center; gap: 12px; small { color: #98a3ad; } } }
.flow-toolbar { display: flex; justify-content: space-between; align-items: center; margin: 16px 0 10px; .flow-legend { display: flex; gap: 16px; color: #87939e; font-size: 12px; span { display: flex; align-items: center; gap: 5px; } i { width: 8px; height: 8px; border-radius: 50%; background: #c5ccd3; &.done { background: #13a884; } &.active { background: #e99516; } } } }
.approval-flow { display: flex; flex-direction: column; gap: 10px; }
.flow-group { border: 1px solid #e3e7ec; border-radius: 6px; overflow: hidden; &.active { border-color: #f0cf91; } .group-header { width: 100%; min-height: 64px; padding: 11px 15px; border: 0; background: #fafbfd; display: flex; align-items: center; gap: 12px; text-align: left; cursor: pointer; .group-no { width: 32px; color: #98a3ad; font: 700 13px monospace; } > div { flex: 1; display: flex; flex-direction: column; gap: 4px; strong { font-size: 14px; } small { color: #939ea8; } } > svg { width: 15px; color: #8995a1; transition: transform .2s; &.collapsed { transform: rotate(-90deg); } } } .group-body { padding: 13px 15px 15px; border-top: 1px solid #edf0f3; } }
.parallel-tip { display: flex; align-items: center; gap: 7px; margin-bottom: 12px; padding: 9px 12px; color: #90620c; background: #fff8e9; border-radius: 4px; font-size: 12px; svg { width: 16px; } }
.branch-grid { display: grid; grid-template-columns: 1fr; gap: 10px; &.parallel { grid-template-columns: repeat(3, minmax(0, 1fr)); } }
.flow-branch { border: 1px solid #e4e8ed; border-radius: 5px; overflow: hidden; &.active { border-color: #e9b95d; } .branch-head { display: flex; justify-content: space-between; align-items: center; padding: 9px 11px; background: #f7f9fb; color: #596775; font-size: 12px; font-weight: 600; } }
.flow-node { min-height: 66px; padding: 10px 11px; display: flex; align-items: center; gap: 9px; border-top: 1px solid #edf0f3; .node-marker { width: 22px; height: 22px; flex: none; svg { width: 20px; color: #13a884; } span { display: block; width: 12px; height: 12px; margin: 4px; border: 2px solid #c7ced5; border-radius: 50%; } } &.active .node-marker svg { color: #e99516; } .node-info { flex: 1; display: flex; flex-direction: column; gap: 3px; b { font-size: 12px; } span, small { color: #8e99a4; font-size: 11px; } } }
.record-toolbar { display: flex; justify-content: flex-end; margin-bottom: 12px; }
.approval-timeline { margin: 0 0 18px 15px; border-left: 1px solid #dfe4e9; }
.timeline-item { position: relative; display: flex; gap: 11px; padding: 0 0 18px 18px; .timeline-dot { position: absolute; top: 12px; left: -5px; width: 9px; height: 9px; border-radius: 50%; background: #1687f8; box-shadow: 0 0 0 4px #e8f4ff; } .comment-avatar { width: 34px; height: 34px; flex: none; display: grid; place-items: center; border-radius: 50%; color: #1687f8; background: #e8f4ff; font-weight: 600; } .timeline-content { flex: 1; > div { display: flex; align-items: center; gap: 8px; b { font-size: 13px; } span { margin-left: auto; color: #9aa5af; font-size: 12px; } } p { margin: 7px 0 0; color: #596775; font-size: 13px; } } &.system .timeline-dot { background: #99a4ae; box-shadow: 0 0 0 4px #edf0f3; } }
.comment-title { margin: 0 0 9px; font-size: 13px; font-weight: 600; }
.attachment-layout { display: grid; grid-template-columns: minmax(360px, .85fr) minmax(480px, 1.75fr); gap: 32px; h3 { margin: 0 0 12px; font-size: 13px; font-weight: 600; } :deep(.el-upload-dragger) { height: 132px; padding: 29px 16px; background: #fff; } .upload-action { display: flex; align-items: center; justify-content: center; gap: 7px; color: #1687f8; svg { width: 18px; } } .upload-help { margin-top: 15px; color: #98a3ad; font-size: 12px; } :deep(.el-textarea__inner) { min-height: 132px !important; } }
.bottom-actions { max-width: 1400px; margin: 2px auto 0; padding: 18px 22px; background: #fff; border: 1px solid #e7eaf0; border-radius: 6px; display: flex; justify-content: center; gap: 8px; }
.ai-recognition-panel { padding: 14px; border: 1px solid #d8e9fb; border-radius: 6px; background: #f7fbff; .ai-recognition-title { display: flex; align-items: center; justify-content: space-between; margin-bottom: 12px; } }
.ai-recognition-grid { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 10px; margin-bottom: 12px; > div { display: flex; flex-direction: column; gap: 5px; padding: 10px 12px; border-radius: 4px; background: #fff; span { color: #8b97a3; font-size: 12px; } b { color: #3d4852; font-size: 13px; } } }
.dialog-comment { margin-top: 16px; }
.product-dialog-toolbar { display: grid; grid-template-columns: minmax(320px, 1fr) 190px 130px; gap: 10px; align-items: center; margin-bottom: 12px; .el-input, .el-select { width: 100%; } }
.product-selection-summary { display: flex; align-items: center; gap: 4px; margin-top: 12px; color: #667685; font-size: 13px; b { color: #1687f8; } span { margin-left: auto; color: #98a3ad; font-size: 12px; } }
.record-budget-code { margin-bottom: 12px; color: #667685; font-size: 13px; }
.precondition-list { display: flex; flex-direction: column; gap: 0; margin-top: 14px; border: 1px solid #e7eaf0; border-radius: 5px; > div { display: flex; align-items: center; gap: 9px; padding: 11px 13px; border-bottom: 1px solid #edf0f3; &:last-child { border-bottom: 0; } svg { width: 18px; &.passed { color: #13a884; } &.failed { color: #e99516; } } span { flex: 1; } b { color: #87939e; font-size: 12px; } } }
:global(.contract-prototype-dialog .el-dialog__body) { max-height: calc(94vh - 132px); padding: 12px 14px; overflow: auto; background: #f3f5f7; }
:global(.contract-prototype-dialog .el-dialog__footer) { padding: 12px 18px; border-top: 1px solid #e7eaf0; }
.contract-prototype-content { display: flex; flex-direction: column; gap: 10px; counter-reset: contract-section; }
.document-page :deep(.section-heading){gap:10px;min-height:28px;margin-bottom:10px;padding-bottom:9px}.document-page :deep(.section-heading>.section-number){width:24px!important;height:24px!important;display:grid!important;place-items:center!important;flex:none!important;border-radius:50%!important;background:#1687f8!important;color:#fff!important;font:700 13px/1 sans-serif!important}.document-page :deep(.section-heading>h2){font-size:15px;line-height:24px}
.budget-readonly-page,.contract-detail-page{article :deep(.el-form-item){display:block;min-height:0;margin-bottom:12px}article :deep(.el-form-item__label){display:block;min-height:0;padding:0 0 6px;line-height:20px;color:#5f6f80;font-size:13px;font-weight:400}article :deep(.el-input__wrapper),article :deep(.el-select__wrapper){min-height:32px;padding:1px 11px;background:#f5f7fa!important;box-shadow:0 0 0 1px #e2e7ed inset!important}article :deep(.el-input.is-disabled .el-input__inner){color:#596775;-webkit-text-fill-color:#596775}article :deep(.el-textarea.is-disabled .el-textarea__inner){background:#f5f7fa;color:#596775;-webkit-text-fill-color:#596775;box-shadow:0 0 0 1px #e2e7ed inset}.readonly-link-field{min-height:32px;box-sizing:border-box;padding:5px 11px;border:1px solid #e2e7ed;border-radius:4px;background:#f5f7fa}}
.contract-edit-page { position: fixed; z-index: 900; top: 64px; right: 0; bottom: 0; left: 220px; background: #eef1f5; .document-header { box-sizing: border-box; width: calc(100% - 18px); max-width: none; min-height: 82px; margin: 0 9px; padding: 13px 20px; border: 1px solid #e7eaf0; border-top: 0; border-radius: 0 0 6px 6px; } .document-scroll { padding-top: 10px; } .contract-block,.contract-directory-bar,.edit-scope-panel,.bottom-actions{max-width:none;margin-right:10px;margin-left:10px} }
.contract-detail-page{.section-card,.contract-key-summary,.approval-focus-panel,.module-directory-bar{max-width:none}.document-scroll{padding-right:12px;padding-left:12px}}
.edit-scope-panel,.approval-focus-panel{max-width:1400px;box-sizing:border-box;margin:0 auto 10px;padding:10px 14px;display:flex;align-items:center;gap:18px;border:1px solid #d9e8f7;border-left:4px solid #1687f8;border-radius:5px;background:#f5faff;color:#667685;font-size:12px}.edit-scope-panel strong,.approval-focus-panel strong{color:#303b47;font-size:13px}.edit-scope-panel span,.approval-focus-panel span{padding-left:14px;border-left:1px solid #d8e2ec}.approval-focus-panel{border-color:#f1d8a9;border-left-color:#e99516;background:#fffaf0}.budget-key-summary{max-width:1400px;box-sizing:border-box;margin:0 auto 10px;display:grid;grid-template-columns:repeat(4,1fr);border:1px solid #e2e8ef;border-radius:6px;background:#fff}.budget-key-summary>div{padding:14px 18px;border-right:1px solid #e8edf3}.budget-key-summary>div:last-child{border-right:0}.budget-key-summary span,.budget-key-summary strong{display:block}.budget-key-summary span{margin-bottom:7px;color:#83909e;font-size:12px}.budget-key-summary strong{font-size:20px}.approval-bottom-actions{margin-top:10px}.contract-edit-scope{margin-top:0}
.contract-key-summary { box-sizing: border-box; width: 100%; max-width: 1400px; margin: 0 auto 10px; padding: 13px 18px; border: 1px solid #e2e7ed; border-radius: 6px; background: #fff; display: grid; grid-template-columns: repeat(5, minmax(0, 1fr)); gap: 0; > div { min-width: 0; padding: 0 18px; border-right: 1px solid #edf0f3; display: flex; flex-direction: column; gap: 5px; &:first-child { padding-left: 0; } &:last-child { border-right: 0; } } span { color: #8995a1; font-size: 11px; } strong { overflow: hidden; color: #323a45; font-size: 15px; font-weight: 600; text-overflow: ellipsis; white-space: nowrap; } .attention-value { color: #d27d0c; } }
.contract-finance-summary { grid-template-columns: repeat(3, minmax(0, 1fr)); }
.contract-detail-page .document-header .eyebrow { display: flex; align-items: center; gap: 6px; span { padding: 2px 7px; border-radius: 3px; background: #edf5fd; } }
.contract-detail-page { .document-scroll { padding: 12px 18px 42px; } .section-card { margin-bottom: 10px; padding: 15px 18px; } .section-heading { margin-bottom: 12px; padding-bottom: 11px; } :deep(.el-descriptions__cell) { padding: 8px 11px !important; } :deep(.el-table .el-table__cell) { padding-top: 7px; padding-bottom: 7px; } }
.contract-approval-view { .approval-summary-view { border-color: #f0d6a4; } .contract-detail-info, .section-card:nth-of-type(4) { border-color: #f0d6a4; } }
.contract-readonly-section { :deep(.el-form-item) { margin-bottom: 12px; } :deep(.el-input__wrapper) { background: #f5f7fa; box-shadow: 0 0 0 1px #e4e7ed inset; } }
.detail-related-summary { display: flex; align-items: center; gap: 26px; padding: 11px 12px; color: #667685; font-size: 12px; }
.detail-budget-collapse :deep(.el-collapse-item__content) { padding: 16px 4px 10px; }
.detail-budget-collapse .source-budget-attachments { margin-top: 0; }
.contract-readonly-section > .contract-content-subtitle { min-height: 46px; margin: 0 0 16px; padding: 0 16px; border: 1px solid #edf0f3; border-radius: 4px; background: #f8fafc; display: flex; align-items: center; color: #323a45; font-size: 15px; font-weight: 600; }
.contract-readonly-section > .detail-file-subtitle { margin-top: 16px; }
.detail-file-subtitle { margin-top: 16px; }
.contract-attention-summary { box-sizing: border-box; width: 100%; max-width: 1400px; min-height: 44px; margin: 0 auto 10px; padding: 10px 14px; border: 1px solid #f1d7a6; border-radius: 6px; background: #fffaf0; display: flex; align-items: center; justify-content: space-between; > div { display: flex; align-items: center; gap: 9px; } svg { width: 17px; color: #d98a13; } strong { color: #9a650d; font-size: 13px; } span { color: #7f8c98; font-size: 12px; } }
.contract-directory-bar { position: sticky; z-index: 8; top: -14px; box-sizing: border-box; width: 100%; max-width: 1400px; min-height: 48px; margin: 0 auto 10px; padding: 8px 14px; border: 1px solid #dfe5ec; border-radius: 0 0 6px 6px; background: #fff; box-shadow: 0 3px 10px rgba(40, 55, 72, .08); display: flex; align-items: center; justify-content: space-between; :deep(.el-button svg) { width: 13px; margin-left: 7px; } }
.module-directory-bar { position: sticky; z-index: 8; top: -14px; box-sizing: border-box; width: 100%; max-width: 1400px; min-height: 46px; margin: 0 auto 10px; padding: 7px 14px; border: 1px solid #dfe5ec; border-radius: 0 0 6px 6px; background: #fff; box-shadow: 0 3px 10px rgba(40, 55, 72, .08); display: flex; align-items: center; justify-content: space-between; :deep(.el-button svg) { width: 13px; margin-left: 7px; } }
.budget-flow-template { .document-scroll { padding-right: 10px; } .auxiliary-module-control { min-height: 56px; padding: 0 14px; border: 1px solid #e2e7ed; border-radius: 5px; background: #f8fafc; display: flex; align-items: center; justify-content: space-between; > div { display: flex; flex-direction: column; gap: 4px; } b { color: #344150; font-size: 13px; } span { color: #8a97a5; font-size: 12px; } svg { width: 14px; margin-left: 5px; transition: transform .2s; } svg.rotated { transform: rotate(180deg); } .attachment-layout { margin-top: 10px; } } }
.flow-module-control { width: 172px; flex: none; padding: 10px 8px; border-left: 1px solid #dfe5ec; background: #fff; transition: width .2s; &.collapsed { width: 48px; padding: 10px 5px; } button { border: 0; cursor: pointer; } .flow-control-toggle { width: 100%; min-height: 34px; padding: 0 8px; border-radius: 4px; background: #f3f6f9; color: #566575; display: flex; align-items: center; gap: 7px; font-size: 12px; svg { width: 15px; flex: none; } svg:last-child { margin-left: auto; transition: transform .2s; } svg.rotated { transform: rotate(90deg); } } .flow-control-item { width: 100%; min-height: 40px; margin-top: 5px; padding: 0 8px; border-left: 3px solid transparent; background: transparent; color: #667685; display: grid; grid-template-columns: 24px minmax(0, 1fr); align-items: center; text-align: left; span { font-size: 11px; } b { overflow: hidden; font-size: 12px; font-weight: 500; text-overflow: ellipsis; white-space: nowrap; } &:hover, &.active { border-left-color: #1687f8; background: #edf6ff; color: #1687f8; } } .flow-control-aux { margin-top: 12px; padding: 12px 5px 0; border-top: 1px solid #edf0f3; display: flex; align-items: center; justify-content: space-between; gap: 6px; color: #7e8b98; font-size: 11px; } }
.budget-subsection-title { margin: 2px 0 10px; padding-left: 9px; border-left: 3px solid #9fc9f3; color: #4a5968; font-size: 13px; font-weight: 600; }
.budget-subsection-title.transaction-title { margin-top: 2px; }
.flow-status-reminders { margin-top: 14px; padding: 12px 5px 0; border-top: 1px solid #e8edf2; > div { display: flex; align-items: center; justify-content: space-between; margin-bottom: 5px; color: #526170; font-size: 12px; } > div span { min-width: 20px; padding: 1px 6px; border-radius: 9px; background: #fff1db; color: #c97800; font-size: 10px; text-align: center; } > button { width: 100%; min-height: 34px; padding: 6px 4px; border-bottom: 1px dashed #e8edf2; background: transparent; color: #697887; font-size: 11px; line-height: 1.45; text-align: left; } > button::before { display: inline-block; width: 6px; height: 6px; margin-right: 7px; border-radius: 50%; background: #e89a23; content: ''; } }
.flow-module-control { width: 300px; flex: 0 0 300px; box-sizing: border-box; padding: 10px 12px; border-left: 0; background: #eef1f5; overflow-y: auto; &.collapsed { width: 48px; flex-basis: 48px; padding: 10px 5px; background: #fff; } }
.flow-revenue-summary, .flow-navigation-card, .flow-status-reminders { box-sizing: border-box; margin: 0 0 10px; padding: 13px 14px; border: 1px solid #dfe6ed; border-radius: 6px; background: #fff; }
.flow-revenue-summary { header { display: flex; align-items: center; justify-content: space-between; margin-bottom: 8px; color: #3d4a57; font-size: 13px; } header span { color: #98a3ae; font-size: 10px; } > div { min-height: 40px; display: grid; grid-template-columns: minmax(0, 1fr) auto; align-items: center; gap: 14px; border-bottom: 1px dashed #e7ecf1; } > div span { min-width: 0; color: #7b8997; font-size: 12px; line-height: 18px; } > div b { color: #303b47; font-size: 15px; white-space: nowrap; } > button { width: 100%; min-height: 34px; padding: 9px 0 0; background: transparent; color: #1687f8; font-size: 12px; text-align: right; } }
.flow-navigation-card { padding: 10px 8px; .flow-control-toggle { background: transparent; font-weight: 600; } .flow-control-item { grid-template-columns: 25px minmax(0, 1fr) auto; column-gap: 5px; } .flow-control-item b { overflow: visible; font-size: 12px; text-overflow: clip; white-space: normal; line-height: 17px; } .flow-control-item em { font-size: 10px; font-style: normal; white-space: nowrap; } .flow-control-item em.done { color: #19a778; } .flow-control-item em.warning { color: #cf7900; } .flow-control-item em.empty { color: #98a3ad; } }
.flow-status-reminders { margin-top: 0; padding: 13px 14px; border-top: 1px solid #dfe6ed; }
.flow-module-control .collapsed-toggle { padding: 0; justify-content: center; svg:last-child { display: none; } }
.budget-approval-summary { box-sizing: border-box; width: 100%; margin: 0 0 10px; padding: 16px 20px; border: 1px solid #dfe6ed; border-radius: 7px; background: #fff; display: grid; grid-template-columns: minmax(420px, 1.25fr) minmax(520px, 1fr); align-items: center; gap: 24px; }
.approval-document-header { box-sizing: border-box; min-height: 168px; margin: 10px 18px 0; padding: 12px 16px 0; border: 1px solid #e1e7ed; border-radius: 5px; display: block; }
.budget-approval-header-content { width: 100%; .eyebrow { margin-bottom: 6px; } }
.approval-header-title { display: flex; align-items: center; justify-content: space-between; margin-bottom: 8px; h1 { font-size: 19px; } }
.approval-header-body { display: block; }
.approval-summary-identity { min-width: 0; > div { display: flex; align-items: center; gap: 10px; } .approval-summary-code { max-width: 260px; overflow: hidden; color: #536170; font-size: 13px; font-weight: 600; text-overflow: ellipsis; white-space: nowrap; } p { margin: 7px 0 10px; color: #6d7a88; font-size: 12px; } p strong { color: #4e5c6a; font-weight: 500; } p i { display: inline-block; height: 14px; margin: 0 16px; border-left: 1px solid #d9dfe6; vertical-align: middle; } }
.approval-summary-metrics { min-width: 0; min-height: 45px; border-top: 1px solid #e7ebf0; background: transparent; display: grid; grid-template-columns: repeat(3, minmax(0, 260px)); align-items: center; > div { min-width: 0; padding: 0 22px; border-right: 1px solid #e4e8ed; display: flex; align-items: baseline; gap: 10px; } > div:first-child { padding-left: 0; } > div:last-child { border-right: 0; } span { color: #8793a0; font-size: 12px; white-space: nowrap; } strong { color: #303b47; font-size: 16px; font-weight: 600; white-space: nowrap; } strong.positive { color: #16a34a; } }
.budget-detail-view, .budget-approval-view {
  #basic :deep(.el-col-6), #basic :deep(.el-col-8) { max-width: 25%; flex: 0 0 25%; }
  #basic :deep(.el-row), #plan .plan-form :deep(.el-row) { row-gap: 0; }
  #basic :deep(.el-form-item), #plan .plan-form :deep(.el-form-item) { display: block; min-height: 0; margin-bottom: 12px; border: 0; overflow: visible; }
  #basic :deep(.el-form-item__label), #plan .plan-form :deep(.el-form-item__label) { display: block; box-sizing: border-box; min-height: 0; margin: 0; padding: 0 0 6px; background: transparent; color: #5f6f80; line-height: 20px; font-size: 12px; font-weight: 400; }
  #basic :deep(.el-form-item__content), #plan .plan-form :deep(.el-form-item__content) { display: block; min-width: 0; min-height: 0; padding: 0; background: transparent; }
  #basic :deep(.el-form-item.is-required .el-form-item__label::before), #plan :deep(.el-form-item.is-required .el-form-item__label::before) { display: none; }
  article :deep(.el-input__wrapper), article :deep(.el-select__wrapper), article :deep(.el-input-number), article :deep(.el-textarea__inner) { min-height: 34px; padding-right: 11px; padding-left: 11px; border: 0; border-radius: 4px; background: #f5f7fa !important; box-shadow: 0 0 0 1px #e1e7ee inset !important; }
  article :deep(.el-input-number) { width: 100%; }
  article :deep(.el-input-number .el-input__wrapper) { padding-right: 11px; padding-left: 11px; }
  article :deep(.el-input-number__decrease), article :deep(.el-input-number__increase), article :deep(.el-select__caret) { display: none; }
  article :deep(.el-input__inner), article :deep(.el-select__selected-item), article :deep(.el-textarea__inner) { color: #4f5d6b !important; font-weight: 400; -webkit-text-fill-color: #4f5d6b !important; }
  #basic .field-tip { display: none; }
  #plan .plan-form :deep(.el-textarea__inner) { min-height: 72px !important; padding-top: 8px; resize: none; }
  .detail-approval-info .approval-extra-info { margin-top: 12px; }
}
@media (max-width: 1500px) {
  .budget-approval-summary { grid-template-columns: minmax(360px, 1fr) minmax(460px, 1fr); gap: 16px; }
  .approval-summary-metrics > div { padding-right: 10px; padding-left: 10px; }
  .approval-summary-metrics > div:first-child { padding-left: 0; }
}
@media (max-width: 1440px) {
  .flow-module-control { width: 260px; flex-basis: 260px; padding-right: 10px; padding-left: 10px; }
  .flow-revenue-summary, .flow-navigation-card, .flow-status-reminders { padding-right: 12px; padding-left: 12px; }
}

/* Contract pages follow the budget create/detail/approval master layouts. */
.contract-edit-page,
.contract-detail-page {
  > .document-scroll { margin-right: 300px; }
  > .contract-side-directory { width: 300px; padding: 10px 12px; background: #eef1f5; }
  > .contract-side-directory .side-directory-title { min-height: 38px; margin-bottom: 8px; padding: 0 12px; background: #fff; border: 1px solid #dfe6ed; border-radius: 6px; }
  > .contract-side-directory > button { min-height: 42px; margin-top: 0; padding: 0 12px; background: #fff; }
  > .contract-side-directory > button + button { border-top: 1px solid #edf1f5; }
  .contract-status-reminders { margin-top: 12px; }
}
.contract-edit-page {
  background: #fff;
  > .document-header { width: 100%; min-height: 56px; margin: 0; padding: 9px 18px; border: 0; border-bottom: 1px solid #dfe6ef; border-radius: 0; }
  > .document-header .eyebrow { display: none; }
  > .document-header h1 { font-size: 18px; }
  > .document-scroll { padding: 8px 16px 24px; background: #fff; }
  > .contract-side-directory { top: 57px; }
  .contract-block { box-sizing: border-box; width: 100%; max-width: none; margin: 0 0 8px; padding: 10px 14px 5px; border-color: #cfe0f5; border-radius: 6px; }
  .contract-block h3 { min-height: 26px; margin-bottom: 8px; padding-bottom: 7px; gap: 10px; font-size: 14px; }
  .contract-block h3 b { width: 22px; height: 22px; font-size: 12px; }
  :deep(.el-form-item) { margin-bottom: 10px; }
  :deep(.el-form-item__label) { padding-bottom: 6px; color: #5f6f80; font-size: 12px; line-height: 20px; }
  :deep(.el-input__wrapper), :deep(.el-select__wrapper), :deep(.el-input-number) { min-height: 32px; }
  .contract-block :deep(.el-row > .el-col-8) { max-width: 25%; flex: 0 0 25%; }
  .contract-block :deep(.el-row > .el-col-12) { max-width: 50%; flex: 0 0 50%; }
  .contract-block :deep(.el-row > .el-col-16) { max-width: 50%; flex: 0 0 50%; }
  .contract-block :deep(.el-row > .el-col-24) { max-width: 100%; flex: 0 0 100%; }
  .bottom-actions { width: 100%; max-width: none; box-sizing: border-box; margin: 4px 0 0; padding: 10px 0 2px; border: 0; }
}
.contract-detail-page {
  > .document-header { min-height: 72px; padding: 12px 22px; }
  > .document-header .budget-create-header-actions { align-self: center; }
  > .document-scroll { padding: 10px 16px 42px; }
  > .contract-side-directory { top: 73px; }
  .section-card { margin-bottom: 8px; padding: 12px 16px 13px; border-color: #e1e7ed; border-radius: 5px; }
  .section-heading { min-height: 28px; margin-bottom: 10px; padding-bottom: 9px; }
  .contract-key-summary { margin-bottom: 8px; padding: 12px 16px; border-color: #e1e7ed; border-radius: 5px; }
  .contract-key-summary strong { font-size: 18px; }
  .section-card :deep(.el-row > .el-col-8) { max-width: 25%; flex: 0 0 25%; }
  .section-card :deep(.el-row > .el-col-12) { max-width: 50%; flex: 0 0 50%; }
  .section-card :deep(.el-row > .el-col-16) { max-width: 50%; flex: 0 0 50%; }
  .section-card :deep(.el-row > .el-col-24) { max-width: 100%; flex: 0 0 100%; }
}
.contract-approval-header {
  box-sizing: border-box;
  min-height: 168px !important;
  margin: 10px 316px 0 16px;
  padding: 12px 16px 0 !important;
  border: 1px solid #e1e7ed !important;
  border-radius: 5px;
  align-items: flex-start !important;
}
.contract-approval-header .title-block { width: 100%; }
.contract-approval-header .eyebrow { margin-bottom: 6px; }
.contract-approval-header h1 { font-size: 19px; }
.contract-approval-header .budget-create-header-actions { position: absolute; top: 23px; right: 324px; }
.contract-approval-metrics { min-height: 45px; margin-top: 10px; border-top: 1px solid #e7ebf0; display: grid; grid-template-columns: repeat(3, minmax(0, 260px)); align-items: center; }
.contract-approval-metrics > div { min-width: 0; padding: 0 22px; border-right: 1px solid #e4e8ed; display: flex; align-items: baseline; gap: 10px; }
.contract-approval-metrics > div:first-child { padding-left: 0; }
.contract-approval-metrics > div:last-child { border-right: 0; }
.contract-approval-metrics span { color: #8793a0; font-size: 12px; white-space: nowrap; }
.contract-approval-metrics strong { color: #303b47; font-size: 16px; font-weight: 600; white-space: nowrap; }
.contract-approval-metrics strong.positive { color: #16a34a; }
@media (max-width: 1440px) {
  .contract-edit-page > .document-scroll,
  .contract-detail-page > .document-scroll { margin-right: 260px; }
  .contract-edit-page > .contract-side-directory,
  .contract-detail-page > .contract-side-directory { width: 260px; padding-right: 10px; padding-left: 10px; }
  .contract-approval-header { margin-right: 276px; }
  .contract-approval-header .budget-create-header-actions { right: 284px; }
  .contract-edit-page .contract-block :deep(.el-row > .el-col-8),
  .contract-detail-page .section-card :deep(.el-row > .el-col-8) { max-width: 33.3333%; flex-basis: 33.3333%; }
}
@media (max-width: 1180px) {
  .contract-edit-page .contract-block :deep(.el-row > .el-col-8),
  .contract-detail-page .section-card :deep(.el-row > .el-col-8),
  .contract-edit-page .contract-block :deep(.el-row > .el-col-12),
  .contract-detail-page .section-card :deep(.el-row > .el-col-12),
  .contract-edit-page .contract-block :deep(.el-row > .el-col-16),
  .contract-detail-page .section-card :deep(.el-row > .el-col-16) { max-width: 50%; flex-basis: 50%; }
}
@media (max-width: 1280px) {
  .flow-module-control:not(.collapsed) { width: 220px; flex-basis: 220px; }
  .flow-revenue-summary > div { gap: 8px; }
  .flow-revenue-summary > div span { font-size: 11px; }
  .flow-revenue-summary > div b { font-size: 13px; }
}
:global(.active-directory-item) { color: #1687f8; font-weight: 600; }
.contract-detail-toolbar-actions { display: flex; align-items: center; gap: 8px; }
.contract-local-attention, .contract-local-status { min-height: 38px; margin: -4px 0 12px; padding: 9px 12px; border-radius: 4px; display: flex; align-items: center; gap: 8px; font-size: 12px; svg { width: 16px; flex: none; } }
.contract-local-attention { color: #94600b; background: #fff8eb; svg { color: #d98a13; } }
.contract-local-status { margin-top: 12px; color: #526f89; background: #f2f8fe; svg { color: #1687f8; } }
:global(.el-dropdown-menu__item.is-current-contract-module) { color: #1687f8; background: #ecf5ff; }
:global(.el-dropdown-menu__item small) { margin-left: 36px; color: #909ca8; font-size: 11px; }
.contract-subtitle { margin: 12px 0 9px; color: #596775; font-size: 13px; }
.budget-reference-block { .contract-subtitle:first-child { margin-top: 0; } :deep(.el-descriptions__label) { color: #667685; font-weight: 500; } }
.contract-budget-plan { align-items: start; .metrics { min-width: 0; } .contract-subtitle { margin: 0 0 16px; padding: 0 0 12px; border-bottom: 1px solid #edf0f3; border-radius: 0; background: transparent; color: #323a45; font-size: 15px; font-weight: 600; } }
.contract-management-page { .filters { justify-content: flex-end; .el-input { margin-left: 0; } } }
.contract-prototype-content > .contract-block:first-child { :deep(.el-row > .el-col:first-child), :deep(.el-row > .el-col:nth-child(6)) { display: none; } :deep(.el-input), :deep(.el-select) { pointer-events: none; } :deep(.el-input__wrapper), :deep(.el-select__wrapper) { background: #f5f7fa; box-shadow: 0 0 0 1px #e4e7ed inset; } }
.upload-contract-code { max-width: 460px; margin-bottom: 4px; }
.compact-file-upload { min-height: 40px; display: flex; align-items: center; gap: 12px; color: #8995a1; font-size: 12px; }
.archive-history-summary { margin: 0 0 14px; padding: 10px 12px; border: 1px solid #d8e9fb; border-radius: 4px; background: #f5faff; display: flex; align-items: center; gap: 14px; color: #73808d; font-size: 12px; strong { color: #44515f; } }
.archive-file-action { min-height: 32px; display: flex; align-items: center; gap: 10px; &::after { color: #98a3ad; font-size: 11px; content: '支持 PDF、JPG、PNG，最多10个，单个不超过100MB'; } }
.archive-history-title { margin: 4px 0 10px; padding-top: 14px; border-top: 1px solid #edf0f3; color: #44515f; font-size: 13px; font-weight: 600; span { margin-left: 8px; color: #929da8; font-size: 12px; font-weight: 400; } }
.contract-file-table { margin-top: 14px; }
.association-context { box-sizing: border-box; width: 100%; max-width: 1400px; margin-right: auto; margin-left: auto; display: flex; align-items: center; justify-content: space-between; padding: 15px 18px; border: 1px solid #cfe4fb; border-radius: 6px 6px 0 0; background: #f5faff; > div { display: flex; align-items: center; gap: 12px; } strong { display: flex; align-items: center; gap: 9px; color: #27313a; font-size: 15px; b { color: #1687f8; font: 700 14px/1 monospace; } } }
.contract-prototype-content { > :nth-child(1) { order: 1; } > .association-context { order: 2; margin-bottom: -11px; border-radius: 6px 6px 0 0; } > :nth-child(3) { order: 3; border-top: 0; border-radius: 0; } > :nth-child(4) { order: 4; } > .contract-list-block { order: 5; } > .contract-info-block { order: 6; } > .contract-terms-fields { order: 7; } > .contract-files-block { order: 8; } > .downstream-task-block { order: 9; } > .related-order-block { order: 10; } }
.contract-prototype-content > .budget-reference-block { margin-top: -11px; margin-bottom: 0; border-top: 0; border-bottom: 0; border-radius: 0; h3 b { display: none; } h3 { margin: 0 0 18px; padding: 0 0 16px; border: 0; border-bottom: 1px solid #edf0f3; border-radius: 0; background: transparent; color: #323a45; font-size: 15px; font-weight: 600; } }
.contract-prototype-content > .budget-reference-block:nth-child(4) { margin-bottom: 10px; border-top: 0; border-bottom: 1px solid #e7eaf0; border-radius: 0 0 6px 6px; }
.contract-prototype-content > .budget-reference-block:nth-child(4) > h3 { display: none; }
.contract-budget-plan :deep(.el-descriptions__body tr:last-child) { display: none; }
.source-budget-attachments { margin-top: 18px; padding-top: 0; border-top: 0; h4 { margin: 0 0 12px; padding: 0 0 12px; border-bottom: 1px solid #edf0f3; border-radius: 0; background: transparent; color: #323a45; font-size: 15px; font-weight: 600; } }
.budget-reference-block :deep(.el-collapse) { margin-top: 18px; border: 0; }
.budget-reference-block :deep(.el-collapse-item__header) { height: 38px; padding: 0 12px; border: 0; border-radius: 4px; background: #f5f8fb; color: #44515f; font-weight: 600; }
.budget-reference-block :deep(.el-collapse-item__wrap) { border: 0; }
.related-budget-details { padding-top: 8px; padding-bottom: 10px; }
.related-budget-details :deep(.el-collapse) { margin-top: 0; }
.related-budget-details :deep(.el-collapse-item) { margin-bottom: 8px; }
.related-budget-details :deep(.el-collapse-item__header) { height: 48px; padding: 0 16px; border: 1px solid #edf0f3; background: #f8fafc; }
.related-budget-details :deep(.el-collapse-item__content) { padding: 16px 4px 8px; }
.related-budget-collapse-title { display: flex; align-items: center; gap: 14px; min-width: 0; strong { color: #323a45; font-size: 15px; } span { color: #8a98a8; font-size: 13px; font-weight: 400; } }
.related-budget-details .source-budget-attachments { margin-top: 0; }
.budget-change-records { margin-top: 20px; }
.budget-change-heading { display: flex; align-items: center; gap: 10px; margin: 0 0 10px; padding: 10px 12px; border-radius: 4px; background: #edf5fd; color: #27313a; font-size: 14px; span { color: #7f8c99; font-size: 12px; font-weight: 400; } }
.budget-change-list { display: flex; flex-direction: column; gap: 10px; }
.budget-change-list :deep(.el-collapse-item) { overflow: hidden; border: 1px solid #dfe5ec; border-radius: 5px; }
.budget-change-list :deep(.el-collapse-item__header) { height: 46px; padding: 0 14px; border-left: 4px solid #8ec5ff; border-radius: 0; background: #f7f9fc; }
.budget-change-list :deep(.el-collapse-item.is-active .el-collapse-item__header) { border-left-color: #1687f8; background: #f1f7fe; }
.budget-change-list :deep(.el-collapse-item__content) { padding: 0; }
.change-batch-body { margin-left: 18px; padding: 12px 14px 16px; border-left: 2px solid #e3edf7; background: #fff; }
.change-batch-title { width: 100%; display: flex; align-items: center; gap: 10px; strong { color: #323a45; } span { color: #7f8c99; font-size: 12px; &:first-of-type { margin-left: auto; } } }
.change-batch-summary { display: flex; align-items: center; justify-content: flex-end; gap: 22px; padding: 9px 13px; border: 1px solid #e7eaf0; border-radius: 4px; background: #fafbfd; color: #667685; font-size: 12px; }
.change-section { margin-top: 16px; h5 { display: flex; align-items: center; gap: 7px; margin: 0 0 9px; color: #44515f; font-size: 13px; &::before { width: 3px; height: 14px; border-radius: 2px; background: #9bbfe4; content: ''; } } }
.changed-value { color: #1687f8; font-weight: 600; }
.change-value { font-weight: 600; &.increase { color: #e67e22; } &.decrease { color: #13a884; } }
.changed-cell { display: inline-block; min-width: 100%; margin: -6px -10px; padding: 6px 10px; background: #fff1c7; color: #9a6400; font-weight: 600; }
.budget-goods-change-table :deep(.el-table__body tr:nth-child(2) td) { background: #fbfdff; }
.contract-prototype-content > :nth-child(6) { margin-bottom: -11px; border-radius: 6px 6px 0 0; }
.contract-prototype-content > .contract-terms-fields { margin-top: 0; margin-bottom: 0; border-top: 0; border-bottom: 0; border-radius: 0; h3 { display: none; } }
.contract-prototype-content > .contract-files-block { margin-top: -11px; border-top: 0; border-radius: 0 0 6px 6px; h3 { margin-bottom: 14px; padding: 9px 12px; border: 0; border-radius: 4px; background: #f5f8fb; font-size: 13px; b { display: none; } } }
.contract-content-subtitle { margin: 0 0 14px; padding: 9px 12px; border-radius: 4px; background: #f5f8fb; color: #44515f; font-size: 13px; }
.generate-contract-action { display: flex; align-items: center; justify-content: space-between; padding: 12px 14px; border: 1px dashed #b9d8f7; border-radius: 5px; background: #f8fbff; div { display: flex; flex-direction: column; gap: 4px; } strong { font-size: 13px; } span { color: #8995a1; font-size: 12px; } }
.contract-code-action { width: 100%; display: flex; gap: 8px; }
.mode-framework .contract-terms-fields { display: none; }
.mode-upload .contract-terms-fields :deep(.el-row > .el-col:nth-child(1)), .mode-upload .contract-terms-fields :deep(.el-row > .el-col:nth-child(2)), .mode-upload .contract-terms-fields :deep(.el-row > .el-col:nth-child(3)) { display: none; }
.type-sale.mode-generated .contract-terms-fields :deep(.el-row > .el-col:nth-child(3)) { display: none; }
.contract-files-block > .contract-file-row { display: none; }
.task-tab-toolbar { display: flex; align-items: center; justify-content: space-between; gap: 16px; margin-bottom: 12px; color: #7f8c98; font-size: 12px; }
.single-task-heading { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; padding: 9px 12px; border-radius: 4px; background: #f5f8fb; color: #44515f; font-size: 13px; span { color: #8995a1; font-size: 12px; } }
.contract-block { max-width: 1400px; width: 100%; margin: 0 auto 10px; padding: 17px 20px 4px; background: #fff; border: 1px solid #e7eaf0; border-radius: 6px; h3 { display: flex; align-items: center; gap: 12px; margin: 0 0 13px; padding-bottom: 12px; border-bottom: 1px solid #edf0f3; font-size: 15px; b { color: #1687f8; font: 700 0/1 monospace; &::before { counter-increment: contract-section; content: counter(contract-section, decimal-leading-zero); font-size: 18px; } } } }
.contract-budget-plan > div:first-child > .contract-subtitle { font-size: 0; &::after { content: '预算计划'; font-size: 15px; font-weight: 600; } }
.contract-terms-fields :deep(.el-row > .el-col:nth-child(4)), .contract-terms-fields :deep(.el-row > .el-col:nth-child(5)), .contract-terms-fields :deep(.el-row > .el-col:nth-child(6)) { display: none; }
.primary-amount-field { :deep(.el-input__inner) { color: #323a45; font-weight: 600; } }
.contract-amount { color: #1687f8; font-size: 15px; }
.secondary-contract-info { margin-top: 14px; :deep(.el-collapse-item__header) { padding: 0 12px; border: 1px solid #e7eaf0; border-radius: 4px; background: #f7f9fb; color: #667685; font-weight: 600; } :deep(.el-collapse-item__wrap) { border: 0; } :deep(.el-collapse-item__content) { padding-top: 12px; padding-bottom: 0; } }
.file-download-trigger { margin-left: 3px; padding: 0 3px; }
.readonly-link-field { box-sizing: border-box; width: 100%; min-height: 32px; padding: 0 11px; border: 1px solid #dcdfe6; border-radius: 4px; background: #f5f7fa; display: flex; align-items: center; justify-content: space-between; color: #1687f8; svg { width: 13px; transform: rotate(-135deg); } }
.supplier-amount-grid { display: grid; grid-template-columns: repeat(4, minmax(0, 1fr)); gap: 12px; > div { min-height: 84px; padding: 14px 16px; border: 1px solid #e7eaf0; border-radius: 5px; background: #f7f9fb; display: flex; flex-direction: column; gap: 7px; span, small { color: #87939e; font-size: 12px; } .el-link { align-self: flex-start; font-size: 18px; font-weight: 700; } } }
.contract-list-block { padding-bottom: 16px; }
.budget-attachment { display: flex; align-items: center; gap: 12px; margin-top: 12px; padding: 10px 12px; background: #f7f9fb; border-radius: 4px; font-size: 12px; > span { color: #667685; font-weight: 600; } small { color: #98a3ad; } }
.contract-file-row, .task-switch-row { display: flex; align-items: center; gap: 12px; margin-bottom: 12px; padding: 12px; border: 1px solid #e5e9ee; border-radius: 5px; .file-icon { display: grid; place-items: center; width: 38px; height: 38px; border-radius: 5px; color: #1687f8; background: #eaf4ff; font-size: 11px; font-weight: 700; } > div { flex: 1; } p { margin: 4px 0 0; color: #8c98a4; font-size: 12px; } }
.upload-copy { padding: 10px; color: #667685; em { color: #1687f8; font-style: normal; } small { display: block; margin-top: 8px; color: #9aa5af; } }
.contract-file-columns { display: grid; grid-template-columns: 120px 1fr 140px 110px 90px; gap: 12px; margin: 0 0 12px; padding: 9px 12px; color: #8a96a1; background: #f7f9fb; border-radius: 4px; font-size: 12px; }
.secondary-contract-fields { padding-bottom: 8px; :deep(.el-collapse) { border: 0; } :deep(.el-collapse-item__header) { color: #667685; font-size: 13px; } :deep(.el-collapse-item__wrap) { border: 0; } }
:deep(.el-form-item) { margin-bottom: 16px; }
:deep(.el-form-item__label) { color: #667685; font-size: 12px; }
:deep(.el-select), :deep(.el-date-editor), :deep(.el-input-number) { width: 100%; }
:deep(.el-table th.el-table__cell) { background: #f6f8fa; color: #667685; font-weight: 600; }
:deep(.el-steps) { margin-bottom: 22px; }

/* 预算计划 / 收益测算：新建预算、预算详情、合同新增与详情统一使用同一套视觉结构。 */
.budget-readonly-page #plan .split-panel,
.contract-edit-page .contract-budget-plan,
.contract-detail-page .contract-budget-plan {
  grid-template-columns: minmax(620px, 65fr) minmax(330px, 35fr);
  gap: 22px;
  align-items: start;
}

.budget-readonly-page #plan .plan-form,
.budget-readonly-page #plan .metrics,
.contract-edit-page .contract-budget-plan > div:first-child,
.contract-edit-page .contract-budget-plan > .metrics,
.contract-detail-page .contract-budget-plan > div:first-child,
.contract-detail-page .contract-budget-plan > .metrics {
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent;
}

.budget-readonly-page #plan .metrics,
.contract-edit-page .contract-budget-plan > .metrics,
.contract-detail-page .contract-budget-plan > .metrics {
  padding-left: 22px;
  border-left: 1px solid #e4eaf1;
}

.budget-readonly-page #plan .plan-form :deep(.el-col-12),
.contract-edit-page .contract-budget-plan :deep(.el-col-12),
.contract-detail-page .contract-budget-plan :deep(.el-col-12) {
  max-width: 50%;
  flex: 0 0 50%;
}

.budget-readonly-page #plan .plan-form :deep(.el-col-24),
.contract-edit-page .contract-budget-plan :deep(.el-col-24),
.contract-detail-page .contract-budget-plan :deep(.el-col-24) {
  max-width: 100%;
  flex: 0 0 100%;
}

.budget-readonly-page #plan .plan-form :deep(.el-form-item),
.contract-edit-page .contract-budget-plan :deep(.el-form-item),
.contract-detail-page .contract-budget-plan :deep(.el-form-item) {
  display: block;
  min-height: 0;
  margin-bottom: 12px;
}

.budget-readonly-page #plan .plan-form :deep(.el-form-item__label),
.contract-edit-page .contract-budget-plan :deep(.el-form-item__label),
.contract-detail-page .contract-budget-plan :deep(.el-form-item__label) {
  display: block;
  padding: 0 0 6px;
  line-height: 20px;
  white-space: normal;
}

.budget-readonly-page #plan .metric-grid,
.contract-edit-page .contract-budget-plan .metric-grid,
.contract-detail-page .contract-budget-plan .metric-grid {
  grid-template-columns: 1fr;
  gap: 0;
}

.budget-readonly-page #plan .metric-grid > div,
.contract-edit-page .contract-budget-plan .metric-grid > div,
.contract-detail-page .contract-budget-plan .metric-grid > div {
  min-height: 42px;
  padding: 7px 0;
  border-bottom: 1px dashed #e8edf3;
  border-radius: 0;
  background: transparent;
  display: grid;
  grid-template-columns: minmax(130px, 1fr) auto;
  align-items: center;
}

.budget-readonly-page #plan .metric-grid > div.warning,
.contract-edit-page .contract-budget-plan .metric-grid > div.warning,
.contract-detail-page .contract-budget-plan .metric-grid > div.warning {
  background: transparent;
}

.budget-readonly-page #plan .metric-grid > div strong,
.contract-edit-page .contract-budget-plan .metric-grid > div strong,
.contract-detail-page .contract-budget-plan .metric-grid > div strong {
  font-size: 16px;
  text-align: right;
}

.contract-edit-page .contract-block h3 b {
  width: 24px !important;
  height: 24px !important;
  display: grid !important;
  place-items: center !important;
  flex: none;
  border-radius: 50% !important;
  background: #1687f8 !important;
  color: #fff !important;
  font: 700 12px/1 sans-serif !important;
}
.contract-edit-page .contract-block h3 b::before { display: none !important; content: none !important; }
.contract-side-directory .flow-navigation-card { width: 100%; margin-bottom: 10px; }
.contract-side-directory .flow-navigation-card .flow-control-toggle { min-height: 34px; display: flex; align-items: center; gap: 7px; padding: 0 8px; color: #566575; font-size: 12px; font-weight: 600; }
.contract-side-directory .flow-navigation-card .flow-control-toggle svg { width: 15px; }

/* Contract content直接展示真实业务分组，不再增加空的“交易与金额”总标题。 */
.contract-info-block > .contract-content-subtitle,
.contract-detail-info > .contract-content-subtitle:first-of-type {
  display: none;
}
.contract-info-block > form > .el-row,
.contract-detail-info > form > .el-row {
  align-items: start;
}
.contract-info-block .primary-amount-field :deep(.el-form-item__label),
.contract-detail-info :deep(.el-form-item):has(.el-input__inner[value*='¥']) .el-form-item__label {
  color: #4e5d6c;
  font-weight: 600;
}
.contract-info-block .primary-amount-field :deep(.el-input__inner),
.contract-detail-info :deep(.el-input__inner[value*='¥']) {
  color: #303b47;
  font-weight: 600;
}
.contract-detail-info .secondary-contract-info :deep(.el-collapse-item__header) {
  font-size: 0;
}
.contract-detail-info .secondary-contract-info :deep(.el-collapse-item__header)::before {
  content: '履约、用印与补充信息';
  color: #667685;
  font-size: 13px;
  font-weight: 600;
}
.contract-terms-fields {
  background: #fbfcfd;
}
.contract-terms-fields :deep(.el-form-item__label) {
  color: #687786;
}
.contract-approval-view .contract-detail-info {
  border-color: #e7d19f;
}
.contract-approval-view .contract-detail-info .primary-amount-field :deep(.el-input__inner),
.contract-approval-view .contract-approval-metrics strong {
  font-weight: 700;
}
.contract-detail-summary-header {
  position: relative;
  min-height: 0;
  display: block;
  padding: 14px 20px 12px;
}
.contract-detail-summary-header > .title-block {
  width: 100%;
  box-sizing: border-box;
  padding-right: 220px;
}
.contract-detail-header-metrics {
  position: static;
  min-height: 44px;
  margin-top: 10px;
  padding-top: 9px;
  border-top: 1px solid #e7ebf0;
  display: grid;
  grid-template-columns: repeat(3, minmax(180px, 260px));
  align-items: center;
}
.contract-detail-header-metrics > div {
  min-width: 0;
  padding: 0 22px;
  border-right: 1px solid #e4e8ed;
  display: flex;
  align-items: baseline;
  gap: 10px;
}
.contract-detail-header-metrics > div:first-child { padding-left: 0; }
.contract-detail-header-metrics > div:last-child { border-right: 0; }
.contract-detail-header-metrics span {
  color: #8793a0;
  font-size: 12px;
  white-space: nowrap;
}
.contract-detail-header-metrics strong {
  color: #303b47;
  font-size: 16px;
  font-weight: 600;
  white-space: nowrap;
}
.contract-detail-summary-header > .budget-create-header-actions,
.contract-approval-header.contract-detail-summary-header > .budget-create-header-actions {
  position: absolute;
  top: 18px;
  right: 20px;
}
.contract-approval-header.contract-detail-summary-header {
  min-height: 0;
  padding: 14px 20px 12px;
}
.budget-edit-page .edit-scope-panel {
  max-width: none;
  margin: 0 0 8px;
}

.contract-edit-page .contract-field-group-title,
.contract-detail-page .contract-field-group-title {
  margin: 2px 0 10px;
  padding: 8px 10px;
  border-left: 3px solid #1687f8;
  background: #f7faff;
  color: #354150;
  font-size: 14px;
  font-weight: 600;
}
.contract-edit-page .contract-mode-form-item :deep(.el-form-item__content) { display: block; }
.contract-edit-page .contract-mode-cards {
  width: 100%;
  display: grid;
  grid-template-columns: repeat(3, minmax(0, 1fr));
  gap: 12px;
}
.contract-edit-page .contract-mode-cards :deep(.el-radio) {
  box-sizing: border-box;
  height: auto;
  min-height: 68px;
  margin: 0;
  padding: 13px 14px;
  border: 1px solid #dfe5ec;
  border-radius: 5px;
  background: #fff;
  align-items: flex-start;
}
.contract-edit-page .contract-mode-cards :deep(.el-radio.is-checked) {
  border-color: #1687f8;
  background: #f3f8ff;
  box-shadow: 0 0 0 1px #1687f8 inset;
}
.contract-edit-page .contract-mode-cards :deep(.el-radio__label) {
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 5px;
  white-space: normal;
}
.contract-edit-page .contract-mode-cards strong { color: #303944; font-size: 14px; }
.contract-edit-page .contract-mode-cards span { color: #8793a0; font-size: 12px; line-height: 18px; }
.order-edit-page { position: fixed; z-index: 910; top: 64px; right: 0; bottom: 0; left: 220px; background: #eef1f5; }
.order-edit-page > .document-header { box-sizing: border-box; min-height: 72px; padding: 11px 20px; }
.order-edit-page > .order-document-scroll { box-sizing: border-box; margin-right: 224px; padding: 10px 14px 42px; }
.order-edit-page > .order-side-directory { top: 72px; width: 224px; padding: 10px; }
.order-edit-page .section-card { max-width: none; margin: 0 0 9px; padding: 14px 16px; border-color: #dfe6ee; box-shadow: none; }
.order-edit-page :deep(.el-form-item) { margin-bottom: 12px; }
.order-edit-page :deep(.el-form-item__label) { padding-bottom: 6px; line-height: 20px; }
.order-edit-page :deep(.el-input), .order-edit-page :deep(.el-select), .order-edit-page :deep(.el-input-number), .order-edit-page :deep(.el-date-editor) { width: 100%; }
.order-edit-page .order-goods-section { border-color: #f1d2a9; }
.order-edit-page .order-goods-section .section-number { background: #f39828 !important; }
.order-edit-page .order-attachments-section { padding-bottom: 8px; }
.order-edit-page .order-attachments-section :deep(.el-form-item) { margin-bottom: 4px; }
.order-edit-page .order-attachments-section :deep(.el-textarea__inner) { min-height: 64px !important; }
.order-edit-page .order-attachments-section :deep(.el-upload), .order-edit-page .order-attachments-section :deep(.el-upload-dragger) { width: 100%; }
.order-edit-page .order-attachments-section :deep(.el-upload-dragger) { box-sizing: border-box; height: 72px; padding: 8px 12px; }
.order-edit-page .order-attachments-section :deep(.el-upload-dragger svg) { width: 20px; margin-bottom: 2px; }
.order-edit-page .order-attachments-section :deep(.el-upload-dragger div), .order-edit-page .order-attachments-section :deep(.el-upload-dragger small) { font-size: 12px; line-height: 18px; }
.order-table-summary { min-height: 38px; display: flex; justify-content: flex-end; align-items: center; gap: 26px; padding: 0 12px; border: 1px solid #ebeef2; border-top: 0; background: #fafbfc; color: #657485; }
.order-table-summary strong { color: #26313c; font-size: 14px; }
.order-overview-card { margin-bottom: 10px; padding: 12px; border: 1px solid #dfe5ec; border-radius: 5px; background: #fff; }
.order-overview-card > div { display: flex; justify-content: space-between; margin-bottom: 7px; }
.order-overview-card small { color: #98a3ad; }
.order-overview-card p { display: flex; justify-content: space-between; align-items: center; margin: 0; padding: 8px 0; border-bottom: 1px dashed #e7ebef; color: #71808f; }
.order-overview-card p:last-child { border-bottom: 0; }
.order-overview-card strong { color: #27313a; }
.order-side-directory .flow-control-item { grid-template-columns: 25px minmax(0,1fr) auto; }
.order-side-directory .flow-control-item em { font-size: 10px; font-style: normal; white-space: nowrap; }
.order-side-directory .flow-control-item em.done { color: #13a884; }
.order-side-directory .flow-control-item em.warning { color: #d98600; }

/* 详情页采用单层模块框 + 轻量文本字段，减少输入框式视觉噪音 */
.budget-readonly-page .section-card,
.contract-detail-page .section-card {
  border-color: #dce4ec;
  background: #fff;
}
.budget-readonly-page article :deep(.el-form-item),
.contract-detail-page article :deep(.el-form-item) {
  display: block;
  min-height: 44px;
  margin-bottom: 8px;
}
.budget-readonly-page article :deep(.el-form-item__label),
.contract-detail-page article :deep(.el-form-item__label) {
  display: block;
  padding: 0 0 4px;
  color: #748291;
  font-size: 12px;
  font-weight: 400;
  line-height: 18px;
}
.budget-readonly-page article :deep(.el-input__wrapper),
.budget-readonly-page article :deep(.el-select__wrapper),
.contract-detail-page article :deep(.el-input__wrapper),
.contract-detail-page article :deep(.el-select__wrapper) {
  min-height: 22px;
  padding: 0;
  border: 0;
  border-radius: 0;
  background: transparent !important;
  box-shadow: none !important;
}
.budget-readonly-page article :deep(.el-input.is-disabled .el-input__inner),
.contract-detail-page article :deep(.el-input.is-disabled .el-input__inner),
.budget-readonly-page article :deep(.el-select__selected-item),
.contract-detail-page article :deep(.el-select__selected-item) {
  color: #2f3a45;
  font-size: 13px;
  font-weight: 500;
  -webkit-text-fill-color: #2f3a45;
}
.budget-readonly-page article :deep(.el-input__suffix),
.contract-detail-page article :deep(.el-input__suffix) { display: none; }
.budget-readonly-page article :deep(.el-input-number),
.contract-detail-page article :deep(.el-input-number),
.budget-readonly-page article :deep(.el-date-editor),
.contract-detail-page article :deep(.el-date-editor) {
  width: 100% !important;
  min-height: 22px;
  border: 0;
  background: transparent !important;
  box-shadow: none !important;
}
.budget-readonly-page article :deep(.el-input-number .el-input__wrapper),
.contract-detail-page article :deep(.el-input-number .el-input__wrapper),
.budget-readonly-page article :deep(.el-date-editor .el-input__wrapper),
.contract-detail-page article :deep(.el-date-editor .el-input__wrapper) {
  min-height: 22px !important;
  padding: 0 !important;
  background: transparent !important;
  box-shadow: none !important;
}
.budget-readonly-page article :deep(.el-input-number__decrease),
.budget-readonly-page article :deep(.el-input-number__increase),
.contract-detail-page article :deep(.el-input-number__decrease),
.contract-detail-page article :deep(.el-input-number__increase),
.budget-readonly-page article :deep(.el-date-editor .el-input__prefix),
.contract-detail-page article :deep(.el-date-editor .el-input__prefix) { display: none !important; }
.budget-readonly-page article :deep(.el-input-number .el-input__inner),
.contract-detail-page article :deep(.el-input-number .el-input__inner),
.budget-readonly-page article :deep(.el-date-editor .el-input__inner),
.contract-detail-page article :deep(.el-date-editor .el-input__inner) {
  padding: 0 !important;
  color: #2f3a45 !important;
  font-size: 13px;
  font-weight: 500;
  text-align: left !important;
  -webkit-text-fill-color: #2f3a45 !important;
}
.budget-readonly-page article :deep(.el-switch.is-disabled),
.contract-detail-page article :deep(.el-switch.is-disabled) { opacity: .72; }
.budget-readonly-page article :deep(.el-textarea__inner),
.contract-detail-page article :deep(.el-textarea__inner) {
  min-height: 44px !important;
  padding: 7px 9px;
  border: 1px solid #e5eaf0;
  background: #fafbfd;
  box-shadow: none;
}
.budget-readonly-page .readonly-link-field,
.contract-detail-page .readonly-link-field {
  min-height: 22px;
  padding: 0;
  border: 0;
  background: transparent;
}
.budget-readonly-page .detail-result,
.contract-detail-page .detail-result {
  min-height: 24px;
  display: flex;
  align-items: center;
  gap: 5px;
  color: #2f3a45;
  font-size: 13px;
  line-height: 24px;
}
.budget-readonly-page .detail-result strong,
.contract-detail-page .detail-result strong {
  color: #263442;
  font-size: 14px;
  font-weight: 600;
}
.budget-readonly-page .detail-result > span:not(.detail-date-mark),
.contract-detail-page .detail-result > span:not(.detail-date-mark) {
  color: #748291;
  font-size: 12px;
}
.budget-readonly-page .detail-date-mark,
.contract-detail-page .detail-date-mark {
  width: 17px;
  height: 17px;
  display: inline-flex;
  align-items: center;
  justify-content: center;
  border: 1px solid #b9c5d0;
  border-radius: 3px;
  color: #7a8997;
  font-size: 9px;
  line-height: 1;
}
.budget-readonly-page .detail-money strong,
.contract-detail-page .detail-money strong { font-variant-numeric: tabular-nums; }
.budget-readonly-page .detail-term-result strong,
.contract-detail-page .contract-budget-plan :deep(.el-col-12:first-child .el-input__inner),
.contract-detail-page .contract-detail-info .secondary-contract-info :deep(.el-row > .el-col-8:nth-child(2) .el-input__inner) {
  color: #263442 !important;
  font-weight: 700 !important;
  -webkit-text-fill-color: #263442 !important;
}
.contract-detail-page .contract-budget-plan :deep(.el-col-12:first-child .el-form-item__label),
.contract-detail-page .contract-detail-info .secondary-contract-info :deep(.el-row > .el-col-8:nth-child(2) .el-form-item__label) {
  color: #526273;
  font-weight: 600;
}
.budget-readonly-page .section-heading,
.contract-detail-page .section-heading {
  margin-bottom: 12px;
  padding-bottom: 10px;
}
.contract-detail-page .contract-field-group-title {
  margin: 0 0 9px;
  padding: 5px 8px;
  background: #f7f9fb;
  font-size: 13px;
}
.contract-detail-page .secondary-contract-info {
  margin-top: 2px;
  border-top: 1px solid #edf0f3;
}
.contract-detail-page .document-scroll > .contract-readonly-section:first-child > .secondary-contract-info {
  display: none;
}
.contract-detail-page .contract-detail-info > :deep(.el-form) {
  margin-bottom: 4px;
}

/* 详情 / 审批统一为字段名在上、结果在下，与新建和编辑保持相同阅读顺序。 */
.budget-readonly-page #basic :deep(.el-form-item),
.contract-detail-page .document-scroll > .contract-readonly-section:first-child :deep(.el-form-item) {
  min-height: 48px;
  display: block;
  margin-bottom: 9px;
}
.budget-readonly-page #basic :deep(.el-form-item__label),
.contract-detail-page .document-scroll > .contract-readonly-section:first-child :deep(.el-form-item__label) {
  display: block;
  padding: 0 0 5px;
  color: #7a8794;
  line-height: 18px;
  white-space: normal;
}
.budget-readonly-page #basic :deep(.el-form-item__content),
.contract-detail-page .document-scroll > .contract-readonly-section:first-child :deep(.el-form-item__content) {
  display: block;
  min-width: 0;
  min-height: 24px;
  line-height: 24px;
}
.budget-readonly-page #basic :deep(.el-input__inner),
.budget-readonly-page #basic :deep(.el-select__selected-item),
.contract-detail-page .document-scroll > .contract-readonly-section:first-child :deep(.el-input__inner) {
  overflow: hidden;
  color: #2f3a45 !important;
  font-size: 13px;
  font-weight: 500;
  text-overflow: ellipsis;
  white-space: nowrap;
}
.budget-readonly-page #basic .detail-result { min-height: 24px; line-height: 24px; }
.budget-readonly-page .detail-entity-link,
.contract-detail-page .detail-entity-link {
  min-height: 24px;
  display: inline-flex;
  align-items: center;
  font-size: 13px;
  font-weight: 500;
  line-height: 24px;
}
.budget-readonly-page #basic .budget-subsection-title { margin: 5px 0 11px; }
.budget-readonly-page #basic .transaction-title { margin-top: 9px; }

.budget-readonly-page #plan :deep(.el-textarea__inner),
.contract-detail-page .contract-budget-plan :deep(.el-textarea__inner),
.contract-detail-page .secondary-contract-info :deep(.el-textarea__inner) {
  min-height: 42px !important;
  padding: 6px 10px;
  border: 0;
  border-left: 2px solid #dce8f4;
  border-radius: 0;
  background: #fafcfe !important;
  box-shadow: none !important;
}
.budget-readonly-page :deep(.el-form-item.is-required .el-form-item__label::before),
.contract-detail-page :deep(.el-form-item.is-required .el-form-item__label::before) { display: none; }
.budget-readonly-page .section-card,
.contract-detail-page .section-card {
  box-shadow: none;
}
.budget-readonly-page :deep(.el-table th.el-table__cell),
.contract-detail-page :deep(.el-table th.el-table__cell) {
  background: #f7f9fb;
  color: #5f6e7c;
}
.budget-readonly-page :deep(.el-table .el-table__cell),
.contract-detail-page :deep(.el-table .el-table__cell) {
  padding-top: 7px;
  padding-bottom: 7px;
}

@media (max-width: 1380px) {
  .budget-readonly-page #basic :deep(.el-col-6),
  .budget-readonly-page #basic :deep(.el-col-8),
  .contract-detail-page .document-scroll > .contract-readonly-section:first-child :deep(.el-col-8) {
    max-width: 50%;
    flex-basis: 50%;
  }
}

/* 合同新建 / 编辑沿用预算单母版：相同模块密度、字段节奏和响应式列数。 */
.contract-edit-page .contract-block {
  padding: 10px 14px 8px;
  border-color: #cfe0f5;
  border-radius: 6px;
  background: #fff;
  box-shadow: none;
}
.contract-edit-page .contract-block > h3 {
  min-height: 26px;
  margin: 0 0 10px;
  padding-bottom: 8px;
  border-bottom: 1px solid #edf0f3;
  color: #273442;
  font-size: 14px;
}
.contract-edit-page .contract-block > h3 b,
.contract-edit-page .association-context strong b {
  width: 22px;
  height: 22px;
  display: inline-grid;
  place-items: center;
  border-radius: 50%;
  background: #1687f8;
  color: #fff;
  font-size: 11px;
  line-height: 22px;
}
.contract-edit-page :deep(.el-form-item) { margin-bottom: 12px; }
.contract-edit-page :deep(.el-form-item__label) {
  display: block;
  padding: 0 0 6px;
  color: #5f6f80;
  font-size: 12px;
  line-height: 20px;
}
.contract-edit-page .budget-reference-block :deep(.el-input.is-disabled .el-input__inner) {
  color: #657383;
  -webkit-text-fill-color: #657383;
}
.contract-edit-page .contract-prototype-content > .budget-reference-block > h3 {
  min-height: 24px;
  display: block;
  margin: 0 0 12px;
  padding: 0 0 9px;
  color: #4a5968;
  font-size: 0;
  line-height: 24px;
}
.contract-edit-page .contract-prototype-content > .budget-reference-block > h3::after {
  content: '预算基础信息';
  padding-left: 9px;
  border-left: 3px solid #9fc9f3;
  font-size: 13px;
  font-weight: 600;
}
.contract-edit-page .contract-prototype-content > .budget-reference-block > h3 b {
  display: none !important;
}
.contract-edit-page .budget-basic-subtitle {
  min-height: 24px;
  margin: 0 0 12px;
  padding: 0 0 9px 9px;
  border-bottom: 1px solid #edf0f3;
  border-left: 3px solid #9fc9f3;
  color: #4a5968;
  font-size: 13px;
  font-weight: 600;
  line-height: 24px;
}
.contract-edit-page :deep(.el-table th.el-table__cell) {
  background: #f7f9fb;
  color: #5f6e7c;
}
.contract-edit-page :deep(.el-table .el-table__cell) {
  padding-top: 7px;
  padding-bottom: 7px;
}
@media (max-width: 1500px) {
  .contract-edit-page .contract-block :deep(.el-row > .el-col-8),
  .contract-detail-page .section-card :deep(.el-row > .el-col-8) {
    max-width: 33.3333%;
    flex-basis: 33.3333%;
  }
}
@media (max-width: 1180px) {
  .contract-edit-page .contract-block :deep(.el-row > .el-col-8),
  .contract-edit-page .contract-block :deep(.el-row > .el-col-12),
  .contract-detail-page .section-card :deep(.el-row > .el-col-8),
  .contract-detail-page .section-card :deep(.el-row > .el-col-12) {
    max-width: 50%;
    flex-basis: 50%;
  }
}

/* 一级模块下的二级标题统一：白底、浅蓝竖线、深色文字。 */
.budget-subsection-title,
.budget-create-page #plan .plan-form > h3,
.budget-readonly-page #plan .plan-form > h3,
.budget-create-page #plan .metrics > h3,
.budget-readonly-page #plan .metrics > h3,
.budget-create-page #attachments .attachment-column > h3,
.budget-readonly-page #attachments .attachment-column > h3,
.budget-create-page #attachments .note-column > h3,
.budget-readonly-page #attachments .note-column > h3,
.contract-content-subtitle,
.contract-subtitle,
.contract-edit-page .contract-field-group-title,
.contract-detail-page .contract-field-group-title,
.contract-edit-page .budget-basic-subtitle,
.single-task-heading {
  box-sizing: border-box;
  min-height: 30px;
  margin: 4px 0 12px;
  padding: 4px 0 4px 10px;
  border: 0;
  border-left: 3px solid #9fc9f3;
  border-radius: 0;
  background: transparent;
  color: #44515f;
  font-size: 13px;
  font-weight: 600;
  line-height: 22px;
}
.budget-create-page #plan .metrics > h3 small,
.budget-readonly-page #plan .metrics > h3 small,
.single-task-heading span {
  margin-left: 8px;
  color: #8a97a5;
  font-size: 12px;
  font-weight: 400;
}
.contract-readonly-section > .contract-content-subtitle {
  min-height: 30px;
  padding-right: 0;
  padding-left: 10px;
  border: 0;
  border-left: 3px solid #9fc9f3;
  background: transparent;
}
.contract-edit-page .related-budget-details :deep(.el-collapse-item__header),
.contract-detail-page .detail-budget-collapse :deep(.el-collapse-item__header),
.contract-detail-page .secondary-contract-info :deep(.el-collapse-item__header) {
  min-height: 38px;
  padding: 4px 10px;
  border: 0;
  border-left: 3px solid #9fc9f3;
  border-radius: 0;
  background: transparent;
  color: #44515f;
}
.contract-edit-page .contract-info-block > .contract-content-subtitle,
.contract-detail-page .contract-detail-info > .contract-content-subtitle:first-of-type {
  display: none;
}

@media (max-width: 1250px) { .task-rail { width: 190px; } .split-panel { grid-template-columns: 1fr; .metrics { padding: 18px 0 0; border-left: 0; border-top: 1px solid #edf0f3; } } .attachment-layout { grid-template-columns: 1fr; } }
</style>
