<template>
  <div class="page">
    <div class="page-head">
      <div>
        <h2>年度总览</h2>
        <p class="page-sub">{{ year }} 年全年收支与储蓄目标一览</p>
      </div>
      <div class="year-switch">
        <button class="year-btn" :disabled="!canPrev" @click="changeYear(-1)" aria-label="上一年">‹</button>
        <select v-model.number="year" class="year-select">
          <option v-for="y in yearOptions" :key="y" :value="y">{{ y }} 年</option>
        </select>
        <button class="year-btn" :disabled="!canNext" @click="changeYear(1)" aria-label="下一年">›</button>
      </div>
    </div>

    <div class="kpis">
      <div class="card kpi">
        <span class="kpi-label">{{ year }} 年总收入</span>
        <b class="kpi-value income">¥{{ money(yearIncome) }}</b>
      </div>
      <div class="card kpi">
        <span class="kpi-label">{{ year }} 年总支出</span>
        <b class="kpi-value expense">¥{{ money(yearExpense) }}</b>
      </div>
      <div class="card kpi">
        <span class="kpi-label">年度结余</span>
        <b class="kpi-value" :class="{ neg: yearBalance < 0 }">¥{{ money(yearBalance) }}</b>
      </div>
      <div class="card kpi">
        <span class="kpi-label">年度储蓄率</span>
        <b class="kpi-value accent">{{ yearSavingsRate }}%</b>
      </div>
    </div>

    <div class="card chart-card">
      <div class="card-head">
        <h3 class="card-title">{{ year }} 年各月收支</h3>
        <div class="chart-legend">
          <span class="legend-dot income-dot"></span>收入
          <span class="legend-dot expense-dot"></span>支出
          <span class="legend-dot balance-dot"></span>结余
        </div>
      </div>
      <div class="chart-scroll">
        <BarChart :items="barItems" :show-value="false" />
      </div>
    </div>

    <div class="dual-grid">
      <div class="card">
        <h3 class="card-title">月度明细</h3>
        <div class="table-wrap">
          <table class="month-table">
            <thead>
              <tr>
                <th>月份</th>
                <th class="num">收入</th>
                <th class="num">支出</th>
                <th class="num">结余</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="m in monthlyRows" :key="m.month">
                <td>{{ Number(m.month.slice(5)) }}月</td>
                <td class="num income">{{ m.income ? '¥' + money(m.income) : '¥0.00' }}</td>
                <td class="num expense">{{ m.expense ? '¥' + money(m.expense) : '¥0.00' }}</td>
                <td class="num" :class="{ neg: m.balance < 0 }">¥{{ money(m.balance) }}</td>
              </tr>
            </tbody>
            <tfoot>
              <tr>
                <td>全年合计</td>
                <td class="num income">¥{{ money(yearIncome) }}</td>
                <td class="num expense">¥{{ money(yearExpense) }}</td>
                <td class="num" :class="{ neg: yearBalance < 0 }">¥{{ money(yearBalance) }}</td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>

      <div class="card">
        <div class="card-head">
          <h3 class="card-title">储蓄目标完成情况</h3>
          <span class="badge">{{ yearGoals.length ? `已完成 ${doneGoals.length}/${yearGoals.length}` : '本年无到期目标' }}</span>
        </div>

        <div class="goal-stats" v-if="yearGoals.length">
          <div class="goal-stat">
            <span class="goal-stat-label">已存合计</span>
            <b class="income">¥{{ money(yearGoalSaved) }}</b>
          </div>
          <div class="goal-stat">
            <span class="goal-stat-label">目标合计</span>
            <b>¥{{ money(yearGoalTarget) }}</b>
          </div>
          <div class="goal-stat">
            <span class="goal-stat-label">总体进度</span>
            <b class="accent">{{ yearGoalPercent }}%</b>
          </div>
        </div>

        <div class="goal-list">
          <div v-for="g in yearGoals" :key="g.id" class="goal-row">
            <div class="mini-ring-wrap">
              <ProgressRing :percent="g.percent" :size="64" :stroke-width="8" />
              <div class="mini-ring-center"><b>{{ g.percent }}%</b></div>
            </div>
            <div class="goal-info">
              <div class="goal-info-head">
                <b class="goal-name">{{ g.name }}</b>
                <span class="badge" :class="badgeClass(g)">{{ statusLabel(g) }}</span>
              </div>
              <div class="goal-bar">
                <div class="goal-bar-fill" :style="{ width: Math.min(100, g.percent) + '%' }"></div>
              </div>
              <div class="goal-meta">
                <span>已存 ¥{{ money(g.savedAmount) }}</span>
                <span>目标 ¥{{ money(g.targetAmount) }}</span>
                <span>目标日期 {{ g.targetDate }}</span>
              </div>
            </div>
          </div>
        </div>

        <div class="empty-inline" v-if="yearGoals.length === 0">
          {{ year }} 年没有计划完成的储蓄目标
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed } from 'vue'
import { useStore, controllersApi } from '../data/store.js'
import { money, yearMonthList } from '../core/utils.js'
import BarChart from '../components/BarChart.vue'
import ProgressRing from '../components/ProgressRing.vue'

