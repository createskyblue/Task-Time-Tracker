<template>
	<el-container>
		<!-- 左侧项目列表侧边栏 -->
		<el-aside width="250px" class="border-r fixed-aside border-gray-300">
			<div class="sidebar-header p-4">
				<div class="flex items-center justify-between">
					<h2 class="text-lg font-medium">项目列表</h2>
					<!-- 添加项目按钮 -->
					<el-button type="primary" size="small" @click="showAddTaskDialog = true">
						新建项目
					</el-button>
				</div>
			</div>
			<!-- 项目列表容器 -->
			<div class="project-list-container">
				<div class="project-list px-3" @mouseleave="handleProjectListLeave">
					<div v-for="task in tasks" :key="task.id"
						class="p-3 mb-2 rounded-lg cursor-pointer transition-colors hover:bg-slate-200"
						:class="{ 'bg-slate-100': selectedTask?.id === task.id }" @click="selectTask(task.id)">
						<div class="flex items-center justify-between">
							<div class="flex-1">
								<div class="font-medium" @dblclick.stop="startEditing(task)" v-if="!task.isEditing">
									{{ task.name }}
								</div>
								<el-input v-else v-model="task.editingName" size="small" @blur="finishEditing(task)"
									@keyup.enter="finishEditing(task)" v-focus />
								<div class="text-xs text-gray-500">
									{{ calculateTotalDuration(task.timers) }}
								</div>
							</div>
							<div class="flex items-center space-x-2">
								<div v-if="!isTaskRunning(task.id)"
									class="w-4 h-4 rounded-full bg-slate-400 flex items-center justify-center cursor-pointer ring-2 ring-white"
									@click.stop="startTimer(task.id)">
									<el-icon class="text-white"><video-pause /></el-icon>
								</div>
								<div v-else
									class="w-4 h-4 rounded-full bg-green-500 flex items-center justify-center cursor-pointer ring-2 ring-white"
									@click.stop="stopTimer(task.id)">
									<el-icon class="text-white"><video-pause /></el-icon>
								</div>
							</div>
						</div>
					</div>
				</div>
			</div>
		</el-aside>

		<!-- 主内容区域 -->
		<el-container class="content-wrapper">
			<el-main class="pt-4">
				<!-- 标题区域 -->
				<div class="py-4 text-center text-3xl font-extrabold">
						<span @click="handleTitleClick" class="bg-clip-text text-transparent bg-gradient-to-r from-pink-500 to-violet-500 cursor-pointer">
						时探客 Task Time Tracker
					</span>
					<img class="mx-auto block pt-2" src="https://img.shields.io/badge/version-250305B-blue">
				</div>

				<!-- 当前选中的项目详情 -->
				<div v-if="selectedTask">
					<div class="flex items-center justify-between mb-4">
						<h2 class="text-xl font-bold">
							{{ selectedTask.name }}
							<el-tag size="small" class="ml-2" :type="isTaskRunning(selectedTask.id) ? 'success' : ''">
								{{ isTaskRunning(selectedTask.id) ? '进行中' : '未开始' }}
							</el-tag>
						</h2>
						<div class="pe-3">
							<!-- 添加时薪统计显示 -->
							<span class="text-gray-600 mr-2" v-if="shouldShowSalary && !salaryDisplayDisabled && selectedTask.salaryConfig?.actualSalary > 0">
								实际时薪: {{ calculateHourlyRate('actual').current }}
							</span>
							<span class="text-gray-600" v-if="shouldShowSalary && !salaryDisplayDisabled && selectedTask.salaryConfig?.expectedSalary > 0">
								期望时薪: {{ calculateHourlyRate('expected').current }}
							</span>
							<span class=" text-blue-600 mr-2 ms-3">{{ getRunningTime(selectedTask.id) }}</span>
							<el-button type="primary" @click="startTimer(selectedTask.id)"
								v-show="!isTaskRunning(selectedTask.id)">
								让我们开始
							</el-button>
							<el-button type="danger" @click="stopTimer(selectedTask.id)"
								v-show="isTaskRunning(selectedTask.id)">
								暂停记录
							</el-button>
						</div>
					</div>

					<!-- 标签页切换 -->
					<el-tabs v-model="activeTabName" class="mb-4">
						<el-tab-pane label="任务说明" name="description">
							<div class="mb-2 flex justify-between items-center">
								<span class="text-sm text-gray-500">支持Markdown格式</span>
							</div>
							<div class="flex gap-4 h-96">
								<div class="flex-1 h-full">
									<el-input v-model="taskDescriptions[selectedTask.id]" type="textarea"
										placeholder="请输入任务说明" class="h-full" :input-style="{
											borderRadius: '0.5rem',
											border: '0.5pt solid #e5e7eb !important',
											fontFamily: 'monospace',
											height: '100%',
											padding: '1rem'
										}" />
								</div>
								<div class="flex-1 border rounded-lg p-4 markdown-preview overflow-y-auto h-full border-gray-200 border-2"
									v-html="renderedTaskDescription"></div>
							</div>
						</el-tab-pane>
						<el-tab-pane label="项目备忘录" name="memo">
							<div class="mb-2 flex justify-between items-center">
								<span class="text-sm text-gray-500">支持Markdown格式</span>
							</div>
							<div class="flex gap-4 h-96">
								<div class="flex-1 h-full">
									<el-input v-model="selectedTask.memo" type="textarea"
										placeholder="在这里添加项目备忘录，支持Markdown格式" class="h-full" :input-style="{
											borderRadius: '0.5rem',
											border: '0.5pt solid #e5e7eb !important',
											fontFamily: 'monospace',
											height: '100%',
											padding: '1rem'
										}" />
								</div>
								<div class="flex-1 border rounded-lg p-4 markdown-preview overflow-y-auto h-full border-gray-200 border-2"
									v-html="renderedMemo"></div>
							</div>
						</el-tab-pane>
						<el-tab-pane label="任务清单" name="todo">
							<div class="mb-2 flex justify-between items-center">
								<span class="text-sm text-gray-500">项目任务清单</span>
								<el-button size="small" type="primary" @click="addNewTodo">
									添加任务
								</el-button>
							</div>
							<div class="border rounded-lg p-4 max-h-96 overflow-y-auto border-gray-300">
								<div v-if="!selectedTask.todos || selectedTask.todos.length === 0"
									class="text-center text-gray-500 py-8">
									<div class="text-lg">没有任务</div>
									<div class="mt-2">点击上方按钮添加新任务</div>
								</div>
								<template v-else>
									<div v-for="(todo, index) in selectedTask.todos" :key="todo.id"
										class="flex items-center p-2 border-b last:border-b-0 gap-2 group border-gray-300">
										<el-checkbox v-model="todo.completed" @change="saveToStorage" />
										<div v-if="todo.editing" class="flex-1">
											<el-input v-model="todo.editText" size="small" @blur="finishEditTodo(todo)"
												@keyup.enter="finishEditTodo(todo)" class="w-full"
												placeholder="输入任务内容" />
										</div>
										<div v-else class="flex-1"
											:class="{ 'line-through text-gray-400': todo.completed }"
											@dblclick="startEditTodo(todo)">
											{{ todo.text }}
										</div>
										<div class="opacity-0 group-hover:opacity-100 transition-opacity">
											<el-button size="small" text @click="startEditTodo(todo)" title="编辑">
												<el-icon>
													<Edit />
												</el-icon>
											</el-button>
											<el-button size="small" text type="danger" @click="deleteTodo(index)"
												title="删除">
												<el-icon>
													<Delete />
												</el-icon>
											</el-button>
										</div>
									</div>

									<div class="flex justify-between mt-4 text-sm text-gray-500">
										<div>{{selectedTask.todos.filter(t => t.completed).length}}/{{
											selectedTask.todos.length }} 已完成</div>
										<el-button size="small" type="danger" text @click="clearCompletedTodos"
											v-if="selectedTask.todos.some(t => t.completed)">
											清除已完成任务
										</el-button>
									</div>
								</template>
							</div>
						</el-tab-pane>

						<!-- 新增时间统计标签页 -->
						<el-tab-pane label="时间统计" name="statistics">
							<div class="border rounded-lg bg-white border-gray-300">
								<!-- Content -->
								<div class="p-4">
									<div class="grid grid-cols-2 gap-4">
										<div class="stats-item">
											<div class="text-gray-600">今日 / 昨日</div>
											<div class="flex items-baseline">
												<span class="text-lg">{{ calculatePeriodDuration(selectedTask.timers,
													'day') }}</span>
												<span class="text-sm text-gray-500 mx-2">/</span>
												<span class="text-sm text-gray-600">{{ getPreviousPeriodDuration('day')
													}}</span>
												<span class="ml-2 text-sm" :class="getComparisonClass('day')">
													{{ getComparison('day') }}
												</span>
											</div>
										</div>
										<div class="stats-item">
											<div class="text-gray-600">本周 / 上周</div>
											<div class="flex items-baseline">
												<span class="text-lg">{{ calculatePeriodDuration(selectedTask.timers,
													'week') }}</span>
												<span class="text-sm text-gray-500 mx-2">/</span>
												<span class="text-sm text-gray-600">{{ getPreviousPeriodDuration('week')
													}}</span>
												<span class="ml-2 text-sm" :class="getComparisonClass('week')">
													{{ getComparison('week') }}
												</span>
											</div>
										</div>
										<div class="stats-item">
											<div class="text-gray-600">本月 / 上月</div>
											<div class="flex items-baseline">
												<span class="text-lg">{{ calculatePeriodDuration(selectedTask.timers,
													'month') }}</span>
												<span class="text-sm text-gray-500 mx-2">/</span>
												<span class="text-sm text-gray-600">{{
													getPreviousPeriodDuration('month') }}</span>
												<span class="ml-2 text-sm" :class="getComparisonClass('month')">
													{{ getComparison('month') }}
												</span>
											</div>
										</div>
										<div class="stats-item">
											<div class="text-gray-600">半年 / 上半年</div>
											<div class="flex items-baseline">
												<span class="text-lg">{{ calculatePeriodDuration(selectedTask.timers,
													'halfYear') }}</span>
												<span class="text-sm text-gray-500 mx-2">/</span>
												<span class="text-sm text-gray-600">{{
													getPreviousPeriodDuration('halfYear') }}</span>
												<span class="ml-2 text-sm" :class="getComparisonClass('halfYear')">
													{{ getComparison('halfYear') }}
												</span>
											</div>
										</div>
										<div class="stats-item">
											<div class="text-gray-600">今年 / 去年</div>
											<div class="flex items-baseline">
												<span class="text-lg">{{ calculatePeriodDuration(selectedTask.timers,
													'year') }}</span>
												<span class="text-sm text-gray-500 mx-2">/</span>
												<span class="text-sm text-gray-600">{{ getPreviousPeriodDuration('year')
													}}</span>
												<span class="ml-2 text-sm" :class="getComparisonClass('year')">
													{{ getComparison('year') }}
												</span>
											</div>
										</div>
										<div class="stats-item">
											<div class="text-gray-600">总计</div>
											<div class="text-lg">{{ calculateTotalDuration(selectedTask.timers) }}</div>
										</div>
										<!-- 添加时薪计算显示 -->
										<div class="stats-item" v-if="shouldShowSalary && !salaryDisplayDisabled && selectedTask.salaryConfig?.actualSalary > 0">
											<div class="text-gray-600">时薪计算</div>
											<div class="flex flex-col">
												<div class="flex items-baseline">
													<span class="text-lg">实际: {{ calculateHourlyRate('actual').current }}</span>
													<span class="text-sm text-gray-500 mx-2">/</span>
													<span class="text-sm text-gray-600">上期: {{ calculateHourlyRate('actual').previous }}</span>
												</div>
												<div class="flex items-baseline mt-1" v-if="selectedTask.salaryConfig?.expectedSalary > 0">
													<span class="text-lg">期望: {{ calculateHourlyRate('expected').current }}</span>
													<span class="text-sm text-gray-500 mx-2">/</span>
													<span class="text-sm text-gray-600">上期: {{ calculateHourlyRate('expected').previous }}</span>
												</div>
												<div class="mt-2" v-if="showWorkloadWarning">
													<span class="text-red-500 text-sm font-medium">
														{{ getWorkloadWarningMessage() }}
													</span>
												</div>
											</div>
										</div>
									</div>
								</div>
							</div>

							<div class="flex justify-end mt-4">
								<el-button size="small" type="primary" @click="addNewEvent">新增记录</el-button>
								<el-button size="small" type="success"
									@click="handleExport(selectedTask)">导出项目</el-button>
							</div>
						</el-tab-pane>

						<!-- 新增设置标签页 -->
						<el-tab-pane label="设置" name="settings">
							<div class="border rounded-lg p-4 bg-white border-gray-300">
								<h3 class="text-lg font-medium mb-4">项目设置</h3>
								<div class="space-y-4">
									<el-switch v-model="autoExport" active-text="自动导出存档" inactive-text="手动导出存档"
										@change="saveSettings" />
									<div>
										<el-switch v-model="autoClearDescription" active-text="计时结束后自动清空任务说明"
											inactive-text="计时结束后保留任务说明内容" @change="saveSettings" />
									</div>
									
									<!-- 添加时薪计算器显示控制开关 -->
									<div v-if="shouldShowSalary">
										<el-switch v-model="salaryDisplayDisabled" active-text="隐藏时薪计算信息" 
											inactive-text="显示时薪计算信息" @change="saveSettings" />
									</div>
								</div>

								<!-- 薪资计算器设置 -->
								<div class="mt-4 border-t border-gray-200" v-if="shouldShowSalary && !salaryDisplayDisabled">
									<h3 class="font-medium mb-3 mt-4 ">时薪计算器设置</h3>
									<div class="grid grid-cols-2 gap-4">
										<el-form-item label="实际月薪">
											<el-input-number 
												v-model="selectedTask.salaryConfig.actualSalary" 
												:min="0" 
												:step="500"
												@change="saveToStorage" 
												placeholder="设置为0关闭计算器" />
										</el-form-item>
										<el-form-item label="期望月薪">
											<el-input-number 
												v-model="selectedTask.salaryConfig.expectedSalary" 
												:min="0" 
												:step="500"
												@change="saveToStorage" 
												placeholder="理想收入水平" />
										</el-form-item>
										<el-form-item label="月结日">
											<el-input-number 
												v-model="selectedTask.salaryConfig.paymentDay" 
												:min="1" 
												:max="31"
												@change="saveToStorage" 
												placeholder="每月几号发薪" />
										</el-form-item>
										<el-form-item label="月工作天数">
											<el-input-number 
												v-model="selectedTask.salaryConfig.workingDays" 
												:min="1" 
												:max="31"
												@change="saveToStorage" 
												placeholder="每月预期工作天数" />
										</el-form-item>
										<el-form-item label="日工作小时数">
											<el-input-number 
												v-model="selectedTask.salaryConfig.dailyHours" 
												:min="0.5" 
												:max="24"
												:step="0.5"
												@change="saveToStorage" 
												placeholder="每天预期工作小时数" />
										</el-form-item>
									</div>
								</div>

								<!-- Global Actions -->
								<div class="flex gap-2 mt-6 pt-4 border-t border-gray-200">
									<el-upload class="upload-demo" action="" :auto-upload="false"
										:show-file-list="false" accept=".json" :on-change="handleFileChange">
										<el-button type="primary">导入数据</el-button>
									</el-upload>
									<el-button type="success" @click="handleGlobalExport('json')"
										class="ms-3">导出所有数据</el-button>
									<el-button type="danger" @click="deleteTask(selectedTask)">删除此项目</el-button>
								</div>

								<div class="flex justify-center mt-6 pt-4 border-t border-gray-200">
									<el-button type="danger" @click="clearAllTasks">清除所有项目数据</el-button>
								</div>
								
							</div>
						</el-tab-pane>
					</el-tabs>
				</div>

				<!-- 时间线 -->
				<div ref="timelineRef" class=" p-5 border border-gray-200 rounded-lg shadow-sm overflow-y-auto">
					<div v-if="!selectedTask">
						<h3 class="text-lg font-medium mb-4">欢迎使用</h3>
						<div class="text-sm text-gray-500 mt-4">
							<span>请先选择或者新建“项目”<br><br>*你知道吗？双击项目名称可以重命名</span>
							<span><br>*本项目由人类策划，AI生成，旨在探索AI工作流，不可避免存在代码维护性较差的问题</span>
						</div>
						<el-upload class="upload-demo pt-2" action="" :auto-upload="false" :show-file-list="false"
							accept=".json" :on-change="handleFileChange">
							<el-button type="primary">导入数据</el-button>
						</el-upload>
					</div>
					<div v-if="selectedTask">
						<h3 class="text-lg font-medium mb-4">时间线 ( {{ selectedTask.name || "" }} )</h3>
						<div class="mb-4">
							<el-button type="primary" @click="addNewEvent">新增记录</el-button>
							<el-button @click.stop="handleExport(selectedTask)" type="success">导出</el-button>
							<el-button @click="deleteTask(selectedTask)" type="danger">删除</el-button>
						</div>
						<div class="relative border border-gray-200 rounded-lg overflow-hidden">
							<div @wheel.prevent="handleWheel" @mousedown="startDrag" @mousemove="onDrag"
								@mouseup="stopDrag" @mouseleave="stopDrag" ref="scrollContainer"
								style="scrollbar-width: none;"
								class="relative w-full overflow-x-auto cursor-grab active:cursor-grabbing select-none">
								<div class="timeline-content" :style="{ minWidth: `${24 * 3600 * baseUnitWidth}rem` }">
									<div
										class="sticky top-0 z-10 flex h-8 items-center bg-gray-50 border-b border-gray-200 pl-29">
										<div v-for="mark in timeScaleMarks" :key="mark.time"
											:style="{ width: `${mark.width}rem` }"
											class="flex-shrink-0 text-xs text-gray-600 text-center border-r border-gray-200 relative ">
											{{ mark.label }}
											<div class="absolute top-0 bottom-0 left-0 w-[0.1px] bg-gray-300"></div>
											<!-- 增加垂直线 -->
										</div>
									</div>

									<div class="time-blocks-container">
										<div v-for="(blocks, date) in formattedTimeBlocks" :key="date"
											class="flex my-2.5 items-center border-b border-gray-300">
											<div class="sticky left-0 py-1 ps-2 text-sm text-gray-700 rounded-md -mt-2 overflow-visible"
												style="width:120px; background-color: white;">
												<div class="font-medium overflow-visible whitespace-nowrap">
													{{ date }} <span class="text-xs text-gray-400">{{ getWeekday(date)
														}}</span>
												</div>
												<div class="text-xs text-gray-500 overflow-visible whitespace-nowrap">{{
													calculateDayTotal(blocks) }}</div>
											</div>
											<!-- Container with relative positioning -->
											<div class="relative w-full">
												<!-- Time scale marks layer -->
												<div class="absolute inset-0 flex  -mt-7">
													<div v-for="mark in timeScaleMarks" :key="mark.time2"
														:style="{ width: `${mark.width}rem` }"
														class="flex-shrink-0 text-center relative">
														<div
															class="absolute top-0 bottom-0 left-0 w-[0.1px] bg-gray-300 h-12">
														</div>
													</div>
												</div>

												<!-- Blocks layer -->
												<div class="absolute inset-0 -mt-6">
													<div class="relative h-full w-full">
														<div v-for="block in blocks" :key="block.id"
															class="absolute h-8 top-1 rounded text-xs text-white px-1 flex items-center justify-center overflow-hidden whitespace-nowrap opacity-100 hover:opacity-90 transition-opacity cursor-pointer"
															:style="{
																left: `${calculateLeftPosition(block.start)}rem`,
																width: `${calculateDuration(block.start, block.end)}rem`,
																backgroundColor: block.color
															}" :title="`
开始：${formatDetailTime(block.start)}
结束：${formatDetailTime(block.end)}
持续：${formatDuration(block.start, block.end)}
相同颜色累计时间：${formatDurationSimple(calculateColorDurations(blocks)[block.color])}
${block.description}`" @click="editEvent(block, date)">
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
						<div class="text-sm text-gray-500 mt-4">
							<span>小提示：鼠标滚轮缩放时间轴，拖拽滚动时间轴</span>
						</div>
					</div>
				</div>

				<!-- 页脚 -->
				<div class="mt-8 text-center text-gray-500 text-sm">
					<p>Created by <a href="https://github.com/createskyblue" target="_blank"
							class="text-blue-500 hover:underline">createskyblue</a></p>
					<p class="mt-1">
						<a href="https://github.com/createskyblue/Task-Time-Tracker" target="_blank"
							class="text-blue-500 hover:underline">
							Visit on GitHub
						</a>
						|
						<a href="https://github.com/createskyblue/Task-Time-Tracker/issues" target="_blank"
							class="text-blue-500 hover:underline">
							反馈问题
						</a>
						|
						<a href="mailto:createskyblue@outlook.com" target="_blank"
							class="text-blue-500 hover:underline">
							反馈邮箱: createskyblue@outlook.com
						</a>
					</p>
					<p class="mt-1">
						您的数据存储在本地浏览器缓存中，除非您清除浏览器缓存，否则数据不会丢失。
					</p>
				</div>
			</el-main>
		</el-container>

		<!-- 添加项目对话框 -->
		<el-dialog v-model="showAddTaskDialog" title="新建项目" width="400px">
			<el-form @submit.native.prevent="addTask">
				<el-form-item label="项目名称">
					<el-input v-model="newTaskName" placeholder="请输入项目名称" @keyup.enter="addTask" />
				</el-form-item>
			</el-form>
			<template #footer>
				<span class="dialog-footer">
					<el-button @click="showAddTaskDialog = false">取消</el-button>
					<el-button type="primary" @click="addTask">确定</el-button>
				</span>
			</template>
		</el-dialog>

		<!-- 其他对话框 -->
		<el-dialog v-model="editDialogVisible" title="编辑时间片" style="width: 70%;">
			<el-form :model="editingEvent">
				<el-form-item label="开始时间">
					<el-date-picker v-model="editingEvent.start" type="datetime" format="YYYY-MM-DD HH:mm:ss"
						placeholder="选择开始日期时间" />
				</el-form-item>
				<el-form-item label="结束时间">
					<el-date-picker v-model="editingEvent.end" type="datetime" format="YYYY-MM-DD HH:mm:ss"
						placeholder="选择结束日期时间" />
				</el-form-item>
				<el-form-item label="描述">
					<el-input v-model="editingEvent.description" type="textarea" :rows="7" />
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
	</el-container>
