<template>
  <div class="container">
    <h5>Transactions</h5>
    <table class="table">
      <thead>
        <tr>
          <th>Account</th>
          <th>Date</th>
          <th>Payee</th>
          <th>Category</th>
          <th>Memo</th>
          <th>Amount</th>
          <th><!-- Flip & Transfer button column --></th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="transaction in transactions" :key="transaction.id">
          <td>{{ transaction.account_name }}</td>
          <td>{{ transaction.date }}</td>
          <td>{{ transaction.payee_name }}</td>
          <td>{{ transaction.category_name }}</td>
          <td>{{ transaction.memo }}</td>
          <td>{{ convertMilliUnitsToCurrencyAmount(transaction.amount).toFixed(2) }}</td>
          <td>
            <button
              class="btn btn-sm btn-warning"
              @click="$emit('flip-transfer', transaction)"
            >
              Flip & Transfer
            </button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>
</template>

<script>
// Import utils from YNAB
import { utils } from 'ynab';

export default {
  props: ['transactions'],
  methods: {
    // Format milliunits to currency
    convertMilliUnitsToCurrencyAmount: utils.convertMilliUnitsToCurrencyAmount,
  },
};
</script>
