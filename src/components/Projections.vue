<template>
  <div class="projections">
    <div class="card">
      <div class="card-header d-flex justify-content-between align-items-center">
        <h5 class="mb-0">Account Balance Projections</h5>
        <div class="d-flex gap-2">
          <select class="form-select form-select-sm" v-model="projectionMonths" style="width: auto;">
            <option value="6">6 Months</option>
            <option value="12">12 Months</option>
            <option value="24">24 Months</option>
            <option value="36">36 Months</option>
          </select>
          <button class="btn btn-sm btn-primary" @click="refreshProjections">
            <i class="bi bi-arrow-clockwise"></i> Refresh
          </button>
        </div>
      </div>
      <div class="card-body">
        <div v-if="loading" class="text-center py-5">
          <div class="spinner-border text-primary" role="status">
            <span class="visually-hidden">Loading...</span>
          </div>
          <p class="mt-2">Loading projections...</p>
        </div>
        
        <div v-else-if="error" class="alert alert-danger">
          <i class="bi bi-exclamation-triangle"></i> {{ error }}
        </div>
        
        <div v-else>
          <div class="chart-container" style="position: relative; height: 500px;">
            <canvas ref="chartCanvas"></canvas>
          </div>
          
          <div class="mt-4">
            <h6>Projection Summary</h6>
            <div class="row">
              <div class="col-md-4">
                <div class="card bg-light">
                  <div class="card-body">
                    <h6 class="card-title">Current Total</h6>
                    <p class="card-text h4">${{ formatAmount(currentTotalBalance) }}</p>
                  </div>
                </div>
              </div>
              <div class="col-md-4">
                <div class="card bg-light">
                  <div class="card-body">
                    <h6 class="card-title">Projected Total ({{ projectionMonths }} months)</h6>
                    <p class="card-text h4">${{ formatAmount(projectedTotalBalance) }}</p>
                  </div>
                </div>
              </div>
              <div class="col-md-4">
                <div class="card bg-light">
                  <div class="card-body">
                    <h6 class="card-title">Net Change</h6>
                    <p class="card-text h4" :class="netChangeClass">
                      {{ netChangeText }}
                    </p>
                  </div>
                </div>
              </div>
            </div>
          </div>
          
          <div class="mt-4">
            <h6>Account Details</h6>
            <div class="table-responsive">
              <table class="table table-sm">
                <thead>
                  <tr>
                    <th>Account</th>
                    <th>Current Balance</th>
                    <th>Projected Balance</th>
                    <th>Change</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-for="account in accounts" :key="account.id">
                    <td>{{ account.name }}</td>
                    <td>${{ formatAmount(account.currentBalance) }}</td>
                    <td>${{ formatAmount(account.projectedBalance) }}</td>
                    <td :class="getChangeClass(account.projectedBalance - account.currentBalance)">
                      {{ getChangeText(account.projectedBalance - account.currentBalance) }}
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { Chart, registerables } from 'chart.js';
import { nextTick } from 'vue';

