<template>
	<el-container>
		<el-main class="mx-auto p-5">
			<div>
				<div class="py-4 text-center text-3xl font-extrabold">
					<span class="bg-clip-text text-transparent bg-gradient-to-r from-pink-500 to-violet-500">
						时探客 Task Time Tracker
					</span>
				</div>
				<div>
					<!-- Task List -->
					<el-table :data="tasks" style="width: 100%">
						<el-table-column prop="name" label="任务名称"></el-table-column>
						<el-table-column label="总时长" width="120px">
							<template v-slot="scope">
								{{ calculateTotalDuration(scope.row.timers) }}
							</template>
						</el-table-column>
						<el-table-column label="任务说明">
							<template v-slot="scope">
								<el-input
									v-model="taskDescriptions[scope.row.id]"
									type="textarea"
									:rows="2"
									placeholder="请输入任务说明"
									class="task-description-input"
								></el-input>
							</template>
						</el-table-column>
						<el-table-column label="计时状态" width="200px">
							<template v-slot="scope">
								<span v-if="isTaskRunning(scope.row.id)" class="text-blue-500 font-bold">
									已计时: {{ getRunningTime(scope.row.id) }}
								</span>
								<span v-else>未开始计时</span>
							</template>
						</el-table-column>
						<el-table-column label="操作" width="420px">
							<template v-slot="scope">
								<el-button @click="startTimer(scope.row.id)" type="primary"
									:disabled="isTaskRunning(scope.row.id)">开始计时</el-button>
								<el-button @click="stopTimer(scope.row.id)" type="danger"
									:disabled="!isTaskRunning(scope.row.id)">结束计时</el-button>
								<el-button @click="selectTask(scope.row.id)" type="info">查看甘特图</el-button>
								<el-dropdown trigger="click" @command="command => handleExport(command, scope.row)">
									<el-button type="success">导出<el-icon
											class="el-icon--right"><arrow-down /></el-icon></el-button>
									<template #dropdown>
										<el-dropdown-menu>
											<el-dropdown-item command="json">JSON</el-dropdown-item>
											<el-dropdown-item command="csv">CSV</el-dropdown-item>
										</el-dropdown-menu>
									</template>
								</el-dropdown>
								<el-button @click="deleteTask(scope.row)" type="danger">删除</el-button>
							</template>
						</el-table-column>
					</el-table>
					<!-- Add Task -->
					<el-input v-model="newTaskName" placeholder="任务名称" style="margin-top: 20px;"></el-input>
					<el-button @click="addTask" type="success" style="margin-top: 10px;">创建任务</el-button>
				</div>
				<!-- Global Actions -->
				<div class="flex gap-2 mt-4">
					<el-upload class="upload-demo" action="" :auto-upload="false" :show-file-list="false"
						accept=".json,.csv" :on-change="handleFileChange">
						<el-button type="primary">导入任务</el-button>
					</el-upload>
					<el-dropdown trigger="click" @command="handleGlobalExport">
						<el-button type="success">导出所有数据<el-icon
								class="el-icon--right"><arrow-down /></el-icon></el-button>
						<template #dropdown>
							<el-dropdown-menu>
								<el-dropdown-item command="json">JSON</el-dropdown-item>
							</el-dropdown-menu>
						</template>
					</el-dropdown>
					<el-button type="danger" @click="clearAllTasks">清除所有数据</el-button>
				</div>
				<!-- Time Records Display -->
				<div v-if="selectedTask" class="mt-5 p-5 border border-gray-200 rounded-lg">
					<h3 class="text-lg font-medium mb-4">{{ selectedTask.name }} - 时间记录</h3>
					<div class="flex items-center gap-4 px-5 mb-4">
						<span>时间单位宽度: {{ (baseUnitWidth * 3600).toFixed(1) }}px/小时</span>
						<el-slider v-model="baseUnitWidth" :min="0.0004" :max="0.004" :step="0.0001"
							@change="handleZoomChange" class="flex-1" />
					</div>

					<div class="relative border border-gray-200 rounded-lg overflow-hidden">
						<div @wheel.prevent="handleWheel" @mousedown="startDrag" @mousemove="onDrag" @mouseup="stopDrag"
							@mouseleave="stopDrag" ref="scrollContainer"
							class="relative w-full overflow-x-auto cursor-grab active:cursor-grabbing select-none">
							<div class="timeline-content" :style="{ minWidth: `${24 * 3600 * baseUnitWidth}rem` }">
								<div
									class="sticky top-0 z-10 flex h-8 items-center bg-gray-50 border-b border-gray-200 pl-24">
									<div v-for="mark in timeScaleMarks" :key="mark.time"
										:style="{ width: `${mark.width}rem` }"
										class="flex-shrink-0 text-xs text-gray-600 text-center border-r border-gray-200">
										{{ mark.label }}
									</div>
								</div>

								<div class="time-blocks-container">
									<div v-for="(blocks, date) in formattedTimeBlocks" :key="date"
										class="flex h-10 my-2.5 items-center border-b border-gray-100">
										<div
											class="sticky left-0 z-20 w-24 px-2.5 bg-white shadow-sm border-r border-gray-200 text-sm text-gray-700">
											{{ date }}
										</div>
										<div class="relative flex-grow h-full border-l border-gray-200">
											<div v-for="block in blocks" :key="block.id"
												class="absolute h-8 top-1 rounded text-xs text-white px-1 flex items-center justify-center overflow-hidden whitespace-nowrap opacity-100 hover:opacity-90 transition-opacity"
												:style="{
													left: `${calculateLeftPosition(block.start)}rem`,
													width: `${calculateDuration(block.start, block.end)}rem`,
													backgroundColor: block.color
												}" :title="`${block.description}
开始：${formatDetailTime(block.start)}
结束：${formatDetailTime(block.end)}
持续：${formatDuration(block.start, block.end)}`">
												{{ block.displayText }}
											</div>
										</div>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>
			</div>
		</el-main>
	</el-container>
