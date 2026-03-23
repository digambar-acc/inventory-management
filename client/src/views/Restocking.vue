<template>
  <div class="restocking-view">
    <h1>{{ t('restocking.title') }}</h1>
    <p class="subtitle">{{ t('restocking.subtitle') }}</p>

    <!-- Budget Slider Section -->
    <div class="budget-section">
      <div class="budget-header">
        <h2>{{ t('restocking.setBudget') }}</h2>
        <div class="budget-display">{{ formatCurrency(budget) }}</div>
      </div>

      <div class="slider-container">
        <input
          type="range"
          v-model.number="budget"
          :min="MIN_BUDGET"
          :max="MAX_BUDGET"
          :step="BUDGET_STEP"
          class="budget-slider"
        >
        <div class="slider-labels">
          <span>{{ formatCurrency(MIN_BUDGET) }}</span>
          <span>{{ formatCurrency(MAX_BUDGET) }}</span>
        </div>
      </div>

      <div class="budget-info">
        <div class="info-card">
          <span class="info-label">{{ t('restocking.availableBudget') }}</span>
          <span class="info-value">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="info-card">
          <span class="info-label">{{ t('restocking.allocatedBudget') }}</span>
          <span class="info-value">{{ formatCurrency(totalCost) }}</span>
        </div>
        <div class="info-card">
          <span class="info-label">{{ t('restocking.remainingBudget') }}</span>
          <span class="info-value" :class="{ 'negative': remainingBudget < 0 }">
            {{ formatCurrency(remainingBudget) }}
          </span>
        </div>
      </div>
    </div>

    <!-- Loading State -->
    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>

    <!-- Error State -->
    <div v-else-if="error" class="error">{{ error }}</div>

    <!-- Recommendations Section -->
    <div v-else class="recommendations-section">
      <div class="section-header">
        <h2>{{ t('restocking.recommendations') }}</h2>
        <button
          v-if="recommendations.length > 0"
          @click="refreshRecommendations"
          class="btn-secondary"
        >
          {{ t('restocking.refresh') }}
        </button>
      </div>

      <div v-if="recommendations.length === 0" class="empty-state">
        <p>{{ t('restocking.noRecommendations') }}</p>
      </div>

      <div v-else class="recommendations-grid">
        <div
          v-for="item in recommendations"
          :key="item.item_sku"
          class="recommendation-card"
        >
          <div class="card-header">
            <h3>{{ item.item_name }}</h3>
            <span class="trend-badge" :class="`trend-${item.trend}`">
              {{ item.trend }}
            </span>
          </div>

          <div class="card-body">
            <div class="info-row">
              <span class="label">{{ t('restocking.sku') }}:</span>
              <span class="value">{{ item.item_sku }}</span>
            </div>
            <div class="info-row">
              <span class="label">{{ t('restocking.currentStock') }}:</span>
              <span class="value">{{ item.current_stock || 0 }}</span>
            </div>
            <div class="info-row">
              <span class="label">{{ t('restocking.currentDemand') }}:</span>
              <span class="value">{{ item.current_demand }}</span>
            </div>
            <div class="info-row">
              <span class="label">{{ t('restocking.forecastedDemand') }}:</span>
              <span class="value highlight">{{ item.forecasted_demand }}</span>
            </div>
            <div class="info-row">
              <span class="label">{{ t('restocking.recommendedQuantity') }}:</span>
              <span class="value strong">{{ item.recommended_quantity }}</span>
            </div>
            <div class="info-row">
              <span class="label">{{ t('restocking.unitCost') }}:</span>
              <span class="value">{{ formatCurrency(item.unit_cost) }}</span>
            </div>
            <div class="info-row total">
              <span class="label">{{ t('restocking.totalCost') }}:</span>
              <span class="value">{{ formatCurrency(item.total_cost) }}</span>
            </div>
          </div>

          <div class="card-footer">
            <div class="quantity-control">
              <button
                @click="adjustQuantity(item, -10)"
                :disabled="item.selected_quantity <= 0"
                class="qty-btn"
              >-</button>
              <input
                type="number"
                v-model.number="item.selected_quantity"
                :min="0"
                :max="item.recommended_quantity * 2"
                class="qty-input"
              >
              <button
                @click="adjustQuantity(item, 10)"
                class="qty-btn"
              >+</button>
            </div>
            <button
              @click="toggleSelection(item)"
              class="btn-toggle"
              :class="{ 'selected': item.is_selected }"
            >
              {{ item.is_selected ? t('restocking.remove') : t('restocking.add') }}
            </button>
          </div>
        </div>
      </div>
    </div>

    <!-- Order Summary & Submit -->
    <div v-if="selectedItems.length > 0" class="order-summary">
      <h2>{{ t('restocking.orderSummary') }}</h2>

      <div class="summary-content">
        <div class="summary-items">
          <div class="summary-stat">
            <span class="stat-label">{{ t('restocking.totalItems') }}:</span>
            <span class="stat-value">{{ selectedItems.length }}</span>
          </div>
          <div class="summary-stat">
            <span class="stat-label">{{ t('restocking.totalUnits') }}:</span>
            <span class="stat-value">{{ totalUnits }}</span>
          </div>
          <div class="summary-stat">
            <span class="stat-label">{{ t('restocking.totalCost') }}:</span>
            <span class="stat-value">{{ formatCurrency(totalCost) }}</span>
          </div>
          <div class="summary-stat">
            <span class="stat-label">{{ t('restocking.estimatedDelivery') }}:</span>
            <span class="stat-value">{{ estimatedDeliveryDays }} {{ t('common.days') }}</span>
          </div>
        </div>

        <button
          @click="placeOrder"
          :disabled="remainingBudget < 0 || submitting"
          class="btn-primary btn-large"
        >
          {{ submitting ? t('restocking.submitting') : t('restocking.placeOrder') }}
        </button>
      </div>

      <div v-if="remainingBudget < 0" class="warning">
        {{ t('restocking.budgetExceeded') }}
      </div>
    </div>

    <!-- Success Modal -->
    <div v-if="showSuccessModal" class="modal active" @click="closeSuccessModal">
      <div class="modal-content" @click.stop>
        <div class="modal-header">
          <h2>{{ t('restocking.orderPlaced') }}</h2>
          <span class="modal-close" @click="closeSuccessModal">&times;</span>
        </div>
        <div class="modal-body">
          <div class="success-icon">✓</div>
          <p>{{ t('restocking.orderSuccess') }}</p>
          <div class="order-details">
            <p><strong>{{ t('restocking.orderNumber') }}:</strong> {{ lastOrderNumber }}</p>
            <p><strong>{{ t('restocking.totalCost') }}:</strong> {{ formatCurrency(lastOrderTotal) }}</p>
            <p><strong>{{ t('restocking.expectedDelivery') }}:</strong> {{ lastOrderDelivery }}</p>
          </div>
        </div>
        <div class="modal-footer">
          <button @click="viewOrders" class="btn-primary">{{ t('restocking.viewOrders') }}</button>
          <button @click="closeSuccessModal" class="btn-secondary">{{ t('common.close') }}</button>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { useRouter } from 'vue-router'
