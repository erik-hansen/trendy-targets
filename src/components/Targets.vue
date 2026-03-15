<template>
  <div class="targets container">
    <h5>Budget Targets by Category</h5>
    
    <div v-if="loading" class="text-center my-3">
      <div class="spinner-border text-primary" role="status">
        <span class="visually-hidden">Loading...</span>
      </div>
    </div>
    
    <div v-else-if="error" class="alert alert-danger">
      {{ error }}
    </div>
    
    <div v-else>
      <!-- Sticky Totals Header -->
      <div 
        class="sticky-totals" 
        style="position: fixed; top: 0; left: 50%; transform: translateX(-50%); z-index: 1000; background: white; padding: 15px 30px; border-bottom: 2px solid #dee2e6; box-shadow: 0 2px 8px rgba(0,0,0,0.15); width: 100%; max-width: 1140px;"
      >
        <div style="display: flex; gap: 20px; justify-content: center;">
          <div style="flex: 1; max-width: 250px; padding: 15px 20px; border-radius: 8px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.15); background: linear-gradient(135deg, #667eea 0%, #764ba2 100%); color: white;">
            <div style="font-size: 0.85rem; font-weight: 600; margin-bottom: 5px; opacity: 0.9;">Monthly Target</div>
            <div style="font-size: 1.5rem; font-weight: bold;">${{ formatAmount(grandTargetTotal) }}</div>
          </div>
          <div style="flex: 1; max-width: 250px; padding: 15px 20px; border-radius: 8px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.15); background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%); color: white;">
            <div style="font-size: 0.85rem; font-weight: 600; margin-bottom: 5px; opacity: 0.9;">What-If Total</div>
            <div style="font-size: 1.5rem; font-weight: bold;">${{ formatAmount(grandWhatIfTotal) }}</div>
          </div>
          <div :style="differenceBoxStyle">
            <div style="font-size: 0.85rem; font-weight: 600; margin-bottom: 5px; opacity: 0.9;">Difference</div>
            <div style="font-size: 1.5rem; font-weight: bold;">
              <span v-if="grandWhatIfTotal === grandTargetTotal">--</span>
              <span v-else>
                {{ grandWhatIfTotal > grandTargetTotal ? '+' : '-' }}${{ formatAmount(Math.abs(grandWhatIfTotal - grandTargetTotal)) }}
              </span>
            </div>
          </div>
        </div>
      </div>
      
      <!-- Spacer to account for fixed header -->
      <div style="height: 100px;"></div>
      
      <!-- Collapse/Expand All Buttons -->
      <div class="mb-3 text-center">
        <button class="btn btn-sm btn-outline-primary me-2" @click="expandAll">
          <i class="bi bi-chevron-down"></i> Expand All
        </button>
        <button class="btn btn-sm btn-outline-secondary" @click="collapseAll">
          <i class="bi bi-chevron-up"></i> Collapse All
        </button>
      </div>
      
      <!-- Group Percentage Background -->
      <div v-for="group in categoryGroups" :key="group.id" class="category-group mb-2 position-relative">
        <h6 class="category-group-header p-2 position-relative" @click="toggleGroup(group.id)" style="cursor: pointer;">
          <strong>{{ group.name }}</strong>
          <span class="float-end">
            <i :class="isGroupCollapsed(group.id) ? 'bi bi-chevron-right' : 'bi bi-chevron-down'" class="me-2"></i>
            <span class="badge bg-light text-dark border group-badge">Target: ${{ formatAmount(group.targetTotal) }}</span>
            <span class="badge group-badge" :class="getGroupDifferenceClass(group)">{{ getGroupDifference(group) }}</span>
            <span class="badge bg-primary text-white group-badge">What-If: ${{ formatAmount(group.whatIfTotal) }}
              <div class="percentage-container d-inline-block">
                <div class="pie-chart" :style="getPieChartStyle(group.whatIfTotal, grandWhatIfTotal)"></div>
                <span class="percentage-value">{{ getGroupPercentage(group) }}%</span>
              </div>
            </span>
          </span>
        </h6>
        
        <div class="collapse" :class="{ show: !isGroupCollapsed(group.id) }">
          <div class="card card-body">
            <table class="table table-sm">
              <thead>
                <tr>
                  <th>Category</th>
                  <th>Target Cadence</th>
                  <th class="text-end">Monthly Target</th>
                  <th class="text-end">Difference</th>
                  <th class="text-end">What-If Monthly</th>
                  <th class="text-end">% of Total</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="category in group.categories" :key="category.id">
                  <td>{{ category.name }}</td>
                  <td>
                    <span 
                      :class="'cadence-pill ' + getCadenceClass(category)"
                      :style="getCadenceStyle(category)"
                    >
                      {{ getCadenceLabel(category) }}
                    </span>
                  </td>
                  <td class="text-end">${{ formatAmount(category.monthlyTarget) }}</td>
                  <td class="text-end">
                    <span :class="getCategoryDifferenceClass(category)">
                      {{ getCategoryDifference(category) }}
                    </span>
                  </td>
                  <td class="text-end">
                    <input 
                      type="number" 
                      class="form-control form-control-sm what-if-input"
                      :value="category.whatIfMonthly"
                      @input="updateWhatIf(category.id, $event)"
                      step="0.01"
                      min="0"
                    />
                  </td>
                  <td class="text-end">
                    <div class="percentage-container">
                      <div class="pie-chart" :style="getPieChartStyle(category.whatIfMonthly, grandWhatIfTotal)"></div>
                      <span class="percentage-value">{{ getCategoryPercentage(category) }}%</span>
                    </div>
                  </td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { utils } from 'ynab';

