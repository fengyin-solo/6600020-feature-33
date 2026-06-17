<template>
  <div class="flex h-screen">
    <!-- Left Sidebar: Control + Device List -->
    <div class="w-64 bg-gray-900 p-4 flex flex-col gap-3 border-r border-gray-800 overflow-y-auto flex-shrink-0">
      <h1 class="text-lg font-bold text-orange-400">Modbus 工业监控</h1>
      <div class="flex gap-2">
        <button @click="startPoll" :disabled="store.isPolling" class="flex-1 bg-green-700 py-1.5 rounded text-xs hover:bg-green-600 disabled:opacity-50">
          {{ store.isPolling ? '采集中...' : '开始采集' }}
        </button>
        <button @click="stopPoll" :disabled="!store.isPolling" class="flex-1 bg-red-700 py-1.5 rounded text-xs hover:bg-red-600 disabled:opacity-50">
          停止
        </button>
      </div>
      <div>
        <label class="text-gray-400 text-xs">轮询间隔: {{ store.pollInterval }}ms</label>
        <input type="range" v-model.number="store.pollInterval" min="200" max="5000" step="100" class="w-full" />
      </div>

      <h3 class="text-gray-400 text-xs mt-2">设备列表</h3>
      <div v-for="d in store.devices" :key="d.id" @click="store.selectedDevice = d"
        class="bg-gray-800 rounded p-2 cursor-pointer text-sm transition-all"
        :class="store.selectedDevice?.id === d.id ? 'ring-1 ring-orange-500 bg-gray-700' : 'hover:bg-gray-700'">
        <div class="flex justify-between">
          <span>{{ d.name }}</span>
          <span class="w-2 h-2 rounded-full mt-1.5" :class="d.online ? 'bg-green-500' : 'bg-red-500'"></span>
        </div>
        <div class="text-xs text-gray-500">{{ d.ip }}:{{ d.port }} [{{ d.slaveId }}]</div>
      </div>

      <div v-if="store.criticalAlarms.length" class="bg-red-900/50 rounded p-2 mt-2">
        <h4 class="text-red-400 text-xs font-bold">⚠ 严重告警 {{ store.criticalAlarms.length }}</h4>
        <div v-for="a in store.criticalAlarms.slice(0, 3)" :key="a.id" class="text-xs text-red-300 mt-1 truncate">
          {{ a.message }}
        </div>
      </div>

      <div class="text-xs text-gray-600 mt-auto">
        在线: {{ store.onlineDevices.length }}/{{ store.devices.length }}
      </div>
    </div>

    <!-- Middle Area: Topology + Trend Chart -->
    <div class="flex-1 flex flex-col gap-3 p-4 overflow-hidden min-w-0">
      <!-- Topology Graph -->
      <div class="bg-gray-900 rounded-xl p-3 h-80 flex-shrink-0">
        <h3 class="text-sm text-gray-400 mb-2">
          网络拓扑 — 点击节点查看详情
        </h3>
        <div class="h-[calc(100%-28px)]">
          <TopologyGraph />
        </div>
      </div>

      <!-- Trend Chart -->
      <div class="bg-gray-900 rounded-xl p-3 flex-1 min-h-0">
        <h3 class="text-sm text-gray-400 mb-2">
          实时趋势 — {{ store.selectedDevice?.name || '请选择设备' }}
        </h3>
        <div class="h-[calc(100%-28px)]">
          <TrendChart />
        </div>
      </div>

      <!-- Alarm List -->
      <div class="bg-gray-900 rounded-xl p-3 max-h-44 overflow-y-auto flex-shrink-0">
        <h3 class="text-sm text-gray-400 mb-2">告警记录</h3>
        <div v-for="a in store.alarms.slice(0, 10)" :key="a.id"
          class="flex justify-between text-xs bg-gray-800 rounded p-2 mb-1"
          :class="{ 'border-l-4 border-red-500': a.level === 'critical', 'border-l-4 border-yellow-500': a.level === 'warning' }">
          <span>{{ a.message }}</span>
          <div class="flex gap-2 items-center">
            <span class="text-gray-500">{{ new Date(a.timestamp).toLocaleTimeString() }}</span>
            <button v-if="!a.acknowledged" @click="store.acknowledgeAlarm(a.id)" class="text-blue-400 hover:underline whitespace-nowrap">确认</button>
          </div>
        </div>
        <div v-if="!store.alarms.length" class="text-xs text-gray-600 text-center py-2">暂无告警</div>
      </div>
    </div>

    <!-- Right Sidebar: Device Details -->
    <div class="w-80 bg-gray-900 p-4 flex flex-col gap-3 border-l border-gray-800 overflow-y-auto flex-shrink-0">
      <template v-if="store.selectedDevice">
        <div class="flex items-center justify-between">
          <h3 class="text-sm font-bold text-orange-400">设备详情</h3>
          <span class="text-xs px-2 py-0.5 rounded" :class="store.selectedDevice.online ? 'bg-green-900 text-green-400' : 'bg-red-900 text-red-400'">
            {{ store.selectedDevice.online ? '在线' : '离线' }}
          </span>
        </div>

        <div class="bg-gray-800 rounded p-3">
          <div class="text-lg font-bold text-white mb-2">{{ store.selectedDevice.name }}</div>
          <div class="grid grid-cols-2 gap-2 text-xs">
            <div>
              <span class="text-gray-500">ID</span>
              <div class="text-gray-300 font-mono">{{ store.selectedDevice.id }}</div>
            </div>
            <div>
              <span class="text-gray-500">Slave ID</span>
              <div class="text-gray-300 font-mono">{{ store.selectedDevice.slaveId }}</div>
            </div>
            <div class="col-span-2">
              <span class="text-gray-500">地址</span>
              <div class="text-gray-300 font-mono">{{ store.selectedDevice.ip }}:{{ store.selectedDevice.port }}</div>
            </div>
          </div>
        </div>

        <h4 class="text-xs text-gray-400 mt-1">寄存器测点</h4>
        <div v-for="r in store.selectedDevice.registers" :key="r.address"
          class="bg-gray-800 rounded p-3">
          <div class="flex justify-between items-center mb-1">
            <span class="text-sm text-gray-300 font-medium">{{ r.name }}</span>
            <span class="text-xs px-1.5 py-0.5 rounded bg-gray-700 text-gray-400 font-mono">0x{{ r.address.toString(16).padStart(2, '0') }}</span>
          </div>
          <div class="flex justify-between items-end">
            <div>
              <span class="text-2xl font-bold" :class="store.selectedDevice.online ? 'text-orange-400' : 'text-gray-600'">
                {{ typeof r.value === 'number' ? r.value.toFixed(r.value > 100 ? 0 : 2) : r.value ? 'ON' : 'OFF' }}
              </span>
              <span class="text-xs text-gray-500 ml-1">{{ r.unit }}</span>
            </div>
            <span class="text-xs text-gray-600">
              {{ new Date(r.updatedAt).toLocaleTimeString() }}
            </span>
          </div>
          <div class="flex items-center gap-2 mt-2">
            <span class="text-xs px-1.5 py-0.5 rounded"
              :class="{
                'bg-blue-900/50 text-blue-400': r.type === 'holding',
                'bg-cyan-900/50 text-cyan-400': r.type === 'input',
                'bg-yellow-900/50 text-yellow-400': r.type === 'coil',
                'bg-purple-900/50 text-purple-400': r.type === 'discrete'
              }">
              {{ r.type }}
            </span>
          </div>
        </div>

        <!-- Register Gauges Summary -->
        <h4 class="text-xs text-gray-400 mt-1">数据概览</h4>
        <div class="grid grid-cols-2 gap-2">
          <div v-for="r in store.selectedDevice.registers" :key="'gauge-'+r.address"
            class="bg-gray-800 rounded p-2">
            <div class="text-xs text-gray-500 truncate">{{ r.name }}</div>
            <div class="text-lg font-bold" :class="store.selectedDevice.online ? 'text-orange-400' : 'text-gray-600'">
              {{ typeof r.value === 'number' ? r.value.toFixed(r.value > 100 ? 0 : 1) : r.value ? 'ON' : 'OFF' }}
            </div>
            <div class="text-xs text-gray-600">{{ r.unit }}</div>
          </div>
        </div>
      </template>

      <div v-else class="flex-1 flex items-center justify-center">
        <div class="text-center text-gray-600">
          <div class="text-4xl mb-3">📡</div>
          <div class="text-sm">请从左侧列表或拓扑图</div>
          <div class="text-sm">选择一个设备</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { onMounted, onUnmounted } from 'vue'
import { useModbusStore } from './store/modbus'
import TrendChart from './components/TrendChart.vue'
import TopologyGraph from './components/TopologyGraph.vue'

const store = useModbusStore()
let timer: number | null = null

function startPoll() {
  store.isPolling = true
  timer = window.setInterval(() => store.simulatePoll(), store.pollInterval)
}

function stopPoll() {
  store.isPolling = false
  if (timer) { clearInterval(timer); timer = null }
}

onMounted(() => store.initMockDevices())
onUnmounted(() => stopPoll())
</script>
