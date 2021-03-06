<template>
  <div class="page-products">
    <el-breadcrumb separator="/" class="breadcrumb">
      <el-breadcrumb-item>{{ rtype }}管理</el-breadcrumb-item>
    </el-breadcrumb>

    <el-card ref="cardItemList">
      <div class="card-title" slot="header">
        <i class="el-icon-arrow-down"></i>
        <span>{{ rtype }}列表</span>
      </div>

      <div>
        <!--<el-input class="search-input"
          placeholder="关键词搜索"
          icon="search"
          v-model="searchInput">
        </el-input>-->
        <form-gen class="search-form" ref="searchForm"
          :schema="searchSchema"
          @submit="handleSearch"></form-gen>

        <el-table :data="itemList">
          <el-table-column type="expand">
            <template scope="s">
              <el-form label-position="left" inline class="demo-table-expand">
                <el-form-item :label="rtype + 'UID'">
                  <span>{{ s.row.uid }}</span>
                </el-form-item>
                <el-form-item :label="rtype + '名称'">
                  <span>{{ s.row.name }}</span>
                </el-form-item>
                <el-form-item label="供应商">
                  <span>{{ getRefValue(s.row, 'provider', 'providers', 'name') }}</span>
                </el-form-item>
                <el-form-item label="时段">
                  <span>{{ getRefValue(s.row, 'category', 'categories', 'name') }}</span>
                </el-form-item>
                <el-form-item label="价格">
                  <span>{{ s.row.price }} 元</span>
                </el-form-item>
                <el-form-item label="图片">
                  <span><img v-if="s.row.image" :src="s.row.image"></span>
                </el-form-item>
                <el-form-item :label="rtype + '描述'">
                  <span>{{ s.row.description }}</span>
                </el-form-item>
                <el-form-item label="创建日期">
                  <span>{{ new Date(s.row.createdAt).toLocaleString() }}</span>
                </el-form-item>
                <el-form-item label="操作">
                  <el-button type="default" size="mini" icon="edit"
                      @click="editItem(s.row)"></el-button>
                  <el-button type="danger" size="mini" icon="delete"
                      @click="deleteItem(s.row)"></el-button>
                </el-form-item>
              </el-form>
            </template>
          </el-table-column>
          <el-table-column
            :label="rtype + 'UID'"
            prop="uid">
          </el-table-column>
          <el-table-column
            :label="rtype + '名称'"
            prop="name">
          </el-table-column>
          <el-table-column
              label="时段">
            <template scope="s">
              {{ getRefValue(s.row, 'category', 'categories', 'name') }}
            </template>
          </el-table-column>
          <el-table-column
              label="价格">
            <template scope="s">
              {{ s.row.price }} 元
            </template>
          </el-table-column>
        </el-table>

        <div class="table-footer">
          <el-button type="primary" size="small" icon="plus"
              @click="gotoItemCreate()">添加{{ rtype }}</el-button>
          <el-pagination
            class="pagination"
            @size-change="handlePageSize"
            @current-change="handlePageCurrent"
            :current-page="pageCurrent"
            :page-size="pageSize"
            :page-sizes="pageSizes"
            layout="total, sizes, prev, pager, next, jumper"
            :total="listTotal">
          </el-pagination>
        </div>
      </div>
    </el-card>

    <item-card v-show="itemInEdit"
      ref="cardItemEdit"
      action="edit" :name="rtype"
      :schema="editSchema"
      @cancel="itemEditCancel"
      @submit="itemEditSubmit">
    </item-card>

    <item-card
      ref="cardItemCreate"
      action="create" :name="rtype"
      :schema="editSchema"
      @submit="itemCreateSubmit">
    </item-card>
  </div>
</template>

<script>
import { categories, providers } from '@/const'
import { fetchApi } from '@/api'
import ItemCard from '@/components/ItemCard'
import FormGen from '@/components/FormGen'
import _ from 'lodash'
import qs from 'qs'

let rtype = '商品'

