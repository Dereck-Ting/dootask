<template>
    <div class="task-rows">
        <div v-for="(item, key) in list" :key="key">
            <Row class="task-row" :style="item.color ? {backgroundColor: item.color, borderBottomColor: item.color} : {}">
                <em v-if="item.p_name" class="priority-color" :style="{backgroundColor:item.p_color}"></em>
                <Col span="12" :class="['row-name', item.complete_at ? 'complete' : '']">
                    <Icon
                        v-if="(item.sub_num > 0 && item.sub_top !== true) || (item.parent_id === 0 && fastAddTask)"
                        :class="['sub-icon', taskOpen[item.id] ? 'active' : '']"
                        type="ios-arrow-forward"
                        @click="getSublist(item)"/>
                    <EDropdown
                        trigger="click"
                        size="small"
                        placement="bottom"
                        @command="dropTask(item, $event)">
                        <div class="drop-icon">
                            <Icon v-if="item.complete_at" class="completed" type="md-checkmark-circle" />
                            <Icon v-else type="md-radio-button-off" />
                            <div v-if="taskLoad[item.id] === true" class="loading"><Loading /></div>
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
                    <div class="item-title" @click="openTask(item)">
                        <span v-if="item.sub_top === true">{{$L('子任务')}}</span>
                        <span v-if="item.sub_my && item.sub_my.length > 0">+{{item.sub_my.length}}</span>
                        {{item.name}}
                    </div>
                    <div class="item-icons" @click="openTask(item)">
                        <div v-if="item.desc" class="item-icon">
                            <i class="taskfont">&#xe71a;</i>
                        </div>
                        <div v-if="item.file_num > 0" class="item-icon">
                            <i class="taskfont">&#xe71c;</i>
                            <em>{{item.file_num}}</em>
                        </div>
                        <div v-if="item.msg_num > 0" class="item-icon">
                            <i class="taskfont">&#xe71e;</i>
                            <em>{{item.msg_num}}</em>
                        </div>
                        <div v-if="item.sub_num > 0" class="item-icon" @click.stop="getSublist(item)">
                            <i class="taskfont">&#xe71f;</i>
                            <em>{{item.sub_complete}}/{{item.sub_num}}</em>
                        </div>
                    </div>
                </Col>
                <Col span="3" class="row-column">
                    <EDropdown
                        trigger="click"
                        size="small"
                        placement="bottom"
                        :disabled="item.sub_top === true"
                        @command="dropTask(item, $event)">
                        <div class="task-column">{{columnName(item.column_id)}}</div>
                        <EDropdownMenu slot="dropdown">
                            <EDropdownItem v-for="column in columnList(item.project_id)" :key="column.id" :command="'column::' + column.id">
                                {{column.name}}
                            </EDropdownItem>
                        </EDropdownMenu>
                    </EDropdown>
                </Col>
                <Col span="3" class="row-priority">
                    <EDropdown
                        trigger="click"
                        size="small"
                        placement="bottom"
                        :disabled="item.sub_top === true"
                        @command="dropTask(item, $event)">
                        <TaskPriority :backgroundColor="item.p_color">{{item.p_name}}</TaskPriority>
                        <EDropdownMenu slot="dropdown">
                            <EDropdownItem v-for="(item, key) in taskPriority" :key="key" :command="'priority::' + key">
                                <i
                                    class="taskfont"
                                    :style="{color:item.color}"
                                    v-html="item.p_name == item.name ? '&#xe61d;' : '&#xe61c;'"></i>
                                {{item.name}}
                            </EDropdownItem>
                        </EDropdownMenu>
                    </EDropdown>
                </Col>
                <Col span="3" class="row-user">
                    <ul @click="openTask(item)">
                        <li v-for="(user, keyu) in ownerUser(item.task_user)" :key="keyu" v-if="keyu < 3">
                            <UserAvatar :userid="user.userid" size="32" :borderWitdh="2" :borderColor="item.color"/>
                        </li>
                        <li v-if="ownerUser(item.task_user).length === 0" class="no-owner">
                            <Button type="primary" size="small" @click.stop="openTask(item, true)">{{$L('领取任务')}}</Button>
                        </li>
                    </ul>
                </Col>
                <Col span="3" class="row-time">
                    <ETooltip
                        v-if="!item.complete_at && item.end_at"
                        :class="['task-time', item.today ? 'today' : '', item.overdue ? 'overdue' : '']"
                        :open-delay="600"
                        :content="item.end_at">
                        <div @click="openTask(item)">{{expiresFormat(item.end_at)}}</div>
                    </ETooltip>
                    <div v-else-if="showCompleteAt && item.complete_at" :title="item.complete_at">{{completeAtFormat(item.complete_at)}}</div>
                </Col>
            </Row>
            <TaskRow
                v-if="taskOpen[item.id]===true"
                :list="subTask(item.id)"
                :parent-id="item.id"
                :fast-add-task="item.parent_id===0 && fastAddTask"
                :open-key="openKey"
                @command="dropTask"/>
        </div>
        <TaskAddSimple v-if="fastAddTask || parentId > 0" :parent-id="parentId" row-mode @on-priority="onPriority"/>
    </div>
