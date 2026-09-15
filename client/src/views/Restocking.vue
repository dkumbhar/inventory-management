<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="successMessage" class="banner success-banner">
        {{ successMessage }}
      </div>
      <div v-if="submitError" class="banner error-banner">
        {{ submitError }}
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-controls">
          <input
            type="range"
            class="budget-slider"
            min="0"
            :max="maxBudget"
            step="100"
            v-model.number="budget"
          />
          <div class="budget-value-group">
            <span class="budget-currency">{{ currencySymbol }}</span>
            <input
              type="number"
              class="budget-input"
              min="0"
              :max="maxBudget"
              step="100"
              v-model.number="budget"
            />
          </div>
          <span class="budget-display">{{ currencySymbol }}{{ Math.round(budget).toLocaleString() }}</span>
        </div>
        <div class="budget-summary">
          <div class="budget-summary-item">
            <span class="budget-summary-label">{{ t('restocking.selectedTotal') }}</span>
            <span class="budget-summary-value">{{ currencySymbol }}{{ Math.round(selectedTotal).toLocaleString() }}</span>
          </div>
          <div class="budget-summary-item">
            <span class="budget-summary-label">{{ t('restocking.remainingBudget') }}</span>
            <span
              class="budget-summary-value"
              :class="{ danger: remainingBudget < 0 }"
            >{{ currencySymbol }}{{ Math.round(remainingBudget).toLocaleString() }}</span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }} ({{ candidates.length }})</h3>
          <button
            class="place-order-btn"
            :disabled="selectedSkus.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('common.loading') : t('restocking.placeOrder') }}
          </button>
        </div>
        <div v-if="candidates.length === 0" class="empty-state">
          {{ t('restocking.noCandidates') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th></th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.priority') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.restockQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineCost') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="candidate in candidates" :key="candidate.sku">
                <td>
                  <input
                    type="checkbox"
                    :checked="selectedSkus.includes(candidate.sku)"
                    @change="toggleSelected(candidate.sku)"
                  />
                </td>
                <td><strong>{{ candidate.sku }}</strong></td>
                <td>{{ candidate.item_name }}</td>
                <td>
                  <span :class="{ danger: candidate.is_urgent }">{{ candidate.current_stock }}</span>
                </td>
                <td>
                  <span v-if="candidate.priority" :class="['badge', `priority-${candidate.priority}`]">
                    {{ t(`priority.${candidate.priority}`) }}
                  </span>
                  <span v-else class="priority-none">&mdash;</span>
                </td>
                <td>{{ candidate.forecasted_demand }}</td>
                <td><strong>{{ candidate.restock_qty }}</strong></td>
                <td>{{ currencySymbol }}{{ candidate.unit_cost.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ candidate.cost.toLocaleString() }}</td>
                <td>
                  <span :class="['badge', candidate.trend]">
                    {{ t(`trends.${candidate.trend}`) }}
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

// Order for urgency-first sort: increasing > stable > decreasing
const TREND_ORDER = { increasing: 0, stable: 1, decreasing: 2 }

// Order for backlog priority: high > medium > low
const PRIORITY_ORDER = { high: 0, medium: 1, low: 2 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const backlogItems = ref([])

    const budget = ref(0)
    const selectedSkus = ref([])
    const submitting = ref(false)
    const submitError = ref(null)
    const successMessage = ref(null)

    // Round a number up to a "nice" step (nearest 100 above)
    const roundUpNice = (value) => Math.ceil(value / 100) * 100

    // Round a number to a "nice" step for the default budget
    const roundNice = (value) => Math.round(value / 100) * 100

    // Join demand forecasts to backlog entries by SKU and compute restock candidates.
    // Demand-forecast SKUs (finished/backlog goods) and inventory SKUs (raw components)
    // are separate SKU universes in this app, so we no longer join against inventory here.
    // Backlog only covers a subset of forecast SKUs, so items without a backlog match fall
    // back to current_demand as a proxy for current stock.
    const candidates = computed(() => {
      const backlogBySku = new Map(backlogItems.value.map(item => [item.item_sku, item]))

      const joined = []
      for (const forecast of forecasts.value) {
        const backlogMatch = backlogBySku.get(forecast.item_sku)

        const current_stock = backlogMatch ? backlogMatch.quantity_available : forecast.current_demand
        const restock_qty = Math.max(forecast.forecasted_demand - current_stock, 0)
        if (restock_qty === 0) continue

        const cost = restock_qty * forecast.unit_cost
        const is_urgent = !!backlogMatch

        joined.push({
          sku: forecast.item_sku,
          item_name: forecast.item_name,
          current_stock,
          forecasted_demand: forecast.forecasted_demand,
          restock_qty,
          unit_cost: forecast.unit_cost,
          cost,
          trend: forecast.trend,
          is_urgent,
          priority: backlogMatch?.priority ?? null,
          days_delayed: backlogMatch?.days_delayed ?? null
        })
      }

      // Backlog-urgent items first (ranked by priority, then most-delayed first),
      // then non-backlog items by trend (increasing > stable > decreasing), then by cost descending
      joined.sort((a, b) => {
        if (a.is_urgent !== b.is_urgent) return a.is_urgent ? -1 : 1

        if (a.is_urgent) {
          const priorityDiff = (PRIORITY_ORDER[a.priority] ?? 99) - (PRIORITY_ORDER[b.priority] ?? 99)
          if (priorityDiff !== 0) return priorityDiff

          return (b.days_delayed ?? 0) - (a.days_delayed ?? 0)
        }

        const trendDiff = (TREND_ORDER[a.trend] ?? 99) - (TREND_ORDER[b.trend] ?? 99)
        if (trendDiff !== 0) return trendDiff

        return b.cost - a.cost
      })

      return joined
    })

    const totalCandidateCost = computed(() => {
      return candidates.value.reduce((sum, c) => sum + c.cost, 0)
    })

    const maxBudget = computed(() => {
      return Math.max(roundUpNice(totalCandidateCost.value), 100)
    })

    // Greedily walk urgency-sorted candidates, adding while within budget
    const recommendedItems = computed(() => {
      const result = []
      let runningTotal = 0
      for (const candidate of candidates.value) {
        if (runningTotal + candidate.cost <= budget.value) {
          result.push(candidate)
          runningTotal += candidate.cost
        }
      }
      return result
    })

    // Default selection follows the recommended set whenever it changes
    watch(recommendedItems, (items) => {
      selectedSkus.value = items.map(item => item.sku)
    })

    const selectedTotal = computed(() => {
      const selectedSet = new Set(selectedSkus.value)
      return candidates.value
        .filter(c => selectedSet.has(c.sku))
        .reduce((sum, c) => sum + c.cost, 0)
    })

    const remainingBudget = computed(() => budget.value - selectedTotal.value)

    const toggleSelected = (sku) => {
      const idx = selectedSkus.value.indexOf(sku)
      if (idx === -1) {
        selectedSkus.value = [...selectedSkus.value, sku]
      } else {
        selectedSkus.value = selectedSkus.value.filter(s => s !== sku)
      }
    }

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null

        const [forecastsData, backlogData] = await Promise.all([
          api.getDemandForecasts(),
          api.getBacklog()
        ])

        forecasts.value = forecastsData
        backlogItems.value = backlogData

        // Default budget: half of total candidate cost, rounded to a nice number
        budget.value = roundNice(totalCandidateCost.value / 2)
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitError.value = null
      successMessage.value = null

      const selectedSet = new Set(selectedSkus.value)
      const selectedCandidates = candidates.value.filter(c => selectedSet.has(c.sku))
      if (selectedCandidates.length === 0) return

      try {
        submitting.value = true
        const payload = {
          items: selectedCandidates.map(c => ({
            sku: c.sku,
            item_name: c.item_name,
            quantity: c.restock_qty,
            unit_cost: c.unit_cost
          })),
          budget: budget.value
        }

        const order = await api.createRestockOrder(payload)
        successMessage.value = t('restocking.orderPlaced', {
          orderNumber: order.order_number,
          date: order.expected_delivery
        })

        selectedSkus.value = []
        await loadData()
      } catch (err) {
        submitError.value = 'Failed to place restock order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      t,
      loading,
      error,
      candidates,
      budget,
      maxBudget,
      selectedSkus,
      selectedTotal,
      remainingBudget,
      submitting,
      submitError,
      successMessage,
      currencySymbol,
      toggleSelected,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-controls {
  display: flex;
  align-items: center;
  gap: 1rem;
  flex-wrap: wrap;
}

.budget-slider {
  flex: 1;
  min-width: 200px;
  accent-color: #2563eb;
}

.budget-value-group {
  display: flex;
  align-items: center;
  gap: 0.375rem;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  padding: 0.375rem 0.625rem;
}

.budget-currency {
  color: #64748b;
  font-weight: 600;
  font-size: 0.875rem;
}

.budget-input {
  width: 110px;
  border: none;
  background: transparent;
  font-size: 0.875rem;
  color: #0f172a;
  font-weight: 600;
}

.budget-input:focus {
  outline: none;
}

.budget-display {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 100px;
  text-align: right;
}

.budget-summary {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 1rem;
  margin-top: 1rem;
  padding-top: 1rem;
  border-top: 1px solid #f1f5f9;
}

.budget-summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-summary-label {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-summary-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-summary-value.danger {
  color: #ef4444;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 6px;
  padding: 0.5rem 1.125rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.banner {
  padding: 0.875rem 1rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.success-banner {
  background: #ecfdf5;
  border: 1px solid #a7f3d0;
  color: #065f46;
}

.error-banner {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
}

.empty-state {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

td span.danger {
  color: #ef4444;
  font-weight: 700;
}

.badge.priority-high {
  background: #fee2e2;
  color: #ef4444;
}

.badge.priority-medium {
  background: #fef3c7;
  color: #b45309;
}

.badge.priority-low {
  background: #dbeafe;
  color: #2563eb;
}

.priority-none {
  color: #64748b;
}
</style>
