<template>
  <div class="app-container">
    <div v-show="show">
      <el-form :model="queryParams" ref="queryForm" :inline="true" v-show="showSearch" label-width="100px" size="medium" class="ry_form">
        <el-form-item label="创建时间">
          <el-date-picker
            size="small"
            v-model="dateRange"
            style="width: 240px"
            value-format="yyyy-MM-dd"
            type="daterange"
            :clearable="true"
            :picker-options='pickerOptions'
            range-separator="-"
            start-placeholder="开始日期"
            end-placeholder="结束日期"
          ></el-date-picker>
        </el-form-item>
  <!--      <el-form-item label="账号启用状态">-->
  <!--        <el-select v-model="queryParams.status" placeholder="请选择" :clearable="true" size="small">-->
  <!--          <el-option label="禁用" value="0">-->
  <!--          </el-option>-->
  <!--          <el-option label="启用" value="1">-->
  <!--          </el-option>-->
  <!--        </el-select>-->
  <!--      </el-form-item>-->
        <el-form-item label="昵称" prop="nickname">
          <el-input
            v-model.trim="queryParams.nickname"
            placeholder="请输入昵称"
            clearable
            size="small"
          />
        </el-form-item>
        <el-form-item label="手机号码" prop="phone">
          <el-input
            v-model.trim="queryParams.phone"
            placeholder="请输入手机号码"
            clearable
            size="small"
          />
        </el-form-item>
        <el-form-item label="备注" prop="mark">
          <el-select v-model="queryParams.hasMark" clearable size="small">
            <el-option value="1" label="有备注" />
            <el-option value="0" label="无备注"/>
          </el-select>
        </el-form-item>
        <el-form-item class="flex_one tr">
          <el-button type="primary" icon="el-icon-search" size="mini" @click="handleQuery">搜索</el-button>
          <el-button icon="el-icon-refresh" size="mini" @click="resetQuery">重置</el-button>
          <!--        <el-button :icon="showMoreCondition ? 'el-icon-arrow-up' : 'el-icon-arrow-down'" size="mini" @click="showMoreCondition = !showMoreCondition">{{showMoreCondition ? '收起条件' : '展开条件'}}</el-button>-->
        </el-form-item>
      </el-form>

      <el-table v-loading="loading" :data="umsMemberList" border>
        <el-table-column label="昵称" prop="nickname" width="150"/>
        <el-table-column label="手机号码" prop="phoneHidden" width="150"/>
        <el-table-column label="佣金" width="120">
          <template v-slot="scope">
            <div>0.00</div>
          </template>
        </el-table-column>
        <el-table-column label="积分" width="120">
          <template v-slot="scope">
            <div>0.00</div>
          </template>
        </el-table-column>
        <el-table-column label="余额" width="120">
          <template v-slot="scope">
            <div>0.00</div>
          </template>
        </el-table-column>
        <el-table-column label="注册时间" prop="createTime" width="180">
          <template v-slot="scope">
            <div>{{ parseTime(scope.row.createTime) }}</div>
          </template>
        </el-table-column>
        <el-table-column label="上次登录" prop="createTime">
          <template v-slot="scope">
            <div>{{ parseTime(scope.row.createTime) }}</div>
          </template>
        </el-table-column>
        <el-table-column label="备注">
          <template v-slot="scope">
            <span class="mr10">{{scope.row.mark}}</span>
            <i class="el-icon-edit pointer" @click="showUpdateMark(scope.row)"></i>
          </template>
        </el-table-column>
        <el-table-column label="操作" class-name="small-padding fixed-width" fix="right" width="200">
          <template slot-scope="scope">
            <el-button
              size="mini"
              type="text"
              @click="showStatistics(scope.row.id)"
              v-hasPermi="['ums:member:statistics']"
            >查看数据
            </el-button>
            <el-button
              size="mini"
              type="text"
              @click="goOrder(scope.row.phoneEncrypted)"
            >查看下单</el-button>
            <el-button
              size="mini"
              type="text"
              @click="goCart(scope.row.phoneEncrypted)"
            >查看购物车</el-button>
          </template>
        </el-table-column>
      </el-table>
      <InBody v-show="total>0">
      <pagination
        :total="total"
        :page.sync="queryParams.pageNum"
        :limit.sync="queryParams.pageSize"
        @pagination="getList"
      />
    </InBody>
    </div>
    <!--  统计  -->
    <el-dialog :title="statisticsObj.title" :visible.sync="statisticsObj.open" width="500px" append-to-body>
      <el-descriptions direction="vertical" :column="2" border>
        <el-descriptions-item label="购物车数">{{ statisticsObj.data.cartCount }}</el-descriptions-item>
        <el-descriptions-item label="订单数">{{ statisticsObj.data.orderCount }}</el-descriptions-item>
        <el-descriptions-item label="下单金额">￥{{ statisticsObj.data.orderAmount.toFixed(2) }}</el-descriptions-item>
        <el-descriptions-item label="售后数">{{ statisticsObj.data.aftersaleCount }}</el-descriptions-item>
      </el-descriptions>
    </el-dialog>
    <el-dialog title="修改备注" :visible.sync="remarkModal.visible" width="30%" append-to-body>
      <el-input type='textarea' :rows='3' placeholder='请输入内容' v-model='remarkModal.mark'/>
      <span class="dialog-footer" slot="footer">
        <el-button @click="remarkModal.visible = false">取 消</el-button>
        <el-button type='primary' @click='updateRemark'>确 定</el-button>
      </span>
    </el-dialog>
  </div>
