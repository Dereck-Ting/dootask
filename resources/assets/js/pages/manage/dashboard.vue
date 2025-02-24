<template>
    <div class="page-dashboard">
        <PageTitle :title="$L('仪表盘')"/>
        <div class="dashboard-wrapper">
            <div class="dashboard-hello">{{$L('欢迎您，' + userInfo.nickname)}}</div>
            <div class="dashboard-desc">{{$L('以下是你当前的任务统计数据')}}</div>
            <ul class="dashboard-block">
                <li @click="dashboard='today'">
                    <div class="block-title">{{$L('今日待完成')}}</div>
                    <div class="block-data">
                        <div class="block-num">{{dashboardTask.today.length}}</div>
                        <i class="taskfont">&#xe6f4;</i>
                    </div>
                </li>
                <li @click="dashboard='overdue'">
                    <div class="block-title">{{$L('超期未完成')}}</div>
                    <div class="block-data">
                        <div class="block-num">{{dashboardTask.overdue.length}}</div>
                        <i class="taskfont">&#xe603;</i>
                    </div>
                </li>
                <li>
                    <div class="block-title">{{$L('参与的项目')}}</div>
                    <div class="block-data">
                        <div class="block-num">{{projects.length}}</div>
                        <i class="taskfont">&#xe6f9;</i>
                    </div>
                </li>
            </ul>
            <template v-if="list.length > 0">
                <div class="dashboard-title">{{title}}</div>
                <ul class="dashboard-list overlay-y">
                    <li
                        v-for="item in list"
                        :key="item.id"
                        :class="{complete: item.complete_at}"
                        :style="item.color ? {backgroundColor: item.color} : {}"
                        @click="openTask(item)">
                        <em
                            v-if="item.p_name"
                            class="priority-color"
                            :style="{backgroundColor:item.p_color}"></em>
                        <EDropdown
                            trigger="click"
                            size="small"
                            placement="bottom"
                            @command="dropTask(item, $event)">
                            <div class="drop-icon" @click.stop="">
                                <i class="taskfont" v-html="item.complete_at ? '&#xe627;' : '&#xe625;'"></i>
                            </div>
                            <EDropdownMenu slot="dropdown" class="project-list-more-dropdown-menu">
                                <EDropdownItem v-if="item.complete_at" command="uncomplete">
                                    <div class="item red">
                                        <Icon type="md-checkmark-circle-outline" />{{$L('标记未完成')}}
                                    </div>
                                </EDropdownItem>
                                <EDropdownItem v-else command="complete">
                                    <div class="item">
                                        <Icon type="md-radio-button-off" />{{$L('完成')}}
                                    </div>
                                </EDropdownItem>
                                <EDropdownItem v-if="item.parent_id === 0" command="archived">
                                    <div class="item">
                                        <Icon type="ios-filing" />{{$L('归档')}}
                                    </div>
                                </EDropdownItem>
                                <EDropdownItem command="remove">
                                    <div class="item">
                                        <Icon type="md-trash" />{{$L('删除')}}
                                    </div>
                                </EDropdownItem>
                                <template v-if="item.parent_id === 0">
                                    <EDropdownItem divided disabled>{{$L('背景色')}}</EDropdownItem>
                                    <EDropdownItem v-for="(c, k) in $store.state.taskColorList" :key="k" :command="c">
                                        <div class="item">
                                            <i class="taskfont" :style="{color:c.color||'#f9f9f9'}" v-html="c.color == item.color ? '&#xe61d;' : '&#xe61c;'"></i>{{$L(c.name)}}
                                        </div>
                                    </EDropdownItem>
                                </template>
                            </EDropdownMenu>
                        </EDropdown>
                        <div class="item-title">
                            <span v-if="item.sub_top === true">{{$L('子任务')}}</span>
                            <span v-if="item.sub_my && item.sub_my.length > 0">+{{item.sub_my.length}}</span>
                            {{item.name}}
                        </div>
                        <div v-if="item.desc" class="item-icon">
                            <i class="taskfont">&#xe71a;</i>
                        </div>
                        <div v-if="item.sub_num > 0" class="item-icon">
                            <i class="taskfont">&#xe71f;</i>
                            <em>{{item.sub_complete}}/{{item.sub_num}}</em>
                        </div>
                        <div :class="['item-icon', item.today ? 'today' : '', item.overdue ? 'overdue' : '']">
                            <i class="taskfont">&#xe71d;</i>
                            <em>{{expiresFormat(item.end_at)}}</em>
                        </div>
                    </li>
                </ul>
            </template>
        </div>
        <AppDown/>
    </div>
