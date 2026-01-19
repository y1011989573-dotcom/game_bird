<template>

	<div class="card">
		【  <a class="text-blue-500!" @click="MapInfoRef?.show(game.player.data?.game_map)"> {{ game.player.data?.game_map.nickname || '未知地图' }}</a>】
		<el-button type="primary" size="small" @click="MapSelectorRef?.show()" class="ml-2">
			升级地图
		</el-button>
	</div>

	<MapInfo ref="MapInfoRef" />
	<MapSelector ref="MapSelectorRef" />
</template>


<script setup>
import {inject, ref} from "vue";
import {ElButton} from "element-plus";

import MapInfo from "@/components/map/MapInfo.vue";
import MapSelector from "@/components/MapSelector.vue";

const game = inject('game')
const MapInfoRef = ref(null)
const MapSelectorRef = ref(null)


</script>

<style scoped>

</style>
