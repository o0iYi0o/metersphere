<template>
  <ms-container>
    <ms-main-container>
      <el-card>
        <section class="report-container">
          <div style="margin-top: 10px">
            <ms-api-report-view-header
              :show-cancel-button="false"
              :debug="debug"
              :export-flag="exportFlag"
              :report="report"
              @reportExport="handleExport"
              @reportSave="handleSave"/>
          </div>
          <main>
            <ms-metric-chart
              :content="content"
              :totalTime="totalTime"
              v-if="!loading"/>
            <div>
              <el-tabs v-model="activeName" @tab-click="handleClick">
                <el-tab-pane :label="$t('api_report.total')" name="total">
                  <ms-scenario-results
                    :treeData="fullTreeNodes"
                    :default-expand="true"
                    :console="content.console"
                    v-on:requestResult="requestResult"
                  />
                </el-tab-pane>
                <el-tab-pane name="fail">
                  <template slot="label">
                    <span class="fail">
                      {{ $t('api_report.fail') }}
                    </span>
                  </template>
                  <ms-scenario-results
                    :console="content.console"
                    :treeData="failsTreeNodes"
                    v-on:requestResult="requestResult"
                  />
                </el-tab-pane>
              </el-tabs>
            </div>
            <ms-api-report-export
              :title="report.testName"
              :content="content"
              :total-time="totalTime"
              id="apiTestReport"
              v-if="reportExportVisible"
            />
          </main>
        </section>
      </el-card>
    </ms-main-container>
  </ms-container>
</template>

<script>

import MsRequestResult from "./components/RequestResult";
import MsRequestResultTail from "./components/RequestResultTail";
import MsScenarioResult from "./components/ScenarioResult";
import MsMetricChart from "./components/MetricChart";
import MsScenarioResults from "./components/ScenarioResults";
import MsContainer from "@/business/components/common/components/MsContainer";
import MsMainContainer from "@/business/components/common/components/MsMainContainer";
import MsApiReportExport from "./ApiReportExport";
import MsApiReportViewHeader from "./ApiReportViewHeader";
import {RequestFactory} from "../../definition/model/ApiTestModel";
import {windowPrint, getCurrentProjectID, getUUID} from "@/common/js/utils";
import {STEP} from "../scenario/Setting";
import {scenario} from "@/business/components/track/plan/event-bus";

