<template>
    <div class="issue-container">
        <el-card shadow="hover" class="main-card">
            <div slot="header" class="card-header">
                <div class="header-title">
                    <i class="el-icon-data-line"></i>
                    <span>数据统计</span>
                </div>
                
                <div class="query-controls">
                    <el-input
                        v-model="queryName"
                        placeholder="请输入查询名称"
                        clearable
                        style="width: 200px; margin-right: 10px;"
                        @clear="handleQueryClear"
                    />
                    
                    <el-date-picker
                        v-model="dateRange"
                        type="daterange"
                        align="right"
                        unlink-panels
                        value-format="yyyy-MM-dd"
                        format="yyyy/MM/dd"
                        range-separator="至"
                        start-placeholder="开始日期"
                        end-placeholder="结束日期"
                        :picker-options="pickerOptions"
                        style="width: 360px; margin-right: 10px;"
                    />
                    
                    <el-button
                        type="primary"
                        icon="el-icon-search"
                        :loading="loading"
                        @click="fetchStatistic"
                    >
                        查询
                    </el-button>
                </div>
            </div>
            
            <div class="data-display">
                <el-alert
                    v-if="error"
                    :title="error"
                    type="error"
                    show-icon
                    style="margin-bottom: 20px;"
                />
                
                <el-table
                    v-if="tableData.length > 0"
                    :data="tableData"
                    border
                    stripe
                    style="width: 100%"
                    v-loading="loading"
                >
                    <!-- 根据实际返回数据结构配置列 -->
                    <el-table-column
                        prop="name"
                        label="产品线"
                        width="180"
                    />
                    <el-table-column
                        prop="count"
                        label="问题数量"
                        width="120"
                    />
                    <!-- 更多列... -->
                </el-table>
                
                <el-empty
                    v-else
                    description="暂无数据"
                    :image-size="100"
                />
            </div>
        </el-card>
    </div>
</template>

<script>
import request from '@/utils/request'

const CONFIG = {
    username: "dongenzhe",
    password: "aa228228!"
};

export default {
    name: 'JiraIssueDisplay',
    data() {
        return {
            loading: false,
            error: null,
            queryName: '故障',  // 默认查询名称
            dateRange: [
                new Date(new Date().setMonth(new Date().getMonth() - 3)).toISOString().split('T')[0],
                new Date().toISOString().split('T')[0]
            ],  // 默认最近3个月
            tableData: [],
            pickerOptions: {
                disabledDate(time) {
                    return time.getTime() > Date.now();
                },
                shortcuts: [{
                    text: '最近一周',
                    onClick(picker) {
                        const end = new Date();
                        const start = new Date();
                        start.setTime(start.getTime() - 3600 * 1000 * 24 * 7);
                        picker.$emit('pick', [start, end]);
                    }
                }, {
                    text: '最近一个月',
                    onClick(picker) {
                        const end = new Date();
                        const start = new Date();
                        start.setTime(start.getTime() - 3600 * 1000 * 24 * 30);
                        picker.$emit('pick', [start, end]);
                    }
                }, {
                    text: '最近三个月',
                    onClick(picker) {
                        const end = new Date();
                        const start = new Date();
                        start.setTime(start.getTime() - 3600 * 1000 * 24 * 90);
                        picker.$emit('pick', [start, end]);
                    }
                }]
            }
        };
    },
    methods: {
        handleQueryClear() {
            this.queryName = '';
        },
        
        async loginToJira() {
            try {
                const response = await request({
                    url: '/api/jira/login',
                    method: 'post',
                    data: {
                        username: CONFIG.username,
                        password: CONFIG.password
                    }
                });
                return response === 'Login successful';
            } catch (error) {
                console.error('JIRA登录失败:', error);
                this.error = error.message || 'JIRA登录失败';
                return false;
            }
        },
        
        async fetchIssueDataByName() {
            if (!this.queryName) {
                this.$message.warning('请输入查询名称');
                return;
            }
            
            if (!this.dateRange || this.dateRange.length !== 2) {
                this.$message.warning('请选择日期范围');
                return;
            }
            
            this.loading = true;
            this.error = null;
            
            try {
                const isLoggedIn = await this.loginToJira();
                if (!isLoggedIn) return;
                
                const response = await request({
                    url: '/api/jira/issues/name',
                    method: 'get',
                    params: {
                        name: this.queryName,
                        startTime: this.dateRange[0],
                        endTime: this.dateRange[1]
                    }
                });
                console.log(response);
                
                this.tableData = this.transformResponse(response);
                this.$message.success('数据获取成功');
            } catch (error) {
                console.error('获取JIRA数据失败:', error);
                this.error = error.message || '获取JIRA数据失败';
                this.$message.error('数据获取失败');
            } finally {
                this.loading = false;
            }
        },
        
        transformResponse(response) {
            // 根据实际API返回结构转换数据
            if (!response) return [];
            
            // 示例：假设返回的是 { productLine1: count1, productLine2: count2 }
            return Object.entries(response).map(([name, count]) => ({
                name,
                count
            }));
        },

        async fetchStatistic() {
            if (!this.dateRange || this.dateRange.length !== 2) {
                this.$message.warning('请选择日期范围');
                return;
            }
            
            this.loading = true;
            this.error = null;
            
            try {
                const isLoggedIn = await this.loginToJira();
                if (!isLoggedIn) return;
                
                const response = await request({
                    url: '/api/jira/export',
                    method: 'post'
                });
                console.log(response);
                
                // this.tableData = this.transformResponse(response);
                this.$message.success('数据获取成功');
            } catch (error) {
                console.error('获取JIRA数据失败:', error);
                this.error = error.message || '获取JIRA数据失败';
                this.$message.error('数据获取失败');
            } finally {
                this.loading = false;
            }
        },
    }
};
</script>

<style scoped>
.issue-container {
    padding: 20px;
    background-color: #f5f7fa;
    min-height: calc(100vh - 40px);
}

.main-card {
    border-radius: 8px;
    min-height: calc(100vh - 100px);
}

.card-header {
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.header-title {
    display: flex;
    align-items: center;
    font-size: 18px;
    font-weight: 500;
    color: #303133;
}

.header-title i {
    margin-right: 8px;
    font-size: 20px;
    color: #409EFF;
}

.query-controls {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 10px;
}

.data-display {
    margin-top: 20px;
    padding: 10px;
    background-color: #fff;
    border-radius: 4px;
}

@media (max-width: 768px) {
    .query-controls {
        flex-direction: column;
        align-items: flex-start;
    }
    
    .query-controls > * {
        width: 100%;
        margin-right: 0 !important;
        margin-bottom: 10px;
    }
}
</style>