import { useI18n } from '../composables/useI18n'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const router = useRouter()
    const { t, formatCurrency } = useI18n()

    // Budget configuration
    const MIN_BUDGET = 10000
    const MAX_BUDGET = 1000000
    const BUDGET_STEP = 10000

    // State
    const budget = ref(100000)
    const loading = ref(true)
    const error = ref(null)
    const submitting = ref(false)
    const showSuccessModal = ref(false)

    const demandForecasts = ref([])
    const inventoryItems = ref([])
    const recommendations = ref([])

    // Last order details for success modal
    const lastOrderNumber = ref('')
    const lastOrderTotal = ref(0)
    const lastOrderDelivery = ref('')

    // Load data on mount
    onMounted(async () => {
      await loadData()
    })

    const loadData = async () => {
      try {
        loading.value = true
        error.value = null

        // Load demand forecasts and inventory in parallel
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])

        demandForecasts.value = forecasts
        inventoryItems.value = inventory

        // Generate recommendations based on budget
        generateRecommendations()
      } catch (err) {
        error.value = 'Failed to load data'
        console.error('Load error:', err)
      } finally {
        loading.value = false
      }
    }

    const generateRecommendations = () => {
      // Priority-based: High demand items first (increasing trend, highest forecasted demand)
      // Map forecasts to recommendations with current stock info
      const items = demandForecasts.value
        .map(forecast => {
          // Find matching inventory item to get current stock and cost
          const inventoryItem = inventoryItems.value.find(
            inv => inv.sku === forecast.item_sku
          )

          // Calculate recommended quantity based on demand gap
          const currentStock = inventoryItem?.quantity_on_hand || 0
          const demandGap = Math.max(0, forecast.forecasted_demand - currentStock)
          const recommendedQty = Math.max(demandGap, forecast.forecasted_demand * 0.5)

          // Use inventory unit cost or estimate based on forecast
          const unitCost = inventoryItem?.unit_cost || 50

          return {
            ...forecast,
            current_stock: currentStock,
            unit_cost: unitCost,
            recommended_quantity: Math.ceil(recommendedQty),
            selected_quantity: Math.ceil(recommendedQty),
            total_cost: Math.ceil(recommendedQty) * unitCost,
            is_selected: false,
            warehouse: inventoryItem?.warehouse || 'San Francisco',
            category: inventoryItem?.category || 'Unknown'
          }
        })
        // Sort by priority: increasing trend first, then by forecasted demand
        .sort((a, b) => {
          if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
          if (a.trend !== 'increasing' && b.trend === 'increasing') return 1
          return b.forecasted_demand - a.forecasted_demand
        })

      // Select items within budget automatically
      let runningTotal = 0
      const selectedRecs = []

      for (const item of items) {
        if (runningTotal + item.total_cost <= budget.value) {
          item.is_selected = true
          runningTotal += item.total_cost
          selectedRecs.push(item)
        }
      }

      recommendations.value = items
    }

    const refreshRecommendations = () => {
      // Reset selections and regenerate
      recommendations.value.forEach(item => {
        item.is_selected = false
        item.selected_quantity = item.recommended_quantity
        item.total_cost = item.selected_quantity * item.unit_cost
      })
      generateRecommendations()
    }

    const adjustQuantity = (item, delta) => {
      const newQty = item.selected_quantity + delta
      if (newQty >= 0 && newQty <= item.recommended_quantity * 2) {
        item.selected_quantity = newQty
        item.total_cost = newQty * item.unit_cost
      }
    }

    const toggleSelection = (item) => {
      item.is_selected = !item.is_selected
    }

    // Computed properties
    const selectedItems = computed(() => {
      return recommendations.value.filter(item => item.is_selected && item.selected_quantity > 0)
    })

    const totalUnits = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.selected_quantity, 0)
    })

    const totalCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - totalCost.value
    })

    const estimatedDeliveryDays = computed(() => {
      // Estimate 7-14 days based on quantity
      return totalUnits.value > 5000 ? 14 : totalUnits.value > 2000 ? 10 : 7
    })

    const placeOrder = async () => {
      if (remainingBudget.value < 0) {
        alert(t('restocking.budgetExceeded'))
        return
      }

      if (selectedItems.value.length === 0) {
        alert(t('restocking.noItemsSelected'))
        return
      }

      try {
        submitting.value = true

        // Prepare order data
        const orderData = {
          items: selectedItems.value.map(item => ({
            sku: item.item_sku,
            name: item.item_name,
            quantity: item.selected_quantity,
            unit_price: item.unit_cost
          })),
          total_value: totalCost.value,
          warehouse: selectedItems.value[0].warehouse,
          category: selectedItems.value[0].category,
          estimated_delivery_days: estimatedDeliveryDays.value
        }

        // Submit order to backend
        const result = await api.submitRestockingOrder(orderData)

        // Store last order details
        lastOrderNumber.value = result.order_number
        lastOrderTotal.value = result.total_value
        lastOrderDelivery.value = result.expected_delivery

        // Show success modal
        showSuccessModal.value = true

        // Reset selections
        recommendations.value.forEach(item => {
          item.is_selected = false
        })

      } catch (err) {
        console.error('Order submission error:', err)
        alert('Failed to submit order. Please try again.')
      } finally {
        submitting.value = false
      }
    }

    const closeSuccessModal = () => {
      showSuccessModal.value = false
    }

    const viewOrders = () => {
      closeSuccessModal()
      router.push('/orders')
    }

    return {
      MIN_BUDGET,
      MAX_BUDGET,
      BUDGET_STEP,
      budget,
      loading,
      error,
      submitting,
      recommendations,
      showSuccessModal,
      lastOrderNumber,
      lastOrderTotal,
      lastOrderDelivery,
      selectedItems,
      totalUnits,
      totalCost,
      remainingBudget,
      estimatedDeliveryDays,
      t,
      formatCurrency,
      refreshRecommendations,
      adjustQuantity,
      toggleSelection,
      placeOrder,
      closeSuccessModal,
      viewOrders
    }
  }
}
</script>

