<template>
    <page-header-wrapper content="对系统中经常使用的一些较为固定的数据进行维护">
        <a-card :bordered="false">
            <div class="table-page-search-wrapper">
                <a-form layout="inline">
                    <a-row :gutter="48">
                        <a-col :md="8" :sm="24">
                            <a-form-item label="字典名称">
                                <a-input placeholder="请输入字典名称" v-model="queryForm.name" allowClear />
                            </a-form-item>
                        </a-col>
                        <a-col :md="8" :sm="24">
                            <a-form-item label="字典类别">
                                <a-input placeholder="请输入字典类别" v-model="queryForm.category" allowClear />
                            </a-form-item>
                        </a-col>
                        <a-col :md="8" :sm="24">
                            <a-space>
                                <a-button type="primary" @click="query" v-action:query> 查询 </a-button>
                                <a-button type="primary" @click="addDictionary" v-action:add> 添加 </a-button>
                            </a-space>
                        </a-col>
                    </a-row>
                </a-form>
            </div>
            <!-- 数据表格 -->
            <a-table
                :columns="columns"
                :data-source="data"
                rowKey="DictionaryID"
                :pagination="pagination"
                @change="tableChange"
                :loading="loading"
            >
                <span slot="category" slot-scope="text, record">
                    <router-link :to="'/subDictionary/category/' + record.DictionaryID">
                        <span>{{ record.category }}</span>
                    </router-link>
                </span>
                <span slot="status" slot-scope="text, record">
                    <a-tag :color="record.status ? 'cyan' : 'purple'">{{ record.status ? '禁用' : '正常' }}</a-tag>
                </span>
                <span slot="action" slot-scope="text, record">
                    <a @click="onEdit(record)" v-action:edit>编辑</a>
                    <a-divider type="vertical" />
                    <a @click="onDelete(record)" v-action:delete>删除</a>
                </span>
            </a-table>
            <!-- 抽屉-新建编辑 -->
            <a-modal
                :title="drawerTitle"
                :visible="visible"
                @cancel="onClose"
                :width="640"
                @ok="handleSubmit"
                :confirm-loading="confirmLoading"
            >
                <a-row :gutter="20">
                    <a-form :form="form">
                        <a-col :span="24">
                            <a-form-item label="字典名称">
                                <a-input v-decorator="rules.name" placeholder="请输入字典名称" allowClear />
                            </a-form-item>
                        </a-col>
                        <a-col :span="24">
                            <a-form-item label="字典类别">
                                <a-input v-decorator="rules.category" placeholder="请输入字典类别" allowClear />
                            </a-form-item>
                        </a-col>
                        <a-col :span="24">
                            <a-form-item label="状态">
                                <a-radio-group v-decorator="rules.status">
                                    <a-radio-button :value="0"> 开启 </a-radio-button>
                                    <a-radio-button :value="1"> 关闭 </a-radio-button>
                                </a-radio-group>
                            </a-form-item>
                        </a-col>
                        <a-col :span="24">
                            <a-form-item label="备注">
                                <a-textarea rows="4" placeholder="请输入备注" v-decorator="rules.remark" />
                            </a-form-item>
                        </a-col>
                    </a-form>
                </a-row>
            </a-modal>
        </a-card>
    </page-header-wrapper>
</template>

