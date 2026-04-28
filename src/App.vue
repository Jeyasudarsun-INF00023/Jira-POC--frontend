<template>
  <q-layout view="lHh Lpr lFf" class="app-root">

    <!-- Header -->
    <q-header class="app-header">
      <q-toolbar class="header-inner">
        <div class="brand row items-center no-wrap">
          <div class="brand-icon-wrap q-mr-sm">
            <q-icon name="hub" size="16px" class="brand-icon" />
          </div>
          <div>
            <div class="brand-name">Service Pilot</div>
            <div class="brand-sub">Jira Incident Automation</div>
          </div>
        </div>

        <q-space />

        <div class="row items-center q-gutter-sm">
          <div class="status-pill row items-center no-wrap">
            <span :class="['status-dot', jiraError ? 'dot-error' : 'dot-ok']"></span>
            <span class="status-label">{{ jiraError ? 'Offline' : 'Auto refreshing' }}</span>
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
        <div class="row items-end justify-between q-mb-xl">
          <div>
            <div class="page-title">Incident Backlog</div>
            <div class="page-sub">AI-powered incident routing and response</div>
          </div>
        </div>

        <!-- KPI Cards -->
        <div class="row q-col-gutter-md q-mb-xl">
          <div class="col-6 col-md-3" v-for="card in summaryCards" :key="card.title">
            <div class="kpi-card">
              <div class="kpi-top row items-start justify-between no-wrap">
                <div>
                  <div class="kpi-label">{{ card.title }}</div>
                  <div class="kpi-value" :style="{ color: card.color }">{{ card.value }}</div>
                </div>
                <div class="kpi-icon-wrap" :style="{ background: card.bg }">
                  <q-icon :name="card.icon" size="20px" :style="{ color: card.color }" />
                </div>
              </div>
              <div class="kpi-footer">
                <div class="kpi-bar-track">
                  <div class="kpi-bar-fill" :style="{ background: card.color, width: card.pct }"></div>
                </div>
              </div>
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
              <div class="table-header row full-width items-center justify-between">
                <div>
                  <span class="table-title">All Incidents</span>
                  <span class="table-count q-ml-sm">{{ incidents.length }} total</span>
                </div>
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
                <span :class="isOverdue(props.value) ? 'overdue' : 'muted'">
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
  { name: 'key',        label: 'Tkt',        field: 'key',          align: 'left',   sortable: true },
  { name: 'project',    label: 'Space',      field: 'project',      align: 'left' },
  { name: 'issuetype',  label: 'Type',       field: 'issuetype',    align: 'left' },
  { name: 'summary',    label: 'Summary',    field: 'summary',      align: 'left',   style: 'max-width:140px', classes: 'ellipsis' },
  { name: 'assignee',   label: 'User',       field: 'assignee',     align: 'left' },
  { name: 'priority',   label: 'Pri',        field: 'priority',     align: 'left' },
  { name: 'status',     label: 'Status',     field: 'status',       align: 'center' },
  { name: 'duedate',    label: 'Due',        field: 'duedate',      align: 'center' },
  { name: 'team',       label: 'Team',       field: 'team',         align: 'center' },
  { name: 'labels',     label: 'Labels',     field: 'labels',       align: 'left',   style: 'max-width:80px', classes: 'ellipsis' },
  { name: 'type',       label: 'AI Intent',  field: 'type',         align: 'center' },
  { name: 'action',     label: 'Action',     field: 'action',       align: 'left',   classes: 'ellipsis', style: 'max-width:120px' },
  { name: 'confidence', label: 'Conf.',      field: 'confidence',   align: 'center' },
  { name: 'actions',    label: 'Ctrl',       field: 'actions',      align: 'center' },
  { name: 'timestamp',  label: 'Time',       field: 'timestamp',    align: 'right',  sortable: true }
]

const resolvedCount   = computed(() => incidents.value.filter(i => i.status === 'Resolved').length)
const escalatedCount  = computed(() => incidents.value.filter(i => i.status.includes('Escalated')).length)
const processingCount = computed(() => incidents.value.filter(i => !['Resolved','Escalated'].some(s => i.status.includes(s))).length)

