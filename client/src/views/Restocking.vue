<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Budget Section -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.availableBudget') }}</h3>
          <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>

        <div class="budget-slider-wrap">
          <input
            type="range"
            min="1000"
            max="100000"
            step="1000"
            v-model.number="budget"
            class="budget-slider"
          />
        </div>

        <div class="budget-bar-section">
          <div class="budget-bar-labels">
            <span class="budget-bar-label">{{ t('restocking.allocated') }}: {{ currencySymbol }}{{ totalAllocated.toLocaleString() }}</span>
            <span class="budget-bar-label">{{ t('restocking.remaining') }}: {{ currencySymbol }}{{ budgetRemaining.toLocaleString() }}</span>
          </div>
          <div class="budget-bar-track">
            <div
              class="budget-bar-fill"
              :style="{ width: budget > 0 ? Math.min((totalAllocated / budget) * 100, 100) + '%' : '0%' }"
            ></div>
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
          <div class="card-header-actions">
            <span v-if="recommendations.length > 0" class="items-count">
              {{ recommendations.length }} item{{ recommendations.length !== 1 ? 's' : '' }}
            </span>
          </div>
        </div>

        <div v-if="submittedOrderNumber" class="success-banner">
          Order {{ submittedOrderNumber }} submitted successfully. Expected delivery in ~14 business days.
        </div>

        <div v-if="recommendations.length === 0 && budget > 0" class="empty-state">
          {{ t('restocking.noItems') }}
        </div>

        <div v-else-if="recommendations.length > 0" class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.forecastedQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.allocatedQty') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.id">
                <td><strong>{{ item.item_name }}</strong></td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td>{{ item.allocated_qty.toLocaleString() }}</td>
                <td><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>

          <div class="table-summary">
            <span class="summary-count">{{ recommendations.length }} item{{ recommendations.length !== 1 ? 's' : '' }} selected</span>
            <span class="summary-total">Total: <strong>{{ currencySymbol }}{{ totalAllocated.toLocaleString() }}</strong></span>
          </div>
        </div>

        <div class="place-order-row">
          <button
            class="btn-place-order"
            :disabled="recommendations.length === 0 || submitting || !!submittedOrderNumber"
            @click="placeOrder"
          >
            <span v-if="submitting">{{ t('restocking.submittingOrder') }}</span>
            <span v-else-if="submittedOrderNumber">{{ t('restocking.orderPlaced') }}</span>
            <span v-else>{{ t('restocking.placeOrder') }}</span>
          </button>
        </div>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useI18n } from '../composables/useI18n'
import { api } from '../api'

const TREND_PRIORITY = { increasing: 1, stable: 2, decreasing: 3 }

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(25000)
    const submitting = ref(false)
    const submittedOrderNumber = ref(null)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const recommendations = computed(() => {
      const sorted = [...forecasts.value].sort(
        (a, b) => (TREND_PRIORITY[a.trend] || 9) - (TREND_PRIORITY[b.trend] || 9)
      )
      let remaining = budget.value
      const result = []
      for (const item of sorted) {
        if (remaining <= 0) break
        const fullCost = item.forecasted_demand * item.unit_cost
        if (fullCost <= remaining) {
          result.push({ ...item, allocated_qty: item.forecasted_demand, line_total: fullCost })
          remaining -= fullCost
        } else {
          const qty = Math.floor(remaining / item.unit_cost)
          if (qty > 0) {
            result.push({ ...item, allocated_qty: qty, line_total: qty * item.unit_cost })
            remaining -= qty * item.unit_cost
          }
        }
      }
      return result
    })

    const totalAllocated = computed(() =>
      recommendations.value.reduce((sum, r) => sum + r.line_total, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalAllocated.value)

    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = t('common.error')
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      try {
        const items = recommendations.value.map(r => ({
          sku: r.item_sku,
          name: r.item_name,
          quantity: r.allocated_qty,
          unit_price: r.unit_cost
        }))
        const result = await api.submitRestockingOrder(items, totalAllocated.value)
        submittedOrderNumber.value = result.order.order_number
      } catch (err) {
        console.error('Failed to submit restocking order:', err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadForecasts())

    return {
      t,
      loading,
      error,
      budget,
      currencySymbol,
      recommendations,
      totalAllocated,
      budgetRemaining,
      submitting,
      submittedOrderNumber,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

/* Budget slider */
.budget-slider-wrap {
  padding: 0.5rem 0 1rem;
}

.budget-slider {
  width: 100%;
  height: 6px;
  appearance: none;
  -webkit-appearance: none;
  background: #e2e8f0;
  border-radius: 4px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 2px 8px rgba(37, 99, 235, 0.5);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #fff;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

/* Budget display in card-header */
.budget-display {
  font-size: 1.75rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

/* Budget progress bar */
.budget-bar-section {
  margin-top: 0.5rem;
}

.budget-bar-labels {
  display: flex;
  justify-content: space-between;
  margin-bottom: 0.5rem;
}

.budget-bar-label {
  font-size: 0.813rem;
  font-weight: 600;
  color: #64748b;
}

.budget-bar-track {
  width: 100%;
  height: 10px;
  background: #e2e8f0;
  border-radius: 6px;
  overflow: hidden;
}

.budget-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 6px;
  transition: width 0.3s ease;
}

/* Card header actions */
.card-header-actions {
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.items-count {
  font-size: 0.813rem;
  color: #64748b;
  font-weight: 500;
}

/* Success banner */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 500;
  margin-bottom: 1rem;
}

/* Empty state */
.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Table summary row */
.table-summary {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 0.75rem;
  border-top: 2px solid #e2e8f0;
  background: #f8fafc;
  font-size: 0.875rem;
  color: #475569;
}

.summary-count {
  font-weight: 500;
}

.summary-total {
  font-weight: 500;
}

.summary-total strong {
  color: #0f172a;
  font-size: 1rem;
}

/* Place order row */
.place-order-row {
  display: flex;
  justify-content: flex-end;
  padding-top: 1rem;
  margin-top: 0.25rem;
  border-top: 1px solid #e2e8f0;
}

.btn-place-order {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease, opacity 0.15s ease;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}
</style>
