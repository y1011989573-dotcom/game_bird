<template>
	<!-- Grid布局容器 -->
	<div class="grid grid-cols-4 gap-3 p-0!">
		<el-card v-for="player_trap in displayTraps" :key="player_trap.id" shadow="hover"  body-class=" p-1!">
			<div class="flex flex-col items-center text-center relative">
				<!-- 锁定状态遮罩 -->
				<div v-if="player_trap.is_locked && !player_trap.is_placeholder" class="absolute inset-0 flex flex-col items-center justify-center bg-white/90 backdrop-blur-sm z-10 rounded">
					<el-icon :size="48" class="text-gray-400 mb-3">
						<Lock />
					</el-icon>
					<div class="text-xl font-bold text-red-500 mb-2">
						{{ getUnlockPrice(player_trap) }}
						<span class="text-base">{{ getCurrencyName(player_trap) }}</span>
					</div>
					<el-button
						type="primary"
						size="small"
						:disabled="!canUnlock(player_trap)"
						@click="handleUnlock(player_trap)"
					>
						解锁陷阱
					</el-button>
				</div>

				<!-- 倒计时 / 解锁价格 - 覆盖在右上角 -->
				<div v-if="!player_trap.is_locked || player_trap.is_placeholder" class="rounded-bl w-full">
					<!-- 占位符陷阱：显示解锁价格 -->
					<div v-if="player_trap.is_placeholder" class="text-center py-1!">
						<span class="text-xs font-bold text-orange-500 ">
							{{ getPlaceholderPrice() }} {{ getPlaceholderCurrency() }}
						</span>
					</div>
					<!-- 普通陷阱：显示倒计时 -->
					<el-countdown
						v-else
						:value="get_trap_deadline(player_trap)"
						format="HH:mm:ss"
						:value-style="{ fontSize: '12px', color: '#4d9ef8', fontWeight: 'bold' }"
					/>
				</div>


				<!-- 圆形陷阱图片 -->
				<el-avatar
					v-if="!player_trap.is_locked || player_trap.is_placeholder"
					:size="80"
					:src="player_trap?.selected_bird ? getImageUrl('bird', player_trap.selected_bird.nickname) : player_trap?.image_url"
					:class="[' border-2', getTrapBorderClass(player_trap)]"
				>
					<img :src="trapPlaceholder" class="slot-icon" alt="trap_placeholder" 
					/>
				</el-avatar>


				<!-- 按钮区域 - 垂直排列 -->
				<div v-if="!player_trap.is_locked || player_trap.is_placeholder" class="w-full space-y-2! pt-2!">
					<!-- 陷阱名称按钮（点击更换陷阱） -->
					<el-button v-if="!player_trap.is_placeholder" class="w-full ml-0!" size="small" @click="changeTrap(player_trap)">
						{{ player_trap?.selected_bird ? player_trap.selected_bird.nickname : player_trap?.nickname }}
					</el-button>
					<el-button v-else class="w-full ml-0!" size="small" disabled>
						{{ player_trap.nickname }}
					</el-button>

					<!-- 占位符陷阱：显示解锁按钮 -->
					<el-button
						v-if="player_trap.is_placeholder"
						class="w-full ml-0!"
						size="small"
						type="warning"
						:disabled="!canUnlockPlaceholder()"
						@click="handleUnlock(player_trap)"
					>
						{{ getNextUnlockLevel() }}级解锁
					</el-button>

					<!-- 条件显示的操作按钮 -->
					<!-- 情况1: 未下饵 - 显示"下饵"按钮 -->
					<el-button
						v-if="!player_trap.is_placeholder && !player_trap?.selected_bird"
						class="w-full ml-0!"
						size="small"
						type="primary"
						@click="show_bait_list(player_trap)"
					>
						下饵
					</el-button>

					<!-- 情况2: 已下饵且倒计时结束 - 显示"收获"按钮 -->
					<el-button
						v-else-if="!player_trap.is_placeholder && get_trap_time(player_trap) <= 0"
						class="w-full ml-0!"
						size="small"
						type="success"
						@click="get_bird(player_trap)"
					>
						收获
					</el-button>

					<!-- 情况3: 已下饵且倒计时进行中 - 显示"使用加速道具"按钮 -->
					<el-button
						v-else-if="!player_trap.is_placeholder"
						class="w-full ml-0!"
						size="small"
						type="warning"
						@click="show_trap_buff_list(player_trap)"
					>
						使用加速道具
					</el-button>
				</div>
			</div>
		</el-card>
	</div>

	<!-- 下饵弹窗 -->
	<el-dialog v-model="vis_item_bait" title="选择诱饵">
		<el-card v-for="bait in game.player_item_bait.data" :key="bait.id" shadow="never" bodyClass="flex p-2! justify-between glass-effect">
			<div>
				<div class="text-lg">{{ bait.game_item_bait.nickname }}x{{ bait.count }}</div>
			</div>

			<div class="text-right">
				<el-button type="primary" @click="set_bait(bait)">下饵</el-button>
			</div>
		</el-card>
	</el-dialog>

	<!-- 使用陷阱加成道具弹窗 -->
	<el-dialog v-model="vis_item_trap_buff" title="选择陷阱加成道具">
		<div v-if="!game.player_item_trap_buff?.data || game.player_item_trap_buff.data.length === 0" class="text-center text-gray-400 py-4">
			暂无陷阱加成道具
		</div>
		<el-card v-for="buff in game.player_item_trap_buff.data" :key="buff.id" shadow="never" bodyClass="flex p-2! justify-between glass-effect">
			<div>
				<div class="text-lg">{{ buff.game_item_trap_buff.nickname }}x{{ buff.count }}</div>
				<div class="text-sm text-gray-500">{{ buff.game_item_trap_buff.desc }}</div>
				<div class="text-sm text-purple-600">时间加成: +{{ buff.game_item_trap_buff.buff_time }}%</div>
			</div>

			<div class="text-right">
				<el-button type="primary" @click="use_trap_buff(buff)">使用</el-button>
			</div>
		</el-card>
	</el-dialog>

	<!-- 鸟信息弹窗 -->
	<PlayerBirdInfo ref="birdInfoDialog" />
