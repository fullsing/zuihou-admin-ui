<template>
  <div class="app-container">
    <div class="filter-container">
      <el-input
        v-model="queryParams.tenantCode"
        :placeholder="$t(&quot;table.globalUser.tenantCode&quot;)"
        class="filter-item search-item"
      />
      <el-input
        v-model="queryParams.account"
        :placeholder="$t(&quot;table.globalUser.account&quot;)"
        class="filter-item search-item"
      />
      <el-input
        v-model="queryParams.name"
        :placeholder="$t(&quot;table.globalUser.name&quot;)"
        class="filter-item search-item"
      />
      <el-date-picker
        v-model="queryParams.timeRange"
        :range-separator="null"
        :start-placeholder="$t(&quot;table.createTime&quot;)"
        value-format="yyyy-MM-dd HH:mm:ss"
        format="yyyy-MM-dd HH:mm:ss"
        class="filter-item search-item date-range-item"
        type="daterange"
      />
      <el-button class="filter-item" type="primary" plain @click="search">{{ $t('table.search') }}</el-button>
      <el-button class="filter-item" type="warning" plain @click="reset">{{ $t('table.reset') }}</el-button>
      <el-dropdown trigger="click" class="filter-item">
        <el-button>
          {{ $t('table.more') }}
          <i class="el-icon-arrow-down el-icon--right" />
        </el-button>
        <el-dropdown-menu slot="dropdown">
          <el-dropdown-item
            icon="el-icon-circle-plus-outline"
            @click.native="add"
          >{{ $t('table.add') }}</el-dropdown-item>
          <el-dropdown-item
            icon="el-icon-delete"
            @click.native="batchDelete"
          >{{ $t('table.delete') }}</el-dropdown-item>
        </el-dropdown-menu>
      </el-dropdown>
    </div>

    <el-table
      ref="table"
      :key="tableKey"
      v-loading="loading"
      :data="tableData.records"
      border
      fit
      style="width: 100%;"
      @selection-change="onSelectChange"
      @sort-change="sortChange"
    >
      <el-table-column type="selection" align="center" width="40px" />
      <el-table-column
        :label="$t(&quot;table.globalUser.tenantCode&quot;)"
        prop="tenantCode"
        :show-overflow-tooltip="true"
        align="center"
        width="100px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.tenantCode }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t(&quot;table.globalUser.account&quot;)"
        prop="account"
        :show-overflow-tooltip="true"
        class-name="status-col"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.account }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t(&quot;table.globalUser.name&quot;)" class-name="status-col">
        <template slot-scope="scope">
          <span>{{ scope.row.name }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t(&quot;table.globalUser.mobile&quot;)" class-name="status-col">
        <template slot-scope="scope">
          <span>{{ scope.row.mobile }}</span>
        </template>
      </el-table-column>
      <el-table-column :label="$t(&quot;table.globalUser.email&quot;)" class-name="status-col">
        <template slot-scope="scope">
          <span>{{ scope.row.email }}</span>
        </template>
      </el-table-column>
      <el-table-column
        sortable
        :label="$t(&quot;table.systemData&quot;)"
        prop="readonly"
        align="center"
        width="120px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.readonly ? $t("common.yes") : $t("common.no") }}</span>
        </template>
      </el-table-column>
      <el-table-column
        sortable
        :label="$t(&quot;table.createTime&quot;)"
        prop="createTime"
        align="center"
        width="170px"
      >
        <template slot-scope="scope">
          <span>{{ scope.row.createTime }}</span>
        </template>
      </el-table-column>
      <el-table-column
        :label="$t(&quot;table.operation&quot;)"
        align="center"
        width="150px"
        class-name="small-padding fixed-width"
      >
        <template slot-scope="{row}">
          <i class="el-icon-edit table-operation" style="color: #2db7f5;" @click="edit(row)" />
          <i
            class="el-icon-delete table-operation"
            style="color: #f50;"
            @click="singleDelete(row)"
          />
        </template>
      </el-table-column>
    </el-table>
    <pagination
      v-show="tableData.total>0"
      :total="Number(tableData.total)"
      :page.sync="pagination.current"
      :limit.sync="pagination.size"
      @pagination="fetch"
    />
    <global-user-edit
      ref="edit"
      :dialog-visible="dialog.isVisible"
      :title="dialog.title"
      @success="editSuccess"
      @close="editClose"
    />
  </div>
</template>

<script>
import Pagination from '@/components/Pagination'
import GlobalUserEdit from './Edit'
import globalUserApi from '@/api/GlobalUser.js'

export default {
  name: 'GlobalUserManage',
  components: { Pagination, GlobalUserEdit },
  data () {
    return {
      dialog: {
        isVisible: false,
        title: ''
      },
      tableKey: 0,
      total: 0,
      queryParams: {},
      sort: {},
      selection: [],
      // 以下已修改
      loading: false,
      tableData: {
        total: 0
      },
      pagination: {
        size: 10,
        current: 1
      }
    }
  },
  computed: {
    currentUser () {
      return this.$store.state.account.user
    }
  },
  mounted () {
    this.fetch()
  },
  methods: {
    editClose () {
      this.dialog.isVisible = false
    },
    editSuccess () {
      this.search()
    },
    onSelectChange (selection) {
      this.selection = selection
    },
    search () {
      this.fetch({
        ...this.queryParams,
        ...this.sort
      })
    },
    reset () {
      this.queryParams = {}
      this.sort = {}
      this.$refs.table.clearSort()
      this.$refs.table.clearFilter()
      this.search()
    },
    add () {
      this.$refs.edit.type = 'add'
      this.dialog.title = this.$t('common.add')
      this.dialog.isVisible = true
    },
    singleDelete (row) {
      this.$refs.table.toggleRowSelection(row, true)
      this.batchDelete()
    },
    batchDelete () {
      if (!this.selection.length) {
        this.$message({
          message: this.$t('tips.noDataSelected'),
          type: 'warning'
        })
        return
      }
      this.$confirm(this.$t('tips.confirmDelete'), this.$t('common.tips'), {
        confirmButtonText: this.$t('common.confirm'),
        cancelButtonText: this.$t('common.cancel'),
        type: 'warning'
      }).then(() => {
        let contain = false
        const userIds = []
        let isSystemData = false
        const tenantCodeList = new Set()
        this.selection.forEach((item) => {
          if (item.id === this.currentUser.id) {
            contain = true
            return
          }
          if (item.readonly) {
            isSystemData = true
            return
          }
          tenantCodeList.add(item.tenantCode)
          userIds.push(item.id)
        })
        if (isSystemData) {
          this.$message({
            message: this.$t('tips.systemData'),
            type: 'warning'
          })
        } else if (contain) {
          this.$message({
            message: this.$t('tips.containCurrentUser'),
            type: 'warning'
          })
          this.clearSelections()
        } else if (tenantCodeList.size > 1) {
          this.$message({
            message: '只能批量删除同一企业下的用户',
            type: 'warning'
          })
          this.clearSelections()
        } else {
          this.delete(Array.from(tenantCodeList)[0], userIds)
        }
      }).catch(() => {
        this.clearSelections()
      })
    },
    clearSelections () {
      this.$refs.table.clearSelection()
    },
    delete (tenantCode, ids) {
      globalUserApi.remove({ tenantCode: tenantCode, ids: ids })
        .then((res) => {
          if (res.isSuccess) {
            this.$message({
              message: this.$t('tips.deleteSuccess'),
              type: 'success'
            })
            this.search()
          }
        })
    },

    edit (row) {
      if (row.readonly) {
        this.$message({
          message: this.$t('tips.systemData'),
          type: 'warning'
        })
        return
      }
      this.$refs.edit.setGlobalUser(row)
      this.$refs.edit.type = 'edit'
      this.dialog.title = this.$t('common.edit')
      this.dialog.isVisible = true
    },
    fetch (params = {}) {
      this.loading = true
      params.size = this.pagination.size
      params.current = this.pagination.current
      if (this.queryParams.timeRange) {
        params.startCreateTime = this.queryParams.timeRange[0]
        params.endCreateTime = this.queryParams.timeRange[1]
      }
      globalUserApi.findPage(params)
        .then((r) => {
          if (r.isError) {
            return
          }
          this.tableData = r.data
          this.loading = false
        })
    },
    sortChange (val) {
      this.sort.field = val.prop
      this.sort.order = val.order
      this.search()
    }
  }
}
</script>
<style lang="scss" scoped>
</style>
