<template>
  <el-card style="margin-top: 5px" @click.native="selectTestCase(apiCase,$event)">
    <el-row>
      <el-col :span="4">

        <el-checkbox v-model="apiCase.selected"/>

        <div class="el-step__icon is-text ms-api-col">
          <div class="el-step__icon-inner">{{ index + 1 }}</div>
        </div>

        <label class="ms-api-label">{{ $t('test_track.case.priority') }}</label>
        <el-select size="small" v-model="apiCase.priority" class="ms-api-select" @change="changePriority(apiCase)">
          <el-option v-for="grd in priorities" :key="grd.id" :label="grd.name" :value="grd.id"/>
        </el-select>
      </el-col>
      <el-col :span="12">
        <i class="icon el-icon-arrow-right" :class="{'is-active': apiCase.active}"
           @click="active(apiCase)"/>
        <el-input v-if="!apiCase.id || isShowInput" size="small" v-model="apiCase.name" :name="index" :key="index"
                  class="ms-api-header-select" style="width: 180px"
                  @blur="saveTestCase(apiCase)" :placeholder="$t('commons.input_name')"/>
        <span v-else>
          {{ apiCase.id ? apiCase.name : '' }}
          <i class="el-icon-edit" style="cursor:pointer" @click="showInput(apiCase)" v-tester/>
        </span>

        <label class="ms-api-label" style="padding-left: 20px; padding-right: 20px;">{{ $t('commons.tag') }}</label>
        <ms-input-tag :currentScenario="apiCase" ref="tag" style="float: right;margin-right: 215px;margin-top: -3px;" @keyup.enter.native="saveTestCase(apiCase)"/>

        <div v-if="apiCase.id" style="color: #999999;font-size: 12px">
          <span>
            {{ apiCase.createTime | timestampFormatDate }}
            {{ apiCase.createUser }} {{ $t('api_test.definition.request.create_info') }}
          </span>
          <span>
            {{ apiCase.updateTime | timestampFormatDate }}
            {{ apiCase.updateUser }} {{ $t('api_test.definition.request.update_info') }}
          </span>
        </div>
      </el-col>

      <el-col :span="4">
        <ms-tip-button @click="singleRun(apiCase)" :tip="$t('api_test.run')" icon="el-icon-video-play"
                       style="background-color: #409EFF;color: white" size="mini" :disabled="!apiCase.id" circle v-tester/>
        <ms-tip-button @click="copyCase(apiCase)" :tip="$t('commons.copy')" icon="el-icon-document-copy"
                       size="mini" :disabled="!apiCase.id || isCaseEdit" circle v-tester/>
        <ms-tip-button @click="deleteCase(index,apiCase)" :tip="$t('commons.delete')" icon="el-icon-delete"
                       size="mini" :disabled="!apiCase.id || isCaseEdit" circle v-tester/>
        <ms-api-extend-btns :is-case-edit="isCaseEdit" :environment="environment" :row="apiCase" v-tester/>
      </el-col>

      <el-col :span="3">
        <el-link type="danger" v-if="apiCase.execResult && apiCase.execResult==='error'" @click="showExecResult(apiCase)">
          {{ getResult(apiCase.execResult) }}
        </el-link>
        <el-link v-else-if="apiCase.execResult && apiCase.execResult==='success'" @click="showExecResult(apiCase)">
          {{ getResult(apiCase.execResult) }}
        </el-link>
        <div v-else> {{ getResult(apiCase.execResult) }}</div>

        <div v-if="apiCase.id" style="color: #999999;font-size: 12px">
          <span> {{ apiCase.execTime | timestampFormatDate }}</span>
          {{ apiCase.updateUser }}
        </div>
      </el-col>
    </el-row>
    <!-- 请求参数-->
    <el-collapse-transition>
      <div v-if="apiCase.active">
        <p class="tip">{{ $t('api_test.definition.request.req_param') }} </p>

        <ms-api-request-form :showScript="true" :is-read-only="isReadOnly" :headers="apiCase.request.headers " :request="apiCase.request" v-if="api.protocol==='HTTP'"/>
        <ms-tcp-basis-parameters :showScript="true" :request="apiCase.request" v-if="api.protocol==='TCP'"/>
        <ms-sql-basis-parameters :showScript="true" :request="apiCase.request" v-if="api.protocol==='SQL'"/>
        <ms-dubbo-basis-parameters :showScript="true" :request="apiCase.request" v-if="api.protocol==='DUBBO'"/>

        <!-- HTTP 请求返回数据 -->
        <p class="tip">
          {{$t('api_test.definition.request.res_param')}}
          <i class="icon el-icon-arrow-right" :class="{'is-responseActive': apiCase.responseActive}" @click="responseActive(apiCase)"/>
        </p>
        <el-collapse-transition>
          <div v-if="apiCase.responseActive">
            <ms-request-result-tail :response="responseData"/>
          </div>
        </el-collapse-transition>
        <!-- 保存操作 -->
        <el-button type="primary" size="small" style="margin: 20px; float: right" @click="saveTestCase(apiCase)" v-tester>
          {{ $t('commons.save') }}
        </el-button>
      </div>
    </el-collapse-transition>
  </el-card>
