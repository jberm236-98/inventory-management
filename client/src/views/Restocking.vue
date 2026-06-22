<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget and review demand-forecast-based restocking recommendations.</p>
    </div>

    <div v-if="loading" class="loading">Loading restocking data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <!-- Success banner -->
      <div v-if="successOrder" class="success-banner">
        Order <strong>{{ successOrder.order_number }}</strong> placed successfully.
        <router-link to="/orders" class="view-orders-link">View in Orders tab</router-link>
      </div>

      <!-- Budget + lead time controls -->
      <div class="card controls-card">
        <div class="card-header">
          <h3 class="card-title">Order Parameters</h3>
        </div>
        <div class="controls-body">
          <div class="budget-section">
            <label class="control-label" for="budget-slider">
              Budget
              <span class="budget-display">${{ budget.toLocaleString() }}</span>
            </label>
            <input
              id="budget-slider"
              v-model.number="budget"
              type="range"
              min="0"
              max="1000000"
              step="1000"
              class="budget-slider"
            />
            <div class="budget-summary">
              ${{ budgetUsed.toLocaleString() }} used of ${{ budget.toLocaleString() }} budget
              ({{ selectedItems.length }} item{{ selectedItems.length !== 1 ? 's' : '' }} selected)
            </div>
          </div>

          <div class="lead-time-section">
            <label class="control-label" for="lead-time">
              Delivery lead time (days)
            </label>
            <input
              id="lead-time"
              v-model.number="leadTimeDays"
              type="number"
              min="1"
              class="lead-time-input"
            />
          </div>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            Recommendations ({{ recommendations.length }} items)
          </h3>
        </div>
        <div class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item Name</th>
                <th>Trend</th>
                <th class="col-num">Forecasted Qty</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Total Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-over-budget': !item.selected }"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-num">{{ item.forecasted_demand }}</td>
                <td class="col-num">
                  <span v-if="item.unit_cost > 0">${{ item.unit_cost.toFixed(2) }}</span>
                  <span v-else class="no-cost">No cost data</span>
                </td>
                <td class="col-num">
                  <span v-if="item.unit_cost > 0">
                    ${{ (item.forecasted_demand * item.unit_cost).toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
                  </span>
                  <span v-else class="no-cost">—</span>
                </td>
                <td>
                  <span v-if="item.selected" class="badge success">Selected</span>
                  <span v-else class="badge over-budget">Over budget</span>
                </td>
              </tr>
              <tr v-if="recommendations.length === 0">
                <td colspan="7" class="empty-row">No demand forecasts available.</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order button -->
      <div class="order-actions">
        <button
          class="btn-primary"
          :disabled="selectedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Placing order...' : 'Place Order' }}
        </button>
        <span v-if="submitError" class="submit-error">{{ submitError }}</span>
      </div>

    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { api } from '../api'

// Trend priority for greedy sort: increasing items are highest priority
const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

export default {
  name: 'Restocking',
  setup() {
    const router = useRouter()

    const loading = ref(true)
    const error = ref(null)
    const demandForecasts = ref([])
    const inventoryMap = ref({})

    const budget = ref(200000)
    const leadTimeDays = ref(14)

    const submitting = ref(false)
    const submitError = ref(null)
    const successOrder = ref(null)

    // Build enriched demand items with unit_cost from inventory, sorted by trend priority.
    // Items not found in inventory get unit_cost = 0 and are excluded from budget calculation.
    const enrichedItems = computed(() => {
      return demandForecasts.value
        .map(d => {
          const invItem = inventoryMap.value[d.item_sku]
          return {
            id: d.id,
            item_sku: d.item_sku,
            item_name: d.item_name,
            forecasted_demand: d.forecasted_demand,
            trend: d.trend,
            unit_cost: invItem ? invItem.unit_cost : 0
          }
        })
        .slice()
        .sort((a, b) => {
          const pa = TREND_PRIORITY[a.trend] ?? 1
          const pb = TREND_PRIORITY[b.trend] ?? 1
          return pa - pb
        })
    })

    // Greedy allocation: walk sorted list and include items whose total cost fits the
    // remaining budget. Items with no cost data (unit_cost = 0) are excluded from budget
    // math but still shown with "over budget" status since we can't evaluate their cost.
    const recommendations = computed(() => {
      let remaining = budget.value

      return enrichedItems.value.map(item => {
        const totalCost = item.forecasted_demand * item.unit_cost
        // Only select if we have cost data and it fits within remaining budget
        const fits = item.unit_cost > 0 && totalCost <= remaining
        if (fits) {
          remaining -= totalCost
        }
        return { ...item, selected: fits }
      })
    })

    const selectedItems = computed(() => recommendations.value.filter(i => i.selected))

    const budgetUsed = computed(() =>
      selectedItems.value.reduce((sum, i) => sum + i.forecasted_demand * i.unit_cost, 0)
    )

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory()
        ])
        demandForecasts.value = forecasts

        // Build a SKU → inventory item lookup for O(1) cross-reference
        const map = {}
        for (const item of inventory) {
          map[item.sku] = item
        }
        inventoryMap.value = map
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return

      submitting.value = true
      submitError.value = null
      successOrder.value = null

      try {
        const payload = {
          items: selectedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.forecasted_demand,
            unit_cost: i.unit_cost
          })),
          warehouse: 'all',
          lead_time_days: leadTimeDays.value,
          notes: null
        }

        const order = await api.submitRestockingOrder(payload)
        successOrder.value = order

        // Scroll to top so the success banner is visible
        window.scrollTo({ top: 0, behavior: 'smooth' })
      } catch (err) {
        submitError.value = 'Failed to place order: ' + (err.response?.data?.detail || err.message)
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      leadTimeDays,
      recommendations,
      selectedItems,
      budgetUsed,
      submitting,
      submitError,
      successOrder,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* page wrapper — inherits main-content padding from App.vue */
}