const store = useStore()
const { report } = controllersApi

const INCOME_BAR = 'linear-gradient(180deg, #4f8df9, #2f6fe4)'
const EXPENSE_BAR = 'linear-gradient(180deg, #f9b86a, #ef8f2d)'

const now = new Date()
const currentYear = now.getFullYear()
const todayISO = `${now.getFullYear()}-${String(now.getMonth() + 1).padStart(2, '0')}-${String(now.getDate()).padStart(2, '0')}`
// 用模块级 ref 保留切换页签后的年份选择
const year = ref(currentYear)

const yearOptions = computed(() => {
  const years = new Set([currentYear])
  for (const t of store.transactions) years.add(Number(t.date.slice(0, 4)))
  for (const g of store.goals) if (g.targetDate) years.add(Number(g.targetDate.slice(0, 4)))
  return [...years].filter(Boolean).sort((a, b) => b - a)
})
const canPrev = computed(() => {
  const min = Math.min(...yearOptions.value)
  return year.value > min
})
const canNext = computed(() => year.value < currentYear)
const changeYear = (delta) => {
  const next = year.value + delta
  if (next >= Math.min(...yearOptions.value) && next <= currentYear) year.value = next
}

// 固定取 12 个月，没有数据的月份由 incomeAndExpense 返回 0
const monthlyRows = computed(() =>
  report.monthlySeries(yearMonthList(year.value)).map((m) => ({
    ...m,
    balance: m.income - m.expense
  }))
)

const yearIncome = computed(() => monthlyRows.value.reduce((s, m) => s + m.income, 0))
const yearExpense = computed(() => monthlyRows.value.reduce((s, m) => s + m.expense, 0))
const yearBalance = computed(() => yearIncome.value - yearExpense.value)
const yearSavingsRate = computed(() => report.savingsRate(yearIncome.value, yearExpense.value))

const barItems = computed(() =>
  monthlyRows.value.flatMap((m) => [
    { label: `${Number(m.month.slice(5))}月`, value: m.income, color: INCOME_BAR },
    { label: '', value: m.expense, color: EXPENSE_BAR }
  ])
)

const yearGoals = computed(() =>
  store.goals
    .filter((g) => g.targetDate && Number(g.targetDate.slice(0, 4)) === year.value)
    .map((g) => {
      const percent = g.targetAmount > 0 ? Math.round((g.savedAmount / g.targetAmount) * 100) : 0
      return { ...g, percent }
    })
)
const doneGoals = computed(() => yearGoals.value.filter((g) => g.percent >= 100))
const yearGoalSaved = computed(() => yearGoals.value.reduce((s, g) => s + g.savedAmount, 0))
const yearGoalTarget = computed(() => yearGoals.value.reduce((s, g) => s + g.targetAmount, 0))
const yearGoalPercent = computed(() =>
  yearGoalTarget.value > 0 ? Math.round((yearGoalSaved.value / yearGoalTarget.value) * 100) : 0
)

const statusLabel = (g) => {
  if (g.percent >= 100) return '已完成'
  if (year.value < currentYear) return '未完成'
  if (year.value > currentYear) return '未开始'
  return g.targetDate < todayISO ? '已逾期' : '进行中'
}
const badgeClass = (g) => {
  if (g.percent >= 100) return 'done'
  if (year.value < currentYear || (year.value === currentYear && g.targetDate < todayISO)) return 'danger'
  if (g.percent >= 60) return 'warn'
  return ''
}
</script>

