<template>
  <v-chart v-if="chartOption" :option="chartOption" class="h-full" autoresize @click="handleClick" />
</template>

<script setup lang="ts">
import { computed } from 'vue'
import { use } from 'echarts/core'
import { CanvasRenderer } from 'echarts/renderers'
import { GraphChart } from 'echarts/charts'
import { TooltipComponent } from 'echarts/components'
import VChart from 'vue-echarts'
import { useModbusStore } from '../store/modbus'
import type { EChartsOption } from 'echarts'

use([CanvasRenderer, GraphChart, TooltipComponent])

const store = useModbusStore()

const chartOption = computed<EChartsOption | null>(() => {
  const devices = store.devices
  if (!devices.length) return null

  const categories = [
    { name: '在线' },
    { name: '离线' }
  ]

  const nodes = devices.map((dev, index) => {
    const angle = (index / devices.length) * Math.PI * 2 - Math.PI / 2
    const radius = 120
    const x = 250 + Math.cos(angle) * radius
    const y = 180 + Math.sin(angle) * radius

    return {
      id: dev.id,
      name: dev.name,
      x,
      y,
      symbolSize: 60,
      category: dev.online ? 0 : 1,
      value: dev.registers.length + ' 测点',
      itemStyle: {
        color: dev.online ? '#22c55e' : '#6b7280',
        borderColor: store.selectedDevice?.id === dev.id ? '#f97316' : '#374151',
        borderWidth: store.selectedDevice?.id === dev.id ? 4 : 2,
        shadowBlur: store.selectedDevice?.id === dev.id ? 20 : 0,
        shadowColor: store.selectedDevice?.id === dev.id ? '#f97316' : 'transparent'
      },
      label: {
        show: true,
        position: 'bottom',
        distance: 10,
        fontSize: 11,
        color: '#d1d5db',
        formatter: (params: any) => {
          const maxLen = 8
          const name = params.name
          return name.length > maxLen ? name.slice(0, maxLen) + '...' : name
        }
      }
    }
  })

  const centerNode = {
    id: 'center',
    name: '主控中心',
    x: 250,
    y: 180,
    symbolSize: 75,
    category: 0,
    value: 'Gateway',
    itemStyle: {
      color: '#f97316',
      borderColor: '#ea580c',
      borderWidth: 2
    },
    label: {
      show: true,
      position: 'inside',
      fontSize: 12,
      fontWeight: 'bold',
      color: '#fff'
    }
  }

  const links = devices.map(dev => ({
    source: 'center',
    target: dev.id,
    lineStyle: {
      color: dev.online ? '#22c55e66' : '#6b728044',
      width: dev.online ? 2 : 1,
      type: dev.online ? 'solid' : 'dashed'
    }
  }))

  return {
    tooltip: {
      trigger: 'item',
      backgroundColor: '#1f2937',
      borderColor: '#374151',
      textStyle: { color: '#f3f4f6' },
      formatter: (params: any) => {
        if (params.dataType === 'node' && params.data.id !== 'center') {
          const dev = store.devices.find(d => d.id === params.data.id)
          if (dev) {
            return `
              <div style="font-weight:bold;margin-bottom:4px;">${dev.name}</div>
              <div style="font-size:12px;color:#9ca3af;">IP: ${dev.ip}:${dev.port}</div>
              <div style="font-size:12px;color:#9ca3af;">SlaveID: ${dev.slaveId}</div>
              <div style="font-size:12px;color:${dev.online ? '#22c55e' : '#ef4444'};">状态: ${dev.online ? '在线' : '离线'}</div>
              <div style="font-size:12px;color:#9ca3af;">测点数: ${dev.registers.length}</div>
            `
          }
        }
        return params.name
      }
    },
    animationDurationUpdate: 500,
    animationEasingUpdate: 'quinticInOut',
    categories,
    legend: [{
      data: categories.map(c => c.name),
      textStyle: { color: '#9ca3af' },
      top: 10,
      right: 20,
      icon: 'circle',
      itemWidth: 10,
      itemHeight: 10
    }],
    series: [
      {
        type: 'graph',
        layout: 'none',
        roam: true,
        draggable: true,
        focusNodeAdjacency: true,
        data: [centerNode, ...nodes],
        links,
        lineStyle: {
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
  if (params.dataType === 'node' && params.data.id !== 'center') {
    const dev = store.devices.find(d => d.id === params.data.id)
    if (dev) {
      store.selectedDevice = dev
    }
  }
}
</script>
