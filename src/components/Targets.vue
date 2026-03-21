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
        style="position: fixed; top: 80px; left: 50%; transform: translateX(-50%); z-index: 999; background: white; padding: 15px 30px; border-bottom: 2px solid #dee2e6; box-shadow: 0 2px 8px rgba(0,0,0,0.15); width: 100%; max-width: 1140px;"
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
      <div style="height: 180px;"></div>
      
      <!-- Filters and Controls -->
      <div class="mb-3 d-flex justify-content-between align-items-center">
        <div class="d-flex gap-2">
          <input 
            type="text" 
            class="form-control form-control-sm" 
            placeholder="Filter categories..."
            v-model="filterText"
          />
          <select class="form-select form-select-sm" v-model="filterCadence" style="width: auto;">
            <option value="">All Cadences</option>
            <option v-for="cadence in uniqueCadences" :key="cadence" :value="cadence">{{ cadence }}</option>
          </select>
        </div>
        <div>
          <button class="btn btn-sm btn-outline-primary me-2" @click="expandAll">
            <i class="bi bi-chevron-down"></i> Expand All
          </button>
          <button class="btn btn-sm btn-outline-secondary" @click="collapseAll">
            <i class="bi bi-chevron-up"></i> Collapse All
          </button>
        </div>
      </div>
      
      <!-- Single Table with All Categories -->
      <div class="table-responsive">
        <table class="table table-sm table-hover">
          <thead class="table-purple">
            <tr>
              <th style="width: 30px;"></th>
              <th @click="sortBy('group')" style="cursor: pointer;">
                Group <i :class="getSortIcon('group')"></i>
              </th>
              <th @click="sortBy('name')" style="cursor: pointer;">
                Category <i :class="getSortIcon('name')"></i>
              </th>
              <th @click="sortBy('cadence')" style="cursor: pointer;">
                Cadence <i :class="getSortIcon('cadence')"></i>
              </th>
              <th @click="sortBy('monthlyTarget')" class="text-end" style="cursor: pointer;">
                Target <i :class="getSortIcon('monthlyTarget')"></i>
              </th>
              <th class="text-end">Difference</th>
              <th @click="sortBy('whatIfMonthly')" class="text-end" style="cursor: pointer;">
                What-If <i :class="getSortIcon('whatIfMonthly')"></i>
              </th>
              <th @click="sortBy('percentage')" class="text-end" style="cursor: pointer;">
                % <i :class="getSortIcon('percentage')"></i>
              </th>
            </tr>
          </thead>
          <tbody>
            <template v-for="group in sortedAndFilteredGroups" :key="group.id">
              <!-- Group Roll-up Row -->
              <tr 
                class="table-purple-light group-row" 
                @click="toggleGroup(group.id)"
                style="cursor: pointer; font-weight: bold;"
              >
                <td>
                  <i :class="isGroupCollapsed(group.id) ? 'bi bi-chevron-right' : 'bi bi-chevron-down'"></i>
                </td>
                <td colspan="2">{{ group.name }}</td>
                <td></td>
                <td class="text-end">${{ formatAmount(group.targetTotal) }}</td>
                <td class="text-end" :class="getGroupDifferenceClass(group)">{{ getGroupDifference(group) }}</td>
                <td class="text-end">${{ formatAmount(group.whatIfTotal) }}</td>
                <td class="text-end">
                  <div class="bar-container">
                    <div class="bar-bg"></div>
                    <div class="bar-purple" :style="getBarWidthStyle(group.whatIfTotal, grandWhatIfTotal)"></div>
                    <span class="bar-text">{{ getGroupPercentage(group) }}%</span>
                  </div>
                </td>
              </tr>
              <!-- Category Rows -->
              <tr 
                v-for="category in group.categories" 
                :key="category.id"
                v-show="!isGroupCollapsed(group.id) && matchesFilter(category, group)"
                class="category-row"
              >
                <td></td>
                <td></td>
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
                  <span :class="getCategoryDifferenceClass(category)">{{ getCategoryDifference(category) }}</span>
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
                  <div class="bar-container">
                    <div class="bar-bg"></div>
                    <div class="bar-purple" :style="getBarWidthStyle(category.whatIfMonthly, grandWhatIfTotal)"></div>
                    <span class="bar-text">{{ getCategoryPercentage(category) }}%</span>
                  </div>
                </td>
              </tr>
            </template>
          </tbody>
        </table>
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
      collapsedGroups: [],
      sortColumn: 'group',
      sortDirection: 'asc',
      filterText: '',
      filterCadence: ''
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
    },
    
    uniqueCadences() {
      const cadences = new Set();
      this.categoryGroups.forEach(group => {
        group.categories.forEach(cat => {
          cadences.add(this.getCadenceLabel(cat));
        });
      });
      return Array.from(cadences).sort();
    },
    
    sortedAndFilteredGroups() {
      let groups = [...this.categoryGroups];
      
      // Filter groups based on category matches
      if (this.filterText || this.filterCadence) {
        groups = groups.map(group => ({
          ...group,
          categories: group.categories.filter(cat => this.matchesFilter(cat, group))
        })).filter(group => group.categories.length > 0 || this.matchesFilter(null, group));
      }
      
      // Sort groups
      groups.sort((a, b) => {
        let comparison = 0;
        switch (this.sortColumn) {
          case 'group':
            comparison = a.name.localeCompare(b.name);
            break;
          case 'monthlyTarget':
            comparison = a.targetTotal - b.targetTotal;
            break;
          case 'whatIfMonthly':
            comparison = a.whatIfTotal - b.whatIfTotal;
            break;
          case 'percentage':
            const pctA = this.grandWhatIfTotal ? (a.whatIfTotal / this.grandWhatIfTotal) : 0;
            const pctB = this.grandWhatIfTotal ? (b.whatIfTotal / this.grandWhatIfTotal) : 0;
            comparison = pctA - pctB;
            break;
          default:
            comparison = a.name.localeCompare(b.name);
        }
        return this.sortDirection === 'asc' ? comparison : -comparison;
      });
      
      // Sort categories within each group
      groups = groups.map(group => ({
        ...group,
        categories: [...group.categories].sort((a, b) => {
          let comparison = 0;
          switch (this.sortColumn) {
            case 'name':
              comparison = a.name.localeCompare(b.name);
              break;
            case 'cadence':
              comparison = this.getCadenceLabel(a).localeCompare(this.getCadenceLabel(b));
              break;
            case 'monthlyTarget':
              comparison = a.monthlyTarget - b.monthlyTarget;
              break;
            case 'whatIfMonthly':
              comparison = a.whatIfMonthly - b.whatIfMonthly;
              break;
            case 'percentage':
              const pctA = this.grandWhatIfTotal ? (a.whatIfMonthly / this.grandWhatIfTotal) : 0;
              const pctB = this.grandWhatIfTotal ? (b.whatIfMonthly / this.grandWhatIfTotal) : 0;
              comparison = pctA - pctB;
              break;
            default:
              comparison = a.name.localeCompare(b.name);
          }
          return this.sortDirection === 'asc' ? comparison : -comparison;
        })
      }));
      
      return groups;
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
        return 'background: #6c757d; color: white; border-radius: 50px; padding: 4px 10px;';
      }
      
      switch (cadence) {
        case 0: return 'background: #6c757d; color: white; border-radius: 50px; padding: 4px 10px;';
        case 1: return frequency === 1 ? 'background: #007bff; color: white; border-radius: 50px; padding: 4px 10px;' : 'background: #004085; color: white; border-radius: 50px; padding: 4px 10px;';
        case 2: return frequency === 1 ? 'background: #28a745; color: white; border-radius: 50px; padding: 4px 10px;' : 'background: #1e7e34; color: white; border-radius: 50px; padding: 4px 10px;';
        case 3: case 6: case 9: case 12: return 'background: #6f42c1; color: white; border-radius: 50px; padding: 4px 10px;';
        case 4: return 'background: #17a2b8; color: white; border-radius: 50px; padding: 4px 10px;';
        case 5: case 7: case 10: case 11: return 'background: #6f42c1; color: white; border-radius: 50px; padding: 4px 10px;';
        case 8: return 'background: #6f42c1; color: white; border-radius: 50px; padding: 4px 10px;';
        case 13: return frequency === 1 ? 'background: #dc3545; color: white; border-radius: 50px; padding: 4px 10px;' : 'background: #721c24; color: white; border-radius: 50px; padding: 4px 10px;';
        case 14: return 'background: #343a40; color: white; border-radius: 50px; padding: 4px 10px;';
        default: return 'background: #6c757d; color: white; border-radius: 50px; padding: 4px 10px;';
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
      const rawValue = parseFloat(event.target.value) || 0;
      const value = Math.round(rawValue * 100) / 100; // Clip to 2 decimal places
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
    
    getBarWidthStyle(value, total) {
      if (total === 0) return 'width: 0%;';
      const percentage = Math.min((value / total) * 100, 100);
      return `width: ${percentage}%;`;
    },
    
    sortBy(column) {
      if (this.sortColumn === column) {
        this.sortDirection = this.sortDirection === 'asc' ? 'desc' : 'asc';
      } else {
        this.sortColumn = column;
        this.sortDirection = 'asc';
      }
    },
    
    getSortIcon(column) {
      if (this.sortColumn !== column) return 'bi bi-sort';
      return this.sortDirection === 'asc' ? 'bi bi-sort-up' : 'bi bi-sort-down';
    },
    
    matchesFilter(category, group) {
      const textMatch = !this.filterText || 
        (category && category.name.toLowerCase().includes(this.filterText.toLowerCase())) ||
        (group && group.name.toLowerCase().includes(this.filterText.toLowerCase()));
      
      const cadenceMatch = !this.filterCadence || 
        (category && this.getCadenceLabel(category) === this.filterCadence);
      
      return textMatch && cadenceMatch;
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

/* Purple Table Theme */
.table-purple {
  background: #6f42c1 !important;
  color: white !important;
}

.table-purple th {
  background: #5a2d96 !important;
  color: white !important;
  border-color: #4a1d86 !important;
}

.table-purple-light {
  background-color: #e9e1f8 !important;
}

.table-purple-light:hover {
  background-color: #dccff0 !important;
}

.group-row {
  background-color: #e9ecef !important;
  font-weight: bold;
}

.group-row:hover {
  background-color: #dee2e6 !important;
}

.category-row {
  background-color: #fff;
}

.category-row:hover {
  background-color: #f8f9fa;
}

.category-row td:first-child,
.category-row td:nth-child(2) {
  border-left: 3px solid #6f42c1;
  padding-left: 10px;
}

.table-responsive {
  border: 1px solid #dee2e6;
  border-radius: 8px;
  overflow: hidden;
}

.table td {
  position: relative;
  vertical-align: middle;
}

/* Bar graph in cells */
.bar-container {
  position: relative;
  width: 80px;
  height: 20px;
  display: inline-block;
  vertical-align: middle;
  margin: 0;
  padding: 0;
}

.bar-bg {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background: #e9ecef;
  border-radius: 4px;
  -webkit-border-radius: 4px;
  -moz-border-radius: 4px;
}

.bar-purple {
  position: absolute;
  top: 0;
  left: 0;
  height: 100%;
  background: #6f42c1;
  border-radius: 4px;
  -webkit-border-radius: 4px;
  -moz-border-radius: 4px;
  opacity: 0.6;
  transition: width 0.3s ease;
  -webkit-transition: width 0.3s ease;
  -moz-transition: width 0.3s ease;
}

.bar-text {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  -webkit-transform: translate(-50%, -50%);
  -moz-transform: translate(-50%, -50%);
  font-size: 0.75rem;
  font-weight: 600;
  color: #333;
  z-index: 10;
}

/* Cell bar graph backgrounds */
.cell-bar-background {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: #e9ecef;
  z-index: 1;
}

.cell-bar-fill {
  height: 100%;
  background: linear-gradient(90deg, rgba(108, 66, 193, 0.4), rgba(108, 66, 193, 0.15));
  transition: width 0.3s ease;
}

.position-relative {
  position: relative;
  z-index: 2;
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
.cadence-pill {
  display: inline-block !important;
  padding: 4px 10px !important;
  border-radius: 50px !important;
  font-size: 0.75rem !important;
  font-weight: 500 !important;
  white-space: nowrap !important;
  border: none !important;
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
.cadence-yearly { background: #dc8e35f8; color: white; }
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