</template>


<script setup>
import {inject, onActivated, onDeactivated, onMounted, onUnmounted, ref, computed} from "vue";
import { ElMessageBox} from 'element-plus'
import { message } from '@/game/notification-center';
import {Lock} from '@element-plus/icons-vue';
import {getImageUrl} from '@/config/oss'
import PlayerBirdInfo from '@/components/playerbird/PlayerBirdInfo.vue'
import trapPlaceholder from './normal_trap.png'


const game = inject('game')
const vis_item_bait = ref(false)
const vis_item_trap_buff = ref(false)
const birdInfoDialog = ref(null)


const select_trap = ref(null)

// 添加用于存储定时器的引用
const timer = ref(null)
const currentTime = ref(Date.now())

// 陷阱配置（用于计算解锁价格）
const trapConfig = ref(null)

// 显示的陷阱列表（包括占位符代表下一个可解锁的陷阱）
const displayTraps = computed(() => {
	if (!game.player_trap.data) return []

	// 显示所有未锁定的陷阱
	const unlockedTraps = game.player_trap.data.filter(t => !t.is_locked)

	// 检查是否还有可以解锁的陷阱
	const nextLevel = getNextUnlockLevel()
	if (nextLevel !== null) {
		// 添加占位符代表下一个可以解锁的陷阱
		unlockedTraps.push({
			id: 'next_unlock',
			nickname: '待解锁陷阱',
			image_url: '',
			is_locked: true,
			is_placeholder: true
		})
	}

	return unlockedTraps
})

const show_bait_list = async (trap) => {
	await game.player_item_bait.update()
	select_trap.value = trap
	vis_item_bait.value = true
}

const set_bait = async (item) => {
	const res = await game.player_trap.set_bait(select_trap.value.id, item.id)
	vis_item_bait.value = false
	if (res.code !== 200) {
		message.error(res.msg)
	}
  await game.player_trap.update()
}

const get_bird = async (trap) => {
	const res = await game.player_trap.get_bird(trap.id)
	if (res.code !== 200) {
		message.error(res.msg)
		return
	}

	// 获取新鸟数据
	const newBird = res.data.new_bird
	const expGained = res.data.exp_gained || 0

	// 添加到鸟仓库
	if (game.player_bird.data && newBird) {
		game.player_bird.data.push(newBird)
	}

	// 显示成功消息（包含经验信息）
	if (expGained > 0) {
		message.success(`捕获成功！获得 ${newBird.game_bird.nickname}，经验 +${expGained}`)
	} else {
		message.success(`捕获成功！获得 ${newBird.game_bird.nickname}`)
	}

	// 显示新鸟信息弹窗
	if (birdInfoDialog.value && newBird) {
		birdInfoDialog.value.show(newBird)
	}

	// 更新陷阱数据
	await game.player_trap.update()
}

