<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div class="card">
        <div class="budget-control">
          <label class="budget-label">{{ t('restocking.budgetLabel') }}</label>
          <div class="budget-slider-row">
            <input
              type="range"
              class="budget-slider"
              v-model.number="budget"
              min="0"
              :max="maxBudget"
              step="100"
            />
            <span class="budget-value">{{ formatCurrency(budget, currentCurrency) }}</span>
          </div>
        </div>

        <div class="stats-grid">
          <div class="stat-card warning">
            <div class="stat-label">{{ t('restocking.budgetUsed') }}</div>
            <div class="stat-value">{{ formatCurrency(budgetUsed, currentCurrency) }}</div>
          </div>
          <div class="stat-card success">
            <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
            <div class="stat-value">{{ formatCurrency(budgetRemaining, currentCurrency) }}</div>
          </div>
          <div class="stat-card info">
            <div class="stat-label">{{ t('restocking.itemsRecommended') }}</div>
            <div class="stat-value">{{ recommendedItems.length }}</div>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }} ({{ recommendedItems.length }})</h3>
        </div>
        <div class="table-container">
          <table v-if="recommendedItems.length > 0">
            <thead>
              <tr>
                <th>{{ t('demand.table.sku') }}</th>
                <th>{{ t('demand.table.itemName') }}</th>
                <th>{{ t('demand.table.trend') }}</th>
                <th>{{ t('restocking.suggestedQuantity') }}</th>
                <th>{{ t('restocking.unitCost') }}</th>
                <th>{{ t('restocking.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendedItems" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">
                    {{ t(`trends.${item.trend}`) }}
                  </span>
                </td>
                <td>{{ item.suggestedQuantity }}</td>
                <td>{{ formatCurrency(item.unit_cost, currentCurrency) }}</td>
                <td><strong>{{ formatCurrency(item.lineTotal, currentCurrency) }}</strong></td>
              </tr>
            </tbody>
          </table>
          <div v-else class="no-items">{{ t('restocking.noItemsRecommended') }}</div>
        </div>

        <div class="place-order-row">
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || placingOrder"
            @click="placeOrder"
          >
            {{ t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="orderSuccess" class="success-message">{{ t('restocking.orderPlaced') }}</div>
        <div v-if="orderError" class="error">{{ orderError }}</div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(0)
    const placingOrder = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)

    const trendScoreMap = {
      increasing: 1.0,
      stable: 0.4,
      decreasing: 0.0
    }

    const maxBudget = computed(() => {
      const total = forecasts.value.reduce((sum, item) => {
        return sum + (item.reorder_point * item.unit_cost)
      }, 0)
      // Round up to a clean number (nearest 1000)
      return Math.max(Math.ceil(total / 1000) * 1000, 1000)
    })

    const rankedForecasts = computed(() => {
      const scored = forecasts.value.map(item => {
        const stockGapRatio = Math.min(
          Math.max(item.reorder_point - item.quantity_on_hand, 0) / item.reorder_point,
          1
        )
        const trendScore = trendScoreMap[item.trend] ?? 0.4
        const demandGrowthRatio = Math.min(
          Math.max(item.forecasted_demand - item.current_demand, 0) / item.current_demand,
          1
        )
        const urgencyScore = 0.5 * stockGapRatio + 0.3 * trendScore + 0.2 * demandGrowthRatio
        const gapQuantity = Math.max(item.reorder_point - item.quantity_on_hand, 0)
        const suggestedQuantity = gapQuantity > 0 ? gapQuantity : 10

        return {
          ...item,
          stockGapRatio,
          urgencyScore,
          suggestedQuantity
        }
      })

      return scored.sort((a, b) => {
        if (b.urgencyScore !== a.urgencyScore) return b.urgencyScore - a.urgencyScore
        if (b.stockGapRatio !== a.stockGapRatio) return b.stockGapRatio - a.stockGapRatio
        return a.item_sku.localeCompare(b.item_sku)
      })
    })

    const recommendedItems = computed(() => {
      let remainingBudget = budget.value
      const selected = []

      for (const item of rankedForecasts.value) {
        const lineTotal = item.suggestedQuantity * item.unit_cost
        if (lineTotal <= remainingBudget) {
          remainingBudget -= lineTotal
          selected.push({ ...item, lineTotal })
        }
      }

      return selected
    })

    const budgetUsed = computed(() => {
      return recommendedItems.value.reduce((sum, item) => sum + item.lineTotal, 0)
    })

    const budgetRemaining = computed(() => {
      return Math.max(budget.value - budgetUsed.value, 0)
    })

    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      placingOrder.value = true
      orderSuccess.value = false
      orderError.value = null
      try {
        await api.createRestockingOrder({
          budget: budget.value,
          items: recommendedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.suggestedQuantity,
            unit_cost: i.unit_cost
          }))
        })
        orderSuccess.value = true
      } catch (err) {
        orderError.value = t('restocking.orderFailed')
        console.error(err)
      } finally {
        placingOrder.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      currentCurrency,
      loading,
      error,
      budget,
      maxBudget,
      recommendedItems,
      budgetUsed,
      budgetRemaining,
      placingOrder,
      orderSuccess,
      orderError,
      placeOrder,
      formatCurrency
    }
  }
}
</script>

<style scoped>
.budget-control {
  margin-bottom: 1.25rem;
}

.budget-label {
  display: block;
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  margin-bottom: 0.5rem;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-slider {
  flex: 1;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  accent-color: #3b82f6;
}

.budget-slider:focus {
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.budget-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
  min-width: 120px;
  text-align: right;
}

.no-items {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

.place-order-row {
  margin-top: 1.25rem;
  padding-top: 1rem;
  border-top: 1px solid #e2e8f0;
  display: flex;
  justify-content: flex-end;
}

.place-order-btn {
  padding: 0.625rem 1.5rem;
  background: #0f172a;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1e293b;
}

.place-order-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.success-message {
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  color: #166534;
  padding: 1rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
}
</style>
