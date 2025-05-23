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

    <div class="table-container">
      <el-table
        :data="displayData"
        stripe
        style="width: 100%"
        :header-cell-style="headerStyle"
        v-loading="loading"
        @expand-change="handleExpandChange"
        @sort-change="handleSortChange"
        :row-key="(row: any) => row.stockCode + row.tradeDate"
      >
        <el-table-column type="expand">
          <template #default="props">
            <div class="stock-detail-row" v-loading="props.row.chartLoading">
              <div class="chart-container">
                <div
                  :id="`chart-${props.row.stockCode}-${props.row.tradeDate}`"
                  class="stock-chart"
                ></div>
              </div>
              <div class="history-button-container">
                <el-button type="primary" size="small" @click="showHistoryDetail(props.row)">
                  查看所有历史数据
                </el-button>
              </div>
              <div
                class="marks-detail"
                v-if="
                  props.row.stockDetail &&
                  props.row.stockDetail.marks &&
                  props.row.stockDetail.marks.length > 0
                "
              >
                <h4>增减持记录</h4>
                <el-table
                  :data="props.row.stockDetail.marks"
                  stripe
                  size="small"
                  style="width: 100%"
                >
                  <el-table-column prop="tradeDate" label="交易日期" width="120" />
                  <el-table-column prop="changeType" label="变动类型" width="100">
                    <template #default="scope">
                      <span :class="scope.row.changeType === '增持' ? 'increase' : 'decrease'">
                        {{ scope.row.changeType }}
                      </span>
                    </template>
                  </el-table-column>
                  <el-table-column label="交易金额" width="150">
                    <template #default="scope">
                      <span :class="scope.row.changeType === '增持' ? 'increase' : 'decrease'">
                        {{ formatNumber(scope.row.totalPrice) }}
                      </span>
                    </template>
                  </el-table-column>
                  <el-table-column prop="price" label="成交价" width="120" />
                  <el-table-column prop="changerName" label="变动人" />
                  <el-table-column prop="changerPosition" label="职位" />
                </el-table>
                <StockHistoryDetail
                  v-model:visible="historyDetailVisible"
                  :stock-code="selectedStockCode"
                  :stock-name="selectedStockName"
                  :changer-names="selectedChangerNames"
                />
              </div>
            </div>
          </template>
        </el-table-column>

        <el-table-column prop="tradeDate" label="交易日期" width="120" />
        <el-table-column prop="stockCode" label="股票代码" width="120" />
        <el-table-column prop="stockName" label="股票名称" width="120" />
        <el-table-column prop="changeAmount" label="变动数量" width="160" sortable="custom">
          <template #default="scope">
            <span v-if="scope.row.totalIncrease != 0" class="increase">
              {{ formatNumber(scope.row.totalIncrease) }}
            </span>
            <span v-if="scope.row.totalDecrease != 0" class="decrease">
              {{ formatNumber(scope.row.totalDecrease) }}
            </span>
          </template>
        </el-table-column>
        <el-table-column prop="changerName" label="变动人" />
        <el-table-column prop="changerPosition" label="职位" />
      </el-table>

      <!-- 分页组件 -->
      <div class="pagination-container" v-if="totalCount > 0">
        <el-pagination
          v-model:current-page="currentPage"
          v-model:page-size="pageSize"
          :page-sizes="[10, 20, 50, 100]"
          layout="total, sizes, prev, pager, next, jumper"
          :total="totalCount"
          @size-change="handleSizeChange"
          @current-change="handleCurrentChange"
        />
      </div>

      <div class="empty-data" v-if="stockData.length === 0 && !loading">
        <el-empty description="暂无数据，请选择日期范围查询" />
      </div>
    </div>
  </div>
</template>

<script lang="ts">
import { ref, computed, onMounted, nextTick } from 'vue'
import axios from 'axios'
import { ElMessage } from 'element-plus'
import * as echarts from 'echarts'
import StockHistoryDetail from './components/StockHistoryDetail.vue'

