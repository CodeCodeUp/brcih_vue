<template>
  <div class="data-monitor">
    <div class="monitor-header">
      <div class="title">
        <h2>数据监控</h2>
        <div class="subtitle">变动情况</div>
      </div>

      <div class="date-range-picker">
        <el-date-picker
          v-model="dateRange"
          type="daterange"
          range-separator="至"
          start-placeholder="开始日期"
          end-placeholder="结束日期"
          :shortcuts="shortcuts"
          format="YYYY-MM-DD"
          value-format="YYYY-MM-DD"
        />
        <el-select
          v-model="changeType"
          placeholder="变动类型"
          style="margin-right: 10px; margin-left: 10px"
        >
          <el-option label="全部" value=""></el-option>
          <el-option label="增持" value="增持"></el-option>
          <el-option label="减持" value="减持"></el-option>
        </el-select>
        <el-button type="primary" @click="fetchData" :loading="loading">Search</el-button>
      </div>
    </div>

    <StockTable
      :display-data="displayData"
      :loading="loading"
      :header-style="headerStyle"
      :current-page="currentPage"
      :page-size="pageSize"
      :total-count="totalCount"
      :stock-data="stockData"
      @expand-change="handleExpandChange"
      @sort-change="handleSortChange"
      @size-change="handleSizeChange"
      @current-change="handleCurrentChange"
    />
  </div>
</template>

<script lang="ts">
import { defineComponent } from 'vue'
import { useStockData } from '@/composables/useStockData'
import { useChart } from '@/composables/useChart'
import StockTable from './StockTable.vue'

export default defineComponent({
  name: 'DataMonitor',
  components: {
    StockTable,
  },
  setup() {
    // Use composables
    const stockDataComposable = useStockData()
    const chartComposable = useChart()

    return {
      // Stock data composable
      ...stockDataComposable,
      
      // Chart composable
      ...chartComposable,
    }
  },
})
</script>

<style scoped>
.data-monitor {
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 2px 12px 0 rgba(0, 0, 0, 0.1);
  padding: 20px;
  width: 100%;
}

.monitor-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 24px;
  border-bottom: 1px solid #eaeaea;
  padding-bottom: 16px;
}

.title h2 {
  margin: 0;
  font-weight: 600;
  color: #303133;
  font-size: 22px;
}

.subtitle {
  color: #909399;
  font-size: 14px;
  margin-top: 5px;
}

.date-range-picker {
  display: flex;
  align-items: center;
  gap: 10px;
}

@media (max-width: 768px) {
  .monitor-header {
    flex-direction: column;
    align-items: flex-start;
  }

  .date-range-picker {
    margin-top: 15px;
    width: 100%;
  }
}
</style>
