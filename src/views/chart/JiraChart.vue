<template>
    <div class="jira-data-container">
        <h1>JIRA 数据展示</h1>

        <div v-if="loading" class="loading-section">
            <p>加载中...</p>
            <p>正在获取数据...</p>
        </div>

        <div v-else-if="error" class="error-section">
            <p>数据加载失败: {{ error }}</p>
            <button @click="retry">重试</button>
        </div>

        <div v-else>
            <div class="action-buttons">
                <!-- <button class="back-button" @click="goBack">← 返回全部数据</button> -->
                <button class="export-button" @click="exportExcel">导出Excel</button>
            </div>

            <div class="defect-trend">
                <h2>{{ originalData && originalData.filter && originalData.filter.filterTitle || '缺陷数量趋势' }}</h2>
                <div class="chart-container" ref="trendChart" style="width: 100%; height: 400px;"></div>
            </div>

            <div class="table-view">
                <h2>表格视图
                </h2>
                <table class="data-table">
                    <thead>
                        <tr>
                            <th v-for="header in tableHeaders" :key="header">{{ header }}</th>
                        </tr>
                    </thead>
                    <tbody>
                        <tr v-for="(row, index) in tableData" :key="index">
                            <td v-for="header in tableHeaders" :key="header">
                                <a v-if="row[header].link" :href="row[header].link" target="_blank">
                                    {{ row[header].display }}
                                </a>
                                <span v-else>{{ row[header].display }}</span>
                            </td>
                        </tr>
                    </tbody>
                </table>

            </div>
        </div>
    </div>
</template>

<script>
import * as echarts from 'echarts';
import * as XLSX from 'xlsx';
import request from '@/utils/request'

const CONFIG = {
    username: "dongenzhe",
    password: "aa228228!",
    filterId: "filter-27427"
};

