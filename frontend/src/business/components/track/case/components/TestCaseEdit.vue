<template>
  <el-card>
    <div class="card-content">
      <div class="ms-main-div" @click="showAll">

        <!--操作按钮-->
        <div class="ms-opt-btn">
          <el-tooltip :content="$t('commons.follow')" placement="bottom" effect="dark" v-if="!showFollow">
            <i class="el-icon-star-off"
               style="color: #783987; font-size: 25px;  margin-right: 15px;cursor: pointer;position: relative;top: 5px "
               @click="saveFollow"/>
          </el-tooltip>
          <el-tooltip :content="$t('commons.cancel')" placement="bottom" effect="dark" v-if="showFollow">
            <i class="el-icon-star-on"
               style="color: #783987; font-size: 28px; margin-right: 15px;cursor: pointer;position: relative;top: 5px "
               @click="saveFollow"/>
          </el-tooltip>
          <el-link type="primary" style="margin-right: 20px" @click="openHis" v-if="form.id">
            {{ $t('operating_log.change_history') }}
          </el-link>
          <ms-table-button v-if="this.path!=='/test/case/add'"
                           id="inputDelay"
                           type="primary"
                           :content="$t('commons.save')"
                           size="small" @click="saveCase"
                           icon=""
                           :disabled="readOnly"
                           title="ctrl + s"/>
          <el-dropdown v-else split-button type="primary" class="ms-api-buttion" @click="handleCommand"
                       @command="handleCommand" size="small" style="float: right;margin-right: 20px">
            {{ $t('commons.save') }}
            <el-dropdown-menu slot="dropdown">
              <el-dropdown-item command="ADD_AND_CREATE">{{
                  $t('test_track.case.save_create_continue')
                }}
              </el-dropdown-item>
            </el-dropdown-menu>
          </el-dropdown>
        </div>
        <el-form :model="form" :rules="rules" ref="caseFrom" v-loading="result.loading" class="case-form">
          <ms-form-divider :title="$t('test_track.plan_view.base_info')"/>
          <el-row>
            <el-col :span="6">
              <el-form-item
                :placeholder="$t('test_track.case.input_name')"
                :label="$t('test_track.case.name')"
                :label-width="formLabelWidth"
                prop="name">
                <el-input :disabled="readOnly" v-model="form.name" size="small" class="ms-case-input"></el-input>
              </el-form-item>
            </el-col>

            <el-col :span="6">
              <el-form-item :label="$t('test_track.case.module')" :label-width="formLabelWidth" prop="module">
                <ms-select-tree :disabled="readOnly" :data="treeNodes" :defaultKey="form.module" :obj="moduleObj"
                                @getValue="setModule" clearable checkStrictly size="small"/>
              </el-form-item>
            </el-col>

            <el-col :span="6">
              <el-form-item :label="$t('commons.tag')" :label-width="formLabelWidth" prop="tag">
                <ms-input-tag :read-only="readOnly" :currentScenario="form" v-if="showInputTag" ref="tag"
                              class="ms-case-input"/>
              </el-form-item>
            </el-col>
          </el-row>

          <!-- 自定义字段 -->
          <el-form v-if="isFormAlive" :model="customFieldForm" :rules="customFieldRules" ref="customFieldForm"
                   class="case-form">
            <el-row>
              <el-col :span="7" v-for="(item, index) in testCaseTemplate.customFields" :key="index">
                <el-form-item :label="item.system ? $t(systemNameMap[item.name]) : item.name" :prop="item.name"
                              :label-width="formLabelWidth">
                  <custom-filed-component :disabled="readOnly" @reload="reloadForm" :data="item" :form="customFieldForm"
                                          prop="defaultValue"/>
                </el-form-item>
              </el-col>
            </el-row>
          </el-form>

          <el-row v-if="isCustomNum">
            <el-col :span="7">
              <el-form-item label="ID" :label-width="formLabelWidth" prop="customNum">
                <el-input :disabled="readOnly" v-model.trim="form.customNum" size="small"
                          class="ms-case-input"></el-input>
              </el-form-item>
            </el-col>
          </el-row>


          <ms-form-divider :title="$t('test_track.case.step_info')"/>

          <form-rich-text-item :disabled="readOnly" :label-width="formLabelWidth"
                               :title="$t('test_track.case.prerequisite')" :data="form" prop="prerequisite"/>

          <step-change-item :label-width="formLabelWidth" :form="form"/>
          <form-rich-text-item :disabled="readOnly" :label-width="formLabelWidth" v-if="form.stepModel === 'TEXT'"
                               :title="$t('test_track.case.step_desc')" :data="form" prop="stepDescription"/>
          <form-rich-text-item :disabled="readOnly" :label-width="formLabelWidth" v-if="form.stepModel === 'TEXT'"
                               :title="$t('test_track.case.expected_results')" :data="form" prop="expectedResult"/>

          <test-case-step-item :label-width="formLabelWidth" v-if="form.stepModel === 'STEP' || !form.stepModel"
                               :form="form" :read-only="readOnly"/>

          <ms-form-divider :title="$t('test_track.case.other_info')"/>

          <test-case-edit-other-info :read-only="readOnly" :project-id="projectIds" :form="form"
                                     :label-width="formLabelWidth" :case-id="form.id" ref="otherInfo"/>

          <el-row style="margin-top: 10px" v-if="type!=='add'">
            <el-col :span="20" :offset="1">{{ $t('test_track.review.comment') }}:
              <el-button icon="el-icon-plus" type="mini" @click="openComment"></el-button>
            </el-col>
          </el-row>
          <el-row v-if="type!=='add'">
            <el-col :span="20" :offset="1">

              <review-comment-item v-for="(comment,index) in comments"
                                   :key="index"
                                   :comment="comment"
                                   @refresh="getComments"/>
              <div v-if="comments.length === 0" style="text-align: center">
                <i class="el-icon-chat-line-square" style="font-size: 15px;color: #8a8b8d;">
                      <span style="font-size: 15px; color: #8a8b8d;">
                        {{ $t('test_track.comment.no_comment') }}
                      </span>
                </i>
              </div>
            </el-col>
          </el-row>
          <test-case-comment :case-id="form.id"
                             @getComments="getComments" ref="testCaseComment"/>

        </el-form>

      </div>
      <ms-change-history ref="changeHistory"/>

    </div>
  </el-card>


