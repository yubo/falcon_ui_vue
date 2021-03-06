<template>
  <div id="content" class="main">
    <div class="form-inline" role="form">
      <div class="form-group">
        <input type="text" v-model="query" @keyup.enter="handleQuery" class="form-control" placeholder="team name">
      </div>
      <button type="button" @click="handleQuery" class="btn btn-primary">search</button>
      <div class="pull-right">
        <button :disabled="!isOperator" type="button" @click="editObj" class="btn btn-default"><span class="glyphicon glyphicon-plus"></span>Add</button>
      </div>
    </div>

    <el-table v-loading="loading" :data="tableData" border style="width: 100%" class="mt20">
      <el-table-column prop="name"   label="name"> </el-table-column>
      <el-table-column prop="note"   label="note"> </el-table-column>
      <el-table-column prop="ctime"  label="created"> </el-table-column>
      <el-table-column label="command">
        <template scope="scope">
          <el-button :disabled="!isOperator" @click="editObj(scope.row)" type="text" size="small">EDIT</el-button>
          <el-button :disabled="!isOperator" @click="deleteObj(scope.row)" type="text" size="small">DELETE</el-button>
        </template>
      </el-table-column>
    </el-table>

    <div class="pull-right mt20">
      <el-pagination
        @size-change="sizeChange"
        @current-change="curChange"
        :current-page="cur"
        :page-sizes="pageSizes"
        :page-size="per"
        layout="total, sizes, prev, pager, next, jumper"
        :total="total">
      </el-pagination>
    </div>
    <el-dialog :title="isEdit ? 'edit team' : 'add team'" v-model="editVisible" :close-on-click-modal="false">
      <div v-loading.lock="dloading">
        <el-form label-position="right" label-width="80px" :model="objForm">

          <el-form-item label="name">   <el-input v-model="objForm.name"></el-input> </el-form-item>
          <el-form-item label="note">   <el-input v-model="objForm.note"></el-input> </el-form-item>
          <el-form-item label="member">
            <el-select
              style="width: 100%"
              placeholder="user name"
              v-model="users"
              multiple
              filterable
              remote
              :remote-method="getUsers"
              :loading="sloading">
              <el-option
                v-for="user in optionUsers"
                :key="user.id"
                :label="user.name"
                :value="user.id">
              </el-option>
            </el-select>
          </el-form-item>
          <el-form-item>
            <el-button type="primary" @click="submitObj">submit</el-button>
            <el-button @click="editVisible = false">Cancel</el-button>
          </el-form-item>
        </el-form>
      </div>
    </el-dialog>
  </div>
</template>

<script>
import { fetch, Msg } from 'src/utils'

export default {
  data () {
    return {
      loading: false,
      sloading: false,
      dloading: false,
      query: '',
      per: 10,
      cur: 1,
      total: 0,
      pageSizes: [5, 10, 20, 50],
      tableData: [],
      curId: 0,
      editVisible: false,
      users: [],
      optionUsers: [],
      objForm: {
        name: '',
        note: ''
      }

    }
  },
  methods: {
    sizeChange (per) {
      this.per = per
      this.fetchObjs()
    },
    curChange (cur) {
      this.cur = cur
      this.fetchObjs()
    },
    handleQuery () {
      this.reFetchObjs()
    },

    reFetchObjs () {
      fetch({
        method: 'get',
        url: 'team/cnt',
        params: {query: this.query}
      }).then((res) => {
        this.total = res.data.total
        this.fetchObjs()
      }).catch((err) => {
        Msg.error('get failed', err)
      })
    },

    fetchObjs (opts = {query: this.query, per: this.per, offset: this.offset}) {
      this.loading = true
      fetch({
        method: 'get',
        url: 'team/search',
        params: opts
      }).then((res) => {
        this.tableData = res.data
        this.loading = false
      }).catch((err) => {
        Msg.error('get failed', err)
        this.loading = false
      })
    },

    getUsers (query) {
      if (query !== '') {
        this.sloading = true
        fetch({
          method: 'get',
          url: 'user/search',
          params: {
            query: query,
            per: 10
          }
        }).then((res) => {
          this.optionUsers = res.data
          this.sloading = false
        }).catch((err) => {
          Msg.error('get failed', err)
          this.sloading = false
        })
      } else {
        this.optionUsers = []
      }
    },
    fetchMember () {
      this.dloading = true
      fetch({
        method: 'get',
        url: 'team/' + this.curId + '/member'
      }).then((res) => {
        this.optionUsers = res.data.users
        this.users = this.optionUsers.map((i) => {
          return i.id
        })
        this.dloading = false
      }).catch((err) => {
        Msg.error('get failed', err)
        this.dloading = false
      })
    },
    editObj (obj = {}) {
      this.curId = obj.id
      for (var k in this.objForm) {
        this.objForm[k] = obj[k]
      }
      this.editVisible = true
      if (this.isEdit) {
        this.fetchMember()
      }
    },

    submitMember () {
      fetch({
        method: 'put',
        url: 'team/' + this.curId + '/member',
        data: {uids: this.users}
      }).then((res) => {
        Msg.success('update success')
        this.fetchObjs()
        this.dloading = false
        this.editVisible = false
      }).catch((err) => {
        Msg.error('update failed', err)
        this.dloading = false
      })
    },
    submitObj () {
      this.dloading = true
      fetch({
        method: this.isEdit ? 'put' : 'post',
        url: this.isEdit ? 'team/' + this.curId : 'team',
        data: this.objForm
      }).then((res) => {
        Msg.success('update success')
        if (!this.isEdit) {
          this.total++
        }
        this.submitMember()
      }).catch((err) => {
        Msg.error('edit failed', err)
        this.dloading = false
      })
    },
    deleteObj (obj) {
      Msg.confirm('此操作将永久删除该记录, 是否继续?', '提示', {
        confirmButtonText: 'Confirm',
        cancelButtonText: 'Cancel',
        type: 'warning'
      }).then(() => {
        fetch({
          method: 'delete',
          url: 'team/' + obj.id
        }).then((res) => {
          Msg.success('success!')
          this.total--
          this.fetchObjs()
        }).catch((err) => {
          Msg.error('delete failed', err)
        })
      }).catch(() => {
        Msg.info('cancel')
      })
    }
  },

  computed: {
    isOperator () {
      return this.$store.state.auth.operator
    },
    offset () {
      return (this.per * (this.cur - 1))
    },
    isEdit () {
      return this.curId
    }
  },
  created () {
    if (this.$route.query.query) {
      this.query = this.$route.query.query
    }
    if (this.$route.query.per) {
      this.per = this.$route.query.per
    }
    if (this.$route.query.cur) {
      this.cur = this.$route.query.cur
    }
    this.reFetchObjs()
  }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style scoped>
</style>
