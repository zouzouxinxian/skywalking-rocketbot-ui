<!-- Licensed to the Apache Software Foundation (ASF) under one or more
contributor license agreements.  See the NOTICE file distributed with
this work for additional information regarding copyright ownership.
The ASF licenses this file to You under the Apache License, Version 2.0
(the "License"); you may not use this file except in compliance with
the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License. -->

<template>
  <aside class="link-topo-aside">
    <Radial v-if="radioStatus" :datas="{ nodes: stateTopo.nodes, calls: stateTopo.calls }" />
    <svg class="link-topo-aside-btn icon cp lg" @click="showRadial()" :style="`position:absolute;left:590px;`">
      <use xlink:href="#issues" />
    </svg>
    <svg
      v-if="showServerInfo"
      class="link-topo-aside-btn icon cp lg"
      @click="show = !show"
      :style="`position:absolute;left:590px;transform: rotate(${show ? 0 : 180}deg);top:45px;`"
    >
      <use xlink:href="#chevron-left" />
    </svg>
    <TopoService />

    <TopoProject />

      <div class='choose-time'>
        <span class="sm b grey mr-5">{{ this.$t('timeRange') }}:</span>
        <RkDate class="sm" v-model="time" position="bottom" format="YYYY-MM-DD HH:mm:ss" />
      </div>

    <div v-if="show">
      <div class="link-topo-aside-box" style="top:45px" v-if="!stateTopo.selectedServiceCall && showServerInfo">
        <div class="mb-20">
          <span class="b dib mr-20">{{ $t('serviceDetail') }}</span>
        </div>
        <div class="mb-10">
          <span class="label grey">{{ $t('name') }}</span>
          <span class="content">{{ stateTopo.currentNode.name }}</span>
        </div>
        <div class="mb-10">
          <span class="label grey">{{ $t('type') }}</span>
          <span class="content">{{ stateTopo.currentNode.type }}</span>
        </div>
        <div>
          <TopoChart
            v-if="stateTopo.serviceApdexScore.length"
            :data="stateTopo.serviceApdexScore"
            :intervalTime="intervalTime"
            title="Service ApdexScore"
            unit=""
          />
          <TopoChart
            v-if="stateTopo.serviceSLA.length"
            :data="stateTopo.serviceSLA"
            :intervalTime="intervalTime"
            title="Service SLA"
            unit="%"
          />
        </div>
      </div>
      <TopoDetectPoint />
    </div>
  </aside>
