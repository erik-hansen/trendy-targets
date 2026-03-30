<template>
  <div class="projections">
    <div class="card">
      <div class="card-header d-flex justify-content-between align-items-center">
        <h5 class="mb-0">Account Balance Projections</h5>
        <div class="d-flex gap-2 align-items-center">
          <div class="btn-group btn-group-sm" role="group">
            <button 
              type="button" 
              class="btn" 
              :class="showBankAccounts ? 'btn-primary' : 'btn-outline-primary'"
              @click="showBankAccounts = !showBankAccounts; updateChart();"
            >
              <i class="bi bi-bank me-1"></i> Bank Accounts
            </button>
            <button 
              type="button" 
              class="btn" 
              :class="showCreditCards ? 'btn-danger' : 'btn-outline-danger'"
              @click="showCreditCards = !showCreditCards; updateChart();"
            >
              <i class="bi bi-credit-card me-1"></i> Credit Cards
            </button>
          </div>
          <select class="form-select form-select-sm" v-model="projectionDays" style="width: auto;">
            <option value="30">30 Days</option>
            <option value="60">60 Days</option>
            <option value="90">90 Days</option>
            <option value="180">180 Days</option>
            <option value="365">1 Year</option>
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
                    <h6 class="card-title">Projected Total ({{ projectionDays }} days)</h6>
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
                    <td>${{ formatAmount(accountProjections[account.id]?.projectedBalance || account.currentBalance) }}</td>
                    <td :class="getChangeClass((accountProjections[account.id]?.projectedBalance || account.currentBalance) - account.currentBalance)">
                      {{ getChangeText((accountProjections[account.id]?.projectedBalance || account.currentBalance) - account.currentBalance) }}
                    </td>
                  </tr>
                </tbody>
              </table>
            </div>
          </div>
          
          <!-- Scheduled Transactions Table -->
          <div class="mt-4">
            <h6 class="mb-3">Scheduled Transactions (Next {{ projectionDays }} Days)</h6>
            <div class="table-responsive">
              <table class="table table-sm table-hover">
                <thead>
                  <tr>
                    <th>Date</th>
                    <th>Payee</th>
                    <th>Amount</th>
                    <th>Account</th>
                    <th>Frequency</th>
                  </tr>
                </thead>
                <tbody>
                  <tr v-if="scheduledTransactionList.length === 0">
                    <td colspan="5" class="text-center text-muted">No scheduled transactions in this period</td>
                  </tr>
                  <tr 
                    v-for="(transaction, index) in scheduledTransactionList" 
                    :key="index"
                  >
                    <td>{{ transaction.dateStr }}</td>
                    <td>{{ transaction.payee }}</td>
                    <td :class="transaction.amount > 0 ? 'text-success' : 'text-danger'">
                      {{ transaction.amount > 0 ? '+' : '' }}${{ formatAmount(Math.abs(transaction.amount)) }}
                    </td>
                    <td>{{ transaction.account }}</td>
                    <td>
                      <span class="badge" :class="transaction.isRepeating ? 'bg-info' : 'bg-secondary'">
                        {{ transaction.frequency }}
                      </span>
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
      projectionDays: 30,
      chart: null,
      showBankAccounts: true,
      showCreditCards: true
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
    },
    scheduledTransactionList() {
      const transactions = [];
      const today = new Date();
      today.setHours(0, 0, 0, 0);
      
      if (!this.accounts || this.accounts.length === 0) return transactions;
      
      // Create account lookup for transfer accounts
      const accountLookup = {};
      this.accounts.forEach(account => {
        accountLookup[account.id] = account;
      });
      
      // Collect all scheduled transactions from all accounts
      const allScheduled = [];
      this.accounts.forEach(account => {
        if (account.scheduledTransactions) {
          account.scheduledTransactions.forEach(scheduled => {
            allScheduled.push({
              ...scheduled,
              accountName: account.name,
              accountType: account.type
            });
            
            // If this is a transfer, create equivalent transaction for receiving account
            if (scheduled.transfer_account_id && accountLookup[scheduled.transfer_account_id]) {
              const targetAccount = accountLookup[scheduled.transfer_account_id];
              allScheduled.push({
                ...scheduled,
                id: scheduled.id + '_transfer',
                account_id: scheduled.transfer_account_id,
                amount: -scheduled.amount, // Opposite amount
                payee_name: account.name, // Source account as payee
                accountName: targetAccount.name,
                accountType: targetAccount.type,
                isTransferTarget: true
              });
            }
          });
        }
      });
      
      // Generate occurrences for each scheduled transaction
      allScheduled.forEach(scheduled => {
        const occurrences = this.getScheduledOccurrences(scheduled, today);
        occurrences.forEach(occurrence => {
          transactions.push({
            date: occurrence.date,
            dateStr: occurrence.date.toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' }),
            payee: scheduled.payee_name || 'Unknown',
            amount: scheduled.amount / 1000,
            accountId: scheduled.account_id,
            account: scheduled.accountName,
            accountType: scheduled.accountType,
            frequency: this.formatFrequency(scheduled.frequency),
            isRepeating: scheduled.frequency !== 'never',
            isTransfer: !!scheduled.transfer_account_id,
            isTransferTarget: scheduled.isTransferTarget
          });
        });
      });
      
      // Sort by date
      return transactions.sort((a, b) => a.date - b.date);
    },
    
    accountProjections() {
      // Build projection data for each account from scheduledTransactionList
      const projections = {};
      const today = new Date();
      today.setHours(0, 0, 0, 0);
      
      if (!this.accounts || this.accounts.length === 0) return projections;
      
      this.accounts.forEach(account => {
        // Start with historical data
        const dailyData = [...account.historicalData];
        let balance = account.currentBalance;
        
        // Apply scheduled transactions for each future day
        for (let day = 1; day <= this.projectionDays; day++) {
          const targetDate = new Date(today);
          targetDate.setDate(targetDate.getDate() + day);
          
          // Find all scheduled transactions for this account on this date
          const dayTransactions = this.scheduledTransactionList.filter(t => 
            t.accountId === account.id && t.date.toDateString() === targetDate.toDateString()
          );
          
          // Apply each transaction
          dayTransactions.forEach(t => {
            balance += t.amount;
          });
          
          dailyData.push(balance);
        }
        
        projections[account.id] = {
          dailyData: dailyData,
          projectedBalance: balance
        };
      });
      
      return projections;
    },
    
    currentTotalBalance() {
      if (!this.accounts || this.accounts.length === 0) return 0;
      return this.accounts.reduce((sum, acc) => sum + acc.currentBalance, 0);
    },
    
    projectedTotalBalance() {
      if (!this.accounts || this.accounts.length === 0) return 0;
      return this.accounts.reduce((sum, acc) => {
        const projection = this.accountProjections[acc.id];
        return sum + (projection ? projection.projectedBalance : acc.currentBalance);
      }, 0);
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
    projectionDays() {
      this.loadProjections();
    }
  },
  methods: {
    async loadProjections() {
      this.loading = true;
      this.error = null;
      
      try {
        // Calculate date range for historical transactions (30 days back)
        const today = new Date();
        const thirtyDaysAgo = new Date(today);
        thirtyDaysAgo.setDate(thirtyDaysAgo.getDate() - 30);
        
        // Get accounts, transactions, and scheduled transactions from YNAB API
        const [accountsResponse, transactionsResponse, scheduledResponse] = await Promise.all([
          this.api.accounts.getAccounts(this.budgetId),
          this.api.transactions.getTransactions(this.budgetId, thirtyDaysAgo.toISOString().split('T')[0]),
          this.api.scheduledTransactions.getScheduledTransactions(this.budgetId)
        ]);
        
        // Process account data with projections
        this.accounts = this.processAccounts(
          accountsResponse.data.accounts, 
          transactionsResponse.data.transactions,
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
    
    processAccounts(accounts, transactions, scheduledTransactions) {
      // Filter to on-budget accounts including credit cards
      const budgetAccounts = accounts.filter(account => 
        account.on_budget && account.deleted === false
      );
      
      return budgetAccounts.map((account, index) => {
        const currentBalance = account.balance / 1000; // Convert from milliunits
        
        // Calculate historical daily balances from transactions
        const historicalData = this.calculateHistoricalBalances(
          account.id, 
          currentBalance, 
          transactions
        );
        
        // Store raw scheduled transactions for the computed property
        const accountScheduled = scheduledTransactions.filter(t => 
          t.account_id === account.id && !t.deleted
        );
        
        return {
          id: account.id,
          name: account.name,
          type: account.type,
          currentBalance: currentBalance,
          historicalData: historicalData,
          scheduledTransactions: accountScheduled,
          color: this.getAccountColor(index, account.type)
        };
      });
    },
    
    calculateHistoricalBalances(accountId, currentBalance, transactions) {
      // Filter transactions for this account
      const accountTransactions = transactions.filter(t => 
        t.account_id === accountId && !t.deleted
      );
      
      const historicalData = [];
      const today = new Date();
      today.setHours(0, 0, 0, 0);
      
      // Create a map of transactions by date for efficient lookup
      const transactionsByDate = {};
      accountTransactions.forEach(t => {
        if (!transactionsByDate[t.date]) {
          transactionsByDate[t.date] = [];
        }
        transactionsByDate[t.date].push(t);
      });
      
      // Build daily balance going backwards from today
      let runningBalance = currentBalance;
      
      // Iterate from today backwards to 30 days ago
      for (let daysAgo = 0; daysAgo <= 30; daysAgo++) {
        const targetDate = new Date(today);
        targetDate.setDate(targetDate.getDate() - daysAgo);
        const dateStr = targetDate.toISOString().split('T')[0];
        
        // Get transactions for this date
        const dayTransactions = transactionsByDate[dateStr] || [];
        
        // Calculate the balance BEFORE this day's transactions
        // by subtracting today's transactions from the running balance
        dayTransactions.forEach(t => {
          // Subtract the transaction amount to get balance before it occurred
          // Positive amount (inflow) means balance was lower before
          // Negative amount (outflow) means balance was higher before
          runningBalance -= t.amount / 1000;
        });
        
        // Store the balance at the START of this day (before transactions)
        historicalData.push(runningBalance);
      }
      
      // Reverse to get chronological order (30 days ago → today)
      // This gives us the balance at the END of each day
      const chronologicalData = historicalData.reverse();
      
      // The last value should match current balance (end of today)
      // Adjust if needed by using current balance as the end point
      chronologicalData[chronologicalData.length - 1] = currentBalance;
      
      return chronologicalData;
    },
    
    calculateFutureProjections(currentBalance, scheduledTransactions, accountId) {
      const futureData = [];
      let balance = currentBalance;
      const today = new Date();
      
      // Get scheduled transactions for this account
      const accountScheduled = scheduledTransactions.filter(t => 
        t.account_id === accountId && !t.deleted
      );
      
      // For each future day, apply scheduled transactions
      for (let day = 1; day <= this.projectionDays; day++) {
        const targetDate = new Date(today);
        targetDate.setDate(targetDate.getDate() + day);
        
        // Check each scheduled transaction
        accountScheduled.forEach(scheduled => {
          if (this.shouldApplyScheduled(scheduled, targetDate, today)) {
            balance += scheduled.amount / 1000;
          }
        });
        
        futureData.push(balance);
      }
      
      return {
        finalBalance: balance,
        futureData: futureData,
        scheduledTransactions: accountScheduled
      };
    },
    
    shouldApplyScheduled(scheduled, targetDate, startDate) {
      const scheduledDate = new Date(scheduled.date_next);
      
      if (scheduled.frequency === 'never') {
        return scheduledDate.toDateString() === targetDate.toDateString();
      }
      
      // Calculate if this scheduled transaction should occur on target date
      const daysDiff = Math.floor((targetDate - startDate) / (1000 * 60 * 60 * 24));
      const scheduledDaysDiff = Math.floor((scheduledDate - startDate) / (1000 * 60 * 60 * 24));
      
      if (scheduledDaysDiff < 0) return false; // Already passed
      
      switch (scheduled.frequency) {
        case 'daily':
          return daysDiff >= scheduledDaysDiff && (daysDiff - scheduledDaysDiff) % 1 === 0;
        case 'weekly':
          return daysDiff >= scheduledDaysDiff && (daysDiff - scheduledDaysDiff) % 7 === 0;
        case 'every_2_weeks':
          return daysDiff >= scheduledDaysDiff && (daysDiff - scheduledDaysDiff) % 14 === 0;
        case 'every_4_weeks':
          return daysDiff >= scheduledDaysDiff && (daysDiff - scheduledDaysDiff) % 28 === 0;
        case 'monthly':
          return scheduledDate.getDate() === targetDate.getDate() && 
                 targetDate >= scheduledDate;
        case 'every_2_months':
          return scheduledDate.getDate() === targetDate.getDate() && 
                 targetDate >= scheduledDate &&
                 this.monthsDiff(scheduledDate, targetDate) % 2 === 0;
        case 'every_3_months':
          return scheduledDate.getDate() === targetDate.getDate() && 
                 targetDate >= scheduledDate &&
                 this.monthsDiff(scheduledDate, targetDate) % 3 === 0;
        case 'quarterly':
          return scheduledDate.getDate() === targetDate.getDate() && 
                 targetDate >= scheduledDate &&
                 this.monthsDiff(scheduledDate, targetDate) % 3 === 0;
        case 'twice_a_year':
          return scheduledDate.getDate() === targetDate.getDate() && 
                 targetDate >= scheduledDate &&
                 this.monthsDiff(scheduledDate, targetDate) % 6 === 0;
        case 'yearly':
          return scheduledDate.getDate() === targetDate.getDate() &&
                 scheduledDate.getMonth() === targetDate.getMonth() &&
                 targetDate >= scheduledDate;
        case 'every_other_year':
          return scheduledDate.getDate() === targetDate.getDate() &&
                 scheduledDate.getMonth() === targetDate.getMonth() &&
                 targetDate >= scheduledDate &&
                 (targetDate.getFullYear() - scheduledDate.getFullYear()) % 2 === 0;
        default:
          return false;
      }
    },
    
    monthsDiff(date1, date2) {
      return (date2.getFullYear() - date1.getFullYear()) * 12 + 
             (date2.getMonth() - date1.getMonth());
    },
    
    getScheduledOccurrences(scheduled, startDate) {
      const occurrences = [];
      const scheduledDate = new Date(scheduled.date_next);
      
      if (scheduled.frequency === 'never') {
        if (scheduledDate >= startDate && scheduledDate <= this.getEndDate(startDate)) {
          occurrences.push({ date: scheduledDate });
        }
        return occurrences;
      }
      
      // Generate occurrences within projection period
      let currentDate = new Date(scheduledDate);
      const endDate = this.getEndDate(startDate);
      
      // If scheduled date is in the past, start from next occurrence
      if (currentDate < startDate) {
        currentDate = this.getNextOccurrence(scheduled, startDate);
      }
      
      // Generate occurrences up to end date
      let maxIterations = 1000; // Safety limit
      while (currentDate <= endDate && maxIterations > 0) {
        if (currentDate >= startDate) {
          occurrences.push({ date: new Date(currentDate) });
        }
        currentDate = this.getNextOccurrence(scheduled, currentDate);
        maxIterations--;
      }
      
      return occurrences;
    },
    
    getNextOccurrence(scheduled, currentDate) {
      const next = new Date(currentDate);
      
      switch (scheduled.frequency) {
        case 'daily':
          next.setDate(next.getDate() + 1);
          break;
        case 'weekly':
          next.setDate(next.getDate() + 7);
          break;
        case 'every_2_weeks':
          next.setDate(next.getDate() + 14);
          break;
        case 'every_4_weeks':
          next.setDate(next.getDate() + 28);
          break;
        case 'monthly':
          next.setMonth(next.getMonth() + 1);
          break;
        case 'every_2_months':
          next.setMonth(next.getMonth() + 2);
          break;
        case 'every_3_months':
        case 'quarterly':
          next.setMonth(next.getMonth() + 3);
          break;
        case 'twice_a_year':
          next.setMonth(next.getMonth() + 6);
          break;
        case 'yearly':
          next.setFullYear(next.getFullYear() + 1);
          break;
        case 'every_other_year':
          next.setFullYear(next.getFullYear() + 2);
          break;
        default:
          next.setMonth(next.getMonth() + 1);
      }
      
      return next;
    },
    
    getEndDate(startDate) {
      const endDate = new Date(startDate);
      endDate.setDate(endDate.getDate() + this.projectionDays);
      return endDate;
    },
    
    getTooltipTransactions(accountId, targetDate, isHistorical) {
      const transactions = [];
      
      if (isHistorical) {
        // For historical data, we don't have transaction details loaded
        transactions.push('');
        transactions.push('Historical data point');
      } else {
        // For future data, get scheduled transactions for this date
        const dayTransactions = this.scheduledTransactionList.filter(t => {
          if (t.accountId !== accountId) return false;
          return t.date.toDateString() === targetDate.toDateString();
        });
        
        if (dayTransactions.length > 0) {
          transactions.push('');
          transactions.push('Scheduled transactions:');
          dayTransactions.slice(0, 5).forEach(t => {
            const sign = t.amount > 0 ? '+' : '';
            transactions.push(`  ${t.payee}: ${sign}$${this.formatAmount(Math.abs(t.amount))}`);
          });
          if (dayTransactions.length > 5) {
            transactions.push(`  ... and ${dayTransactions.length - 5} more`);
          }
        } else {
          transactions.push('');
          transactions.push('No scheduled transactions');
        }
      }
      
      return transactions;
    },
    
    formatFrequency(frequency) {
      const frequencyMap = {
        'never': 'One-time',
        'daily': 'Daily',
        'weekly': 'Weekly',
        'every_2_weeks': 'Bi-weekly',
        'every_4_weeks': 'Monthly',
        'monthly': 'Monthly',
        'every_2_months': 'Every 2 months',
        'every_3_months': 'Every 3 months',
        'quarterly': 'Quarterly',
        'twice_a_year': 'Twice a year',
        'yearly': 'Yearly',
        'every_other_year': 'Every 2 years'
      };
      return frequencyMap[frequency] || frequency;
    },
    
    getAccountColor(index, accountType) {
      // Use different colors for credit cards
      if (accountType === 'creditCard') {
        const creditCardColors = ['#dc3545', '#fd7e14', '#e83e8c', '#6f42c1'];
        return creditCardColors[index % creditCardColors.length];
      }
      
      const regularColors = ['#007bff', '#28a745', '#6f42c1', '#17a2b8', '#20c997', '#6c757d'];
      return regularColors[index % regularColors.length];
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
        
        // Generate date labels (historical + future)
        const labels = [];
        const today = new Date();
        
        // Historical labels (1 month back)
        for (let day = 30; day >= 0; day--) {
          const date = new Date(today);
          date.setDate(date.getDate() - day);
          labels.push(date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' }));
        }
        
        // Future labels
        for (let day = 1; day <= this.projectionDays; day++) {
          const date = new Date(today);
          date.setDate(date.getDate() + day);
          labels.push(date.toLocaleDateString('en-US', { month: 'short', day: 'numeric' }));
        }
        
        // Create datasets for each account (filtered by visibility settings)
        const filteredAccounts = this.accounts.filter(account => {
          if (account.type === 'creditCard') {
            return this.showCreditCards;
          }
          return this.showBankAccounts;
        });
        
        const datasets = filteredAccounts.map(account => {
          const projection = this.accountProjections[account.id];
          return {
            label: account.name + (account.type === 'creditCard' ? ' (CC)' : ''),
            data: projection ? projection.dailyData : account.historicalData,
            borderColor: account.color,
            backgroundColor: account.color + '20',
            borderWidth: 2,
            fill: false,
            tension: 0.1,
            accountId: account.id
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
                text: 'Account Balance: Historical (1 Month) + Future Projections'
              },
              legend: {
                display: true,
                position: 'top'
              },
              tooltip: {
                mode: 'nearest',
                intersect: true,
                callbacks: {
                  title: function(context) {
                    return context[0].label;
                  },
                  label: function(context) {
                    const accountName = context.dataset.label;
                    const balance = context.parsed.y;
                    return accountName + ': $' + formatAmount(balance);
                  },
                  footer: function(tooltipItems) {
                    const context = tooltipItems[0];
                    const dataIndex = context.dataIndex;
                    const accountId = context.dataset.accountId;
                    
                    // Calculate the actual date from the data index
                    const today = new Date();
                    today.setHours(0, 0, 0, 0);
                    
                    let targetDate;
                    if (dataIndex <= 30) {
                      // Historical date
                      targetDate = new Date(today);
                      targetDate.setDate(targetDate.getDate() - (30 - dataIndex));
                    } else {
                      // Future date
                      targetDate = new Date(today);
                      targetDate.setDate(targetDate.getDate() + (dataIndex - 30));
                    }
                    
                    // Get transactions for this account and date
                    const transactions = this.getTooltipTransactions(accountId, targetDate, dataIndex <= 30);
                    return transactions;
                  }.bind(this)
                }
              },
              verticalLine: {
                x: 30 // Position of "today" line (after 30 historical days)
              }
            },
            scales: {
              x: {
                display: true,
                title: {
                  display: true,
                  text: 'Date'
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
          },
          plugins: [{
            id: 'verticalLine',
            afterDraw: (chart) => {
              // Safety check: ensure we have at least 31 data points
              if (!chart.getDatasetMeta(0) || 
                  !chart.getDatasetMeta(0).data || 
                  chart.getDatasetMeta(0).data.length <= 30) {
                return;
              }
              
              const x = chart.getDatasetMeta(0).data[30].x;
              const scale = chart.scales.y;
              const ctx = chart.ctx;
              
              ctx.save();
              ctx.beginPath();
              ctx.moveTo(x, scale.top);
              ctx.lineTo(x, scale.bottom);
              ctx.lineWidth = 2;
              ctx.strokeStyle = 'rgba(255, 99, 132, 0.8)';
              ctx.setLineDash([6, 6]);
              ctx.stroke();
              
              // Add "Today" label
              ctx.fillStyle = 'rgba(255, 99, 132, 1)';
              ctx.textAlign = 'center';
              ctx.font = 'bold 12px Arial';
              ctx.fillText('Today', x, scale.top - 5);
              ctx.restore();
            }
          }]
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
