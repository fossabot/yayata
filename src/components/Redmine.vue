<template lang="pug">
div.row
  .col
    .row
      .col
        h3 Redmine
        p.subtitle Overview of external entries
    template(v-if='user && user.userinfo.redmine_id === "None"')
      .alert.alert-danger.text-md-center No redmine id found for the current user.
    template(v-else)
      .row.mb-3
        .col
          .btn-group(role='group')
            button.btn.btn-outline-dark.dropdown-toggle#btnGroupDropContract(type='button' data-toggle='dropdown' aria-haspopup='true' aria-expanded='false') {{ selectedContract.displayLabel }}
            .dropdown-menu(aria-labelledby='btnGroupDropContract')
              a.dropdown-item(v-for='contract in contracts' @click='selectContract(contract)') {{ contract.display_label }}
          .btn-group.pull-right
            button.btn.btn-outline-primary(@click='importToYayata()') Import
      .row.mb-3
        .col
          h3.text-center Not imported
          table.table.table-striped
            thead
              tr
                th Activity
                th Date
                th Hours
                th Issue
                th Comments
                th 
                  b-form-checkbox(v-model='checkedStateImported') &nbsp;
            tbody(v-if='contractTimeEntries')
              template(v-for='timeEntry in contractTimeEntries')
                template(v-if='!getImportStatus(timeEntry)')
                  tr(class='timesheetPending ? text-muted : ""')
                    td {{ timeEntry.activity.name }}
                    td {{ timeEntry.spent_on }}
                    td {{ timeEntry.hours }}
                    td(v-if='timeEntry.issue') 
                      a(:href="'https://redmine.inuits.eu/issues/' + timeEntry.issue.id") {{ timeEntry.issue.id }}
                    td(v-else) N.A.
                    td {{ timeEntry.comments }}
                    td
                      b-form-checkbox(v-model='timeEntry.checked', :disabled='timesheetPending(timeEntry)') &nbsp;
      .alert.alert-info.text-md-center(v-if='notImportedTimeEntries && notImportedTimeEntries.length == 0') No redmine time entries.
      .row.mb-3
        .col
          h3.text-md-center Imported
          table.table.table-striped
            thead
              tr
                th Activity
                th Date
                th Hours
                th Issue
                th Comments
                th 
                  b-form-checkbox(v-model='checkedStateNotImported') &nbsp;
            tbody(v-if='contractTimeEntries')
              template(v-for='timeEntry in contractTimeEntries')
                template(v-if='getImportStatus(timeEntry)')
                  tr
                    td(:class='checkDiff(timeEntry)') {{ timeEntry.activity.name }}
                    td {{ timeEntry.spent_on }}
                    td(:class='checkDiffHours(timeEntry)') {{ timeEntry.hours }}
                    td(:class='checkDiff(timeEntry)') 
                      template(v-if='timeEntry.issue')
                        a(:href="'https://redmine.inuits.eu/issues/' + timeEntry.issue.id", target='_blank') {{ timeEntry.issue.id }}
                      span(v-else) N.A.
                    td(:class='checkDiffComments(timeEntry)') {{ timeEntry.comments }}
                    td(:class='checkDiff(timeEntry)')
                      b-form-checkbox(v-model='timeEntry.checked') &nbsp;
      .alert.alert-info.text-md-center(v-if='!selectedContract.value') Please select a contract
</template>
<script>
import store from '../store';
import moment from 'moment';
import * as types from '../store/mutation-types'
import ToastMixin from './mixins/ToastMixin.vue';