export default {
  name: 'DataMonitor',
  // 注册组件
  components: {
    StockHistoryDetail,
  },
  setup() {
    interface StockDataItem {
      totalIncrease: number
      totalDecrease: number
      tradeDate: string
      stockCode: string
      stockName: string
      changerName: string
      changerPosition: string
      chartLoading?: boolean
      stockDetail?: StockDetailData | null
      expanded?: boolean // 添加展开状态跟踪
      price?: number // 添加价格属性
    }

    interface PriceDataItem {
      trackTime: string
      currentPrice: number
    }

    interface MarkItem {
      price: number
      tradeDate: string
      stockCode: string | null
      stockName: string | null
      changeType: string
      totalPrice: number
      changerName: string | null
      changerPosition: string | null
    }

    interface StockDetailData {
      priceData: PriceDataItem[]
      marks: MarkItem[]
    }

    // 在setup函数中添加新的响应式变量
    const changeType = ref('')
    const changeSort = ref('')
    const stockData = ref<StockDataItem[]>([])
    const loading = ref(false)
    const dateRange = ref<string[]>([])
    const chartInstances: Record<string, echarts.ECharts> = {}

    // 分页相关状态
    const currentPage = ref(1)
    const pageSize = ref(10)
    const totalCount = ref(0)

    // 计算当前页显示的数据
    const displayData = computed(() => {
      const startIndex = (currentPage.value - 1) * pageSize.value
      const endIndex = startIndex + pageSize.value
      return stockData.value.slice(startIndex, endIndex)
    })

    const shortcuts = [
      {
        text: '最近一周',
        value: () => {
          const end = new Date()
          const start = new Date()
          start.setTime(start.getTime() - 3600 * 1000 * 24 * 7)
          return [start, end]
        },
      },
      {
        text: '最近一个月',
        value: () => {
          const end = new Date()
          const start = new Date()
          start.setMonth(start.getMonth() - 1)
          return [start, end]
        },
      },
      {
        text: '最近三个月',
        value: () => {
          const end = new Date()
          const start = new Date()
          start.setMonth(start.getMonth() - 3)
          return [start, end]
        },
      },
    ]

    const headerStyle = {
      background: '#f5f7fa',
      color: '#303133',
      fontWeight: '600',
    }

    const fetchData = async () => {
      if (!dateRange.value || dateRange.value.length !== 2) {
        ElMessage.warning('请选择日期范围')
        return
      }

      loading.value = true
      try {
        const [startDate, endDate] = dateRange.value
        const params: Record<string, string | number> = {
          start: startDate,
          end: endDate,
        }

        // 添加变动类型筛选
        if (changeType.value) {
          params.changeType = changeType.value
        }

        // 添加排序参数
        if (changeSort.value) {
          params.changeSort = changeSort.value
        }

        const response = await axios.get(`http://116.205.244.106:8080/api/stocks/changes`, {
          params,
        })
        // 为每条记录添加额外属性
        stockData.value = response.data.map((item: StockDataItem) => ({
          ...item,
          chartLoading: false,
          stockDetail: null,
          expanded: false,
        }))
        // 更新总记录数
        totalCount.value = stockData.value.length

        // 重置到第一页
        currentPage.value = 1
      } catch (error) {
        console.error('获取数据失败', error)
        ElMessage.error('获取数据失败，请重试')
      } finally {
        loading.value = false
      }
    }

    const formatNumber = (num: number) => {
      return new Intl.NumberFormat('zh-CN').format(num)
    }

    const getIncreaseCount = () => {
      return stockData.value.filter((item) => item.totalIncrease > 0).length
    }

    const getDecreaseCount = () => {
      return stockData.value.filter((item) => item.totalDecrease > 0).length
    }

    const handleExpandChange = async (row: StockDataItem, expandedRows: StockDataItem[]) => {
      // 更新展开状态
      row.expanded = expandedRows.includes(row)

      if (row.expanded) {
        // 如果当前行被展开且没有股票详情数据，则获取数据
        if (!row.stockDetail) {
          await fetchStockDetail(row)
        } else {
          // 如果已经有数据，仅重新初始化图表
          nextTick(() => {
            initChart(row)
          })
        }
      }
    }

    // 添加排序处理函数
    const handleSortChange = (column: { prop: string; order: string | null }) => {
      console.log('排序:', column)
      if (column.prop === 'changeAmount') {
        if (column.order === 'ascending') {
          changeSort.value = 'asc'
        } else if (column.order === 'descending') {
          changeSort.value = 'desc'
        } else {
          changeSort.value = ''
        }
        fetchData()
      }
    }

    const fetchStockDetail = async (row: StockDataItem) => {
      // 标记为加载中
      row.chartLoading = true

      try {
        const response = await axios.get(
          `http://116.205.244.106:8080/api/stocks/${row.stockCode}/chart`,
        )
        row.stockDetail = response.data

        // 等待DOM更新后初始化图表
        nextTick(() => {
          initChart(row)
        })
      } catch (error) {
        console.error('获取股票详情失败', error)
        ElMessage.error('获取股票详情失败，请重试')
        row.stockDetail = null
      } finally {
        row.chartLoading = false
      }
    }

    const initChart = (row: StockDataItem) => {
      if (!row.stockDetail || !row.stockDetail.priceData) return

      // 获取图表DOM元素 - 使用组合ID确保唯一性
      const chartDomId = `chart-${row.stockCode}-${row.tradeDate}`
      const chartDom = document.getElementById(chartDomId)
      if (!chartDom) {
        console.error(`图表容器 ${chartDomId} 不存在`)
        return
      }

      // 销毁旧的图表实例
      if (chartInstances[chartDomId]) {
        chartInstances[chartDomId].dispose()
      }

      // 初始化图表
      chartInstances[chartDomId] = echarts.init(chartDom)

      // 准备数据
      const dates: string[] = []
      const prices: number[] = []
      const markData: {
        name: string
        coord: [string, number]
        value: string
        itemStyle: { color: string }
      }[] = []

      row.stockDetail.priceData.forEach((item) => {
        const date = new Date(item.trackTime).toLocaleDateString()
        const time = new Date(item.trackTime).toLocaleTimeString([], {
          hour: '2-digit',
          minute: '2-digit',
        })
        const dateTime = `${date} ${time}`

        dates.push(dateTime)
        prices.push(item.currentPrice)
      })

      // 处理增减持标记
      if (row.stockDetail.marks) {
        row.stockDetail.marks.forEach((mark) => {
          // 首先找到对应日期的所有价格数据点
          const sameDayPrices = row.stockDetail!.priceData.filter(
            (price) => new Date(price.trackTime).toISOString().split('T')[0] === mark.tradeDate,
          )

          if (sameDayPrices.length > 0) {
            // 如果标记有价格信息，找到最接近该价格的数据点
            let closestPriceData = sameDayPrices[0]
            let closestPriceIndex = row.stockDetail!.priceData.indexOf(closestPriceData)

            // 如果有明确的价格信息，查找最接近的价格点
            if (mark.price) {
              let minDiff = Math.abs(sameDayPrices[0].currentPrice - mark.price)

              for (const priceData of sameDayPrices) {
                const priceDiff = Math.abs(priceData.currentPrice - mark.price)
                if (priceDiff < minDiff) {
                  minDiff = priceDiff
                  closestPriceData = priceData
                  closestPriceIndex = row.stockDetail!.priceData.indexOf(priceData)
                }
              }
            }

            if (closestPriceIndex >= 0) {
              markData.push({
                name: mark.changeType,
                coord: [dates[closestPriceIndex], prices[closestPriceIndex]],
                value: `${formatNumber(mark.totalPrice)} (¥${mark.price})`,
                itemStyle: {
                  color: mark.changeType === '增持' ? '#f56c6c' : '#67c23a',
                },
                //symbolSize: 45, // 稍微增大标记点尺寸以显示更多信息
              })
            }
          }
        })
      }

      // 配置图表选项
      const option = {
        title: {
          text: `${row.stockName} (${row.stockCode}) 价格走势`,
          left: 'center',
          textStyle: {
            fontSize: 14,
          },
        },
        grid: {
          left: '3%',
          right: '4%',
          bottom: '3%',
          containLabel: true,
        },
        tooltip: {
          trigger: 'axis',
          formatter: function (params: { name: string; marker: string; value: number }[]) {
            let result = params[0].name + '<br/>'
            result += params[0].marker + ' 价格: ' + params[0].value

            // 查找是否有标记点
            const markInfo = markData.find((mark) => mark.coord[0] === params[0].name)
            if (markInfo) {
              result +=
                '<br/><span style="color:' +
                markInfo.itemStyle.color +
                '">● ' +
                markInfo.name +
                ': ' +
                markInfo.value +
                '</span>'
            }

            return result
          },
        },
        xAxis: {
          type: 'category',
          data: dates,
          axisLabel: {
            interval: Math.floor(dates.length / 8),
            formatter: function (value: string) {
              return value.split(' ')[0] + '\n' + value.split(' ')[1]
            },
          },
        },
        yAxis: {
          type: 'value',
          scale: true,
          axisLabel: {
            formatter: '{value}',
          },
        },
        dataZoom: [
          {
            type: 'inside',
            start: 0,
            end: 100,
          },
          {
            start: 0,
            end: 100,
          },
        ],
        series: [
          {
            type: 'line',
            data: prices,
            smooth: true,
            lineStyle: {
              width: 2,
              color: '#409EFF',
            },
            markPoint: {
              symbol: 'pin',
              symbolSize: 40,
              data: markData,
            },
          },
        ],
      }

      // 设置图表选项
      chartInstances[chartDomId].setOption(option)
      console.log(`图表 ${chartDomId} 已初始化`)
    }

    // 处理分页大小变化
    const handleSizeChange = (size: number) => {
      pageSize.value = size
      // 调整页码，确保数据正确显示
      if (currentPage.value > Math.ceil(totalCount.value / pageSize.value)) {
        currentPage.value = 1
      }
    }

    // 处理页码变化
    const handleCurrentChange = (page: number) => {
      currentPage.value = page
    }

    // 调整图表大小
    const handleResize = () => {
      Object.values(chartInstances).forEach((chart) => {
        chart.resize()
      })
    }

    onMounted(() => {
      // 设置默认日期为当前月份
      const end = new Date()
      const start = new Date()
      start.setDate(1) // 当月第一天
      dateRange.value = [start.toISOString().split('T')[0], end.toISOString().split('T')[0]]

      fetchData()

      // 添加窗口调整大小的监听
      window.addEventListener('resize', handleResize)
    })

    // 添加历史详情相关状态
    const historyDetailVisible = ref(false)
    const selectedStockCode = ref('')
    const selectedStockName = ref('')
    const selectedChangerNames = ref('')

    // 显示历史详情对话框
    const showHistoryDetail = (row: {
      stockCode: string
      stockName: string
      changerName: string
      stockDetail: { marks: MarkItem[] }
    }) => {
      selectedStockCode.value = row.stockCode
      selectedStockName.value = row.stockName

      // 如果有变动人信息，提取姓名
      if (row.changerName) {
        selectedChangerNames.value = row.changerName
      } else if (row.stockDetail && row.stockDetail.marks && row.stockDetail.marks.length > 0) {
        // 从marks中提取所有不同的changerName，用逗号分隔
        const uniqueChangers = [
          ...new Set(
            row.stockDetail.marks
              .filter((mark) => mark.changerName)
              .map((mark) => mark.changerName),
          ),
        ]
        selectedChangerNames.value = uniqueChangers.join(',')
      } else {
        selectedChangerNames.value = ''
      }

      historyDetailVisible.value = true
    }
    return {
      stockData,
      displayData,
      loading,
      dateRange,
      shortcuts,
      headerStyle,
      currentPage,
      pageSize,
      totalCount,
      fetchData,
      formatNumber,
      getIncreaseCount,
      getDecreaseCount,
      handleExpandChange,
      handleSizeChange,
      handleCurrentChange,
      // 添加新的返回值
      changeType,
      handleSortChange,
      // 添加新的返回值
      historyDetailVisible,
      selectedStockCode,
      selectedStockName,
      selectedChangerNames,
      showHistoryDetail,
    }
  },
}
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

