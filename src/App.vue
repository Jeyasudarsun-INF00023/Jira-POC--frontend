<template>
  <q-layout view="lHh Lpr lFf" class="dashboard-root">
    <!-- Header with Glassmorphism -->
    <q-header class="header-glass text-white shadow-10">
      <q-toolbar class="q-py-sm">
        <div class="row items-center no-wrap">
          <div class="logo-container q-mr-md">
            <q-icon name="hub" size="md" class="logo-icon pulse-blue" />
          </div>
          <div>
            <div class="text-h6 text-weight-bold tracking-tight">INCIDENT<span class="text-accent">.</span>AI</div>
            <div class="text-caption text-grey-4 opacity-70 font-size-10">Jira Incident Automation Hub</div>
          </div>
        </div>
        <q-space />
        <div class="row q-gutter-md items-center">
          <div class="connection-status row items-center no-wrap bg-dark-layer q-px-md q-py-xs rounded-borders shadow-2">
            <div :class="`status-dot q-mr-sm ${jiraError ? 'bg-negative' : 'bg-positive'}`"></div>
            <span class="text-caption text-weight-medium">{{ jiraError ? 'Connection Failed' : 'System Operational' }}</span>
          </div>
          <q-btn
            round
            flat
            dense
            class="refresh-btn bg-dark-layer"
            icon="refresh"
            @click="fetchIncidents"
            :loading="loading"
          >
            <q-tooltip class="bg-dark text-white">Refresh Backlog</q-tooltip>
          </q-btn>
        </div>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <q-page padding class="q-col-gutter-md">
        <!-- Error Banner -->
        <q-banner v-if="jiraError" dense class="banner-error text-white q-mb-md rounded-xl shadow-14">
          <template v-slot:avatar>
            <q-icon name="wifi_off" color="white" size="sm" />
          </template>
          <div class="text-caption text-weight-bold">Jira sync error. Check API credentials in .env.</div>
          <template v-slot:action>
            <q-btn flat label="Retry" @click="fetchIncidents" color="white" size="sm" />
          </template>
        </q-banner>

        <!-- KPI Summary row -->
        <div class="row q-col-gutter-sm">
          <div class="col-12 col-sm-6 col-md-3" v-for="card in summaryCards" :key="card.title">
            <q-card class="stat-card shadow-14 overflow-hidden" :class="card.class">
              <q-card-section class="q-pa-md">
                <div class="row items-center no-wrap justify-between">
                  <div>
                    <div class="text-caption text-uppercase tracking-wider opacity-60 font-size-10">{{ card.title }}</div>
                    <div class="text-h4 text-weight-bolder tabular-nums">{{ card.value }}</div>
                  </div>
                  <q-icon :name="card.icon" size="sm" class="opacity-40" />
                </div>
              </q-card-section>
              <div class="card-bottom-accent"></div>
            </q-card>
          </div>
        </div>

        <!-- Main Data Table Container -->
        <div class="row q-mt-md">
          <div class="col-12">
            <q-table
              :rows="incidents"
              :columns="columns"
              row-key="key"
              dark
              flat
              bordered
              :pagination="{ rowsPerPage: 12 }"
              class="incident-table shadow-20"
              no-data-label="No active incidents in Jira backlog."
              dense
              :grid="$q.screen.lt.md"
            >
              <template v-slot:top>
                <div class="row full-width items-center q-px-sm q-gutter-y-sm">
                  <div class="row items-center col-12 col-sm-auto">
                    <div class="text-subtitle1 text-weight-bold text-accent">Service Backlog</div>
                    <q-badge outline color="accent" class="q-ml-md p-badge q-px-sm">LIVE</q-badge>
                  </div>
                  <q-space class="gt-xs" />
                  <!-- <q-input
                    v-model="filter"
                    dense
                    outlined
                    placeholder="Search backlog..."
                    class="search-input col-12 col-sm-auto"
                  >
                    <template v-slot:append>
                      <q-icon name="search" size="xs" />
                    </template>
                  </q-input> -->
                </div>
              </template>

              <!-- Custom Cell Renderers -->
              <template v-slot:body-cell-key="props">
                <q-td :props="props">
                  <span class="text-accent text-weight-bold">{{ props.value }}</span>
                </q-td>
              </template>

              <template v-slot:body-cell-status="props">
                <q-td :props="props">
                  <q-chip
                    :class="['status-chip', getStatusClass(props.value)]"
                    text-color="white"
                    size="10px"
                    dense
                  >
                    <q-spinner-dots v-if="isProcessing(props.value)" size="10px" class="q-mr-xs" />
                    {{ props.value }}
                  </q-chip>
                </q-td>
              </template>

              <template v-slot:body-cell-priority="props">
                <q-td :props="props">
                  <div class="row items-center no-wrap">
                    <div :class="['p-dot q-mr-sm', getPriorityColor(props.value)]"></div>
                    <span class="text-caption">{{ props.value }}</span>
                  </div>
                </q-td>
              </template>

              <template v-slot:body-cell-confidence="props">
                <q-td :props="props">
                  <q-linear-progress
                    :value="getConfidenceValue(props.value)"
                    :color="getConfidenceColor(props.value)"
                    size="4px"
                    rounded
                    class="q-mt-xs shadow-2"
                  />
                  <div class="text-tiny q-mt-xs opacity-60 text-uppercase">{{ props.value }}</div>
                </q-td>
              </template>

              <template v-slot:body-cell-actions="props">
                <q-td :props="props">
                  <div class="row q-gutter-xs justify-center no-wrap">
                    <q-btn
                      round
                      dense
                      flat
                      color="accent"
                      icon="auto_awesome"
                      size="sm"
                      @click="analyzeAndFix(props.row.key)"
                      :disable="!['To Do', 'Received'].includes(props.row.status)"
                    >
                      <q-tooltip class="bg-accent">AI Auto-Fix</q-tooltip>
                    </q-btn>
                    
                    <q-btn
                      round
                      dense
                      flat
                      color="warning"
                      icon="refresh"
                      size="sm"
                      @click="retryIncident(props.row.key)"
                      :disable="props.row.status === 'Resolved'"
                    >
                      <q-tooltip class="bg-warning text-dark">Retry Flow</q-tooltip>
                    </q-btn>

                    <q-btn
                      round
                      dense
                      flat
                      color="negative"
                      icon="person_search"
                      size="sm"
                      @click="escalateIncident(props.row.key)"
                      :disable="props.row.status === 'Resolved'"
                    >
                      <q-tooltip class="bg-negative">Escalate</q-tooltip>
                    </q-btn>

                    <q-btn
                      round
                      dense
                      flat
                      color="grey-7"
                      icon="delete"
                      size="sm"
                      @click="deleteIncident(props.row.key)"
                      class="action-btn-hover"
                    >
                      <q-tooltip class="bg-dark text-white">Dismiss / Delete</q-tooltip>
                    </q-btn>
                  </div>
                </q-td>
              </template>

              <template v-slot:body-cell-timestamp="props">
                <q-td :props="props">
                  <div class="text-caption text-grey-5">{{ formatTime(props.value) }}</div>
                </q-td>
              </template>
            </q-table>
          </div>
        </div>
      </q-page>
    </q-page-container>
  </q-layout>
