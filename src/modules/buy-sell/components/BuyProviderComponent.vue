<template>
  <div class="py-8 px-8 pt-3">
    <div
      class="d-flex align-center textDark--text mb-5 cursor--pointer"
      @click="closeProviders"
    >
      <v-icon color="textDark">mdi-arrow-left mr-4</v-icon>
      <div class="mew-heading-2">Select a provider to buy</div>
    </div>
    <div class="mew-heading-2 mb-5">
      Spending {{ formattedFiat }} to buy {{ selectedCurrency.symbol }} on
      {{ network.type.name_long }} Network
    </div>
    <div>
      <mew-button
        v-for="(provider, idx) in buyQuote"
        :key="provider.provider + idx"
        btn-style="outline"
        :color-theme="$vuetify.theme.dark ? 'white' : 'basic'"
        has-full-width
        style="height: initial; min-height: 130px; position: relative"
        class="custom-provider-button mb-2"
        @click.native="buyProvider(provider)"
      >
        <div
          v-if="idx === 0"
          class="best-rate"
          :style="$vuetify.breakpoint.smAndDown ? 'top: -25px;' : 'top: -30px;'"
        >
          Best Rate
        </div>
        <div style="width: 100%">
          <div class="d-flex mew-heading-3">
            {{ provider.crypto_amount }}
            <div class="d-flex align-center">
              <span class="mew-heading-3 pl-1 mr-1">{{
                provider.crypto_currency
              }}</span>
              <mew-tooltip>
                <template #contentSlot>
                  <div class="elevated-box pa-3">
                    {{ generateFeeLabel(provider) }}
                    <br />
                    <br />
                    <br />
                    {{ generateLimits(provider.provider) }}
                    <br />
                    {{ generateLimits(provider.provider, false) }}
                  </div>
                </template>
              </mew-tooltip>
            </div>
          </div>
          <div class="d-flex align-center justify-space-between mt-3">
            <img
              :src="parseProviderLogo(provider)"
              :alt="provider.provider + ' logo'"
              width="100"
            />
            <div class="d-flex flex-column align-center mb-1">
              <div
                :class="[
                  'd-flex align-center mb-1 justify-end',
                  $vuetify.breakpoint.smAndDown ? 'flex-wrap' : ''
                ]"
                :style="
                  $vuetify.breakpoint.smAndDown
                    ? 'width : 150px;'
                    : 'width: 100%'
                "
              >
                <img
                  v-for="(logo, providerIdx) in parsePaymentMethods(
                    provider.payment_methods
                  )"
                  :key="providerIdx + logo"
                  :src="logo"
                  :alt="provider.provider + ' payment method'"
                  width="40"
                  class="mr-1"
                />
              </div>
              <div class="mew-label d-none d-md-block">
                {{ parsePaymentMethods(provider.payment_methods, true) }}
              </div>
            </div>
          </div>
        </div>
      </mew-button>
    </div>
    <div class="pt-2 text-center mew-label">
      Fees, availability, and purchase limits vary between providers, you can
      check their quotes and select one that works for you.
    </div>
  </div>
</template>

<script>
import { mapGetters } from 'vuex';
import { currencySymbols } from './tokenList';
import visaLogo from '@/assets/images/icons/moonpay/icon-visa.svg';
import mastercardLogo from '@/assets/images/icons/moonpay/icon-master.svg';
import bankLogo from '@/assets/images/icons/moonpay/icon-bank.svg';
import applePayLogo from '@/assets/images/icons/moonpay/icon-apple-pay.svg';
import paypalLogo from '@/assets/images/icons/moonpay/icon-paypal-logo.svg';
import googlePayLogo from '@/assets/images/icons/moonpay/icon-google-pay-logo.svg';
import pixLogo from '@/assets/images/icons/moonpay/icon-pix-logo.svg';