</template>

<script>
import {mapGetters, mapState} from "vuex";
import AppDown from "../../components/AppDown";
import {Store} from "le5le-store";

export default {
    components: {AppDown},
    data() {
        return {
            nowTime: $A.Time(),
            nowInterval: null,

            loadIng: 0,
            dashboard: 'today',

            taskLoad: {},

            tempShowTasks: [],
        }
    },

    mounted() {
        this.nowInterval = setInterval(() => {
            this.nowTime = $A.Time();
        }, 1000)
    },

    destroyed() {
        clearInterval(this.nowInterval)
    },

    activated() {
        this.$store.dispatch("getTaskForDashboard");
    },

    computed: {
        ...mapState(['userInfo', 'projects', 'tasks', 'taskId']),

        ...mapGetters(['dashboardTask', 'transforTasks']),

        title() {
            const {dashboard} = this;
            switch (dashboard) {
                case 'today':
                    return this.$L('今日任务');
                case 'overdue':
                    return this.$L('超期任务');
                default:
                    return '';
            }
        },

        list() {
            const {dashboard, tempShowTasks} = this;
            let data = [];
            switch (dashboard) {
                case 'today':
                    data = this.transforTasks(this.dashboardTask.today);
                    break
                case 'overdue':
                    data = this.transforTasks(this.dashboardTask.overdue);
                    break
            }
            if (tempShowTasks.length > 0) {
                data.push(...tempShowTasks);
            }
            return data.sort((a, b) => {
                return $A.Date(a.end_at) - $A.Date(b.end_at);
            });
        },
    },

    watch: {
        '$route'() {
            this.tempShowTasks = [];
        },
        dashboard() {
            this.tempShowTasks = [];
        }
    },

    methods: {
        dropTask(task, command) {
            switch (command) {
                case 'complete':
                    if (task.complete_at) return;
                    this.updateTask(task, {
                        complete_at: $A.formatDate("Y-m-d H:i:s")
                    }).then(() => {
                        this.tempShowTasks.push(task)
                    })
                    break;
                case 'uncomplete':
                    if (!task.complete_at) return;
                    this.updateTask(task, {
                        complete_at: false
                    }).then(() => {
                        this.tempShowTasks = this.tempShowTasks.filter(({id}) => id != task.id)
                    })
                    break;
                case 'archived':
                case 'remove':
                    this.archivedOrRemoveTask(task, command);
                    break;
                default:
                    if (command.name) {
                        this.updateTask(task, {
                            color: command.color
                        })
                    }
                    break;
            }
        },

        openTask(task) {
            this.$store.dispatch("openTask", task)
        },

        updateTask(task, updata) {
            return new Promise((resolve, reject) => {
                if (this.taskLoad[task.id] === true) {
                    reject()
                    return;
                }
                this.$set(this.taskLoad, task.id, true);
                //
                Object.keys(updata).forEach(key => this.$set(task, key, updata[key]));
                //
                this.$store.dispatch("taskUpdate", Object.assign(updata, {
                    task_id: task.id,
                })).then(() => {
                    this.$set(this.taskLoad, task.id, false);
                    resolve()
                }).catch(({msg}) => {
                    $A.modalError(msg);
                    this.$set(this.taskLoad, task.id, false);
                    this.$store.dispatch("getTaskOne", task.id);
                    reject();
                });
            })
        },

        archivedOrRemoveTask(task, type) {
            let typeDispatch = type == 'remove' ? 'removeTask' : 'archivedTask';
            let typeName = type == 'remove' ? '删除' : '归档';
            let typeTask = task.parent_id > 0 ? '子任务' : '任务';
            $A.modalConfirm({
                title: typeName + typeTask,
                content: '你确定要' + typeName + typeTask + '【' + task.name + '】吗？',
                loading: true,
                onOk: () => {
                    if (this.taskLoad[task.id] === true) {
                        this.$Modal.remove();
                        return;
                    }
                    this.$set(this.taskLoad, task.id, true);
                    this.$store.dispatch(typeDispatch, task.id).then(({msg}) => {
                        $A.messageSuccess(msg);
                        this.$Modal.remove();
                        this.$set(this.taskLoad, task.id, false);
                    }).catch(({msg}) => {
                        $A.modalError(msg, 301);
                        this.$Modal.remove();
                        this.$set(this.taskLoad, task.id, false);
                    });
                }
            });
        },

        expiresFormat(date) {
            return $A.countDownFormat(date, this.nowTime)
        },
    }
}
</script>