</template>

<script>
import './styles/index.css'
import { ref, onMounted, onUnmounted, nextTick, computed, watch } from 'vue'
import { ElTable, ElTableColumn, ElButton, ElInput, ElContainer, ElMain, ElMessage, ElMessageBox } from 'element-plus'
import { ArrowDown, Edit, Delete } from '@element-plus/icons-vue'
import 'element-plus/dist/index.css'
import { marked } from 'marked'

export default {
	components: {
		ElTable,
		ElTableColumn,
		ElButton,
		ElInput,
		ElContainer,
		ElMain,
		ArrowDown,
		Edit,
		Delete
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
			saveMshShowTimer: null,
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
			autoExport: true,
			autoClearDescription: false, // 计时结束后是否自动清空任务说明
			taskNameTooltipText: '双击可编辑项目名称',
			showAddTaskDialog: false,
			editingTask: null,
			lastClientY: 0,       // 添加Y轴位置记录
			scrollTop: 0,         // 添加垂直滚动位置
			activeTabName: 'description', // 当前激活的标签页
			taskScrollPositions: {}, // 存储每个项目的滚动位置
			// 多标签页检测
			tabId: null,
			lastStorageUpdate: null,
			// 默认薪资配置
			defaultSalaryConfig: {
				actualSalary: 0,         // 实际月薪
				expectedSalary: 0,       // 期望月薪
				paymentDay: 1,           // 月结日（每月几号发工资）
				workingDays: 22,         // 每月工作天数
				dailyHours: 8,           // 每天工作小时数
			},
			// 隐藏功能的相关状态
			titleClickCount: 0,
			titleClickTimer: null,
			titleClickTimestamp: null,
			salaryFeatureUnlocked: false,
			salaryDisplayDisabled: false, // 添加显示控制标志
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
		},
		// 添加计算属性来渲染markdown
		renderedMarkdown() {
			if (!this.selectedTask || !this.taskMemos[this.selectedTask.id]) {
				return '';
			}
			try {
				return marked(this.taskMemos[this.selectedTask.id] || '');
			} catch (error) {
				console.error('Markdown parsing error:', error);
				return '<p>错误：Markdown解析失败</p>';
			}
		},
		// 渲染项目备忘录的markdown
		renderedMemo() {
			if (!this.selectedTask || !this.selectedTask.memo) {
				return '';
			}
			try {
				return marked(this.selectedTask.memo || '');
			} catch (error) {
				console.error('Markdown parsing error:', error);
				return '<p>错误：Markdown解析失败</p>';
			}
		},

		// 渲染任务说明的markdown
		renderedTaskDescription() {
			if (!this.selectedTask || !this.taskDescriptions[this.selectedTask.id]) {
				return '';
			}
			try {
				return marked(this.taskDescriptions[this.selectedTask.id] || '');
			} catch (error) {
				console.error('Markdown parsing error:', error);
				return '<p>错误：Markdown解析失败</p>';
			}
		},
		// 判断是否需要显示工作量超标警告
		showWorkloadWarning() {
			if (!this.shouldShowSalary || !this.selectedTask?.salaryConfig?.actualSalary) return false;

			const currentPeriodStats = this.calculatePeriodWorkload('month');
			return currentPeriodStats.daysWorked > 0 &&
				(currentPeriodStats.averageHoursPerDay > this.selectedTask.salaryConfig.dailyHours * 1.2);
		},
		// 判断是否应该显示薪资功能
		shouldShowSalary() {
			return this.salaryFeatureUnlocked;
		},
	},
	mounted() {
		// 生成唯一的标签页ID
		this.tabId = Date.now().toString() + Math.random().toString().substring(2);

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

		// 添加全局快捷键监听
		window.addEventListener('keydown', this.handleKeydown);

		// 监听storage事件，用于检测多标签页
		window.addEventListener('storage', this.handleStorageChange);

		// 检查本地存储中的解锁状态
		this.checkSalaryFeatureUnlocked();
	},
	beforeUnmount() {
		if (this.timerInterval) {
			clearInterval(this.timerInterval);
		}
		// 移除全局快捷键监听
		window.removeEventListener('keydown', this.handleKeydown);

		// 移除storage事件监听
		window.removeEventListener('storage', this.handleStorageChange);

		// 保存当前项目的滚动位置
		if (this.selectedTask) {
			this.saveScrollPosition(this.selectedTask.id);
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
		},
		taskMemos: {
			handler(newMemos) {
				this.saveToStorage();
			},
			deep: true
		},

		selectedTask(newTask, oldTask) {
			if (oldTask) {
				// 保存旧项目的滚动位置
				this.saveScrollPosition(oldTask.id);
			}

			if (newTask) {
				// 恢复新项目的滚动位置
				this.$nextTick(() => {
					this.restoreScrollPosition(newTask.id);
				});
			}
		}
	},
	methods: {
		addTask() {
			if (this.newTaskName.trim()) {
				const newTask = {
					id: Date.now(),
					name: this.newTaskName,
					timers: [],
					lastModified: Date.now(), // Add lastModified timestamp
					memo: '', // 给新任务添加空备忘录
					todos: [], // 给新任务添加空待办事项列表
					salaryConfig: { ...this.defaultSalaryConfig } // 添加默认薪资配置
				};
				this.tasks.unshift(newTask); // Add to beginning of array
				this.newTaskName = '';
				this.showAddTaskDialog = false;
				this.selectTask(newTask.id);
				this.saveToStorage();
			}
		},
		updateTaskModificationTime(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			if (task) {
				task.lastModified = Date.now();
				// Re-sort tasks array
				this.tasks.sort((a, b) => (b.lastModified || 0) - (a.lastModified || 0));
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
				this.updateTaskModificationTime(taskId);
				this.saveToStorage(); // 保存到本地存储以保持计时信息
			}
		},
		stopTimer(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			const description = this.taskDescriptions[taskId]?.trim();
			if (task) {
				const timer = task.timers.find(t => t.end === null);
				if (timer) {
					const duration = (new Date() - timer.start) / 1000; // Duration in seconds
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
						this.updateTaskModificationTime(taskId);
						this.saveToStorage(); // 保存到本地存储以保持颜色信息

						// 如果启用了自动清空备注栏，清空当前任务的描述
						if (this.autoClearDescription) {
							this.taskDescriptions[taskId] = '';
						}

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
				const displayText = timer.description || '未命名时间段';

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
			// 不再在这里调用排序逻辑
			this.selectedTask = task;
			this.formattedTimeBlocks = this.formatTimeBlocks(task);
		},

		selectTask(taskId) {
			const task = this.tasks.find(t => t.id === taskId);
			if (task) {
				// 只显示任务详情，不进行排序
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
			this.lastClientY = e.clientY;                    // 记录Y轴位置
			this.scrollLeft = this.$refs.scrollContainer?.scrollLeft || 0;
			this.scrollTop = window.scrollY;                 // 记录页面垂直滚动位置
		},

		onDrag(e) {
			if (!this.isDragging) return;
			e.preventDefault();

			// 水平滚动（时间轴）
			const dX = e.clientX - this.lastClientX;
			if (this.$refs.scrollContainer) {
				this.$refs.scrollContainer.scrollLeft = this.scrollLeft - dX;
			}

			// 垂直滚动（整个页面）
			const dY = e.clientY - this.lastClientY;
			window.scrollTo(0, this.scrollTop - dY);
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
				taskScrollPositions: this.taskScrollPositions,
				developer: 'createskyblue',
				updatedAt: Date.now(),
				updatedBy: this.tabId
			};

			// 记录最后一次更新时间，用于多标签页检测
			this.lastStorageUpdate = Date.now();

			localStorage.setItem(this.STORAGE_KEY, JSON.stringify(data));

			// Clear the previous timeout if it exists
			if (this.saveMshShowTimer) {
				clearTimeout(this.saveMshShowTimer);
			}
			// Set a new timeout to show the save message after 1 second of inactivity
			this.saveMshShowTimer = setTimeout(() => {
				ElMessage.success('保存成功');
			}, 1500);
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

					// 确保每个任务都有memo属性
					if (!task.memo) {
						task.memo = '';
					}
					// 确保每个任务都有todos属性
					if (!task.todos) {
						task.todos = [];
					}
					// 确保每个任务都有salaryConfig属性
					if (!task.salaryConfig) {
						task.salaryConfig = { ...this.defaultSalaryConfig };
					}
				});
				this.tasks = parsed.tasks;
				this.taskDescriptions = parsed.taskDescriptions || {};
				this.taskScrollPositions = parsed.taskScrollPositions || {}; // 加载滚动位置
				this.splitCrossDayTimers(); // 拆分跨天时间记录
			}
			// Load settings
			const settings = localStorage.getItem('taskTimeTracker_settings');
			if (settings) {
				const parsed = JSON.parse(settings);
				this.autoExport = parsed.autoExport ?? true;
				this.autoClearDescription = parsed.autoClearDescription ?? false;
				this.salaryDisplayDisabled = parsed.salaryDisplayDisabled ?? false; // 读取显示控制选项
			}
			
			// 检查薪资功能是否已解锁
			this.checkSalaryFeatureUnlocked();
		},
		handleExport(task) {
			const data = {
				...task,
				developer: 'createskyblue'
			};
			const dataStr = JSON.stringify(data, null, 2);
			const timestamp = new Date().toISOString().replace(/T/, '_').replace(/\..+/, '').replace(/:/g, '-');
			this.downloadFile(dataStr, `时探客_TaskTimeTracker_Task_${task.name}_${timestamp}.json`, 'application/json');
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
				this.downloadFile(dataStr, `时探客_TaskTimeTracker_AllTasks_${timestamp}.json`, 'application/json');
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

				// 确保每个导入的任务都有memo属性
				data.tasks.forEach(task => {
					if (!task.memo) task.memo = '';
					if (!task.todos) task.todos = [];
					if (!task.salaryConfig) task.salaryConfig = { ...this.defaultSalaryConfig };
				});

				return data.tasks.map(this.convertTimerDates);
			}
			// Handle single task format
			else if (data.id && data.name && Array.isArray(data.timers)) {
				if (!data.memo) data.memo = '';
				if (!data.todos) data.todos = [];
				if (!data.salaryConfig) data.salaryConfig = { ...this.defaultSalaryConfig };
				return [this.convertTimerDates(data)];
			}
			// Handle array of tasks format
			else if (Array.isArray(data) && data.every(task => task.id && task.name && Array.isArray(task.timers))) {
				data.forEach(task => {
					if (!task.memo) task.memo = '';
					if (!task.todos) task.todos = [];
					if (!task.salaryConfig) task.salaryConfig = { ...this.defaultSalaryConfig };
				});
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

						// Add lastModified to imported tasks if not present
						processedData.forEach(task => {
							if (!task.lastModified) {
								task.lastModified = Date.now();
							}
							const exists = this.tasks.some(t => t.id === task.id);
							if (exists) {
								task.id = Date.now() + Math.floor(Math.random() * 1000);
							}
						});

						this.tasks = [...this.tasks, ...processedData];
						// Sort tasks by lastModified
						this.tasks.sort((a, b) => (b.lastModified || 0) - (a.lastModified || 0));
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
				// 根据自动导出设置决定是否先导出数据
				if (this.autoExport) {
					this.handleExport(task);
					ElMessage.success('已自动导出项目数据');
				}

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

			const now = new Date();
			// 初始化新记录数据
			this.editingEventOriginal = null; // 清除原始记录引用，表示这是新增操作
			this.editingEvent = {
				start: now,
				end: new Date(now.getTime() + 60000), // 默认1分钟后
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
			// 如果点击的是操作列或正在编辑的输入框，则不进行跳转
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
				this.updateTaskModificationTime(row.id);
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
			switch (period) {
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
				autoExport: this.autoExport,
				autoClearDescription: this.autoClearDescription,
				salaryFeatureUnlocked: this.salaryFeatureUnlocked,
				salaryDisplayDisabled: this.salaryDisplayDisabled // 保存显示控制选项
			}));
		},
		calculateDayTotal(blocks) {
			const totalSeconds = blocks.reduce((acc, block) => {
				const duration = (block.end - block.start) / 1000;
				return acc + duration;
			}, 0);

			const hours = Math.floor(totalSeconds / 3600);
			const minutes = Math.floor((totalSeconds % 3600) / 60);
			return `${hours}h${minutes}m (${blocks.length})`;
		},
		startEditing(task) {
			// Cancel any other editing
			this.tasks.forEach(t => {
				if (t.id !== task.id) {
					t.isEditing = false;
				}
			});

			task.isEditing = true;
			task.editingName = task.name;
		},

		finishEditing(task) {
			if (task.editingName && task.editingName.trim()) {
				task.name = task.editingName.trim();
				this.updateTaskModificationTime(task.id);
				this.saveToStorage();
			}
			task.isEditing = false;
		},
		handleKeydown(event) {
			// 检测 Ctrl+S 或 Command+S (Mac)
			if ((event.ctrlKey || event.metaKey) && event.key === 's') {
				event.preventDefault(); // 阻止默认的保存行为
				ElMessage.info('保存操作中...');
				this.saveToStorage();

				if (this.autoExport) {
					this.handleGlobalExport('json');
					ElMessage.success('已导出所有数据');
				}
			}
		},
		handleProjectListLeave() {
			if (this.selectedTask) {
				// 只在鼠标离开侧边栏时进行排序
				this.tasks = [
					this.selectedTask,
					...this.tasks.filter(t => t.id !== this.selectedTask.id)
				];
				this.saveToStorage();
			}
		},
		calculatePeriodSeconds(timers, startTime, endTime) {
			if (!timers?.length) return 0;
			return timers.reduce((acc, timer) => {
				const start = new Date(timer.start);
				const end = timer.end ? new Date(timer.end) : new Date();

				if (end < startTime || start > endTime) return acc;

				const effectiveStart = start < startTime ? startTime : start;
				const effectiveEnd = end > endTime ? endTime : end;

				return acc + (effectiveEnd - effectiveStart) / 1000;
			}, 0);
		},

		getComparison(period) {
			if (!this.selectedTask?.timers?.length) return '';

			const now = new Date();
			let currentStart, currentEnd, previousStart, previousEnd;

			switch (period) {
				case 'day':
					currentStart = new Date(now.setHours(0, 0, 0, 0));
					currentEnd = new Date();
					previousStart = new Date(currentStart);
					previousStart.setDate(previousStart.getDate() - 1);
					previousEnd = new Date(currentStart);
					break;
				case 'week':
					currentStart = new Date(now.setDate(now.getDate() - now.getDay()));
					currentEnd = new Date();
					previousStart = new Date(currentStart);
					previousStart.setDate(previousStart.getDate() - 7);
					previousEnd = new Date(currentStart);
					break;
				case 'month':
					currentStart = new Date(now.setDate(1));
					currentEnd = new Date();
					previousStart = new Date(currentStart);
					previousStart.setMonth(previousStart.getMonth() - 1);
					previousEnd = new Date(currentStart);
					break;
				case 'halfYear':
					currentStart = new Date(now.setMonth(now.getMonth() - 6));
					currentEnd = new Date();
					previousStart = new Date(currentStart);
					previousStart.setMonth(previousStart.getMonth() - 6);
					previousEnd = new Date(currentStart);
					break;
				case 'year':
					currentStart = new Date(now.setMonth(0, 1));
					currentEnd = new Date();
					previousStart = new Date(currentStart);
					previousStart.setFullYear(previousStart.getFullYear() - 1);
					previousEnd = new Date(currentStart);
					break;
			}

			const currentSeconds = this.calculatePeriodSeconds(this.selectedTask.timers, currentStart, currentEnd);
			const previousSeconds = this.calculatePeriodSeconds(this.selectedTask.timers, previousStart, previousEnd);

			if (previousSeconds === 0) return '';

			const percentage = ((currentSeconds - previousSeconds) / previousSeconds * 100).toFixed(1);
			return percentage > 0 ? `+${percentage}%` : `${percentage}%`;
		},

		getComparisonClass(period) {
			const comparison = this.getComparison(period);
			if (!comparison) return '';
			return comparison.startsWith('+') ? 'text-green-500' : 'text-red-500';
		},
		getPreviousPeriodDuration(period) {
			if (!this.selectedTask?.timers?.length) return '0小时0分钟';

			const now = new Date();
			let startTime, endTime;

			switch (period) {
				case 'day':
					startTime = new Date(now);
					startTime.setDate(startTime.getDate() - 1);
					startTime.setHours(0, 0, 0, 0);
					endTime = new Date(startTime);
					endTime.setHours(23, 59, 59, 999);
					break;
				case 'week':
					startTime = new Date(now);
					startTime.setDate(startTime.getDate() - startTime.getDay() - 7);
					startTime.setHours(0, 0, 0, 0);
					endTime = new Date(startTime);
					endTime.setDate(endTime.getDate() + 6);
					endTime.setHours(23, 59, 59, 999);
					break;
				case 'month':
					startTime = new Date(now);
					startTime.setMonth(startTime.getMonth() - 1);
					startTime.setDate(1);
					startTime.setHours(0, 0, 0, 0);
					endTime = new Date(startTime.getFullYear(), startTime.getMonth() + 1, 0, 23, 59, 59, 999);
					break;
				case 'halfYear':
					startTime = new Date(now);
					startTime.setMonth(startTime.getMonth() - 12);
					startTime.setHours(0, 0, 0, 0);
					endTime = new Date(startTime);
					endTime.setMonth(endTime.getMonth() + 6);
					endTime.setHours(23, 59, 59, 999);
					break;
				case 'year':
					startTime = new Date(now);
					startTime.setFullYear(startTime.getFullYear() - 1);
					startTime.setMonth(0, 1);
					startTime.setHours(0, 0, 0, 0);
					endTime = new Date(startTime);
					endTime.setMonth(11, 31);
					endTime.setHours(23, 59, 59, 999);
					break;
			}

			const totalSeconds = this.calculatePeriodSeconds(this.selectedTask.timers, startTime, endTime);
			const hours = Math.floor(totalSeconds / 3600);
			const minutes = Math.floor((totalSeconds % 3600) / 60);

			const recordCount = this.selectedTask.timers.filter(timer => {
				const start = new Date(timer.start);
				const end = timer.end ? new Date(timer.end) : now;
				return !(end < startTime || start > endTime);
			}).length;

			return `${hours}小时${minutes}分钟 (${recordCount})`;
		},
		getWeekday(dateString) {
			try {
				const date = new Date(dateString);
				const weekdays = ['周日', '周一', '周二', '周三', '周四', '周五', '周六'];
				return weekdays[date.getDay()];
			} catch (e) {
				return '';
			}
		},
		saveScrollPosition(taskId) {
			if (this.$refs.timelineRef) {
				this.taskScrollPositions[taskId] = {
					timelineScroll: this.$refs.timelineRef.scrollTop,
					windowScroll: window.scrollY
				};
			}
		},
		// 多标签页检测
		handleStorageChange(event) {
			if (event.key === this.STORAGE_KEY) {
				// 检查数据是否是由当前标签页更新的
				if (this.lastStorageUpdate && (Date.now() - this.lastStorageUpdate < 100)) {
					// 如果是当前标签页更新的，忽略
					return;
				}

				// 如果是其他标签页更新的，重新加载数据
				ElMessage.info('检测到数据在其他标签页中被修改，正在同步...');

				// 如果当前没有选中任务，直接重载
				const selectedTaskId = this.selectedTask?.id;

				this.loadFromStorage();

				// 如果之前有选中的任务，尝试重新选择
				if (selectedTaskId) {
					const task = this.tasks.find(t => t.id === selectedTaskId);
					if (task) {
						this.selectedTask = task;
						this.formattedTimeBlocks = this.formatTimeBlocks(task);
					}
				}
			}
		},

		// 待办事项相关方法
		addNewTodo() {
			if (!this.selectedTask) return;

			// 确保selectedTask有todos属性
			if (!this.selectedTask.todos) {
				this.$set(this.selectedTask, 'todos', []);
			}

			const newTodo = {
				id: Date.now(),
				text: '',
				completed: false,
				editing: true,
				editText: '',
				createdAt: new Date(),
			};

			this.selectedTask.todos.unshift(newTodo);
			this.$nextTick(() => {
				// 聚焦到新添加的Todo的输入框
				const input = document.querySelector('.el-input__inner');
				if (input) input.focus();
			});
		},

		startEditTodo(todo) {
			todo.editing = true;
			todo.editText = todo.text;
			this.$nextTick(() => {
				// 聚焦到输入框
				const input = document.querySelector('.el-input__inner');
				if (input) input.focus();
			});
		},

		finishEditTodo(todo) {
			if (todo.editText.trim()) {
				todo.text = todo.editText.trim();
			}
			todo.editing = false;
			this.saveToStorage();
		},

		deleteTodo(index) {
			if (!this.selectedTask || !this.selectedTask.todos) return;

			this.selectedTask.todos.splice(index, 1);
			this.saveToStorage();
		},

		clearCompletedTodos() {
			if (!this.selectedTask || !this.selectedTask.todos) return;

			this.selectedTask.todos = this.selectedTask.todos.filter(todo => !todo.completed);
			this.saveToStorage();
		},
		// 计算当前周期的工作负荷
		calculatePeriodWorkload(period) {
			if (!this.selectedTask?.timers?.length) {
				return { totalHours: 0, daysWorked: 0, averageHoursPerDay: 0 };
			}

			const now = new Date();
			let startDate;

			// 根据月结日计算当前计薪周期的起始日期
			if (period === 'month') {
				const paymentDay = this.selectedTask.salaryConfig.paymentDay || 1;
				startDate = new Date(now.getFullYear(), now.getMonth(), paymentDay);

				// 如果当前日期小于月结日，则计薪周期从上个月的月结日开始
				if (now.getDate() < paymentDay) {
					startDate.setMonth(startDate.getMonth() - 1);
				}
			} else {
				// 其他周期的处理逻辑（如果需要）
				startDate = new Date(now);
				startDate.setDate(1);
				startDate.setHours(0, 0, 0, 0);
			}

			// 计算在这个���期内工作的天数和总小时数
			const workDays = new Set();
			let totalHours = 0;

			this.selectedTask.timers.forEach(timer => {
				const start = new Date(timer.start);
				const end = timer.end ? new Date(timer.end) : now;

				// 如果时间段不在当前计薪周期内，忽略
				if (end < startDate || start > now) return;

				// 计算有效的开始和结束时间
				const effectiveStart = start < startDate ? startDate : start;
				const effectiveEnd = end > now ? now : end;

				// 计算工作小时数
				const hours = (effectiveEnd - effectiveStart) / 3600000; // 毫秒转小时
				totalHours += hours;

				// 记录工作日
				const dateKey = start.toISOString().split('T')[0];
				workDays.add(dateKey);
			});

			const daysWorked = workDays.size;
			const averageHoursPerDay = daysWorked > 0 ? totalHours / daysWorked : 0;

			return { totalHours, daysWorked, averageHoursPerDay };
		},
		// 计算小时工资率
		calculateHourlyRate(type) {
			if (!this.selectedTask?.salaryConfig) return { current: '未设置', previous: '未设置' };

			const config = this.selectedTask.salaryConfig;
			const salary = type === 'actual' ? config.actualSalary : config.expectedSalary;

			if (!salary) return { current: '未设置', previous: '未设置' };

			// 计算当前周期的工作负荷
			const now = new Date();
			const currentPeriod = this.calculatePeriodWorkload('month');

			// 计算上一个周期的工作负荷
			const previousMonth = new Date(now);
			previousMonth.setMonth(previousMonth.getMonth() - 1);

			// 理想状况下的小时工资
			const idealMonthlyHours = config.workingDays * config.dailyHours;
			const idealHourlyRate = salary / idealMonthlyHours;

			// 如果当前周期没有工作记录，返回理想小时工资
			if (currentPeriod.totalHours === 0) {
				return {
					current: `¥${idealHourlyRate.toFixed(2)}/小时 (理想)`,
					previous: `¥${idealHourlyRate.toFixed(2)}/小时 (理想)`
				};
			}

			// 实际小时工资
			const actualHourlyRate = salary / currentPeriod.totalHours;

			// 格式化返回结果
			let currentRateText = `¥${actualHourlyRate.toFixed(2)}/小时`;
			if (actualHourlyRate < idealHourlyRate * 0.8) {
				currentRateText = `<span class="text-red-500">${currentRateText}</span>`;
			}

			return {
				current: currentRateText,
				previous: `¥${idealHourlyRate.toFixed(2)}/小时 (理想)`
			};
		},
		// 获取工作量警告信息
		getWorkloadWarningMessage() {
			const config = this.selectedTask.salaryConfig;
			const stats = this.calculatePeriodWorkload('month');

			const overworkPercent = Math.round((stats.averageHoursPerDay / config.dailyHours - 1) * 100);

			let message = `日均工作时间超出预期${overworkPercent}%`;

			// 计算实际时薪与期望时薪的差距
			if (config.expectedSalary > 0) {
				const actualHourlyRate = config.actualSalary / stats.totalHours;
				const expectedHourlyRate = config.expectedSalary / (config.workingDays * config.dailyHours);
				const rateDiff = Math.round((1 - actualHourlyRate / expectedHourlyRate) * 100);

				if (rateDiff > 0) {
					message += `，实际时薪低于期望${rateDiff}%`;
				}
			}

			return message;
		},
		// 处理标题点击事件
		handleTitleClick() {
			const now = Date.now();
			
			// 如果功能已解锁，不需要计数
			if (this.salaryFeatureUnlocked) return;

			// 首次点击或超过1分钟，重置计数
			if (!this.titleClickTimestamp || (now - this.titleClickTimestamp) > 60000) {
				this.titleClickCount = 1;
				this.titleClickTimestamp = now;
				
				// 设置1分钟后重置计数的定时器
				clearTimeout(this.titleClickTimer);
				this.titleClickTimer = setTimeout(() => {
					this.titleClickCount = 0;
					this.titleClickTimestamp = null;
				}, 60000);
			} else {
				// 在1分钟内的后续点击
				this.titleClickCount++;
				
				// 检查是否达到解锁条件
				if (this.titleClickCount >= 60) {
					this.unlockSalaryFeature();
				}
			}
		},

		// 解锁薪资功能
		unlockSalaryFeature() {
			this.salaryFeatureUnlocked = true;
			// 存储解锁状态到本地存储
			localStorage.setItem('taskTimeTracker_salaryUnlocked', 'true');
			ElMessage.success({
				message: '🎉 恭喜！时薪计算功能已解锁，你可以随时在设置中开启或关闭显示',
				duration: 5000
			});
		},

		// 检查薪资功能是否已解锁
		checkSalaryFeatureUnlocked() {
			const unlocked = localStorage.getItem('taskTimeTracker_salaryUnlocked');
			this.salaryFeatureUnlocked = unlocked === 'true';
		},
	},

	// 恢复项目的滚动位置
	restoreScrollPosition(taskId) {
		const position = this.taskScrollPositions[taskId];
		if (position) {
			if (this.$refs.timelineRef) {
				this.$refs.timelineRef.scrollTop = position.timelineScroll || 0;
			}
			window.scrollTo(0, position.windowScroll || 0);
		}
	},
}
</script>