</template>
<script lang="ts">
  import { initState } from '@/store/modules/dashboard/dashboard-data-layout';
  import topo, { State as topoState } from '@/store/modules/topology';
  import { Component, Vue, Watch } from 'vue-property-decorator';
  import { Action, Getter, Mutation, State } from 'vuex-class';
  import Radial from './radial.vue';
  import TopoChart from './topo-chart.vue';
  import TopoService from './topo-services.vue';
  import TopoProject from './topo-project.vue';

  import TopoDetectPoint from './topo-detect-point.vue';
  import { Duration, Option } from '@/types/global';
  import TraceSelect from '../common/trace-select.vue';
  
  @Component({
    components: {
      TopoChart,
      TopoService,
      TopoProject,
      Radial,
      TopoDetectPoint,
    },
  })
  export default class TopoAside extends Vue {
       @State('rocketbot') private rocketbotGlobal: any;
    @State('rocketTrace') private rocketTrace: any;
    // @Getter('durationTime') private durationTime: any;
    @Getter('duration') private duration: any;
    @Action('RESET_DURATION') private RESET_DURATION: any;
    @Action('rocketTrace/GET_SERVICES') private GET_SERVICES: any;
    @Action('rocketTrace/GET_INSTANCES') private GET_INSTANCES: any;
    @Action('rocketTrace/GET_TRACELIST') private GET_TRACELIST: any;
    @Action('rocketTrace/SET_TRACE_FORM') private SET_TRACE_FORM: any;
    @Mutation('rocketTrace/SET_TRACE_FORM_ITEM')
    private SET_TRACE_FORM_ITEM: any;
    private service: Option = { label: 'All', key: '' };
    private time!: Date[];
    private status: boolean = true;
    private maxTraceDuration: string = localStorage.getItem('maxTraceDuration') || '';
    private minTraceDuration: string = localStorage.getItem('minTraceDuration') || '';
    private instance: Option = { label: 'All', key: '' };
    private endpointName: string = localStorage.getItem('endpointName') || '';
    private traceId: string = localStorage.getItem('traceId') || '';
    private traceState: Option = { label: 'All', key: 'ALL' };


    @State('rocketTopo') private stateTopo!: topoState;
    @Getter('intervalTime') private intervalTime: any;
    @Getter('durationTime') private durationTime: any;
    @Action('rocketTopo/CLEAR_TOPO') private CLEAR_TOPO: any;
    @Action('rocketTopo/CLEAR_TOPO_INFO') private CLEAR_TOPO_INFO: any;
    @Mutation('SET_COMPS_TREE') private SET_COMPS_TREE: any;
    @Mutation('SET_EVENTS') private SET_EVENTS: any;
    @Mutation('rocketTopo/SET_MODE_STATUS') private SET_MODE_STATUS: any;
    private dialogTopoVisible = false;
    private drawerMainBodyHeight = '100%';
    private initState = initState;
    private radioStatus: boolean = false;
    private show: boolean = true;

    private get showServerInfo() {
      return this.stateTopo.currentNode.name && this.stateTopo.currentNode.isReal;
    }

    private resize() {
      this.drawerMainBodyHeight = `${document.body.clientHeight - 50}px`;
    }

    private beforeMount() {
      this.SET_EVENTS([this.handleRefresh]);
    }

    private created() {
      this.SET_COMPS_TREE(this.initState.tree);
        this.endpointName = this.$route.query.endpointname
        ? this.$route.query.endpointname.toString()
        : this.endpointName;
      this.traceId = this.$route.query.traceid ? this.$route.query.traceid.toString() : this.traceId;
      this.time = [this.rocketbotGlobal.durationRow.start, this.rocketbotGlobal.durationRow.end];
    }

    private handleRefresh() {
      this.$store.dispatch(
        this.stateTopo.mode ? 'rocketTopo/GET_TOPO_SERVICE_INFO' : 'rocketTopo/GET_TOPO_CLIENT_INFO',
        { ...this.stateTopo.currentLink, duration: this.durationTime },
      );
    }

    private mounted() {
      this.resize();
      window.addEventListener('resize', this.resize);
            this.getTraceList();
      if (this.service && this.service.key) {
        this.GET_INSTANCES({
          duration: this.durationTime,
          serviceId: this.service.key,
        });
      }
    }

    private beforeDestroy() {
      window.removeEventListener('resize', this.resize);
      this.CLEAR_TOPO_INFO();
      this.CLEAR_TOPO();
      this.SET_EVENTS([]);
    }

    get types() {
      const result: any = {};
      this.stateTopo.nodes.forEach((i: any) => {
        if (result[i.type]) {
          result[i.type] += 1;
        } else {
          result[i.type] = 1;
        }
      });
      return result;
    }

    private showRadial() {
      this.radioStatus = !this.radioStatus;
    }

    
    private globalTimeFormat(time: Date[]) {
      let step = 'MINUTE';
      const unix = Math.round(time[1].getTime()) - Math.round(time[0].getTime());
      if (unix <= 60 * 60 * 1000) {
        step = 'MINUTE';
      } else if (unix <= 24 * 60 * 60 * 1000) {
        step = 'HOUR';
      } else {
        step = 'DAY';
      }
      return {
        start: this.dateFormat(time[0], step),
        end: this.dateFormat(time[1], step),
        step,
      };
    }
    private dateFormat(date: Date, step: string) {
      const year = date.getFullYear();
      const monthTemp = date.getMonth() + 1;
      let month: string = `${monthTemp}`;
      if (monthTemp < 10) {
        month = `0${monthTemp}`;
      }

      const dayTemp = date.getDate();
      let day: string = `${dayTemp}`;
      if (dayTemp < 10) {
        day = `0${dayTemp}`;
      }
      if (step === 'DAY' || step === 'MONTH') {
        return `${year}-${month}-${day}`;
      }
      const hourTemp = date.getHours();
      let hour: string = `${hourTemp}`;
      if (hourTemp < 10) {
        hour = `0${hourTemp}`;
      }
      if (step === 'HOUR') {
        return `${year}-${month}-${day} ${hour}`;
      }
      const minuteTemp = date.getMinutes();
      let minute: string = `${minuteTemp}`;
      if (minuteTemp < 10) {
        minute = `0${minuteTemp}`;
      }
      if (step === 'MINUTE') {
        return `${year}-${month}-${day} ${hour}${minute}`;
      }
    }
        private chooseService(i: any) {
      if (this.service.key === i.key) {
        return;
      }
      this.instance = { label: 'All', key: '' };
      this.service = i;
      if (i.key === '') {
        return;
      }
      this.GET_INSTANCES({ duration: this.durationTime, serviceId: i.key });
    }

    private chooseStatus(i: any) {
      this.traceState = i;
    }

    private getTraceList() {
      this.GET_SERVICES({ duration: this.durationTime });
      const temp: any = {
        queryDuration: this.globalTimeFormat([
          new Date(
            this.time[0].getTime() +
              (parseInt(this.rocketbotGlobal.utc, 10) + new Date().getTimezoneOffset() / 60) * 3600000,
          ),
          new Date(
            this.time[1].getTime() +
              (parseInt(this.rocketbotGlobal.utc, 10) + new Date().getTimezoneOffset() / 60) * 3600000,
          ),
        ]),
        traceState: this.traceState.key,
        paging: { pageNum: 1, pageSize: 15, needTotal: true },
        queryOrder: this.rocketTrace.traceForm.queryOrder,
      };

      if (this.service.key) {
        temp.serviceId = this.service.key;
      }
      if (this.instance.key) {
        temp.serviceInstanceId = this.instance.key;
      }
      if (this.maxTraceDuration) {
        temp.maxTraceDuration = this.maxTraceDuration;
        localStorage.setItem('maxTraceDuration', this.maxTraceDuration);
      }
      if (this.minTraceDuration) {
        temp.minTraceDuration = this.minTraceDuration;
        localStorage.setItem('minTraceDuration', this.minTraceDuration);
      }
      if (this.endpointName) {
        temp.endpointName = this.endpointName;
        localStorage.setItem('endpointName', this.endpointName);
      }
      if (this.traceId) {
        temp.traceId = this.traceId;
        localStorage.setItem('traceId', this.traceId);
      }
      this.SET_TRACE_FORM(temp);

      this.$eventBus.$emit('SET_LOADING_TRUE', () => {
        this.GET_TRACELIST().then(() => {
          this.$eventBus.$emit('SET_LOADING_FALSE');
        });
      });
    }

    private clearSearch() {
      this.RESET_DURATION();
      this.status = true;
      this.maxTraceDuration = '';
      localStorage.removeItem('maxTraceDuration');
      this.minTraceDuration = '';
      localStorage.removeItem('minTraceDuration');
      this.service = { label: 'All', key: '' };
      this.instance = { label: 'All', key: '' };
      this.endpointName = '';
      localStorage.removeItem('endpointName');
      this.traceId = '';
      localStorage.removeItem('traceId');
      this.traceState = { label: 'All', key: 'ALL' };
      this.SET_TRACE_FORM_ITEM({ type: 'queryOrder', data: '' });
      this.getTraceList();
    }

    @Watch('rocketbotGlobal.durationRow')
    private durationRowWatch(value: Duration) {
      this.time = [value.start, value.end];
    }


  }
