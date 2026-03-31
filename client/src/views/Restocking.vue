<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Recommended items to restock based on current inventory levels</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- ROW 1: Controls + Summary side by side -->
      <div class="controls-row">
        <!-- LEFT: Budget + Lead Time controls -->
        <div class="card controls-card">
          <div class="card-header">
            <h3 class="card-title">Budget Controls</h3>
          </div>

          <div class="control-group">
            <div class="control-label">Budget</div>
            <!-- Large prominent display of the currently selected budget amount -->
            <div class="budget-display">${{ budget.toLocaleString() }}</div>
            <input
              type="range"
              class="budget-slider"
              :min="0"
              :max="totalRestockCost"
              :step="100"
              v-model.number="budget"
            />
            <div class="slider-bounds">
              <span>$0</span>
              <span>${{ totalRestockCost.toLocaleString() }}</span>
            </div>
          </div>

          <hr class="control-divider" />

          <div class="control-group">
            <div class="control-label">Lead Time</div>
            <div class="lead-time-row">
              <input
                type="number"
                class="lead-time-input"
                v-model.number="leadTime"
                :min="1"
              />
              <span class="lead-time-unit">days</span>
            </div>
          </div>
        </div>

        <!-- RIGHT: Order summary + Place Order button -->
        <div class="card summary-card">
          <div class="card-header">
            <h3 class="card-title">Order Summary</h3>
          </div>

          <div class="summary-stats">
            <div class="summary-stat">
              <div class="stat-label">Items Included</div>
              <div class="stat-value info-value">{{ includedItems.length }}</div>
            </div>
            <div class="summary-stat">
              <div class="stat-label">Total Cost</div>
              <div class="stat-value">${{ includedCost.toLocaleString() }}</div>
            </div>
            <div class="summary-stat">
              <div class="stat-label">Remaining Budget</div>
              <!-- Remaining budget goes negative if slider is moved below already-committed cost; show clearly -->
              <div :class="['stat-value', remainingBudget < 0 ? 'danger-value' : 'success-value']">
                ${{ remainingBudget.toLocaleString() }}
              </div>
            </div>
          </div>

          <button
            class="place-order-btn"
            :disabled="includedItems.length === 0 || orderPlaced"
            @click="placeOrder"
          >
            {{ orderPlaced ? 'Order Submitted' : 'Place Order' }}
          </button>

          <!-- Success banner shown after the order is placed; includes the generated order number -->
          <div v-if="orderPlaced" class="success-banner">
            Restock order {{ placedOrderNumber }} has been submitted. Navigate to Orders to view it.
          </div>
        </div>
      </div>

      <!-- ROW 2: Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking Items</h3>
          <span class="badge warning">{{ restockableItems.length }} items need restocking</span>
        </div>

        <div v-if="restockableItems.length === 0" class="empty-state">
          All inventory items are at or above reorder points.
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Name</th>
                <th>Category</th>
                <th class="col-num">On Hand</th>
                <th class="col-num">Reorder Pt</th>
                <th class="col-num">Restock Qty</th>
                <th class="col-num">Unit Cost</th>
                <th class="col-num">Total Cost</th>
                <th>Demand</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <!-- Included items render at full opacity; over-budget items are dimmed -->
              <tr
                v-for="item in restockableItems"
                :key="item.sku"
                :class="{ 'over-budget-row': item.overBudget }"
              >
                <td><strong>{{ item.sku }}</strong></td>
                <td>{{ item.name }}</td>
                <td>{{ item.category }}</td>
                <td class="col-num">{{ item.quantity_on_hand }}</td>
                <td class="col-num">{{ item.reorder_point }}</td>
                <td class="col-num">{{ item.restock_qty }}</td>
                <td class="col-num">${{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-num"><strong>${{ item.restock_cost.toLocaleString() }}</strong></td>
                <td>
                  <!-- No demand data for this SKU shows a neutral dash instead of a badge -->
                  <span v-if="item.trend" :class="['badge', trendClass(item.trend)]">
                    {{ item.trend }}
                  </span>
                  <span v-else class="no-demand">—</span>
                </td>
                <td>
                  <span v-if="item.overBudget" class="badge warning">Over budget</span>
                  <span v-else class="badge success">Included</span>
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
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)

    // Raw data from the API
    const inventoryItems = ref([])
    const demandForecasts = ref([])

    // User-controlled inputs
    const leadTime = ref(14)
    const budget = ref(0)          // initialized to totalRestockCost after load
    const orderPlaced = ref(false)
    const placedOrderNumber = ref('')

    // --- Data loading ---

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        // Fetch all inventory (no filters — we want the full picture for restocking)
        const [inv, demand] = await Promise.all([
          api.getInventory({}),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inv
        demandForecasts.value = demand

        // Default budget to the full cost of everything that needs restocking
        budget.value = totalRestockCost.value
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // --- Derived data ---

    // Build a Map from SKU → demand trend so we can do O(1) lookups per inventory item.
    // The demand forecast uses the field `item_sku`; inventory uses `sku`.
    const trendBySku = computed(() => {
      const map = new Map()
      for (const forecast of demandForecasts.value) {
        if (forecast.item_sku) {
          map.set(forecast.item_sku, forecast.trend)
        }
      }
      return map
    })

    // Priority order for sorting: items with increasing demand should be restocked first
    // because they are the most likely to stock out. Decreasing-demand items come last.
    const TREND_PRIORITY = { increasing: 0, stable: 1, decreasing: 2 }

    // Items whose on-hand quantity has fallen below the reorder point, enriched with
    // restock_qty, restock_cost, and the demand trend for their SKU.
    const restockableItems = computed(() => {
      const items = inventoryItems.value
        .filter(item => item.quantity_on_hand < item.reorder_point)
        .map(item => {
          const restock_qty = item.reorder_point - item.quantity_on_hand
          const restock_cost = restock_qty * item.unit_cost
          const trend = trendBySku.value.get(item.sku) || null
          return { ...item, restock_qty, restock_cost, trend }
        })

      // Sort by demand trend priority (increasing first), then by restock cost descending
      // within each priority group so the most expensive included items sit at the top.
      items.sort((a, b) => {
        const pa = a.trend ? (TREND_PRIORITY[a.trend] ?? 3) : 3
        const pb = b.trend ? (TREND_PRIORITY[b.trend] ?? 3) : 3
        if (pa !== pb) return pa - pb
        return b.restock_cost - a.restock_cost
      })

      // Greedy budget allocation: walk the priority-sorted list and include items as
      // long as they fit within the slider budget. Once the running total would exceed
      // the budget, mark the remaining items "over budget" but still show them in the
      // table so users can see what they are missing.
      let accumulated = 0
      return items.map(item => {
        if (accumulated + item.restock_cost <= budget.value) {
          accumulated += item.restock_cost
          return { ...item, overBudget: false }
        }
        return { ...item, overBudget: true }
      })
    })

    // Total cost to restock every item below its reorder point — used as slider max.
    const totalRestockCost = computed(() => {
      return inventoryItems.value
        .filter(item => item.quantity_on_hand < item.reorder_point)
        .reduce((sum, item) => {
          const restock_qty = item.reorder_point - item.quantity_on_hand
          return sum + restock_qty * item.unit_cost
        }, 0)
    })

    // Items that fit within the budget (not marked over-budget)
    const includedItems = computed(() => {
      return restockableItems.value.filter(item => !item.overBudget)
    })

    // Total cost of only the included items
    const includedCost = computed(() => {
      return includedItems.value.reduce((sum, item) => sum + item.restock_cost, 0)
    })

    // How much of the budget is still unspent after the greedy selection
    const remainingBudget = computed(() => {
      return budget.value - includedCost.value
    })

    // --- Actions ---

    const placeOrder = async () => {
      if (includedItems.value.length === 0) return

      const orderNumber = 'RST-' + Date.now()
      const orderData = {
        order_number: orderNumber,
        customer: 'Internal Restock',
        items: includedItems.value.map(item => ({
          sku: item.sku,
          name: item.name,
          quantity: item.restock_qty,
          unit_price: item.unit_cost
        })),
        status: 'Submitted',
        order_date: new Date().toISOString(),
        // Expected delivery is today + the configured lead time in days
        expected_delivery: new Date(
          Date.now() + leadTime.value * 24 * 60 * 60 * 1000
        ).toISOString(),
        total_value: includedCost.value
      }

      try {
        await api.createOrder(orderData)
        placedOrderNumber.value = orderNumber
        orderPlaced.value = true
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
        console.error(err)
      }
    }

    // --- Helpers ---

    // Map demand trend strings to their badge CSS class names (defined globally in App.vue)
    const trendClass = (trend) => {
      const map = {
        increasing: 'increasing',
        stable: 'stable',
        decreasing: 'decreasing'
      }
      return map[trend] || 'info'
    }

    onMounted(loadData)

    return {
      loading,
      error,
      budget,
      leadTime,
      orderPlaced,
      placedOrderNumber,
      restockableItems,
      totalRestockCost,
      includedItems,
      includedCost,
      remainingBudget,
      placeOrder,
      trendClass
    }
  }
}
</script>

<style scoped>
/* Two-column layout for the controls row */
.controls-row {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 1.25rem;
  margin-bottom: 1.25rem;
}

.controls-card,
.summary-card {
  margin-bottom: 0; /* gap handles spacing in grid */
}

/* Budget + lead time control groups */
.control-group {
  margin-bottom: 1rem;
}

.control-group:last-child {
  margin-bottom: 0;
}

.control-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.5rem;
}

