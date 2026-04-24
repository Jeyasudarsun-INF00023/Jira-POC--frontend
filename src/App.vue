<template>
  <q-layout view="lHh Lpr lFf" class="app-root">

    <!-- Minimal Header -->
    <q-header class="app-header">
      <q-toolbar class="header-inner">
        <div class="brand row items-center no-wrap">
          <div class="brand-icon-wrap q-mr-sm">
            <q-icon name="hub" size="18px" class="brand-icon" />
          </div>
          <div>
            <div class="brand-name" style="color:white;">Service Pilot</div>
            <div class="brand-sub" style="color:white;">Jira Incident Automation</div>
          </div>
        </div>

        <q-space />

        <div class="row items-center q-gutter-sm">
          <div class="status-pill row items-center no-wrap">
            <span :class="['status-dot', jiraError ? 'dot-error' : 'dot-ok']"></span>
            <span class="status-label">{{ jiraError ? 'Offline' : 'Live' }}</span>
          </div>
          <q-btn flat round dense icon="refresh" class="icon-btn" @click="fetchIncidents" :loading="loading">
            <q-tooltip>Refresh</q-tooltip>
          </q-btn>
        </div>
      </q-toolbar>
    </q-header>

    <q-page-container>
      <q-page class="page-body">

        <!-- Error Banner -->
        <div v-if="jiraError" class="error-strip row items-center q-mb-md">
          <q-icon name="wifi_off" size="14px" class="q-mr-xs" />
          <span>Jira connection failed. Check your API credentials.</span>
          <q-space />
          <span class="retry-link" @click="fetchIncidents">Retry</span>
        </div>

        <!-- Page Title Row -->
        <div class="row items-end justify-between q-mb-lg">
          <div>
            <div class="page-title">Incident Backlog</div>
            <div class="page-sub">AI-powered incident routing and response</div>
          </div>
          <div class="live-badge row items-center no-wrap">
            <span class="live-dot"></span>
            <span>Auto-refreshing</span>
          </div>
        </div>

        <div class="row q-col-gutter-md q-mb-xl">
          <div class="col-6 col-md-3" v-for="card in summaryCards" :key="card.title">
            <div class="kpi-card">
              <div class="row items-center justify-between no-wrap">
                <div>
                  <div class="kpi-label">{{ card.title }}</div>
                  <div class="kpi-value" :style="{ color: card.color }">{{ card.value }}</div>
                </div>
                <div class="kpi-icon-wrap" :style="{ background: card.color + '15' }">
                  <q-icon :name="card.icon" size="24px" :style="{ color: card.color }" />
                </div>
              </div>
              <div class="kpi-bar" :style="{ background: card.color }"></div>
            </div>
          </div>
        </div>

        <!-- Table -->
        <div class="table-wrap">
          <q-table
            :rows="incidents"
            :columns="columns"
            row-key="key"
            flat
            :pagination="{ rowsPerPage: 12 }"
            class="clean-table"
            no-data-label="No incidents found."
            dense
            :grid="$q.screen.lt.md"
          >
            <template v-slot:top>
              <div class="table-header row full-width items-center">
                <span class="table-title">All Incidents</span>
              </div>
            </template>

            <!-- Key -->
            <template v-slot:body-cell-key="props">
              <q-td :props="props">
                <span class="ticket-id">{{ props.value }}</span>
              </q-td>
            </template>

            <!-- Summary -->
            <template v-slot:body-cell-summary="props">
              <q-td :props="props">
                <div class="ellipsis summary-text">
                  {{ props.value }}
                  <q-tooltip v-if="props.row.description" max-width="280px" class="mini-tooltip">
                    {{ props.row.description }}
                  </q-tooltip>
                </div>
              </q-td>
            </template>

            <!-- Status -->
            <template v-slot:body-cell-status="props">
              <q-td :props="props">
                <span :class="['status-tag', getStatusClass(props.value)]">
                  {{ props.value }}
                </span>
              </q-td>
            </template>

            <!-- Priority -->
            <template v-slot:body-cell-priority="props">
              <q-td :props="props">
                <div class="row items-center no-wrap">
                  <span :class="['pri-dot', getPriorityClass(props.value)]"></span>
                  <span class="pri-label">{{ props.value }}</span>
                </div>
              </q-td>
            </template>

            <!-- Confidence -->
            <template v-slot:body-cell-confidence="props">
              <q-td :props="props">
                <div class="conf-wrap">
                  <q-linear-progress
                    :value="getConfidenceValue(props.value)"
                    :color="getConfidenceColor(props.value)"
                    size="3px"
                    rounded
                  />
                  <span class="conf-label">{{ props.value || 'N/A' }}</span>
                </div>
              </q-td>
            </template>

            <!-- Labels -->
            <template v-slot:body-cell-labels="props">
              <q-td :props="props">
                <span v-if="props.value" class="label-tag">{{ props.value }}</span>
                <span v-else class="muted">—</span>
              </q-td>
            </template>

            <!-- Due Date -->
            <template v-slot:body-cell-duedate="props">
              <q-td :props="props">
                <span :class="isOverdue(props.value) ? 'overdue' : ''">
                  {{ props.value === 'No Due Date' ? '—' : props.value }}
                </span>
              </q-td>
            </template>

            <!-- Actions -->
            <template v-slot:body-cell-actions="props">
              <q-td :props="props">
                <div class="row items-center q-gutter-xs no-wrap justify-center">
                  <q-btn flat round dense icon="refresh" size="xs" class="act-btn act-warn"
                    @click="retryIncident(props.row.key)" :disable="props.row.status === 'Resolved'">
                    <q-tooltip>Retry</q-tooltip>
                  </q-btn>
                  <q-btn flat round dense icon="person_search" size="xs" class="act-btn act-err"
                    @click="escalateIncident(props.row.key)" :disable="props.row.status === 'Resolved'">
                    <q-tooltip>Escalate</q-tooltip>
                  </q-btn>
                  <q-btn flat round dense icon="check_circle" size="xs" class="act-btn act-ok"
                    @click="resolveIncident(props.row.key)" :disable="props.row.status === 'Resolved'">
                    <q-tooltip>Resolve</q-tooltip>
                  </q-btn>
                  <q-btn flat round dense icon="delete_outline" size="xs" class="act-btn act-del"
                    @click="deleteIncident(props.row.key)">
                    <q-tooltip>Delete</q-tooltip>
                  </q-btn>
                </div>
              </q-td>
            </template>

            <!-- Time -->
            <template v-slot:body-cell-timestamp="props">
              <q-td :props="props">
                <span class="muted">{{ formatTime(props.value) }}</span>
              </q-td>
            </template>
          </q-table>
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
const API_BASE = import.meta.env.VITE_API_BASE_URL || import.meta.env.VITE_API_LOCAL_URL || (window.location.origin.includes('localhost') ? 'http://localhost:8000' : window.location.origin)

