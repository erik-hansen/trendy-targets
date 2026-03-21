<template>
  <div class="navbar navbar-dark bg-primary d-flex flex-column flex-md-row align-items-center p-3 px-md-4 mb-3 border-bottom box-shadow">
    <div class="container">
      <div class="d-flex align-items-center justify-content-between">
        <div class="d-flex align-items-center">
          <!-- Hamburger Menu -->
          <div v-if="showMenu" class="dropdown me-3 position-relative">
            <button 
              class="btn btn-outline-light dropdown-toggle" 
              type="button" 
              @click="toggleDropdown"
              :class="{ 'show': isOpen }"
            >
              <i class="bi bi-list"></i> {{ currentView.charAt(0).toUpperCase() + currentView.slice(1) }}
            </button>
            <ul 
              class="dropdown-menu" 
              :class="{ 'show': isOpen }"
              style="position: absolute; top: 100%; left: 0;"
            >
              <li>
                <a 
                  class="dropdown-item" 
                  href="#"
                  :class="currentView === 'transactions' ? 'active' : ''"
                  @click.prevent="selectView('transactions')"
                >
                  <i class="bi bi-list-ul me-2"></i> Transactions
                </a>
              </li>
              <li>
                <a 
                  class="dropdown-item" 
                  href="#"
                  :class="currentView === 'targets' ? 'active' : ''"
                  @click.prevent="selectView('targets')"
                >
                  <i class="bi bi-bullseye me-2"></i> Targets
                </a>
              </li>
              <li>
                <a 
                  class="dropdown-item" 
                  href="#"
                  :class="currentView === 'projections' ? 'active' : ''"
                  @click.prevent="selectView('projections')"
                >
                  <i class="bi bi-graph-up me-2"></i> Projections
                </a>
              </li>
            </ul>
          </div>
          
          <h5 class="my-0 font-weight-normal navbar-brand">YNAB API Starter Kit</h5>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
export default {
  name: 'Nav',
  props: {
    showMenu: {
      type: Boolean,
      default: false
    },
    currentView: {
      type: String,
      default: 'targets'
    }
  },
  data() {
    return {
      isOpen: false
    }
  },
  emits: ['view-change'],
  methods: {
    toggleDropdown() {
      this.isOpen = !this.isOpen;
    },
    selectView(view) {
      this.$emit('view-change', view);
      this.isOpen = false;
    },
    closeDropdown(e) {
      if (!this.$el.contains(e.target)) {
        this.isOpen = false;
      }
    }
  },
  mounted() {
    document.addEventListener('click', this.closeDropdown);
  },
  beforeUnmount() {
    document.removeEventListener('click', this.closeDropdown);
  }
}
</script>
