<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Allocate budget to forecasted demand and submit a restocking order.</p>
    </div>

    <div class="card budget-card">
      <div class="card-header">
        <h3 class="card-title">Available Budget</h3>
        <div class="budget-value">{{ formatCurrency(budget) }}</div>
      </div>
      <div class="slider-row">
        <span class="slider-bound">{{ formatCurrency(MIN_BUDGET) }}</span>
        <input
          type="range"
          class="budget-slider"
          :min="MIN_BUDGET"
          :max="MAX_BUDGET"
          :step="STEP_BUDGET"
          v-model.number="budget"
        />
        <span class="slider-bound">{{ formatCurrency(MAX_BUDGET) }}</span>
      </div>
    </div>

    <div v-if="banner" :class="['banner', banner.type]">{{ banner.message }}</div>

    <div v-if="loading" class="loading">Calculating recommendations...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ rows.length }})</h3>
        </div>
        <div v-if="rows.length === 0" class="empty-state">
          No items can be recommended within this budget.
        </div>
        <div v-else class="table-container">
          <table class="recs-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item</th>
                <th>Warehouse</th>
                <th>Category</th>
                <th class="num">Forecast</th>
                <th class="num">On hand</th>
                <th class="num">Quantity</th>
                <th class="num">Unit cost</th>
                <th class="num">Line total</th>
                <th class="num">Lead time</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in rows" :key="row.item_sku">
                <td><strong>{{ row.item_sku }}</strong></td>
                <td>{{ row.item_name }}</td>
                <td>{{ row.warehouse }}</td>
                <td>{{ row.category }}</td>
                <td class="num">{{ row.forecasted_demand.toLocaleString() }}</td>
                <td class="num">{{ row.quantity_on_hand.toLocaleString() }}</td>
                <td class="num">
                  <input
                    type="number"
                    class="qty-input"
                    min="0"
                    :max="row.forecasted_demand"
                    v-model.number="row.quantity"
                  />
                </td>
                <td class="num">{{ formatCurrency(row.unit_cost) }}</td>
                <td class="num">{{ formatCurrency(row.quantity * row.unit_cost) }}</td>
                <td class="num">{{ row.lead_time_days }} days</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="card summary-card">
        <div class="summary-grid">
          <div class="summary-item">
            <div class="summary-label">Total items</div>
            <div class="summary-value">{{ totalUnits.toLocaleString() }}</div>
          </div>
          <div class="summary-item">
            <div class="summary-label">Total cost</div>
            <div class="summary-value">{{ formatCurrency(totalCost) }}</div>
          </div>
          <div class="summary-item">
            <div class="summary-label">Budget remaining</div>
            <div :class="['summary-value', { negative: budgetRemaining < 0 }]">
              {{ formatCurrency(budgetRemaining) }}
            </div>
          </div>
          <div class="summary-action">
            <button
              class="place-order-btn"
              :disabled="!canSubmit"
              @click="placeOrder"
            >
              {{ submitting ? 'Submitting...' : 'Place Order' }}
            </button>
          </div>
        </div>
        <div v-if="totalCost > budget" class="over-budget-note">
          Order total exceeds available budget by {{ formatCurrency(totalCost - budget) }}.
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'

const MIN_BUDGET = 0
const MAX_BUDGET = 500000
const STEP_BUDGET = 1000
const DEFAULT_BUDGET = 100000
const DEBOUNCE_MS = 300

