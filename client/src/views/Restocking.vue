<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Success banner (shown after order placed) -->
    <div v-if="submitted" class="success-banner">
      {{ t('restocking.orderSuccess') }}
    </div>

    <div v-if="loading">{{ t('common.loading') }}</div>
    <div v-else>
      <!-- Budget Configuration Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetTitle') }}</h3>
        </div>
        <div class="budget-config">
          <div class="budget-label">{{ t('restocking.budgetLabel') }}</div>
          <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <input
            type="range"
            min="0"
            max="100000"
            step="2500"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-range-labels">
            <span>{{ currencySymbol }}0</span>
            <span>{{ currencySymbol }}100,000</span>
          </div>
        </div>
      </div>

      <!-- Recommended Items Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.recommendationsTitle') }}
            <span class="item-count">({{ recommendedItems.length }})</span>
          </h3>
        </div>

        <div v-if="budget === 0 || recommendedItems.length === 0" class="empty-state">
          {{ t('restocking.noBudget') }}
        </div>

        <div v-else>
          <div class="table-container">
            <table class="restock-table">
              <thead>
                <tr>
                  <th>{{ t('restocking.sku') }}</th>
                  <th>{{ t('restocking.itemName') }}</th>
                  <th>{{ t('restocking.trend') }}</th>
                  <th>{{ t('restocking.forecastedQty') }}</th>
                  <th>{{ t('restocking.unitCost') }}</th>
                  <th>{{ t('restocking.estimatedCost') }}</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendedItems" :key="item.id">
                  <td class="sku-cell"><code>{{ item.item_sku }}</code></td>
                  <td>{{ item.item_name }}</td>
                  <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                  <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                  <td>{{ currencySymbol }}{{ item.unit_cost.toFixed(2) }}</td>
                  <td class="cost-cell">
                    <strong>{{ currencySymbol }}{{ item.item_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Budget progress bar -->
          <div class="budget-summary">
            <div class="budget-summary-labels">
              <span>{{ t('restocking.budgetUsed') }}: <strong>{{ currencySymbol }}{{ budgetUsed.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></span>
              <span>{{ t('restocking.budgetAvailable') }}: <strong>{{ currencySymbol }}{{ budget.toLocaleString() }}</strong></span>
            </div>
            <div class="budget-bar">
              <div
                class="budget-bar-fill"
                :style="{ width: Math.min(100, (budgetUsed / budget) * 100) + '%' }"
              ></div>
            </div>
          </div>

          <div class="order-actions">
            <button
              class="btn-primary"
              :disabled="submitting || recommendedItems.length === 0"
              @click="placeOrder"
            >
              {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const submitting = ref(false)
    const submitted = ref(false)
    const demandForecasts = ref([])
    const budget = ref(50000)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    // Sort forecasts by trend priority (increasing first) then by demand delta descending,
    // then greedily include items that fit within the budget.
    const recommendedItems = computed(() => {
      const trendPriority = { increasing: 0, stable: 1, decreasing: 2 }

      const sorted = [...demandForecasts.value].sort((a, b) => {
        const trendDiff = (trendPriority[a.trend] ?? 3) - (trendPriority[b.trend] ?? 3)
        if (trendDiff !== 0) return trendDiff
        // Break ties by demand delta descending
        const deltaA = a.forecasted_demand - a.current_demand
        const deltaB = b.forecasted_demand - b.current_demand
        return deltaB - deltaA
      })

      const result = []
      let cumulative = 0

      for (const forecast of sorted) {
        const itemCost = forecast.forecasted_demand * forecast.unit_cost
        if (cumulative + itemCost <= budget.value) {
          result.push({ ...forecast, item_cost: itemCost })
          cumulative += itemCost
        }
      }

      return result
    })

    const budgetUsed = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
    )

    const loadForecasts = async () => {
      loading.value = true
      try {
        demandForecasts.value = await api.getDemandForecasts()
      } catch (err) {
        console.error('Failed to load demand forecasts:', err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      try {
        const items = recommendedItems.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.forecasted_demand,
          unit_price: item.unit_cost
        }))
        await api.submitRestockOrder({
          items,
          budget: budget.value,
          total_value: budgetUsed.value
        })
        submitted.value = true
      } catch (err) {
        console.error('Failed to submit restock order:', err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      loading,
      submitting,
      submitted,
      budget,
      currencySymbol,
      recommendedItems,
      budgetUsed,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-config {
  padding: 1.5rem;
  display: flex;
  flex-direction: column;
  gap: 1rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.025em;
  color: #64748b;
}

.budget-display {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #3b82f6;
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
}

.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.restock-table th,
.restock-table td {
  padding: 0.75rem 1rem;
  text-align: left;
  border-bottom: 1px solid #e2e8f0;
  font-size: 0.875rem;
}

.restock-table th {
  font-weight: 600;
  color: #64748b;
  background: #f8fafc;
  text-transform: uppercase;
  font-size: 0.75rem;
  letter-spacing: 0.025em;
}

.restock-table tbody tr:hover {
  background: #f8fafc;
}

.restock-table tbody tr:last-child td {
  border-bottom: none;
}

.sku-cell code {
  font-family: monospace;
  background: #f1f5f9;
  padding: 2px 6px;
  border-radius: 4px;
  font-size: 0.813rem;
}

.cost-cell {
  text-align: right;
}

.budget-summary {
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
}

.budget-summary-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.875rem;
  color: #64748b;
  margin-bottom: 0.5rem;
}

.budget-bar {
  height: 8px;
  background: #e2e8f0;
  border-radius: 4px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  background: #3b82f6;
  border-radius: 4px;
  transition: width 0.3s ease;
}

.order-actions {
  padding: 1rem 1.5rem;
  display: flex;
  justify-content: flex-end;
}

.btn-primary {
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.5rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.15s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #2563eb;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.empty-state {
  padding: 3rem 1.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.75rem 1rem;
  color: #065f46;
  font-weight: 500;
  margin-bottom: 1rem;
}

.item-count {
  font-weight: 400;
  color: #64748b;
  font-size: 0.875rem;
  margin-left: 0.5rem;
}
</style>