export default {
    name: 'JiraDataDisplay',
    data() {
        return {
            loading: true,
            error: null,
            authToken: null,
            tableHeaders: [],  // 动态生成表头
            tableData: [],
            trendData: {
                months: [],
                defectCounts: []
            },
            chartInstance: null,
            originalData: null  // 保存原始数据
        }
    },
    methods: {
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

        async fetchJiraData() {
            try {
                const response = await request({
                    url: '/api/jira/filter-stats',
                    method: 'get',
                    params: {
                        filterId: CONFIG.filterId,
                        xstattype: "reporter",
                        ystattype: "project",
                        sortDirection: "asc",
                        sortBy: "natural",
                        numberToShow: 20
                    }
                });

                this.originalData = response;
                this.processJiraData(response);
                this.loading = false;

                this.$nextTick(() => {
                    this.initChart();
                });
            } catch (error) {
                console.error('获取JIRA数据失败:', error);
                this.error = error.message || '获取JIRA数据失败';
                this.loading = false;
            }
        },

        processJiraData(data) {
            // 1. 处理表头 - 提取第一行作为列名
            this.tableHeaders = ['项目']; // 第一列固定为项目名称
            data.firstRow.cells.forEach(cell => {
                this.tableHeaders.push(cell.markup.replace(/<[^>]*>/g, '').trim());
            });

            // 2. 处理表格数据
            this.tableData = data.rows.map(row => {
                const rowItem = {};

                // 第一列是项目名称
                const projectName = row.cells[0].markup.replace(/<[^>]*>/g, '').trim();
                rowItem['项目'] = {
                    display: projectName,
                    value: projectName,
                    link: null,
                    raw: row.cells[0].markup
                };

                // 处理其他列数据
                row.cells.slice(1).forEach((cell, index) => {
                    const header = this.tableHeaders[index + 1]; // +1因为第一列是项目

                    // 提取显示文本（移除HTML标签）
                    const displayText = cell.markup.replace(/<[^>]*>/g, '').trim();

                    // 提取链接（如果有）
                    const linkMatch = cell.markup.match(/href='([^']*)'/);
                    const link = linkMatch ? linkMatch[1] : null;

                    // 提取数值（从类似">15<"的文本中提取数字）
                    const numericValue = parseInt(displayText) || 0;

                    rowItem[header] = {
                        display: displayText,
                        value: numericValue,
                        link: link,
                        raw: cell.markup,
                        isTotal: cell.classes && cell.classes.includes('totals') // 标记总计行
                    };
                });

                return rowItem;
            });

            // 3. 生成趋势数据（使用最后一列作为数据点）
            if (this.tableData.length > 0 && this.tableHeaders.length > 0) {
                const lastHeader = this.tableHeaders[this.tableHeaders.length - 1];
                this.trendData = {
                    // 使用项目名作为X轴标签
                    months: this.tableData.map(row => row['项目'].display),
                    defectCounts: this.tableData.map(row => row[lastHeader].value)
                };
            }
        },

        initChart() {
            if (!this.$refs.trendChart || !this.trendData) return;

            this.chartInstance = echarts.init(this.$refs.trendChart);

            // 使用更美观的配色方案
            const colorPalette = [
                '#5470C6', '#91CC75', '#FAC858', '#EE6666',
                '#73C0DE', '#3BA272', '#FC8452', '#9A60B4',
                '#EA7CCC', '#1E90FF', '#FF6347', '#32CD32'
            ];

            // 准备系列数据 - 每个人一个系列，过滤掉"合计:"字段
            const series = [];
            const reporters = this.tableHeaders
                .slice(1) // 去掉"项目"列，剩下的都是报告人
                .filter(reporter => !reporter.includes('合计:')); // 过滤掉包含"合计:"的字段

            reporters.forEach((reporter, index) => {
                const data = this.tableData.map(row => row[reporter] ? row[reporter].value : 0);
                series.push({
                    name: reporter,
                    type: 'line',
                    data: data,
                    smooth: true,
                    symbol: 'circle',
                    symbolSize: 8,
                    itemStyle: {
                        color: colorPalette[index % colorPalette.length]
                    },
                    lineStyle: {
                        width: 3,
                        color: colorPalette[index % colorPalette.length]
                    },
                    label: {
                        show: true,
                        position: 'top',
                        formatter: '{c}',
                        fontSize: 12
                    },
                    emphasis: {
                        focus: 'series',
                        label: {
                            show: true
                        }
                    }
                });
            });

            // 准备x轴数据 - 项目名称
            const projects = this.tableData.map(row => row['项目'].display);

            const option = {
                title: {
                    text: (this.originalData && this.originalData.filter && this.originalData.filter.filterTitle) || '缺陷数量趋势',
                    left: 'center',
                    textStyle: {
                        fontSize: 18,
                        fontWeight: 'bold'
                    }
                },
                tooltip: {
                    trigger: 'axis',
                    axisPointer: {
                        type: 'cross',
                        label: {
                            backgroundColor: '#6a7985'
                        }
                    }
                },
                legend: {
                    data: reporters,
                    bottom: 0,
                    type: 'scroll',  // 如果图例过多可以滚动
                    padding: [5, 20],
                    itemGap: 10,
                    textStyle: {
                        fontSize: 12
                    }
                },
                grid: {
                    left: '3%',
                    right: '4%',
                    bottom: '15%',
                    top: '15%',
                    containLabel: true
                },
                toolbox: {
                    feature: {
                        saveAsImage: {
                            title: '保存图片',
                            pixelRatio: 2
                        },
                        magicType: {
                            title: {
                                line: '切换为折线图',
                                bar: '切换为柱状图'
                            },
                            type: ['line', 'bar']
                        },
                        restore: {
                            title: '还原'
                        }
                    },
                    right: 20,
                    top: 0
                },
                xAxis: {
                    type: 'category',
                    boundaryGap: false,
                    data: projects,
                    axisLabel: {
                        interval: 0,
                        // 自动换行处理长标签
                        formatter: function (value) {
                            const maxLength = 10; // 每行最大字符数
                            const result = [];
                            let currentLine = '';

                            value.split('').forEach(char => {
                                if (currentLine.length < maxLength) {
                                    currentLine += char;
                                } else {
                                    result.push(currentLine);
                                    currentLine = char;
                                }
                            });

                            if (currentLine) {
                                result.push(currentLine);
                            }

                            return result.join('\n');
                        }
                    },
                    axisTick: {
                        alignWithLabel: true
                    }
                },
                yAxis: {
                    type: 'value',
                    name: '缺陷数量',
                    nameLocation: 'middle',
                    nameGap: 30,
                    axisLine: {
                        show: true
                    },
                    axisTick: {
                        show: true
                    },
                    splitLine: {
                        lineStyle: {
                            type: 'dashed'
                        }
                    }
                },
                series: series,
                // 添加动画效果
                animationDuration: 1000,
                animationEasing: 'cubicInOut'
            };

            this.chartInstance.setOption(option);
            window.addEventListener('resize', this.handleResize);
        },

        generateTrendData() {
            // 使用已处理的trendData
            return this.trendData;
        },

        exportExcel() {
            // 转换数据格式为扁平结构
            const excelData = this.tableData.map(row => {
                const flatRow = {};
                Object.keys(row).forEach(key => {
                    flatRow[key] = row[key].value;
                });
                return flatRow;
            });

            const worksheet = XLSX.utils.json_to_sheet(excelData);
            const workbook = XLSX.utils.book_new();
            XLSX.utils.book_append_sheet(workbook, worksheet, "JIRA数据");
            XLSX.writeFile(workbook, "jira_data.xlsx");
        },

        handleResize() {
            this.chartInstance && this.chartInstance.resize();
        },
        async retry() {
            this.error = null;
            this.loading = true;
            await this.fetchData();
        },
        async fetchData() {
            const isLoggedIn = await this.loginToJira();
            if (isLoggedIn) {
                await this.fetchJiraData();
            }
        }
    },
    async mounted() {
        await this.fetchData();
    },
    beforeDestroy() {
        window.removeEventListener('resize', this.handleResize);
        if (this.chartInstance) {
            this.chartInstance.dispose();
            this.chartInstance = null;
        }
    }
}
</script>

