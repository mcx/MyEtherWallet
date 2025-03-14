<template>
  <div class="pa-8 moonpay-sell-component">
    <!-- ========================================================================= -->
    <!-- Selling amount in crypto -->
    <!-- ========================================================================= -->
    <div class="mt-2">
      <div class="mb-2 d-flex align-center justify-space-between">
        <div class="font-weight-medium textDark--text">
          How much do you want to sell?
        </div>
        <app-button-balance
          v-if="!loading && !fetchingBalance"
          style="position: relative; top: 0; right: 0"
          :balance="selectedBalance"
          :loading="loading"
        />
      </div>

      <div class="d-flex align-start">
        <mew-input
          v-model="amount"
          hide-clear-btn
          class="no-right-border"
          type="number"
          placeholder="Enter amount to sell"
          style="max-width: 251px; max-height: 86px"
          :max-btn-obj="maxButton"
          :disabled="loading"
          :error-messages="errorMessages"
          :persistent-hint="hasPersistentHint"
          :hint="persistentHintMessage"
          @keydown.native="preventCharE($event)"
        />
        <div
          class="d-flex align-center token-select-button"
          @click="openTokenSelect = true"
        >
          <mew-token-container :img="selectedCurrency.img" size="28px" />
          <div class="basic--text" style="margin-left: 8px">
            {{ selectedCurrency.symbol }}
          </div>
          <v-icon class="ml-auto" size="20px" color="titlePrimary">
            mdi-chevron-down
          </v-icon>
        </div>
      </div>
    </div>

    <!-- ========================================================================= -->
    <!-- Receiving amount in fiat -->
    <!-- ========================================================================= -->
    <div class="mt-2">
      <div class="font-weight-medium textDark--text mb-2">You will get</div>
      <div class="d-flex align-start">
        <mew-input
          is-read-only
          :value="cryptoAmount"
          hide-clear-btn
          type="number"
          class="no-right-border"
        />
        <mew-select
          v-model="selectedFiat"
          style="max-width: 135px"
          :items="fiatCurrencyItems"
          is-custom
          class="selectedFiat no-left-border"
        />
      </div>
    </div>

    <div class="pt-2 pb-10">
      <div class="mew-body textMedium--text">
        After submitting your sell order, you will have to send your crypto to
        Moonpay. Make sure to have enough currency in your wallet to cover
        network transaction fees.
      </div>

      <div
        v-if="errorMsg === ''"
        class="d-flex align-center justify-space-between mt-4"
      >
        <div class="mew-body textMedium--text">Estimated Network Fee</div>
        <div v-if="!estimatingFees" class="mew-body textMedium--text">
          ~{{ txFeeInEth }}
        </div>
        <v-skeleton-loader v-else type="text" width="150px" />
      </div>
      <div v-else class="d-flex align-center justify-space-between mt-4">
        <div class="mew-body textMedium--text error--text">{{ errorMsg }}</div>
      </div>
    </div>

    <!-- ============================================================== -->
    <!-- Sell button -->
    <!-- ============================================================== -->
    <mew-button
      title="SELL WITH MOONPAY"
      btn-size="xlarge"
      has-full-width
      :disabled="disableSell"
      @click.native="sell"
    />

    <!-- ========================================================================= -->
    <!-- Token select popup -->
    <!-- ========================================================================= -->
    <buy-sell-token-select
      :open="openTokenSelect"
      :networks="sellNetworks"
      :selected-currency="selectedCurrency"
      :set-currency="setCurrency"
      is-sell
      @close="openTokenSelect = false"
    />
  </div>
</template>

<script>
import { sha3 } from 'web3-utils';
import { mapActions, mapGetters, mapState } from 'vuex';
import { isEmpty, debounce, isNumber, isEqual } from 'lodash';
import BigNumber from 'bignumber.js';
import { fromWei, toBN } from 'web3-utils';

