<template>

  <KModal
    :title="$attrs.title || $tr('header')"
    :submitText="coreString('continueAction')"
    :cancelText="coreString('cancelAction')"
    size="medium"
    @submit="handleSubmit"
    @cancel="$emit('cancel')"
  >
    <template>
      <p v-if="filterLODAvailable">
        {{ $tr('lodSubHeader') }}
      </p>
      <p v-if="initialFetchingComplete && !combinedAddresses.length">
        {{ $tr('noDeviceText') }}
      </p>
      <UiAlert
        v-if="uiAlertProps"
        v-show="showUiAlerts"
        :type="uiAlertProps.type"
        :dismissible="false"
      >
        {{ uiAlertProps.text }}
        <KButton
          v-if="requestsFailed"
          appearance="basic-link"
          :text="$tr('refreshDevicesButtonLabel')"
          @click="refreshSavedAddressList"
        />
      </UiAlert>

      <!-- Static Addresses -->
      <template v-for="(a, idx) in savedAddresses">
        <div :key="`div-${idx}`">
          <KRadioButton
            :key="idx"
            v-model="selectedAddressId"
            class="radio-button"
            :value="a.id"
            :label="a.nickname"
            :description="a.base_url"
            :disabled="formDisabled || !isAddressAvailable(a.id)"
          />
          <KButton
            v-if="!hideSavedAddresses"
            :key="`forget-${idx}`"
            :text="$tr('forgetDeviceButtonLabel')"
            appearance="basic-link"
            @click="removeSavedAddress(a.id)"
          />
        </div>
      </template>

      <hr
        v-if="!hideSavedAddresses && discoveredAddresses.length > 0"
        :style="{ border: 0, borderBottom: `1px solid ${$themeTokens.fineLine}` }"
      >

      <!-- Dynamic Addresses -->
      <template v-for="d in discoveredAddresses">
        <div :key="`div-${d.id}`">
          <KRadioButton
            :key="d.id"
            v-model="selectedAddressId"
            class="radio-button"
            :value="d.instance_id"
            :label="formatNameAndId(d.device_name, d.id)"
            :description="formatBaseAddress(d)"
            :disabled="formDisabled || discoveryFailed || !isAddressAvailable(d.id)"
          />
        </div>
      </template>
    </template>

    <slot name="underbuttons"></slot>

    <template #actions>
      <KFixedGrid class="actions" numCols="4">
        <KFixedGridItem span="1">
          <transition name="spinner-fade">
            <div v-if="discoveringPeers">
              <KLabeledIcon>
                <template #icon>
                  <KCircularLoader :size="16" :stroke="6" class="loader" />
                </template>
              </KLabeledIcon>
            </div>
          </transition>
        </KFixedGridItem>
        <KFixedGridItem span="3" alignment="right">
          <KButtonGroup style="margin-top: 8px;">
            <KButton
              :text="coreString('cancelAction')"
              appearance="flat-button"
              :disabled="formDisabled"
              @click="$emit('cancel')"
            />
            <KButton
              :text="coreString('continueAction')"
              :primary="true"
              :disabled="formDisabled || submitDisabled"
              type="submit"
            />
          </KButtonGroup>
        </KFixedGridItem>
      </KFixedGrid>
    </template>

    <KButton
      v-show="!newAddressButtonDisabled && !formDisabled"
      class="new-address-button"
      :text="$tr('newDeviceButtonLabel')"
      appearance="basic-link"
      @click="$emit('click_add_address')"
    />


  </KModal>

</template>


