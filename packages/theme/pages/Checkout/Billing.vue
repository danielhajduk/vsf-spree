<template>
  <ValidationObserver v-slot="{ handleSubmit }">
    <SfHeading
      v-e2e="'billing-heading'"
      :level="3"
      :title="$t('pages.checkout.billing.billing_heading')"
      class="sf-heading--left sf-heading--no-underline title"
    />
    <AddressPicker
      v-if="isAuthenticated && savedAddresses"
      v-model="selectedSavedAddressId"
      :addresses="savedAddresses.addresses"
      :saved-address="checkoutBillingAddress"
    />
    <form @submit.prevent="handleSubmit(handleFormSubmit)">
      <div v-if="!selectedSavedAddressId" class="form">
        <ValidationProvider
          name="firstName"
          rules="required|min:2"
          v-slot="{ errors }"
          slim
        >
          <SfInput
            v-e2e="'billing-firstName'"
            v-model="form.firstName"
            :label="$t('pages.checkout.billing.first_name_label')"
            name="firstName"
            class="form__element form__element--half"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          />
        </ValidationProvider>
        <ValidationProvider
          name="lastName"
          rules="required|min:2"
          v-slot="{ errors }"
          slim
        >
          <SfInput
            v-e2e="'billing-lastName'"
            v-model="form.lastName"
            :label="$t('pages.checkout.billing.last_name_label')"
            name="lastName"
            class="form__element form__element--half form__element--half-even"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          />
        </ValidationProvider>
        <ValidationProvider
          name="streetName"
          rules="required|min:2"
          v-slot="{ errors }"
          slim
        >
          <SfInput
            v-e2e="'billing-streetName'"
            v-model="form.addressLine1"
            :label="$t('pages.checkout.billing.street_label')"
            name="streetName"
            class="form__element"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          />
        </ValidationProvider>
        <SfInput
          v-e2e="'billing-apartment'"
          v-model="form.addressLine2"
          :label="$t('pages.checkout.billing.apartment_label')"
          name="apartment"
          class="form__element"
        />
        <ValidationProvider
          name="city"
          rules="required|min:2"
          v-slot="{ errors }"
          slim
        >
          <SfInput
            v-e2e="'billing-city'"
            v-model="form.city"
            :label="$t('pages.checkout.billing.city_label')"
            name="city"
            class="form__element"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          />
        </ValidationProvider>
        <ValidationProvider
          v-if="states && states.length > 0"
          v-slot="{ errors }"
          name="state"
          rules="required"
          slim
        >
          <SfSelect
            class="form__element form form__select sf-select--underlined"
            v-model="form.state"
            name="state"
            :label="$t('pages.checkout.billing.state_label')"
            :required="isStateRequired"
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          >
            <SfSelectOption
              v-for="{ code, name } in states"
              :key="code"
              :value="code"
            >
              {{ name }}
            </SfSelectOption>
          </SfSelect>
        </ValidationProvider>
        <ValidationProvider
          name="country"
          rules="required|min:2"
          v-slot="{ errors }"
          slim
        >
          <SfSelect
            v-e2e="'billing-country'"
            v-model="form.country"
            :label="$t('pages.checkout.billing.country_label')"
            name="country"
            class="form__element form__element--half form__select sf-select--underlined"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          >
            <SfSelectOption
              v-for="countryOption in countries"
              :key="countryOption.key"
              :value="countryOption.key"
            >
              {{ countryOption.label }}
            </SfSelectOption>
          </SfSelect>
        </ValidationProvider>
        <ValidationProvider
          name="zipCode"
          rules="required|min:2"
          v-slot="{ errors }"
          slim
        >
          <SfInput
            v-e2e="'billing-zipcode'"
            v-model="form.postalCode"
            :label="$t('pages.checkout.billing.postal_code_label')"
            name="zipCode"
            class="form__element form__element--half form__element--half-even"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          />
        </ValidationProvider>
        <ValidationProvider
          name="phone"
          rules="required|digits:9"
          v-slot="{ errors }"
          slim
        >
          <SfInput
            v-e2e="'billing-phone'"
            v-model="form.phone"
            :label="$t('pages.checkout.billing.phone_number_label')"
            name="phone"
            class="form__element form__element--half"
            required
            :valid="!errors[0]"
            :errorMessage="errors[0]"
          />
        </ValidationProvider>
      </div>
      <div class="form">
        <div class="form__action">
          <SfButton
            class="sf-button color-secondary form__back-button"
            type="button"
            @click="router.push(localePath({ name: 'shipping' }))"
          >
            {{ $t('pages.checkout.billing.button_go_back_label') }}
          </SfButton>
          <SfButton
            v-e2e="'continue-to-payment'"
            class="form__action-button"
            type="submit"
          >
            {{ $t('pages.checkout.billing.button_continue_to_payment_label') }}
          </SfButton>
        </div>
      </div>
    </form>
  </ValidationObserver>
