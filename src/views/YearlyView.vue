<template>
  <div class="page">
    <div class="page-head">
      <div>
        <h2>年度总览</h2>
        <p class="page-sub">{{ year }} 年收支走势与储蓄目标完成情况</p>
      </div>
      <div class="year-switch">
        <button class="btn year-btn" :disabled="!canPrev" @click="shiftYear(-1)" aria-label="上一年">‹</button>
        <select v-model.number="year" class="year-select" aria-label="选择年份">
          <option v-for="y in years" :key="y" :value="y">{{ y }} 年</option>
        </select>
        <button class="btn year-btn" :disabled="!canNext" @click="shiftYear(1)" aria-label="下一年">›</button>
      </div>
    </div>

    <div class="kpis">
      <div class="card kpi">
        <span class="kpi-label">年度总收入</span>
        <b class="kpi-value income">¥{{ money(overview.income) }}</b>
      </div>
      <div class="card kpi">
        <span class="kpi-label">年度总支出</span>
        <b class="kpi-value expense">¥{{ money(overview.expense) }}</b>
      </div>
      <div class="card kpi">
        <span class="kpi-label">年度结余</span>
        <b class="kpi-value" :class="{ neg: overview.balance < 0 }">¥{{ money(overview.balance) }}</b>
      </div>
      <div class="card kpi">
        <span class="kpi-label">年度储蓄率</span>
        <b class="kpi-value accent">{{ savingsRate }}%</b>
      </div>
    </div>

    <div class="card chart-card">
      <h3 class="card-title">{{ year }} 年各月收支对比</h3>
      <div class="chart-scroll">
        <BarChart :items="trendData" :show-value="false" />
      </div>
      <div class="chart-legend">
        <span class="legend-dot income-dot"></span>收入
        <span class="legend-dot expense-dot"></span>支出
      </div>
    </div>

    <div class="card table-card">
      <h3 class="card-title">各月收支明细</h3>
      <div class="table-scroll">
        <table class="year-table">
          <thead>
            <tr>
              <th>月份</th>
              <th>收入</th>
              <th>支出</th>
              <th>结余</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="m in overview.monthly" :key="m.month">
              <td>{{ monthLabel(m.month) }}</td>
              <td class="num income">¥{{ money(m.income) }}</td>
              <td class="num expense">¥{{ money(m.expense) }}</td>
              <td class="num" :class="{ neg: m.balance < 0 }">¥{{ money(m.balance) }}</td>
            </tr>
          </tbody>
          <tfoot>
            <tr>
              <td>全年合计</td>
              <td class="num income">¥{{ money(overview.income) }}</td>
              <td class="num expense">¥{{ money(overview.expense) }}</td>
              <td class="num" :class="{ neg: overview.balance < 0 }">¥{{ money(overview.balance) }}</td>
            </tr>
          </tfoot>
        </table>
      </div>
    </div>

    <div class="card goals-card">
      <h3 class="card-title">储蓄目标完成情况</h3>
      <div v-for="g in goalRows" :key="g.id" class="goal-row">
        <div class="goal-row-head">
          <span class="goal-name">
            {{ g.name }}
            <em v-if="g.done" class="tag ok">已达成</em>
            <em v-else-if="g.overdue" class="tag danger">已逾期</em>
            <em v-else-if="g.dueThisYear" class="tag warn">本年到期</em>
          </span>
          <span class="goal-nums"><b>{{ money(g.savedAmount) }}</b> / {{ money(g.targetAmount) }}</span>
        </div>
        <div class="bar-track">
          <div class="bar" :class="g.done ? 'ok' : 'doing'" :style="{ width: Math.min(100, g.percent) + '%' }"></div>
        </div>
        <div class="goal-row-foot">
          <span>已完成 {{ g.percent }}%</span>
          <span>目标日期 {{ g.targetDate }}</span>
        </div>
      </div>
      <p class="empty" v-if="goalRows.length === 0">还没有储蓄目标，去「储蓄目标」页面新建一个吧。</p>
    </div>

    <div class="card empty" v-if="overview.income === 0 && overview.expense === 0">
      <p>{{ year }} 年还没有收支记录，切换年份或去「记账」页面记一笔吧。</p>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useStore, controllersApi } from '../data/store.js'
import { money, monthLabel, todayStr } from '../core/utils.js'
import BarChart from '../components/BarChart.vue'

const store = useStore()
const { report } = controllersApi

const years = computed(() => report.availableYears())
const year = ref(new Date().getFullYear())

const canPrev = computed(() => year.value > Math.min(...years.value))
const canNext = computed(() => year.value < Math.max(...years.value))
const shiftYear = (delta) => {
  const next = year.value + delta
  if (next < Math.min(...years.value) || next > Math.max(...years.value)) return
  year.value = next
}

const overview = computed(() => report.yearlyOverview(year.value))
const savingsRate = computed(() => report.savingsRate(overview.value.income, overview.value.expense))