<script>

  import { computed } from 'kolibri.lib.vueCompositionApi';
  import { useLocalStorage } from '@vueuse/core';
  import find from 'lodash/find';
  import UiAlert from 'kolibri-design-system/lib/keen/UiAlert';
  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import commonSyncElements from 'kolibri.coreVue.mixins.commonSyncElements';
  import useDynamicAddresses from './useDynamicAddresses.js';
  import useSavedAddresses from './useSavedAddresses.js';

  export default {
    name: 'SelectDeviceForm',
    components: {
      UiAlert,
    },
    mixins: [commonCoreStrings, commonSyncElements],
    setup(props, context) {
      const {
        addresses: discoveredAddresses,
        discoveringPeers,
        discoveryFailed,
        discoveredAddressesInitiallyFetched,
      } = useDynamicAddresses(props);

      const {
        addresses: savedAddresses,
        removeSavedAddress,
        refreshSavedAddressList,
        savedAddressesInitiallyFetched,
        requestsFailed,
        deletingAddress,
        fetchingAddresses,
      } = useSavedAddresses(props, context);

      const combinedAddresses = computed(() => {
        return [...savedAddresses.value, ...discoveredAddresses.value];
      });

      const initialFetchingComplete = computed(() => {
        return savedAddressesInitiallyFetched.value && discoveredAddressesInitiallyFetched.value;
      });

      const storageAddressId = useLocalStorage('kolibri-lastSelectedNetworkLocationId', '');

      return {
        combinedAddresses,
        initialFetchingComplete,
        discoveredAddresses,
        discoveringPeers,
        discoveryFailed,
        discoveredAddressesInitiallyFetched,
        savedAddresses,
        savedAddressesInitiallyFetched,
        removeSavedAddress,
        refreshSavedAddressList,
        requestsFailed,
        deletingAddress,
        fetchingAddresses,
        storageAddressId,
      };
    },
    props: {
      // eslint-disable-next-line kolibri/vue-no-unused-properties
      discoverySpinnerTime: { type: Number, default: 2500 },
      // Facility filter only needed on SyncFacilityModalGroup
      // eslint-disable-next-line kolibri/vue-no-unused-properties
      filterByFacilityId: {
        type: String,
        default: null,
      },
      // Channel filter only needed on ManageContentPage/SelectNetworkDeviceModal
      // eslint-disable-next-line kolibri/vue-no-unused-properties
      filterByChannelId: {
        type: String,
        default: null,
      },
      // Hides "New address" button and other saved locations
      hideSavedAddresses: {
        type: Boolean,
        default: false,
      },
      // If an ID is provided, that address's radio button will be automatically selected
      selectedId: {
        type: String,
        default: null,
      },
      // Disables all the form controls
      formDisabled: {
        type: Boolean,
        default: false,
      },
      filterLODAvailable: {
        type: Boolean,
        default: false,
      },
    },
    data() {
      return {
        availableAddressIds: [],
        selectedAddressId: '',
        showUiAlerts: false,
      };
    },
    computed: {
      isAddressAvailable() {
        return function(addressId) {
          return Boolean(this.availableAddressIds.find(id => id === addressId));
        };
      },
      submitDisabled() {
        return Boolean(
          this.selectedAddressId === '' ||
            this.fetchingAddresses & !this.filterLODAvailable ||
            this.deletingAddress ||
            this.discoveryFailed ||
            this.availableAddressIds.length === 0
        );
      },
      newAddressButtonDisabled() {
        return this.filterLODAvailable || this.hideSavedAddresses || this.fetchingAddresses;
      },
      uiAlertProps() {
        let text;
        if (this.fetchingFailed) {
          text = this.$tr('fetchingFailedText');
        }
        if (this.discoveryFailed) {
          text = this.$tr('fetchingFailedText');
        }
        if (this.deletingFailed) {
          text = this.$tr('deletingFailedText');
        }
        return text ? { text, type: 'error' } : null;
      },
    },
    watch: {
      selectedAddressId(newVal) {
        this.storageAddressId = newVal;
      },
      combinedAddresses(addrs) {
        this.availableAddressIds = addrs
          .filter(
            address =>
              address.available &&
              (this.$route.path === '/content' || address.application === 'kolibri')
          )
          .map(address => address.id);
        if (!this.availableAddressIds.includes(this.selectedAddressId)) {
          this.selectedAddressId = '';
        }
        if (!this.selectedAddressId) {
          this.resetSelectedAddress();
        }
      },
    },
    mounted() {
      // Wait a little bit of time before showing UI alerts so there is no flash
      // if data comes back quickly
      setTimeout(() => {
        this.showUiAlerts = true;
      }, 100);
    },
    methods: {
      formatBaseAddress(device) {
        const url = device.base_url;
        if (this.filterLODAvailable) {
          const version = device.kolibri_version
            .split('.')
            .slice(0, 3)
            .join('.');
          return `${url}, Kolibri ${version}`;
        } else return url;
      },
      resetSelectedAddress() {
        if (this.availableAddressIds.length !== 0) {
          const selectedId = this.selectedId || this.storageAddressId || this.selectedAddressId;
          this.selectedAddressId =
            this.availableAddressIds.find(id => id === selectedId) || this.availableAddressIds[0];
        } else {
          this.selectedAddressId = '';
        }
      },
      handleSubmit() {
        if (this.selectedAddressId) {
          const match = find(this.combinedAddresses, { id: this.selectedAddressId });
          match.isDynamic = Boolean(find(this.discoveredAddresses, { id: this.selectedAddressId }));
          this.$emit('submit', match);
        }
      },
    },
    $trs: {
      deletingFailedText: {
        message: 'There was a problem removing this device',
        context:
          'Error message that displays when an admin attempts to remove a device, but is unable to do so.',
      },
      fetchingFailedText: {
        message: 'There was a problem getting the available devices',
        context:
          'Error message that displays when an admin attempts to find a device, but the device is not found.',
      },
      forgetDeviceButtonLabel: {
        message: 'Remove',
        context:
          'Removes a device from the list of devices which have been registered in the Device > Facilities section.',
      },
      header: {
        message: 'Select device',
        context:
          "In the Device > Facilities section, you select the 'SYNC' option to choose the device you want to sync from.\n\nYou do this in the 'Select device' section which displays a list of devices.",
      },
      newDeviceButtonLabel: {
        message: 'Add new device',
        context:
          'The "Add new device" link appears in the \'Select device\' screen. This option allows you to add a new device from which to sync data.',
      },
      lodSubHeader: {
        message: 'Select a device with Kolibri version 0.15 to import learner user accounts',
        context:
          "In the first startup wizard, when you select to 'Import one or more user accounts from an existing facility' option to choose the device you want to sync from.\n\nYou do this in the 'Select device' section which displays a list of devices.",
      },
      noDeviceText: {
        message: 'There are no devices yet',
        context:
          "This message displays when there are no devices to sync with.\n\nIt appears when selecting 'SYNC' in the Device > Facilities section if there are no devices.",
      },
      refreshDevicesButtonLabel: {
        message: 'Refresh devices',
        context:
          'This message displays if there was a problem getting the devices. It allows the user to refresh the application to be able to see all the devices available.',
      },
    },
  };

</script>


<style lang="scss" scoped>

  .new-address-button {
    margin: 0 0 1em;
  }

  .radio-button {
    display: inline-block;
    width: 75%;
  }

  .spinner-fade-leave-active,
  .spinner-fade-enter-active {
    transition: opacity 0.5s;
  }

  .spinnner-fade-enter-to,
  .spinner-fade-leave {
    opacity: 1;
  }

  .spinner-fade-enter,
  .spinner-fade-leave-to {
    opacity: 0;
  }

  .ui-progress-circular {
    display: inline-block;
    margin-right: 2px;
    margin-bottom: 2px;
    vertical-align: middle;
  }

  .loader {
    position: relative;
    top: 12px;
  }

</style>