const columns = [
  { name: 'key',        label: 'Ticket',    field: 'key',          align: 'left',   sortable: true },
  { name: 'project',    label: 'Space',     field: 'project',      align: 'left' },
  { name: 'issuetype',  label: 'Type',      field: 'issuetype',    align: 'left' },
  { name: 'summary',    label: 'Summary',   field: 'summary',      align: 'left',   style: 'max-width:220px', classes: 'ellipsis' },
  { name: 'assignee',   label: 'Assignee',  field: 'assignee',     align: 'left' },
  { name: 'priority',   label: 'Priority',  field: 'priority',     align: 'left' },
  { name: 'status',     label: 'Status',    field: 'status',       align: 'center' },
  { name: 'duedate',    label: 'Due',       field: 'duedate',      align: 'center' },
  { name: 'team',       label: 'Team',      field: 'team',         align: 'center' },
  { name: 'labels',     label: 'Labels',    field: 'labels',       align: 'left',   style: 'max-width:120px', classes: 'ellipsis' },
  { name: 'type',       label: 'AI Intent', field: 'type',         align: 'center' },
  { name: 'action',     label: 'Action',    field: 'action',       align: 'left',   classes: 'ellipsis', style: 'max-width:180px' },
  { name: 'confidence', label: 'Confidence',field: 'confidence',   align: 'center' },
  { name: 'actions',    label: 'Controls',  field: 'actions',      align: 'center' },
  { name: 'timestamp',  label: 'Time',      field: 'timestamp',    align: 'right',  sortable: true }
]