const trendData = computed(() =>
  overview.value.monthly.flatMap((m) => [
    { label: `${Number(m.month.slice(5))}月`, value: m.income },
    { label: '', value: m.expense }
  ])
)

const goalRows = computed(() => {
  const today = todayStr()
  return store.goals
    .map((g) => {
      const percent = g.targetAmount > 0 ? Math.round((g.savedAmount / g.targetAmount) * 100) : 0
      return {
        ...g,
        percent,
        done: percent >= 100,
        overdue: percent < 100 && Boolean(g.targetDate) && g.targetDate < today,
        dueThisYear: Number(String(g.targetDate || '').slice(0, 4)) === year.value
      }
    })
    .sort((a, b) => Number(b.dueThisYear) - Number(a.dueThisYear) || a.percent - b.percent)
})
</script>

<style scoped>
.year-switch {
  display: flex;
  align-items: center;
  gap: 8px;
}
.year-btn {
  padding: 8px 14px;
  font-size: 16px;
  line-height: 1;
}
.year-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}
.year-select {
  padding: 9px 12px;
  border-radius: 10px;
  border: 1px solid var(--border-color);
  background: var(--card-bg);
  color: var(--text-primary);
  font-size: 14px;
  font-weight: 700;
  cursor: pointer;
}
.kpis {
  display: grid;
  grid-template-columns: repeat(4, 1fr);
  gap: 12px;
  margin-bottom: 16px;
}
@media (max-width: 720px) {
  .kpis { grid-template-columns: repeat(2, 1fr); }
}
.kpi {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.kpi-label {
  font-size: 12px;
  color: var(--text-secondary);
}
.kpi-value {
  font-size: 24px;
  font-weight: 800;
}
.kpi-value.income { color: var(--income); }
.kpi-value.expense { color: var(--expense); }
.kpi-value.neg { color: var(--expense); }
.kpi-value.accent { color: var(--accent); }
.card-title {
  margin: 0 0 12px;
  font-size: 15px;
}
.chart-card {
  margin-bottom: 16px;
}
.chart-scroll {
  overflow-x: auto;
}
.chart-scroll > :deep(.chart-wrap) {
  min-width: 720px;
}
.chart-legend {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: var(--text-secondary);
  margin-top: 6px;
}
.legend-dot {
  width: 10px;
  height: 10px;
  border-radius: 3px;
  display: inline-block;
}
.income-dot { background: #4f8df9; }
.expense-dot { background: #f9a54f; }
.table-card {
  margin-bottom: 16px;
}
.table-scroll {
  overflow-x: auto;
}
.year-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
  min-width: 480px;
}
.year-table th,
.year-table td {
  padding: 9px 12px;
  text-align: left;
  border-bottom: 1px solid var(--border-color);
}
.year-table th {
  color: var(--text-secondary);
  font-size: 12px;
  font-weight: 600;
}
.year-table .num {
  text-align: right;
  font-variant-numeric: tabular-nums;
  font-weight: 600;
}
.year-table th:nth-child(n + 2) {
  text-align: right;
}
.year-table .income { color: var(--income); }
.year-table .expense { color: var(--expense); }
.year-table .neg { color: var(--expense); }
.year-table tfoot td {
  border-bottom: none;
  border-top: 2px solid var(--border-color);
  font-weight: 800;
}
.goals-card {
  margin-bottom: 16px;
}
.goal-row {
  padding: 10px 0;
  border-bottom: 1px solid var(--border-color);
}
.goal-row:last-of-type {
  border-bottom: none;
}
.goal-row-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
  margin-bottom: 8px;
}
.goal-name {
  font-weight: 700;
  font-size: 14px;
}
.tag {
  font-style: normal;
  font-size: 11px;
  font-weight: 700;
  padding: 2px 8px;
  border-radius: 999px;
  margin-left: 6px;
  vertical-align: 1px;
}
.tag.ok { background: rgba(87, 199, 133, 0.15); color: var(--income); }
.tag.warn { background: rgba(240, 201, 87, 0.18); color: #b8860b; }
.tag.danger { background: rgba(244, 91, 105, 0.15); color: var(--expense); }
.goal-nums {
  font-size: 13px;
  color: var(--text-secondary);
}
.goal-nums b {
  color: var(--text-primary);
}
.bar-track {
  height: 10px;
  background: var(--bg-elevated);
  border-radius: 999px;
  overflow: hidden;
}
.bar {
  height: 100%;
  border-radius: 999px;
  transition: width 0.3s ease;
}
.bar.ok { background: linear-gradient(90deg, #57c785, #3aa66f); }
.bar.doing { background: linear-gradient(90deg, #4f8df9, #2f6fe4); }
.goal-row-foot {
  display: flex;
  justify-content: space-between;
  margin-top: 6px;
  font-size: 12px;
  color: var(--text-secondary);
}
</style>