<style>
/* Reduce custom styles and use Tailwind where possible */
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

.project-list {
	max-height: calc(100vh - 120px);
	overflow-y: auto;
}

.el-aside {
	background-color: #f9fafb;
	border-right: 1px solid #e5e7eb;
}

/* Keep necessary Markdown styles */
.markdown-preview {
	font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Helvetica, Arial, sans-serif;
	line-height: 1.6;
}

.markdown-preview h1 {
	font-size: 2em;
	border-bottom: 1px solid #eaecef;
	padding-bottom: .3em;
}

.markdown-preview h2 {
	font-size: 1.5em;
	border-bottom: 1px solid #eaecef;
	padding-bottom: .3em;
}

.markdown-preview h3 {
	font-size: 1.25em;
}

.markdown-preview h4 {
	font-size: 1em;
}

.markdown-preview h5 {
	font-size: .875em;
}

.markdown-preview h6 {
	font-size: .85em;
	color: #6a737d;
}

/* 列表支持需要有开头原点装饰 */
.markdown-preview ul {
	list-style-type: disc;
	padding-left: 1.5em;
}

/* Basic elements */
.markdown-preview p {
	margin-bottom: 1em;
}

/* Ordered lists */
.markdown-preview ol {
	list-style-type: decimal;
	padding-left: 1.5em;
	margin-bottom: 1em;
}

