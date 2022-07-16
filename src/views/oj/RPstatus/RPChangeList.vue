<template>
  <el-row class="flex-container">
    <div id="main">
      <el-card shadow>
        <div slot="header">
          <el-row :gutter="18">
            <el-col :md="4" :lg="2">
              <span class="panel-title hidden-md-and-down">{{
                  $t('m.Status')
                }}</span>
            </el-col>
            <el-col :xs="10" :sm="8" :md="4" :lg="4">
              <el-switch
                  style="display: block"
                  v-model="formFilter.onlyMine"
                  :active-text="$t('m.Mine')"
                  :width="40"
                  @change="handleOnlyMine"
                  :inactive-text="$t('m.All')"
              >
              </el-switch>
            </el-col>
            <el-col :sm="8" :md="5" :lg="4" class="hidden-xs-only">
              <el-button
                  type="primary"
                  size="small"
                  icon="el-icon-refresh"
                  round
                  @click="getRPChange"
              >{{ $t('m.Refresh') }}</el-button
              >
            </el-col>
            <el-col :xs="4" class="hidden-sm-and-up">
              <el-button
                  type="primary"
                  size="small"
                  icon="el-icon-refresh"
                  circle
                  @click="getRPChange"
              ></el-button>
            </el-col>

            <el-col :xs="24" :sm="12" :md="5" :lg="5" class="search">
              <vxe-input
                  v-model="formFilter.problemID"
                  :placeholder="$t('m.Enter_Problem_ID')"
                  type="search"
                  size="medium"
                  @keyup.enter.native="handleQueryChange('probemID')"
                  @search-click="handleQueryChange('probemID')"
              ></vxe-input>
            </el-col>
            <el-col :xs="24" :sm="12" :md="5" :lg="5" class="search">
              <vxe-input
                  v-model="formFilter.username"
                  :disabled="formFilter.onlyMine"
                  :placeholder="$t('m.Enter_Author')"
                  type="search"
                  size="medium"
                  @keyup.enter.native="handleQueryChange('username')"
                  @search-click="handleQueryChange('username')"
              ></vxe-input>
            </el-col>
          </el-row>
        </div>
        <vxe-table
            border="inner"
            stripe
            keep-source
            ref="xTable"
            highlight-current-row
            highlight-hover-row
            align="center"
            auto-resize
            :row-class-name="tableRowClassName"
            :data="RpChanges"
            :loading="loadingTable"
        >
          <vxe-table-column
              field="RPChangeID"
              :title="$t('m.RPChange_ID')"
              width="100"
          ></vxe-table-column>

          <vxe-table-column
              field="RPChange"
              :title="$t('m.RPChange')"
              min-width="80"
          >
          </vxe-table-column>

          <vxe-table-column
              field="username"
              :title="$t('m.User')"
              min-width="96"
              show-overflow
          >
            <template v-slot="{ row }">
              <a
                  @click="goUserHome(row.username, row.uid)"
                  style="color: rgb(87, 163, 243)"
              >{{ row.username }}</a
              >
            </template>
          </vxe-table-column>
          <vxe-table-column
              field="RPChangeTime"
              :title="$t('m.RPChange_Time')"
              min-width="96"
          >
          </vxe-table-column>
          <vxe-table-column
              field="Description"
              :title="$t('m.Description')"
              min-width="80"
          >
          </vxe-table-column>
        </vxe-table>
      </el-card>
      <Pagination
          :total="total"
          :page-size="limit"
          @on-change="changeRoute"
          :current.sync="currentPage"
          @on-page-size-change="onPageSizeChange"
          :layout="'prev, pager, next, sizes'"
      ></Pagination>
    </div>
  </el-row>
</template>