<style scoped>
.restocking-view {
  padding: 20px;
}

h1 {
  font-size: 28px;
  margin-bottom: 8px;
  color: #0f172a;
}

.subtitle {
  color: #64748b;
  margin-bottom: 30px;
}

/* Budget Section */
.budget-section {
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  margin-bottom: 30px;
}

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.budget-header h2 {
  font-size: 20px;
  color: #0f172a;
  margin: 0;
}

.budget-display {
  font-size: 32px;
  font-weight: 700;
  color: #667eea;
}

.slider-container {
  margin: 20px 0;
}

.budget-slider {
  width: 100%;
  height: 8px;
  border-radius: 5px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #667eea;
  cursor: pointer;
}

.budget-slider::-moz-range-thumb {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #667eea;
  cursor: pointer;
  border: none;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
  font-size: 14px;
  color: #64748b;
}

.budget-info {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
  gap: 15px;
  margin-top: 25px;
}

.info-card {
  background: #f8fafc;
  padding: 15px;
  border-radius: 8px;
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.info-label {
  font-size: 13px;
  color: #64748b;
}

.info-value {
  font-size: 24px;
  font-weight: 700;
  color: #0f172a;
}

.info-value.negative {
  color: #ef4444;
}

/* Recommendations Section */
.recommendations-section {
  background: white;
  padding: 30px;
  border-radius: 12px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.08);
  margin-bottom: 30px;
}

