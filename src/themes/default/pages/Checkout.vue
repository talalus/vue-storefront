<template>
  <div id="checkout">
    <div class="container">
      <div class="row" v-show="!orderPlaced">
        <div class="col-sm-7 col-xs-12 pb70 pl40">
          <header>
            <h1>Checkout</h1>
          </header>
          <personal-details :is-active="activeSection.personalDetails"/>
          <shipping :is-active="activeSection.shipping"/>
          <payment :is-active="activeSection.payment"/>
          <order-review :is-active="activeSection.orderReview"/>
        </div>
        <div class="col-sm-5 col-xs-12 bg-lightgray">
            <cart-summary />
        </div>
      </div>
      <div class="row" v-show="orderPlaced">
        <div class="col-xs-12">
          <thank-you-page />
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { corePage } from 'lib/themes'
import EventBus from 'src/event-bus/event-bus'

import PersonalDetails from 'theme/components/core/blocks/Checkout/PersonalDetails.vue'
import Shipping from 'theme/components/core/blocks/Checkout/Shipping.vue'
import Payment from 'theme/components/core/blocks/Checkout/Payment.vue'
import OrderReview from 'theme/components/core/blocks/Checkout/OrderReview.vue'
import CartSummary from 'theme/components/core/blocks/Checkout/CartSummary.vue'

