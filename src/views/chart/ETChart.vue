<template>
    <div class="kpi-container">
        <el-card shadow="hover">
            <div slot="header" class="clearfix">
                <h1 style="margin: 0; line-height: 36px;">
                    <i class="el-icon-data-line"></i> ET数据展示
                </h1>
                <div style="float: right; display: flex; align-items: center;">
                    <el-button 
                        type="primary" 
                        icon="el-icon-setting" 
                        @click="showColumnDialog = true"
                        style="margin-right: 10px;">
                        字段配置
                    </el-button>
                    <el-button 
                        type="text" 
                        icon="el-icon-refresh" 
                        @click="fetchData">
                        刷新数据
                    </el-button>
                </div>
            </div>

            <!-- 列选择对话框 -->
            <el-dialog title="选择显示列" :visible.sync="showColumnDialog" width="50%">
                <el-transfer
                    v-model="selectedColumns"
                    :data="allColumns"
                    :titles="['可选列', '已选列']"
                    :props="{key: 'key', label: 'label'}"
                    filterable>
                </el-transfer>
                <span slot="footer" class="dialog-footer">
                    <el-button @click="showColumnDialog = false">取消</el-button>
                    <el-button type="primary" @click="saveColumnSelection">确定</el-button>
                </span>
            </el-dialog>

            <!-- 加载状态 -->
            <el-skeleton v-if="loading" :rows="10" animated />

            <!-- 错误提示 -->
            <el-alert 
                v-if="error"
                :title="error"
                type="error"
                show-icon
                :closable="false">
            </el-alert>

            <!-- 数据表格 -->
            <div v-if="tableData.length > 0" class="table-container">
                <el-table 
                    :data="currentTableData" 
                    border 
                    style="width: 100%" 
                    height="600"
                    stripe
                    highlight-current-row
                    v-loading="loading"
                    element-loading-text="数据加载中"
                    element-loading-spinner="el-icon-loading">
                    
                    <!-- 固定的人员列 -->
                    <el-table-column 
                        prop="userName" 
                        label="人员" 
                        width="120"
                        fixed="left">
                        <template #default="scope">
                            <span>{{ scope.row.userName }}</span>
                        </template>
                    </el-table-column>
                    
                    <!-- 动态列 -->
                    <el-table-column 
                        v-for="key in visibleColumns" 
                        :key="key" 
                        :prop="key"
                        :label="fieldMap[key] || key" 
                        min-width="180"
                        sortable>
                        <template #default="scope">
                            <el-tooltip 
                                v-if="Array.isArray(scope.row[key]) || typeof scope.row[key] === 'object'"
                                effect="light" 
                                placement="top">
                                <template #content>
                                    <pre style="margin: 0">{{ 
                                        Array.isArray(scope.row[key]) 
                                            ? scope.row[key].join(', ') 
                                            : JSON.stringify(scope.row[key], null, 2)
                                    }}</pre>
                                </template>
                                <el-tag type="info" size="small">
                                    {{ 
                                        Array.isArray(scope.row[key]) 
                                            ? scope.row[key].length + '项' 
                                            : '对象数据'
                                    }}
                                </el-tag>
                            </el-tooltip>
                            <span v-else>
                                {{ scope.row[key] }}
                            </span>
                        </template>
                    </el-table-column>
                </el-table>
                
                <div class="pagination-container">
                    <el-pagination
                        @size-change="handleSizeChange"
                        @current-change="handleCurrentChange"
                        :current-page="currentPage"
                        :page-sizes="[10, 20, 50, 100]"
                        :page-size="pageSize"
                        layout="total, sizes, prev, pager, next, jumper"
                        :total="tableData.length">
                    </el-pagination>
                </div>
            </div>
            
            <div v-else-if="!loading && !error" class="empty-container">
                <el-empty description="暂无数据"></el-empty>
            </div>
        </el-card>
    </div>
</template>

<script>
import request from '@/utils/request'

const AUTH_REQUEST = {
    username: "dongenzhe",
    password: "aa228228!",
    appId: "appId"
};

const KPI_REQUEST = {
    startDate: "2025-03-21",
    endDate: "2025-06-20",
    personnelDimension: 1,
    filterLeavePerson: false,
    searchDeptFullName: null,
    excludeUserIds: []
}

