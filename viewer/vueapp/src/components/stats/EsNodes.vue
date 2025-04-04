<!--
Copyright Yahoo Inc.
SPDX-License-Identifier: Apache-2.0
-->
<template>

  <div class="container-fluid mt-2">

    <arkime-loading v-if="initialLoading && !error">
    </arkime-loading>

    <arkime-error v-if="error"
      :message="error">
    </arkime-error>

    <div v-show="!error">

      <arkime-paging v-if="stats"
        class="mt-1 ml-2"
        :info-only="true"
        :records-total="recordsTotal"
        :records-filtered="filteredStats.length">
      </arkime-paging>

      <arkime-table
        id="esNodesTable"
        v-on:toggle-data-node-only="showOnlyDataNodes = !showOnlyDataNodes"
        :data="filteredStats"
        :loadData="loadData"
        :columns="columns"
        :no-results="true"
        :show-avg-tot="true"
        :action-column="true"
        :desc="query.desc"
        :sortField="query.sortField"
        :no-results-msg="`No results match your search.${cluster ? 'Try selecting a different cluster.' : ''}`"
        page="esNodes"
        table-animation="list"
        table-state-name="esNodesCols"
        table-widths-state-name="esNodesColWidths"
        table-classes="table-sm table-hover text-right small mt-2">
        <template slot="actions"
          slot-scope="{ item }"
          class="display-inline">
          <span class="no-wrap">
            <b-dropdown size="sm"
              class="row-actions-btn"
              v-has-role="{user:user,roles:'arkimeAdmin'}">
              <b-dropdown-item v-if="!item.nodeExcluded"
                @click="exclude('name', item)">
                Exclude node {{ item.name }}
              </b-dropdown-item>
              <b-dropdown-item v-else
                @click="include('name', item)">
                Include node {{ item.name }}
              </b-dropdown-item>
              <b-dropdown-item v-if="!item.ipExcluded"
                @click="exclude('ip', item)">
                Exclude IP {{ item.ip }}
              </b-dropdown-item>
              <b-dropdown-item v-else
                @click="include('ip', item)">
                Include IP {{ item.ip }}
              </b-dropdown-item>
            </b-dropdown>
            <span class="ml-1">
              <span class="node-badge badge badge-primary badge-pill"
                :class="{'show-badge cursor-help': item.roles.indexOf('master') > -1, 'badge-master':item.isMaster}">
                <span v-if="item.roles.indexOf('master') > -1"
                  title="Master Node"
                  v-b-tooltip.hover>
                  M
                </span>
                <span v-else>&nbsp;</span>
              </span>
              <span class="node-badge badge badge-primary badge-pill"
                style="padding-left:.5rem;"
                :class="{'show-badge cursor-help': item.roles.some(role => role.startsWith('data'))}">
                <span v-if="item.roles.some(role => role.startsWith('data'))"
                  title="Data Node"
                  v-b-tooltip.hover>
                  D
                </span>
                <span v-else>&nbsp;</span>
              </span>
            </span>
          </span>
        </template>
      </arkime-table>

    </div>

  </div>

</template>

<script>
import Utils from '../utils/utils';
import ArkimeTable from '../utils/Table';
import ArkimeError from '../utils/Error';
import ArkimeLoading from '../utils/Loading';
import ArkimePaging from '../utils/Pagination';

let reqPromise; // promise returned from setInterval for recurring requests
let respondedAt; // the time that the last data load successfully responded