import { ERROR, Toast } from '@/modules/toast/handler/handlerToast';
import handlerSend from '@/modules/send/handlers/handlerSend.js';
import { MAIN_TOKEN_ADDRESS } from '@/core/helpers/common.js';
import abi from '@/modules/balance/handlers/abiERC20.js';
import { toBase } from '@/core/helpers/unit';
import handlerWallet from '@/core/mixins/handlerWallet.mixin';
import BuySellTokenSelect from '@/modules/buy-sell/components/TokenSelect.vue';
import { ARB, ETH, OP } from '@/utils/networks/types';
import handlerAnalytics from '@/modules/analytics-opt-in/handlers/handlerAnalytics.mixin';
import { BUY_SELL } from '@/modules/analytics-opt-in/handlers/configs/events';
export default {
  name: 'ModuleSellEth',
  components: {
    BuySellTokenSelect
  },
  mixins: [handlerWallet, handlerAnalytics],
  props: {
    close: {
      type: Function,
      default: () => {}
    },
    defaultCurrency: {
      type: Object,
      default: () => {}
    },
    sellFiats: {
      type: Array,
      default: () => []
    },
    sellNetworks: {
      type: Array,
      default: () => []
    }
  },
  data() {
    return {
      selectedFiat: {
        name: 'USD',
        value: 'USD',
        // eslint-disable-next-line
        img: require(`@/assets/images/currencies/USD.svg`)
      },
      openTokenSelect: false,
      selectedCurrency: {},
      amount: '0',
      locGasPrice: '0',
      sendHandler: {},
      loading: true,
      hasPersistentHint: false,
      fetchingBalance: true,
      gasLimit: 21000,
      estimatingFees: true,
      maxBalance: '0',
      selectedBalance: '0',
      toAddress: '',
      errorMsg: ''
    };
  },
  computed: {
    ...mapState('wallet', ['address', 'instance', 'web3']),
    ...mapState('global', ['gasPriceType']),
    ...mapGetters('wallet', ['balanceInETH', 'balanceInWei', 'tokensList']),
    ...mapGetters('global', ['isEthNetwork', 'network', 'gasPriceByType']),
    fiatCurrencyItems() {
      return this.sellFiats;
    },
    persistentHintMessage() {
      return this.hasPersistentHint
        ? `Max adjusted to leave sufficient ${this.selectedCurrency.symbol} for network fee`
        : '';
    },
    maxButton() {
      return BigNumber(this.selectedBalance).gt(0)
        ? {
            title: 'Max',
            method: this.setMax,
            disabled: BigNumber(this.txFee).gte(this.selectedBalance)
          }
        : {};
    },
    supportedNetworks() {
      return [ETH.name, ARB.name, OP.name, 'Polygon'];
    },
    name() {
      return this.selectedCurrency.symbol;
    },
    disableSell() {
      return (
        !this.amount ||
        this.amount === '' ||
        BigNumber(this.amount).eq(0) ||
        this.loading ||
        this.errorMessages !== ''
      );
    },
    foundFiat() {
      const foundFiat = this.sellFiats.find(
        fiat => fiat.name === this.selectedFiat.name
      );
      return foundFiat
        ? foundFiat
        : {
            name: 'USD',
            value: 'USD',
            // eslint-disable-next-line
            img: require(`@/assets/images/currencies/USD.svg`),
            limits: {
              min: 0.015,
              max: 3
            }
          };
    },
    min() {
      return BigNumber(this.foundFiat.limits.min).div(
        this.selectedCurrency.price
      );
    },
    max() {
      return BigNumber(this.foundFiat.limits.max).div(
        this.selectedCurrency.price
      );
    },
    txFee() {
      return fromWei(
        BigNumber(this.locGasPrice).times(this.gasLimit).toString()
      );
    },
    txFeeInEth() {
      return `${BigNumber(this.txFee).decimalPlaces(10)} ${
        this.network.type.currencyName
      }`;
    },
    errorMessages() {
      const symbol = this.selectedCurrency?.symbol
        ? this.name
        : this.network.type.currencyName;
      const amount = BigNumber(this.amount);

      if (BigNumber(this.selectedBalance).eq(0)) {
        return `Address provided has no ${this.selectedCurrency.symbol}`;
      }

      if (amount.isNaN() || amount.eq(0)) {
        return 'Amount required';
      }

      if (amount.lt(0)) {
        return "Amount can't be negative.";
      }

      if (amount.gt(0) && amount.lt(this.min)) {
        return `The minimum amount to sell is ${this.min.toString()} ${symbol}.`;
      }
      if (amount.gt(0) && amount.gt(this.max)) {
        return `The maximum amount to sell is ${this.max.toString()} ${symbol}.`;
      }

      if (amount.gt(this.selectedBalance)) {
        return `You do not have enough ${symbol} to sell.`;
      }
      if (!isEmpty(this.sendHandler) && !this.sendHandler.hasEnoughBalance()) {
        return `You do not have enough ETH to pay for network fee.`;
      }
      if (
        this.amount &&
        !handlerSend.helpers.hasValidDecimals(
          this.amount,
          this.selectedCurrency.decimals
        )
      ) {
        return `Invalid decimals! Max decimals for selected currency is ${this.selectedCurrency.decimals}`;
      }

      return '';
    },
    isValidAmount() {
      /** !amount */
      if (!this.amount) {
        return false;
      }
      if (!isNumber(this.selectedCurrency?.decimals)) {
        return false;
      }
      /** amount is negative */
      if (BigNumber(this.amount).lt(0)) {
        return false;
      }
      /** return amount has valid decimals */
      return handlerSend.helpers.hasValidDecimals(
        this.amount,
        this.selectedCurrency.decimals
      );
    },
    getCalculatedAmount() {
      const amount = new BigNumber(this.amount ? this.amount : 0)
        .times(new BigNumber(10).pow(this.selectedCurrency.decimals))
        .toFixed(0);
      return toBN(amount);
    },
    getAmountBN() {
      // Duplicate of getCalculatedAmount
      if (!this.isValidAmount) return toBN(0);
      const amount = toBase(
        this.amount ? this.amount : 0,
        this.selectedCurrency.decimals
      );
      return toBN(amount);
    },
    hasEnoughAssets() {
      try {
        const bal = toBase(
          this.selectedBalance,
          this.selectedCurrency.decimals
        );
        return toBN(bal).gte(this.getAmountBN);
      } catch (e) {
        Toast(e, {}, ERROR);
        return false;
      }
    },
    cryptoAmount() {
      return BigNumber(this.amount)
        .times(this.selectedCurrency.price)
        .toString();
    }
  },
  watch: {
    toAddress() {
      this.amount = '0';
    },
    selectedCurrency: {
      handler: function (newVal) {
        this.maxBalance = '0';
        this.hasPersistentHint = false;
        this.selectedBalance = '0';
        if (
          !isEmpty(this.sendHandler) &&
          this.selectedCurrency.hasOwnProperty('name')
        ) {
          this.sendHandler.setCurrency(newVal);
        }
        this.fetchSellInfo();
      },
      deep: true
    },
    selectedFiat: {
      handler: function (newVal, oldVal) {
        if (!isEqual(newVal, oldVal)) {
          this.$emit('selectedFiat', newVal);
        }
      },
      deep: true
    },
    amount(newVal) {
      this.debouncedSetAmount(newVal);
    },
    gasPriceType(newVal) {
      this.locGasPrice = this.gasPriceByType(newVal);
    },
    locGasPrice(val) {
      this.sendHandler.setLocalGasPrice(val);
    },
    gasLimit(val) {
      this.sendHandler.setGasLimit(val);
    },
    network() {
      this.maxBalance = '0';
      this.hasPersistentHint = false;
      this.selectedBalance = '0';
      this.amount = '0';
      this.selectedCurrency = {};
      this.selectedCurrency = this.defaultCurrency;
      if (this.supportedNetworks.includes(this.network.type.name)) {
        this.sendHandler = new handlerSend();
        this.fetchSellInfo();
        this.locGasPrice = this.gasPriceByType(this.gasPriceType);
      }
    }
  },
  mounted() {
    this.sendHandler = new handlerSend();
    this.fetchSellInfo();
    const foundFiat = this.sellFiats.find(
      fiat => fiat.name === this.selectedFiat.name
    );
    if (foundFiat) {
      this.selectedFiat = foundFiat;
    }

    const currentNetwork = this.sellNetworks.find(
      network => network.name === this.network.type.name
    );

    const findMain = currentNetwork.assets.find(
      asset => asset.contract === MAIN_TOKEN_ADDRESS
    );

    this.selectedCurrency = findMain
      ? findMain
      : this.defaultCurrency || currentNetwork.assets[0];
    this.locGasPrice = this.gasPriceByType(this.gasPriceType);
  },
  methods: {
    ...mapActions('external', ['setCoinGeckoTokens', 'getCoinGeckoTokenById']),
    getEthBalance() {
      const web3Instance = this.web3;
      web3Instance.eth.getBalance(this.address).then(res => {
        this.fetchingBalance = false;
        this.selectedBalance = fromWei(res);
      });
    },
    getTokenBalance() {
      const web3Instance = this.web3;
      const contract = new web3Instance.eth.Contract(
        abi,
        this.selectedCurrency.contract
      );
      contract.methods
        .balanceOf(this.address)
        .call()
        .then(res => {
          this.fetchingBalance = false;
          this.selectedBalance = BigNumber(res)
            .div(BigNumber(10).pow(this.selectedCurrency.decimals))
            .toString();
        });
    },
    debouncedSetAmount: debounce(function (newVal) {
      if (!BigNumber(newVal).eq(this.maxBalance)) {
        this.hasPersistentHint = false;
      }

      if (BigNumber(newVal).lt(0)) {
        return;
      }
      if (newVal && !isEmpty(this.sendHandler) && this.isValidAmount) {
        const newValue = BigNumber(newVal ? newVal : 0)
          .times(
            BigNumber(10).pow(
              this.selectedCurrency?.decimals
                ? this.selectedCurrency.decimals
                : 18
            )
          )
          .toString();
        this.sendHandler.setValue(newValue);
        if (this.errorMessages === '' && this.hasEnoughAssets) {
          this.estimatingFees = true;
          this.sendHandler
            .estimateGas()
            .then(res => {
              this.estimatingFees = false;
              this.gasLimit = res;
            })
            .catch(err => {
              Toast(err, {}, ERROR);
            });
        }
      }
    }, 500),
    setCurrency(e) {
      this.amount = '0';
      this.selectedCurrency = e;
    },
    setMax() {
      if (this.selectedCurrency.contract !== MAIN_TOKEN_ADDRESS) {
        const bal = this.sendHandler.getEntireBal();
        if (bal) {
          this.amount = BigNumber(bal)
            .div(
              BigNumber(10).pow(
                this.selectedCurrency.hasOwnProperty('name')
                  ? this.selectedCurrency.decimals
                  : 18
              )
            )
            .toString();
        } else {
          this.amount = this.selectedBalance;
        }
      } else {
        this.amount = BigNumber(this.max).lt(this.selectedBalance)
          ? this.max.toString()
          : BigNumber(this.selectedBalance).minus(this.txFee).toString();
      }
      this.maxBalance = this.amount;
      this.hasPersistentHint = true;
    },
    async sell() {
      this.trackBuySell(BUY_SELL.SELL_WITH_MOONPAY);
      const id = sha3(this.address)?.substring(0, 42);
      const request = await fetch(
        `https://mainnet.mewwallet.dev/v5/purchase/sell?id=${id}&address=${this.address}&fiatCurrency=${this.selectedFiat.name}&amount=${this.amount}&cryptoCurrency=${this.selectedCurrency.symbol}&chain=ETH&iso=US`
      );
      const response = await request.json();
      if (response.msg) {
        const { errors } = response;
        let error = '';
        errors.forEach(err => {
          error += `${err.msg}. `;
        });

        this.errorMsg = error
          ? `${response.msg} ${error.trim()}`
          : response.msg;
        return;
      }
      const url = response[0].url;
      window.open(url, '_blank');
      this.close();
    },
    fetchSellInfo() {
      if (this.selectedCurrency.contract) {
        this.fetchingBalance = true;
        if (this.selectedCurrency.contract === MAIN_TOKEN_ADDRESS) {
          this.getEthBalance();
        } else {
          this.getTokenBalance();
        }
        if (this.hasEnoughAssets) {
          this.sendHandler.setFrom(this.address);
          this.sendHandler.setCurrency(this.selectedCurrency);
          this.sendHandler.setValue(this.getCalculatedAmount);
          // eslint-disable-next-line
          this.sendHandler.setTo(ETH_DONATION_ADDRESS, 'TYPED');
          this.estimatingFees = true;
          this.sendHandler
            .estimateGas()
            .then(res => {
              this.estimatingFees = false;
              this.gasLimit = res;
            })
            .catch(err => {
              Toast(err, {}, ERROR);
            });
        }
      } else {
        this.fetchingBalance = false;
        this.selectedBalance = fromWei('0');
      }
      this.loading = false;
    },
    preventCharE(e) {
      if (e.key === 'e') e.preventDefault();
    }
  }
};
</script>

<style lang="scss" scoped>
.token-select-button {
  height: 56px;
  border: 1px solid var(--v-inputBorder-base);
  border-radius: 0 8px 8px 0;
  width: 135px;
  padding: 0 11px 0 14px;
  line-height: initial;
  user-select: none;
  cursor: pointer;
  &:hover {
    border: 1px solid var(--v-greyPrimary-base);
  }
}
</style>
<style lang="scss">
.moonpay-sell-component {
  .v-input__slot {
    height: 47px !important;
  }

  .no-right-border {
    fieldset {
      border-radius: 8px 0 0 8px !important;
    }
  }
  .no-left-border fieldset {
    border-radius: 0 8px 8px 0 !important;
  }
}
</style>