/* Large, prominent budget value display */
.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 0.75rem;
}

/* Range slider styled to match slate/blue palette */
.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0,0,0,0.2);
}

.slider-bounds {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 0.375rem;
}

.control-divider {
  border: none;
  border-top: 1px solid #e2e8f0;
  margin: 1rem 0;
}

.lead-time-row {
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.lead-time-input {
  width: 80px;
  padding: 0.375rem 0.625rem;
  border: 1px solid #e2e8f0;
  border-radius: 6px;
  font-size: 0.938rem;
  color: #0f172a;
  outline: none;
}

.lead-time-input:focus {
  border-color: #2563eb;
  box-shadow: 0 0 0 3px rgba(37, 99, 235, 0.1);
}

.lead-time-unit {
  font-size: 0.938rem;
  color: #64748b;
}

/* Order summary card stat display */
.summary-stats {
  display: flex;
  flex-direction: column;
  gap: 0.875rem;
  margin-bottom: 1.25rem;
}

.summary-stat .stat-label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
  margin-bottom: 0.25rem;
}

.summary-stat .stat-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.info-value { color: #2563eb; }
.success-value { color: #059669; }
.danger-value { color: #dc2626; }

/* Place Order button */
.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 1.5rem;
  border-radius: 8px;
  font-weight: 600;
  width: 100%;
  border: none;
  cursor: pointer;
  font-size: 1rem;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Green success banner shown after order submission */
.success-banner {
  margin-top: 0.75rem;
  padding: 0.75rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  color: #065f46;
  font-size: 0.875rem;
  font-weight: 500;
}

/* Restocking recommendations table */
.restock-table {
  width: 100%;
}

.col-num {
  text-align: right;
}

/* Over-budget rows are dimmed to convey that they won't be included in the order */
.over-budget-row {
  opacity: 0.5;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.no-demand {
  color: #94a3b8;
  font-size: 0.875rem;
  font-weight: 500;
}
</style>