export default {
  name: 'Targets',
  props: ['api', 'budgetId'],
  emits: ['error'],
  
  data() {
    return {
      loading: false,
      error: null,
      categoryGroups: [],
      whatIfValues: {},
      collapsedGroups: []
    };
  },
  
  computed: {
    grandTargetTotal() {
      return this.categoryGroups.reduce((sum, group) => sum + group.targetTotal, 0);
    },
    
    grandWhatIfTotal() {
      return this.categoryGroups.reduce((sum, group) => sum + group.whatIfTotal, 0);
    },
    
    differenceClass() {
      const diff = this.grandWhatIfTotal - this.grandTargetTotal;
      if (diff > 0) return 'alert-warning';
      if (diff < 0) return 'alert-success';
      return 'alert-info';
    },
    
    differenceBoxClass() {
      const diff = this.grandWhatIfTotal - this.grandTargetTotal;
      if (diff > 0) return 'difference-more';
      if (diff < 0) return 'difference-less';
      return 'difference-neutral';
    },
    
    differenceBoxStyle() {
      const baseStyle = 'flex: 1; max-width: 250px; padding: 15px 20px; border-radius: 8px; text-align: center; box-shadow: 0 2px 8px rgba(0,0,0,0.15);';
      const diff = this.grandWhatIfTotal - this.grandTargetTotal;
      
      if (diff > 0) {
        return baseStyle + ' background: linear-gradient(135deg, #ff416c 0%, #ff4b2b 100%); color: white;';
      } else if (diff < 0) {
        return baseStyle + ' background: linear-gradient(135deg, #56ab2f 0%, #a8e063 100%); color: white;';
      }
      return baseStyle + ' background: #f8f9fa; color: #6c757d; border: 2px solid #dee2e6;';
    }
  },
  
  watch: {
    budgetId: {
      immediate: true,
      handler(newVal) {
        if (newVal) {
          this.loadCategories();
        }
      }
    }
  },
  
  methods: {
    async loadCategories() {
      this.loading = true;
      this.error = null;
      
      try {
        const response = await this.api.categories.getCategories(this.budgetId);
        this.processCategories(response.data.category_groups || []);
      } catch (err) {
        this.error = err.error?.detail || 'Failed to load categories';
        this.$emit('error', this.error);
      } finally {
        this.loading = false;
      }
    },
    
    processCategories(groups) {
      const whatIfValues = this.whatIfValues;
      
      this.categoryGroups = groups
        .filter(group => !group.deleted && !group.hidden)
        .map(group => {
          const categories = (group.categories || [])
            .filter(cat => !cat.deleted && !cat.hidden && cat.goal_target)
            .map(cat => {
              const monthlyTarget = this.calculateMonthlyTarget(cat);
              const whatIfMonthly = whatIfValues[cat.id] ?? monthlyTarget;
              
              return {
                ...cat,
                monthlyTarget,
                whatIfMonthly
              };
            });
          
          return {
            id: group.id,
            name: group.name,
            categories,
            targetTotal: categories.reduce((sum, cat) => sum + cat.monthlyTarget, 0),
            whatIfTotal: categories.reduce((sum, cat) => sum + cat.whatIfMonthly, 0)
          };
        })
        .filter(group => group.categories.length > 0);
    },
    
    calculateMonthlyTarget(category) {
      if (!category.goal_target) return 0;
      
      // goal_target is in milliunits, convert to dollars
      const targetAmount = category.goal_target / 1000;
      const cadence = category.goal_cadence;
      const frequency = category.goal_cadence_frequency || 1;
      
      // Handle null/undefined cadence
      if (cadence === null || cadence === undefined) {
        return 0;
      }
      
      // Calculate months per period based on cadence
      // Documentation: https://raw.githubusercontent.com/ynab/ynab-sdk-js/main/dist/models/Category.d.ts
      // 
      // For values 0, 1, 2, and 13: due date repeats every goal_cadence * goal_cadence_frequency
      // For values 3-12 and 14: goal_cadence_frequency is ignored
      
      let monthsPerPeriod;
      
      switch (cadence) {
        case 0: // None
          return 0;
        case 1: // Monthly - frequency tells us every N months
          monthsPerPeriod = frequency;
          break;
        case 2: // Weekly - 52 weeks per year
          // Weekly target: spread across 12 months
          return (targetAmount * 52 / 12) * frequency;
        case 3: // Every 2 months
          monthsPerPeriod = 2;
          break;
        case 4: // Every 3 months (Quarterly)
          monthsPerPeriod = 3;
          break;
        case 5: // Every 4 months
          monthsPerPeriod = 4;
          break;
        case 6: // Every 5 months
          monthsPerPeriod = 5;
          break;
        case 7: // Every 6 months (Semi-annually)
          monthsPerPeriod = 6;
          break;
        case 8: // Every 7 months
          monthsPerPeriod = 7;
          break;
        case 9: // Every 8 months
          monthsPerPeriod = 8;
          break;
        case 10: // Every 9 months
          monthsPerPeriod = 9;
          break;
        case 11: // Every 10 months
          monthsPerPeriod = 10;
          break;
        case 12: // Every 11 months
          monthsPerPeriod = 11;
          break;
        case 13: // Yearly
          monthsPerPeriod = 12 * frequency;
          break;
        case 14: // Every 2 years
          monthsPerPeriod = 24;
          break;
        default:
          console.warn(`Unknown cadence value: ${cadence} for category: ${category.name}`);
          return 0;
      }
      
      return targetAmount / monthsPerPeriod;
    },
    
    getCadenceLabel(category) {
      const cadence = category.goal_cadence;
      const frequency = category.goal_cadence_frequency || 1;
      
      // Handle null/undefined cadence
      if (cadence === null || cadence === undefined) {
        return 'No target';
      }
      
      switch (cadence) {
        case 0: return 'None';
        case 1: return frequency === 1 ? 'Monthly' : `Every ${frequency} months`;
        case 2: return frequency === 1 ? 'Weekly' : `Every ${frequency} weeks`;
        case 3: return 'Every 2 months';
        case 4: return 'Quarterly (Every 3 months)';
        case 5: return 'Every 4 months';
        case 6: return 'Every 5 months';
        case 7: return 'Semi-annually (Every 6 months)';
        case 8: return 'Every 7 months';
        case 9: return 'Every 8 months';
        case 10: return 'Every 9 months';
        case 11: return 'Every 10 months';
        case 12: return 'Every 11 months';
        case 13: return frequency === 1 ? 'Yearly' : `Every ${frequency} years`;
        case 14: return 'Every 2 years';
        default: return `Cadence ${cadence}`;
      }
    },
    
    getCadenceClass(category) {
      const cadence = category.goal_cadence;
      const frequency = category.goal_cadence_frequency || 1;
      
      if (cadence === null || cadence === undefined) {
        return 'cadence-none';
      }
      
      switch (cadence) {
        case 0: return 'cadence-none';
        case 1: return frequency === 1 ? 'cadence-monthly' : 'cadence-custom-months';
        case 2: return frequency === 1 ? 'cadence-weekly' : 'cadence-custom-weeks';
        case 3: case 6: case 9: case 12: return 'cadence-every-n'; // 2, 6, 8, 11 months
        case 4: return 'cadence-quarterly'; // 3 months
        case 5: case 7: case 10: case 11: return 'cadence-every-n'; // 4, 6, 9, 10 months
        case 8: return 'cadence-every-n'; // 7 months
        case 13: return frequency === 1 ? 'cadence-yearly' : 'cadence-custom-years';
        case 14: return 'cadence-two-years';
        default: return 'cadence-unknown';
      }
    },
    
    getCadenceStyle(category) {
      const cadence = category.goal_cadence;
      const frequency = category.goal_cadence_frequency || 1;
      
      if (cadence === null || cadence === undefined) {
        return 'background: #6c757d; color: white;';
      }
      
      switch (cadence) {
        case 0: return 'background: #6c757d; color: white;';
        case 1: return frequency === 1 ? 'background: #007bff; color: white;' : 'background: #004085; color: white;';
        case 2: return frequency === 1 ? 'background: #28a745; color: white;' : 'background: #1e7e34; color: white;';
        case 3: case 6: case 9: case 12: return 'background: #6f42c1; color: white;'; // Every N months
        case 4: return 'background: #17a2b8; color: white;'; // Quarterly
        case 5: case 7: case 10: case 11: return 'background: #6f42c1; color: white;'; // Every N months
        case 8: return 'background: #6f42c1; color: white;'; // Every 7 months
        case 13: return frequency === 1 ? 'background: #dc3545; color: white;' : 'background: #721c24; color: white;';
        case 14: return 'background: #343a40; color: white;';
        default: return 'background: #6c757d; color: white;';
      }
    },
    
    getCategoryDifference(category) {
      const diff = category.whatIfMonthly - category.monthlyTarget;
      if (diff === 0) return '--';
      return (diff > 0 ? '+' : '-') + this.formatAmount(Math.abs(diff));
    },
    
    getCategoryDifferenceClass(category) {
      const diff = category.whatIfMonthly - category.monthlyTarget;
      if (diff > 0) return 'text-danger';
      if (diff < 0) return 'text-success';
      return 'text-muted';
    },
    
    getGroupDifference(group) {
      const diff = group.whatIfTotal - group.targetTotal;
      if (diff === 0) return '--';
      return (diff > 0 ? '+' : '-') + this.formatAmount(Math.abs(diff));
    },
    
    getGroupDifferenceClass(group) {
      const diff = group.whatIfTotal - group.targetTotal;
      if (diff > 0) return 'text-danger';
      if (diff < 0) return 'text-success';
      return 'text-muted';
    },
    
    formatAmount(milliunits) {
      // Input is in milliunits (or already converted to dollars)
      // Check if value seems to be in milliunits (large number)
      const value = typeof milliunits === 'number' ? milliunits : 0;
      return value.toFixed(2);
    },
    
    updateWhatIf(categoryId, event) {
      const value = parseFloat(event.target.value) || 0;
      this.whatIfValues[categoryId] = value;
      
      // Update the category's whatIfMonthly value
      for (const group of this.categoryGroups) {
        const category = group.categories.find(c => c.id === categoryId);
        if (category) {
          category.whatIfMonthly = value;
          // Recalculate group totals
          group.whatIfTotal = group.categories.reduce((sum, cat) => sum + cat.whatIfMonthly, 0);
          break;
        }
      }
    },
    
    isGroupCollapsed(groupId) {
      return this.collapsedGroups.includes(groupId);
    },
    
    toggleGroup(groupId) {
      const index = this.collapsedGroups.indexOf(groupId);
      if (index > -1) {
        this.collapsedGroups.splice(index, 1);
      } else {
        this.collapsedGroups.push(groupId);
      }
    },
    
    expandAll() {
      this.collapsedGroups = [];
    },
    
    collapseAll() {
      this.collapsedGroups = this.categoryGroups.map(group => group.id);
    },
    
    getCategoryPercentage(category) {
      if (this.grandWhatIfTotal === 0) return '0.0';
      return ((category.whatIfMonthly / this.grandWhatIfTotal) * 100).toFixed(1);
    },
    
    getGroupPercentage(group) {
      if (this.grandWhatIfTotal === 0) return '0.0';
      return ((group.whatIfTotal / this.grandWhatIfTotal) * 100).toFixed(1);
    },
    
    getPieChartStyle(value, total) {
      if (total === 0) return '--pie-fill: 0deg;';
      const percentage = Math.min((value / total) * 100, 100);
      const degrees = (percentage / 100) * 360;
      return `--pie-fill: ${degrees}deg;`;
    },
  }
};
</script>