export default {
  name: 'ModuleBuyEthProvider',
  props: {
    buyQuote: {
      type: Array,
      default: () => []
    },
    selectedCurrency: {
      type: Object,
      default: () => ({})
    },
    buyProvider: {
      type: Function,
      default: () => {}
    },
    closeProviders: {
      type: Function,
      default: () => {}
    },
    selectedFiat: {
      type: Object,
      default: () => ({})
    }
  },
  data() {
    return {};
  },
  computed: {
    ...mapGetters('global', ['network']),
    formattedFiat() {
      const symbol = currencySymbols[this.selectedFiat.name]
        ? currencySymbols[this.selectedFiat.name]
        : '';
      return `${symbol}${this.buyQuote[0]?.fiat_amount || 0}`;
    },
    isEUR() {
      return this.selectedFiat.name === 'EUR';
    }
  },
  methods: {
    parsePaymentMethods(paymentMethods, label = false) {
      const logos = [];
      let paymentMethodsLabel = '';
      if (
        paymentMethods.includes('CREDIT_CARD') ||
        paymentMethods.includes('DEBIT_CARD') ||
        paymentMethods.includes('CARD')
      ) {
        logos.push(visaLogo);
        logos.push(mastercardLogo);
        paymentMethodsLabel += 'Visa, Mastercard';
      }

      if (
        (paymentMethods.includes('ACH') ||
          paymentMethods.includes('ACH_BANK_ACCOUNT') ||
          paymentMethods.includes('SEPA_OPEN_BANKING')) &&
        this.isEUR
      ) {
        logos.push(bankLogo);
        paymentMethodsLabel += `${
          paymentMethodsLabel.length > 0 ? ', ' : ''
        }Bank account`;
      }

      if (paymentMethods.includes('PAYPAL')) {
        logos.push(paypalLogo);
        paymentMethodsLabel += `${
          paymentMethodsLabel.length > 0 ? ', ' : ''
        }Paypal`;
      }

      if (paymentMethods.includes('APPLE_PAY')) {
        logos.push(applePayLogo);
        paymentMethodsLabel += `${
          paymentMethodsLabel.length > 0 ? ', ' : ''
        }Apple Pay`;
      }

      if (paymentMethods.includes('GOOGLE_PAY')) {
        logos.push(googlePayLogo);
        paymentMethodsLabel += `${
          paymentMethodsLabel.length > 0 ? ', ' : ''
        }Google Pay`;
      }

      if (paymentMethods.includes('PIX')) {
        logos.push(pixLogo);
        paymentMethodsLabel += `${
          paymentMethodsLabel.length > 0 ? ', ' : ''
        }Pix`;
      }

      return label ? paymentMethodsLabel : logos;
    },
    parseProviderLogo(provider) {
      const dark = this.$vuetify.theme.dark;
      const providerLogos = {
        SIMPLEX: require('@/assets/images/providers/icon-simplex-logo.png'),
        MOONPAY: require('@/assets/images/providers/icon-moonpay.svg'),
        TOPPER: require('@/assets/images/providers/icon-topper.svg'),
        COINBASE: require('@/assets/images/providers/icon-coinbase-light.webp')
      };
      const providerLogosDarkMode = {
        SIMPLEX: require('@/assets/images/providers/icon-simplex-light.png'),
        MOONPAY: require('@/assets/images/providers/icon-moonpay-white.svg'),
        TOPPER: require('@/assets/images/providers/icon-topper-dark.svg'),
        COINBASE: require('@/assets/images/providers/icon-coinbase.webp')
      };
      return dark
        ? providerLogosDarkMode[provider.provider]
        : providerLogos[provider.provider];
    },
    generateFeeLabel(provider) {
      const feeLabel = {
        MOONPAY: `Includes ${this.isEUR ? 0.7 : 4.99}% fee`,
        SIMPLEX: 'Includes fee 5.25% fee',
        TOPPER: 'Includes 4.65% fee. First transaction is free.',
        COINBASE: 'Includes 2.5% fee.'
      };

      return feeLabel[provider.provider];
    },
    generateLimits(provider, daily = true) {
      const dailyLimits = {
        MOONPAY: 'Daily limit: $10,000',
        SIMPLEX: 'Daily limit: $20,000',
        TOPPER: 'Daily limit: $20,000',
        COINBASE: 'Daily limit: $20,000'
      };

      const monthlyLimits = {
        MOONPAY: 'Monthly limit: $50,000',
        SIMPLEX: 'Monthly limit: $50,000',
        TOPPER: 'Monthly limit: $50,000',
        COINBASE: 'Monthly limit: $50,000'
      };

      return daily ? dailyLimits[provider] : monthlyLimits[provider];
    }
  }
};
</script>

<style lang="scss" scoped>
.section-block {
  border-radius: 12px;
  left: 0px;
  top: 0px;
  box-sizing: border-box;
  border: 2px solid var(--v-greyMedium-base);
  flex: none;
  order: 0;
  align-self: stretch;
  flex-grow: 0;
  margin: 8px 0px;
  position: relative;
}
.section-block:hover {
  cursor: pointer;
  border: 2px solid #1eb19b;
  background-color: #e5eaee;
}
.provider-logo {
  position: absolute;
  top: 18px;
  right: 20px;
}

.best-rate {
  background-color: #05c0a5 !important;
  text-align: center;
  font-size: small;
  width: 80px;
  border-radius: 4px;
  left: 0;
  padding: 4px 8px;
  position: absolute;
  color: white;
}
</style>

<style lang="scss">
.custom-provider-button {
  .v-btn__content > div {
    position: relative;
  }
}
</style>