const resolvedCount  = computed(() => incidents.value.filter(i => i.status === 'Resolved').length)
const escalatedCount = computed(() => incidents.value.filter(i => i.status.includes('Escalated')).length)
const processingCount = computed(() => incidents.value.filter(i => !['Resolved','Escalated'].some(s => i.status.includes(s))).length)

const summaryCards = computed(() => [
  { title: 'Total',      value: incidents.value.length, color: '#08203e', icon: 'list_alt' },
  { title: 'Resolved',   value: resolvedCount.value,    color: '#10b981', icon: 'check_circle' },
  { title: 'Escalated',  value: escalatedCount.value,   color: '#f59e0b', icon: 'report_problem' },
  { title: 'Active',     value: processingCount.value,  color: '#3b82f6', icon: 'smart_toy' },
])

const fetchIncidents = async () => {
  loading.value = true
  try {
    console.log(`Fetching from: ${API_BASE}/incidents`)
    const response = await axios.get(`${API_BASE}/incidents`, {
      headers: { 'ngrok-skip-browser-warning': 'true' }
    })
    incidents.value = response.data.incidents || []
    jiraError.value = response.data.jira_error || false
    console.log(`Received ${incidents.value.length} incidents`)
  } catch (err) { 
    console.error('Fetch failed:', err)
    jiraError.value = true 
  }
  finally { loading.value = false }
}

const retryIncident = async (key) => {
  await axios.post(`${API_BASE}/retry/${key}`)
  $q.notify({ message: 'Retrying flow…', color: 'warning', dense: true, position: 'bottom-right' })
  fetchIncidents()
}

const escalateIncident = async (key) => {
  await axios.post(`${API_BASE}/escalate/${key}`)
  $q.notify({ message: 'Escalated to human', color: 'negative', dense: true, position: 'bottom-right' })
  fetchIncidents()
}

const resolveIncident = async (key) => {
  try {
    const res = await axios.post(`${API_BASE}/resolve/${key}`)
    if (res.data.status === 'resolved') {
      $q.notify({ message: `${key} resolved`, color: 'positive', icon: 'check_circle', dense: true, position: 'bottom-right' })
      fetchIncidents()
    } else throw new Error(res.data.message)
  } catch (e) {
    $q.notify({ message: `Failed: ${e.message}`, color: 'negative', dense: true, position: 'bottom-right' })
  }
}

const deleteIncident = async (key) => {
  try {
    await axios.delete(`${API_BASE}/incidents/${key}`)
    $q.notify({ message: 'Removed', color: 'grey-7', dense: true, position: 'bottom-right' })
    fetchIncidents()
  } catch { $q.notify({ message: 'Delete failed', color: 'negative', dense: true }) }
}

const getStatusClass = (s) => {
  if (!s) return 'tag-neutral'
  const u = s.toUpperCase()
  if (u === 'RESOLVED' || u === 'DONE') return 'tag-green'
  if (u.includes('PROGRESS') || u.includes('PROCESSING') || u.includes('EXECUTING')) return 'tag-blue'
  if (u.includes('ESCALATED') || u.includes('ERROR') || u.includes('FAILED')) return 'tag-red'
  if (u.includes('AWAITING') || u.includes('WAITING')) return 'tag-amber'
  return 'tag-neutral'
}

const getPriorityClass = (p) => {
  if (p === 'High' || p === 'Highest') return 'pri-red'
  if (p === 'Medium') return 'pri-amber'
  return 'pri-blue'
}

const getConfidenceValue = (c) => {
  if (!c) return 0
  const l = c.toLowerCase()
  if (l.includes('high')) return 1.0
  if (l.includes('medium')) return 0.6
  return 0.2
}