</template>

<script>
import {TokenKey} from '@/common/js/constants';
import MsDialogFooter from '../../../common/components/MsDialogFooter'
import {
  getCurrentProjectID,
  getCurrentUser,
  getNodePath, getUUID,
  handleCtrlSEvent, hasPermission,
  listenGoBack,
  removeGoBackListener
} from "@/common/js/utils";
import TestCaseAttachment from "@/business/components/track/case/components/TestCaseAttachment";
import CaseComment from "@/business/components/track/case/components/CaseComment";
import MsInputTag from "@/business/components/api/automation/scenario/MsInputTag";
import MsPreviousNextButton from "../../../common/components/MsPreviousNextButton";
import {STEP} from "@/business/components/api/automation/scenario/Setting";
import TestCaseComment from "@/business/components/track/case/components/TestCaseComment";
import ReviewCommentItem from "@/business/components/track/review/commom/ReviewCommentItem";
import {API_STATUS, REVIEW_STATUS} from "@/business/components/api/definition/model/JsonData";
import MsTableButton from "@/business/components/common/components/MsTableButton";
import MsSelectTree from "../../../common/select-tree/SelectTree";
import MsTestCaseStepRichText from "./MsRichText";
import CustomFiledComponent from "@/business/components/settings/workspace/template/CustomFiledComponent";
import {
  buildCustomFields,
  buildTestCaseOldFields,
  getTemplate,
  parseCustomField
} from "@/common/js/custom_field";
import {SYSTEM_FIELD_NAME_MAP} from "@/common/js/table-constants";
import MsFormDivider from "@/business/components/common/components/MsFormDivider";
import TestCaseEditOtherInfo from "@/business/components/track/case/components/TestCaseEditOtherInfo";
import FormRichTextItem from "@/business/components/track/case/components/FormRichTextItem";
import TestCaseStepItem from "@/business/components/track/case/components/TestCaseStepItem";
import StepChangeItem from "@/business/components/track/case/components/StepChangeItem";
import MsChangeHistory from "../../../history/ChangeHistory";
import {getTestTemplate} from "@/network/custom-field-template";