const show_trap_buff_list = async (trap) => {
	await game.player_item_trap_buff.update()
	select_trap.value = trap
	vis_item_trap_buff.value = true
}

const use_trap_buff = async (buff) => {
	if (!select_trap.value) {
		message.error('请先选择陷阱')
		return
	}

	const res = await game.player_trap.use_player_item_trap_buff(select_trap.value.id, buff.game_item_trap_buff.id)
	vis_item_trap_buff.value = false
	if (res.code !== 200) {
		message.error(res.msg)
		return
	}
	message.success('使用成功')
	await game.player_trap.update()
	await game.player_item_trap_buff.update()
}

const changeTrap = (trap) => {
	message.info('更换陷阱功能开发中')
}

const get_trap_time = (trap) => {
	if (!trap.selected_bird) {
		return 0
	}

	const need_time = trap.selected_bird.need_time;
	const over_time = Number(trap.use_time) + need_time;
	return Math.ceil(over_time - currentTime.value / 1000);
}

// 计算陷阱收获的目标时间戳（毫秒）
const get_trap_deadline = (trap) => {
	if (!trap?.selected_bird || !trap?.use_time) {
		return Date.now()
	}
	const useTime = Number(trap.use_time)
	const needTime = trap.selected_bird.need_time
	return (useTime + needTime) * 1000
}

// 获取陷阱边框颜色类
const getTrapBorderClass = (trap) => {
	if (!trap?.selected_bird) {
		// 未下饵 - 灰色边框
		return 'border-gray-300 bg-gray-100'
	}

	const timeLeft = get_trap_time(trap)
	if (timeLeft <= 0) {
		// 可收获 - 绿色边框
		return 'border-green-500 bg-green-50'
	}

	// 进行中 - 橙色边框
	return 'border-orange-500 bg-orange-50'
}

// 获取解锁价格
const getUnlockPrice = (trap) => {
	if (!trap.is_locked || !trapConfig.value) return 0

	const trapIndex = game.player_trap.data.findIndex(t => t.id === trap.id)
	if (trapIndex === -1) return 0

	const config = trapConfig.value

	// 初始陷阱不需要解锁价格
	if (trapIndex < config.initial_trap_count) {
		return 0
	}

	// 计算该陷阱属于哪个等级奖励
	let accumulatedTraps = config.initial_trap_count

	if (config.level_rewards) {
		for (const reward of config.level_rewards) {
			const rewardEndIndex = accumulatedTraps + reward.traps
			if (trapIndex < rewardEndIndex) {
				return reward.unlock_price
			}
			accumulatedTraps = rewardEndIndex
		}
	}

	return 0
}

// 获取货币类型
const getCurrencyType = (trap) => {
	if (!trap.is_locked || !trapConfig.value) return 1

	const trapIndex = game.player_trap.data.findIndex(t => t.id === trap.id)
	if (trapIndex === -1) return 1

	const config = trapConfig.value

	// 初始陷阱不需要解锁
	if (trapIndex < config.initial_trap_count) {
		return 1
	}

	// 计算该陷阱属于哪个等级奖励
	let accumulatedTraps = config.initial_trap_count

	if (config.level_rewards) {
		for (const reward of config.level_rewards) {
			const rewardEndIndex = accumulatedTraps + reward.traps
			if (trapIndex < rewardEndIndex) {
				return reward.unlock_price_type
			}
			accumulatedTraps = rewardEndIndex
		}
	}

	return 1
}

// 获取货币名称
const getCurrencyName = (trap) => {
	const type = getCurrencyType(trap)
	const names = { 1: '金币', 2: '元宝', 3: '星币' }
	return names[type] || '金币'
}

// 检查是否可以解锁（必须是第一个锁定的陷阱）
const canUnlock = (trap) => {
	if (!trap.is_locked) return false
	// 占位符陷阱总是可以点击（显示解锁条件）
	if (trap.is_placeholder) return true
	const firstLocked = game.player_trap.data.find(t => t.is_locked)
	return firstLocked?.id === trap.id
}

