<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div class="budget-control">
          <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          <input
            type="range"
            class="budget-slider"
            min="10000"
            max="500000"
            step="5000"
            v-model.number="budget"
          />
          <div class="slider-labels">
            <span>{{ currencySymbol }}10K</span>
            <span>{{ currencySymbol }}500K</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Table Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendationsTitle') }} ({{ recommendations.length }})</h3>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noRecommendations') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.onHand') }}</th>
                <th>{{ t('restocking.table.forecast') }}</th>
                <th>{{ t('restocking.table.restockQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="rec in recommendations"
                :key="rec.sku"
                :class="{ 'row-critical': rec.is_critical }"
              >
                <td><strong>{{ rec.sku }}</strong></td>
                <td>
                  {{ rec.name }}
                  <span v-if="rec.is_critical" class="badge danger">{{ t('restocking.critical') }}</span>
                </td>
                <td>{{ rec.quantity_on_hand.toLocaleString() }}</td>
                <td>{{ rec.forecasted_demand.toLocaleString() }}</td>
                <td><strong>{{ rec.recommended_quantity.toLocaleString() }}</strong></td>
                <td>{{ currencySymbol }}{{ rec.unit_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
                <td>{{ currencySymbol }}{{ rec.line_total.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="summary-row">
                <td colspan="6">{{ t('restocking.totalCost') }}</td>
                <td>{{ currencySymbol }}{{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
              </tr>
              <tr class="summary-row">
                <td colspan="6">{{ t('restocking.budgetRemaining') }}</td>
                <td>{{ currencySymbol }}{{ budgetRemaining.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</td>
              </tr>
            </tfoot>
          </table>
        </div>

        <div class="order-actions">
          <div v-if="successMessage" class="success-message">{{ successMessage }}</div>
          <button
            class="place-order-btn"
            :disabled="recommendations.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, computed } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const budget = ref(100000)
    const recommendations = ref([])
    const totalCost = ref(0)
    const budgetRemaining = ref(0)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const successMessage = ref('')

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockRecommendations(budget.value)
        recommendations.value = data.recommendations
        totalCost.value = data.total_cost
        budgetRemaining.value = data.budget_remaining
      } catch (err) {
        error.value = 'Failed to load restock recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    let debounceTimer = null
    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    const placeOrder = async () => {
      submitting.value = true
      try {
        const items = recommendations.value.map(rec => ({
          sku: rec.sku,
          name: rec.name,
          quantity: rec.recommended_quantity,
          unit_cost: rec.unit_cost,
          line_total: rec.line_total
        }))
        const result = await api.createRestockOrder({
          items,
          total_cost: totalCost.value,
          budget: budget.value
        })
        successMessage.value = t('restocking.orderSuccess', { orderNumber: result.order_number })
        setTimeout(() => {
          successMessage.value = ''
        }, 5000)
        await loadRecommendations()
      } catch (err) {
        error.value = 'Failed to place restock order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      t,
      currencySymbol,
      budget,
      recommendations,
      totalCost,
      budgetRemaining,
      loading,
      error,
      submitting,
      successMessage,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

.page-header {
  margin-bottom: 2rem;
}

.page-header h2 {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  margin: 0 0 0.5rem 0;
}

.page-header p {
  color: #64748b;
  margin: 0;
  font-size: 0.875rem;
}

.card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.05);
}

.card-header {
  margin-bottom: 1.25rem;
  padding-bottom: 1rem;
  border-bottom: 1px solid #f1f5f9;
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: #0f172a;
  margin: 0;
}

.budget-control {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 600;
  color: #0f172a;
  letter-spacing: -0.02em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #0f172a;
  cursor: pointer;
  border-radius: 3px;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 500;
}

.loading {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}

.error {
  padding: 1rem 1.5rem;
  background: #fef2f2;
  border: 1px solid #fca5a5;
  border-radius: 8px;
  color: #dc2626;
  font-size: 0.875rem;
}

.empty-state {
  padding: 3rem;
  text-align: center;
  color: #64748b;
  font-size: 0.875rem;
}

.table-container {
  overflow-x: auto;
}

table {
  width: 100%;
  border-collapse: collapse;
  font-size: 0.875rem;
}

thead th {
  text-align: left;
  padding: 0.75rem 1rem;
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  background: #f8fafc;
  border-bottom: 1px solid #e2e8f0;
  white-space: nowrap;
}

tbody td {
  padding: 0.875rem 1rem;
  border-bottom: 1px solid #f1f5f9;
  color: #0f172a;
  vertical-align: middle;
}

tbody tr:last-child td {
  border-bottom: none;
}

tbody tr:hover {
  background: #f8fafc;
}

.row-critical {
  border-left: 3px solid #ef4444;
}

.badge {
  display: inline-block;
  padding: 0.2rem 0.5rem;
  border-radius: 4px;
  font-size: 0.7rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-left: 0.5rem;
  vertical-align: middle;
}

.badge.danger {
  background: #fee2e2;
  color: #dc2626;
}

.summary-row td {
  padding: 0.875rem 1rem;
  font-weight: 700;
  background: #f8fafc;
  color: #0f172a;
  border-top: 2px solid #e2e8f0;
}

.order-actions {
  margin-top: 1.5rem;
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 0.75rem;
}

.success-message {
  padding: 0.875rem 1.25rem;
  background: #f0fdf4;
  border: 1px solid #10b981;
  border-radius: 8px;
  color: #065f46;
  font-size: 0.875rem;
  font-weight: 500;
  width: 100%;
  box-sizing: border-box;
}

.place-order-btn {
  background: #0f172a;
  color: white;
  border: none;
  padding: 0.75rem 1.75rem;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
  letter-spacing: 0.01em;
}

.place-order-btn:hover:not(:disabled) {
  background: #1e293b;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
  opacity: 0.7;
}
</style>