<script>
import { mapActions, mapGetters } from 'vuex';
import api from '@/common/api';
import utils from '@/common/utils';
import Pagination from '@/components/oj/common/Pagination';
import myMessage from '@/common/message';
import 'element-ui/lib/theme-chalk/display.css';
export default {
  name: 'RPChangeList',
  components: {
    Pagination,
  },
  data() {
    return {
      formFilter: {
        onlyMine: false,
        status: '',
        username: '',
        problemID: '',
      },
      loadingTable: false,
      RPchanges: [],
      needCheckSubmitIds: {}, // 当前状态为6和7的提交记录Id 需要重新检查更新
      total: 30,
      limit: 15,
      currentPage: 1,
      contestID: null,
      groupID: null,
      routeName: '',
      checkStatusNum: 0,
      JUDGE_STATUS: '',
      JUDGE_STATUS_LIST: '',
      autoCheckOpen: false,
      JUDGE_STATUS_RESERVE: {},
      CONTEST_STATUS: {},
      RULE_TYPE: {},
    };
  },
  created() {
    this.init();
  },
  mounted() {
    this.JUDGE_STATUS = Object.assign({}, JUDGE_STATUS);
    this.JUDGE_STATUS_RESERVE = Object.assign({}, JUDGE_STATUS_RESERVE);
    this.CONTEST_STATUS = Object.assign({}, CONTEST_STATUS);
    this.RULE_TYPE = Object.assign({}, RULE_TYPE);
    this.getData();
  },
  methods: {
    init() {
      this.checkStatusNum = 0;
      this.contestID = this.$route.params.contestID;
      this.groupID = this.$route.params.groupID;
      let query = this.$route.query;
      this.formFilter.problemID = query.problemID;
      this.formFilter.username = query.username || '';
      this.formFilter.onlyMine = query.onlyMine + '' == 'true' ? true : false; // 统一换成字符串判断
      this.formFilter.status = query.status;
      this.formFilter.completeProblemID = query.completeProblemID || false;
      if (this.formFilter.onlyMine) {
        // 当前为搜索自己的提交 那么不可搜索别人的提交
        this.formFilter.username = '';
      }
      this.currentPage = parseInt(query.currentPage) || 1;
      if (this.currentPage < 1) {
        this.currentPage = 1;
      }
      this.limit = parseInt(query.limit) || 15;
      this.routeName = this.$route.name;
    },

    getData() {
      this.getSubmissions();
    },

    buildQuery() {
      return {
        onlyMine: this.formFilter.onlyMine,
        status: this.formFilter.status,
        username: this.formFilter.username,
        problemID: this.formFilter.problemID,
        currentPage: this.currentPage,
        limit: this.limit,
        completeProblemID: this.formFilter.completeProblemID,
      };
    },

    getRPChange() {
      let params = this.buildQuery();
      if (this.formFilter.onlyMine) {
        // 需要判断是否为登陆状态
        if (this.isAuthenticated) {
          params.username = ''; // 如果是搜索当前用户的提交记录，那么用户名搜索应该无效
          this.formFilter.username = '';
        } else {
          this.formFilter.onlyMine = false;
          myMessage.error(this.$i18n.t('m.Please_login_first'));
          return;
        }
      }
      this.loadingTable = true;
      this.submissions = [];
      this.needCheckSubmitIds = {};
      let func = 'getRPChangeList';
      api[func](this.limit, utils.filterEmptyValue(params))
          .then((res) => {
            let data = res.data.data;
            let index = 0;
            this.loadingTable = false;
            this.submissions = data.records;
            this.total = data.total;
          })
          .catch(() => {
            this.loadingTable = false;
          });
    },
    onPageSizeChange(pageSize) {
      this.limit = pageSize;
      this.changeRoute();
    },
    // 改变route， 通过监听route变化请求数据，这样可以产生route history， 用户返回时就会保存之前的状态
    changeRoute() {
      let query = this.buildQuery();
      query.contestID = this.contestID;
      let queryParams = utils.filterEmptyValue(query);
      // 判断新路径请求参数与当前路径请求的参数是否一致，避免重复访问路由报错
      let equal = true;
      for (let key in queryParams) {
        if (queryParams[key] != this.$route.query[key]) {
          equal = false;
          break;
        }
      }
      if (equal) {
        // 判断请求参数的长短
        if (
            Object.keys(queryParams).length !=
            Object.keys(this.$route.query).length
        ) {
          equal = false;
        }
      }
    },
    goRoute(route) {
      this.$router.push(route);
    },
    goUserHome(username, uid) {
      this.$router.push({
        path: '/user-home',
        query: { uid, username },
      });
    },
    handleStatusChange(status) {
      if (status == 'All') {
        this.formFilter.status = '';
      } else {
        this.formFilter.status = status;
      }
      this.currentPage = 1;
      this.changeRoute();
    },
    handleQueryChange(searchParam) {
      if (searchParam == 'probemID') {
        this.formFilter.completeProblemID = false; // 并非走完全检索displayID了
      }
      this.currentPage = 1;
      this.changeRoute();
    },
    handleOnlyMine() {
      if (this.formFilter.onlyMine) {
        // 需要判断是否为登陆状态
        if (this.isAuthenticated) {
          this.formFilter.username = '';
        } else {
          this.formFilter.onlyMine = false;
          myMessage.error(this.$i18n.t('m.Please_login_first'));
          return;
        }
      }
      this.currentPage = 1;
      this.changeRoute();
    },
    ...mapActions(['changeModalStatus']),
    getProblemUri(pid) {
      if (this.contestID) {
        this.$router.push({
          name: 'ContestProblemDetails',
          params: {
            contestID: this.$route.params.contestID,
            problemID: pid,
          },
        });
      } else if (this.groupID) {
        this.$router.push({
          name: 'GroupProblemDetails',
          params: {
            problemID: pid,
            groupID: this.groupID,
          },
        });
      } else {
        this.$router.push({
          name: 'ProblemDetails',
          params: {
            problemID: pid,
          },
        });
      }
    },

    tableRowClassName({ row, rowIndex }) {
      if (row.username == this.userInfo.username && this.isAuthenticated) {
        return 'own-submit-row';
      }
    },
  },
  computed: {
    ...mapGetters([
      'isAuthenticated',
      'userInfo',
      'isSuperAdmin',
      'isAdminRole',
      'contestRuleType',
      'contestStatus',
      'ContestRealTimePermission',
    ]),
    title() {
      if (!this.contestID) {
        return 'Status';
      } else if (this.problemID) {
        return 'Problem Submissions';
      } else {
        return 'Submissions';
      }
    },
    status() {
      return this.formFilter.status === ''
          ? this.$i18n.t('m.RPStatus')
          : JUDGE_STATUS[this.formFilter.status]
              ? JUDGE_STATUS[this.formFilter.status].name
              : this.$i18n.t('m.RPStatus');
    },
    rejudgeColumnVisible() {
      return this.isSuperAdmin;
    },
    scoreColumnVisible() {
      return (
          (this.contestID && this.contestRuleType == this.RULE_TYPE.OI) ||
          !this.contestID
      );
    },
  },
  watch: {
    $route(newVal, oldVal) {
      if (newVal !== oldVal) {
        if (this.autoCheckOpen) {
          clearInterval(this.refreshStatus);
        }
        this.init();
        this.getData();
      }
    },
    isAuthenticated() {
      this.init();
      this.getData();
    },
  },
  beforeRouteLeave(to, from, next) {
    // 防止切换组件后仍然不断请求
    clearInterval(this.refreshStatus);
    next();
  },
};
</script>

<style scoped>
@media only screen and (max-width: 767px) {
  .search {
    margin-top: 20px;
  }
}

.flex-container #main {
  flex: auto;
}
.flex-container .filter {
  margin-right: -10px;
}
.flex-container #contest-menu {
  flex: none;
  width: 210px;
}
/deep/ .el-card__header {
  border-bottom: 0px;
  padding-bottom: 0px;
  text-align: center;
}

/deep/ .el-dialog {
  border-radius: 6px !important;
  text-align: center;
}
/deep/ .el-switch {
  padding-top: 6px;
}
@media only screen and (min-width: 768px) and (max-width: 992px) {
  .el-col-sm-12 {
    padding-top: 10px;
  }
}
@media screen and (min-width: 1050px) {
  /deep/ .vxe-table--body-wrapper {
    overflow-x: hidden !important;
  }
}
</style>