// 处理解锁
const handleUnlock = async (trap) => {
	// 只处理占位符陷阱
	if (!trap.is_placeholder) {
		message.error('该陷阱已解锁')
		return
	}

	// 检查是否满足等级要求
	if (!canUnlockPlaceholder()) {
		const nextLevel = getNextUnlockLevel()
		message.info(`需要达到 ${nextLevel} 级才能解锁下一个陷阱`)
		return
	}

	// 满足等级要求，执行解锁
	const price = getPlaceholderPrice()
	const currencyName = getPlaceholderCurrency()

	try {
		await ElMessageBox.confirm(
			`确定要花费 ${price} ${currencyName} 解锁新陷阱吗？`,
			'解锁陷阱',
			{
				type: 'warning',
				confirmButtonText: '确认解锁',
				cancelButtonText: '取消'
			}
		)
	} catch {
		return
	}

	const res = await game.player_trap.unlock_trap()

	if (res.code !== 200) {
		message.error(res.msg || '解锁失败')
		return
	}

	message.success('解锁成功')
	await game.player_trap.update()
}

// 获取下一个解锁所需等级
const getNextUnlockLevel = () => {
	if (!trapConfig.value || !game.player_trap.data) return null

	// 计算已解锁的陷阱数量（不包括锁定的）
	const currentCount = game.player_trap.data.filter(t => !t.is_locked).length
	const config = trapConfig.value
	let accumulatedTraps = config.initial_trap_count

	if (config.level_rewards && Array.isArray(config.level_rewards)) {
		for (const reward of config.level_rewards) {
			accumulatedTraps += reward.traps
			if (currentCount < accumulatedTraps) {
				return reward.level
			}
		}
	}

	return null
}

// 检查占位符陷阱是否可以解锁（玩家等级是否满足）
const canUnlockPlaceholder = () => {
	const requiredLevel = getNextUnlockLevel()
	if (requiredLevel === null) return false
	return (game.player.data?.lv || 0) >= requiredLevel
}

// 获取占位符陷阱的解锁价格
const getPlaceholderPrice = () => {
	if (!trapConfig.value || !game.player_trap.data) return 0

	// 计算已解锁的陷阱数量（不包括锁定的）
	const currentCount = game.player_trap.data.filter(t => !t.is_locked).length
	const config = trapConfig.value
	let accumulatedTraps = config.initial_trap_count

	if (config.level_rewards) {
		for (const reward of config.level_rewards) {
			accumulatedTraps += reward.traps
			if (currentCount < accumulatedTraps) {
				return reward.unlock_price
			}
		}
	}

	return 0
}

// 获取占位符陷阱的货币类型
const getPlaceholderCurrency = () => {
	if (!trapConfig.value || !game.player_trap.data) return '金币'

	// 计算已解锁的陷阱数量（不包括锁定的）
	const currentCount = game.player_trap.data.filter(t => !t.is_locked).length
	const config = trapConfig.value
	let accumulatedTraps = config.initial_trap_count

	if (config.level_rewards) {
		for (const reward of config.level_rewards) {
			accumulatedTraps += reward.traps
			if (currentCount < accumulatedTraps) {
				const names = { 1: '金币', 2: '元宝', 3: '星币' }
				return names[reward.unlock_price_type] || '金币'
			}
		}
	}

	return '金币'
}

// 加载陷阱配置
const loadTrapConfig = async () => {
	// 如果配置已存在，直接使用
	if (game.player_trap.config) {
		trapConfig.value = game.player_trap.config;
		return;
	}

	// 否则从服务器加载
	const res = await game.player_trap.loadConfig();
	if (res.code === 200) {
		trapConfig.value = game.player_trap.config;
	} else {
		message.error('加载配置失败：' + res.msg);
	}
}

const startTimer = ()=>{
	timer.value = setInterval(() => {
		// 只更新当前时间，不强制刷新整个数组
		currentTime.value = Date.now()
	}, 1000);
}

const stopTimer = ()=>{
  if (timer.value) {
    clearInterval(timer.value);
  }
}

// 页面挂载时启动定时器
onMounted(async () => {
  startTimer()
  await loadTrapConfig()
})

onActivated(() => {
  startTimer()
})


// 组件卸载前清除定时器
onUnmounted(() => {
  stopTimer();
})

onDeactivated(() => {
  stopTimer()

})
</script>

<style scoped>

</style>
