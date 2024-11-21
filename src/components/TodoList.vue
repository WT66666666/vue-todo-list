<template>
    <div class="todo-container">
        <h1>{{ title }}</h1>

        <!-- 控制面板组件 Control panel component -->
        <div class="control-panel">
            <!-- 输入组件 Input component -->
            <div class="todo-input">
                <input type="text" v-model="newTask" @keyup.enter="addTask" placeholder="Enter a new task...">
                <select v-model="newTaskPriority">
                    <option value="high">High Priority</option>
                    <option value="medium">Medium Priority</option>
                    <option value="low">Low Priority</option>
                </select>
                <button @click="addTask">Add Task</button>
            </div>

            <!-- 筛选组件 Filter component -->
            <div class="filter-section">
                <select v-model="statusFilter">
                    <option value="all">All Tasks</option>
                    <option value="active">Active</option>
                    <option value="completed">Completed</option>
                </select>
                <select v-model="priorityFilter">
                    <option value="all">All Priorities</option>
                    <option value="high">High Priority</option>
                    <option value="medium">Medium Priority</option>
                    <option value="low">Low Priority</option>
                </select>
                <input type="text" v-model="searchQuery" placeholder="Search tasks..." class="search-input">
            </div>
        </div>

        <!-- 任务列表 Task list -->
        <transition-group name="list" tag="ul" class="todo-list">
            <li v-for="task in filteredTasks" :key="task.id" :class="['priority-' + task.priority]">
                <template v-if="editingTask !== task.id">
                    <input type="checkbox" v-model="task.completed" @change="toggleTask(task)" :id="'task-' + task.id">
                    <label :for="'task-' + task.id" :class="{ completed: task.completed }"
                        @dblclick="startEditing(task)">
                        <i :class="priorityIcon(task.priority)"></i>
                        {{ task.text }}
                    </label>
                    <div class="task-actions">
                        <button @click="startEditing(task)" class="edit-btn">
                            <i class="fas fa-edit"></i>
                        </button>
                        <button @click="deleteTask(task.id)" class="delete-btn">
                            <i class="fas fa-trash"></i>
                        </button>
                    </div>
                </template>
                <template v-else>
                    <div class="edit-mode">
                        <input type="text" v-model="editingTaskText" @keyup.enter="saveTask" @keyup.esc="cancelEditing"
                            ref="editInput">
                        <select v-model="editingTaskPriority">
                            <option value="high">High</option>
                            <option value="medium">Medium</option>
                            <option value="low">Low</option>
                        </select>
                        <button @click="saveTask" class="save-btn">
                            <i class="fas fa-check"></i>
                        </button>
                        <button @click="cancelEditing" class="cancel-btn">
                            <i class="fas fa-times"></i>
                        </button>
                    </div>
                </template>
            </li>
        </transition-group>

        <!-- 通知组件 Notification component -->
        <div class="notification" :class="{ show: showNotification, [notificationType]: true }">
            {{ notificationMessage }}
        </div>
    </div>
</template>