/* Nested lists */
.markdown-preview ul ul,
.markdown-preview ol ul {
	list-style-type: circle;
}

.markdown-preview ul ul ul,
.markdown-preview ol ul ul {
	list-style-type: square;
}

/* Code blocks and inline code */
.markdown-preview code {
	font-family: SFMono-Regular, Consolas, "Liberation Mono", Menlo, monospace;
	background-color: rgba(27, 31, 35, 0.05);
	border-radius: 3px;
	padding: 0.2em 0.4em;
	font-size: 85%;
}

.markdown-preview pre {
	background-color: #f6f8fa;
	border-radius: 3px;
	padding: 16px;
	overflow: auto;
	margin-bottom: 1em;
}

.markdown-preview pre code {
	background-color: transparent;
	padding: 0;
}

/* Blockquotes */
.markdown-preview blockquote {
	border-left: 4px solid #dfe2e5;
	padding-left: 1em;
	color: #6a737d;
	margin-left: 0;
	margin-right: 0;
}

/* Links */
.markdown-preview a {
	color: #0366d6;
	text-decoration: none;
}

.markdown-preview a:hover {
	text-decoration: underline;
}

/* Tables */
.markdown-preview table {
	border-collapse: collapse;
	margin-bottom: 1em;
	width: 100%;
	overflow: auto;
}