</script>
<style lang="scss">
  .link-topo-aside {
    width: 280px;
    position: absolute;
    z-index: 2;
    left: 20px;
    top: 7px;

    .to-apm {
      padding-top: 10px;
      border-top: 1px solid #d8d8d866;
    }
  }

  .link-topo-aside-btn {
    display: block;
    background: #252a2f9a;
    color: #ddd;
    border-radius: 4px 4px 4px 4px;
    text-align: center;
    padding: 10px;
    z-index: 101;
  }

  .link-topo-aside-box {
    border-radius: 4px;
    position: absolute;
    width: 280px;
    z-index: 101;
    color: #ddd;
    background-color: #2b3037;
    padding: 15px 20px 10px;

    .label {
      display: inline-block;
      width: 40%;
    }

    .content {
      vertical-align: top;
      display: inline-block;
      width: 60%;
    }

    .circle {
      width: 8px;
      height: 8px;
      border-radius: 4px;
      background-color: #ee5b5b;
      margin-top: 6px;
    }
  }

  .link-topo-aside-box {
    p {
      margin-block-start: auto !important;
      margin-block-end: auto !important;
    }
  }

  .link-topo-aside-box-min {
    width: 280px;
    animation: 0.5s linkTopoAsideBoxMin 1 running;
  }

  .link-topo-aside-box-max {
    width: 60%;
    animation: 0.5s linkTopoAsideBoxMax 1 running;
  }

  @keyframes linkTopoAsideBoxMax {
    from {
      width: 280px;
    }
    to {
      width: 60%;
    }
  }

  @keyframes linkTopoAsideBoxMin {
    from {
      width: 60%;
    }
    to {
      width: 280px;
    }
  }

  .choose-time{
    position:absolute;
    left:310px;
    color: #fff;
  }
</style>
