<template>
  <el-dialog
    v-model="dialogVisible"
    :title="`${stockName} (${stockCode}) 所有历史数据`"
    width="80%"
    destroy-on-close
  >
    <div v-loading="loading">
      <div class="history-chart-container">
        <div id="history-chart" class="history-chart"></div>
      </div>

      <div
        class="history-marks-detail"
        v-if="stockDetail && stockDetail.marks && stockDetail.marks.length > 0"
      >
        <h4>历史增减持记录</h4>
        <el-table :data="stockDetail.marks" stripe size="small" style="width: 100%">
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
      </div>
    </div>
  </el-dialog>
</template>

<script lang="ts">
import { ref, onMounted, watch, nextTick } from 'vue'
import axios from 'axios'
import { ElMessage } from 'element-plus'
import * as echarts from 'echarts'

export default {
  name: 'StockHistoryDetail',
  props: {
    visible: Boolean,
    stockCode: String,
    stockName: String,
    changerNames: String,
  },
  emits: ['update:visible'],
  setup(props, { emit }) {
    const dialogVisible = ref(false)
    const loading = ref(false)
    const stockDetail = ref<{
      marks?: { tradeDate: string; changeType: string; totalPrice: number; price?: number }[]
      priceData?: { trackTime: string; currentPrice: number }[]
    } | null>(null)
    let chartInstance: echarts.ECharts | null = null

    // 监听visible属性变化
    watch(
      () => props.visible,
      (newValue) => {
        dialogVisible.value = newValue
        if (newValue) {
          fetchHistoryData()
        }
      },
    )

    // 监听对话框关闭
    watch(
      () => dialogVisible.value,
      (newValue) => {
        if (!newValue) {
          emit('update:visible', false)
        }
      },
    )

    const fetchHistoryData = async () => {
      if (!props.stockCode) return

      loading.value = true
      try {
        const response = await axios.get(
          `http://116.205.244.106:8080/data/api/detail/chart?code=${props.stockCode}&name=${props.changerNames || ''}`,
        )
        stockDetail.value = response.data

        nextTick(() => {
          initChart()
        })
      } catch (error) {
        console.error('获取历史数据失败', error)
        ElMessage.error('获取历史数据失败，请重试')
      } finally {
        loading.value = false
      }
    }

    const initChart = () => {
      if (!stockDetail.value || !stockDetail.value.priceData) return

      const chartDom = document.getElementById('history-chart')
      if (!chartDom) {
        console.error('图表容器不存在')
        return
      }

      // 销毁旧的图表实例
      if (chartInstance) {
        chartInstance.dispose()
      }

      // 初始化图表
      chartInstance = echarts.init(chartDom)

      // 准备数据
      const dates: string[] = []
      const prices: number[] = []
      const markData: {
        name: string
        coord: [string, number]
        value: string
        itemStyle: { color: string }
      }[] = []

      stockDetail.value.priceData.forEach((item) => {
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
      if (stockDetail.value.marks) {
        stockDetail.value.marks.forEach((mark) => {
          // 首先找到对应日期的所有价格数据点
          const sameDayPrices =
            stockDetail.value?.priceData?.filter(
              (price) => new Date(price.trackTime).toISOString().split('T')[0] === mark.tradeDate,
            ) || []

          if (sameDayPrices.length > 0) {
            // 如果标记有价格信息，找到最接近该价格的数据点
            let closestPriceData = sameDayPrices[0]
            let closestPriceIndex = stockDetail.value?.priceData?.indexOf(closestPriceData) ?? -1

            // 如果有明确的价格信息，查找最接近的价格点
            if (mark.price) {
              let minDiff = Math.abs(sameDayPrices[0].currentPrice - mark.price)

              for (const priceData of sameDayPrices) {
                const priceDiff = Math.abs(priceData.currentPrice - mark.price)
                if (priceDiff < minDiff) {
                  minDiff = priceDiff
                  closestPriceData = priceData
                  closestPriceIndex = stockDetail.value?.priceData?.indexOf(priceData) ?? -1
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
              })
            }
          }
        })
      }

      // 配置图表选项
      const option = {
        title: {
          text: `${props.stockName} (${props.stockCode}) 价格走势`,
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
      chartInstance.setOption(option)
    }

    const formatNumber = (num: number | bigint) => {
      return new Intl.NumberFormat('zh-CN').format(num)
    }

    // 监听窗口大小变化
    const handleResize = () => {
      if (chartInstance) {
        chartInstance.resize()
      }
    }

    onMounted(() => {
      window.addEventListener('resize', handleResize)
    })

    return {
      dialogVisible,
      loading,
      stockDetail,
      formatNumber,
    }
  },
}
</script>

<style scoped>
.history-chart-container {
  width: 100%;
  height: 400px;
  margin-bottom: 20px;
}

.history-chart {
  width: 100%;
  height: 100%;
}

.history-marks-detail {
  margin-top: 20px;
}

.history-marks-detail h4 {
  margin-bottom: 16px;
  font-weight: 500;
  color: #303133;
  font-size: 15px;
}

.increase {
  color: #f56c6c;
  font-weight: 600;
}

.decrease {
  color: #67c23a;
  font-weight: 600;
}
</style>