.section-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 25px;
}

.section-header h2 {
  font-size: 20px;
  color: #0f172a;
  margin: 0;
}

.recommendations-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(350px, 1fr));
  gap: 20px;
}

.recommendation-card {
  border: 2px solid #e2e8f0;
  border-radius: 12px;
  padding: 20px;
  transition: all 0.3s ease;
}

.recommendation-card:hover {
  border-color: #667eea;
  box-shadow: 0 4px 12px rgba(102, 126, 234, 0.2);
}

.card-header {
  display: flex;
  justify-content: space-between;
  align-items: start;
  margin-bottom: 15px;
  padding-bottom: 15px;
  border-bottom: 2px solid #f1f5f9;
}

.card-header h3 {
  font-size: 16px;
  color: #0f172a;
  margin: 0;
  flex: 1;
}

.trend-badge {
  padding: 4px 10px;
  border-radius: 12px;
  font-size: 12px;
  font-weight: 600;
  text-transform: capitalize;
}

.trend-increasing {
  background: #dcfce7;
  color: #15803d;
}

.trend-stable {
  background: #dbeafe;
  color: #1e40af;
}

.trend-decreasing {
  background: #fee2e2;
  color: #991b1b;
}

.card-body {
  margin-bottom: 15px;
}

.info-row {
  display: flex;
  justify-content: space-between;
  padding: 8px 0;
  font-size: 14px;
}

.info-row.total {
  border-top: 2px solid #f1f5f9;
  margin-top: 10px;
  padding-top: 12px;
  font-weight: 600;
}

.info-row .label {
  color: #64748b;
}

.info-row .value {
  color: #0f172a;
  font-weight: 500;
}

.info-row .value.highlight {
  color: #667eea;
  font-weight: 700;
}

.info-row .value.strong {
  font-weight: 700;
  font-size: 16px;
}