<script>
import { DictionaryList, addDictionary, deleteDictionary } from '@/api/system'
import { dataFormat } from '@/utils/util.js'
import cloneDeep from 'lodash/cloneDeep'
export default {
    name: 'dictionary',
    data() {
        return {
            queryForm: {},
            columns: [
                { title: '字典编号', dataIndex: 'DictionaryID', key: 'DictionaryID' },
                { title: '字典名称', dataIndex: 'name', key: 'name' },
                {
                    title: '字典类别',
                    dataIndex: 'category',
                    key: 'category',
                    scopedSlots: { customRender: 'category' },
                },
                { title: '字典状态', dataIndex: 'status', key: 'status', scopedSlots: { customRender: 'status' } },
                { title: '备注', dataIndex: 'remark', key: 'remark' },
                { title: '创建时间', dataIndex: 'createTime', key: 'createTime' },
                {
                    title: '操作',
                    width: '120px',
                    dataIndex: 'action',
                    key: 'action',
                    align: 'center',
                    fixed: 'right',
                    scopedSlots: { customRender: 'action' },
                },
            ],
            data: [],
            pagination: {
                total: 0,
                defaultCurrent: 1,
                defaultPageSize: 20,
                showTotal: (total, range) => `第${range[0]}-${range[1]}条/总共 ${total} 条数据`,
                showSizeChanger: true,
                pageSizeOptions: ['20', '40', '60', '100'],
            },
            loading: false,
            drawerTitle: '',
            visible: false,
            form: this.$form.createForm(this),
            rules: {
                name: ['name', { rules: [{ required: true, message: '请输入字典名称' }] }],
                category: ['category', { rules: [{ required: true, message: '请输入字典类别' }] }],
                status: ['status', { initialValue: 0 }],
                remark: ['remark'],
            },
            confirmLoading: false,
            DictionaryID: '',
        }
    },
    methods: {
        // 切换分页
        tableChange(e) {
            let _this = this
            _this.pagination.defaultCurrent = e.current
            _this.pagination.defaultPageSize = e.pageSize
            _this.initDictionary()
        },
        // 初始化数据
        async initDictionary() {
            let _this = this
            _this.loading = true
            let params = {
                name: _this.queryForm.name,
                category: _this.queryForm.category,
                current: _this.pagination.defaultCurrent,
                pageSize: _this.pagination.defaultPageSize,
            }
            await DictionaryList(params).then((res) => {
                if (res.state == 1) {
                    _this.data = res.result.list
                    _this.data.forEach((v) => {
                        v.createTime = dataFormat(v.createTime, 'yyyy-MM-dd hh:mm:ss')
                    })
                } else {
                    _this.$message.error(res.msg)
                }
                _this.loading = false
            })
        },
        // 提交
        handleSubmit(e) {
            let _this = this
            e.preventDefault()
            const {
                form: { validateFields },
            } = _this
            _this.confirmLoading = true
            const validateFieldsKey = ['name', 'category', 'status', 'remark']
            validateFields(validateFieldsKey, { force: true }, (err, values) => {
                if (!err) {
                    const params = { ...values }
                    params.DictionaryID = _this.DictionaryID
                    _this.$confirm({
                        title: '确认操作',
                        content: '您确认提交吗?',
                        onOk: async () => {
                            await addDictionary(params)
                                .then((res) => {
                                    if (res.state == 1) {
                                        _this.form.resetFields()
                                        _this.DictionaryID = ''
                                        _this.visible = false
                                        _this.$message.success(res.msg)
                                        _this.initDictionary()
                                    } else {
                                        _this.$message.error(res.msg)
                                    }
                                })
                                .finally(() => {
                                    setTimeout(() => {
                                        _this.confirmLoading = false
                                    }, 600)
                                })
                        },
                        onCancel() {
                            _this.confirmLoading = false
                        },
                    })
                } else {
                    setTimeout(() => {
                        _this.confirmLoading = false
                    }, 600)
                }
            })
        },
        // 新增
        addDictionary() {
            let _this = this
            _this.visible = true
            _this.drawerTitle = '新增字典'
        },
        // 编辑行
        onEdit(record) {
            let _this = this,
                cloneData = cloneDeep(record)
            _this.visible = true
            _this.title = '编辑字典:' + cloneData.name
            _this.DictionaryID = cloneData.DictionaryID
            _this.$nextTick(() => {
                _this.form.setFieldsValue({
                    ['name']: cloneData.name,
                    ['category']: cloneData.category,
                    ['status']: cloneData.status,
                    ['remark']: cloneData.remark,
                })
            })
            if (record.state == '2') {
                _this.sliderDis = false
            } else {
                _this.sliderDis = true
            }
        },
        // 删除行
        onDelete(record) {
            let _this = this
            let params = {
                DictionaryID: record.DictionaryID,
                name: record.name,
            }
            _this.$confirm({
                title: '确认操作',
                content: '您确认删除吗?',
                onOk: async () => {
                    await deleteDictionary(params).then((res) => {
                        if (res.state == 1) {
                            _this.$message.success(res.msg)
                            _this.initDictionary()
                        } else {
                            _this.$message.error(res.msg)
                        }
                    })
                },
            })
        },
        // 筛选
        query() {
            this.pagination.defaultCurrent = 1
            this.initDictionary()
        },
        onClose() {
            let _this = this
            _this.visible = false
            _this.form.resetFields()
        },
    },
    mounted() {
        this.$nextTick(() => {
            this.initDictionary()
        })
    },
}
</script>

<style>
</style>