.markdown-preview table th,
.markdown-preview table td {
	border: 1px solid #dfe2e5;
	padding: 6px 13px;
}

.markdown-preview table th {
	background-color: #f6f8fa;
	font-weight: 600;
}

.markdown-preview table tr:nth-child(2n) {
	background-color: #f6f8fa;
}

/* Images */
.markdown-preview img {
	max-width: 100%;
	box-sizing: border-box;
}

/* Horizontal rule */
.markdown-preview hr {
	height: 0.25em;
	padding: 0;
	margin: 24px 0;
	background-color: #e1e4e8;
	border: 0;
}

/* Task list */
.markdown-preview input[type="checkbox"] {
	margin-right: 0.5em;
}


/* 添加新的固定侧边栏样式 */
.fixed-aside {
	position: fixed;
	top: 0;
	left: 0;
	height: 100vh;
	display: flex;
	flex-direction: column;
	z-index: 100;
}

.sidebar-header {
	background-color: #f9fafb;
}

.el-aside {
	background-color: #f9fafb;
	border-right: 1px solid #e5e7eb;
}

.project-list-container {
	flex: 1;
	overflow: hidden;
}

.project-list {
	height: 100%;
	overflow-y: auto;
}

/* 调整主内容区域的位置 */
.content-wrapper {
	margin-left: 250px;
}

.stats-item {
	padding: 1rem;
	background-color: #f8fafc;
	border-radius: 0.5rem;
	border: 1px solid #e2e8f0;
}
</style>