<style>
    .vertical-center-modal {
        display: flex;
        align-items: center;
        justify-content: center;
        .ivu-modal{
            top: 0;
        }
    }
    div.charts > div {
        float: left;
        margin-bottom: 20px;
    }
    #date-picker-wrapper {
        margin-bottom: 10px;
    }
    .count-badge {
        background: #5cb85c !important;
    }
</style>
<template>
<div class="metrics-area">
    <div id="date-picker-wrapper">
        from:
        <Date-picker type="date" placeholder="请选择日期"
            :value="dateTimeRange.begin.date" format="yyyy-MM-dd"
            style="width: 120px; display: inline-block;"
            @on-change="onBeginDateChange"
        ></Date-picker>
        <Time-picker type="time" placeholder="选择时间"
            :value="dateTimeRange.begin.time" format="HH:mm"
            style="width: 100px; display: inline-block;"
            @on-change="onBeginTimeChange"
        ></Time-picker>
        &nbsp;to:
        <Date-picker type="date" placeholder="请选择日期"
            :value="dateTimeRange.end.date" format="yyyy-MM-dd"
            style="width: 120px; display: inline-block;"
            @on-change="onEndDateChange"
        ></Date-picker>
        <Time-picker type="time" placeholder="选择时间"
            :value="dateTimeRange.end.time" format="HH:mm"
            style="width: 100px; display: inline-block;"
            @on-change="onEndTimeChange"
        ></Time-picker>
        <div style="margin-top: 10px; display: inline-block;">
        最近
        <RadioGroup v-model="shortcutButton" type="button"
            @on-change="onShortcutButtonChange">
            <!-- <Radio label="10m"></Radio>
            <Radio label="30m"></Radio>
            <Radio label="1h"></Radio>
            <Radio label="2h"></Radio> -->
            <Radio label="6h"></Radio>
            <Radio label="12h"></Radio>
            <Radio label="1d"></Radio>
            <Radio label="7d"></Radio>
        </RadioGroup>
        </div>
        &nbsp;&nbsp;
        <Select v-model="filter.project" style="width:125px" placeholder="请选择服务">
            <Option v-for="project in projects" :value="project" :key="project">{{ project || '全部' }}</Option>
        </Select>
        &nbsp;&nbsp;
        <Badge :count="displayedRows" overflow-count="10000" class-name="count-badge">
            <Button type="primary" @click="exportData()">导出</Button>
        </Badge>
    </div>
    <div>
        <metrics-table ref="metricsTable" :data="filteredData" :height="tableHeight" :fix-column="fixColumn"
            @on-show-distribution="renderDistributionChart"
            @on-show-request="renderRequestChart"
            @on-filter-change="onFilterChange"
        ></metrics-table>
        <Modal v-model="distribution.show" width="640"
                class-name="vertical-center-modal">
            <p slot="header" style="text-align:center">
                <span>耗时分布图</span>
            </p>
            <distribution-chart :data="distribution.data"></distribution-chart>
            <div slot="footer">
            </div>
        </Modal>
        <Modal v-model="request.show" width="640"
                class-name="vertical-center-modal">
            <p slot="header" style="text-align:center">
                <span>请求分布图</span>
            </p>
            <request-chart :data="request.data"></request-chart>
            <div slot="footer">
            </div>
        </Modal>
    </div>