.card-footer {
  display: flex;
  gap: 10px;
  align-items: center;
}

.quantity-control {
  display: flex;
  align-items: center;
  gap: 8px;
  flex: 1;
}

.qty-btn {
  width: 36px;
  height: 36px;
  border: 2px solid #e2e8f0;
  background: white;
  border-radius: 6px;
  font-size: 18px;
  font-weight: 700;
  cursor: pointer;
  transition: all 0.2s ease;
}

.qty-btn:hover:not(:disabled) {
  border-color: #667eea;
  color: #667eea;
}

.qty-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
}

.qty-input {
  width: 80px;
  padding: 8px;
  border: 2px solid #e2e8f0;
  border-radius: 6px;
  text-align: center;
  font-size: 14px;
  font-weight: 600;
}

.btn-toggle {
  padding: 8px 16px;
  border: 2px solid #667eea;
  background: white;
  color: #667eea;
  border-radius: 6px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.2s ease;
}

.btn-toggle:hover {
  background: #f1f5f9;
}

.btn-toggle.selected {
  background: #667eea;
  color: white;
}

/* Order Summary */
.order-summary {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  padding: 30px;
  border-radius: 12px;
  color: white;
  margin-bottom: 30px;
}

.order-summary h2 {
  font-size: 24px;
  margin-bottom: 20px;
}

.summary-content {
  display: flex;
  justify-content: space-between;
  align-items: center;
  gap: 30px;
}

.summary-items {
  display: grid;
  grid-template-columns: repeat(2, 1fr);
  gap: 20px;
  flex: 1;
}

.summary-stat {
  display: flex;
  flex-direction: column;
  gap: 5px;
}

.stat-label {
  font-size: 14px;
  opacity: 0.9;
}

.stat-value {
  font-size: 28px;
  font-weight: 700;
}

.btn-large {
  padding: 18px 36px;
  font-size: 18px;
}

.warning {
  margin-top: 15px;
  padding: 12px;
  background: rgba(239, 68, 68, 0.2);
  border-left: 4px solid #ef4444;
  border-radius: 4px;
}

/* Buttons */
.btn-primary,
.btn-secondary {
  padding: 12px 24px;
  border: none;
  border-radius: 8px;
  font-weight: 600;
  cursor: pointer;
  transition: all 0.3s ease;
}

.btn-primary {
  background: white;
  color: #667eea;
}

.btn-primary:hover:not(:disabled) {
  background: #f8fafc;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-secondary {
  background: #e2e8f0;
  color: #0f172a;
}

.btn-secondary:hover {
  background: #cbd5e1;
}

/* Modal */
.modal {
  display: none;
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: rgba(0, 0, 0, 0.6);
  z-index: 1000;
  align-items: center;
  justify-content: center;
}

.modal.active {
  display: flex;
}

.modal-content {
  background: white;
  border-radius: 16px;
  padding: 30px;
  max-width: 500px;
  width: 90%;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.modal-close {
  font-size: 28px;
  cursor: pointer;
  color: #64748b;
}

.modal-close:hover {
  color: #ef4444;
}

.modal-body {
  text-align: center;
  margin-bottom: 20px;
}

.success-icon {
  width: 80px;
  height: 80px;
  border-radius: 50%;
  background: #dcfce7;
  color: #15803d;
  font-size: 48px;
  display: flex;
  align-items: center;
  justify-content: center;
  margin: 0 auto 20px;
}

.order-details {
  background: #f8fafc;
  padding: 20px;
  border-radius: 8px;
  margin-top: 20px;
  text-align: left;
}

.order-details p {
  margin: 8px 0;
}

.modal-footer {
  display: flex;
  gap: 10px;
  justify-content: center;
}

/* Loading & Error States */
.loading,
.error,
.empty-state {
  padding: 40px;
  text-align: center;
  color: #64748b;
}

.error {
  color: #ef4444;
}

/* Responsive */
@media (max-width: 768px) {
  .recommendations-grid {
    grid-template-columns: 1fr;
  }

  .summary-content {
    flex-direction: column;
  }

  .summary-items {
    grid-template-columns: 1fr;
    width: 100%;
  }
}
</style>