</template>

<script>
import {
  SfHeading,
  SfInput,
  SfButton,
  SfSelect,
  SfRadio,
  SfCheckbox
} from '@storefront-ui/vue';
import { ref, watch, computed, onMounted, useRouter } from '@nuxtjs/composition-api';
import { onSSR } from '@vue-storefront/core';
import { useBilling, useCountry, useUser, useUserBilling } from '@vue-storefront/spree';
import { required, min, digits } from 'vee-validate/dist/rules';
import { ValidationProvider, ValidationObserver, extend } from 'vee-validate';
import AddressPicker from '~/components/Checkout/AddressPicker';
import _ from 'lodash';

export default {
  name: 'Billing',
  components: {
    SfHeading,
    SfInput,
    SfButton,
    SfSelect,
    SfRadio,
    SfCheckbox,
    AddressPicker,
    ValidationProvider,
    ValidationObserver
  },
  created() {
    extend('required', {
      ...required,
      message: this.$i18n.t('shared.validation.required')
    });
    extend('min', {
      ...min,
      message: this.$i18n.t('shared.validation.min')
    });
    extend('digits', {
      ...digits,
      message: this.$i18n.t('shared.validation.digits.phone_number')
    });
  },
  setup(_props, { root }) {
    const { billing: checkoutBillingAddress, load, save } = useBilling();
    const { countries, states, load: loadCountries, loadStates } = useCountry();
    const { isAuthenticated } = useUser();
    const { billing: savedAddresses, load: loadSavedAddresses } = useUserBilling();
    const router = useRouter();

    const selectedSavedAddressId = ref(undefined);
    const form = ref({
      firstName: '',
      lastName: '',
      addressLine1: '',
      addressLine2: '',
      city: '',
      state: '',
      country: '',
      postalCode: '',
      phone: null
    });

    const selectedSavedAddress = computed(() => {
      if (!selectedSavedAddressId.value) {
        return undefined;
      }

      return savedAddresses.value.addresses.find(e => e._id === selectedSavedAddressId.value);
    });
    const isStateRequired = computed(() => form.value.country && countries.value.find(e => e.key === form.value.country).stateRequired);

    const handleFormSubmit = async () => {
      if (isAuthenticated.value && selectedSavedAddress.value) {
        await save({ billingDetails: selectedSavedAddress.value });
      } else {
        await save({ billingDetails: form.value });
      }
      router.push(root.localePath('/checkout/payment'));
    };

    const isEqualAddress = (a, b) => {
      const aWithoutId = _.omit(a, ['_id']);
      const bWithoutId = _.omit(b, ['_id']);
      return _.isEqual(aWithoutId, bWithoutId);
    };

    const populateSelectedAddressId = () => {
      if (checkoutBillingAddress.value && savedAddresses.value?.addresses) {
        selectedSavedAddressId.value = savedAddresses.value.addresses.find(e => isEqualAddress(e, checkoutBillingAddress.value))?._id;
      }
    };

    onMounted(async () => {
      await load();
      await loadSavedAddresses();
      await loadCountries();

      if (checkoutBillingAddress.value) {
        form.value = _.omit(checkoutBillingAddress.value, ['_id']);
      }

      populateSelectedAddressId();
    });

    onSSR(async () => {
      await load();
      await loadSavedAddresses();
      await loadCountries();

      if (checkoutBillingAddress.value) {
        form.value = _.omit(checkoutBillingAddress.value, ['_id']);
      }

      populateSelectedAddressId();
    });

    watch(() => form.value.country, async (newCountryValue, oldCountryValue) => {
      if (newCountryValue === oldCountryValue) return;
      form.value.state = null;
      await loadStates(newCountryValue);
    }, {
      immediate: true
    });

    watch(() => isAuthenticated.value, async (isAuthenticatedNow) => {
      if (!isAuthenticatedNow) return;
      await load();
      await loadSavedAddresses();

      if (checkoutBillingAddress.value) {
        form.value = _.omit({...form.value, ...checkoutBillingAddress.value}, ['_id']);
      }

      populateSelectedAddressId();
    });

    return {
      router,
      isAuthenticated,
      form,
      countries,
      states,
      isStateRequired,
      savedAddresses,
      selectedSavedAddressId,
      checkoutBillingAddress,
      handleFormSubmit
    };
  }
};
</script>
<style lang="scss" scoped>
.title {
  margin: var(--spacer-xl) 0 var(--spacer-base) 0;
}
.form {
  &__select {
    display: flex;
    align-items: center;
    --select-option-font-size: var(--font-size--lg);
    ::v-deep .sf-select__dropdown {
      font-size: var(--font-size--lg);
      margin: 0;
      color: var(--c-text);
      font-family: var(--font-family--secondary);
      font-weight: var(--font-weight--normal);
    }
  }
  @include for-desktop {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
  }
  &__element {
    margin: 0 0 var(--spacer-xl) 0;
    @include for-desktop {
      flex: 0 0 100%;
    }
    &--half {
      @include for-desktop {
        flex: 1 1 50%;
      }
      &-even {
        @include for-desktop {
          padding: 0 0 0 var(--spacer-xl);
        }
      }
    }
  }
  &__group {
    display: flex;
    align-items: center;
  }
  &__action {
    @include for-desktop {
      flex: 0 0 100%;
      display: flex;
    }
  }
  &__action-button, &__back-button {
    --button-width: 100%;
    @include for-desktop {
      --button-width: auto;
    }
  }
  &__action-button {
    &--secondary {
      @include for-desktop {
        order: -1;
        --button-margin: 0;
        text-align: left;
      }
    }
     &--add-address {
      width: 100%;
      margin: 0;
      @include for-desktop {
        margin: 0 0 var(--spacer-lg) 0;
        width: auto;
      }
    }
  }
  &__back-button {
    margin: var(--spacer-xl) 0 var(--spacer-sm);
    &:hover {
      color:  white;
    }
    @include for-desktop {
      margin: 0 var(--spacer-xl) 0 0;
    }
  }
  &__back-button {
    margin: 0 0 var(--spacer-sm) 0;
    @include for-desktop {
      margin: 0 var(--spacer-xl) 0 0;
    }
  }
}
.payment-methods {
  @include for-desktop {
    display: flex;
    padding: var(--spacer-lg) 0;
    border: 1px solid var(--c-light);
    border-width: 1px 0;
  }
}
.payment-method {
  --radio-container-align-items: center;
  --ratio-content-margin: 0 0 0 var(--spacer-base);
  --radio-label-font-size: var(--font-base);
  --radio-background: transparent;
  white-space: nowrap;
  border: 1px solid var(--c-light);
  border-width: 1px 0 0 0;
  &:last-child {
    border-width: 1px 0;
  }
  --radio-background: transparent;
  @include for-desktop {
    border: 0;
    --radio-border-radius: 4px;
  }
}
</style>
