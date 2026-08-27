<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t("reports.title") }}</h2>
      <p>{{ t("reports.description") }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t("common.loading") }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Quarterly Performance -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t("reports.quarterlyPerformance.title") }}
          </h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t("reports.quarterlyPerformance.quarter") }}</th>
                <th>{{ t("reports.quarterlyPerformance.totalOrders") }}</th>
                <th>{{ t("reports.quarterlyPerformance.totalRevenue") }}</th>
                <th>{{ t("reports.quarterlyPerformance.avgOrderValue") }}</th>
                <th>
                  {{ t("reports.quarterlyPerformance.fulfillmentRate") }}
                </th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="q in quarterlyData" :key="q.quarter">
                <td>
                  <strong>{{ q.quarter }}</strong>
                </td>
                <td>{{ q.total_orders }}</td>
                <td>{{ formatCurrency(q.total_revenue) }}</td>
                <td>{{ formatCurrency(q.avg_order_value) }}</td>
                <td>
                  <span :class="getFulfillmentClass(q.fulfillment_rate)">
                    {{ q.fulfillment_rate }}%
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Monthly Trends Chart -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t("reports.monthlyTrend.title") }}</h3>
        </div>
        <div class="chart-container">
          <div class="bar-chart">
            <div
              v-for="month in monthlyData"
              :key="month.month"
              class="bar-wrapper"
            >
              <div class="bar-container">
                <div
                  class="bar"
                  :style="{ height: getBarHeight(month.revenue) + 'px' }"
                  :title="formatCurrency(month.revenue)"
                ></div>
              </div>
              <div class="bar-label">{{ formatMonth(month.month) }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Month-over-Month Comparison -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t("reports.monthOverMonth.title") }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t("reports.monthOverMonth.month") }}</th>
                <th>{{ t("reports.monthOverMonth.orders") }}</th>
                <th>{{ t("reports.monthOverMonth.revenue") }}</th>
                <th>{{ t("reports.monthOverMonth.change") }}</th>
                <th>{{ t("reports.monthOverMonth.growthRate") }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="(month, index) in monthlyData" :key="month.month">
                <td>
                  <strong>{{ formatMonth(month.month) }}</strong>
                </td>
                <td>{{ month.order_count }}</td>
                <td>{{ formatCurrency(month.revenue) }}</td>
                <td>
                  <span
                    v-if="index > 0"
                    :class="
                      getChangeClass(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    "
                  >
                    {{
                      getChangeValue(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    }}
                  </span>
                  <span v-else>-</span>
                </td>
                <td>
                  <span
                    v-if="index > 0"
                    :class="
                      getChangeClass(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    "
                  >
                    {{
                      getGrowthRate(
                        month.revenue,
                        monthlyData[index - 1].revenue,
                      )
                    }}
                  </span>
                  <span v-else>-</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">
            {{ t("reports.summary.totalRevenueYtd") }}
          </div>
          <div class="stat-value">{{ formatCurrency(totalRevenue) }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">
            {{ t("reports.summary.avgMonthlyRevenue") }}
          </div>
          <div class="stat-value">
            {{ formatCurrency(avgMonthlyRevenue) }}
          </div>
        </div>
        <div class="stat-card">
          <div class="stat-label">
            {{ t("reports.summary.totalOrdersYtd") }}
          </div>
          <div class="stat-value">{{ totalOrders }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t("reports.summary.bestQuarter") }}</div>
          <div class="stat-value">{{ bestQuarter }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, computed, watch, onMounted } from "vue";
import { api } from "../api";
import { useFilters } from "../composables/useFilters";
import { useI18n } from "../composables/useI18n";
import { formatCurrencyWithDecimals as formatCurrencyWithDecimalsUtil } from "../utils/currency";

const { t, currentCurrency } = useI18n();

const { selectedLocation, selectedCategory, selectedStatus } = useFilters();

const loading = ref(true);
const error = ref(null);
const quarterlyData = ref([]);
const monthlyData = ref([]);

const formatCurrency = (amount) =>
  formatCurrencyWithDecimalsUtil(amount, currentCurrency.value, 2);

const totalRevenue = computed(() =>
  monthlyData.value.reduce((sum, m) => sum + m.revenue, 0),
);

const avgMonthlyRevenue = computed(() =>
  monthlyData.value.length > 0
    ? totalRevenue.value / monthlyData.value.length
    : 0,
);

const totalOrders = computed(() =>
  monthlyData.value.reduce((sum, m) => sum + m.order_count, 0),
);

const bestQuarter = computed(() => {
  let bestQ = "";
  let bestRevenue = 0;
  quarterlyData.value.forEach((q) => {
    if (q.total_revenue > bestRevenue) {
      bestRevenue = q.total_revenue;
      bestQ = q.quarter;
    }
  });
  return bestQ;
});

const maxMonthlyRevenue = computed(() => {
  if (monthlyData.value.length === 0) return 0;
  return Math.max(...monthlyData.value.map((m) => m.revenue));
});

const loadData = async () => {
  try {
    loading.value = true;
    error.value = null;

    const filters = {
      warehouse: selectedLocation.value,
      category: selectedCategory.value,
      status: selectedStatus.value,
    };

    const [quarterlyResponse, monthlyResponse] = await Promise.all([
      api.getQuarterlyReports(filters),
      api.getMonthlyTrends(filters),
    ]);

    quarterlyData.value = quarterlyResponse;
    monthlyData.value = monthlyResponse;
  } catch (err) {
    error.value = "Failed to load reports: " + err.message;
    console.error(err);
  } finally {
    loading.value = false;
  }
};

const formatMonth = (monthStr) => {
  // Convert YYYY-MM to translated "Mon YYYY" format
  const parts = monthStr.split("-");
  const year = parts[0];
  const monthKeys = [
    "jan",
    "feb",
    "mar",
    "apr",
    "may",
    "jun",
    "jul",
    "aug",
    "sep",
    "oct",
    "nov",
    "dec",
  ];
  const monthIndex = parseInt(parts[1], 10) - 1;
  const monthKey = monthKeys[monthIndex];
  const monthLabel = monthKey ? t(`months.${monthKey}`) : parts[1];

  return `${monthLabel} ${year}`;
};

const getBarHeight = (revenue) => {
  // Calculate bar height (max height 200px)
  if (maxMonthlyRevenue.value === 0) return 0;
  return (revenue / maxMonthlyRevenue.value) * 200;
};

const getFulfillmentClass = (rate) => {
  if (rate >= 90) return "badge success";
  if (rate >= 75) return "badge warning";
  return "badge danger";
};

const getChangeValue = (current, previous) => {
  const change = current - previous;
  if (change > 0) return `+${formatCurrency(change)}`;
  if (change < 0) return `-${formatCurrency(Math.abs(change))}`;
  return formatCurrency(0);
};

const getChangeClass = (current, previous) => {
  const change = current - previous;
  if (change > 0) return "positive-change";
  if (change < 0) return "negative-change";
  return "";
};

const getGrowthRate = (current, previous) => {
  if (previous === 0) return "N/A";
  const rate = ((current - previous) / previous) * 100;
  const sign = rate > 0 ? "+" : "";
  return `${sign}${rate.toFixed(1)}%`;
};

watch([selectedLocation, selectedCategory, selectedStatus], () => loadData());

onMounted(() => loadData());
</script>

<style scoped>
.reports {
  padding: 0;
}

.card {
  background: var(--color-bg-surface);
  border-radius: 12px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  box-shadow: var(--shadow-md);
}

.card-header {
  margin-bottom: 1.5rem;
}

.card-title {
  font-size: 1.25rem;
  font-weight: 600;
  color: var(--color-text-primary);
  margin: 0;
}

.reports-table {
  width: 100%;
  border-collapse: collapse;
}

.reports-table th {
  background: var(--color-bg-page);
  padding: 0.75rem;
  text-align: left;
  font-weight: 600;
  color: var(--color-text-muted);
  border-bottom: 2px solid var(--color-border-default);
}

.reports-table td {
  padding: 0.75rem;
  border-bottom: 1px solid var(--color-border-default);
}

.reports-table tr:hover {
  background: var(--color-bg-page);
}

.chart-container {
  padding: 2rem 1rem;
  min-height: 300px;
}

.bar-chart {
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  height: 250px;
  gap: 0.5rem;
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  max-width: 80px;
}

.bar-container {
  height: 200px;
  display: flex;
  align-items: flex-end;
  width: 100%;
}

.bar {
  width: 100%;
  background: linear-gradient(
    to top,
    var(--color-accent),
    var(--color-accent-hover)
  );
  border-radius: 4px 4px 0 0;
  transition: all 0.3s;
  cursor: pointer;
}

.bar:hover {
  background: linear-gradient(
    to top,
    var(--color-accent-hover),
    var(--color-accent)
  );
}

.bar-label {
  margin-top: 0.5rem;
  font-size: 0.75rem;
  color: var(--color-text-muted);
  text-align: center;
  transform: rotate(-45deg);
  white-space: nowrap;
  margin-top: 1.5rem;
}

.stats-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(250px, 1fr));
  gap: 1rem;
  margin-top: 1.5rem;
}

.stat-card {
  background: var(--color-bg-surface);
  border-radius: 12px;
  padding: 1.5rem;
  box-shadow: var(--shadow-md);
  border-left: 4px solid var(--color-accent);
}

.stat-label {
  font-size: 0.875rem;
  color: var(--color-text-muted);
  margin-bottom: 0.5rem;
}

.stat-value {
  font-size: 1.875rem;
  font-weight: 700;
  color: var(--color-text-primary);
}

.badge {
  padding: 0.25rem 0.75rem;
  border-radius: 9999px;
  font-size: 0.875rem;
  font-weight: 500;
}

.badge.success {
  background: var(--color-success-bg);
  color: var(--color-success-text);
}

.badge.warning {
  background: var(--color-warning-bg);
  color: var(--color-warning-text);
}

.badge.danger {
  background: var(--color-danger-bg);
  color: var(--color-danger-text);
}

.positive-change {
  color: var(--color-success);
  font-weight: 600;
}

.negative-change {
  color: var(--color-danger);
  font-weight: 600;
}

.loading {
  text-align: center;
  padding: 3rem;
  color: var(--color-text-muted);
}

.error {
  background: var(--color-danger-subtle-bg);
  color: var(--color-danger-text);
  padding: 1rem;
  border-radius: 8px;
  margin: 1rem 0;
}
</style>