export default {
  name: 'Restocking',
  setup() {
    const budget = ref(DEFAULT_BUDGET)
    const rows = ref([])
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const banner = ref(null)

    let debounceHandle = null

    const formatCurrency = (value) => {
      const num = Number.isFinite(value) ? value : 0
      return num.toLocaleString('en-US', {
        style: 'currency',
        currency: 'USD',
        maximumFractionDigits: 0
      })
    }

    const totalUnits = computed(() =>
      rows.value.reduce((sum, r) => sum + (Number(r.quantity) || 0), 0)
    )

    const totalCost = computed(() =>
      rows.value.reduce((sum, r) => sum + (Number(r.quantity) || 0) * r.unit_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    const canSubmit = computed(() =>
      !submitting.value &&
      totalUnits.value > 0 &&
      totalCost.value <= budget.value
    )

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        const data = await api.getRestockingRecommendations(budget.value)
        rows.value = data.map(r => ({ ...r, quantity: r.recommended_quantity }))
      } catch (err) {
        error.value = 'Failed to load recommendations: ' + (err.response?.data?.detail || err.message)
        rows.value = []
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      const payloadItems = rows.value
        .filter(r => (Number(r.quantity) || 0) > 0)
        .map(r => ({ item_sku: r.item_sku, quantity: Number(r.quantity) }))

      if (payloadItems.length === 0) return

      try {
        submitting.value = true
        banner.value = null
        const result = await api.submitRestockingOrder({
          budget: budget.value,
          items: payloadItems
        })
        banner.value = {
          type: 'success',
          message: `Order ${result.order_number} submitted — expected delivery ${result.expected_delivery}.`
        }
        // Reset quantities so the user doesn't accidentally re-submit
        rows.value = rows.value.map(r => ({ ...r, quantity: 0 }))
      } catch (err) {
        banner.value = {
          type: 'error',
          message: 'Failed to submit order: ' + (err.response?.data?.detail || err.message)
        }
      } finally {
        submitting.value = false
      }
    }

    // Debounce slider changes to avoid spamming the API while dragging
    watch(budget, () => {
      if (debounceHandle) clearTimeout(debounceHandle)
      debounceHandle = setTimeout(loadRecommendations, DEBOUNCE_MS)
    })

    onMounted(loadRecommendations)

    return {
      MIN_BUDGET,
      MAX_BUDGET,
      STEP_BUDGET,
      budget,
      rows,
      loading,
      error,
      submitting,
      banner,
      totalUnits,
      totalCost,
      budgetRemaining,
      canSubmit,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  border-bottom: none;
  margin-bottom: 0.5rem;
  padding-bottom: 0;
}

.budget-value {
  font-size: 1.75rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding-top: 0.5rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 0 0 1px #2563eb;
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 0 0 1px #2563eb;
}

.slider-bound {
  font-size: 0.813rem;
  color: #64748b;
  min-width: 70px;
}

.slider-bound:last-of-type {
  text-align: right;
}

.recs-table {
  width: 100%;
}

.recs-table th.num,
.recs-table td.num {
  text-align: right;
}

.qty-input {
  width: 90px;
  padding: 0.375rem 0.5rem;
  font-size: 0.875rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  text-align: right;
  font-family: inherit;
}

.qty-input:focus {
  outline: none;
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.15);
}

.empty-state {
  text-align: center;
  padding: 2.5rem 1rem;
  color: #64748b;
  font-size: 0.938rem;
}

.summary-card {
  background: #f8fafc;
}

.summary-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr) auto;
  align-items: end;
  gap: 1.5rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.375rem;
}

.summary-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.summary-value.negative {
  color: #dc2626;
}

.summary-action {
  display: flex;
  align-items: center;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.938rem;
  border: none;
  cursor: pointer;
  transition: background-color 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
}

.over-budget-note {
  margin-top: 0.75rem;
  font-size: 0.875rem;
  color: #dc2626;
  text-align: right;
}

.banner {
  padding: 0.875rem 1rem;
  border-radius: 8px;
  font-size: 0.938rem;
  margin-bottom: 1.25rem;
  border: 1px solid transparent;
}

.banner.success {
  background: #ecfdf5;
  border-color: #a7f3d0;
  color: #065f46;
}

.banner.error {
  background: #fef2f2;
  border-color: #fecaca;
  color: #991b1b;
}

@media (max-width: 900px) {
  .summary-grid {
    grid-template-columns: 1fr 1fr;
  }
  .summary-action {
    grid-column: span 2;
    justify-content: flex-end;
  }
}
</style>
