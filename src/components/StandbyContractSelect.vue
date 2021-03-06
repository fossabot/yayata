<template lang="pug">
div.contract-select
  .text-md-center 
    strong Select contract
  template(v-if='loaded')
    div.custom-controls-stacked(v-for='(contract, index) in supportContracts') 
      label.custom-control.custom-label
        input(:id='"contract_" + index', :value='contract.id', type='checkbox', v-model='selectedContracts', @click='toggleStandby(contract.id)')
        span.custom-control-description {{ contract.display_label }}
</template>
<script>
import Vue from 'vue';
import store from '../store';
import * as types from '../store/mutation-types';
import moment from 'moment'
import ToastMixin from './mixins/ToastMixin.vue';

export default {
  name: 'standbycontractselect',
  mixins: [ToastMixin],
  data() {
    return {
      selectedContracts: [],
      standbyPerformances: [],
      loaded: false
     }
  },
  created () {
    store.dispatch(types.NINETOFIVER_RELOAD_FILTERED_CONTRACTS, {
      path: '/contracts/support/',
      params: {
        contractuser__user__id: store.getters.user.id
      }
    }).then(() => this.loaded = true)
      .catch((error) => {
        this.showDangerToast('Something went wrong while loading contracts. Check the console for more info.');
        console.error(error);
      });

    store.dispatch(types.NINETOFIVER_API_REQUEST, {
      path: '/my_performances/standby/',
      params: {
        timesheet: this.properties.data.timesheet.id,
        day: moment(this.properties.date).date()
      }
    }).then((res) => {
      this.standbyPerformances = res.data.results;
      this.standbyPerformances.forEach((standbyPerf) => {
        if(this.selectedContracts.findIndex(x => x.id == standbyPerf.contract) === -1) {
          let contract = this.supportContracts.find((contract) => contract.id == standbyPerf.contract);
          this.selectedContracts.push(contract.id);
        }
      });
    }).catch((error) => {
      this.showDangerToast('Something went wrong while creating the standby performance. Check the console for more info.');
      console.log(error);
    });
  },
  methods: {
    //Get whether user is on standby
    getStandbyStatus: function(contractId) {
      return (this.standbyPerformances.findIndex(x => x.contract == contractId) !== -1)
    },

    //Make the call to standby
    toggleStandby: function(contractId) {
      if(this.getStandbyStatus(contractId))
        this.deleteStandby(this.standbyPerformances.find(x => x.contract == contractId));
      else
        this.setStandby(contractId);
    },

    //Delete standbyperformance for specific day
    deleteStandby: function(standby) {
      store.dispatch(
        types.NINETOFIVER_API_REQUEST,
        {
          path: '/my_performances/standby/' + standby.id + '/',
          method: 'DELETE',
          params: {
            id: standby.id
          }
        }).then((delRes) => {
          if(delRes.status == 204) {
            let index = this.standbyPerformances.findIndex(x => x.id == standby.id);
            this.standbyPerformances.splice(index, 1);
            this.showInfoToast('No longer on standby');
            store.dispatch(types.NINETOFIVER_RELOAD_STANDBY_PERFORMANCES);
          }
        }).catch((error) => {
          console.log( error );
          this.showDangerToast('Something went wrong. Check console for more information');  
        });
    },

    //Set standbyperformance for specific day
    setStandby: function(contractId) {
      if(this.properties.data.timesheet) {
        store.dispatch(
          types.NINETOFIVER_API_REQUEST,
          {
            path: '/my_performances/standby/',
            method: 'POST',
            body: {
              timesheet: this.properties.data.timesheet.id,
              contract: contractId,
              day: this.day
            },
            emulateJSON: true,
          }
        ).then((response) => {
          if(response.status == 201) {
            this.standbyPerformances.push(response.data)
            store.dispatch(types.NINETOFIVER_RELOAD_STANDBY_PERFORMANCES);
            this.showSuccessToast('On standby');
          } 
        }).catch((error) => {
          console.log(error);
          this.showDangerToast('Something went wrong. Check console for more information')
        });
      } else {
        //Timesheet was not found, so a new one is made for that date
        store.dispatch(
          types.NINETOFIVER_API_REQUEST,
          {
            path: '/my_timesheets/',
            method: 'POST',
            body: {
              month: moment(this.properties.date).month() + 1,
              year: moment(this.properties.date).year(),
              closed: false
            },
            emulateJSON: true,
          }
        ).then((tsRes) => {
          store.dispatch(
            types.NINETOFIVER_API_REQUEST,
            {
              path: '/my_performances/standby/',
              method: 'POST',
              body: {
                timesheet: tsRes.data.id,
                contract: contractId,
                day: this.day 
              },
              emulateJSON: true,
            }
          ).then((spRes) => {
            if(spRes.status == 201) {
              store.dispatch(types.NINETOFIVER_RELOAD_STANDBY_PERFORMANCES);
              this.showSuccessToast('On standby');
            }
          }).catch((error) => {
            console.log(error);
            this.showDangerToast('Something went wrong. Check console for more information')
          });
        }).catch((error) => {
          console.log(error);
          this.showDangerToast('Something went wrong. Check console for more information')
        });
      }
    }
  },
  props: {
    properties: {
      type: Object,
      default: null,
      validator(value) {
        return (value !== null && value !== undefined && typeof value === 'object')
      }
    }
  },
  computed: {
    supportContracts: function() {
      if(store.getters.filtered_contracts){
        return store.getters.filtered_contracts.filter((contract) => {
          if(contract.active) {
            contract.checked=false;
            return contract;
          }
        });
      }
    },

    day: function() {
      if(this.properties.date){
        return moment(this.properties.date).date()
      }
    }
  }
}
</script>
<style scoped>
.contract-select {
  color: black
}
</style>