export default {
  name: "SysnApiReportDetail",
  components: {
    MsApiReportViewHeader,
    MsApiReportExport,
    MsMainContainer,
    MsContainer, MsScenarioResults, MsRequestResultTail, MsMetricChart, MsScenarioResult, MsRequestResult
  },
  data() {
    return {
      activeName: "total",
      content: {total: 0, scenarioTotal: 1},
      report: {},
      loading: false,
      fails: [],
      failsTreeNodes: [],
      totalTime: 0,
      isRequestResult: false,
      startTime: 99991611737506593,
      endTime: 0,
      request: {},
      isActive: false,
      scenarioName: null,
      reportExportVisible: false,
      requestType: undefined,
      fullTreeNodes: [],
      debugResult: new Map,
      scenarioMap: new Map,
      exportFlag: false,
      messageWebSocket: {},
      websocket: {},
      stepFilter: new STEP,
    }
  },
  activated() {
    this.isRequestResult = false;
  },
  created() {
    if (this.scenario && this.scenario.scenarioDefinition) {
      this.content.scenarioStepTotal = this.scenario.scenarioDefinition.hashTree.length;
      this.initTree();
      this.initWebSocket();
      this.initMessageSocket();
      this.clearDebug();
    }
  },
  props: {
    reportId: String,
    currentProjectId: String,
    infoDb: Boolean,
    debug: Boolean,
    scenario: {},
    scenarioId: String
  },
  methods: {
    initTree() {
      this.fullTreeNodes = [];
      let obj = {index: 1, label: this.scenario.name, value: {responseResult: {}, unexecute: true, testing: false}, children: [], unsolicited: true};
      this.formatContent(this.scenario.scenarioDefinition.hashTree, obj);
      this.fullTreeNodes.push(obj);
    },
    compare(){
      return function(a,b){
        let v1 = a.value.sort;
        let v2 = b.value.sort;
        return v1 - v2;
      }
    },
    setTreeValue(arr) {
      arr.forEach(item => {
        if (this.debugResult && this.debugResult.get(item.resId)) {
          let arrValue = this.debugResult.get(item.resId);
          if (arrValue.length > 1) {
            for (let i = 0; i < arrValue.length; i++) {
              let obj = {resId: item.resId, index: i, label: item.resId, value: arrValue[i], children: []};
              let isAdd = true;
              for (let h = 0; h < arr.length; h++) {
                let node = arr [h];
                if (node.value.name === arrValue[i].name) {
                  isAdd = false;
                }
                if (node.resId === arrValue[i].resourceId && node.value.unexecute) {
                  arr.splice(h, 1);
                }
              }
              if (isAdd) {
                arr.push(obj);
              }
            }
          } else {
            item.value = arrValue[0];
          }
        }
        if (item.children && item.children.length > 0) {
          this.setTreeValue(item.children);
        }
      })
      arr.sort(this.compare());
    },
    getType(type) {
      switch (type) {
        case "LoopController":
          return "循环控制器";
        case "TransactionController":
          return "事物控制器";
        case "ConstantTimer":
          return "等待控制器";
        case "IfController":
          return "条件控制器";
      }
      return type;
    },
    formatContent(hashTree, tree, fullPath) {
      if (hashTree) {
        hashTree.forEach(item => {
          if (item.enable) {
            item.parentIndex = fullPath ? fullPath + "_" + item.index : item.index;
            let name = item.name ? item.name : this.getType(item.type);
            let obj = {resId: item.resourceId + "_" + item.parentIndex, index: Number(item.index), label: name, value: {name: name, responseResult: {}, unexecute: true, testing: false}, children: [], unsolicited: true};
            tree.children.push(obj);
            if (this.stepFilter.get("AllSamplerProxy").indexOf(item.type) !== -1) {
              obj.unsolicited = false;
              obj.type = item.type;
            } else if (item.type === 'scenario') {
              this.content.scenarioTotal += 1;
            }
            if (item.hashTree && item.hashTree.length > 0 && this.stepFilter && this.stepFilter.get("AllSamplerProxy").indexOf(item.type) === -1) {
              this.formatContent(item.hashTree, obj, item.parentIndex);
            }
          }
        })
      }
    },
    handleExport() {
      this.reportExportVisible = true;
      let reset = this.exportReportReset;
      this.$nextTick(() => {
        windowPrint('apiTestReport', 0.57);
        reset();
      });
    },
    handleClick(tab, event) {
      this.isRequestResult = false
    },
    exportReportReset() {
      this.$router.go(0);
    },
    requestResult(requestResult) {
      this.active();
      this.isRequestResult = false;
      this.requestType = undefined;
      if (requestResult.request.body.indexOf('[Callable Statement]') > -1) {
        this.requestType = RequestFactory.TYPES.SQL;
      }
      this.$nextTick(function () {
        this.isRequestResult = true;
        this.request = requestResult.request;
        this.scenarioName = requestResult.scenarioName;
      });
    },

    clearDebug() {
      this.totalTime = 0;
      this.content.total = 0;
      this.content.error = 0;
      this.content.success = 0;
      this.content.passAssertions = 0;
      this.content.totalAssertions = 0;
      this.content.scenarioSuccess = 0;
      this.content.scenarioError = 0;
    },
    initWebSocket() {
      let protocol = "ws://";
      if (window.location.protocol === 'https:') {
        protocol = "wss://";
      }
      const uri = protocol + window.location.host + "/api/scenario/report/get/real/" + this.reportId;
      this.websocket = new WebSocket(uri);
      this.websocket.onmessage = this.onMessage;
    },
    onMessage(e) {
      if (e.data) {
        let data = JSON.parse(e.data);
        this.formatResult(data);
        if (data.end) {
          this.removeReport();
          this.getReport();
          this.$emit('refresh', this.debugResult);
        }
      }
    },
    removeReport() {
      let url = "/api/scenario/report/remove/real/" + this.reportId;
      this.$get(url, response => {
        scenario.$emit('hide', this.scenarioId);
        this.$success(this.$t('schedule.event_success'));
        this.websocket.close();
        this.messageWebSocket.close();
      });
    },
    getTransaction(transRequests, resMap) {
      transRequests.forEach(subItem => {
        if (subItem.method === 'Request') {
          this.getTransaction(subItem.subRequestResults, resMap);
        }
        this.reqTotal++;
        let key = subItem.resourceId;
        if (resMap.get(key)) {
          if (resMap.get(key).indexOf(subItem) === -1 && subItem.method !== 'Request') {
            resMap.get(key).push(subItem);
          }
        } else {
          resMap.set(key, [subItem]);
        }
        if (subItem.success) {
          this.reqSuccess++;
        } else {
          this.reqError++;
        }
        if (subItem.startTime && Number(subItem.startTime) < this.startTime) {
          this.startTime = subItem.startTime;
        }
        if (subItem.endTime && Number(subItem.endTime) > this.endTime) {
          this.endTime = subItem.endTime;
        }
      })
    },
    formatResult(res) {
      let resMap = new Map;
      this.startTime = 99991611737506593;
      this.endTime = 0;
      this.clearDebug();
      if (res && res.scenarios) {
        res.scenarios.forEach(item => {
          this.content.total += item.requestResults.length;
          this.content.passAssertions += item.passAssertions
          this.content.totalAssertions += item.totalAssertions;
          if (item && item.requestResults) {
            for(let index in item.requestResults){
              let req = item.requestResults[index];
              req.sort = index;
              req.responseResult.console = res.console;
              if (req.method === 'Request' && req.subRequestResults && req.subRequestResults.length > 0) {
                this.getTransaction(req.subRequestResults, resMap);
              } else {
                this.reqTotal++;
                let key = req.resourceId;
                if (resMap.get(key)) {
                  if (resMap.get(key).indexOf(req) === -1) {
                    resMap.get(key).push(req);
                  }
                } else {
                  resMap.set(key, [req]);
                }
                if (req.success) {
                  this.reqSuccess++;
                } else {
                  this.reqError++;
                }
                if (req.startTime && Number(req.startTime) < this.startTime) {
                  this.startTime = req.startTime;
                }
                if (req.endTime && Number(req.endTime) > this.endTime) {
                  this.endTime = req.endTime;
                }
              }
            }
          }
        })
      }
      if (this.startTime < this.endTime) {
        this.totalTime = this.endTime - this.startTime + 100;
      }
      this.debugResult = resMap;
      this.setTreeValue(this.fullTreeNodes);
      this.reload();
    },
    reload() {
      this.loading = true;
      this.$nextTick(() => {
        this.loading = false;
      });
    },
    getReport() {
      let url = "/api/scenario/report/get/" + this.reportId;
      this.$get(url, response => {
        this.report = response.data || {};
        if (response.data) {
          try {
            this.content = JSON.parse(this.report.content);
            if (!this.content) {
              this.content = {scenarios: []};
            }
          } catch (e) {
            throw e;
          }
          this.getFails();
        }
      });
    },
    getFails() {
      this.fails = [];
      let array = [];
      if (this.content.scenarios) {
        this.content.scenarios.forEach((scenario) => {
          let failScenario = Object.assign({}, scenario);
          if (scenario.error > 0) {
            this.fails.push(failScenario);
            failScenario.requestResults = [];
            scenario.requestResults.forEach((request) => {
              if (!request.success) {
                let failRequest = Object.assign({}, request);
                failScenario.requestResults.push(failRequest);
                array.push(request);
              }
            })
          }
        })
        this.formatTree(array, this.failsTreeNodes);
        this.recursiveSorting(this.failsTreeNodes);
        this.exportFlag = true;
      }
    },
    formatTree(array, tree) {
      array.map((item) => {
        let key = item.name;
        let nodeArray = key.split('^@~@^');
        let children = tree;
        let scenarioId = "";
        let scenarioName = "";
        if (item.scenario) {
          let scenarioArr = JSON.parse(item.scenario);
          if (scenarioArr.length > 1) {
            let scenarioIdArr = scenarioArr[0].split("_");
            scenarioId = scenarioIdArr[0];
            scenarioName = scenarioIdArr[1];
          }
        }
        // 循环构建子节点
        for (let i = 0; i < nodeArray.length; i++) {
          if (!nodeArray[i]) {
            continue;
          }
          let node = {
            label: nodeArray[i],
            value: item,
          };
          if (i !== nodeArray.length) {
            node.children = [];
          }

          if (children.length === 0) {
            children.push(node);
          }

          let isExist = false;
          for (let j in children) {
            if (children[j].label === node.label) {

              let idIsPath = true;
              //判断ID是否匹配  目前发现问题的只有重复场景，而重复场景是在第二个节点开始合并的。所以这里暂时只判断第二个场景问题。
              //如果出现了其他问题，则需要检查其他问题的数据结构。暂时采用具体问题具体分析的策略
              if (i === nodeArray.length - 2) {
                idIsPath = false;
                let childId = "";
                let childName = "";
                if (children[j].value && children[j].value.scenario) {
                  let scenarioArr = JSON.parse(children[j].value.scenario);
                  if (scenarioArr.length > 1) {
                    let childArr = scenarioArr[0].split("_");
                    childId = childArr[0];
                    if (childArr.length > 1) {
                      childName = childArr[1];
                    }
                  }
                }
                if (scenarioId === "") {
                  idIsPath = true;
                } else if (scenarioId === childId) {
                  idIsPath = true;
                } else if (scenarioName !== childName) {
                  //如果两个名字不匹配则默认通过，不匹配ID
                  idIsPath = true;
                }
              }
              if (idIsPath) {
                if (i !== nodeArray.length - 1 && !children[j].children) {
                  children[j].children = [];
                }
                children = (i === nodeArray.length - 1 ? children : children[j].children);
                isExist = true;
                break;
              }
            }
          }
          if (!isExist) {
            children.push(node);
            if (i !== nodeArray.length - 1 && !children[children.length - 1].children) {
              children[children.length - 1].children = [];
            }
            children = (i === nodeArray.length - 1 ? children : children[children.length - 1].children);
          }
        }
      })
    },
    recursiveSorting(arr) {
      for (let i in arr) {
        if (arr[i]) {
          arr[i].index = Number(i) + 1;
          if (arr[i].children && arr[i].children.length > 0) {
            this.recursiveSorting(arr[i].children);
          }
        }
      }
    },
    handleSave() {
      if (!this.report.name) {
        this.$warning(this.$t('api_test.automation.report_name_info'));
        return;
      }
      this.loading = true;
      this.report.projectId = this.projectId;
      let url = "/api/scenario/report/update";
      this.result = this.$post(url, this.report, response => {
        this.$success(this.$t('commons.save_success'));
        this.loading = false;
        this.$emit('refresh');
      }, error => {
        this.loading = false;
      });
    },

    initMessageSocket() {
      let protocol = "ws://";
      if (window.location.protocol === 'https:') {
        protocol = "wss://";
      }
      const uri = protocol + window.location.host + "/ws/" + this.reportId;
      this.messageWebSocket = new WebSocket(uri);
      this.messageWebSocket.onmessage = this.onMessage2;
    },

    runningNodeChild(arr, resourceId) {
      arr.forEach(item => {
        if (resourceId === item.resId) {
          item.value.testing = true;
        }
        if (item.children && item.children.length > 0) {
          this.runningNodeChild(item.children, resourceId);
        }
      })
    },
    runningEvaluation(resourceId) {
      if (this.fullTreeNodes) {
        this.fullTreeNodes.forEach(item => {
          if (resourceId === item.resId) {
            item.value.testing = true;
          }
          if (item.children && item.children.length > 0) {
            this.runningNodeChild(item.children, resourceId);
          }
        })
      }
    },
    onMessage2(e) {
      if (e.data) {
        this.runningEvaluation(e.data);
      }
    },
  },
  computed: {
    projectId() {
      return getCurrentProjectID();
    },
  }
}
</script>
<style>
.report-container .el-tabs__header {
  margin-bottom: 1px;
}
</style>

<style scoped>

.report-container {
  height: calc(100vh - 155px);
  min-height: 600px;
  overflow-y: auto;
}

.report-header {
  font-size: 15px;
}

.report-header a {
  text-decoration: none;
}

.report-header .time {
  color: #909399;
  margin-left: 10px;
}

.report-container .fail {
  color: #F56C6C;
}

.report-container .is-active .fail {
  color: inherit;
}

.export-button {
  float: right;
}

.scenario-result .icon.is-active {
  transform: rotate(90deg);
}

</style>
