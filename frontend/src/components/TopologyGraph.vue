<template>
  <div class="w-full h-full relative">
    <v-chart
      ref="chartRef"
      :option="chartOption"
      autoresize
      class="w-full h-full"
      @click="handleClick"
    />
    <div v-if="!store.devices.length" class="absolute inset-0 flex items-center justify-center text-gray-600 text-sm bg-gray-900">
      暂无设备数据
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, computed, watch } from 'vue'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { GraphChart } from 'echarts/charts'
import { TooltipComponent, LegendComponent } from 'echarts/components'
import VChart from 'vue-echarts'
import { useModbusStore } from '../store/modbus'

use([CanvasRenderer, GraphChart, TooltipComponent, LegendComponent])

const store = useModbusStore()
const chartRef = ref<any>(null)

const chartOption = computed<any>(() => {
  const devices = store.devices

  const nodes: any[] = [
    {
      id: 'center',
      name: '主控中心',
      symbolSize: 60,
      fixed: true,
      value: 'Gateway',
      x: 0,
      y: 0,
      itemStyle: {
        color: '#f97316',
        borderColor: '#ea580c',
        borderWidth: 2
      },
      label: {
        show: true,
        position: 'inside',
        fontSize: 11,
        fontWeight: 'bold',
        color: '#fff'
      }
    }
  ]

  devices.forEach((dev) => {
    const isSelected = store.selectedDevice?.id === dev.id
    nodes.push({
      id: dev.id,
      name: dev.name,
      symbolSize: isSelected ? 55 : 45,
      value: `${dev.registers.length} 测点`,
      itemStyle: {
        color: dev.online ? '#22c55e' : '#6b7280',
        borderColor: isSelected ? '#f97316' : '#374151',
        borderWidth: isSelected ? 4 : 2,
        shadowBlur: isSelected ? 20 : 0,
        shadowColor: isSelected ? '#f97316' : 'transparent'
      },
      label: {
        show: true,
        position: 'bottom',
        distance: 8,
        fontSize: 10,
        color: '#d1d5db',
        formatter: (params: any) => {
          const name = params.name || ''
          return name.length > 8 ? name.slice(0, 8) + '...' : name
        }
      }
    })
  })

  const links = devices.map((dev) => ({
    source: 'center',
    target: dev.id,
    lineStyle: {
      color: dev.online ? '#22c55e88' : '#6b728044',
      width: dev.online ? 2 : 1,
      type: dev.online ? 'solid' : 'dashed'
    }
  }))

  return {
    backgroundColor: 'transparent',
    tooltip: {
      trigger: 'item',
      backgroundColor: '#1f2937',
      borderColor: '#374151',
      borderWidth: 1,
      textStyle: { color: '#f3f4f6', fontSize: 12 },
      formatter: (params: any) => {
        if (params.dataType === 'node') {
          if (params.data.id === 'center') {
            return `<div style="font-weight:bold;color:#f97316;">主控中心</div>
                    <div style="font-size:11px;color:#9ca3af;">Modbus Gateway</div>`
          }
          const dev = store.devices.find((d) => d.id === params.data.id)
          if (dev) {
            return `
              <div style="font-weight:bold;margin-bottom:4px;">${dev.name}</div>
              <div style="font-size:11px;color:#9ca3af;">IP: ${dev.ip}:${dev.port}</div>
              <div style="font-size:11px;color:#9ca3af;">SlaveID: ${dev.slaveId}</div>
              <div style="font-size:11px;color:${dev.online ? '#22c55e' : '#ef4444'};">状态: ${dev.online ? '在线' : '离线'}</div>
              <div style="font-size:11px;color:#9ca3af;">测点数: ${dev.registers.length}</div>
            `
          }
        }
        return params.name
      }
    },
    series: [
      {
        type: 'graph',
        layout: 'force',
        force: {
          repulsion: 250,
          edgeLength: 90,
          gravity: 0.2
        },
        roam: true,
        draggable: true,
        focusNodeAdjacency: true,
        data: nodes,
        links: links,
        lineStyle: {
          color: '#4b5563',
          width: 1,
          curveness: 0
        },
        emphasis: {
          focus: 'adjacency',
          lineStyle: {
            width: 3
          },
          itemStyle: {
            shadowBlur: 15,
            shadowColor: '#f97316'
          }
        }
      }
    ]
  }
})

function handleClick(params: any) {
  if (params && params.data && params.dataType === 'node' && params.data.id !== 'center') {
    const dev = store.devices.find((d) => d.id === params.data.id)
    if (dev) {
      store.selectedDevice = dev
    }
  }
}

watch(
  () => store.selectedDevice,
  () => {
    if (chartRef?.value?.getEchartsInstance) {
      const instance = chartRef.value.getEchartsInstance()
      instance.dispatchAction({
        type: 'highlight',
        seriesIndex: 0,
        name: store.selectedDevice?.name
      })
      setTimeout(() => {
        instance.dispatchAction({
          type: 'downplay',
          seriesIndex: 0,
          name: store.selectedDevice?.name
        })
      }, 300)
    }
  }
)
</script>