const summaryCards = computed(() => {
  const total = incidents.value.length || 1
  return [
    { title: 'Total',     value: incidents.value.length, color: '#4f46e5', bg: '#eef2ff', icon: 'list_alt',      pct: '100%' },
    { title: 'Resolved',  value: resolvedCount.value,    color: '#059669', bg: '#ecfdf5', icon: 'check_circle',  pct: (resolvedCount.value / total * 100) + '%' },
    { title: 'Escalated', value: escalatedCount.value,   color: '#d97706', bg: '#fffbeb', icon: 'report_problem',pct: (escalatedCount.value / total * 100) + '%' },
    { title: 'Active',    value: processingCount.value,  color: '#2563eb', bg: '#eff6ff', icon: 'smart_toy',     pct: (processingCount.value / total * 100) + '%' },
  ]
})

const fetchIncidents = async () => {
  loading.value = true
  try {
    const response = await axios.get(`${API_BASE}/incidents`, {
      headers: { 'ngrok-skip-browser-warning': 'true' }
    })
    incidents.value = response.data.incidents || []
    jiraError.value = response.data.jira_error || false
  } catch (err) {
    jiraError.value = true
  } finally { loading.value = false }
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
const isOverdue  = (date) => date && date !== 'No Due Date' && new Date(date) < new Date()

let pollInterval
onMounted(() => { fetchIncidents(); pollInterval = setInterval(fetchIncidents, 5000) })
onUnmounted(() => clearInterval(pollInterval))
</script>

<style>
@import url('https://fonts.googleapis.com/css2?family=DM+Sans:wght@300;400;500;600;700&family=DM+Mono:wght@400;500&display=swap');

:root {
  --bg-page:    #f7f8fc;
  --bg-card:    #ffffff;
  --bg-header:  #ffffff;
  --border:     #e4e7f0;
  --border-light: #f0f2f8;
  --text-primary: #141928;
  --text-secondary: #5c6380;
  --text-muted:   #a8afc8;
  --accent:       #4f46e5;
  --accent-soft:  #eef2ff;
  --shadow-card:  0 1px 3px rgba(20,25,60,0.06), 0 4px 16px rgba(20,25,60,0.04);
  --shadow-hover: 0 4px 24px rgba(79,70,229,0.10), 0 1px 4px rgba(20,25,60,0.08);
  --radius-card:  14px;
  --radius-pill:  100px;
}

* { box-sizing: border-box; }

body {
  font-family: 'DM Sans', sans-serif;
  background: var(--bg-page);
  color: var(--text-primary);
  margin: 0;
}

/* ── Background texture ─── */
.app-root {
  background: var(--bg-page);
  min-height: 100vh;
  background-image:
    radial-gradient(ellipse 80% 50% at 10% -10%, rgba(79,70,229,0.06) 0%, transparent 60%),
    radial-gradient(ellipse 60% 40% at 90% 110%, rgba(37,99,235,0.05) 0%, transparent 60%);
}

/* ── Header ──────────────────────── */
.app-header {
  background: var(--bg-header);
  border-bottom: 1px solid var(--border);
  box-shadow: 0 1px 0 var(--border-light);
}

.header-inner {
  max-width: 1440px;
  margin: 0 auto;
  padding: 0 32px;
  min-height: 56px;
}

@media (max-width: 600px) {
  .header-inner {
    padding: 0 16px;
  }
}

/* ── Brand ──────────────── */
.brand-icon-wrap {
  width: 32px; height: 32px;
  background: linear-gradient(135deg, #4f46e5 0%, #6366f1 100%);
  border-radius: 9px;
  display: flex; align-items: center; justify-content: center;
  box-shadow: 0 2px 8px rgba(79,70,229,0.28);
}
.brand-icon { color: #fff; }
.brand-name { font-size: 14px; font-weight: 700; color: var(--text-primary); line-height: 1.2; letter-spacing: -0.01em; }
.brand-sub  { font-size: 10px; color: var(--text-muted); font-weight: 400; letter-spacing: 0.01em; }

/* ── Status Pill ──────────── */
.status-pill {
  padding: 5px 12px;
  background: var(--bg-page);
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  gap: 6px;
}
.status-dot { width: 6px; height: 6px; border-radius: 50%; }
.dot-ok    { background: #10b981; box-shadow: 0 0 0 2px rgba(16,185,129,0.2); }
.dot-error { background: #ef4444; box-shadow: 0 0 0 2px rgba(239,68,68,0.2); }
.status-label { font-size: 11px; font-weight: 600; color: var(--text-secondary); letter-spacing: 0.02em; }

.icon-btn { color: var(--text-muted) !important; }
.icon-btn:hover { color: var(--accent) !important; background: var(--accent-soft) !important; }

/* ── Page Body ─────────────── */
.page-body {
  max-width: 100%;
  margin: 0 auto;
  padding: 24px 20px;
}

@media (max-width: 600px) {
  .page-body {
    padding: 24px 16px;
  }
}

/* ── Error Strip ─────────────── */
.error-strip {
  background: #fff5f5;
  border: 1px solid #fecdd3;
  border-radius: 10px;
  padding: 11px 16px;
  font-size: 12px;
  color: #be123c;
  font-weight: 500;
}
.retry-link { font-weight: 700; cursor: pointer; text-decoration: underline; text-underline-offset: 2px; }

/* ── Page Title ─────────────── */
.page-eyebrow {
  font-size: 10px;
  font-weight: 600;
  color: var(--accent);
  letter-spacing: 0.12em;
  text-transform: uppercase;
  margin-bottom: 4px;
}
.page-title { font-size: 24px; font-weight: 700; color: var(--text-primary); letter-spacing: -0.02em; line-height: 1.15; }

@media (max-width: 600px) {
  .page-title { font-size: 20px; }
}

.page-sub   { font-size: 13px; color: var(--text-secondary); margin-top: 3px; font-weight: 400; }

/* ── Live Badge ──────────────── */
.live-badge {
  display: flex; align-items: center;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: var(--radius-pill);
  padding: 5px 13px;
  font-size: 11px;
  color: #059669;
  font-weight: 600;
  gap: 6px;
  letter-spacing: 0.01em;
}
.live-dot {
  width: 6px; height: 6px; border-radius: 50%;
  background: #10b981;
  animation: pulse 2s infinite;
  flex-shrink: 0;
}
@keyframes pulse {
  0%, 100% { opacity: 1; box-shadow: 0 0 0 0 rgba(16,185,129,0.4); }
  50%       { opacity: 0.7; box-shadow: 0 0 0 4px rgba(16,185,129,0); }
}

/* ── KPI Cards ──────────────── */
.kpi-card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 10px;
  padding: 12px 16px 10px;
  position: relative;
  overflow: hidden;
  box-shadow: var(--shadow-card);
  transition: box-shadow 0.22s ease, transform 0.22s ease;
}
.kpi-card:hover {
  box-shadow: var(--shadow-hover);
  transform: translateY(-2px);
}

.kpi-top { margin-bottom: 16px; }

.kpi-label {
  font-size: 10px; font-weight: 600; color: var(--text-muted);
  text-transform: uppercase; letter-spacing: 0.1em; margin-bottom: 6px;
}
.kpi-value {
  font-size: 28px; font-weight: 700; line-height: 1;
  letter-spacing: -0.02em; font-variant-numeric: tabular-nums;
}

@media (max-width: 600px) {
  .kpi-value { font-size: 28px; }
}
.kpi-icon-wrap {
  width: 42px; height: 42px; border-radius: 11px;
  display: flex; align-items: center; justify-content: center;
  flex-shrink: 0;
}

.kpi-footer { margin-top: 4px; }
.kpi-bar-track {
  height: 3px; background: var(--border-light); border-radius: 99px; overflow: hidden;
}
.kpi-bar-fill {
  height: 100%; border-radius: 99px;
  transition: width 0.6s cubic-bezier(0.4, 0, 0.2, 1);
  opacity: 0.7;
}

/* ── Table Wrapper ──────────────── */
.table-wrap {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius-card);
  overflow: hidden;
  box-shadow: var(--shadow-card);
}

.clean-table { background: transparent !important; }

.clean-table thead tr th {
  background: #fafbff !important;
  color: var(--text-muted) !important;
  font-size: 9px !important;
  font-weight: 700 !important;
  letter-spacing: 0.05em !important;
  text-transform: uppercase !important;
  border-bottom: 1px solid var(--border) !important;
  padding: 6px 8px !important;
  font-family: 'DM Sans', sans-serif !important;
}

.clean-table tbody tr {
  border-bottom: 1px solid var(--border-light);
  transition: background 0.12s;
}
.clean-table tbody tr:hover { background: #fafbff !important; }
.clean-table tbody tr:last-child { border-bottom: none; }
.clean-table tbody td {
  color: var(--text-primary); font-size: 11px;
  padding: 6px 8px !important;
  font-family: 'DM Sans', sans-serif !important;
}

.table-header {
  padding: 16px 20px 14px;
  border-bottom: 1px solid var(--border-light);
}

@media (max-width: 600px) {
  .table-header {
    padding: 12px 16px;
    flex-direction: column;
    align-items: flex-start;
    gap: 8px;
  }
}

.table-title { font-size: 14px; font-weight: 700; color: var(--text-primary); letter-spacing: -0.01em; }
.table-count {
  font-size: 11px; font-weight: 500;
  color: var(--text-muted);
  background: var(--bg-page);
  border: 1px solid var(--border);
  border-radius: var(--radius-pill);
  padding: 2px 10px;
}

/* ── Ticket ID ─────────── */
.ticket-id {
  font-family: 'DM Mono', monospace;
  font-weight: 500;
  color: var(--accent);
  font-size: 11.5px;
  background: var(--accent-soft);
  padding: 2px 8px;
  border-radius: 5px;
  letter-spacing: 0.02em;
}

/* ── Summary ─────────── */
.summary-text { font-size: 12.5px; color: var(--text-primary); font-weight: 400; }

/* ── Status Tags ─────────── */
.status-tag {
  display: inline-block;
  padding: 3px 10px;
  border-radius: var(--radius-pill);
  font-size: 10px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  white-space: nowrap;
}
.tag-green   { background: #ecfdf5; color: #065f46; border: 1px solid #a7f3d0; }
.tag-blue    { background: #eff6ff; color: #1d4ed8; border: 1px solid #bfdbfe; }
.tag-red     { background: #fff1f2; color: #9f1239; border: 1px solid #fecdd3; }
.tag-amber   { background: #fffbeb; color: #92400e; border: 1px solid #fde68a; }
.tag-neutral { background: #f8fafc; color: #64748b; border: 1px solid #e2e8f0; }

/* ── Priority ─────────── */
.pri-dot { width: 7px; height: 7px; border-radius: 50%; flex-shrink: 0; margin-right: 6px; }
.pri-red   { background: #ef4444; box-shadow: 0 0 0 2px rgba(239,68,68,0.15); }
.pri-amber { background: #f59e0b; box-shadow: 0 0 0 2px rgba(245,158,11,0.15); }
.pri-blue  { background: #3b82f6; box-shadow: 0 0 0 2px rgba(59,130,246,0.15); }
.pri-label { font-size: 12px; color: var(--text-primary); font-weight: 500; }

/* ── Confidence ─────────── */
.conf-wrap { min-width: 72px; }
.conf-label {
  font-size: 9px; color: var(--text-muted); text-transform: uppercase;
  margin-top: 3px; display: block; letter-spacing: 0.06em; font-weight: 600;
}

/* ── Label Tag ─────────── */
.label-tag {
  background: #f8fafc;
  color: var(--text-secondary);
  border: 1px solid var(--border);
  font-size: 10px;
  border-radius: 5px;
  padding: 2px 8px;
  font-weight: 500;
}

/* ── Misc ─────────── */
.muted   { color: var(--text-muted); font-size: 12px; }
.overdue { color: #dc2626; font-weight: 600; font-size: 12px; }

/* ── Action Buttons ─────────── */
.act-btn { border-radius: 7px !important; transition: background 0.15s !important; }
.act-warn { color: #d97706 !important; }
.act-warn:hover { background: #fffbeb !important; }
.act-err  { color: #dc2626 !important; }
.act-err:hover  { background: #fff1f2 !important; }
.act-ok   { color: #059669 !important; }
.act-ok:hover   { background: #ecfdf5 !important; }
.act-del  { color: var(--text-muted) !important; }
.act-del:hover  { background: var(--bg-page) !important; }

/* ── Tooltip ─────────── */
.mini-tooltip {
  background: #1e293b !important;
  color: #e2e8f0 !important;
  font-size: 11px !important;
  font-family: 'DM Sans', sans-serif !important;
  border-radius: 6px !important;
  padding: 5px 10px !important;
}
</style>