<style>
/* Sticky Totals Header - non-scoped for proper sticky behavior */
.sticky-totals {
  position: sticky !important;
  top: 0 !important;
  z-index: 1000 !important;
  background: white;
  padding: 15px 0;
  margin-bottom: 20px;
  margin-left: -15px;
  margin-right: -15px;
  padding-left: 15px;
  padding-right: 15px;
  border-bottom: 2px solid #dee2e6;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
}

.totals-row {
  display: flex;
  gap: 20px;
  justify-content: center;
}

.total-box {
  flex: 1;
  max-width: 250px;
  padding: 15px 20px;
  border-radius: 8px;
  text-align: center;
  box-shadow: 0 2px 8px rgba(0,0,0,0.15);
}

.total-label {
  font-size: 0.85rem;
  font-weight: 600;
  margin-bottom: 5px;
  opacity: 0.9;
}

.total-amount {
  font-size: 1.5rem;
  font-weight: bold;
}

.target-box {
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
}

.whatif-box {
  background: linear-gradient(135deg, #11998e 0%, #38ef7d 100%);
  color: white;
}

.difference-more {
  background: linear-gradient(135deg, #ff416c 0%, #ff4b2b 100%);
  color: white;
}

.difference-less {
  background: linear-gradient(135deg, #56ab2f 0%, #a8e063 100%);
  color: white;
}

.difference-neutral {
  background: #f8f9fa;
  color: #6c757d;
  border: 2px solid #dee2e6;
}
</style>

<style scoped>
.targets {
  margin-top: 20px;
  padding: 20px;
  border: 1px solid #dee2e6;
  border-radius: 8px;
  background: #fff;
}

.targets h5 {
  margin-bottom: 20px;
  padding-bottom: 10px;
  border-bottom: 2px solid #007bff;
}

.group-badge {
  margin-right: 20px;
}

.group-badge:last-child {
  margin-right: 0;
}

.group-header {
  display: flex;
  align-items: center;
  padding: 4px 8px;
  background: #f8f9fa;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  margin-bottom: 4px;
  position: relative;
}

/* Group Percentage Background */
.category-group {
  margin-bottom: 8px;
  border: 1px solid #dee2e6;
  border-radius: 4px;
  overflow: hidden;
}

.group-percentage-bg {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 38px;
  background: #f8f9fa;
  z-index: 1;
}

.group-bar-fill {
  height: 100%;
  background: rgba(0, 123, 255, 0.15);
  transition: width 0.3s ease;
}

.category-group-header {
  font-weight: bold;
  border-bottom: 1px solid #dee2e6;
  padding: 6px 10px;
  margin-bottom: 0;
  background: transparent;
}

/* Cadence Pills */
td .cadence-pill {
  display: inline-block;
  padding: 4px 8px;
  border-radius: 16px !important;
  font-size: 0.75rem;
  font-weight: 500;
  white-space: nowrap;
}

/* Percentage with Pie Chart */
.percentage-container {
  display: inline-flex;
  align-items: center;
  gap: 6px;
}

.pie-chart {
  width: 16px;
  height: 16px;
  border-radius: 50%;
  background: conic-gradient(#007bff var(--pie-fill), #e9ecef 0);
  flex-shrink: 0;
}

.percentage-value {
  font-size: 0.75rem;
  font-weight: 600;
  color: #333;
}

.cadence-none { background: #6c757d; color: white; }
.cadence-monthly { background: #007bff; color: white; }
.cadence-custom-months { background: #004085; color: white; }
.cadence-weekly { background: #28a745; color: white; }
.cadence-custom-weeks { background: #1e7e34; color: white; }
.cadence-quarterly { background: #17a2b8; color: white; }
.cadence-every-n { background: #6f42c1; color: white; }
.cadence-yearly { background: #dc3545; color: white; }
.cadence-custom-years { background: #721c24; color: white; }
.cadence-two-years { background: #343a40; color: white; }
.cadence-unknown { background: #6c757d; color: white; }


.table th {
  font-size: 0.85rem;
  padding: 4px 6px;
}

.table td {
  font-size: 0.9rem;
  padding: 4px 6px;
}

.table-sm th,
.table-sm td {
  padding: 3px 5px;
}
</style>