const getConfidenceColor = (c) => {
  if (!c) return 'grey'
  const l = c.toLowerCase()
  if (l.includes('high')) return 'positive'
  if (l.includes('medium')) return 'warning'
  return 'negative'
}

const formatTime = (t) => t ? new Date(t).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }) : '—'
const isOverdue = (date) => date && date !== 'No Due Date' && new Date(date) < new Date()

let pollInterval
onMounted(() => { fetchIncidents(); pollInterval = setInterval(fetchIncidents, 5000) })
onUnmounted(() => clearInterval(pollInterval))
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=Poppins:wght@300;400;500;600;700&display=swap');

* { box-sizing: border-box; }

body {
  font-family: 'Poppins', sans-serif;
  background: #f5f6fa;
  color: #1a1d27;
  margin: 0;
}

/* ── Layout ─────────────────────────────── */
.app-root { background: #f5f6fa; min-height: 100vh; }

.app-header {
  background: #ffffff;
  border-bottom: 1px solid #e8eaef;
  box-shadow: none;
  color: #1a1d27;
}

.header-inner {
  background-color: #1a1a1a;
  max-width: 1400px;
  margin: 0 auto;
  padding: 0 24px;
  min-height: 52px;
}

/* ── Brand ──────────────────────────────── */
.brand-icon-wrap {
  width: 30px; height: 30px;
  background: #6366f1;
  border-radius: 8px;
  display: flex; align-items: center; justify-content: center;
}
.brand-icon { color: #fff; }
.brand-name { font-size: 14px; font-weight: 600; color: #1a1d27; line-height: 1.2; }
.brand-sub  { font-size: 10px; color: #9da3b4; }

/* ── Status Pill ────────────────────────── */
.status-pill {
  padding: 4px 10px;
  background: #f5f6fa;
  border: 1px solid #e8eaef;
  border-radius: 20px;
  gap: 6px;
}
.status-dot { width: 6px; height: 6px; border-radius: 50%; }
.dot-ok    { background: #10b981; }
.dot-error { background: #ef4444; }
.status-label { font-size: 11px; font-weight: 500; color: #6b7280; }

.icon-btn { color: #9da3b4 !important; }
.icon-btn:hover { color: #6366f1 !important; }

/* ── Page Body ──────────────────────────── */
.page-body {
  max-width: 1400px;
  margin: 0 auto;
  padding: 28px 24px;
}

/* ── Error ──────────────────────────────── */
.error-strip {
  background: #fef2f2;
  border: 1px solid #fecaca;
  border-radius: 8px;
  padding: 10px 14px;
  font-size: 12px;
  color: #dc2626;
}
.retry-link { font-weight: 600; cursor: pointer; text-decoration: underline; }

/* ── Page Title ─────────────────────────── */
.page-title { font-size: 20px; font-weight: 700; color: #1a1d27; }
.page-sub   { font-size: 12px; color: #9da3b4; margin-top: 2px; }

.live-badge-wrap { display: flex; align-items: center; gap: 6px; }
.live-badge {
  background: #eef2ff !important;
  color: #6366f1 !important;
  font-size: 10px;
  font-weight: 700;
  border-radius: 4px;
  padding: 2px 6px;
}
.live-dot {
  width: 6px; height: 6px; border-radius: 50%;
  background: #6366f1;
  display: inline-block;
  margin-right: 4px;
  animation: blink 1.4s infinite;
}
@keyframes blink {
  0%, 100% { opacity: 1; }
  50%       { opacity: 0.3; }
}
.live-badge {
  display: flex; align-items: center;
  font-size: 11px; color: #6366f1; font-weight: 500;
}

/* ── KPI Cards ──────────────────────────── */
.kpi-card {
  background: #ffffff;
  border: 1px solid #e8eaef;
  border-radius: 12px;
  padding: 18px 20px 14px;
  position: relative;
  overflow: hidden;
  transition: box-shadow 0.2s;
}
.kpi-card:hover { box-shadow: 0 4px 16px rgba(0,0,0,0.07); }
.kpi-label { font-size: 11px; font-weight: 500; color: #9da3b4; text-transform: uppercase; letter-spacing: 0.05em; margin-bottom: 6px; }
.kpi-value { font-size: 32px; font-weight: 700; line-height: 1; font-variant-numeric: tabular-nums; }
.kpi-icon-wrap {
  width: 44px;
  height: 44px;
  border-radius: 10px;
  display: flex;
  align-items: center;
  justify-content: center;
  flex-shrink: 0;
}
.kpi-bar   { position: absolute; bottom: 0; left: 0; right: 0; height: 3px; opacity: 0.6; border-radius: 0 0 12px 12px; }

/* ── Table ──────────────────────────────── */
.table-wrap {
  background: #ffffff;
  border: 1px solid #e8eaef;
  border-radius: 12px;
  overflow: hidden;
}

.clean-table { background: transparent !important; }

.clean-table thead tr th {
  background: #f9fafb !important;
  color: #9da3b4 !important;
  font-size: 10px !important;
  font-weight: 600 !important;
  letter-spacing: 0.06em !important;
  text-transform: uppercase !important;
  border-bottom: 1px solid #e8eaef !important;
}

.clean-table tbody tr { border-bottom: 1px solid #f3f4f6; }
.clean-table tbody tr:hover { background: #fafbff !important; }
.clean-table tbody tr:last-child { border-bottom: none; }
.clean-table tbody td { color: #374151; font-size: 12px; }

.table-header { padding: 14px 16px 12px; border-bottom: 1px solid #f3f4f6; }
.table-title  { font-size: 13px; font-weight: 600; color: #1a1d27; }

/* ── Ticket ID ──────────────────────────── */
.ticket-id { font-weight: 600; color: #6366f1; font-size: 12px; }

/* ── Summary ────────────────────────────── */
.summary-text { font-size: 12px; color: #374151; }

/* ── Status Tags ────────────────────────── */
.status-tag {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 20px;
  font-size: 10px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
  white-space: nowrap;
}
.tag-green   { background: #d1fae5; color: #065f46; }
.tag-blue    { background: #dbeafe; color: #1e40af; }
.tag-red     { background: #fee2e2; color: #991b1b; }
.tag-amber   { background: #fef3c7; color: #92400e; }
.tag-neutral { background: #f3f4f6; color: #6b7280; }

/* ── Priority ───────────────────────────── */
.pri-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; margin-right: 6px; }
.pri-red   { background: #ef4444; }
.pri-amber { background: #f59e0b; }
.pri-blue  { background: #3b82f6; }
.pri-label { font-size: 12px; color: #374151; }

/* ── Confidence ─────────────────────────── */
.conf-wrap { min-width: 70px; }
.conf-label { font-size: 9px; color: #9da3b4; text-transform: uppercase; margin-top: 2px; display: block; }

/* ── Label Tag ──────────────────────────── */
.label-tag {
  background: #f3f4f6;
  color: #6b7280;
  font-size: 10px;
  border-radius: 4px;
  padding: 2px 6px;
}

/* ── Misc ───────────────────────────────── */
.muted   { color: #c4c9d6; font-size: 12px; }
.overdue { color: #ef4444; font-weight: 500; }

/* ── Action Buttons ─────────────────────── */
.act-btn { border-radius: 6px !important; }
.act-warn { color: #d97706 !important; }
.act-warn:hover { background: #fef3c7 !important; }
.act-err  { color: #dc2626 !important; }
.act-err:hover  { background: #fee2e2 !important; }
.act-ok   { color: #059669 !important; }
.act-ok:hover   { background: #d1fae5 !important; }
.act-del  { color: #9da3b4 !important; }
.act-del:hover  { background: #f3f4f6 !important; }

/* ── Tooltip ────────────────────────────── */
.mini-tooltip { background: #1a1d27 !important; color: #e5e7eb !important; font-size: 11px !important; }
</style>
