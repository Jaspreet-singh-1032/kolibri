<template>

  <div>
    <SelectSyncSourceModal
      v-if="atSelectSource"
      :formDisabled="syncSubmitDisabled"
      @submit="handleSourceSubmit"
      @cancel="closeModal()"
    />

    <SelectDeviceModalGroup
      v-else-if="atSelectAddress"
      :filterByFacilityId="facilityForSync.id"
      :selectAddressDisabled="syncSubmitDisabled"
      @submit="handleAddressSubmit"
      @cancel="closeModal()"
    />
  </div>

</template>


<script>

  import commonCoreStrings from 'kolibri.coreVue.mixins.commonCoreStrings';
  import commonSyncElements from 'kolibri.coreVue.mixins.commonSyncElements';
  import SelectDeviceModalGroup from './SelectDeviceModalGroup';
  import SelectSyncSourceModal from './SelectSyncSourceModal';

  const Steps = Object.freeze({
    SELECT_SOURCE: 'SELECT_SOURCE',
    SELECT_ADDRESS: 'SELECT_ADDRESS',
  });

  export default {
    name: 'SyncFacilityModalGroup',
    components: {
      SelectSyncSourceModal,
      SelectDeviceModalGroup,
    },
    mixins: [commonCoreStrings, commonSyncElements],
    props: {
      // If facility has not been KDP-registered, skip to SelectDeviceForm
      // and use facility ID to filter the selectable addresses
      facilityForSync: {
        type: Object,
        required: true,
      },
    },
    data() {
      return {
        step: this.facilityForSync.dataset.registered ? Steps.SELECT_SOURCE : Steps.SELECT_ADDRESS,
        syncSubmitDisabled: false,
      };
    },
    computed: {
      atSelectSource() {
        return this.step === Steps.SELECT_SOURCE;
      },
      atSelectAddress() {
        return this.step === Steps.SELECT_ADDRESS;
      },
    },
    methods: {
      handleSourceSubmit(data) {
        if (data.source === 'PEER') {
          this.step = Steps.SELECT_ADDRESS;
        } else {
          this.startKdpSync();
        }
      },
      handleAddressSubmit(data) {
        if (!data.device_name) {
          data.device_name = data.nickname;
        }
        this.startPeerSync(data);
      },
      closeModal() {
        this.$emit('close');
      },
      startKdpSync() {
        this.syncSubmitDisabled = true;
        this.startKdpSyncTask(this.facilityForSync.id)
          .then(task => {
            this.$emit('success', task.id);
          })
          .catch(() => {
            this.$emit('failure');
          });
      },
      startPeerSync(peerData) {
        this.syncSubmitDisabled = true;
        this.startPeerSyncTask({
          facility: this.facilityForSync.id,
          device_id: peerData.id,
        })
          .then(task => {
            this.$emit('success', task.id);
          })
          .catch(() => {
            this.$emit('failure');
          });
      },
    },
  };

</script>


<style lang="scss" scoped></style>