export default {
    name: 'KpiDataView',
    data() {
        return {
            loading: false,
            error: null,
            tableData: [],
            accessToken: "",
            fieldMap: {
                "weightedExecuteDays": "加权执行天数",
                "highAnalysisLeakBugNumber": "高分析漏测缺陷数",
                "rdLowLeakBugNumber": "研发低漏测缺陷数",
                "validBugNumPerWeightedExecuteDays": "每加权执行天数有效缺陷数",
                "validBugNumberPerWorkingDays": "每工作日有效缺陷数",
                "highMiddleAnalysisBugNumber": "高中分析缺陷数",
                "loginName": "登录名",
                "bugSuggestNumber": "建议缺陷数",
                "middleAnalysisBugNumber": "中分析缺陷数",
                "id": "ID",
                "weightedValidBugNumberPerWeightedExecuteDays": "每加权执行天数加权有效缺陷数",
                "workingHours": "工作小时数",
                "weightedVaildBugsPerweightedTestDaysRemoveAuto": "去除自动化后每加权测试天数加权有效缺陷数",
                "defectValidity": "缺陷有效性",
                "notABugNumber": "非缺陷数",
                "unitDefectCost": "单位缺陷成本",
                "params": "参数",
                "executionCaseRate": "执行用例率",
                "executionCaseNumPerExecuteDays": "每执行天数执行用例数",
                "testnos": "测试编号",
                "validHighSafeBugNumber": "有效高安全缺陷数",
                "reportOrLegalExecutedTestCaseNumbers": "报告或合法执行测试用例数",
                "assignCaseNumber": "分配用例数",
                "rdHighLeakBugNumber": "研发高漏测缺陷数",
                "hoursPerDays": "每天小时数",
                "autoExecutionManualRate": "自动化执行手动率",
                "userStatus": "用户状态",
                "validLowBugNumber": "有效低缺陷数",
                "weightedValidBugNumber": "加权有效缺陷数",
                "highMiddleAnalysisBugRate": "高中分析缺陷率",
                "testCycleSize": "测试周期大小",
                "executionCaseNumberRemoveAuto": "去除自动化后执行用例数",
                "stabilityTestPersonDays": "稳定性测试人天数",
                "executionCaseNumberRemoveAutoPerweightedTestDays": "去除自动化后每加权测试天数执行用例数",
                "manualTestCaseNumber": "手动测试用例数",
                "middleAnalysisLeakBugNumber": "中分析漏测缺陷数",
                "addCaseNumber": "新增用例数",
                "questionRate": "问题率",
                "bugLowNumber": "低缺陷数",
                "rdHighMiddleLeakBugNumber": "研发高中漏测缺陷数",
                "autoExecutionComplianceRate": "自动化执行合规率",
                "executionCaseNumber": "执行用例数",
                "manualTestCaseWithoutAutoRunNumber": "无自动运行的手动测试用例数",
                "validMiddleBugNumber": "有效中缺陷数",
                "executionCaseNumberRemoveAutoPerWorkingDays": "去除自动化后每工作日执行用例数",
                "userId": "用户ID",
                "pautoTestCaseRate": "部分自动化测试用例率",
                "autoExecutionCaseNumber": "自动化执行用例数",
                "stabilityExecutionCaseNumber": "稳定性执行用例数",
                "hasChild": "有子项",
                "validBugNumber": "有效缺陷数",
                "legalExecutedAutoCaseNumbers": "合法执行自动化用例数",
                "validBugSuggestNumber": "有效建议缺陷数",
                "lowAnalysisBugNumber": "低分析缺陷数",
                "workingDays": "工作日",
                "rdMiddleLeakBugNumber": "研发中漏测缺陷数",
                "fautoTestCaseNumber": "全自动化测试用例数",
                "fullAutoCaseNumber": "全自动化用例数",
                "highAnalysisBugNumber": "高分析缺陷数",
                "modifyCaseNumber": "修改用例数",
                "autoExeCaseNumber": "自动化执行用例数",
                "highAnalysisBugRate": "高分析缺陷率",
                "autoExecutionRate": "自动化执行率",
                "manualTestCaseExcludeBlockNumber": "排除阻塞的手动测试用例数",
                "weightedTestDaysRemoveAutoPerWorkingDays": "去除自动化后加权测试天数占工作日比例",
                "environmentBuildPersonDays": "环境构建人天数",
                "bugMiddleNumber": "中缺陷数",
                "weightedTestDaysRemoveAuto": "去除自动化后加权测试天数",
                "allWorkDays": "所有工作日",
                "validBugNumPerExecuteDays": "每执行天数有效缺陷数",
                "weightedExecuteDaysPerWorkingDays": "加权执行天数占工作日比例",
                "validSafeBugNumber": "有效安全缺陷数",
                "reviewBugNumber": "评审缺陷数",
                "totalBugNumber": "总缺陷数",
                "rdMarkLeakHighBugErrorRate": "研发标记高漏测缺陷错误率",
                "validBugNumberPerweightedTestDaysRemoveAuto": "去除自动化后每加权测试天数有效缺陷数",
                "reportTestCaseNumbers": "报告测试用例数",
                "validMiddleSafeBugNumber": "有效中安全缺陷数",
                "rdMarkLeakHighMiddleBugErrorRate": "研发标记高中漏测缺陷错误率",
                "projectTestPersonDays": "项目测试人天数",
                "validLowSafeBugNumber": "有效低安全缺陷数",
                "fautoTestCaseRate": "全自动化测试用例率",
                "pautoTestCaseNumber": "部分自动化测试用例数",
                "onlyStabilityExecutionCaseNumber": "仅稳定性执行用例数",
                "autoExecutionUnComplianceRate": "自动化执行不合规率",
                "executionCaseNumPerWorkingHours": "每工作小时执行用例数",
                "lowAnalysisLeakBugNumber": "低分析漏测缺陷数",
                "executionCaseNumberPerWorkingDays": "每工作日执行用例数",
                "weightedBugsPerWorkingDays": "每工作日加权缺陷数",
                "caseValidityRemoveAuto": "去除自动化后用例有效性",
                "executionCaseNumPerWeightedExecuteDays": "每加权执行天数执行用例数",
                "exeAutoCaseNumberRate": "执行自动化用例率",
                "workTimeHours": "工作时间小时数",
                "executedAutoCaseNumbers": "执行自动化用例数",
                "testDesignPersonDays": "测试设计人天数",
                "bugHighNumber": "高缺陷数",
                "autoExeAutoTestCaseNumber": "自动化执行自动化测试用例数",
                "userName": "人员",
                "autoQuestionNumber": "自动化问题数",
                "highMiddleAnalysisLeakBugNumber": "高中分析漏测缺陷数",
                "caseValidity": "用例有效性",
                "executeDays": "执行天数",
                "validHighBugNumber": "有效高缺陷数",
                "validBugNumPerWorkingHours": "每工作小时有效缺陷数"
            },
            currentPage: 1,
            pageSize: 20,
            showColumnDialog: false,
            allColumns: [],
            selectedColumns: [],
            visibleColumns: []
        };
    },
    computed: {
        currentTableData() {
            const start = (this.currentPage - 1) * this.pageSize;
            const end = start + this.pageSize;
            return this.tableData.slice(start, end);
        }
    },
    async created() {
        await this.fetchData();
    },
    methods: {
        async fetchData() {
            this.loading = true;
            this.error = null;

            try {
                // 1. 获取Token
                const authResponse = await request({
                    url: '/api/et/accessToken',
                    method: 'post',
                    data: AUTH_REQUEST
                });

                this.accessToken = authResponse.accessToken;

                // 2. 使用Token获取KPI数据
                const kpiResponse = await request({
                    url: '/api/et/kpi',
                    method: 'post',
                    data: {
                        kpiRequest: KPI_REQUEST,
                        accessToken: this.accessToken
                    }
                });
                
                if (kpiResponse.success) {
                    this.tableData = kpiResponse.data;
                    // 初始化列选择器
                    this.initColumnSelector();
                } else {
                    this.error = kpiResponse.message || '获取数据失败';
                }
            } catch (err) {
                console.error('获取数据出错:', err);
                this.error = '获取数据出错，请稍后再试';
            } finally {
                this.loading = false;
            }
        },
        initColumnSelector() {
            if (this.tableData.length > 0) {
                // 获取所有列名，排除userName（因为已经固定显示）
                const allKeys = Object.keys(this.tableData[0]).filter(key => key !== 'userName');
                this.allColumns = allKeys.map(key => ({
                    key: key,
                    label: this.fieldMap[key] || key
                }));
                
                // 默认显示前10列（不包括userName）
                this.selectedColumns = allKeys.slice(10, 20);
                this.visibleColumns = [...this.selectedColumns];
            }
        },
        saveColumnSelection() {
            this.visibleColumns = [...this.selectedColumns];
            this.showColumnDialog = false;
        },
        handleSizeChange(val) {
            this.pageSize = val;
        },
        handleCurrentChange(val) {
            this.currentPage = val;
        }
    }
};
</script>

<style scoped>
.kpi-container {
    padding: 20px;
    background-color: #f5f7fa;
    min-height: calc(100vh - 40px);
}

.table-container {
    margin-top: 20px;
}

.pagination-container {
    margin-top: 20px;
    text-align: right;
}

.empty-container {
    height: 400px;
    display: flex;
    justify-content: center;
    align-items: center;
}

.el-card {
    border-radius: 4px;
}

.el-table {
    font-size: 14px;
}

.el-table::before {
    height: 0;
}
</style>