export default {
  name: 'EsStats',
  props: [
    'dataInterval',
    'refreshData',
    'searchTerm',
    'cluster'
  ],
  components: {
    ArkimeTable,
    ArkimeError,
    ArkimePaging,
    ArkimeLoading
  },
  data: function () {
    return {
      error: '',
      showOnlyDataNodes: false,
      initialLoading: true,
      stats: null,
      recordsTotal: null,
      query: {
        filter: this.searchTerm || undefined,
        sortField: 'nodeName',
        desc: false,
        cluster: this.cluster || undefined
      },
      columns: [ // es stats table columns
        // default columns
        { id: 'name', name: 'Name', classes: 'text-left', sort: 'nodeName', doStats: false, default: true, width: 120 },
        { id: 'docs', name: 'Documents', sort: 'docs', doStats: true, default: true, width: 120, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.docs); } },
        { id: 'storeSize', name: 'Disk Used', sort: 'storeSize', doStats: true, default: true, width: 105, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.storeSize); } },
        { id: 'freeSize', name: 'Disk Free', sort: 'freeSize', doStats: true, default: true, width: 100, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.freeSize); } },
        { id: 'heapSize', name: 'Heap Size', sort: 'heapSize', doStats: true, default: true, width: 105, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.heapSize); } },
        { id: 'load', name: 'OS Load', sort: 'load', doStats: true, default: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.load, 2); } },
        { id: 'cpu', name: 'CPU', sort: 'cpu', doStats: true, default: true, width: 80, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.cpu, 1) + '%'; } },
        { id: 'read', name: 'Read/s', sort: 'read', doStats: true, default: true, width: 90, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.read); } },
        { id: 'write', name: 'Write/s', sort: 'write', doStats: true, default: true, width: 90, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.write); } },
        { id: 'searches', name: 'Search/s', sort: 'searches', doStats: true, width: 100, default: true, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.searches); } },
        // all the rest of the available stats
        { id: 'ip', name: 'IP', sort: 'ip', doStats: false, width: 100 },
        { id: 'ipExcluded', name: 'IP Excluded', sort: 'ipExcluded', doStats: false, width: 100 },
        { id: 'nodeExcluded', name: 'Node Excluded', classes: 'text-left', sort: 'nodeExcluded', doStats: false, width: 125 },
        { id: 'nonHeapSize', name: 'Non Heap Size', sort: 'nonHeapSize', doStats: false, width: 100, dataFunction: (item) => { return this.$options.filters.humanReadableBytes(item.nonHeapSize); } },
        { id: 'searchesTime', name: 'Search Time', sort: 'searchesTime', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.searchesTime); } },
        { id: 'writesRejectedDelta', name: 'Write Tasks Rejected/s', sort: 'writesRejectedDelta', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesRejectedDelta); } },
        { id: 'writesCompleted', name: 'Write Tasks Completed', sort: 'writesCompleted', doStats: true, width: 100, canClear: true, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesCompleted); } },
        { id: 'writesCompletedDelta', name: 'Write Tasks Completed/s', sort: 'writesCompletedDelta', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesCompletedDelta); } },
        { id: 'writesQueueSize', name: 'Write Tasks Q Limit', sort: 'writesQueueSize', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesQueueSize); } },
        { id: 'molochtype', name: 'Hot/Warm', sort: 'molochtype', doStats: false, width: 100 },
        { id: 'molochzone', name: 'Zone', sort: 'molochzone', doStats: false, width: 100 },
        { id: 'shards', name: 'Shards', sort: 'shards', doStats: true, default: false, width: 80, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.shards); } },
        { id: 'segments', name: 'Segments', sort: 'segments', doStats: true, default: false, width: 100, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.segments); } },
        { id: 'uptime', name: 'Uptime', sort: 'uptime', doStats: true, width: 100, dataFunction: (item) => { return this.$options.filters.moment(Date.now() - (item.uptime * 60 * 1000), 'from', true); } },
        { id: 'version', name: 'Version', sort: 'version', doStats: false, width: 100 },
        { id: 'writesRejected', name: 'Write Tasks Rejected', sort: 'writesRejected', doStats: true, width: 100, default: false, canClear: true, dataFunction: (item) => { return this.$options.filters.roundCommaString(item.writesRejected); } }
      ]
    };
  },
  computed: {
    loading: {
      get: function () {
        return this.$store.state.loadingData;
      },
      set: function (newValue) {
        this.$store.commit('setLoadingData', newValue);
      }
    },
    filteredStats: function () {
      if (this.showOnlyDataNodes) {
        return this.stats.filter(s => s.roles.some(role => role.startsWith('data')));
      }
      return this.stats;
    },
    user () {
      return this.$store.state.user;
    }
  },
  watch: {
    dataInterval: function () {
      if (reqPromise) { // cancel the interval and reset it if necessary
        clearInterval(reqPromise);
        reqPromise = null;

        if (this.dataInterval === '0') { return; }

        this.setRequestInterval();
      } else if (this.dataInterval !== '0') {
        this.loadData();
        this.setRequestInterval();
      }
    },
    refreshData: function () {
      if (this.refreshData) {
        this.loadData();
      }
    },
    cluster: function (newValue) {
      this.query.cluster = this.cluster;
      this.loadData();
    }
  },
  created: function () {
    // set a recurring server req if necessary
    if (this.dataInterval !== '0') {
      this.setRequestInterval();
    }
  },
  methods: {
    /* exposed page functions ------------------------------------ */
    exclude: function (type, column) {
      if (!Utils.checkClusterSelection(this.query.cluster, this.$store.state.esCluster.availableCluster.active, this).valid) {
        return;
      }

      this.$http.post(`api/esshards/${type}/${column[type]}/exclude`, {}, { params: { cluster: this.query.cluster } })
        .then((response) => {
          if (type === 'name') {
            column.nodeExcluded = true;
          } else {
            column.ipExcluded = true;
          }
        }, (error) => {
          this.error = error.text || error;
        });
    },
    include: function (type, column) {
      if (!Utils.checkClusterSelection(this.query.cluster, this.$store.state.esCluster.availableCluster.active, this).valid) {
        return;
      }

      this.$http.post(`api/esshards/${type}/${column[type]}/include`, {}, { params: { cluster: this.query.cluster } })
        .then((response) => {
          if (type === 'name') {
            column.nodeExcluded = false;
          } else {
            column.ipExcluded = false;
          }
        }, (error) => {
          this.error = error.text || error;
        });
    },
    /* helper functions ------------------------------------------ */
    setRequestInterval: function () {
      reqPromise = setInterval(() => {
        if (respondedAt && Date.now() - respondedAt >= parseInt(this.dataInterval)) {
          this.loadData();
        }
      }, 500);
    },
    loadData: function (sortField, desc) {
      if (!Utils.checkClusterSelection(this.query.cluster, this.$store.state.esCluster.availableCluster.active, this).valid) {
        return;
      }

      this.loading = true;
      respondedAt = undefined;

      this.query.filter = this.searchTerm;

      if (desc !== undefined) { this.query.desc = desc; }
      if (sortField) { this.query.sortField = sortField; }

      this.$http.get('esstats.json', { params: this.query })
        .then((response) => {
          respondedAt = Date.now();
          this.error = '';
          this.loading = false;
          this.initialLoading = false;
          this.stats = response.data.data;
          this.recordsTotal = response.data.recordsTotal;
        }, (error) => {
          respondedAt = undefined;
          this.loading = false;
          this.initialLoading = false;
          this.error = error.text || error;
        });
    }
  },
  beforeDestroy: function () {
    if (reqPromise) {
      clearInterval(reqPromise);
      reqPromise = null;
    }
  }
};
</script>

<style scoped>
td {
  white-space: nowrap;
}
tr.bold {
  font-weight: bold;
}
table.table tr.border-bottom-bold > td {
  border-bottom: 2px solid #dee2e6;
}
table.table tr.border-top-bold > td {
  border-top: 2px solid #dee2e6;
}

.table .hover-menu:hover .btn-group {
  visibility: visible;
}
.table .hover-menu .btn-group {
  visibility: hidden;
}

.node-badge {
  opacity: 0;
  width: 1.6rem;
  line-height: 1.2;
  font-size: 0.75rem;
  background-color: var(--color-primary);

}
.node-badge.show-badge {
  opacity: 1;
}
.badge-master {
  background-color: var(--color-quaternary);
}
</style>