/* ── Success banner ── */
.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.view-orders-link {
  color: #065f46;
  font-weight: 600;
  text-decoration: underline;
  margin-left: auto;
}

.view-orders-link:hover {
  color: #047857;
}

/* ── Controls card ── */
.controls-card {
  margin-bottom: 1.25rem;
}

.controls-body {
  display: grid;
  grid-template-columns: 1fr auto;
  gap: 2rem;
  align-items: start;
  padding: 0.25rem 0;
}

.budget-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.lead-time-section {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  min-width: 220px;
}

.control-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  display: flex;
  align-items: center;
  gap: 0.75rem;
}

.budget-display {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  cursor: pointer;
}

.budget-summary {
  font-size: 0.813rem;
  color: #64748b;
}

.lead-time-input {
  width: 100%;
  padding: 0.5rem 0.75rem;
  border: 1px solid #cbd5e1;
  border-radius: 8px;
  font-size: 0.875rem;
  color: #0f172a;
  background: #f8fafc;
  transition: all 0.2s;
}

.lead-time-input:focus {
  outline: none;
  border-color: #3b82f6;
  background: white;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

/* ── Recommendations table ── */
.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.col-num {
  text-align: right;
  width: 130px;
}

/* Greyed-out rows for items that don't fit the budget */
.row-over-budget td {
  color: #94a3b8;
}

.row-over-budget td strong {
  color: #94a3b8;
}

.row-over-budget .badge {
  opacity: 0.7;
}

.no-cost {
  color: #94a3b8;
  font-style: italic;
  font-size: 0.813rem;
}

.empty-row {
  text-align: center;
  color: #64748b;
  padding: 2rem;
}

/* "Over budget" status badge — uses a neutral slate style */
.badge.over-budget {
  background: #f1f5f9;
  color: #94a3b8;
}

/* ── Order actions ── */
.order-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-top: 0.5rem;
}

.btn-primary {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #93c5fd;
  cursor: not-allowed;
}

.submit-error {
  font-size: 0.875rem;
  color: #991b1b;
}
</style>