</template>

<script>
import TaskPriority from "./TaskPriority";
import TaskAddSimple from "./TaskAddSimple";
import {mapState} from "vuex";
import {Store} from "le5le-store";

export default {
    name: "TaskRow",
    components: {TaskAddSimple, TaskPriority},
    props: {
        list: {
            type: Array,
            default: () => {
                return [];
            }
        },
        parentId: {
            type: Number,
            default: 0
        },
        fastAddTask: {
            type: Boolean,
            default: false
        },
        openKey: {
            type: String,
            default: 'default'
        },
        showCompleteAt: {
            type: Boolean,
            default: false
        },
    },
    data() {
        return {
            nowTime: $A.Time(),
            nowInterval: null,

            taskLoad: {},
            taskOpen: {}
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

    computed: {
        ...mapState(['tasks', 'taskPriority', 'columns']),

        subTask() {
            return function(task_id) {
                return this.tasks.filter(({parent_id}) => {
                    return parent_id == task_id
                }).sort((a, b) => {
                    return a.id - b.id;
                });
            }
        },
    },
    methods: {
        columnName(column_id) {
            const column = this.columns.find(({id}) => id == column_id)
            return column ? column.name : '';
        },

        dropTask(task, command) {
            this.$emit("command", task, command)
        },

        onPriority(data) {
            this.$emit("on-priority", data)
        },

        getSublist(task) {
            if (task.sub_top === true) {
                this.openTask(task);
                return;
            }
            if (this.taskOpen[task.id] === true) {
                this.$set(this.taskOpen, task.id, false);
                return;
            }
            if (this.taskLoad[task.id] === true) {
                return;
            }
            this.$set(this.taskLoad, task.id, true);
            //
            this.$store.dispatch("getTaskForParent", task.id).then(() => {
                this.$set(this.taskLoad, task.id, false);
                this.$set(this.taskOpen, task.id, true);
            }).catch(({msg}) => {
                $A.modalError(msg);
                this.$set(this.taskLoad, task.id, false);
            });
        },

        columnList(id) {
            return this.columns.filter(({project_id}) => project_id == id);
        },

        openTask(task, receive) {
            this.$store.dispatch("openTask", task)
            if (receive === true) {
                // 向任务窗口发送领取任务请求
                setTimeout(() => {
                    Store.set('receiveTask', true);
                }, 300)
            }
        },

        ownerUser(list) {
            return list.filter(({owner}) => owner == 1).sort((a, b) => {
                return a.id - b.id;
            });
        },

        expiresFormat(date) {
            return $A.countDownFormat(date, this.nowTime)
        },

        completeAtFormat(date) {
            let time = Math.round($A.Date(date).getTime() / 1000);
            if ($A.formatDate('Y') === $A.formatDate('Y', time)) {
                return $A.formatDate('m-d H:i', time)
            } else {
                return $A.formatDate('Y-m-d', time)
            }
        }
    }
}
</script>