export default {
  name: 'Projections',
  props: {
    api: {
      type: Object,
      required: true
    },
    budgetId: {
      type: String,
      required: true
    }
  },
  data() {
    return {
      loading: false,
      error: null,
      accounts: [],
      projectionMonths: 12,
      chart: null,
      currentTotalBalance: 0,
      projectedTotalBalance: 0
    };
  },
  computed: {
    netChange() {
      return this.projectedTotalBalance - this.currentTotalBalance;
    },
    netChangeClass() {
      if (this.netChange > 0) return 'text-success';
      if (this.netChange < 0) return 'text-danger';
      return 'text-muted';
    },
    netChangeText() {
      if (this.netChange > 0) return '+' + this.formatAmount(this.netChange);
      if (this.netChange < 0) return '-' + this.formatAmount(Math.abs(this.netChange));
      return '$0.00';
    }
  },
  async mounted() {
    Chart.register(...registerables);
    // Wait a bit for component to be fully rendered
    setTimeout(async () => {
      await this.loadProjections();
    }, 100);
  },
  beforeUnmount() {
    if (this.chart) {
      this.chart.destroy();
    }
  },
  watch: {
    projectionMonths() {
      this.loadProjections();
    }
  },
  methods: {
    async loadProjections() {
      this.loading = true;
      this.error = null;
      
      try {
        // Get accounts, targets, and scheduled transactions from YNAB API
        const [accountsResponse, targetsResponse, scheduledResponse] = await Promise.all([
          this.api.accounts.getAccounts(this.budgetId),
          this.api.categories.getCategories(this.budgetId),
          this.api.scheduledTransactions.getScheduledTransactions(this.budgetId)
        ]);
        
        // Process account data with projections
        this.accounts = this.processAccounts(
          accountsResponse.data.accounts, 
          targetsResponse.data.category_groups,
          scheduledResponse.data.scheduled_transactions
        );
        this.currentTotalBalance = this.accounts.reduce((sum, acc) => sum + acc.currentBalance, 0);
        this.projectedTotalBalance = this.accounts.reduce((sum, acc) => sum + acc.projectedBalance, 0);
        
        // Wait for DOM to be ready before creating chart
        nextTick(() => {
          this.createChart();
        });
      } catch (err) {
        this.error = 'Failed to load projections: ' + err.message;
        this.$emit('error', this.error);
      } finally {
        this.loading = false;
      }
    },
    
    processAccounts(accounts, categoryGroups, scheduledTransactions) {
      // Filter to on-budget accounts only (excluding credit cards for now)
      const budgetAccounts = accounts.filter(account => 
        account.on_budget && account.deleted === false && 
        account.type !== 'creditCard'
      );
      
      // Calculate monthly impacts
      const monthlyTargets = this.calculateMonthlyTargets(categoryGroups);
      const monthlyScheduled = this.calculateMonthlyScheduledTransactions(scheduledTransactions);
      
      return budgetAccounts.map((account, index) => {
        const currentBalance = account.balance / 1000; // Convert from milliunits
        const projectionData = this.calculateAccountProjection(
          currentBalance, 
          monthlyTargets, 
          monthlyScheduled,
          budgetAccounts.length,
          index
        );
        
        return {
          id: account.id,
          name: account.name,
          currentBalance: currentBalance,
          projectedBalance: projectionData.finalBalance,
          monthlyData: projectionData.monthlyData,
          color: this.getAccountColor(index)
        };
      });
    },
    
    calculateMonthlyTargets(categoryGroups) {
      let total = 0;
      const targetsByAccount = {};
      
      categoryGroups.forEach(group => {
        group.categories.forEach(category => {
          if (category.goal_type === 'MF' && category.goal_target_monthly) {
            const monthlyAmount = category.goal_target_monthly / 1000;
            total += monthlyAmount;
            // For now, distribute evenly - could be enhanced with account-specific targets
          }
        });
      });
      
      return { total, targetsByAccount };
    },
    
    calculateMonthlyScheduledTransactions(scheduledTransactions) {
      let monthlyIncome = 0;
      let monthlyExpenses = 0;
      
      scheduledTransactions.forEach(transaction => {
        if (transaction.deleted) return;
        
        const amount = Math.abs(transaction.amount) / 1000;
        const frequency = this.getTransactionFrequency(transaction);
        
        if (transaction.amount > 0) {
          monthlyIncome += (amount * frequency);
        } else {
          monthlyExpenses += (amount * frequency);
        }
      });
      
      return { monthlyIncome, monthlyExpenses, netMonthly: monthlyIncome - monthlyExpenses };
    },
    
    getTransactionFrequency(transaction) {
      // Simple frequency calculation - could be enhanced
      switch (transaction.frequency) {
        case 'weekly': return 4.33; // Average weeks per month
        case 'every_2_weeks': return 2.17;
        case 'every_4_weeks': return 1.08;
        case 'monthly': return 1;
        case 'every_2_months': return 0.5;
        case 'every_3_months': return 0.33;
        case 'quarterly': return 0.25;
        case 'twice_a_year': return 0.17;
        case 'yearly': return 0.083;
        case 'every_other_year': return 0.042;
        default: return 1;
      }
    },
    
    calculateAccountProjection(currentBalance, monthlyTargets, monthlyScheduled, numAccounts, accountIndex) {
      const monthlyData = [currentBalance];
      let balance = currentBalance;
      
      // Distribute targets evenly across accounts for now
      const accountMonthlyTarget = monthlyTargets.total / numAccounts;
      const netMonthlyChange = monthlyScheduled.netMonthly - accountMonthlyTarget;
      
      for (let month = 1; month <= this.projectionMonths; month++) {
        balance += netMonthlyChange;
        monthlyData.push(balance);
      }
      
      return {
        finalBalance: balance,
        monthlyData: monthlyData
      };
    },
    
    getAccountColor(index) {
      const colors = ['#007bff', '#28a745', '#6f42c1', '#dc3545', '#fd7e14', '#20c997', '#6c757d', '#e83e8c'];
      return colors[index % colors.length];
    },
    
    createChart() {
      if (this.chart) {
        this.chart.destroy();
      }
      
      // Retry mechanism for canvas element
      const tryCreateChart = (retryCount = 0) => {
        if (!this.$refs.chartCanvas) {
          if (retryCount < 10) {
            setTimeout(() => tryCreateChart(retryCount + 1), 100);
          } else {
            console.error('Chart canvas element not found after retries');
          }
          return;
        }
        
        const ctx = this.$refs.chartCanvas.getContext('2d');
        if (!ctx) {
          console.error('Could not get canvas context');
          return;
        }
        
        // Generate monthly data points
        const labels = [];
        const today = new Date();
        for (let i = 0; i <= this.projectionMonths; i++) {
          const date = new Date(today);
          date.setMonth(date.getMonth() + i);
          labels.push(date.toLocaleDateString('en-US', { month: 'short', year: 'numeric' }));
        }
        
        // Create datasets for each account
        const datasets = this.accounts.map(account => {
          return {
            label: account.name,
            data: account.monthlyData,
            borderColor: account.color,
            backgroundColor: account.color + '20',
            borderWidth: 2,
            fill: false,
            tension: 0.1
          };
        });
        
        this.chart = new Chart(ctx, {
          type: 'line',
          data: {
            labels: labels,
            datasets: datasets
          },
          options: {
            responsive: true,
            maintainAspectRatio: false,
            plugins: {
              title: {
                display: true,
                text: 'Account Balance Projections Over Time'
              },
              legend: {
                display: true,
                position: 'top'
              },
              tooltip: {
                mode: 'index',
                intersect: false,
                callbacks: {
                  label: function(context) {
                    return context.dataset.label + ': $' + formatAmount(context.parsed.y);
                  }
                }
              }
            },
            scales: {
              x: {
                display: true,
                title: {
                  display: true,
                  text: 'Time'
                }
              },
              y: {
                display: true,
                title: {
                  display: true,
                  text: 'Balance ($)'
                },
                ticks: {
                  callback: function(value) {
                    return '$' + formatAmount(value);
                  }
                }
              }
            }
          }
        });
      };
      
      tryCreateChart();
    },
    
    refreshProjections() {
      this.loadProjections();
    },
    
    updateChart() {
      nextTick(() => {
        this.createChart();
      });
    },
    
    formatAmount(value) {
      return (value || 0).toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
    },
    
    getChangeClass(change) {
      if (change > 0) return 'text-success';
      if (change < 0) return 'text-danger';
      return 'text-muted';
    },
    
    getChangeText(change) {
      if (change > 0) return '+' + this.formatAmount(change);
      if (change < 0) return '-' + this.formatAmount(Math.abs(change));
      return '$0.00';
    }
  }
};

// Helper function for formatting
function formatAmount(value) {
  return (value || 0).toFixed(2).replace(/\B(?=(\d{3})+(?!\d))/g, ',');
}
</script>

<style scoped>
.projections {
  padding: 20px;
}

.chart-container {
  background: white;
  border-radius: 8px;
  padding: 20px;
  border: 1px solid #dee2e6;
}

.card-header {
  background: #f8f9fa;
  border-bottom: 1px solid #dee2e6;
}

.table th {
  font-weight: 600;
  background: #f8f9fa;
}
</style>
