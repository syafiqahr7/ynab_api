<template>
  <div id="app">
    <Nav />
    <div class="container">

      <!-- Loading -->
      <h1 v-if="loading" class="display-4">Loading...</h1>

      <!-- Error -->
      <div v-if="error">
        <h1 class="display-4">Oops!</h1>
        <p class="lead">{{ error }}</p>
        <button class="btn btn-primary" @click="resetToken">Try Again &gt;</button>
      </div>

      <!-- Main Content -->
      <div v-else>

        <!-- Not authorized -->
        <form v-if="!ynab.token">
          <h1 class="display-4">Congrats!</h1>
          <p class="lead">You have successfully initialized a new YNAB API Application!</p>
          <p>The next step is the OAuth configuration, you can
            <a href="https://github.com/ynab/ynab-api-starter-kit#step-2-obtain-an-oauth-client-id-so-the-app-can-access-the-ynab-api" target="_blank" rel="noopener noreferrer">read detailed instructions in the README.md</a>.</p>
          <ul>
            <li>Make sure to be logged into your YNAB account, go to your <a href="https://app.ynab.com/settings/developer" target="_blank" rel="noopener noreferrer">YNAB Developer Settings</a> and create a new OAuth Application.</li>
            <li>Enter the URL of this project as a Redirect URI (in addition to the existing three options), then "Save Application."</li>
            <li>Copy your Client ID and Redirect URI into the <em>src/config.json</em> file of your project.</li>
            <li>Then build your amazing app!</li>
          </ul>
          <p>If you have any questions please reach out to us at <strong>api@ynab.com</strong>.</p>
          <p>&nbsp;</p>

          <div class="form-group">
            <h2>Hello!</h2>
            <p class="lead">If you would like to use this App, please authorize with YNAB!</p>
            <button @click="authorizeWithYNAB" class="btn btn-primary">Authorize This App With YNAB &gt;</button>
          </div>
        </form>

        <!-- Budgets List -->
        <Budgets v-else-if="!budgetId" :budgets="budgets" :selectBudget="selectBudget" />

        <!-- Transactions + Flip & Transfer -->
        <div v-else>
          <Transactions :transactions="transactions" @flip-transfer="openFlipTransferModal" />
          <button class="btn btn-info mt-2" @click="budgetId = null">&lt; Select Another Budget</button>
        </div>

        <!-- Flip & Transfer Modal -->
        <div v-if="showFlipTransferModal" class="modal-overlay">
          <div class="modal-content">
            <h3>Select Target Budget to Transfer Flipped Transaction</h3>
            <select v-model="targetBudgetId" class="form-control mb-3">
              <option disabled value="">-- Select a Budget --</option>
              <option v-for="b in budgets" :key="b.id" :value="b.id" :disabled="b.id === budgetId">{{ b.name }}</option>
            </select>
            <button class="btn btn-primary" :disabled="!targetBudgetId" @click="flipAndTransferTransaction">Transfer</button>
            <button class="btn btn-secondary ml-2" @click="closeFlipTransferModal">Cancel</button>
          </div>
        </div>

      </div>

      <Footer />
    </div>
  </div>
</template>

<script>
// YNAB API import
import * as ynab from 'ynab';
import config from './config.json';

// Components
import Nav from './components/Nav.vue';
import Footer from './components/Footer.vue';
import Budgets from './components/Budgets.vue';
import Transactions from './components/Transactions.vue';

export default {
  data() {
    return {
      ynab: {
        clientId: config.clientId,
        redirectUri: config.redirectUri,
        token: null,
        api: null,
      },
      loading: false,
      error: null,
      budgetId: null,
      budgets: [],
      transactions: [],

      // Flip & Transfer state
      showFlipTransferModal: false,
      targetBudgetId: '',
      transactionToFlip: null,
    };
  },
  created() {
    this.ynab.token = this.findYNABToken();
    if (this.ynab.token) {
      this.api = new ynab.api(this.ynab.token);
      if (!this.budgetId) {
        this.getBudgets();
      } else {
        this.selectBudget(this.budgetId);
      }
    }
  },
  methods: {
    getBudgets() {
      this.loading = true;
      this.error = null;
      this.api.budgets.getBudgets()
        .then(res => {
          this.budgets = res.data.budgets;
        })
        .catch(err => {
          this.error = err.error.detail;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    selectBudget(id) {
      this.loading = true;
      this.error = null;
      this.budgetId = id;
      this.transactions = [];
      this.api.transactions.getTransactions(id)
        .then(res => {
          this.transactions = res.data.transactions;
        })
        .catch(err => {
          this.error = err.error.detail;
        })
        .finally(() => {
          this.loading = false;
        });
    },
    authorizeWithYNAB(e) {
      e.preventDefault();
      const uri = `https://app.ynab.com/oauth/authorize?client_id=${this.ynab.clientId}&redirect_uri=${this.ynab.redirectUri}&response_type=token`;
      location.replace(uri);
    },
    findYNABToken() {
      let token = null;
      const search = window.location.hash.substring(1).replace(/&/g, '","').replace(/=/g, '":"');
      if (search && search !== '') {
        const params = JSON.parse('{"' + search + '"}', function (key, value) {
          return key === '' ? value : decodeURIComponent(value);
        });
        token = params.access_token;
        sessionStorage.setItem('ynab_access_token', token);
        window.location.hash = '';
      } else {
        token = sessionStorage.getItem('ynab_access_token');
      }
      return token;
    },
    resetToken() {
      sessionStorage.removeItem('ynab_access_token');
      this.ynab.token = null;
      this.error = null;
    },

    // --- Flip & Transfer methods ---
    openFlipTransferModal(transaction) {
      this.transactionToFlip = transaction;
      this.targetBudgetId = '';
      this.showFlipTransferModal = true;
    },
    closeFlipTransferModal() {
      this.showFlipTransferModal = false;
      this.transactionToFlip = null;
    },
    async flipAndTransferTransaction() {
      if (!this.targetBudgetId || !this.transactionToFlip) return;

      this.loading = true;
      this.error = null;

      // Flip amount by negating it
      const flippedTransaction = {
        account_id: this.transactionToFlip.account_id, // same account - consider changing if needed
        date: this.transactionToFlip.date,
        amount: -this.transactionToFlip.amount,
        payee_name: this.transactionToFlip.payee_name,
        category_id: this.transactionToFlip.category_id, // Be careful: categories may not exist in target budget
        memo: this.transactionToFlip.memo ? this.transactionToFlip.memo + ' (flipped)' : 'Flipped transaction',
        cleared: 'cleared',
        approved: true,
      };

      try {
        await this.api.transactions.createTransaction(this.targetBudgetId, { transaction: flippedTransaction });
        alert('Transaction flipped and transferred successfully!');
        this.closeFlipTransferModal();

        // Optionally refresh transactions if the target budget is currently selected
        if (this.budgetId === this.targetBudgetId) {
          this.selectBudget(this.budgetId);
        }
      } catch (err) {
        this.error = err.error?.detail || 'Failed to transfer transaction';
      } finally {
        this.loading = false;
      }
    }
  },
  components: {
    Nav,
    Footer,
    Budgets,
    Transactions
  }
}
</script>

<style>
/* Simple modal styling */
.modal-overlay {
  position: fixed;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  background-color: rgba(0,0,0,0.5);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 9999;
}
.modal-content {
  background-color: white;
  padding: 20px;
  border-radius: 6px;
  width: 90%;
  max-width: 400px;
}
</style>