</template>

<script>
import {
  addUmsMember,
  changeAccountStatus,
  decryptedPhone,
  delUmsMember,
  exportUmsMember,
  getUmsMember,
  listUmsMember,
  updateUmsMember,
  updateUmsMemberMark,
  viewStatistics
} from "@/api/ums/member";
import dateUtil from '@/utils/DateUtil';
import {isStarRepo} from "@/utils/is-star-plugin";
import {mapGetters} from "vuex";
import SeeAdsComponent from "@/components/SeeAdsComponent.vue";

export default {
  name: "UmsMember",
  components: {SeeAdsComponent},
  data() {
    return {
      show: false,
      pickerOptions: {
        shortcuts: dateUtil.getTimeShort()
      },
      remarkModal: {
        visible: false,
        mark: null,
        memberId: null,
      },
      // 遮罩层
      loading: true,
      // 导出遮罩层
      exportLoading: false,
      // 选中数组
      ids: [],
      // 非单个禁用
      single: true,
      // 非多个禁用
      multiple: true,
      // 显示搜索条件
      showSearch: true,
      // 总条数
      total: 0,
      // 会员信息表格数据
      umsMemberList: [],
      // 弹出层标题
      title: "",
      // 是否显示弹出层
      open: false,
      // 查询参数
      queryParams: {
        pageNum: 1,
        pageSize: 10,
        nickname: null,
        phone: null,
        status: undefined,
        hasMark: undefined
      },
      dateRange:[],
      // 表单参数
      form: {},
      // 表单校验
      rules: {
        level: [
          { required: true, message: "等级不能为空", trigger: "blur" }
        ],
        integral: [
          { required: true, message: "用户剩余积分不能为空", trigger: "blur" }
        ],
      },
      showMoreCondition: false,
      statisticsObj: {
        open: false,
        data: {
          cartCount: 0,
          orderCount: 0,
          orderAmount: 0.00,
          aftersaleCount: 0
        },
        title: '用户数据统计'
      }
    };
  },
  async created() {
    this.$nextTick(()=>{
      this.$refs.seeAdsComponentRef.show()
    })
  },
  computed:{
    ...mapGetters(['userId']),
  },
  methods: {
    // async confirmOk(success) {
    //   if (success) {
    //     const res = await isStarRepo('zccbbg', 'RuoYi-Mall', this.userId, 'https://www.siyuanedu.top/member/member', 'mall-商城', 'https://gitee.com/zccbbg/RuoYi-Mall')
    //     this.show = res;
    //     if (res) {
    //       this.getList();
    //     }
    //   }
    // },
    showUpdateMark(record){
      this.remarkModal = {
        visible: true,
        mark: record.mark,
        memberId: record.id
      }
    },
    updateRemark(){
      updateUmsMemberMark({id:this.remarkModal.memberId,mark:this.remarkModal.mark})
        .then(res=>{
          if (res > 0) {
            this.$message.success('修改成功');
            this.remarkModal.visible = false;
            const obj = this.umsMemberList.filter(it=>it.id === this.remarkModal.memberId)[0]
            obj.mark = this.remarkModal.mark;
          } else {
            this.$message.success('修改失败');
          }
        })
    },
    /** 查询会员信息列表 */
    getList() {
      this.loading = true;
      const {pageNum, pageSize} = this.queryParams;
      let query = {...this.queryParams, pageNum: undefined, pageSize: undefined};
      const pageReq = {page: pageNum - 1, size: pageSize};
      if (!this.dateRange || this.dateRange.length > 0){
        query = { ...this.addDateRange2(query, this.dateRange) }
      }
      listUmsMember(query, pageReq).then(response => {
        const { content, totalElements } = response
        this.umsMemberList = content;
        this.total = totalElements;
        this.loading = false;
      });
    },
    // 取消按钮
    cancel() {
      this.open = false;
      this.reset();
    },
    // 表单重置
    reset() {
      this.form = {
        id: null,
        nickname: null,
        password: null,
        phone: null,
        mark: null,
        status: 0,
        avatar: null,
        gender: null,
        city: null,
        province: null,
        country: null,
        remark: null,
        birthday: null,
        spreadUid: null,
        spreadTime: null,
        level: null,
        integral: null,
        createBy: null,
        createTime: null,
        updateBy: null,
        updateTime: null
      };
      this.resetForm("form");
    },
    /** 搜索按钮操作 */
    handleQuery() {
      this.queryParams.pageNum = 1;
      this.getList();
    },
    /** 重置按钮操作 */
    resetQuery() {
      this.resetForm("queryForm");
      this.handleQuery();
    },
    // 多选框选中数据
    handleSelectionChange(selection) {
      this.ids = selection.map(item => item.id)
      this.single = selection.length!==1
      this.multiple = !selection.length
    },
    /** 新增按钮操作 */
    handleAdd() {
      this.reset();
      this.open = true;
      this.title = "添加会员信息";
    },
    /** 修改按钮操作 */
    handleUpdate(row) {
      this.reset();
      const id = row.id || this.ids
      getUmsMember(id).then(response => {
        this.form = response;
        this.open = true;
        this.title = "修改会员信息";
      });
    },
    /** 提交按钮 */
    submitForm() {
      this.$refs["form"].validate(valid => {
        if (valid) {
          if (this.form.id != null) {
            updateUmsMember(this.form).then(response => {
              this.$modal.msgSuccess("修改成功");
              this.open = false;
              this.getList();
            });
          } else {
            addUmsMember(this.form).then(response => {
              this.$modal.msgSuccess("新增成功");
              this.open = false;
              this.getList();
            });
          }
        }
      });
    },
    /** 删除按钮操作 */
    handleDelete(row) {
      const ids = row.id || this.ids;
      this.$modal.confirm('是否确认删除会员信息编号为"' + ids + '"的数据项？').then(function() {
        return delUmsMember(ids);
      }).then(() => {
        this.getList();
        this.$modal.msgSuccess("删除成功");
      }).catch(() => {});
    },
    /** 导出按钮操作 */
    handleExport() {
      const queryParams = this.queryParams;
      this.$modal.confirm('是否确认导出所有会员信息数据项？').then(() => {
        this.exportLoading = true;
        return exportUmsMember(queryParams);
      }).then(response => {
        this.$download.download(response);
        this.exportLoading = false;
      }).catch(() => {});
    },
    // 更改账户状态
    changeStatus(row){
      const data = {
        memberId: row.id,
        status: row.status
      }
      changeAccountStatus(data).then(response => {
        if (response < 1){
          this.$modal.msgError('操作失败')
          this.getList()
        }
      })
    },
    goOrder(phone){
      decryptedPhone(phone).then(res => {
        this.$router.push({
          path: '/order/order',
          query: {
            phone: res
          }
        })
      })
    },
    goCart(phone){
      decryptedPhone(phone).then(res => {
        this.$router.push({
          path: '/member/shoppingCart',
          query: {
            phone: res
          }
        })
      })
    },
    showStatistics(memberId){
      viewStatistics(memberId).then((response) => {
        this.statisticsObj.data = response
        this.statisticsObj.open = true
      })
    }
  }
};
</script>