</template>

<script setup>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import axios from 'axios'
import { useQuasar } from 'quasar'

const $q = useQuasar()
const incidents = ref([])
const jiraError = ref(false)
const loading = ref(false)
const filter = ref('')
const API_BASE = 'http://localhost:8000'

const columns = [
  { name: 'key', label: 'TICKET ID', field: 'key', align: 'left', sortable: true },
  { name: 'summary', label: 'SUMMARY', field: 'summary', align: 'left', style: 'max-width: 250px', classes: 'ellipsis' },
  { name: 'reporter_email', label: 'USER', field: 'reporter_email', align: 'left' },
  { name: 'priority', label: 'PRIORITY', field: 'priority', align: 'left' },
  { name: 'type', label: 'AI INTENT', field: 'type', align: 'center' },
  { name: 'confidence', label: 'CONFIDENCE', field: 'confidence', align: 'left', style: 'width: 100px' },
  { name: 'action', label: 'ACTION', field: 'action', align: 'left', classes: 'ellipsis', style: 'max-width: 200px' },
  { name: 'status', label: 'STATUS', field: 'status', align: 'center' },
  { name: 'actions', label: 'CONTROLS', field: 'actions', align: 'center' },
  { name: 'timestamp', label: 'TIME', field: 'timestamp', align: 'right', sortable: true }
]

// Stats
const resolvedCount = computed(() => incidents.value.filter(i => i.status === 'Resolved').length)
const escalatedCount = computed(() => incidents.value.filter(i => i.status.includes('Escalated')).length)
const processingCount = computed(() => incidents.value.filter(i => !['Resolved', 'Escalated'].some(s => i.status.includes(s))).length)

const summaryCards = computed(() => [
  { title: 'Total Backlog', value: incidents.value.length, icon: 'view_list', class: 'card-neutral' },
  { title: 'Auto Resolved', value: resolvedCount.value, icon: 'auto_mode', class: 'card-success' },
  { title: 'Need Attention', value: escalatedCount.value, icon: 'priority_high', class: 'card-warn' },
  { title: 'Active Bots', value: processingCount.value, icon: 'smart_toy', class: 'card-info' }
])

const fetchIncidents = async () => {
  loading.value = true
  try {
    const response = await axios.get(`${API_BASE}/incidents`)
    incidents.value = response.data.incidents
    jiraError.value = response.data.jira_error
  } catch (error) {
    jiraError.value = true
  } finally {
    loading.value = false
  }
}

const analyzeAndFix = async (key) => {
  try {
    await axios.post(`${API_BASE}/analyze-fix/${key}`)
    $q.notify({ message: 'Auto-fix started', color: 'accent', dense: true })
    fetchIncidents()
  } catch (e) { $q.notify({ message: 'Error', color: 'negative', dense: true }) }
}