export default {
  components: { ItemCard, FormGen },

  data () {
    return {
      rtype,
      listLoading: false,
      listTotal: 0,
      itemList: [],
      pageCurrent: 1,
      pageSizes: [10, 20, 50],
      pageSize: 10,
      searchModel: {},

      refMap: this.toRefMap({
        categories,
        providers
      }),
      itemInEdit: false,

      searchSchema: {
        inline: true,
        size: 'small',
        fields: [
          {
            input: 'select',
            key: 'category',
            cls: 'category-select',
            placeholder: '时段',
            allowEmpty: true,
            options: categories.map(({ uid, name }) => {
              return { value: uid, label: name }
            })
          },
          { input: 'number', key: 'price', placeholder: '价格' },
          { input: 'text', key: 'keyword', placeholder: '关键词' }
        ],
        buttons: [
          { type: 'primary', text: '查询', emit: 'submit' }
        ]
      },

      // todo: request /produts, /categories, /providers seperately
      editSchema: {
        fields: [
          { input: 'text', key: 'name', label: `${rtype}名称` },
          {
            input: 'radio-group',
            key: 'provider',
            label: '供应商',
            options: providers.map(({ uid, name }) => {
              return { value: uid, label: name }
            })
          },
          {
            input: 'radio-group',
            key: 'category',
            label: '时段',
            options: categories.map(({ uid, name }) => {
              return { value: uid, label: name }
            })
          },
          { input: 'number', key: 'price', label: '价格' },
          { input: 'image-upload', key: 'image', label: '图片' },
          { input: 'textarea', key: 'description', label: `${rtype}描述` }
        ]
      }
    }
  },

  created () {
    this.loadList()
  },

  methods: {
    getRefValue (srcObj, fromKey, tarRes, toKey) {
      let uid = srcObj[fromKey]
      let refObj = this.refMap[tarRes][uid]
      return refObj ? refObj[toKey] : null
    },
    toRefMap (refs) {
      return _.reduce(refs, (acc, docs, k) => {
        acc[k] = docs.reduce((ac, doc) => {
          ac[doc.uid] = doc
          return ac
        }, {})
        return acc
      }, {})
    },

    loadList () {
      this.listLoading = true
      let { pageSize, pageCurrent, searchModel } = this
      let searchObj = _.reduce(searchModel, (acc, v, k) => {
        if (v != null && v !== '') {
          acc[k] = v
        }
        return acc
      }, {})
      let url = '/a/products/list?' + qs.stringify({
        limit: pageSize,
        page: pageCurrent,
        search: searchObj
      }, {
        encodeValuesOnly: true
      })
      fetchApi(url)
        .then(data => {
          let { total, docs } = data
          this.itemList = docs
          this.listTotal = total
          this.listLoading = false
        })
    },

    deleteItem ({ uid }) {
      this.$confirm('是否确认删除该条记录？', '删除记录', {
        confirmButtonText: '立即删除',
        cancelButtonText: '取消',
        type: 'warning'
      })
      .then(() => {
        return fetchApi('/a/products/delete', {
          method: 'POST',
          body: JSON.stringify({ uid })
        })
      })
      .then(data => {
        this.loadList()
      })
    },

    editItem (row) {
      if (this.itemInEdit) { // has item being edited
        this.$alert('有一条记录正在修改中，请先关闭当前修改。', '正在修改记录')
        return
      }
      this.$refs.cardItemEdit.reset(row)
      this.itemInEdit = true
      this.$nextTick(() => { // wait to be shown
        this.gotoItemEdit()
      })
    },

    itemEditCancel () {
      this.$confirm('是否确认放弃修改该条记录？', '放弃修改', {
        confirmButtonText: '放弃修改',
        cancelButtonText: '继续修改',
        type: 'warning'
      })
      .then(() => {
        this.$refs.cardItemEdit.reset()
        this.itemInEdit = false
      })
    },
    itemEditSubmit (model) {
      fetchApi('/a/products/update', {
        method: 'POST',
        body: JSON.stringify(model)
      })
      .then(() => {
        // this.$refs.cardItemEdit.reset()
        this.loadList()
        // this.gotoItemList()
        // this.itemInEdit = false
        this.$notify({
          type: 'success',
          title: '修改成功',
          message: `${rtype}信息已成功修改`
        })
      })
    },
    itemCreateSubmit (model) {
      fetchApi('/a/products/create', {
        method: 'POST',
        body: JSON.stringify(model)
      })
      .then(() => {
        this.$refs.cardItemCreate.reset()
        this.loadList()
        this.gotoItemList()
        this.$notify({
          type: 'success',
          title: '添加成功',
          message: `${rtype}信息已成功添加`
        })
      })
      .catch(err => {
        this.$notify({
          type: 'error',
          title: '添加失败',
          message: `${err}`
        })
      })
    },

    gotoItemList () {
      this.$refs.cardItemList.$el.scrollIntoView()
    },
    gotoItemCreate () {
      this.$refs.cardItemCreate.$el.scrollIntoView()
      this.$refs.cardItemCreate.$el.querySelector('input').focus()
    },
    gotoItemEdit () {
      this.$refs.cardItemEdit.$el.scrollIntoView()
    },

    handleSearch () {
      this.searchModel = this.$refs.searchForm.getModel()
      this.loadList()
    },
    handlePageSize (v) {
      this.pageSize = v
      this.loadList()
    },
    handlePageCurrent (v) {
      this.pageCurrent = v
      this.loadList()
    }
  }
}
</script>

<style lang="scss" src="./ItemManage.scss"></style>

<style lang="scss">
.page-products {
  .search-form {
    .el-input {
      width: 140px;
    }
    .category-select .el-input {
      width: 100px;
    }
  }
}
</style>