</template>

<script>
  import {_getBodyUploadFiles, getCurrentProjectID} from "../../../../../../common/js/utils";
  import {PRIORITY, RESULT_MAP} from "../../model/JsonData";
  import MsTag from "../../../../common/components/MsTag";
  import MsTipButton from "../../../../common/components/MsTipButton";
  import MsApiRequestForm from "../request/http/ApiHttpRequestForm";
  import ApiEnvironmentConfig from "../environment/ApiEnvironmentConfig";
  import MsApiAssertions from "../assertion/ApiAssertions";
  import MsSqlBasisParameters from "../request/database/BasisParameters";
  import MsTcpBasisParameters from "../request/tcp/TcpBasisParameters";
  import MsDubboBasisParameters from "../request/dubbo/BasisParameters";
  import MsApiExtendBtns from "../reference/ApiExtendBtns";
  import MsInputTag from "@/business/components/api/automation/scenario/MsInputTag";
  import MsRequestResultTail from "../response/RequestResultTail";

  export default {
    name: "ApiCaseItem",
    components: {
      MsInputTag,
      MsTag,
      MsTipButton,
      MsApiRequestForm,
      ApiEnvironmentConfig,
      MsApiAssertions,
      MsSqlBasisParameters,
      MsTcpBasisParameters,
      MsDubboBasisParameters,
      MsApiExtendBtns,
      MsRequestResultTail
    },
    data() {
      return {
        result: {},
        grades: [],
        isReadOnly: false,
        selectedEvent: Object,
        priorities: PRIORITY,
        runData: [],
        reportId: "",
        checkedCases: new Set(),
        visible: false,
        condition: {},
        responseData: {type: 'HTTP', responseResult: {}, subRequestResults: []},
        isShowInput: false,
      }
    },
    props: {
      apiCase: {
        type: Object,
        default() {
          return {}
        },
      },
      environment: {},
      index: {
        type: Number,
        default() {
          return 0
        }
      },
      api: {
        type: Object,
        default() {
          return {}
        }
      },
      isCaseEdit: Boolean,
    },
    watch: {},
    methods: {
      deleteCase(index, row) {
        this.$alert(this.$t('api_test.definition.request.delete_confirm') + ' ' + row.name + " ？", '', {
          confirmButtonText: this.$t('commons.confirm'),
          callback: (action) => {
            if (action === 'confirm') {
              this.$get('/api/testcase/delete/' + row.id, () => {
                this.$success(this.$t('commons.delete_success'));
                this.$emit('refresh');
              });
            }
          }
        });
      },
      getExecResult() {
        // 执行结果信息
        if (this.apiCase) {
          let url = "/api/definition/report/getReport/" + this.apiCase.id;
          this.$get(url, response => {
            if (response.data) {
              let data = JSON.parse(response.data.content);
              this.responseData = data;
            }
          });
        }
      },
      singleRun(data) {
        data.message = true;
        this.saveTestCase(data);
        this.$emit('singleRun', data);
      },
      copyCase(data) {
        let obj = {name: "copy_" + data.name, priority: data.priority, active: true, request: data.request};
        this.$emit('copyCase', obj);
      },
      selectTestCase(item, $event) {
        if (!item.id || !this.loaded) {
          return;
        }
        if ($event.currentTarget.className.indexOf('is-selected') > 0) {
          $event.currentTarget.className = "el-card is-always-shadow";
          this.$emit('selectTestCase', null);
        } else {
          if (this.selectedEvent.currentTarget != undefined) {
            this.selectedEvent.currentTarget.className = "el-card is-always-shadow";
          }
          this.selectedEvent.currentTarget = $event.currentTarget;
          $event.currentTarget.className = "el-card is-always-shadow is-selected";
          this.$emit('selectTestCase', item);
        }
      },
      changePriority(row) {
        if (row.id) {
          this.saveTestCase(row);
        }
      },
      saveTestCase(row) {
        let tmp = JSON.parse(JSON.stringify(row));
        this.isShowInput = false;
        if (this.validate(tmp)) {
          return;
        }
        tmp.request.body = row.request.body;
        let bodyFiles = this.getBodyUploadFiles(tmp);
        tmp.projectId = getCurrentProjectID();
        tmp.active = true;
        tmp.request.path = this.api.path;
        tmp.request.method = this.api.method;
        tmp.apiDefinitionId = tmp.apiDefinitionId || this.api.id;
        let url = "/api/testcase/create";
        if (tmp.id) {
          url = "/api/testcase/update";
        }
        if (tmp.tags instanceof Array) {
          tmp.tags = JSON.stringify(tmp.tags);
        }
        this.$fileUpload(url, null, bodyFiles, tmp, (response) => {
          let data = response.data;
          row.id = data.id;
          row.createTime = data.createTime;
          row.updateTime = data.updateTime;
          if (!row.message) {
            this.$success(this.$t('commons.save_success'));
          }
        });
      },
      showInput(row) {
        // row.type = "create";
        this.isShowInput = true;
        row.active = true;
        this.active(row);
      },
      active(item) {
        item.active = !item.active;
      },
      responseActive(item) {
        item.responseActive = !item.responseActive;
        if (item.responseActive) {
          this.getExecResult();
        }
      },
      getResult(data) {
        if (RESULT_MAP.get(data)) {
          return RESULT_MAP.get(data);
        } else {
          return RESULT_MAP.get("default");
        }
      },
      validate(row) {
        if (!row.name) {
          this.$warning(this.$t('api_test.input_name'));
          return true;
        }
      },
      showExecResult(data) {
        this.$emit('showExecResult', data);
      },
      getBodyUploadFiles(row) {
        let bodyUploadFiles = [];
        row.bodyUploadIds = [];
        _getBodyUploadFiles(row.request, bodyUploadFiles, row);
        return bodyUploadFiles;
      },
    }
  }
</script>

<style scoped>
  .ms-api-select {
    margin-left: 20px;
    width: 80px;
  }

  .ms-api-header-select {
    margin-left: 20px;
    min-width: 100px;
  }

  .ms-api-label {
    color: #CCCCCC;
  }

  .ms-api-col {
    background-color: #7C3985;
    border-color: #7C3985;
    margin-right: 10px;
    color: white;
  }

  .icon.is-active {
    transform: rotate(90deg);
  }

  .icon.is-responseActive {
    transform: rotate(90deg);
  }

  .tip {
    padding: 3px 5px;
    font-size: 16px;
    border-radius: 4px;
    border-left: 4px solid #783887;
    margin: 20px 0;
  }

  .is-selected {
    background: #EFF7FF;
  }
</style>