<style scoped>
.data-table tr td[class*="totals"],
.data-table tr[class*="totals"] {
    font-weight: bold;
    background-color: #f0f0f0;
}

.jira-data-container {
    padding: 20px;
    font-family: Arial, sans-serif;
    max-width: 1200px;
    margin: 0 auto;
}

.loading-section {
    margin: 20px 0;
    color: #666;
    text-align: center;
    padding: 50px;
}

.error-section {
    margin: 20px 0;
    color: #ff4d4f;
    text-align: center;
    padding: 50px;
}

.error-section button {
    margin-top: 15px;
    padding: 8px 16px;
    background-color: #ff4d4f;
    color: white;
    border: none;
    border-radius: 4px;
    cursor: pointer;
}

.action-buttons {
    margin: 20px 0;
    display: flex;
    gap: 10px;
}

.back-button,
.export-button {
    padding: 8px 16px;
    border: 1px solid #ccc;
    background-color: #f5f5f5;
    cursor: pointer;
    border-radius: 4px;
    transition: background-color 0.3s;
}

.back-button:hover,
.export-button:hover {
    background-color: #e5e5e5;
}

.defect-trend,
.table-view {
    margin-top: 30px;
    border-top: 1px solid #eee;
    padding-top: 20px;
}

h1,
h2 {
    color: #333;
}

.chart-container {
    height: 400px;
    margin: 20px 0;
}

.data-table {
    width: 100%;
    border-collapse: collapse;
    margin-top: 15px;
}

.data-table th,
.data-table td {
    border: 1px solid #ddd;
    padding: 8px;
    text-align: left;
}

.data-table th {
    background-color: #f2f2f2;
    font-weight: bold;
}

.data-table tr:nth-child(even) {
    background-color: #f9f9f9;
}

.data-table tr:hover {
    background-color: #f1f1f1;
}
</style>