</div>
</template>
<script>
    import { distributionChart } from '../components/metricsCharts.vue';
    import { requestChart } from '../components/metricsCharts.vue';
    import { metricsTable } from '../components/metricsTables.vue';

    export default {
        components: {
            'distribution-chart': distributionChart,
            'request-chart': requestChart,
            'metrics-table': metricsTable
        },
        data () {
            const $this = this;
            return {
                shortcutButton: '10m',
                tableHeight: 600,
                dateTimeRange: {
                    begin: {
                        date: '',
                        time: ''
                    },
                    end: {
                        date: '',
                        time: ''
                    }
                },
                projects: [],
                filter: {
                    project: '',
                    url: '',
                    avgColumn: null
                },
                tableData: [],
                distribution: {
                    data: {},
                    show: false
                },
                request: {
                    data: {},
                    show: false
                },
                fixColumn: false
            }
        },
        computed: {
            beginDateTime() {
                if (!this.dateTimeRange || !this.dateTimeRange.begin) {
                    return '';
                }
                const { date, time } = this.dateTimeRange.begin;
                return date + ' ' + time;
            },
            endDateTime() {
                if (!this.dateTimeRange || !this.dateTimeRange.end) {
                    return '';
                }
                const { date, time } = this.dateTimeRange.end;
                return date + ' ' + time;
            },
            filteredData() {
                const { project, url } = this.filter;
                if (!project && !url) {
                    return this.tableData;
                }
                const filterProject = data => !project ? true : project == data.project;
                const urlRe = new RegExp(url);
                const filterUrl = data => !url ? true : urlRe.test(url);
                return this.tableData.filter(data => {
                    return filterProject(data) && filterUrl(data);
                });
            },
            displayedRows() {
                if (!this.filter.avgColumn || this.filter.avgColumn.options.length == 0) {
                    return this.filteredData.length;
                }
                return this.filteredData.filter(data => {
                    return this.filter.avgColumn.options
                        .find(v => this.filter.avgColumn.filterMethod(v, data))
                }).length;
            }
        },
        mounted: async function() {
            this.changeDateTimeRange(Date.now() - 1000 * 60 * 10, Date.now());
            this.projects = await this.loadProjects();
            this.renderTable();
            this.refreshTableColumns();
            this.refreshTableHeight();
            this.initRefreshTableSizeTimer();
        },
        methods: {
            changeDateTimeRange(beginMilliSeconds, endMilliSeconds) {
                const beginDateTime = this.$moment(beginMilliSeconds);
                const endDateTime = this.$moment(endMilliSeconds);
                this.dateTimeRange = {
                    begin: {
                        date: beginDateTime.format('YYYY-MM-DD'),
                        time: beginDateTime.format('HH:mm')
                    },
                    end: {
                        date: endDateTime.format('YYYY-MM-DD'),
                        time: endDateTime.format('HH:mm')
                    }
                };
            },
            loadProjects() {
                return new Promise((resolve, reject) => {
                    this.$http.get(`/monitor/projects.json`).then(resp => {
                        resolve(resp.data.data);
                    });
                });
            },
            onShortcutButtonChange(value) {
                const matchedResult = /(\d+)([mhd])/.exec(value);
                if (!matchedResult) {
                    return;
                }
                const units = {
                    m: 60 * 1000,
                    h: 60 * 60 * 1000,
                    d: 24 * 60 * 60 * 1000
                };
                const beginMilliSeconds = Date.now() - units[matchedResult[2]] * matchedResult[1];
                const endMilliSeconds = Date.now();
                this.changeDateTimeRange(beginMilliSeconds, endMilliSeconds);
                this.renderTable();
            },
            onBeginDateChange(date) {
                this.dateTimeRange.begin.date = date;
                this.shortcutButton = '';
                this.renderTable();
            },
            onBeginTimeChange(time) {
                this.dateTimeRange.begin.time = time;
                this.shortcutButton = '';
                this.renderTable();
            },
            onEndDateChange(date) {
                this.dateTimeRange.end.date = date;
                this.shortcutButton = '';
                this.renderTable();
            },
            onEndTimeChange(time) {
                this.dateTimeRange.end.time = time;
                this.shortcutButton = '';
                this.renderTable();
            },
            onFilterChange(column) {
                if (column.key == 'avg') {
                    this.filter.avgColumn = {
                        filterMethod: column.filterMethod,
                        options: column._filterChecked
                    }
                }
            },
            refreshTableColumns() {
                if (this.$refs.metricsTable && this.$refs.metricsTable.width() > 1200) {
                    this.fixColumn = false;
                } else {
                    this.fixColumn = true;
                }
            },
            refreshTableHeight() {
                this.tableHeight = window.innerHeight - this.$refs.metricsTable.offsetTop() - 15;
            },
            initRefreshTableSizeTimer() {
                let timer = null;
                this.$root.eventBus.$on('window-size-changed', () => {
                    if (timer) {
                        return;
                    }
                    timer = setTimeout(() => {
                        this.refreshTableColumns();
                        this.refreshTableHeight();
                        timer = null;
                    }, 800);
                });
            },
            renderTable() {
                const beginTime = this.$moment(this.beginDateTime, 'YYYY-MM-DD HH:mm').format('x');
                const endTime = this.$moment(this.endDateTime, 'YYYY-MM-DD HH:mm').format('x');
                const url = `/monitor/metrics/between/list.json?beginTime=${beginTime}&endTime=${endTime}`;
                this.$http.get(url).then(resp => {
                    const data = resp.data.data;
                    if (!data) {
                        return;
                    }
                    data.forEach(row => {
                        const { cnt, distributionList } = row;
                        let i = 0, len = distributionList.length, cur = 0, ceil = cnt * 0.95;
                        for (; i < len - 1; i++) {
                            cur += distributionList[i]['value'];
                            if (cur >= ceil) {
                                break;
                            }
                        }
                        row['95percentile'] = distributionList[i]['key'];
                    });
                    this.tableData = resp.data.data;
                });
            },
            renderDistributionChart(data) {
                const list = data.row.distributionList;
                this.distribution.show = true;
                this.distribution.data = {
                    title: data.row.urlPattern,
                    xData: list.map(entry => entry.key),
                    yData: list.map(entry => entry.value)
                };
            },
            renderRequestChart(data) {
                const beginTime = this.$moment(this.beginDateTime, 'YYYY-MM-DD HH:mm').format('x');
                const endTime = this.$moment(this.endDateTime, 'YYYY-MM-DD HH:mm').format('x');
                const urlPattern = data.row.urlPattern;
                const url = `/monitor/metrics/url/list.json?beginTime=${beginTime}&endTime=${endTime}&url=${encodeURIComponent(urlPattern)}`
                this.$http.get(url).then(resp => {
                    const list = resp.data.data;
                    this.request.data = {
                        title: data.row.urlPattern,
                        xData: list.map(entry => this.$moment(entry.endTime).format('YYYY/MM/DD HH:mm')),
                        yData: list.map(entry => entry.cnt)
                    };
                    this.request.show = true;
                });
            },
            exportData() {
                const { begin, end } = this.dateTimeRange;
                this.$refs.metricsTable.exportCsv({
                    filename: `url_metrics_${begin.date}_${begin.time}-${end.date}_${end.time}.csv`,
                    original: false
                })
            }
        }
    }
</script>