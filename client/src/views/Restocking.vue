<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Review demand forecasts and place restocking orders within your available budget.</p>
    </div>

    <div v-if="loading" class="loading">Loading restocking data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <div class="card budget-card">
        <div class="budget-label">Available Budget: {{ formatCurrency(budget) }}</div>
        <input
          type="range"
          class="budget-slider"
          :min="0"
          :max="100000"
          :step="1000"
          v-model.number="budget"
        />
        <div class="budget-range-labels">
          <span>$0</span>
          <span>$100,000</span>
        </div>
      </div>

      <div v-if="successMessage" class="success-banner">
        {{ successMessage }}
      </div>
      <div v-if="orderError" class="error">{{ orderError }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Restocking Items ({{ recommendedItems.length }})</h3>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-message">
          No items fit within your budget. Try increasing the budget.
        </div>

        <div v-else class="table-container">
          <table class="restocking-table">
            <thead>
              <tr>
                <th>Item Name</th>
                <th>SKU</th>
                <th>Demand Gap</th>
                <th>Order Qty</th>
                <th>Unit Cost</th>
                <th>Item Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.forecast_id">
                <td>{{ item.item_name }}</td>
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td class="demand-gap">+{{ item.demand_gap }} units</td>
                <td>{{ item.order_quantity }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.item_cost) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="footer-row">
                <td colspan="4"></td>
                <td class="footer-label">Total Cost:</td>
                <td class="footer-value"><strong>{{ formatCurrency(totalCost) }}</strong></td>
              </tr>
              <tr class="footer-row">
                <td colspan="4"></td>
                <td class="footer-label">Remaining Budget:</td>
                <td class="footer-value"><strong>{{ formatCurrency(remainingBudget) }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>

        <div class="order-action">
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting || submitted"
            @click="placeOrder"
          >
            <span v-if="submitting">Placing Order...</span>
            <span v-else-if="submitted">Order Placed</span>
            <span v-else>Place Restocking Order</span>
          </button>
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
    const { currentLocale } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const orderError = ref(null)
    const successMessage = ref(null)
    const submitting = ref(false)
    const submitted = ref(false)

    const budget = ref(50000)

    const forecasts = ref([])
    const inventory = ref([])

    const formatCurrency = (value) => {
      return new Intl.NumberFormat('en-US', {
        style: 'currency',
        currency: 'USD',
        minimumFractionDigits: 0,
        maximumFractionDigits: 2
      }).format(value)
    }

    // Join forecasts with inventory by SKU, compute gaps, filter and sort
    const candidateItems = computed(() => {
      const inventoryMap = {}
      for (const item of inventory.value) {
        inventoryMap[item.sku] = item
      }

      const items = []
      for (const forecast of forecasts.value) {
        const inv = inventoryMap[forecast.item_sku]
        if (!inv) continue

        const demand_gap = forecast.forecasted_demand - forecast.current_demand
        if (demand_gap <= 0) continue

        const order_quantity = inv.reorder_point
        const item_cost = order_quantity * inv.unit_cost

        items.push({
          forecast_id: forecast.id,
          item_sku: forecast.item_sku,
          item_name: forecast.item_name,
          demand_gap,
          order_quantity,
          unit_cost: inv.unit_cost,
          item_cost
        })
      }

      // Sort by demand_gap descending
      items.sort((a, b) => b.demand_gap - a.demand_gap)
      return items
    })

    // Greedy budget allocation — recomputes when budget changes
    const recommendedItems = computed(() => {
      let running_total = 0
      const result = []
      for (const item of candidateItems.value) {
        if (running_total + item.item_cost <= budget.value) {
          result.push(item)
          running_total += item.item_cost
        }
      }
      return result
    })

    const totalCost = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - totalCost.value
    })

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || submitting.value || submitted.value) return

      orderError.value = null
      submitting.value = true

      try {
        const payload = {
          items: recommendedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.order_quantity,
            unit_cost: item.unit_cost,
            total_cost: item.item_cost
          })),
          total_value: totalCost.value,
          budget: budget.value
        }

        const response = await api.placeRestockingOrder(payload)
        submitted.value = true
        successMessage.value = `Order placed successfully! Order #${response.order_number} submitted. Expected delivery in 7 days.`
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(async () => {
      try {
        const [forecastData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        forecasts.value = forecastData
        inventory.value = inventoryData
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
      } finally {
        loading.value = false
      }
    })

    return {
      loading,
      error,
      orderError,
      successMessage,
      submitting,
      submitted,
      budget,
      recommendedItems,
      totalCost,
      remainingBudget,
      formatCurrency,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin-bottom: 0.75rem;
}

.budget-slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  accent-color: #2563eb;
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
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.375rem;
  font-size: 0.813rem;
  color: #64748b;
}

.empty-message {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.restocking-table {
  width: 100%;
  border-collapse: collapse;
}

.sku-cell {
  font-family: monospace;
  font-size: 0.813rem;
  color: #64748b;
}

.demand-gap {
  color: #059669;
  font-weight: 600;
}

.footer-row td {
  background: #f8fafc;
  border-top: 1px solid #e2e8f0;
}

.footer-label {
  text-align: right;
  font-weight: 600;
  color: #475569;
  font-size: 0.875rem;
}

.footer-value {
  color: #0f172a;
  font-size: 0.875rem;
}

.order-action {
  display: flex;
  justify-content: flex-end;
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 500;
  margin-bottom: 1.25rem;
}
</style>
