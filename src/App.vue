<template>

	<body>
		<div>
			<div class="py-4 text-center text-3xl font-extrabold">
				<span class="bg-clip-text text-transparent bg-gradient-to-r from-pink-500 to-violet-500">
					TEXIO(德士) PAR20-4H 控制台
				</span>
			</div>
			<!-- 数据面板 -->
			<div class="flex flex-row px-3">
				<!-- 实际 电压 电流 数据回显区 -->
				<div class="grid grid-cols-1 gap-2">
					<!-- 电压 -->
					<div class="px-5 py-2 bg-blue-50 text-blue-500 rounded-lg border border-blue-500 text-3xl">
						<div class="flex flex-row items-center text-nowrap">
							<h2 class="text-nowrap">{{isOutputActive?"电压":"设置电压"}}</h2>
							<div v-show="isOutputActive">
								<input type="text" :value="voltage.toFixed(3)"
								class="w-full p-2 text-center bg-transparent border-none focus:outline-none " readonly>
								/
							</div>
							
							<input type="number" v-model="voltageSetValue" @wheel="handleWheel($event, 'voltage')" @blur="updateVoltage"
								:disabled="isAutoRefreshing"
								class="w-full p-2 text-center bg-transparent border-none focus:outline-none">
							<span class="ml-2 bg-blue-200 px-2 py-1 rounded w-12 text-center">V</span>
						</div>
					</div>

					<!-- 电流 -->
					<div class="px-5 py-2 bg-green-50 text-green-500 rounded-lg border border-green-500 text-3xl">
						<div class="flex flex-row items-center text-nowrap">
							<h2 class="text-nowrap">{{isOutputActive?"电流":"设置电流"}}</h2>
							<div v-show="isOutputActive">
								<input type="text" :value="current.toFixed(4)"
									class="w-full p-2 text-center bg-transparent border-none focus:outline-none" readonly>
								/
							</div>
							<input type="number" v-model="currentSetValue" @wheel="handleWheel($event, 'current')" @blur="updateCurrent"
								:disabled="isAutoRefreshing"
								class="w-full p-2 text-center bg-transparent border-none focus:outline-none">
							<span class="ml-2 bg-green-200 px-2 py-1 rounded w-12 text-center">A</span>
						</div>
					</div>
					<!-- 功率 -->
					<div class="px-5 py-2 bg-red-50 text-red-500 rounded-lg border border-red-500 text-3xl" 
						@click="manualRefresh">  <!-- 添加点击事件 -->
						<div class="flex flex-row items-center text-nowrap">
							<h2 class="text-nowrap">{{isOutputActive?"功率":"最大功率"}}</h2>
							<div v-show="!isOutputActive">
								<input type="text" :value="(voltageSetValue*currentSetValue).toFixed(2)"
									class="w-full p-2 text-center bg-transparent border-none focus:outline-none " readonly>
							</div>
							<div v-show="isOutputActive">
								<input type="text" :value="power.toFixed(4)"
									class="w-full p-2 text-center bg-transparent border-none focus:outline-none " readonly>
							</div>
							<span class="ml-2 bg-red-200 px-2 py-1 rounded w-12 text-center">W</span>
						</div>
					</div>
					<!-- 新增提示文字 -->
					<div class="text-xs text-red-400">* 状态查询指令ST会严重影响该古董仪器的实时性<br>点击此框手动同步面板</div>
				</div>
			</div>
			<!-- 操作面板 -->
			<div class="flex flex-col px-3">
				<!-- 删除长按提示文字 -->
				<div v-if="successMessage" class="text-sm text-green-500 text-center p-2">
					{{ successMessage }}
				</div>
				<div v-else class="text-sm text-red-500 text-center p-2 ">
					{{ autoCloseMessage }}
				</div>
				<!-- 来4个按钮并列，分别是 记忆1 记忆2 ��忆3 工作区 -->
				<div class="grid grid-cols-4 gap-2 text-nowrap">
					<button
						:class="[activeMemoryIndex === index ? 'bg-stone-500 p-2 text-white rounded-lg border border-stone-500' : 'border-dashed border-2 border-stone-600 rounded-lg']"
						:disabled="isAutoRefreshing"
						v-for="(memory, index) in memories" :key="index" @click="activateMemory(index)">
						<ul>
							<li>记忆{{ index + 1 }}</li>
							<li>{{ memory.voltage }}V</li>
							<li>{{ memory.current }}A/{{ memory.current_ua }}A</li>
						</ul>
					</button>
					<button
						:class="[activeMemoryIndex === 'workspace' ? 'bg-stone-500 p-2 text-white rounded-lg border border-stone-500' : 'border-dashed border-2 border-stone-600 rounded-lg']"
						:disabled="isAutoRefreshing"
						@click="activateMemory('workspace')">
						<ul>
							<li>工作区</li>
							<li>{{ workspace.voltage }}V</li>
							<li>{{ workspace.current }}A/{{ workspace.current_ua }}A</li>
						</ul>
					</button>
				</div>
				<!-- 按钮面板拆分为1(嵌套2x2)x2 -->
				<div class="flex flex-row">
					<div class="grid grid-cols-3 grid-rows-2 gap-2 text-nowrap pt-3 basis-2/3">
						<button :disabled="isAutoRefreshing"
							class="p-2 rounded-lg border-2 active:bg-orange-500 active:text-white active:border-orange-500 border-dashed border-orange-600 bg-orange-50 text-orange-700 flex items-center justify-center"
							@click="toggleFrontLock">
							<div class="text-center">
								物理面板<br>解锁按钮
							</div>
						</button>
						<button
							:disabled="isAutoRefreshing"
							:class="[isOutputProtected ? 'bg-orange-500 text-white border-orange-500' : 'border-dashed border-green-600 bg-green-50 text-green-700']"
							class="p-2 rounded-lg border-2" @click="toggleOutputProtection"
							v-html="isOutputProtected ? '输出保护<br>已锁定' : '输出保护<br>已解锁'"></button>
						<button
							:class="[isMicroampereMode ? 'bg-emerald-500 text-white border-emerald-500' : 'border-dashed border-slate-600 bg-slate-50 text-slate-700']"
							class="p-2 rounded-lg border-2" @click="toggleMicroampereMode"
							v-html="isMicroampereMode ? '微安精度<br>已激活' : '微安精度<br>已关闭'"
							:disabled="isOutputActive || isAutoRefreshing"></button>
						<!-- 新增刷新控制按钮 -->
						<button
							:class="[isAutoRefreshing ? 'bg-purple-500 text-white border-purple-500' : 'border-dashed border-purple-600 bg-purple-50 text-purple-700']"
							class="p-2 rounded-lg border-2" @click="toggleAutoRefresh"
							v-html="isAutoRefreshing ? '实时刷新<br>已开启' : '实时刷新<br>已关闭'"></button>
						<div
							:class="[isOverVolProtected ? 'bg-orange-500 text-white border-orange-500' : 'border-dashed border-green-600 bg-green-50 text-green-700']"
							class="p-2 rounded-lg border-2  flex items-center justify-center">
							<div class="w-full text-center">
								<div class="mb-1">
									<span v-if="isOverVolProtected">OVP警告</span>
									<span v-else>OVP正常</span>
								</div>
								<input type="number" v-model="ovpValue"
									class="w-12 text-center bg-transparent border-none focus:outline-none"
									placeholder="--.-" />V
							</div>
						</div>
						<!-- 添加 CC/CV 状态显示按钮 -->
						<div
							class="p-2 rounded-lg border-2 border-blue-600 bg-blue-50 text-blue-700 flex items-center justify-center">
							<div class="text-center flex items-center">
								工作模式<br>
								{{ isConstantCurrent ? 'CC模式' : 'CV模式' }}
							</div>
						</div>
					</div>
					<!-- 激活按钮 -->
					<button :disabled="isAutoRefreshing"
						:class="[isOutputActive ? 'bg-emerald-500 text-white border-emerald-500' : 'border-dashed border-orange-600 bg-orange-50 text-orange-700']"
						class="mt-3 ms-2 rounded-lg border-2 grow text-4xl leading-loose"
						@click="toggleOutputActivation" v-html="isOutputActive ? '输出<br>已激活' : '输出<br>已关闭'"></button>
				</div>
			</div>
		</div>
	</body>
