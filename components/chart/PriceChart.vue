<template>
  <div class="common-price-chart">
    <span class="chart-y-description is-size-7">Price ({{ unit }})</span>
    <b-dropdown aria-role="list" class="time-range-dropdown py-0">
      <template #trigger>
        <div
          class="time-range-button is-flex is-justify-content-center is-align-items-center">
          {{ selectedTimeRange.label }}
        </div>
      </template>
      <b-dropdown-item
        v-for="range in timeRangeList"
        :key="range.value"
        class="is-flex is-justify-content-center px-0"
        aria-role="listitem"
        :value="selectedTimeRange"
        :class="{ 'is-active': selectedTimeRange.value === range.value }"
        @click="setTimeRange(range)">
        {{ range.label }}
      </b-dropdown-item>
    </b-dropdown>

    <div class="content">
      <canvas id="priceChart" />
    </div>
  </div>
</template>

<script lang="ts" setup>
import ChartJS from 'chart.js/auto'
import 'chartjs-adapter-date-fns'
import zoomPlugin from 'chartjs-plugin-zoom'
import { getChartData } from '@/utils/chart'
import { format } from 'date-fns'

ChartJS.register(zoomPlugin)
const { $i18n, $colorMode } = useNuxtApp()
const { unit } = useChain()

const daysTranslation = (day: number) => $i18n.t('priceChart.days', [day])

const timeRangeList = [
  {
    value: 0,
    label: $i18n.t('priceChart.all'),
  },
  {
    value: 14,
    label: daysTranslation(14),
  },
  {
    value: 30,
    label: daysTranslation(30),
  },
  {
    value: 90,
    label: daysTranslation(90),
  },
]

const selectedTimeRange = ref(timeRangeList[0])

const setTimeRange = (value: { value: number; label: string }) => {
  selectedTimeRange.value = value
}
const isDarkMode = computed(
  () =>
    $colorMode.preference === 'dark' ||
    document.documentElement.className.includes('dark-mode')
)

const props = defineProps<{
  priceChartData?: [Date, number][][]
}>()
let Chart: ChartJS<'line', any, unknown>

onMounted(() => {
  window.addEventListener('resize', onWindowResize)
  getPriceChartData()
})

onBeforeUnmount(() => {
  window.removeEventListener('resize', onWindowResize)
})

const lineColor = computed(() => {
  if (isDarkMode.value) {
    return 'white'
  } else {
    return '#181717'
  }
})
const displayChartData = computed(() => {
  if (props.priceChartData) {
    const timeRangeValue = selectedTimeRange.value.value
    return [
      getChartDataByTimeRange(props.priceChartData[0], timeRangeValue),
      getChartDataByTimeRange(props.priceChartData[1], timeRangeValue),
    ]
  } else {
    return []
  }
})

const getChartDataByTimeRange = (data: [Date, number][], timeRange: number) => {
  if (!data) {
    return
  }
  if (timeRange === 0) {
    return data
  } else {
    const now = new Date()
    const startDate = new Date(
      now.getFullYear(),
      now.getMonth(),
      now.getDate() - selectedTimeRange.value.value
    )
    return data.filter((item) => item[0] >= startDate)
  }
}

const getPriceChartData = () => {
  Chart?.destroy()
  const priceChartData = displayChartData.value

  if (priceChartData?.length) {
    const ctx = (
      document?.getElementById('priceChart') as HTMLCanvasElement
    )?.getContext('2d')
    if (ctx) {
      const commonStyle = {
        tension: 0,
        pointRadius: 6,
        pointHoverRadius: 6,
        pointHoverBackgroundColor: isDarkMode.value ? '#181717' : 'white',
        borderJoinStyle: 'miter' as const,
        radius: 0,
        pointStyle: 'rect',
        borderWidth: 1,
      }
      const chart = new ChartJS(ctx, {
        type: 'line',
        data: {
          datasets: [
            {
              label: 'Sale',
              data: getChartData(priceChartData[0]),
              borderColor: '#FF7AC3',
              pointBackgroundColor: '#FF7AC3',
              pointBorderColor: '#FF7AC3',
              ...commonStyle,
            },
            {
              label: 'List',
              data: getChartData(priceChartData[1]),
              borderColor: '#6188E7',
              pointBackgroundColor: '#6188E7',
              pointBorderColor: '#6188E7',
              ...commonStyle,
            },
          ],
        },
        options: {
          maintainAspectRatio: false,
          plugins: {
            customCanvasBackgroundColor: {
              color: isDarkMode.value ? '#181717' : 'white',
            },
            tooltip: {
              callbacks: {
                label: function (context) {
                  return `Price: ${context.parsed.y}${unit.value}`
                },
                title: function (context) {
                  return format(context[0].parsed.x, 'MMM dd HH:mm')
                },
              },
            },
            zoom: {
              limits: {
                x: { min: 0, minRange: 0 },
                y: { min: 0, minRange: 0 },
              },
              pan: {
                enabled: false,
              },
              zoom: {
                wheel: {
                  enabled: false,
                },
                pinch: {
                  enabled: false,
                },
                mode: 'xy',
                onZoomComplete({ chart }) {
                  chart.update('none')
                },
              },
            },
          },
          scales: {
            x: {
              type: 'time',
              time: {
                displayFormats: {
                  hour: 'HH:mm',
                  minute: 'HH:mm',
                },
                unit: 'hour',
              },
              grid: {
                drawOnChartArea: false,
                borderColor: lineColor.value,
                color: lineColor.value,
              },
              ticks: {
                callback: (value) => {
                  return value
                },
                major: {
                  enabled: true,
                },
                maxRotation: 0,
                minRotation: 0,
                color: lineColor.value,
              },
            },
            y: {
              ticks: {
                callback: (value) => {
                  return `${Number(value).toFixed(2)}  `
                },
                maxTicksLimit: 7,
                color: lineColor.value,
              },
              grid: {
                drawTicks: false,
                color: '#ccc',
                borderColor: lineColor.value,
              },
            },
          },
        },
        plugins: [
          {
            id: 'customCanvasBackgroundColor',
            beforeDraw: (chart, args, options) => {
              const { ctx } = chart
              ctx.save()
              ctx.globalCompositeOperation = 'destination-over'
              ctx.fillStyle = options.color || '#FFFFFF'
              ctx.fillRect(0, 0, chart.width, chart.height)
              ctx.restore()
            },
          },
        ],
      })

      Chart = chart
    }
  }
}
watch(
  () => props.priceChartData,
  () => {
    getPriceChartData()
  }
)
watch([isDarkMode, selectedTimeRange], () => {
  getPriceChartData()
})

const onWindowResize = () => {
  Chart?.resize()
}
</script>

<style scoped>
.content {
  height: 15rem;
}
</style>