export default {
  name: 'Checkout',
  beforeMount () {
    if (this.$store.state.cart.cartItems.length === 0) {
      EventBus.$emit('notification', {
        type: 'warning',
        message: 'Shopping cart is empty. Please add some products before entering Checkout',
        action1: { label: 'OK', action: 'close' }
      })
      this.$router.push('/')
    } else {
      this.stockCheckCompleted = false
      for (let product of this.$store.state.cart.cartItems) { // check the results of online stock check
        const checkPromises = []
        if (product.onlineStockCheckid) {
          checkPromises.push(new Promise((resolve, reject) => {
            global.db.syncTaskCollection.getItem(product.onlineStockCheckid, function (err, item) {
              if (err) {
                console.error(err)
                resolve(null)
              } else {
                product.stock = item.result
                resolve(product)
              }
            })
          }))
          Promise.all(checkPromises).then(checkedProducts => {
            this.stockCheckCompleted = true
            this.stockCheckOK = true
            for (let chp of checkedProducts) {
              if (chp) {
                if (!chp.stock.is_in_stock) {
                  this.stockCheckOK = false
                  chp.warning_message = 'Out of stock!'
                  EventBus.$emit('notification', {
                    type: 'error',
                    message: chp.name + ' is out of the stock!',
                    action1: { label: 'OK', action: 'close' }
                  })
                }
              }
            }
          })
        }
      }
    }
  },
  created () {
    // TO-DO: Dont use event bus ad use v-on at components (?)
    EventBus.$on('network.status', (status) => { this.checkConnection(status) })
    // TO-DO: Use one event with name as apram
    EventBus.$on('checkout.personalDetails', (receivedData, validationResult) => {
      this.personalDetails = receivedData
      this.validationResults.personalDetails = validationResult
      this.activateSection('shipping')
    })
    EventBus.$on('checkout.shipping', (receivedData, validationResult) => {
      this.shipping = receivedData
      this.validationResults.shipping = validationResult
      global.__TAX_COUNTRY__ = this.shipping.country
      this.activateSection('payment')
    })
    EventBus.$on('checkout.payment', (receivedData, validationResult) => {
      this.payment = receivedData
      this.validationResults.payment = validationResult
      this.activateSection('orderReview')
    })
    EventBus.$on('checkout.cartSummary', (receivedData) => {
      this.cartSummary = receivedData
    })
    EventBus.$on('checkout.placeOrder', () => this.placeOrder())
    EventBus.$on('checkout.edit', (section) => {
      this.activateSection(section)
    })
  },
  computed: {
    isValid () {
      let isValid = true
      for (let child of this.$children) {
        console.log(child)
        if (child.hasOwnProperty('$v')) {
          if (child.$v.$invalid) {
            isValid = false
            break
          }
        }
      }

      if (navigator.onLine) {
        if (this.stockCheckCompleted) {
          if (!this.stockCheckOK) {
            isValid = false
            EventBus.$emit('notification', {
              type: 'error',
              message: 'Some of the ordered products are not available!',
              action1: { label: 'OK', action: 'close' }
            })
          }
        } else {
          EventBus.$emit('notification', {
            type: 'warning',
            message: 'Stock check in progress, please wait while available stock quantities are checked',
            action1: { label: 'OK', action: 'close' }
          })
          isValid = false
        }
      }
      return isValid
    }
  },
  data () {
    return {
      stockCheckCompleted: false,
      stockCheckOK: false,
      orderPlaced: false,
      activeSection: {
        personalDetails: true,
        shipping: false,
        payment: false,
        orderReview: false
      },
      order: {},
      personalDetails: {},
      shipping: {},
      payment: { paymentMethod: 'cashondelivery' },
      orderReview: {},
      cartSummary: {},
      validationResults: {
        personalDetails: { $invalid: true },
        shipping: { $invalid: true },
        payment: { $invalid: true }
      }
    }
  },
  methods: {
    checkConnection (status) {
      if (!status.online) {
        EventBus.$emit('notification', {
          type: 'warning',
          message: 'There is no Internet connection. You can still place your order. We will notify you if any of ordered products is not avaiable because we cannot check it right now.',
          action1: { label: 'OK', action: 'close' }
        })
      }
    },
    activateSection (sectionToActivate) {
      for (let section in this.activeSection) {
        this.activeSection[section] = false
      }
      this.activeSection[sectionToActivate] = true
    },
    prepareOrder () {
      this.order = {
        products: this.$store.state.cart.cartItems,
        addressInformation: {
          shippingAddress: {
            region: this.shipping.state,
            region_id: 0,
            country_id: this.shipping.country,
            street: [this.shipping.streetAddress, this.shipping.apartmentNumber],
            company: 'NA', // TODO: Fix me! https://github.com/DivanteLtd/vue-storefront/issues/224
            telephone: this.shipping.phoneNumber,
            postcode: this.shipping.zipCode,
            city: this.shipping.city,
            firstname: this.shipping.firstName,
            lastname: this.shipping.lastName,
            email: this.personalDetails.emailAddress,
            region_code: ''
          },
          billingAddress: {
            region: this.shipping.state,
            region_id: 0,
            country_id: this.shipping.country,
            street: [this.shipping.streetAddress, this.shipping.apartmentNumber],
            company: 'NA',
            telephone: this.shipping.phoneNumber,
            postcode: this.shipping.zipCode,
            city: this.shipping.city,
            firstname: this.personalDetails.firstName,
            lastname: this.personalDetails.lastName,
            email: this.personalDetails.emailAddress,
            region_code: ''
          },
          shipping_method_code: this.shipping.shippingMethod,
          shipping_carrier_code: this.shipping.shippingMethod,
          payment_method_code: this.payment.paymentMethod
        }
      }
      return this.order
    },
    placeOrder () {
      this.checkConnection({ online: navigator.onLine })
      if (this.isValid) {
        this.$store.dispatch('checkout/placeOrder', { order: this.prepareOrder() })
        this.orderPlaced = true
        console.log(this.order)
      } else {
        EventBus.$emit('notification', {
          type: 'error',
          message: 'Please do correct validation errors',
          action1: { label: 'OK', action: 'close' }
        })
      }
    }
  },
  components: {
    PersonalDetails,
    Shipping,
    Payment,
    OrderReview,
    CartSummary
  },
  mixins: [corePage('Checkout')]
}
</script>

<style lang="scss">
@import '../css/text.scss';

#checkout {
  input[type=text], input[type=email], input[type=tel], select {
    @extend .h4;
    padding: 10px 0;
    border: none;
    border-bottom: 1px solid #BDBDBD;
    width: calc(100% - 35px);
  }
  input::-webkit-input-placeholder {
    color: #BDBDBD;
  }
  input:-moz-placeholder {
    color: #BDBDBD;
  }
  input:focus, select:focus {
    outline: none;
    border-color: black;
    transition: 0.3s all;
  }
  select {
    -moz-appearance: none;
    -webkit-appearance: none;
    border-radius: 0;
    background-color: transparent;
  }
  @media (max-width: 1023px) {
    select {
      max-width: 188px;
    }
  }
  h4 {
    @extend .weight-200;
  }
  .button-disabled {
    opacity: 0.3;
    pointer-events: none;
  }
  .validation-error{
    color: red;
    display: block;
  }
  .number-circle {
    width: 40px;
    height: 40px;
  }
  .section-disabled {
    cursor: not-allowed;
  }
  .section-editable {
    cursor: pointer;
  }
}
</style>