table {
  table-layout: auto;
}

.table-container {
  width: auto;
  max-width: 100%;
  overflow-x: auto;
  margin-top: 20px;
  border-radius: 4px;
  overflow: hidden;
}

/* 分页容器样式 */
.pagination-container {
  margin-top: 20px;
  text-align: right;
  padding: 10px 0;
}

@media (max-width: 600px) {
  .table-container {
    overflow-x: scroll;
  }
}

.increase {
  color: #f56c6c;
  font-weight: 600;
}

.decrease {
  color: #67c23a;
  font-weight: 600;
}

.data-summary {
  display: flex;
  margin-top: 10px;
  background-color: #f5f7fa;
  border-radius: 8px;
  padding: 15px;
  margin-bottom: 20px;
}

.summary-item {
  display: flex;
  flex-direction: column;
  margin-right: 40px;
}

.summary-item .label {
  font-size: 14px;
  color: #606266;
}

.summary-item .value {
  font-size: 24px;
  font-weight: bold;
  color: #303133;
}

.empty-data {
  padding: 40px 0;
}

/* 股票详情行样式 */
.stock-detail-row {
  padding: 20px;
  margin-left: 50px;
  background-color: #fafafa;
  border-radius: 4px;
}

.chart-container {
  width: 100%;
  height: 300px;
  margin-bottom: 20px;
}

.stock-chart {
  width: 100%;
  height: 100%;
}

.marks-detail {
  margin-top: 20px;
}

.marks-detail h4 {
  margin-bottom: 16px;
  font-weight: 500;
  color: #303133;
  font-size: 15px;
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

  .data-summary {
    overflow-x: auto;
  }

  .chart-container {
    height: 250px;
  }
}
</style>