export default {
  name: "TestCaseEdit",
  components: {
    StepChangeItem,
    TestCaseStepItem,
    FormRichTextItem,
    TestCaseEditOtherInfo,
    MsFormDivider,
    CustomFiledComponent,
    MsTableButton,
    MsSelectTree,
    ReviewCommentItem,
    TestCaseComment, MsPreviousNextButton, MsInputTag, CaseComment, MsDialogFooter, TestCaseAttachment,
    MsTestCaseStepRichText,
    MsChangeHistory
  },
  data() {
    return {
      // sysList: [],//一级选择框的数据
      path: "/test/case/add",
      testCaseTemplate: {},
      options: REVIEW_STATUS,
      statuOptions: API_STATUS,
      comments: [],
      result: {},
      dialogFormVisible: false,
      showFollow: false,
      form: {
        name: '',
        module: 'default-module',
        nodePath: '/未规划用例',
        maintainer: getCurrentUser().id,
        priority: 'P0',
        type: '',
        method: '',
        prerequisite: '',
        testId: '',
        otherTestName: '',
        steps: [{
          num: 1,
          desc: '',
          result: ''
        }],
        stepDesc: '',
        stepResult: '',
        selected: [],
        remark: '',
        tags: [],
        demandId: '',
        demandName: '',
        status: 'Prepare',
        reviewStatus: 'Prepare',
        stepDescription: '',
        expectedResult: '',
        stepModel: 'STEP',
        customNum: '',
        followPeople: '',
      },
      maintainerOptions: [],
      // testOptions: [],
      workspaceId: '',
      rules: {
        name: [
          {required: true, message: this.$t('test_track.case.input_name'), trigger: 'blur'},
          {max: 255, message: this.$t('test_track.length_less_than') + '255', trigger: 'blur'}
        ],
        module: [{required: true, message: this.$t('test_track.case.input_module'), trigger: 'change'}],
        customNum: [
          {required: true, message: "ID必填", trigger: 'blur'},
          {max: 50, message: this.$t('test_track.length_less_than') + '50', trigger: 'blur'}
        ],
        demandName: [{required: true, message: this.$t('test_track.case.input_demand_name'), trigger: 'change'}],
        maintainer: [{required: true, message: this.$t('test_track.case.input_maintainer'), trigger: 'change'}],
        priority: [{required: true, message: this.$t('test_track.case.input_priority'), trigger: 'change'}],
        method: [{required: true, message: this.$t('test_track.case.input_method'), trigger: 'change'}],
        // prerequisite: [{max: 500, message: this.$t('test_track.length_less_than') + '500', trigger: 'blur'}],
        // remark: [{max: 1000, message: this.$t('test_track.length_less_than') + '1000', trigger: 'blur'}]
      },
      customFieldRules: {},
      customFieldForm: {},
      formLabelWidth: "100px",
      operationType: '',
      isCreateContinue: false,
      isStepTableAlive: true,
      isFormAlive: true,
      methodOptions: [
        {value: 'auto', label: this.$t('test_track.case.auto')},
        {value: 'manual', label: this.$t('test_track.case.manual')}
      ],
      testCase: {},
      testCases: [],
      index: 0,
      showInputTag: true,
      tableType: "",
      stepFilter: new STEP,
      moduleObj: {
        id: 'id',
        label: 'name',
      },
      tabId: getUUID()
    };
  },
  props: {
    treeNodes: {
      type: Array
    },
    currentTestCaseInfo: {},
    selectNode: {
      type: Object
    },
    selectCondition: {
      type: Object
    },
    type: String,
  },
  computed: {
    projectIds() {
      return getCurrentProjectID();
    },
    moduleOptions() {
      return this.$store.state.testCaseModuleOptions;
    },
    systemNameMap() {
      return SYSTEM_FIELD_NAME_MAP;
    },
    isCustomNum() {
      return this.$store.state.currentProjectIsCustomNum;
    },
    readOnly() {
      const {rowClickHasPermission} = this.currentTestCaseInfo;
      if (rowClickHasPermission !== undefined) {
        return !rowClickHasPermission;
      }
      return !hasPermission('PROJECT_TRACK_CASE:READ+CREATE') &&
        !hasPermission('PROJECT_TRACK_CASE:READ+EDIT');
    }
  },
  watch: {
    form: {
      handler(val) {
        if (val && this.$store.state.testCaseMap) {
          let change = this.$store.state.testCaseMap.get(this.form.id);
          change = change + 1;
          this.$store.state.testCaseMap.set(this.form.id, change);
        }
      },
      deep: true
    },
    'testCaseTemplate.customFields': {
      handler(val) {
        if (val && this.$store.state.testCaseMap) {
          let change = this.$store.state.testCaseMap.get(this.form.id);
          change = change + 1;
          this.$store.state.testCaseMap.set(this.form.id, change);
        }
      },
      deep: true
    }
  },
  beforeDestroy() {
    this.removeListener();
  },
  mounted() {
    this.getSelectOptions();
    if (this.type === 'edit' || this.type === 'copy') {
      this.open(this.currentTestCaseInfo)
      this.getComments(this.currentTestCaseInfo)
    }
    // Cascader 级联选择器: 点击文本就让它自动点击前面的input就可以触发选择。
    setInterval(function () {
      document.querySelectorAll('.el-cascader-node__label').forEach(el => {
        el.onclick = function () {
          if (this.previousElementSibling) this.previousElementSibling.click();
        };
      });
    }, 1000);
    if (this.selectNode && this.selectNode.data && !this.form.id) {
      this.form.module = this.selectNode.data.id;
      this.form.nodePath = this.selectNode.data.path;
    }
    if ((!this.form.module || this.form.module === "default-module" || this.form.module === "root") && this.treeNodes.length > 0) {
      this.form.module = this.treeNodes[0].id;
      this.form.nodePath = this.treeNodes[0].path;
    }
    if (!(this.$store.state.testCaseMap instanceof Map)) {
      this.$store.state.testCaseMap = new Map();
    }
    this.$store.state.testCaseMap.set(this.form.id, 0);
  },
  created() {
    this.projectId = this.projectIds;
    let initAddFuc = this.initAddFuc;
    getTestTemplate()
      .then((template) => {
        this.testCaseTemplate = template;
        this.$store.commit('setTestCaseTemplate', this.testCaseTemplate);
        initAddFuc();
      });
    if (this.selectNode && this.selectNode.data && !this.form.id) {
      this.form.module = this.selectNode.data.id;
      this.form.nodePath = this.selectNode.data.path;
    } else {
      this.form.module = this.treeNodes && this.length > 0 ? this.treeNodes[0].id : "";
    }
    if (this.type === 'edit' || this.type === 'copy') {
      this.form.module = this.currentTestCaseInfo.nodeId;
      this.form.nodePath = this.currentTestCaseInfo.nodePath;
    }
    if ((!this.form.module || this.form.module === "default-module" || this.form.module === "root") && this.treeNodes.length > 0) {
      this.form.module = this.treeNodes[0].id;
      this.form.nodePath = this.treeNodes[0].path;
    }
    this.$get('/test/case/follow/' + this.currentTestCaseInfo.id, response => {
      this.form.follows = response.data;
      for (let i = 0; i < response.data.length; i++) {
        if (response.data[i] === this.currentUser().id) {
          this.showFollow = true;
          break;
        }
      }
    })
  },
  methods: {
    currentUser: () => {
      return getCurrentUser();
    },
    openHis() {
      this.$refs.changeHistory.open(this.form.id, ["测试用例", "測試用例", "Test case"]);
    },
    setModule(id, data) {
      this.form.module = id;
      this.form.nodePath = data.path;
    },
    initAddFuc() {
      // this.loadOptions();
      this.addListener(); //  添加 ctrl s 监听
      if (this.selectNode && this.selectNode.data && !this.form.id) {
        this.form.module = this.selectNode.data.id;
        this.form.nodePath = this.selectNode.data.path;
      } else {
        this.form.module = this.treeNodes && this.length > 0 ? this.treeNodes[0].id : "";
      }
      if (this.type === 'edit' || this.type === 'copy') {
        this.form.module = this.currentTestCaseInfo.nodeId;
        this.form.nodePath = this.currentTestCaseInfo.nodePath;
      }
      if ((!this.form.module || this.form.module === "default-module" || this.form.module === "root") && this.treeNodes.length > 0) {
        this.form.module = this.treeNodes[0].id;
        this.form.nodePath = this.treeNodes[0].path;
      }
      if (this.type === 'add') {
        //设置自定义熟悉默认值
        parseCustomField(this.form, this.testCaseTemplate, this.customFieldForm, this.customFieldRules);
        this.form.name = this.testCaseTemplate.caseName;
        this.form.stepDescription = this.testCaseTemplate.stepDescription;
        this.form.expectedResult = this.testCaseTemplate.expectedResult;
        this.form.prerequisite = this.testCaseTemplate.prerequisite;
        this.form.stepModel = this.testCaseTemplate.stepModel;
        if (this.testCaseTemplate.steps) {
          this.form.steps = JSON.parse(this.testCaseTemplate.steps);
        }
      }
    },
    setDefaultValue() {
      if (!this.form.prerequisite) {
        this.form.prerequisite = "";
      }
      if (!this.form.stepDescription) {
        this.form.stepDescription = "";
      }
      if (!this.form.expectedResult) {
        this.form.expectedResult = "";
      }
      if (!this.form.remark) {
        this.form.remark = "";
      }
      this.$store.state.testCaseMap.set(this.form.id, 0);
    },
    handleCommand(e) {
      if (e === "ADD_AND_CREATE") {
        this.$refs['caseFrom'].validate((valid) => {
          if (!valid) {
            this.saveCase();
          } else {
            this.saveCase(function (t) {
              let tab = {};
              tab.name = 'add';
              t.$emit('addTab', tab);
            });
          }
        })
      } else {
        this.saveCase();
      }
    },
    openComment() {
      this.$refs.testCaseComment.open()
    },
    getComments(testCase) {
      let id = '';
      if (testCase) {
        id = testCase.id;
      } else {
        id = this.form.id;
      }
      this.result = this.$get('/test/case/comment/list/' + id, res => {
        this.comments = res.data;
      })
    },
    showAll() {
      if (!this.customizeVisible) {
        this.operatingElements = this.stepFilter.get("ALL");
        this.selectedTreeNode = undefined;
      }
      //this.reload();
    },
    reload() {
      this.isStepTableAlive = false;
      this.$nextTick(() => {
        this.isStepTableAlive = true;
        this.$store.state.testCaseMap.set(this.form.id, 0);
      });
    },
    reloadForm() {
      this.isFormAlive = false;
      this.$nextTick(() => (this.isFormAlive = true));
    },
    open(testCase) {
      /*
             this.form.selected=[["automation", "3edaaf31-3fa4-4a53-9654-320205c2953a"],["automation", "3aa58bd1-c986-448c-8060-d32713dbd4eb"]]
      */
      this.projectId = this.projectIds;
      let initFuc = this.initEdit;
      getTemplate('field/template/case/get/relate/', this)
        .then((template) => {
          this.testCaseTemplate = template;
          this.$store.commit('setTestCaseTemplate', this.testCaseTemplate);
          initFuc(testCase);
        });
    },
    initEdit(testCase) {
      if (window.history && window.history.pushState) {
        history.pushState(null, null, document.URL);
        window.addEventListener('popstate', this.close);
      }
      this.resetForm();
      listenGoBack(this.close);
      this.operationType = 'add';
      if (testCase) {
        //修改
        this.operationType = 'edit';
        //复制
        if (this.type === 'copy') {
          this.operationType = 'add';
          this.setFormData(testCase);
          this.setTestCaseExtInfo(testCase);
          this.getSelectOptions();
          //设置自定义熟悉默认值
          parseCustomField(this.form, this.testCaseTemplate, this.customFieldForm, this.customFieldRules, buildTestCaseOldFields(this.form));
          this.reload();
        } else {
          this.initTestCases(testCase);
        }
      } else {
        if (this.selectNode.data) {
          this.form.module = this.selectNode.data.id;
        } else {
          if (this.moduleOptions.length > 0) {
            this.form.module = this.moduleOptions[0].id;
          }
        }
        let user = JSON.parse(localStorage.getItem(TokenKey));
        this.form.priority = 'P3';
        this.form.type = 'functional';
        this.form.method = 'manual';
        this.form.maintainer = user.id;
        this.form.tags = [];
        this.getSelectOptions();
        parseCustomField(this.form, this.testCaseTemplate, this.customFieldForm, this.customFieldRules);
        this.reload();
      }
    },
    handlePre() {
      this.index--;
      this.getTestCase(this.index)
    },
    handleNext() {
      this.index++;
      this.getTestCase(this.index);
    },
    initTestCases(testCase) {
      this.result = this.$post('/test/case/list/ids', this.selectCondition, response => {
        this.testCases = response.data;
        for (let i = 0; i < this.testCases.length; i++) {
          if (this.testCases[i].id === testCase.id) {
            this.index = i;
            this.getTestCase(i);
          }
        }
      });
    },
    getTestCase(index) {
      let id = "";
      this.showInputTag = false;
      let testCase = this.testCases[index];
      if (typeof (index) == "undefined") {
        id = this.currentTestCaseInfo.id;

      } else {
        id = testCase.id;
      }
      this.result = this.$get('/test/case/get/' + id, response => {
        if (response.data) {
          this.path = "/test/case/edit";
          if (this.currentTestCaseInfo.isCopy) {
            this.path = "/test/case/add";
          }
        } else {
          this.path = "/test/case/add";
        }
        let testCase = response.data;
        this.setFormData(testCase);
        this.setTestCaseExtInfo(testCase);
        this.getSelectOptions();
        this.reload();
        this.$nextTick(() => {
          this.showInputTag = true;
        });
      });
    },
    async setFormData(testCase) {
      try {
        testCase.selected = JSON.parse(testCase.testId);
      } catch (error) {
        testCase.selected = testCase.testId
      }
      let tmp = {};
      Object.assign(tmp, testCase);
      tmp.steps = JSON.parse(testCase.steps);
      if (!tmp.steps || tmp.steps.length < 1) {
        tmp.steps = [{
          num: 1,
          desc: '',
          result: ''
        }];
      }
      tmp.tags = JSON.parse(tmp.tags);
      Object.assign(this.form, tmp);
      if (!this.form.stepModel) {
        this.form.stepModel = "STEP";
      }
      this.form.module = testCase.nodeId;
      //设置自定义熟悉默认值
      parseCustomField(this.form, this.testCaseTemplate, this.customFieldForm, this.customFieldRules, testCase ? buildTestCaseOldFields(this.form) : null);
      this.setDefaultValue();
      // 重新渲染，显示自定义字段的必填校验
      this.reloadForm();
    },
    setTestCaseExtInfo(testCase) {
      this.testCase = {};
      if (testCase) {
        // 复制 不查询评论
        this.testCase = testCase.isCopy ? {} : testCase;
      }
    },
    close() {
      //移除监听，防止监听其他页面
      removeGoBackListener(this.close);
      this.dialogFormVisible = false;
    },
    saveCase(callback) {
      let isValidate = true;
      this.$refs['caseFrom'].validate((valid) => {
        if (!valid) {
          isValidate = false;
          return false;
        }
      });
      this.$refs['customFieldForm'].validate((valid) => {
        if (!valid) {
          isValidate = false;
          return false;
        }
      });
      if (isValidate) {
        this._saveCase(callback);
      }
    },
    _saveCase(callback) {
      let param = this.buildParam();
      if (this.validate(param)) {
        let option = this.getOption(param);
        this.result = this.$request(option, (response) => {
          this.$success(this.$t('commons.save_success'));
          this.path = "/test/case/edit";
          // this.operationType = "edit"
          this.$emit("refreshTestCase",);
          this.$store.state.testCaseMap.delete(this.form.id);
          //this.tableType = 'edit';
          this.$emit("refresh", this.form);
          if (this.form.id) {
            this.$emit("caseEdit", param);
          } else {
            param.id = response.data.id;
            this.$emit("caseCreate", param);
            this.close();
          }
          this.form.id = response.data.id;

          if (callback) {
            callback(this);
          }
          // 保存用例后刷新附件
        });
      }
    },
    buildParam() {
      let param = {};
      Object.assign(param, this.form);
      param.steps = JSON.stringify(this.form.steps);
      param.nodeId = this.form.module;
      param.nodePath = getNodePath(this.form.module, this.moduleOptions);
      if (this.projectId) {
        param.projectId = this.projectId;
      }
      param.name = param.name.trim();

      if (this.form.tags instanceof Array) {
        this.form.tags = JSON.stringify(this.form.tags);
      }
      param.testId = JSON.stringify(this.form.selected);
      param.tags = this.form.tags;
      param.type = 'functional';
      buildCustomFields(this.form, param, this.testCaseTemplate);
      this.parseOldFields(param);
      return param;
    },
    parseOldFields(param) {
      let customFieldsStr = param.customFields;
      if (customFieldsStr) {
        let customFields = JSON.parse(customFieldsStr);
        customFields.forEach(item => {
          if (item.name === '用例等级') {
            param.priority = item.value;
          }
          if (item.name === '责任人') {
            param.maintainer = item.value;
          }
          if (item.name === '用例状态') {
            param.status = item.value;
          }
        });
      }
    },
    getOption(param) {
      let formData = new FormData();
      if (this.$refs.otherInfo && this.$refs.otherInfo.uploadList) {
        this.$refs.otherInfo.uploadList.forEach(f => {
          formData.append("file", f);
        });
      }

      if (this.$refs.otherInfo && this.$refs.otherInfo.fileList) {
        if (param.isCopy) {
          // 如果是copy，则把文件的ID传到后台进行文件复制
          param.fileIds = this.$refs.otherInfo.fileList.map(f => f.id);
        }
        param.updatedFileList = this.$refs.otherInfo.fileList;
      } else {
        param.fileIds = [];
        param.updatedFileList = [];
      }

      let requestJson = JSON.stringify(param, function (key, value) {
        return key === "file" ? undefined : value
      });

      formData.append('request', new Blob([requestJson], {
        type: "application/json"
      }));
      return {
        method: 'POST',
        url: this.path,
        data: formData,
        headers: {
          'Content-Type': undefined
        }
      };
    },
    validate(param) {
      for (let i = 0; i < param.steps.length; i++) {
        if ((param.steps[i].desc && param.steps[i].desc.length > 300) ||
          (param.steps[i].result && param.steps[i].result.length > 300)) {
          this.$warning(this.$t('test_track.case.step_desc') + ","
            + this.$t('test_track.case.expected_results') + this.$t('test_track.length_less_than') + '300');
          return false;
        }
      }
      if (param.name === '') {
        this.$warning(this.$t('test_track.case.input_name'));
        return false;
      }
      return true;
    },
    typeChange() {
      this.form.testId = '';
    },
    getMaintainerOptions() {
      this.$post('/user/project/member/tester/list', {projectId: getCurrentProjectID()}, response => {
        this.maintainerOptions = response.data;
      });
    },
    getSelectOptions() {
      this.getMaintainerOptions();
    },
    resetForm() {
      //防止点击修改后，点击新建触发校验
      if (this.$refs['caseFrom']) {
        this.$refs['caseFrom'].validate((valid) => {
          this.$refs['caseFrom'].resetFields();
          this._resetForm();
          return true;
        });
      } else {
        this._resetForm();
      }
    },
    _resetForm() {
      this.form.name = '';
      this.form.module = '';
      this.form.type = '';
      this.form.method = '';
      this.form.maintainer = '';
      this.form.priority = '';
      this.form.prerequisite = '';
      this.form.remark = '';
      this.form.testId = '';
      this.form.testName = '';
      this.form.steps = [{
        num: 1,
        desc: '',
        result: ''
      }];
      this.form.customNum = '';
    },
    addListener() {
      document.addEventListener("keydown", this.createCtrlSHandle);
    },
    removeListener() {
      document.removeEventListener("keydown", this.createCtrlSHandle);
    },
    createCtrlSHandle(event) {
      let curTabId = this.$store.state.curTabId;
      if (curTabId === this.tabId)
        handleCtrlSEvent(event, this.saveCase);
    },
    saveFollow() {
      if (this.showFollow) {
        this.showFollow = false;
        for (let i = 0; i < this.form.follows.length; i++) {
          if (this.form.follows[i] === this.currentUser().id) {
            this.form.follows.splice(i, 1)
            break;
          }
        }
        if (this.path === "/test/case/edit") {
          this.result.loading = true
          this.$post('/test/case/edit/follows/' + this.form.id, this.form.follows, () => {
            this.result.loading = false
            this.$success(this.$t('commons.cancel_follow_success'));
          });
        }

      } else {
        this.showFollow = true;
        if(!this.form.follows){
          this.form.follows = [];
        }
        this.form.follows.push(this.currentUser().id)

        if (this.path === "/test/case/edit") {
          this.result.loading = true
          this.$post('/test/case/edit/follows/' + this.form.id, this.form.follows, () => {
            this.result.loading = false
            this.$success(this.$t('commons.follow_success'));
          });
        }
      }
    }
  }
}
</script>

<style scoped>

.el-switch {
  margin-bottom: 10px;
}

.case-name {
  width: 194px;
}

.container {
  height: 100vh;
}

.case-form {
  height: 95%;
  overflow: auto;
}

.case-dialog >>> .el-dialog__body {
  padding: 0 20px 10px 20px;
}

.container >>> .el-card__body {
  height: calc(100vh - 120px);
}

.comment-card >>> .el-card__header {
  padding: 27px 20px;
}

.comment-card >>> .el-card__body {
  height: calc(100vh - 120px);
}

.head-right {
  text-align: right;
}

.ms-main-div {
  background-color: white;
}

.ms-opt-btn {
  position: fixed;
  right: 50px;
  z-index: 1;
}

.ms-case-input {
  width: 100%;
}

.ms-case {
  width: 100%;
}

/deep/ .el-button-group > .el-button:first-child {
  border-top-right-radius: 0;
  border-bottom-right-radius: 0;
  height: 32px;
  width: 56px;
}

.other-info-tabs {
  padding-left: 60px;
  margin-left: 40px;
}
</style>