const retryIncident = async (key) => {
  await axios.post(`${API_BASE}/retry/${key}`)
  $q.notify({ message: 'Retrying flow', color: 'warning', dense: true })
  fetchIncidents()
}

const escalateIncident = async (key) => {
  await axios.post(`${API_BASE}/escalate/${key}`)
  $q.notify({ message: 'Escalated to human', color: 'negative', dense: true })
  fetchIncidents()
}

const deleteIncident = async (key) => {
  try {
    await axios.delete(`${API_BASE}/incidents/${key}`)
    $q.notify({ message: 'Incident removed', color: 'dark', icon: 'delete', dense: true })
    fetchIncidents()
  } catch (e) {
    $q.notify({ message: 'Delete failed', color: 'negative', dense: true })
  }
}

const getStatusClass = (status) => {
  if (status === 'Resolved') return 'chip-success'
  if (status.includes('Escalated') || status.includes('Error')) return 'chip-error'
  if (['Executing', 'Validating', 'Detecting Intent'].includes(status)) return 'chip-active'
  return 'chip-neutral'
}

const isProcessing = (status) => ['Executing', 'Validating', 'Detecting Intent'].includes(status)

const getPriorityColor = (p) => {
  if (p === 'High' || p === 'Highest') return 'bg-negative'
  if (p === 'Medium') return 'bg-warning'
  return 'bg-info'
}

const getConfidenceValue = (c) => {
  if (typeof c !== 'string') return 0
  const l = c.toLowerCase()
  if (l.includes('high')) return 1.0
  if (l.includes('medium')) return 0.6
  return 0.2
}

const getConfidenceColor = (c) => {
  if (typeof c !== 'string') return 'grey'
  const l = c.toLowerCase()
  if (l.includes('high')) return 'positive'
  if (l.includes('medium')) return 'warning'
  return 'negative'
}

const formatTime = (t) => t ? new Date(t).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }) : '--:--'

let pollInterval
onMounted(() => {
  fetchIncidents()
  pollInterval = setInterval(fetchIncidents, 5000)
})
onUnmounted(() => clearInterval(pollInterval))
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Outfit:wght@300;400;600;800&display=swap');

:root {
  --bg-color: #0c111d;
  --panel-color: rgba(255, 255, 255, 0.05);
  --accent-color: #7f56d9;
}

body {
  font-family: 'Outfit', sans-serif;
  background-color: var(--bg-color);
  color: white;
  margin: 0;
  overflow-x: hidden;
}

.dashboard-root {
  background: radial-gradient(circle at top right, #1d1e4e 0%, #0c111d 40%);
  min-height: 100vh;
}

.header-glass {
  background: rgba(12, 17, 29, 0.8);
  backdrop-filter: blur(12px);
  border-bottom: 1px solid rgba(255, 255, 255, 0.08);
}

.bg-dark-layer {
  background: rgba(255, 255, 255, 0.05);
  border: 1px solid rgba(255, 255, 255, 0.08);
}

.status-dot { width: 6px; height: 6px; border-radius: 50%; box-shadow: 0 0 8px currentColor; }
.text-accent { color: var(--accent-color); }
.font-size-10 { font-size: 10px; }

.stat-card {
  background: var(--panel-color);
  border: 1px solid rgba(255, 255, 255, 0.1);
  border-radius: 12px;
  transition: transform 0.2s;
}

.card-bottom-accent { height: 3px; width: 100%; opacity: 0.5; }
.card-success .card-bottom-accent { background: #10b981; }
.card-warn .card-bottom-accent { background: #f59e0b; }
.card-info .card-bottom-accent { background: #3b82f6; }
.card-neutral .card-bottom-accent { background: var(--accent-color); }

.incident-table {
  background: rgba(255, 255, 255, 0.02) !important;
  border-radius: 16px;
  border: 1px solid rgba(255, 255, 255, 0.08) !important;
}

.incident-table thead tr th {
  color: #94a3b8;
  font-weight: 700;
  font-size: 10px;
  letter-spacing: 0.05em;
}

.search-input { min-width: 240px; color: #ffffff; }
@media (max-width: 600px) {
  .search-input { min-width: 100%; }
  .logo-container { margin-right: 8px; }
}

.status-chip { font-weight: 700; text-transform: uppercase; }
.chip-success { background: #059669; }
.chip-error { background: #dc2626; }
.chip-active { background: #7c3aed; }
.chip-neutral { background: rgba(255, 255, 255, 0.1); }

.pulse-blue { animation: pulse-animation 2s infinite; color: var(--accent-color); }
@keyframes pulse-animation {
  0% { opacity: 1; }
  50% { opacity: 0.5; }
  100% { opacity: 1; }
}

.p-dot { width: 8px; height: 8px; border-radius: 50%; }
.text-tiny { font-size: 8px; }

.banner-error { background: #991b1b; }
</style>
