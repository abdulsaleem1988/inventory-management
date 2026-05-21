<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.subtitle') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="loadError" class="error">{{ loadError }}</div>
    <div v-else>
      <div class="card budget-section">
        <div class="budget-label">
          <label class="field-label">{{ t('restocking.budget') }}</label>
          <div class="budget-summary">
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
            <span class="budget-items">{{ recommendations.length }} {{ t('restocking.recommended') }}</span>
          </div>
        </div>
        <input
          type="range"
          class="budget-slider"
          min="10000"
          max="500000"
          step="5000"
          v-model.number="budget"
        />
        <div class="slider-bounds">
          <span>{{ formatCurrency(10000) }}</span>
          <span>{{ formatCurrency(500000) }}</span>
        </div>
      </div>

      <div v-if="submitted" class="success-banner">
        {{ t('restocking.orderSuccess') }} —
        <router-link to="/orders" class="success-link">View Orders</router-link>
      </div>

      <div v-if="error" class="error-banner">{{ error }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommended') }}</h3>
          <span v-if="recommendations.length > 0" class="total-cost-header">
            {{ t('restocking.totalCost') }}: <strong>{{ formatCurrency(totalCost) }}</strong>
          </span>
        </div>

        <div v-if="recommendations.length === 0">
          <p class="no-items">{{ t('restocking.noItems') }}</p>
        </div>
        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>Item</th>
                <th>SKU</th>
                <th>{{ t('restocking.trend') }}</th>
                <th class="col-num">{{ t('restocking.qty') }}</th>
                <th class="col-num">{{ t('restocking.unitCost') }}</th>
                <th class="col-num">Item Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.sku">
                <td>{{ item.item_name }}</td>
                <td><strong>{{ item.sku }}</strong></td>
                <td>
                  <span :class="['badge', getTrendClass(item.trend)]">
                    {{ t(`trends.${item.trend}`) }}
                  </span>
                </td>
                <td class="col-num">{{ item.qty }}</td>
                <td class="col-num">{{ formatCurrency(item.unit_cost) }}</td>
                <td class="col-num"><strong>{{ formatCurrency(item.item_total) }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="total-row">
                <td colspan="5" class="total-label">Total</td>
                <td class="col-num total-value">
                  <strong>{{ formatCurrency(totalCost) }}</strong>
                </td>
              </tr>
            </tfoot>
          </table>
        </div>

        <div class="card-footer">
          <button
            class="btn-primary"
            :disabled="recommendations.length === 0 || submitting || submitted"
            @click="placeOrder"
          >
            <span v-if="submitting">Submitting...</span>
            <span v-else>{{ t('restocking.placeOrder') }}</span>
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()
    const { selectedLocation, getCurrentFilters } = useFilters()

    const budget = ref(100000)
    const demandForecasts = ref([])
    const inventoryMap = ref({})
    const submitting = ref(false)
    const submitted = ref(false)
    const error = ref(null)
    const loading = ref(false)
    const loadError = ref(null)

    const formatCurrency = (value) => {
      if (currentCurrency.value === 'JPY') {
        return '¥' + Math.round(value).toLocaleString()
      }
      return '$' + Number(value).toLocaleString('en-US', {
        minimumFractionDigits: 0,
        maximumFractionDigits: 2
      })
    }

    const recommendations = computed(() => {
      let forecasts = demandForecasts.value

      if (selectedLocation.value !== 'all') {
        forecasts = forecasts.filter(f => {
          const inv = inventoryMap.value[f.item_sku]
          return inv && inv.warehouse === selectedLocation.value
        })
      }

      // Sort: increasing trend first, then by forecasted_demand descending
      const sorted = [...forecasts].sort((a, b) => {
        if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
        if (b.trend === 'increasing' && a.trend !== 'increasing') return 1
        return b.forecasted_demand - a.forecasted_demand
      })

      const result = []
      let running_total = 0

      for (const forecast of sorted) {
        const inv = inventoryMap.value[forecast.item_sku]
        if (!inv || inv.unit_cost == null) continue

        const unit_cost = inv.unit_cost
        const item_cost = forecast.forecasted_demand * unit_cost

        if (running_total + item_cost <= budget.value) {
          running_total += item_cost
          result.push({
            sku: forecast.item_sku,
            item_name: forecast.item_name,
            qty: forecast.forecasted_demand,
            unit_cost,
            item_total: item_cost,
            trend: forecast.trend
          })
        }
      }

      return result
    })

    const totalCost = computed(() => {
      return recommendations.value.reduce((sum, item) => sum + item.item_total, 0)
    })

    const loadData = async () => {
      loading.value = true
      loadError.value = null
      try {
        const filters = getCurrentFilters()

        const [forecastsData, inventoryData] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({
            warehouse: filters.warehouse,
            category: filters.category
          })
        ])

        demandForecasts.value = forecastsData

        const map = {}
        for (const item of inventoryData) {
          if (item.sku) {
            map[item.sku] = {
              unit_cost: item.unit_cost,
              warehouse: item.warehouse
            }
          }
        }
        inventoryMap.value = map
      } catch (err) {
        loadError.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value || submitted.value) return

      submitting.value = true
      error.value = null

      try {
        const warehouse = selectedLocation.value !== 'all'
          ? selectedLocation.value
          : 'San Francisco'

        await api.submitRestockOrder({
          items: recommendations.value.map(item => ({
            sku: item.sku,
            item_name: item.item_name,
            qty: item.qty,
            unit_cost: item.unit_cost,
            item_total: item.item_total
          })),
          warehouse,
          total_value: totalCost.value
        })

        submitted.value = true
      } catch (err) {
        error.value = 'Failed to submit order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const getTrendClass = (trend) => {
      const map = {
        increasing: 'increasing',
        stable: 'stable',
        decreasing: 'decreasing'
      }
      return map[trend] || 'stable'
    }

    watch(selectedLocation, loadData)

    onMounted(loadData)

    return {
      t,
      budget,
      demandForecasts,
      inventoryMap,
      submitting,
      submitted,
      error,
      loading,
      loadError,
      recommendations,
      totalCost,
      loadData,
      placeOrder,
      formatCurrency,
      getTrendClass
    }
  }
}
</script>

<style scoped>
.budget-section {
  margin-bottom: 1.25rem;
}

.budget-label {
  display: flex;
  justify-content: space-between;
  align-items: baseline;
  margin-bottom: 0.875rem;
}

.field-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #475569;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-summary {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-items {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  display: block;
  margin-bottom: 0.5rem;
}

.slider-bounds {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.success-link {
  color: #059669;
  font-weight: 600;
  text-decoration: underline;
}

.success-link:hover {
  color: #047857;
}

.error-banner {
  background: #fef2f2;
  border: 1px solid #fecaca;
  color: #991b1b;
  padding: 0.875rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
}

.no-items {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}

.restock-table {
  width: 100%;
}

.col-num {
  text-align: right;
}

.total-row {
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.total-label {
  text-align: right;
  font-weight: 600;
  color: #475569;
  font-size: 0.875rem;
  padding: 0.75rem;
}

.total-value {
  font-size: 1rem;
  color: #0f172a;
  padding: 0.75rem;
}

.total-cost-header {
  font-size: 0.938rem;
  color: #475569;
}

.total-cost-header strong {
  color: #0f172a;
}

.card-footer {
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 1.25rem;
  display: flex;
  justify-content: flex-end;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