export default {
  name: 'redmine',
  mixins: [ToastMixin],
  data() {
    return {
      selectedContract: {
        displayLabel: 'Select Contract',
        value: null
      },
      contractTimeEntries: null,
      checkedStateImported: true, 
      checkedStateNotImported: true, 
    }
  },

  created: function() {
    if(!store.getters.timesheets) {
      store.dispatch(types.NINETOFIVER_RELOAD_TIMESHEETS);
    }

    this.reload_time_entries();
    // Reload filtered contracts for this user
    store.dispatch(types.NINETOFIVER_RELOAD_FILTERED_CONTRACTS, {
      params: {
        contractuser__user__id: store.getters.user.id
      }
    });
    if(!store.getters.activity_performances) {
      store.dispatch(types.NINETOFIVER_RELOAD_ACTIVITY_PERFORMANCES);
    }
    if(!store.getters.contract_roles) {
      store.dispatch(types.NINETOFIVER_RELOAD_CONTRACT_ROLES);
    }
    if(!store.getters.contract_users) {
      store.dispatch(types.NINETOFIVER_RELOAD_CONTRACT_USERS);
    }
  },

  computed: {
    user: function() {
      if(store.getters.user) {
        return store.getters.user;
      }
    },

    contracts: function() {
      if(store.getters.filtered_contracts) {
        let contracts = store.getters.filtered_contracts;
        return contracts.sort((a, b) => {
          return a.display_label < b.display_label ? -1 : a.display_label > b.display_label ? 1 : 0;
        });
      }
    },

    contractRoles: function() {
      if(store.getters.contract_roles && store.getters.contract_users && this.selectedContract) {
        let contract_users = store.getters.contract_users.filter((cu) => {
          return (cu.user === store.getters.user.id && cu.contract === this.selectedContract.value)
        })
        let contract_roles = [];
        contract_users.forEach( cu => {
          contract_roles.push(store.getters.contract_roles.find( cr => cr.id === cu.contract_role));
        });
        
        return contract_roles;
      }
    },

    timeEntries: function() {
      if(store.getters.redmine_time_entries) {
        let timeEntries = store.getters.redmine_time_entries;
        timeEntries.forEach((te) => {
          te.checked = !this.timesheetPending(te);
          // te.checked = true;
        });
        return timeEntries
      }
    },

    importedTimeEntries: function() {
      if(this.contractTimeEntries) {
        return this.contractTimeEntries.filter((te) => {
          if(this.getImportStatus(te)){
            return te;
          }
        });
      }
    },

    notImportedTimeEntries: function() {
      if(this.contractTimeEntries) {
        return this.contractTimeEntries.filter((te) => {
          if(!this.getImportStatus(te)){
            return te;
          }
        });
      }
    },

    timesheet: function() {
      return store.getters.timesheets.find(x => 
          x.month == moment().month() + 1
          &&
          x.year == moment().year()
        );
    },

    performances: function() {
      if(store.getters.activity_performances && this.timesheet) {
        return store.getters.activity_performances.filter((p) => p.redmine_id && p.timesheet === this.timesheet.id)
      }
    }
  },

  watch: {
    // Watch checkedStateImported and update the checked status of all time entries that are imported.
    checkedStateImported: function(newCheckedState) {
      this.toggleAllImported();
    },

    // Watch checkedStateNotImported and update the checked status of all time entries that are not imported.
    checkedStateNotImported: function(newCheckedState) {
      this.toggleAllNotImported();
    },

  },

  methods: {
    timesheetPending: function(te) {
     let timesheet = store.getters.timesheets.find(ts => {
        return ts.month == moment(te.spent_on).month() + 1;
      });
      if(timesheet) {
        return timesheet.status === 'PENDING';
      }
      return false;
    }, 
    toggleAllImported() {
      this.contractTimeEntries.forEach((te) => {
        if(!this.getImportStatus(te)){
          te.checked = this.checkedStateImported
        }
      });
    },

    toggleAllNotImported() {
      this.contractTimeEntries.forEach((te) => {
        if(this.getImportStatus(te)) {
          te.checked = this.checkedStateNotImported;
        }
      });
    },

    selectContract: function(contract) {
      this.selectedContract.displayLabel = contract.display_label;
      this.selectedContract.value = contract.id;

      // // Reset checked states
      this.checkedStateImported = true;
      this.checkedStateNotImported = true;

      // Filter contract timeEntries on contract 
      this.contractTimeEntries = this.timeEntries.filter((te) => {
        return contract.redmine_id && te.project.id === contract.redmine_id;
      });

      // Reset checked for each contractTimeEntry
      this.toggleAllImported();
      this.toggleAllNotImported();
    },
    
    submitTimeEntries: function() {
      // Create a new timesheet if timesheet is undefined
      if(!this.timesheet) {
        this.createTimesheet().then((response) => {
          if(response.status === 201){
            this.importToYayata();
          } else {
            console.log(response);
            this.showDangerToast('Could not create a new timesheet, check the console for more information.');
          }
        });
      } else {
        this.importToYayata();
      }
    },

    importToYayata: async function() {
      let success = true;
      let x = await this.contractTimeEntries.forEach((te) => {
        // Check if Time Entry has been imported already and if it has been updated.
        if(!this.timesheetPending(te)) {

          if(this.getImportStatus(te)){
            // Patch performance
            if(te.checked && this.checkDiff(te)) {
              let performance = this.performances.find((perf) => perf.redmine_id === te.id);
              let body = {
                timesheet: performance.timesheet,
                day: performance.day,
                duration: te.hours,
                description: te.comments,
                performance_type: performance.performance_type,
                contract: performance.contract,
                contract_role: performance.contract_role,
                redmine_id: performance.redmine_id
              }
              store.dispatch(types.NINETOFIVER_API_REQUEST, {
                path: '/my_performances/activity/' + performance.id + '/',
                method: 'PATCH',
                body: body,
                emulateJSON: true,
              }).then((res) => {
              }).catch((error) => {
                console.log(error);
                success = false;
              })
            }
          } else {
            // Create a new performance for each selected time entry
            if(te.checked) {
              let contract_id = this.contracts.find((c) => te.project.id === c.redmine_id).id;
              let body = {
                timesheet: this.timesheet.id,
                day: moment(te.spent_on).date(),
                duration: te.hours * 1,
                description: te.comments,
                performance_type: 1,
                contract: contract_id,
                contract_role: this.contractRoles[0].id,
                redmine_id: te.id
              }
              // Create new performance
              store.dispatch(types.NINETOFIVER_API_REQUEST, {
                path: '/my_performances/activity/',
                method: 'POST',
                body: body,
                emulateJSON: true,
              }).then((res) => {
              }).catch((error) => {
                console.log(error);
                success = false;
              });
            }
          };
        } else {
          this.showDangerToast("You can't import time entries into pending timesheets.")  
        }
      });
      let message = success ? 'Time entries imported!' : 'Something went wrong.';
      success ? this.showSuccessToast(message) : this.showDangerToast(message);
      this.reload_time_entries();
      store.dispatch(types.NINETOFIVER_RELOAD_ACTIVITY_PERFORMANCES);
    },

    createTimesheet: function() {
      return store.dispatch(types.NINETOFIVER_API_REQUEST, {
        path: '/my_timesheets/',
        method: 'POST',
        params: {
          year: moment().year(),
          month: moment().month() + 1
        }
      });
    },

    getImportStatus(timeEntry) {
      let performance = this.performances.find((p) => {
        return p.redmine_id === timeEntry.id
      })
      return !!performance;
    },

    checkDiff(timeEntry) {
      // Check if the time entry has been updated.
      let performance = this.performances.find((perf) => perf.redmine_id === timeEntry.id);
      return timeEntry.hours * 1 != performance.duration *1 ? true : timeEntry.comments !== performance.description ? true : false;
    },

    checkDiffHours(timeEntry) {
      // Check if the time entry hours have been updated.
      let performance = this.performances.find((perf) => perf.redmine_id === timeEntry.id);
      return timeEntry.hours * 1 == performance.duration * 1 ? '' : 'diff';
    },

    checkDiffComments(timeEntry) {
      // Check if the time entry comments have been updated.
      let performance = this.performances.find((perf) => perf.redmine_id === timeEntry.id);
      return timeEntry.comments === performance.description ? '' : 'diff';
    },

    reload_time_entries() {
      if(store.getters.user.userinfo.redmine_id !== 'None') {
        // Reload redmine time entries for this user and this month
        store.dispatch(types.NINETOFIVER_RELOAD_REDMINE_TIME_ENTRIES, {
          params: {
            filter_imported: false
          }
        });
      }
    },
  }
}
</script>
<style>
.diff {
  background: orange;
}
</style>