<style scoped>
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

.year-switch {
  display: flex;
  align-items: center;
  gap: 6px;
}
.year-btn {
  width: 34px;
  height: 34px;
  border-radius: 10px;
  border: 1px solid var(--border-color);
  background: var(--card-bg);
  color: var(--text-primary);
  font-size: 18px;
  line-height: 1;
  cursor: pointer;
}
.year-btn:hover:not(:disabled) { background: var(--bg-elevated); }
.year-btn:disabled { opacity: 0.4; cursor: not-allowed; }
.year-select {
  padding: 8px 10px;
  border-radius: 10px;
  border: 1px solid var(--border-color);
  background: var(--bg-elevated);
  color: var(--text-primary);
  font-size: 14px;
  font-weight: 700;
  outline: none;
  cursor: pointer;
}
.year-select:focus { border-color: var(--accent); }

.chart-card { margin-bottom: 16px; }
.card-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 10px;
  margin-bottom: 12px;
  flex-wrap: wrap;
}
.card-title {
  margin: 0;
  font-size: 15px;
}
.chart-legend {
  display: flex;
  align-items: center;
  gap: 6px;
  font-size: 12px;
  color: var(--text-secondary);
}
.legend-dot {
  width: 10px;
  height: 10px;
  border-radius: 3px;
  margin-left: 8px;
}
.legend-dot:first-of-type { margin-left: 0; }
.income-dot { background: #4f8df9; }
.expense-dot { background: #f9a54f; }
.balance-dot { background: var(--text-secondary); }
.chart-scroll { overflow-x: auto; }
.chart-scroll > :deep(.chart-wrap) {
  min-width: 640px;
}

.dual-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  align-items: start;
}
@media (max-width: 860px) {
  .dual-grid { grid-template-columns: 1fr; }
}

.table-wrap { overflow-x: auto; }
.month-table {
  width: 100%;
  border-collapse: collapse;
  font-size: 13px;
}
.month-table th,
.month-table td {
  padding: 8px 10px;
  border-bottom: 1px solid var(--border-color);
  white-space: nowrap;
}
.month-table th {
  text-align: left;
  color: var(--text-secondary);
  font-weight: 600;
}
.month-table .num { text-align: right; }
.month-table .income { color: var(--income); }
.month-table .expense { color: var(--expense); }
.month-table .neg { color: var(--expense); }
.month-table tfoot td {
  border-bottom: none;
  border-top: 2px solid var(--border-color);
  font-weight: 800;
}

.goal-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 8px;
  margin-bottom: 14px;
}
.goal-stat {
  background: var(--bg-elevated);
  border-radius: 10px;
  padding: 10px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  text-align: center;
}
.goal-stat-label {
  font-size: 11px;
  color: var(--text-secondary);
}
.goal-stat b {
  font-size: 15px;
}
.goal-stat b.income { color: var(--income); }
.goal-stat b.accent { color: var(--accent); }

.goal-list {
  display: flex;
  flex-direction: column;
  gap: 14px;
}
.goal-row {
  display: flex;
  gap: 12px;
  align-items: center;
}
.mini-ring-wrap {
  position: relative;
  flex-shrink: 0;
}
.mini-ring-center {
  position: absolute;
  inset: 0;
  display: flex;
  align-items: center;
  justify-content: center;
}
.mini-ring-center b {
  font-size: 12px;
}
.goal-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 5px;
}
.goal-info-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 8px;
}
.goal-name {
  font-size: 13px;
}
.goal-bar {
  height: 6px;
  border-radius: 999px;
  background: var(--bg-elevated);
  overflow: hidden;
}
.goal-bar-fill {
  height: 100%;
  border-radius: 999px;
  background: linear-gradient(90deg, #4f8df9, #936df0);
  transition: width 0.35s ease;
}
.goal-meta {
  display: flex;
  gap: 10px;
  flex-wrap: wrap;
  font-size: 11px;
  color: var(--text-secondary);
}
.empty-inline {
  text-align: center;
  color: var(--text-secondary);
  font-size: 13px;
  padding: 24px 0;
}
</style>
