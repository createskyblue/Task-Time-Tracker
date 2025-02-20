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
					<el-table :data="tasks" style="width: 100%" @row-click="handleRowClick">
					  <el-table-column label="项目名称" min-width="150">
						<template v-slot="scope">
						  <el-tooltip :content="taskNameTooltip(scope.row)" placement="top">
							<span>
							  <el-input
								v-if="scope.row.isEditing"
								v-model="scope.row.editingName"
								@blur="finishEdit(scope.row)"
								@keyup.enter="finishEdit(scope.row)"
								ref="nameInput"
							  />
							  <span v-else @dblclick="startEdit(scope.row)" class="cursor-pointer hover:text-blue-500">
								{{ scope.row.name }}
							  </span>
							  <el-icon-question class="ml-1 cursor-pointer" @click="showTooltip(scope.row)" />
							</span>
						  </el-tooltip>
						</template>
					  </el-table-column>
					  <el-table-column label="时长统计(事件计数)" min-width="200">
						<template v-slot="scope">
							<template>
							  <span class="cursor-pointer hover:text-blue-500" @click="selectTask(scope.row.id)">
								{{ calculateTotalDuration(scope.row.timers) }}
							  </span>
							</template>
							<div class="text-sm">
							  <div>今日：{{ calculatePeriodDuration(scope.row.timers, 'day') }}</div>
							  <div>本周：{{ calculatePeriodDuration(scope.row.timers, 'week') }}</div>
							  <div>本月：{{ calculatePeriodDuration(scope.row.timers, 'month') }}</div>
							  <div>半年：{{ calculatePeriodDuration(scope.row.timers, 'halfYear') }}</div>
							  <div>今年：{{ calculatePeriodDuration(scope.row.timers, 'year') }}</div>
							  <div class="mt-1 pt-1 border-t">总计：{{ calculateTotalDuration(scope.row.timers) }}</div>
							</div>
						</template>
					  </el-table-column>
					  <el-table-column label="任务说明" min-width="300">
						<template v-slot="scope">
						  <el-input
							v-model="taskDescriptions[scope.row.id]"
							type="textarea"
							:rows="7"
							placeholder="请输入任务说明"
							class="task-description-input"
						  ></el-input>
						</template>
					  </el-table-column>
					  <el-table-column label="计时状态" min-width="100">
						<template v-slot="scope">
						  <span v-if="isTaskRunning(scope.row.id)" class="text-blue-500 font-bold">
							已计时: {{ getRunningTime(scope.row.id) }}
						  </span>
						  <span v-else>未开始计时</span>
						</template>
					  </el-table-column>
					  <el-table-column label="部分操作" min-width="100">
						<template v-slot="scope">
						  <div class="flex flex-wrap">
							<el-button style="margin-left: 10px;" class="mb-2" @click.stop="startTimer(scope.row.id)" type="primary" :disabled="isTaskRunning(scope.row.id)">开始计时</el-button>
							<el-button style="margin-left: 10px;" class="mb-2" @click.stop="stopTimer(scope.row.id)" type="danger" :disabled="!isTaskRunning(scope.row.id)">结束计时</el-button>
						  </div>
						</template>
					  </el-table-column>
					</el-table>
					<!-- Add Task -->
					<div class="flex mt-5 w-2/4">
						<el-input v-model="newTaskName" placeholder="项目名称"></el-input>
						<el-button @click="addTask" type="success">添加项目</el-button>
					</div>
				</div>
				
				<!-- Time Records Display -->
				<div  class="mt-5 p-5 border border-gray-200 rounded-lg">
					<div v-if="!selectedTask">
						<h3 class="text-lg font-medium mb-4">时间线</h3>
						<div class="text-sm text-gray-500 mt-4">
							<span>请先选择一个项目以查看时间线（任务列表）<br>*你知道吗？双击项目名称可以重命名</span>
						</div>
					</div>
					<div v-if="selectedTask">
						<h3 class="text-lg font-medium mb-4">时间线 ( {{ selectedTask.name||"" }} )</h3>
						<!-- <div class="flex items-center gap-4 px-5 mb-4">
							<span>时间单位宽度: {{ (baseUnitWidth * 3600).toFixed(1) }}px/小时</span>
							<el-slider v-model="baseUnitWidth" :min="0.0004" :max="1" :step="0.0001"
								@change="handleZoomChange" class="flex-1" />
						</div> -->
						<!-- 增加一行小字提示：小提示：滚轮缩放时间轴，拖拽滚动时间轴 -->
						<div class="mb-4">
	
							<el-button type="success" @click="addNewEvent">新增记录</el-button>
							<el-button @click.stop="handleExport(selectedTask)" type="success">导出</el-button>
							<el-button @click="deleteTask(selectedTask)" type="danger">删除</el-button>
						</div>
	
						<div class="relative border border-gray-200 rounded-lg overflow-hidden">
							<div @wheel.prevent="handleWheel" @mousedown="startDrag" @mousemove="onDrag" @mouseup="stopDrag"
								@mouseleave="stopDrag" ref="scrollContainer"
								style="scrollbar-width: none;  min-height: 200px;"
								class="relative w-full overflow-x-auto cursor-grab active:cursor-grabbing select-none">
								<div class="timeline-content" :style="{ minWidth: `${24 * 3600 * baseUnitWidth}rem` }">
									<div class="sticky top-0 z-10 flex h-8 items-center bg-gray-50 border-b border-gray-200 pl-24">
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
												class="sticky left-0 z-20 w-24 px-2.5 py-5 text-sm text-gray-700">
												{{ date }}
											</div>
											<div class="relative flex-grow h-full border-l border-gray-200">
												<!-- 原有的时间块显示 -->
												<div v-for="block in blocks" :key="block.id"
													class="absolute h-8 top-1 rounded text-xs text-white px-1 flex items-center justify-center overflow-hidden whitespace-nowrap opacity-100 hover:opacity-90 transition-opacity cursor-pointer"
													:style="{
														left: `${calculateLeftPosition(block.start)}rem`,
														width: `${calculateDuration(block.start, block.end)}rem`,
														backgroundColor: block.color
													}" :title="`${block.description}
	开始：${formatDetailTime(block.start)}
	结束：${formatDetailTime(block.end)}
	持续：${formatDuration(block.start, block.end)}
	相同颜色累计时间：${formatDurationSimple(calculateColorDurations(blocks)[block.color])}`"
													@click="editEvent(block, date)">
													{{ block.displayText }}
												</div>
											</div>
										</div>
									</div>
								</div>
							</div>
						</div>
						<div class="text-sm text-gray-500 mt-4">
							<span>小提示：鼠标滚轮缩放时间轴，拖拽滚动时间轴</span>
						</div>
					</div>
				</div>
				<el-dialog v-model="editDialogVisible" title="编辑时间片" style="width: 70%;">
					<el-form :model="editingEvent">
						<el-form-item label="开始时间">
							<el-time-picker v-model="editingEvent.start" format="HH:mm:ss" />
						</el-form-item>
						<el-form-item label="结束时间">
							<el-time-picker v-model="editingEvent.end" format="HH:mm:ss" />
						</el-form-item>
						<el-form-item label="描述">
							<el-input v-model="editingEvent.description" type="textarea" :rows="7"/>
						</el-form-item>
						<el-form-item label="颜色">
							<el-color-picker v-model="editingEvent.color" />
						</el-form-item>
					</el-form>
					<template #footer>
						<span class="dialog-footer">
							<el-button type="danger" @click="deleteEvent">删除</el-button>
							<el-button @click="editDialogVisible = false">取消</el-button>
							<el-button type="primary" @click="saveEventEdit">保存</el-button>
						</span>
					</template>
				</el-dialog>
			</div>
			<!-- Settings Section -->
			<div class="mt-4 p-4 border border-gray-200 rounded-lg">
				<h3 class="text-lg font-medium mb-4">设置</h3>
				<el-switch
					v-model="autoExport"
					active-text="自动导出存档"
					inactive-text="手动导出存档"
					@change="saveSettings"
				/>
				<!-- Global Actions -->
				<div class="flex gap-2 mt-4">
					<el-upload class="upload-demo" action="" :auto-upload="false" :show-file-list="false"
						accept=".json" :on-change="handleFileChange">
						<el-button type="primary">导入任务</el-button>
					</el-upload>
					<el-button type="success" @click="handleGlobalExport('json')">导出所有数据</el-button>
					<el-button type="danger" @click="clearAllTasks">清除所有数据</el-button>
				</div>
			</div>
			<!-- Footer -->
			<div class="mt-8 text-center text-gray-500 text-sm">
				<p>Created by <a href="https://github.com/createskyblue" target="_blank" class="text-blue-500 hover:underline">createskyblue</a></p>
				<p class="mt-1">
					<a href="https://github.com/createskyblue/Task-Time-Tracker" target="_blank" class="text-blue-500 hover:underline">
						Visit on GitHub
					</a>
					|
					<a href="https://github.com/createskyblue/Task-Time-Tracker/issues" target="_blank" class="text-blue-500 hover:underline">
						反馈问题
					</a>
					|
					<a href="mailto:createskyblue@outlook.com" target="_blank" class="text-blue-500 hover:underline">
						反馈邮箱: createskyblue@outlook.com
					</a>
				</p>
			</div>
		</el-main>
	</el-container>