<script>
export default {
    name: 'TodoList',

    // 数据定义 Data definitions
    data() {
        return {
            // 应用标题 Application title
            title: 'My Todo List',
            // 新任务输入 New task input
            newTask: '',
            // 新任务优先级 New task priority
            newTaskPriority: 'medium',
            // 任务列表 Task list
            tasks: [],
            // 用于生成唯一ID Generate unique ID
            nextId: 1,
            // 通知相关数据 Notification related data
            showNotification: false,
            notificationMessage: '',
            notificationType: 'success',
            notificationTimeout: null,
            // 筛选相关数据 Filter related data
            statusFilter: 'all',
            priorityFilter: 'all',
            searchQuery: '',
            // 编辑相关数据 Edit related data
            editingTask: null,
            editingTaskText: '',
            editingTaskPriority: ''
        }
    },

    // 计算属性 Computed properties
    computed: {
        // 过滤后的任务列表 Filtered task list
        filteredTasks() {
            return this.tasks.filter(task => {
                // 状态筛选 Status filter
                const statusMatch = this.statusFilter === 'all' ||
                    (this.statusFilter === 'completed' && task.completed) ||
                    (this.statusFilter === 'active' && !task.completed);

                // 优先级筛选 Priority filter
                const priorityMatch = this.priorityFilter === 'all' ||
                    this.priorityFilter === task.priority;

                // 搜索筛选 Search filter
                const searchMatch = task.text.toLowerCase()
                    .includes(this.searchQuery.toLowerCase());

                return statusMatch && priorityMatch && searchMatch;
            });
        }
    },

    // 方法定义 Method definitions
    methods: {
        // 显示通知消息 Show notification message
        showNotificationMessage(message, type = 'success') {
            if (this.notificationTimeout) {
                clearTimeout(this.notificationTimeout);
            }

            this.notificationMessage = message;
            this.notificationType = type;
            this.showNotification = true;

            this.notificationTimeout = setTimeout(() => {
                this.showNotification = false;
            }, 3000);
        },

        // 获取优先级图标 Get priority icon
        priorityIcon(priority) {
            const iconMap = {
                high: 'fas fa-exclamation-circle priority-icon',
                medium: 'fas fa-circle priority-icon',
                low: 'fas fa-arrow-circle-down priority-icon'
            };
            return iconMap[priority] || '';
        },

        // 添加新任务 Add new task
        addTask() {
            if (this.newTask.trim()) {
                this.tasks.push({
                    id: this.nextId++,
                    text: this.newTask.trim(),
                    completed: false,
                    priority: this.newTaskPriority
                });
                this.newTask = '';
                this.newTaskPriority = 'medium';
                this.saveTasks();
                this.showNotificationMessage('Task added successfully!');
            } else {
                this.showNotificationMessage('Please enter a task', 'error');
            }
        },

        // 删除任务 Delete task
        deleteTask(id) {
            this.tasks = this.tasks.filter(task => task.id !== id);
            this.saveTasks();
            this.showNotificationMessage('Task deleted successfully!');
        },

        // 切换任务状态 Toggle task status
        toggleTask(task) {
            this.saveTasks();
            this.showNotificationMessage(
                task.completed ? 'Task completed!' : 'Task uncompleted!'
            );
        },

        // 开始编辑任务 Start editing task
        startEditing(task) {
            this.editingTask = task.id;
            this.editingTaskText = task.text;
            this.editingTaskPriority = task.priority;
            this.$nextTick(() => {
                // 检查 editInput 是否存在 Check if editInput exists
                if (this.$refs.editInput) {
                    // 如果是数组，取第一个元素 If it's an array, take the first element
                    const input = Array.isArray(this.$refs.editInput)
                        ? this.$refs.editInput[0]
                        : this.$refs.editInput;
                    if (input && input.focus) {
                        input.focus();
                    }
                }
            });
        },

        // 保存编辑的任务 Save edited task
        saveTask() {
            const task = this.tasks.find(task => task.id === this.editingTask);
            if (task && this.editingTaskText.trim()) {
                task.text = this.editingTaskText.trim();
                task.priority = this.editingTaskPriority;
                this.editingTask = null;
                this.saveTasks();
                this.showNotificationMessage('Task updated successfully!');
            } else {
                this.showNotificationMessage('Task text cannot be empty', 'error');
            }
        },

        // 取消编辑 Cancel editing
        cancelEditing() {
            this.editingTask = null;
        },

        // 保存任务到localStorage Save tasks to localStorage
        saveTasks() {
            localStorage.setItem('tasks', JSON.stringify(this.tasks));
        },

        // 从localStorage加载任务 Load tasks from localStorage
        loadTasks() {
            const savedTasks = localStorage.getItem('tasks');
            if (savedTasks) {
                this.tasks = JSON.parse(savedTasks);
                // 更新nextId Update nextId
                this.nextId = Math.max(...this.tasks.map(task => task.id), 0) + 1;
            }
        }
    },

    // 生命周期钩子 - 当应用挂载完成时 Lifecycle hook - when application is mounted
    mounted() {
        this.loadTasks();
    },

    // 监听任务变化 Watch for task changes
    watch: {
        'tasks': {
            handler() {
                this.saveTasks();
            },
            deep: true
        }
    }
}
</script>

<style lang="scss">
/* 全局样式 Global styles */
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

/* 容器样式 Container styles */
.todo-container {
    max-width: 800px;
    margin: 2rem auto;
    padding: 2rem;
    background-color: white;
    border-radius: 10px;
    box-shadow: 0 2px 10px rgba(0, 0, 0, 0.1);

    h1 {
        text-align: center;
        color: #333;
        margin-bottom: 2rem;
    }
}

/* 控制面板样式 Control panel styles */
.control-panel {
    margin-bottom: 2rem;
}