</template>

<script>
import { ref, computed, onMounted, onUnmounted } from 'vue'
import { RouterLink, RouterView } from 'vue-router'
import axios from 'axios'
import './styles/index.css'

export default {
	setup() {
	},
	data() {
		return {
			voltage: 0,
			current: 0,
			set_voltage: 0,
			set_current: 0,
			workspace: { voltage: 0, current: 0, current_ua: 0 },
			memories: [
				{ voltage: 0, current: 0, current_ua: 0 },
				{ voltage: 0, current: 0, current_ua: 0 },
				{ voltage: 0, current: 0, current_ua: 0 }
			],
			activeMemoryIndex: "workspace", // 添加 activeMemoryIndex 来跟踪当前激活的记忆索引
			successMessage: '', // 添加 successMessage 来存储成功提示信息
			// 删除 isFrontLocked: true,
			isButtonPressed: false, // 添加按钮按压状态
			isOutputProtected: true, // 添加 isOutputProtected 来跟踪输出是否受保护
			isOutputActive: false, // 添加 isOutputActive 来跟踪输出是否激活
			isMicroampereMode: true, // 添加 isMicroampereMode 来跟踪是否为微安模式
			isOverVolProtected: false,
			autoCloseMessage: '', // 添加 autoCloseMessage 来存储自动关闭微安模式的提示信息
			ovpValue: 0, // 添加 ovpValue 来存储OVP数值
			apiBaseUrl: '/api', // 修改为相对路径
			updateInterval: null, // 添加状态更新定时器
			isConstantCurrent: false, // 添加恒流模式状态
			current_ua: 0, // 添加微安模式下的电流值
			updateTimeout: null,  // 添加防抖定时器
			isAutoRefreshing: false,
			refreshInterval: null,
			isLoading: false, // 添加加载状态标志
			isMemorySwitching: false, // 添加新的标志位
			voltageUpdateTimeout: null,  // 添加电压防抖定时器
			currentUpdateTimeout: null,   // 添加电流防抖定时器
			isRefreshing: false, // 添加标识符表示是否正在刷新中
		}
	},
	async mounted() {
		// 初始化数据
		await this.loadInitialData();
		// 启动定时更新
		// this.updateInterval = setInterval(this.updateOutputStatus, 3000);
	},
	beforeUnmount() {
		// 清理定时器
		if (this.updateInterval) {
			clearInterval(this.updateInterval);
		}
		if (this.refreshInterval) {
			clearInterval(this.refreshInterval);
		}
		this.isAutoRefreshing = false; // 确保组件销毁时停止刷新
	},
	watch: {
		// ...其他 watch 保持不变...
	},
	computed: {
		power() {
			return this.voltage * this.current
		},
		currentSetValue: {
			get() {
				if (this.activeMemoryIndex === 'workspace') {
					return this.isMicroampereMode ? this.workspace.current_ua : this.workspace.current;
				} else {
					return this.isMicroampereMode ?
						this.memories[this.activeMemoryIndex].current_ua :
						this.memories[this.activeMemoryIndex].current;
				}
			},
			set(value) {
				const shouldUpdate = this.isValueChanged(value);
				if (this.activeMemoryIndex === 'workspace') {
					if (this.isMicroampereMode) {
						this.workspace.current_ua = value;
					} else {
						this.workspace.current = value;
					}
				} else {
					if (this.isMicroampereMode) {
						this.memories[this.activeMemoryIndex].current_ua = value;
					} else {
						this.memories[this.activeMemoryIndex].current = value;
					}
				}
				 // 修改这里：改用updateCurrent方法替代之前的debounceCurrentUpdate
				if (shouldUpdate) {
					this.updateCurrent();
				}
			}
		},
		voltageSetValue: {
			get() {
				if (this.activeMemoryIndex === 'workspace') {
					return this.workspace.voltage;
				} else {
					return this.memories[this.activeMemoryIndex].voltage;
				}
			},
			set(value) {
				const shouldUpdate = Math.abs(value - this.voltageSetValue) > 0.0001;
				if (this.activeMemoryIndex === 'workspace') {
					this.workspace.voltage = value;
				} else {
					this.memories[this.activeMemoryIndex].voltage = value;
				}
				if (shouldUpdate) {
					this.updateVoltage();
				}
			}
		}
	},
	methods: {
		async loadInitialData() {
			if (this.isLoading) return;
			
			try {
				this.isLoading = true;
				// 只在启动时加载预设记忆
				await this.loadMemoryPreset();
				await this.updateOutputStatus();
				 // 添加系统状态同步
				await this.syncSystemStatus();
			} catch (error) {
				console.error('Failed to load initial data:', error);
			} finally {
				this.isLoading = false;
			}
		},

		// 新增加载预设记忆的独立方法
		async loadMemoryPreset() {
			try {
				const memoryResponse = await axios.get(`${this.apiBaseUrl}/get_memory_preset`);
				if (memoryResponse.data.code === 0) {
					const { data } = memoryResponse.data;
					this.workspace = data.workspace;
					this.memories = [data.memory1, data.memory2, data.memory3];
				}
			} catch (error) {
				console.error('Failed to load memory preset:', error);
			}
		},

		async updateOutputStatus() {
			try {
				const response = await axios.get(`${this.apiBaseUrl}/get_output_status`);
				if (response.data.code === 0) {
					// 更新为新的数据结构
					const { data } = response.data;
					this.voltage = data.voltage;
					this.current = data.current;
					this.current_ua = data.current_ua || 0; // 添加微安模式下的电流值
					this.ovpValue = data.OVP;
					this.isConstantCurrent = data.is_CC;
				}
			} catch (error) {
				console.error('Failed to update output status:', error);
			}
		},

		async toggleOutputActivation() {
			try {
				const response = await axios.post(
					`${this.apiBaseUrl}/control_output?enable=${!this.isOutputActive}`
				);
				if (response.data.code === 0) {
					this.isOutputActive = !this.isOutputActive;
					// 激活输出时自动锁定面板
					if (this.isOutputActive) {
						this.isFrontLocked = true;
					}
					//刷新数据
					// await this.updateOutputStatus();
				}
			} catch (error) {
				console.error('Failed to toggle output:', error);
			}
		},

		async toggleFrontLock() {
			try {
				await axios.post(`${this.apiBaseUrl}/unlock_panel`);
			} catch (error) {
				console.error('Failed to toggle front lock:', error);
			}
		},

		async toggleOutputProtection() {
			try {
				const response = await axios.post(
					`${this.apiBaseUrl}/toggle_protection?enable=${!this.isOutputProtected}`
				);
				if (response.data.code === 0) {
					this.isOutputProtected = !this.isOutputProtected;
				}
			} catch (error) {
				console.error('Failed to toggle protection:', error);
			}
		},

		async toggleMicroampereMode() {
			try {
				const response = await axios.post(
					`${this.apiBaseUrl}/set_ua_accuracy?enable=${!this.isMicroampereMode}`
				);
				if (response.data.code === 0) {
					this.isMicroampereMode = !this.isMicroampereMode;
				}
			} catch (error) {
				console.error('Failed to toggle microampere mode:', error);
			}
		},

		async activateMemory(memoryIndex) {
			if (this.isOutputProtected) {
				this.isOutputActive = false;
			}

			try {
				const memoryObj = memoryIndex === 'workspace' ? 'workspace' : `memory${memoryIndex + 1}`;
				const response = await axios.post(`${this.apiBaseUrl}/select_output`, {
					memoryObj: memoryObj
				});

				if (response.data.code === 0) {
					// 先更新内部状态
					this.activeMemoryIndex = memoryIndex;
					
					// 根据选择的记忆区获取对应的预设值
					const targetMemory = memoryIndex === 'workspace' ? 
						this.workspace : 
						this.memories[memoryIndex];

					// 直接设置值到组件状态，不触发watch
					Object.assign(this.$data, {
						set_voltage: targetMemory.voltage,
						set_current: this.isMicroampereMode ? 
							targetMemory.current_ua : 
							targetMemory.current
					});
				}
			} catch (error) {
				console.error('Failed to activate memory:', error);
			}
		},

		async handleWheel(event, type) {
			if (this.isAutoRefreshing) return; // 如果是刷新状态,直接返回

			const delta = Math.sign(event.deltaY);
			if (type === 'voltage') {
				const newVoltage = parseFloat((this.voltageSetValue - delta * 0.1).toFixed(3));
				this.voltageSetValue = Math.min(20.5, Math.max(0, newVoltage));
			} else if (type === 'current') {
				const step = this.isMicroampereMode ? 0.01 : 0.1;
				const newCurrent = parseFloat((this.currentSetValue - delta * step).toFixed(3));
				this.currentSetValue = Math.min(this.isMicroampereMode ? 1 : 4, Math.max(0, newCurrent));
				await this.updateCurrent();  // 直接调用更新方法
			}
		},

		async executeAutoRefresh() {
			if (!this.isAutoRefreshing) return;
			
			try {
				await this.updateOutputStatus();
				// 如果仍然开启自动刷新,立即开始下一次刷新
				if (this.isAutoRefreshing) {
					this.executeAutoRefresh();
				}
			} catch (error) {
				console.error('Auto refresh failed:', error);
				// 发生错误时如果仍在自动刷新,继续尝试
				if (this.isAutoRefreshing) {
					this.executeAutoRefresh();
				}
			}
		},

		toggleAutoRefresh() {
			this.isAutoRefreshing = !this.isAutoRefreshing;
			
			if (this.isAutoRefreshing) {
				// 开始自动刷新
				this.executeAutoRefresh();
			}
		},

		async manualRefresh() {
			if (this.isLoading) return; // 防止重复执行
			
			try {
				this.isLoading = true;
				 // 添加系统状态同步
				await this.syncSystemStatus();
				await this.loadMemoryPreset();
				await this.updateOutputStatus();
				
				// 在刷新完成后,根据当前选择的工作区同步设置值
				if(this.activeMemoryIndex === 'workspace') {
					this.set_voltage = this.workspace.voltage;
					this.set_current = this.isMicroampereMode ? this.workspace.current_ua : this.workspace.current;
				} else {
					const memory = this.memories[this.activeMemoryIndex];
					this.set_voltage = memory.voltage;
					this.set_current = this.isMicroampereMode ? memory.current_ua : memory.current;
				}
			} catch (error) {
				console.error('Failed to refresh data:', error);
			} finally {
				this.isLoading = false;
			}
		},

		// 添加新方法用于检查值是否真正改变
		isValueChanged(newValue) {
			const currentValue = this.activeMemoryIndex === 'workspace' 
				? (this.isMicroampereMode ? this.workspace.current_ua : this.workspace.current)
				: (this.isMicroampereMode ? this.memories[this.activeMemoryIndex].current_ua : this.memories[this.activeMemoryIndex].current);
			
			return Math.abs(newValue - currentValue) > 0.0001; // 添加一个小的阈值来处理浮点数比较
		},

		// 添加防抖函数
		debounceUpdate(fn, timeout) {
			if (timeout) {
				clearTimeout(timeout);
			}
			return setTimeout(fn, 200);
		},

		// 修改电压更新方法
		async updateVoltage() {
			this.voltageUpdateTimeout = this.debounceUpdate(async () => {
				const newVal = this.voltageSetValue;  // 修改这里：使用 voltageSetValue 而不是 set_voltage
				if (newVal < 0) {
					this.voltageSetValue = 0;  // 修改这里
				} else if (newVal > 20.5) {
					this.voltageSetValue = 20.5;  // 修改这里
				} else {
					this.isFrontLocked = true;
					try {
						const memoryObj = this.activeMemoryIndex === 'workspace' ?
							'workspace' : `memory${this.activeMemoryIndex + 1}`;
						await axios.post(`${this.apiBaseUrl}/set_voltage`, {
							voltage: parseFloat(newVal.toFixed(3)),
							memoryObj: memoryObj
						});
					} catch (error) {
						console.error('Failed to set voltage:', error);
					}
				}
			}, this.voltageUpdateTimeout);
		},

		// 修改电流更新方法
		async updateCurrent() {
			this.currentUpdateTimeout = this.debounceUpdate(async () => {
				const newVal = this.currentSetValue;
				const maxCurrent = this.isMicroampereMode ? 1 : 4;
				
				if (newVal < 0) {
					this.currentSetValue = 0;
				} else if (newVal > maxCurrent) {
					this.currentSetValue = maxCurrent;
				} else {
					this.isFrontLocked = true;
					try {
						const memoryObj = this.activeMemoryIndex === 'workspace' ?
							'workspace' : `memory${this.activeMemoryIndex + 1}`;
						await axios.post(`${this.apiBaseUrl}/set_current`, {
							current: parseFloat(newVal.toFixed(3)),
							is_uaAccuracy: this.isMicroampereMode,
							memoryObj: memoryObj
						});
					} catch (error) {
						console.error('Failed to set current:', error);
					}
				}
			}, this.currentUpdateTimeout);
		},

		// 添加新的系统状态同步方法
		async syncSystemStatus() {
			try {
				const response = await axios.get(`${this.apiBaseUrl}/getSystemStatus`);
				if (response.data.code === 0) {
					const { data } = response.data;
					// 同步输出状态
					this.isOutputActive = data.is_output_on === 3;
					// 同步保护状态
					this.isOutputProtected = data.is_protection_on === 1;
					// 同步微安精度模式
					this.isMicroampereMode = data.is_ua_accuracy === 1;
					// 同步记忆区域
					const memoryMap = {
						0: 'workspace',
						1: 0,
						2: 1,
						3: 2
					};
					this.activeMemoryIndex = memoryMap[data.memory_preset];
					
					// 新增: 同步设置电压和电流值
					if(this.activeMemoryIndex === 'workspace') {
						this.set_voltage = this.workspace.voltage;
						this.set_current = this.isMicroampereMode ? this.workspace.current_ua : this.workspace.current;
					} else {
						const memory = this.memories[this.activeMemoryIndex];
						this.set_voltage = memory.voltage;
						this.set_current = this.isMicroampereMode ? memory.current_ua : memory.current;
					}
				}
			} catch (error) {
				console.error('Failed to sync system status:', error);
			}
		},
	}
}
</script>

<style scoped>
.label {
	white-space: nowrap;
	width: 50px;
}

button:disabled,
input:disabled {
	opacity: 0.5;
	cursor: not-allowed;
}
</style>