</template>

<script>
import './styles/index.css'
import { ref, onMounted, onUnmounted, nextTick, computed, watch } from 'vue'
import { ElTable, ElTableColumn, ElButton, ElInput, ElContainer, ElMain, ElMessage, ElMessageBox } from 'element-plus'
import { ArrowDown } from '@element-plus/icons-vue'
import 'element-plus/dist/index.css'

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
			lastClientX: 0,
			scrollLeft: 0,
			timerInterval: null,
			formattedTimeBlocks: {},
			editDialogVisible: false,
			editingEvent: null,
			editingEventDate: null,
			editingEventOriginal: null,
			requireCtrlForZoom: false, // 新增：控制是否需要Ctrl键进行缩放
			autoExport: true, // Add this line
			taskNameTooltipText: '双击可编辑项目名称',
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
		// 确保 scrollContainer 被正确赋值
		this.$nextTick(() => {
			this.scrollContainer = this.$refs.scrollContainer;
		});
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
		},
		taskDescriptions: {
			handler(newDescriptions) {
				// 遍历所有任务，更新正在计时的描述
				this.tasks.forEach(task => {
					const activeTimer = task.timers.find(t => t.end === null);
					if (activeTimer) {
						activeTimer.description = newDescriptions[task.id] || '';
						if (this.selectedTask && this.selectedTask.id === task.id) {
							this.formattedTimeBlocks = this.formatTimeBlocks(task);
						}
					}
				});
				this.saveToStorage();
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
			// 生成随机的 H (0-360), S (50-100), V (50-100)
			let h = Math.floor(Math.random() * 361);
			let s = Math.floor(Math.random() * 51) + 45;
			//75~95
			let v = Math.floor(Math.random() * 21) + 75;
		
			// 将 HSV 转换为 RGB
			let rgb = this.hsvToRgb(h, s, v);
		
			// 返回 RGB 颜色值
			return `rgb(${rgb.r}, ${rgb.g}, ${rgb.b})`;
		},
		
		hsvToRgb(h, s, v) {
			s /= 100;
			v /= 100;
			let k = (n) => (n + h / 60) % 6;
			let f = (n) => v - v * s * Math.max(Math.min(k(n), 4 - k(n), 1), 0);
			return {
				r: Math.round(f(5) * 255),
				g: Math.round(f(3) * 255),
				b: Math.round(f(1) * 255)
			};
		},
		startTimer(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			if (task && !this.isTaskRunning(taskId)) {
				task.timers.push({
					start: new Date(),
					end: null,
					description: this.taskDescriptions[taskId] || '', // 使用现有描述或空字符串
					color: this.getRandomColor() // 直接分配颜色
				});
				this.saveToStorage(); // 保存到本地存储以保持计时信息
			}
		},
		stopTimer(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			const description = this.taskDescriptions[taskId]?.trim();
			if (task) {
				const timer = task.timers.find(t => t.end === null);
				if (timer) {
					const duration = (timer.end - timer.start) / 1000; // Duration in seconds
					if (duration < 10) {
						ElMessage.warning('时间不足10秒，计时已丢弃');
						task.timers = task.timers.filter(t => t !== timer); // Remove the timer
						this.saveToStorage(); // 保存到本地存储以保持计时信息
					} else {
						if (!description) {
							ElMessage.error('请先填写任务说明再结束计时');
							return;
						}
						timer.end = new Date();
						timer.description = description;
						ElMessage.success('计时已结束');
						this.saveToStorage(); // 保存到本地存储以保持颜色信息
						// Only auto-export if enabled
						if (this.autoExport) {
							this.handleGlobalExport('json');
						}
					}
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
					color: timer.color || '#909399', // 使用保存的颜色，如果没有则使用默认灰色
					originalTimer: timer  // 添加对原始timer的引用
				});

				return acc;
			}, {});

			// 对日期进行排序，最新的日期在前面
			const sortedBlocks = {};
			Object.keys(blocks)
				.sort((a, b) => new Date(b) - new Date(a))
				.forEach(key => {
					sortedBlocks[key] = blocks[key];
				});

			return sortedBlocks;
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
			const container = this.$refs.scrollContainer;
			if (container) {
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
			const container = this.$refs.scrollContainer;
			if (!container) return;

			const oldWidth = this.baseUnitWidth;
			const rect = container.getBoundingClientRect();
			const mouseX = container._lastMouseX || rect.left + (rect.width / 2);
			const relativeX = mouseX - rect.left;
			const timeAtCursor = (container.scrollLeft + relativeX) / oldWidth;

			this.baseUnitWidth = newWidth;

			this.$nextTick(() => {
				const newScrollPos = timeAtCursor * newWidth - relativeX;
				container.scrollLeft = Math.max(0, newScrollPos);
			});
		},
		handleWheel(e) {
			if (!this.requireCtrlForZoom || e.ctrlKey || e.metaKey) {
				e.preventDefault();
				const container = this.$refs.scrollContainer;
				if (!container) return;

				// 记录当前鼠标位置用于滑块缩放
				container._lastMouseX = e.clientX;

				const rect = container.getBoundingClientRect();
				const relativeX = e.clientX - rect.left;
				const timeAtCursor = (container.scrollLeft + relativeX) / this.baseUnitWidth;

				// 计算新的缩放级别
				const zoomFactor = e.deltaY > 0 ? 0.9 : 1.1; // 每次缩放10%
				const newWidth = Math.min(1, Math.max(0.0004, this.baseUnitWidth * zoomFactor));

				if (newWidth !== this.baseUnitWidth) {
					this.baseUnitWidth = newWidth;

					// 保持鼠标指向的时间点不变
					this.$nextTick(() => {
						const newScrollPos = timeAtCursor * newWidth - relativeX;
						container.scrollLeft = Math.max(0, newScrollPos);
					});
				}
			}
		},
		startDrag(e) {
			this.isDragging = true;
			this.lastClientX = e.clientX;
			this.scrollLeft = this.$refs.scrollContainer?.scrollLeft || 0;
		},
		onDrag(e) {
			if (!this.isDragging || !this.$refs.scrollContainer) return;
			e.preventDefault();
			const dx = e.clientX - this.lastClientX;
			this.$refs.scrollContainer.scrollLeft = this.scrollLeft - dx;
		},
		stopDrag() {
			this.isDragging = false;
		},
		calculateTotalDuration(timers) {
			if (!timers?.length) return '0小时0分钟 (0)';
			const totalSeconds = timers.reduce((acc, timer) => {
				const end = timer.end ? new Date(timer.end) : new Date();
				return acc + (end - new Date(timer.start)) / 1000;
			}, 0);

			const hours = Math.floor(totalSeconds / 3600);
			const minutes = Math.floor((totalSeconds % 3600) / 60);
			return `${hours}小时${minutes}分钟 (${timers.length})`;
		},
		splitCrossDayTimers() {
            this.tasks.forEach(task => {
                task.timers = task.timers.flatMap(timer => {
                    if (!timer.end) return [timer]; // 如果任务仍在进行中，不拆分

                    const start = new Date(timer.start);
                    const end = new Date(timer.end);
                    const timers = [];

                    let currentStart = start;
                    while (currentStart < end) {
                        const currentEnd = new Date(currentStart);
                        currentEnd.setHours(23, 59, 59, 999);
                        if (currentEnd > end) {
                            currentEnd.setTime(end.getTime());
                        }
                        // 确保不会在0点处多创建一个0秒的记录
                        // 检查毫秒级别的误差
                        if (currentStart.getTime() < currentEnd.getTime()) {
                            const duration = (currentEnd - currentStart) / 1000; // Duration in seconds
                            if (duration > 1) {
                                timers.push({
                                    start: new Date(currentStart),
                                    end: new Date(currentEnd),
                                    description: timer.description,
                                    color: timer.color
                                });
                            }
                        }
                        currentStart.setDate(currentStart.getDate() + 1); // 移动到下一天的零点
                        currentStart.setHours(0, 0, 0, 0);
                    }
                    return timers;
                });
            });
        },

        saveToStorage() {
            this.splitCrossDayTimers(); // 拆分跨天时间记录
            const data = {
                tasks: this.tasks,
                taskDescriptions: this.taskDescriptions,
                developer: 'createskyblue'
            };
            localStorage.setItem(this.STORAGE_KEY, JSON.stringify(data));
        },

        loadFromStorage() {
            const data = localStorage.getItem(this.STORAGE_KEY);
            if (data) {
                const parsed = JSON.parse(data);
                // 转换时间字符串为Date对象
                parsed.tasks.forEach(task => {
                    task.timers.forEach(timer => {
                        timer.start = new Date(timer.start);
                        if (timer.end) timer.end = new Date(timer.end);
                    });
                });
                this.tasks = parsed.tasks;
                this.taskDescriptions = parsed.taskDescriptions || {};
                this.splitCrossDayTimers(); // 拆分跨天时间记录
            }
			// Load settings
			const settings = localStorage.getItem('taskTimeTracker_settings');
			if (settings) {
				const parsed = JSON.parse(settings);
				this.autoExport = parsed.autoExport ?? true;
			}
        },
		handleExport(task) {
		const data = {
			...task,
			developer: 'createskyblue'
		};
		const dataStr = JSON.stringify(data, null, 2);
		const timestamp = new Date().toISOString().replace(/T/, '_').replace(/\..+/, '').replace(/:/g, '-');
		this.downloadFile(dataStr, `task_${task.name}_${timestamp}.json`, 'application/json');
		},
		handleGlobalExport(format) {
		if (format === 'json') {
			const data = {
			tasks: this.tasks,
			taskDescriptions: this.taskDescriptions,
			developer: 'createskyblue'
			};
			const dataStr = JSON.stringify(data, null, 2);
			const timestamp = new Date().toISOString().replace(/T/, '_').replace(/\..+/, '').replace(/:/g, '-');
			this.downloadFile(dataStr, `all_tasks_${timestamp}.json`, 'application/json');
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
			// Handle full data format
			if (data.tasks && Array.isArray(data.tasks)) {
				// Import task descriptions if available
				if (data.taskDescriptions) {
					this.taskDescriptions = {
						...this.taskDescriptions,
						...data.taskDescriptions
					};
				}
				return data.tasks.map(this.convertTimerDates);
			}
			// Handle single task format
			else if (data.id && data.name && Array.isArray(data.timers)) {
				return [this.convertTimerDates(data)];
			}
			// Handle array of tasks format
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
			`请输入完整的项目名称以确认删除：<span class="bg-red-100 text-red-600 px-1 mx-1 rounded-md" ">${task.name}</span>`,
			'警告',
			{
			  confirmButtonText: '确认',
			  cancelButtonText: '取消',
			  inputPattern: new RegExp(`^${task.name}$`),
			  inputErrorMessage: '项目名称不正确',
			  dangerouslyUseHTMLString: true
			}
		  ).then(({ value }) => {
			this.tasks = this.tasks.filter(t => t.id !== task.id);
			this.saveToStorage();
			ElMessage.success('删除成功');
			// 刷新页面
			location.reload();
		  }).catch(() => {
			// 用户取消或输入错误
		  });
		},
		clearAllTasks() {
		  ElMessageBox.prompt(
			`请输入<span class="bg-red-100 text-red-600 px-1 mx-1 rounded-md">我确定删除所有数据</span>以确认`,
			'警告',
			{
			  confirmButtonText: '确认',
			  cancelButtonText: '取消',
			  inputPattern: /^我确定删除所有数据$/,
			  inputErrorMessage: '输入内容不正确',
			  dangerouslyUseHTMLString: true
			}
		  ).then(({ value }) => {
			// 强制下载存档
			this.handleGlobalExport('json');
			// 删除开始
			this.tasks = [];
			this.saveToStorage();
			ElMessage.success('已清除所有数据');
			// 刷新页面
			location.reload();
		  }).catch(() => {
			// 用户取消或输入错误
		  });
		},
		editEvent(block, date) {
			// 保存对原始timer的引用和日期
			this.editingEventOriginal = block.originalTimer;
			this.editingEventDate = date;
			
			// 初始化编辑数据
			this.editingEvent = {
			  start: new Date(block.start),
			  end: new Date(block.end),
			  description: block.description || '',
			  color: block.color || '#409EFF'
			};
			
			this.editDialogVisible = true;
		  },
		
		  saveEventEdit() {
			if (!this.editingEvent || !this.selectedTask) return;

			const newEvent = {
				start: new Date(this.editingEvent.start),
				end: new Date(this.editingEvent.end),
				description: this.editingEvent.description || '未命名时间段',
				color: this.editingEvent.color || '#909399'
			};

			// 验证时间范围
			if (newEvent.end <= newEvent.start) {
				ElMessage.error('结束时间必须晚于开始时间');
				return;
			}

			if (this.editingEventOriginal) {
				// 编辑模式：更新现有记录
				Object.assign(this.editingEventOriginal, newEvent);
			} else {
				// 新增模式：添加新记录
				this.selectedTask.timers.push(newEvent);
			}

			this.formattedTimeBlocks = this.formatTimeBlocks(this.selectedTask);
			this.saveToStorage();

			ElMessage.success(this.editingEventOriginal ? '记录已更新' : '新记录已保存');
			this.editDialogVisible = false;
		  },

		  addNewEvent() {
			if (!this.selectedTask) {
				ElMessage.error('请先打开一个任务的时间线');
				return;
			}

			// 初始化新记录数据
			this.editingEventOriginal = null; // 清除原始记录引用，表示这是新增操作
			this.editingEvent = {
				start: new Date(),
				end: new Date(new Date().getTime() + 60000), // 默认1分钟后
				description: '',
				color: this.getRandomColor()
			};

			this.editDialogVisible = true;
		},
		deleteEvent() {
		if (!this.editingEventOriginal || !this.selectedTask) return;

		this.$confirm('此操作将永久删除该记录, 是否继续?', '提示', {
			confirmButtonText: '确定',
			cancelButtonText: '取消',
			type: 'warning'
		}).then(() => {
			// 从任务的时间记录中删除
			this.selectedTask.timers = this.selectedTask.timers.filter(timer => timer !== this.editingEventOriginal);
			this.formattedTimeBlocks = this.formatTimeBlocks(this.selectedTask);
			this.saveToStorage();
			ElMessage.success('记录已删除');
			this.editDialogVisible = false;
		}).catch(() => {
			ElMessage.info('已取消删除');
		});
		},
		handleRowClick(row, column) {
			// 如果点击的是操作列或正在编辑的输入框，不进行跳转
			this.selectTask(row.id);
		},

		startEdit(row) {
			if (!row.isEditing) {
				row.isEditing = true;
				row.editingName = row.name;
				this.$nextTick(() => {
					const input = this.$refs.nameInput;
					if (input && input.length) {
						input[0].focus();
					}
				});
			}
		},

		finishEdit(row) {
			if (row.editingName && row.editingName.trim()) {
				row.name = row.editingName.trim();
				this.saveToStorage();
			}
			row.isEditing = false;
		},

		watchTaskDescription(taskId) {
			return {
				handler(newDesc) {
					const task = this.tasks.find(t => t.id === taskId);
					if (task) {
						const activeTimer = task.timers.find(t => t.end === null);
						if (activeTimer) {
							activeTimer.description = newDesc;
							this.formattedTimeBlocks = this.formatTimeBlocks(task);
						}
					}
				},
				immediate: true
			};
		},
		// 添加计算每个颜色的累计时间方法
		calculateColorDurations(blocks) {
			const colorDurations = {};
			blocks.forEach(block => {
				const duration = (block.end - block.start) / 1000; // 转换为秒
				colorDurations[block.color] = (colorDurations[block.color] || 0) + duration;
			});
			return colorDurations;
		},

		// 添加简化的时间格式化方法
		formatDurationSimple(seconds) {
			const hours = Math.floor(seconds / 3600);
			const minutes = Math.floor((seconds % 3600) / 60);
			if (hours > 0) {
				return `${hours}h${minutes}m`;
			}
			return `${minutes}m`;
		},
		calculatePeriodDuration(timers, period) {
			if (!timers?.length) return '0小时0分钟';
			
			const now = new Date();
			const startTime = new Date();
			
			// 设置时间段的起始时间
			switch(period) {
			  case 'day':
				startTime.setHours(0, 0, 0, 0);
				break;
			  case 'week':
				startTime.setDate(now.getDate() - now.getDay());
				startTime.setHours(0, 0, 0, 0);
				break;
			  case 'month':
				startTime.setDate(1);
				startTime.setHours(0, 0, 0, 0);
				break;
			  case 'halfYear':
				startTime.setMonth(now.getMonth() - 6);
				startTime.setHours(0, 0, 0, 0);
				break;
			  case 'year':
				startTime.setMonth(0, 1);
				startTime.setHours(0, 0, 0, 0);
				break;
			}
	  
			// 计算指定时间段内的总时长
			const totalSeconds = timers.reduce((acc, timer) => {
			  const start = new Date(timer.start);
			  const end = timer.end ? new Date(timer.end) : now;
			  
			  // 如果时间段完全在范围外，跳过
			  if (end < startTime || start > now) return acc;
			  
			  // 计算有效的开始和结束时间
			  const effectiveStart = start < startTime ? startTime : start;
			  const effectiveEnd = end > now ? now : end;
			  
			  return acc + (effectiveEnd - effectiveStart) / 1000;
			}, 0);
	  
			const hours = Math.floor(totalSeconds / 3600);
			const minutes = Math.floor((totalSeconds % 3600) / 60);
			
			// 获取该时间段内的记录数
			const recordCount = timers.filter(timer => {
			  const start = new Date(timer.start);
			  const end = timer.end ? new Date(timer.end) : now;
			  return !(end < startTime || start > now);
			}).length;
	  
			return `${hours}小时${minutes}分钟 (${recordCount})`;
		  },
		  taskNameTooltip(row) {
			return this.taskNameTooltipText;
		  },
		  showTooltip(row) {
			// 可以在这里添加显示提示的逻辑，如果需要
		  },
		  // Add new method for saving settings
		  saveSettings() {
			localStorage.setItem('taskTimeTracker_settings', JSON.stringify({
			  autoExport: this.autoExport
			}));
		  },
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

.el-dialog__body {
  padding: 20px;
}

.dialog-footer {
  display: flex;
  justify-content: flex-end;
  gap: 10px;
}

.el-table .el-input {
  margin: -5px 0;
}
</style>