/* 输入区域样式 Input area styles */
.todo-input {
    display: flex;
    gap: 10px;
    margin-bottom: 1rem;

    input {
        flex: 1;
        padding: 0.5rem;
        border: 1px solid #ddd;
        border-radius: 4px;
        font-size: 1rem;

        &:focus {
            outline: none;
            border-color: #4CAF50;
            box-shadow: 0 0 0 2px rgba(76, 175, 80, 0.2);
        }
    }

    select {
        padding: 0.5rem;
        border: 1px solid #ddd;
        border-radius: 4px;
        min-width: 120px;
        background-color: white;
        cursor: pointer;

        &:focus {
            outline: none;
            border-color: #4CAF50;
        }
    }

    button {
        padding: 0.5rem 1rem;
        background-color: #4CAF50;
        color: white;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: background-color 0.3s ease;

        &:hover {
            background-color: #45a049;
        }

        &:active {
            transform: scale(0.98);
        }
    }
}

/* 筛选区域样式 Filter section styles */
.filter-section {
    display: flex;
    gap: 10px;
    margin-top: 1rem;

    select,
    input {
        padding: 0.5rem;
        border: 1px solid #ddd;
        border-radius: 4px;
        background-color: white;

        &:focus {
            outline: none;
            border-color: #4CAF50;
        }
    }

    input {
        flex: 1;
    }
}

/* 任务列表样式 Task list styles */
.todo-list {
    list-style: none;

    li {
        display: flex;
        align-items: center;
        padding: 0.75rem;
        border-bottom: 1px solid #eee;
        gap: 10px;
        background-color: #fff;
        transition: all 0.3s ease;

        &:last-child {
            border-bottom: none;
        }

        &:hover {
            background-color: #f9f9f9;
        }
    }

    label {
        flex: 1;
        cursor: pointer;
        display: flex;
        align-items: center;
        gap: 5px;
    }

    .completed {
        text-decoration: line-through;
        color: #888;
    }
}

/* 优先级样式 Priority styles */
.priority-high {
    border-left: 4px solid #ff4444;
}

.priority-medium {
    border-left: 4px solid #ffbb33;
}

.priority-low {
    border-left: 4px solid #33b5e5;
}

/* 编辑模式样式 Edit mode styles */
.edit-mode {
    display: flex;
    gap: 10px;
    width: 100%;

    input {
        flex: 1;
        padding: 0.5rem;
        border: 1px solid #ddd;
        border-radius: 4px;

        &:focus {
            outline: none;
            border-color: #4CAF50;
        }
    }

    select {
        padding: 0.5rem;
        border: 1px solid #ddd;
        border-radius: 4px;
        min-width: 100px;
    }
}

/* 任务操作按钮样式 Task action buttons styles */
.task-actions {
    display: flex;
    gap: 5px;

    button {
        padding: 0.25rem 0.5rem;
        border: none;
        border-radius: 4px;
        cursor: pointer;
        transition: all 0.2s ease;
        color: white;

        &:hover {
            opacity: 0.9;
        }

        &:active {
            transform: scale(0.95);
        }
    }
}

.edit-btn {
    background-color: #33b5e5;
}

.delete-btn {
    background-color: #ff4444;
}

.save-btn {
    background-color: #4CAF50;
}

.cancel-btn {
    background-color: #ff4444;
}

/* 优先级图标样式 Priority icon styles */
.priority-icon {
    margin-right: 8px;
    font-size: 0.9em;
}

.priority-high .priority-icon {
    color: #ff4444;
}

.priority-medium .priority-icon {
    color: #ffbb33;
}

.priority-low .priority-icon {
    color: #33b5e5;
}

/* 通知样式 Notification styles */
.notification {
    position: fixed;
    top: 20px;
    right: 20px;
    padding: 10px 20px;
    border-radius: 4px;
    color: white;
    opacity: 0;
    transition: opacity 0.3s ease;
    z-index: 1000;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.2);

    &.show {
        opacity: 1;
    }

    &.success {
        background-color: #4CAF50;
    }

    &.error {
        background-color: #ff4444;
    }
}

/* 响应式布局 Responsive layout */
@media (max-width: 768px) {
    .todo-container {
        margin: 1rem;
        padding: 1rem;
    }

    .todo-input,
    .filter-section {
        flex-direction: column;

        button,
        select,
        input {
            width: 100%;
        }
    }

    .edit-mode {
        flex-wrap: wrap;

        input,
        select,
        button {
            width: 100%;
            margin-bottom: 0.5rem;
        }
    }

    .task-actions {
        width: 100%;
        justify-content: space-between;
    }
}

/* 动画效果 Animation effects */
.list-enter-active,
.list-leave-active {
    transition: all 0.5s ease;
}

.list-enter-from,
.list-leave-to {
    opacity: 0;
    transform: translateX(-30px);
}

/* 复选框样式 Checkbox styles */
input[type="checkbox"] {
    width: 18px;
    height: 18px;
    cursor: pointer;
}
</style>