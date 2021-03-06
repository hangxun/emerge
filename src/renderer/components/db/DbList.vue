<template>
	<div class="menu-box">
		<Menu theme="dark" @on-select="selectTable">
			<div class="search-box">
				<Input v-model="searchText" clearable placeholder="搜索..." style="width: auto">
				</Input>
				<Button slot="append" icon="ios-search"></Button>
			</div>
			<div @contextmenu="showContextMenu($event, item)" v-for="item in dbList" :key="item.id">
				<Submenu :name="item.id" v-show="dbTableMap[item.id] != null && dbTableMap[item.id].length > 0">
					<div class="menu_box" slot="title">
						<Icon type="ios-loading" size=18 class="load_db_icon" v-if="selectDbId === item.id && isRefresh"></Icon>
						<span class="iconfont icon-shujuku" v-else></span>
						<span class="name">{{ item.name }}</span>
					</div>
					<div @contextmenu.prevent.stop="">
						<MenuItem :name="item.id + '|' + tableItem.tableName" :key="index" v-for="(tableItem, index) in dbTableMap[item.id]">{{ tableItem.lowerCaseTableName }}</MenuItem>
					</div>
				</Submenu>
			</div>
		</Menu>
		<context-menu ref="contextMenu" :offset="{x: -35, y: 15}" class="left_menu" :show="isContextMenuShow" @update:show="(show) => isContextMenuShow = show">
			<div @click="refreshDb">刷新</div>
			<div @click="exportDb">导出</div>
			<div @click="deleteDb">删除</div>
		</context-menu>
	</div>
</template>

<script>
	import ContextMenu from '@/components/common/ContextMenu'
	import { getTableByMysql } from '@/util/JdbcUtil'
	import clone from 'clone'
	const { dialog } = require('electron').remote
	import path from 'path'
	import fs from "fs-extra"
	export default {
		name: 'table-menu',
		data () {
			return {
				searchText: null,
				isContextMenuShow: false,
				selectDbId: null,
				selectDb: null,
				isRefresh: false
			}
		},
		components: {
			ContextMenu
		},
		computed: {
			dbList () {
				return this.$store.state.db.dbList
			},
			dbTableMap () {
				if (this.searchText == null || this.searchText == '' || this.$store.state.db.dbTableMap == null) {
					return clone(this.$store.state.db.dbTableMap)
				}
				let dbTableMap = clone(this.$store.state.db.dbTableMap)
				let result = {}
				let keys = Object.keys(dbTableMap)
				let values = Object.values(dbTableMap)
				for(let i = 0; i < keys.length; i++) {
					if (values[i] != null) {
						let filterList = values[i].filter(item => {
							return item.lowerCaseTableName.indexOf(this.searchText) >= 0
						})
						result[keys[i]] = filterList
					}
				}
				return result
			},
		},
		methods: {
			showContextMenu ($event, db) {
				if (this.isRefresh) {
					return
				}
				this.isContextMenuShow = true
				this.selectDbId = db.id
				this.selectDb = db
				this.$refs.contextMenu.contextMenuHandler($event)
			},
			async refreshDb () {
				try {
					this.isRefresh = true
					this.isContextMenuShow = false
					const tableList = await getTableByMysql(this.selectDb, this.selectDb.mysqlConnectModel)
					await this.$store.dispatch('addDbTableList', {
						id: this.selectDbId,
						tableList: tableList
					})
					this.$Message.success('刷新数据库成功!')
				} catch (err) {
					console.error(err)
					this.$Message.error('刷新数据库失败: ' + err.message)
				} finally {
					this.isRefresh = false
				}
			},
			selectTable (name) {
				console.log(name)
				const dbId = name.split('|')[0]
				const tableName = name.split('|')[1]
				this.$emit('selectTable', {
					dbId: dbId,
					tableName: tableName
				})
			},
			deleteDb () {
				this.isContextMenuShow = false
				this.$Modal.confirm({
					title: '提示',
					content: '<p>确认删除此数据库连接?</p>',
					onOk: async () => {
						try {
							await this.$store.dispatch('removeDb', this.selectDbId)
							this.$Message.success('删除连接成功')
						} catch (err) {
							this.$Message.error('删除连接失败!')
						}
					}
				})
			},
			exportDb () {
				this.isContextMenuShow = false
				dialog.showSaveDialog({
					title: '选择数据库链接导出',
					defaultPath: path.normalize('./default.ec'),
					filters: [
						{ name: 'ec', extensions: ['ec'] }
					]
				}, (filePaths) => {
					try {
						if (filePaths) {
							console.log(this.selectDb)
							const outputDb = clone(this.selectDb.mysqlConnectModel)
							outputDb.name = this.selectDb.name
							fs.writeFileSync(path.normalize(filePaths), JSON.stringify(outputDb))
							this.$Message.success('导出数据库链接成功!')
						}
					} catch (e) {
						console.error(e)
						this.$Message.error('导出数据库链接失败!')
					}
				})
			}
		}
	}
</script>

<style lang="stylus" scoped>
.menu-box
	height 100vh
	width 240px
	background #515a6e
	position relative
	>>> li
		word-break break-all
	.search-box
		padding 10px
		display flex
		>>> input
			border-bottom-right-radius 0px
			border-top-right-radius 0px
		>>> button
			border-top-left-radius 0px
			border-bottom-left-radius 0px
	.menu_box
		height 25px
		line-height 25px
		.name
			vertical-align top
			padding-left 5px
	.left_menu
		position fixed
		background #fff
		border solid 1px rgba(0, 0, 0, .2)
		border-radius 3px
		z-index 999
		display none
		user-select none
		cursor pointer
		div
			width 100px
			height 20px
			line-height 20px
			font-size 14px
	.load_db_icon
		animation ani-demo-spin 1s linear infinite
	@keyframes ani-demo-spin
		from
			transform rotate(0deg)
		50%
			transform: rotate(180deg)
		to
			transform: rotate(360deg)
</style>