</template>

<script>
import { ref, onMounted, onUnmounted, nextTick, computed, watch } from 'vue'
import { ElTable, ElTableColumn, ElButton, ElInput, ElContainer, ElMain, ElMessage, ElMessageBox } from 'element-plus'
import { ArrowDown } from '@element-plus/icons-vue'
import 'element-plus/dist/index.css'
import './styles/index.css'

export default {
	components: {
		ElTable,
		ElTableColumn,
		ElButton,
		ElInput,
		ElContainer,
		ElMain,
		ArrowDown
	},
	data() {
		return {
			tasks: [],
			newTaskName: '',
			ganttChart: null,
			selectedTask: null,
			ganttData: [],
			baseUnitWidth: 0.002, // 默认0.002rem/秒，约7.2rem/小时
			scrollContainer: null,
			taskDescriptions: {},
			STORAGE_KEY: 'taskTimeTracker_data',
			isDragging: false,
			startX: 0,
			scrollLeft: 0,
			taskColors: {
				colorList: [
					'#409EFF', // 蓝色
					'#67C23A', // 绿色
					'#E6A23C', // 黄色
					'#F56C6C', // 红色
					'#626aef', // 靛蓝
					'#6f7ad3', // 紫色
					'#5cb87a', // 浅绿
					'#ff9f7f'  // 橙色
				]
			},
			standardColors: [
				'#409EFF', // 蓝色
				'#67C23A', // 绿色
				'#E6A23C', // 黄色
				'#F56C6C', // 红色
				'#909399', // 灰色
				'#626aef', // 靛蓝
				'#6f7ad3', // 紫色
				'#1989fa', // 浅蓝
				'#5cb87a', // 浅绿
				'#ff9f7f'  // 橙色
			],
			timerInterval: null,
			formattedTimeBlocks: {}
		}
	},
	mounted() {
		this.loadFromStorage();
		this.timerInterval = setInterval(() => {
			this.tasks.forEach(task => {
				if (this.isTaskRunning(task.id)) {
					// Force update
					task.timers = [...task.timers];
				}
			});
		}, 1000);
	},
	beforeUnmount() {
		if (this.timerInterval) {
			clearInterval(this.timerInterval);
		}
	},
	watch: {
		tasks: {
			handler() {
				if (this.selectedTask) {
					const updatedTask = this.tasks.find(t => t.id === this.selectedTask.id);
					if (updatedTask) {
						this.selectedTask = updatedTask;
						this.formattedTimeBlocks = this.formatTimeBlocks(updatedTask);
					}
				}
			},
			deep: true
		}
	},
	computed: {
		timeScaleMarks() {
			const interval = this.getTimeInterval(this.baseUnitWidth);
			const marks = [];
			const totalMinutes = 24 * 60;

			for (let minute = 0; minute < totalMinutes; minute += interval.interval) {
				if (interval.unit === 'hour' && minute % 60 !== 0) continue;

				const hours = Math.floor(minute / 60);
				const minutes = minute % 60;
				const label = interval.unit === 'hour'
					? `${hours.toString().padStart(2, '0')}:00`
					: `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}`;

				const width = interval.unit === 'hour'
					? this.baseUnitWidth * 3600
					: this.baseUnitWidth * interval.interval * 60;

				marks.push({ time: minute, label, width });
			}
			return marks;
		}
	},
	methods: {
		addTask() {
			if (this.newTaskName.trim()) {
				this.tasks.push({
					id: Date.now(),
					name: this.newTaskName,
					timers: []
				});
				this.newTaskName = '';
			}
		},
		isTaskRunning(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			return task && task.timers.some(timer => timer.end === null);
		},
		getRunningTime(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			if (!task) return '';

			const activeTimer = task.timers.find(timer => timer.end === null);
			if (!activeTimer) return '';

			const diff = new Date() - new Date(activeTimer.start);
			const hours = Math.floor(diff / 3600000);
			const minutes = Math.floor((diff % 3600000) / 60000);
			const seconds = Math.floor((diff % 60000) / 1000);
			return `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
		},
		getRandomColor() {
			return this.taskColors.colorList[Math.floor(Math.random() * this.taskColors.colorList.length)];
		},
		startTimer(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			if (task && !this.isTaskRunning(taskId)) {
				this.taskDescriptions[taskId] = ''; // Initialize empty description
				task.timers.push({
					start: new Date(),
					end: null,
					description: '',
					color: null // 先不分配颜色
				});
			}
		},
		stopTimer(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			const description = this.taskDescriptions[taskId]?.trim();

			if (!description) {
				ElMessage.error('请先填写任务说明再结束计时');
				return;
			}

			if (task) {
				const timer = task.timers.find(t => t.end === null);
				if (timer) {
					timer.end = new Date();
					timer.description = description;
					timer.color = this.getRandomColor(); // 结束时分配固定颜色
					this.taskDescriptions[taskId] = ''; // Clear description after stopping
					this.saveToStorage(); // 保存到本地存储以保持颜色信息
				}
			}
		},
		formatTime(date) {
			return `${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}`;
		},
		calculateLeftPosition(date) {
			const totalSeconds = date.getHours() * 3600 + date.getMinutes() * 60 + date.getSeconds();
			return totalSeconds * this.baseUnitWidth;
		},
		calculateDuration(start, end) {
			const durationInSeconds = (end - start) / 1000;
			return durationInSeconds * this.baseUnitWidth;
		},
		formatDetailTime(date) {
			return `${date.getHours().toString().padStart(2, '0')}:${date.getMinutes().toString().padStart(2, '0')}:${date.getSeconds().toString().padStart(2, '0')}`;
		},
		formatDuration(start, end) {
			const seconds = Math.floor((end - start) / 1000);
			const hours = Math.floor(seconds / 3600);
			const minutes = Math.floor((seconds % 3600) / 60);
			const remainingSeconds = seconds % 60;

			const parts = [];
			if (hours > 0) parts.push(`${hours}小时`);
			if (minutes > 0) parts.push(`${minutes}分钟`);
			if (remainingSeconds > 0 || parts.length === 0) parts.push(`${remainingSeconds}秒`);

			return parts.join(' ');
		},
		formatTimeBlocks(taskData) {
			if (!taskData?.timers?.length) return {};

			const blocks = taskData.timers.reduce((acc, timer) => {
				const date = new Date(timer.start).toLocaleDateString('zh-CN');
				if (!acc[date]) acc[date] = [];

				const start = new Date(timer.start);
				const end = timer.end ? new Date(timer.end) : new Date();
				const duration = (end - start) / 1000; // Duration in seconds

				// 根据持续时间决定显示内容
				const displayText = duration < 60 ? // 小于1分钟
					`${Math.floor(duration)}秒` :
					timer.description || '未命名时间段';

				acc[date].push({
					id: Date.now() + Math.random(),
					description: timer.description || '未命名时间段',
					start,
					end,
					duration,
					displayText,
					color: timer.color || '#909399' // 使用保存的颜色，如果没有则使用默认灰色
				});

				return acc;
			}, {});

			return blocks;
		},
		onTaskSelected(task) {
			console.log('Selected task:', task);
		},
		showTaskGantt(task) {
			this.selectedTask = task;
			this.formattedTimeBlocks = this.formatTimeBlocks(task);
		},
		selectTask(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			if (task) {
				this.showTaskGantt(task);
			}
		},
		handleScroll(e) {
			if (this.scrollContainer) {
				const container = this.scrollContainer;
				container.querySelector('.time-scale').scrollLeft = container.scrollLeft;
				container.querySelector('.time-blocks-container').scrollLeft = container.scrollLeft;
			}
		},
		getTimeInterval(unitWidth) {
			const pixelsPerHour = unitWidth * 3600;
			if (pixelsPerHour <= 100) return { unit: 'hour', interval: 6 };      // 6小时
			if (pixelsPerHour <= 150) return { unit: 'hour', interval: 4 };      // 4小时
			if (pixelsPerHour <= 200) return { unit: 'hour', interval: 2 };      // 2小时
			if (pixelsPerHour <= 300) return { unit: 'hour', interval: 1 };      // 1小时
			return { unit: 'minute', interval: 30 };                             // 30分钟
		},
		handleZoomChange(newWidth) {
			if (!this.scrollContainer) return;

			const container = this.scrollContainer;
			const oldWidth = this.baseUnitWidth;

			// 使用上次记录的鼠标位置作为缩放中心
			const rect = container.getBoundingClientRect();
			const mouseX = container._lastMouseX || rect.left + (rect.width / 2);
			const relativeX = mouseX - rect.left;

			// 计算鼠标位置对应的时间点
			const timeAtCursor = (container.scrollLeft + relativeX) / oldWidth;

			// 更新缩放级别
			this.baseUnitWidth = newWidth;

			// 保持鼠标指向的时间点不变
			nextTick(() => {
				const newScrollPos = timeAtCursor * newWidth - relativeX;
				container.scrollLeft = Math.max(0, newScrollPos);
			});
		},
		handleWheel(e) {
			if (e.ctrlKey || e.metaKey) {
				e.preventDefault();
				const container = this.scrollContainer;

				// 记录当前鼠标位置用于滑块缩放
				container._lastMouseX = e.clientX;

				const rect = container.getBoundingClientRect();
				const relativeX = e.clientX - rect.left;
				const timeAtCursor = (container.scrollLeft + relativeX) / this.baseUnitWidth;

				// 计算新的缩放级别
				const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1; // 每次缩放10%
				const newWidth = Math.min(0.004, Math.max(0.0004, this.baseUnitWidth * zoomFactor));

				if (newWidth !== this.baseUnitWidth) {
					this.baseUnitWidth = newWidth;

					// 保持鼠标指向的时间点不变
					nextTick(() => {
						const newScrollPos = timeAtCursor * newWidth - relativeX;
						container.scrollLeft = Math.max(0, newScrollPos);
					});
				}
			}
		},
		startDrag(e) {
			this.isDragging = true;
			this.startX = e.pageX - this.scrollContainer.offsetLeft;
			this.scrollLeft = this.scrollContainer.scrollLeft;
		},
		onDrag(e) {
			if (!this.isDragging) return;
			e.preventDefault();
			const x = e.pageX - this.scrollContainer.offsetLeft;
			const walk = (x - this.startX) * 2;
			this.scrollContainer.scrollLeft = this.scrollLeft - walk;
		},
		stopDrag() {
			this.isDragging = false;
		},
		calculateTotalDuration(timers) {
			if (!timers?.length) return '0小时0分钟';
			const totalSeconds = timers.reduce((acc, timer) => {
				const end = timer.end ? new Date(timer.end) : new Date();
				return acc + (end - new Date(timer.start)) / 1000;
			}, 0);

			const hours = Math.floor(totalSeconds / 3600);
			const minutes = Math.floor((totalSeconds % 3600) / 60);
			return `${hours}小时${minutes}分钟`;
		},
		saveToStorage() {
			localStorage.setItem(this.STORAGE_KEY, JSON.stringify(this.tasks));
		},
		loadFromStorage() {
			const data = localStorage.getItem(this.STORAGE_KEY);
			if (data) {
				const parsed = JSON.parse(data);
				// 转换时间字符串为Date对象
				parsed.forEach(task => {
					task.timers.forEach(timer => {
						timer.start = new Date(timer.start);
						if (timer.end) timer.end = new Date(timer.end);
					});
				});
				this.tasks = parsed;
			}
		},
		handleExport(format, task) {
			if (format === 'json') {
				const dataStr = JSON.stringify(task, null, 2);
				this.downloadFile(dataStr, `task_${task.id}.json`, 'application/json');
			} else if (format === 'csv') {
				const headers = ['开始时间,结束时间,描述\n'];
				const rows = task.timers.map(timer =>
					`${timer.start},${timer.end || ''},${timer.description}\n`
				);
				this.downloadFile(headers.concat(rows).join(''), `task_${task.id}.csv`, 'text/csv');
			}
		},
		handleGlobalExport(format) {
			if (format === 'json') {
				const dataStr = JSON.stringify(this.tasks, null, 2);
				this.downloadFile(dataStr, 'all_tasks.json', 'application/json');
			}
		},
		downloadFile(content, filename, type) {
			const blob = new Blob([content], { type });
			const url = URL.createObjectURL(blob);
			const link = document.createElement('a');
			link.href = url;
			link.download = filename;
			document.body.appendChild(link);
			link.click();
			document.body.removeChild(link);
			URL.revokeObjectURL(url);
		},
		processImportedData(data) {
			// 处理单个任务导入
			if (data.id && data.name && Array.isArray(data.timers)) {
				return [this.convertTimerDates(data)];
			}
			// 处理多个任务导入
			else if (Array.isArray(data) && data.every(task => task.id && task.name && Array.isArray(task.timers))) {
				return data.map(this.convertTimerDates);
			}
			throw new Error('数据格式不正确');
		},
		convertTimerDates(task) {
			const processedTask = { ...task };
			processedTask.timers = task.timers.map(timer => ({
				...timer,
				start: new Date(timer.start),
				end: timer.end ? new Date(timer.end) : null
			}));
			return processedTask;
		},
		handleFileChange(file) {
			const reader = new FileReader();
			reader.onload = (e) => {
				try {
					if (file.raw.name.endsWith('.json')) {
						const rawData = JSON.parse(e.target.result);
						const processedData = this.processImportedData(rawData);
						
						// 检查任务ID是否重复
						processedData.forEach(task => {
							const exists = this.tasks.some(t => t.id === task.id);
							if (exists) {
								task.id = Date.now() + Math.floor(Math.random() * 1000);
							}
						});

						this.tasks = [...this.tasks, ...processedData];
						this.saveToStorage();
						ElMessage.success(`成功导入 ${processedData.length} 个任务`);
					} else if (file.raw.name.endsWith('.csv')) {
						// CSV导入逻辑
						ElMessage.info('CSV导入功能尚未实现');
					}
				} catch (err) {
					console.error('Import error:', err);
					ElMessage.error('导入失败：' + (err.message || '格式错误'));
				}
			};
			reader.readAsText(file.raw);
		},
		deleteTask(task) {
			ElMessageBox.prompt(
				'请输入完整的任务名称以确认删除',
				'警告',
				{
					confirmButtonText: '确认',
					cancelButtonText: '取消',
					inputPattern: new RegExp(`^${task.name}$`),
					inputErrorMessage: '任务名称不正确'
				}
			).then(({ value }) => {
				this.tasks = this.tasks.filter(t => t.id !== task.id);
				this.saveToStorage();
				ElMessage.success('删除成功');
			}).catch(() => {
				// 用户取消或输入错误
			});
		},
		clearAllTasks() {
			ElMessageBox.prompt(
				'请输入"我确定删除时探客 Task Time Tracker所有数据"以确认',
				'警告',
				{
					confirmButtonText: '确认',
					cancelButtonText: '取消',
					inputPattern: /^我确定删除时探客 Task Time Tracker所有数据$/,
					inputErrorMessage: '输入内容不正确'
				}
			).then(({ value }) => {
				this.tasks = [];
				this.saveToStorage();
				ElMessage.success('已清除所有数据');
			}).catch(() => {
				// 用户取消或输入错误
			});
		}
	}
}
</script>

<style>
/* 只保留必要的自定义样式 */
.timeline-content {
	position: relative;
	width: 100%;
}

.time-blocks-container {
	position: relative;
	z-index: 1;
}

.sticky {
	position: sticky;
	left: 0;
	background-color: white;
	z-index: 20;
}

/* 任务说明输入框样式 */
.task-description-input .el-textarea__inner {
  min-height: 60px !important;
  font-size: 12px;
  line-height: 1.4;
  padding: 4px 8px;
}

.task-description-input.el-textarea {
  --el-input-hover-border-color: var(--el-border-color-hover);
}
</style>