<template>
	<view class="content">
		<top-bar class="top-bar">
			<navigator :url="`../userProfile/userProfile?id=${uID}`" slot="left" class="left">
				<image :src="imgUrl"></image>
			</navigator>
			<view slot="mid" class="mid">
				<image src="~@/static/images/index/火@2x.png"></image>
			</view>
			<view slot="right" class="right">
				<view class="search" @tap="toSearch">
					<image src="~@/static/images/index/search@2x.png"></image>
				</view>
				<view class="add" @tap="toBulidGroup">
					<image src="~@/static/images/index/add group@2x.png"></image>
				</view>
			</view>
		</top-bar>
		<view class="main">
			<view class="refresh" v-if="!refresh">
				<image src="../../static/images/index/refresh@2x.png"></image>
				<view class="ref-title">下拉刷新</view>
			</view>
			<view class="friends" v-if='requestLength>0' @tap='toFriendRequest'>
				<view class="friends-list">
					<view class="friends-list-l">
						<text class="tips">{{requestLength}}</text>
						<image src="~@/static/images/index/apply.svg"></image>
					</view>
					<view class="friends-list-r">
						<view class="top">
							<view class="name">好友申请</view>
							<view class="time">{{ showTime(requestTime)}}</view>
						</view>
						<view class="news">沧海茫茫，相聚便是缘。</view>
					</view>
				</view>
			</view>
			<view class="noone" v-if="noone">
				<image src="../../static/images/index/graffiti@2x.png" mode="aspectFill"></image>
				<view class="no-title">你还没有好友呢~</view>
				<view class="search-btn" @tap="toSearch">搜索好友</view>
			</view>
			<view class="friends" v-if="friendsData.length > 0">
				<view class="friends-list" v-for="(item,index) in friendsData" :key="index" @tap='toChatRomm(item)'>
					<view class="friends-list-l">
						<text class="tips" v-if="item.tips>0">{{ item.tips }}</text>
						<image :src="item.imgUrl"></image>
					</view>
					<view class="friends-list-r">
						<view class="top">
							<view class="name">{{ item.name }}</view>
							<view class="time">{{ showTime(item.lastTime) }}</view>
						</view>
						<view class="news">{{ item.message }}</view>
					</view>
				</view>
			</view>
			<view class="friends" v-if="groupsData.length > 0">
				<view class="friends-list" v-for="(item,index) in groupsData" :key="index" @tap='toChatRomm(item)'>
					<view class="friends-list-l">
						<text class="tips" v-if="item.tips>0">{{ item.tips }}</text>
						<image :src="item.imgUrl"></image>
					</view>
					<view class="friends-list-r">
						<view class="top">
							<view class="name">{{ item.name }}</view>
							<view class="time">{{ showTime(item.lastTime) }}</view>
						</view>
						<view class="news">{{ item.message }}</view>
					</view>
				</view>
			</view>
		</view>
	</view>
</template>

<script>
	import topBar from "@/components/topBar.vue"

	import datas from "@/common/datas";
	import myfun from "@/common/myfun";

	export default {
		data() {
			return {
				friendsData: [], // 好友列表
				groupsData: [], // 群列表
				uID: '',
				imgUrl: '',
				token: '',
				myName: '',
				requestLength: 0, // 好友申请数
				requestTime: '', // 最后申请时间
				messageTypesMap: {
					1: '[图片]',
					2: '[语音]',
					3: '[位置]',
				}, // 消息类型判断
				refresh: false, // 下拉刷新状态
				noone: false, // 是否为好友
			};
		},
		components: {
			topBar,
		},
		onLoad() {
			this.getStorages()
			this.getFriendsList()
			this.getGroupList()
			// this.firendApply()
			uni.startPullDownRefresh();
			this.socketJoin(this.uID)
			this.receiveSocketMessage()
			this.groupSocket();
		},
		// 下拉刷新事件
		onPullDownRefresh() {
			this.friendsData = []
			this.groupsData = []
			this.getStorages()
			this.getFriendsList()
			this.getGroupList()
			this.firendApply()
			setTimeout(function() {
				uni.stopPullDownRefresh();
			}, 1000);
		},
		methods: {
			// 获取缓存信息
			getStorages: function() {
				try {
					const value = uni.getStorageSync('user');
					if (value) {
						// console.log(value);
						this.uID = value.id
						this.imgUrl = this.serverUrl + value.imgUrl
						this.token = value.token
						this.myName = value.userName
					} else {
						// 如果没有用户缓存，则跳转到登陆页面
						uni.navigateTo({
							url: '../login/login'
						})
					}
				} catch (e) {
					// error
				}
			},
			showTime: function(date) {
				return myfun.dateTime(date);
			},
			// 去搜索页面
			toSearch: function() {
				uni.navigateTo({
					url: '../search/search'
				});
			},
			// 去 建群页面
			toBulidGroup: function() {
				uni.navigateTo({
					url: '../buildGroup/buildGroup'
				});
			},

			//  获取用户信息 渲染主页
			getFriendsList: function() {
				uni.request({
					url: this.serverUrl + "/index/getFriendList",
					data: {
						uID: this.uID,
						state: 0,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {
						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
							this.refresh = true
							// result 为 好友的信息
							let result = data.data.result
							this.noone = false
							if (result.length === 0) {
								this.noone = true
							}
							if (result.length > 0) {
								for (let i = 0; i < result.length; i++) {
									result[i].imgUrl = this.serverUrl + result[i].imgUrl
									if (result[i].markName) {
										result[i].name = result[i].markName
									}
									this.friendsData.push(result[i])
								}
								// this.friendAndGroupSort(this.friendsData) 
							}
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},
			// 获取群列表
			getGroupList: function() {
				uni.request({
					url: this.serverUrl + "/index/getGroupList",
					data: {
						uID: this.uID,
						state: 0,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {
						this.refresh = true
						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
							// result 为 群的信息
							let result = data.data.result
							if (result.length > 0) {
								for (let i = 0; i < result.length; i++) {
									result[i].imgUrl = this.serverUrl + result[i].imgUrl
									this.groupsData.push(result[i])
									this.socket.emit('group',result[i].id) 
								}
							}
							// this.friendAndGroupSort(this.groupsData)
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},
			// 群与好友的排序
			friendAndGroupSort: function(e) {
				if (e.length > 0) {
					// 数组拼接
					this.friendsData = this.friendsData.concat(this.groupsData)
					//排序
					this.friendsData = myfun.sort(this.friendsData, 'lastTime', 0)
				}
			},

			// 获取好友最后一条消息  (废弃部分， 先保存这，以防万一)
			getFriendLastMessage: function(arr, i) {
				uni.request({
					url: this.serverUrl + "/index/getLastMessage",
					data: {
						uID: this.uID,
						fID: arr[i].id,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {
						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
							let result = data.data.result
							if (result.messageTypes != 0) {
								// 如果是 图片，音频 ，地图 就给相应的 字符串
								result.message = this.messageTypesMap[result.messageTypes]
							}
							let friendInfo = arr[i]
							friendInfo.message = result.message
							arr.splice(i, 1, friendInfo)
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},

			// 获取好友消息未读数   (废弃部分， 先保存这，以防万一)
			getUnReadMessage: function(arr, i) {
				uni.request({
					url: this.serverUrl + "/index/unReadMessage",
					data: {
						uID: this.uID,
						fID: arr[i].id,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {

						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
							let result = data.data.result
							let friendInfo = arr[i]
							friendInfo.tips = result
							arr.splice(i, 1, friendInfo)
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},

			// 好友申请列表
			firendApply: function() {
				uni.request({
					url: this.serverUrl + "/index/getFriendList",
					data: {
						uID: this.uID,
						state: 1,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {
						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
							let result = data.data.result
							// 获取好友申请数
							this.requestLength = result.length

							if (this.requestLength.length > 0 || result.length > 0) {
								this.requestTime = result[result.length - 1].lastTime
								// this.requestTime = result[0].lastTime
								// for (let i = 0; i < this.requestLength.length; i++) {
								// 	if (this.requestTime < result[i].lastTime) {
								// 		this.requestTime = result[i].lastTime
								// 	}
								// }
							}
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},
			

			// socket模块
			// 用户登录socket注册
			socketJoin: function(uID) {
				this.socket.emit('login', uID)
			},
			// socket 聊天数据接收  用户一对一消息发送
			receiveSocketMessage: function() {
				this.socket.on('message', (msg, fromID) => {
					let indexMessage = ''
					if (msg.messageTypes == 0) {
						indexMessage = msg.message
					}
					if (msg.messageTypes != 0) {
						// 如果是 图片，音频 ，地图 就给相应的 字符串
						indexMessage = this.messageTypesMap[msg.messageTypes]
					}
					// 对比到对应项， 修改
					for (let i = 0; i < this.friendsData.length; i++) {
						if (this.friendsData[i].id == fromID) {
							let item = this.friendsData[i]
							item.tips++ // 未读消息数
							// 更新时间
							item.lastTime = new Date()
							item.message = indexMessage
							// 删除原来的消息
							this.friendsData.splice(i, 1)
							// 把新消息插入到最顶部
							this.friendsData.unshift(item)
						}
					}
				})
			},
			groupSocket:function(){
				this.socket.on('groupMessage',(msg,fromID,groupID,name)=>{
					// console.log(groupID,name,this.myName)
					let indexMessage = ''
					if (msg.messageTypes == 0) {
						indexMessage = msg.message
					}
					if (msg.messageTypes != 0) {
						// 如果是 图片，音频 ，地图 就给相应的 字符串
						indexMessage = this.messageTypesMap[msg.messageTypes]
					}
					// 对比到对应项， 修改
					for (let i = 0; i < this.groupsData.length; i++) {
						if (this.groupsData[i].id == groupID) {
							let item = this.groupsData[i]
							item.tips++ // 未读消息数
							// 更新时间
							item.lastTime = new Date()
							if(fromID == this.uID){
								item.message =  indexMessage
							}else{
								item.message = name + " : "+ indexMessage
							}
							// 删除原来的消息
							this.groupsData.splice(i, 1)
							// 把新消息插入到最顶部
							this.groupsData.unshift(item)
						}
					}
				})
			},
			toFriendRequest: function() {
				uni.navigateTo({
					url: '../friendRequest/friendRequest'
				});
			},
			// 好友消息标记已读
			readedFriendMessage:function(uID,fID){
				uni.request({
					url: this.serverUrl + "/index/updateMessage",
					data: {
						uID:uID,
						fID:fID,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {
						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},
			readedGroupMessage:function(gID){
				uni.request({
					url: this.serverUrl + "/index/updateGroupMessage",
					data: {
						gID:gID,
						token: this.token,
					},
					method: 'POST',
					success: (data) => {
						let status = data.data.status
						// 访问后端成功
						if (status === 200) {
						} else if (status === 500) {
							uni.showToast({
								title: '服务器出错了！',
								icon: 'none',
								duration: 1500
							})
						} else if (status === 300) {
							// token 过期了 需要重新登录再次生成token
							uni.navigateTo({
								url: '../login/login?name=' + this.myName
							});
						}
					}
				})
			},
			toChatRomm: function(data) {
				// console.log('进入')
				// console.log(data)
				// console.log(this.uID) // 自己
				// // data.tips = 0
				this.readedFriendMessage(data.id,this.uID)
				this.readedGroupMessage(data.id)
				uni.navigateTo({
					url: `../chatRoom/chatRoom?id=${data.id}&name=${data.name}&img=${data.imgUrl}&type=${data.type}`
				});
			},
		}
	};
</script>

<style lang="scss" scoped>
	.content {
		display: flex;
		flex-direction: column;
		align-items: center;
		justify-content: center;
		padding-top: var(--status-bar-height);

		.top-bar {
			border-bottom: 1px solid $uni-border-color;

			.left {
				padding-top: 10rpx;

				image {
					width: 68rpx;
					height: 68rpx;
					border-radius: 16rpx;
				}
			}

			.mid {

				image {
					width: 88rpx;
					height: 42rpx;
				}
			}

			.right {
				display: flex;
				align-items: center; // 垂直居中

				.search {
					// border: 1px solid red;
					flex: 1;

					image {
						width: 52rpx;
						height: 52rpx;
					}
				}

				.add {
					// border: 1px solid red;
					flex: 1;

					image {
						width: 48rpx;
						height: 48rpx;
					}
				}

			}
		}

		.main {
			padding: 104rpx $uni-spacing-row-sm 0; // 104rpx

			.refresh {
				text-align: center;
				padding-top: 480rpx;

				image {
					margin-right: 18rpx;
					width: 32rpx;
					height: 32rpx;
				}

				.ref-title {
					padding-top: 30rpx;
					font-size: $uni-font-size-base;
					font-family: PingFangSC-Regular;
					color: rgba(39, 40, 50, 0.40);
					line-height: 40rpx;
				}
			}

			.noone {
				text-align: center;
				padding-top: 400rpx;

				image {
					height: 250rpx;
					width: 158rpx;
				}

				.no-title {
					padding: 32rpx 0;
					font-family: PingFangSC-Regular;
					font-size: $uni-font-size-base;
					color: rgba(39, 40, 50, 0.40);
					line-height: 40rpx;
				}

				.search-btn {
					display: inline-block;
					width: 240rpx;
					height: 96rpx;
					background: $uni-color-primary;
					box-shadow: 0 52rpx 36rpx -32rpx rgba(255, 228, 49, 0.40);
					border-radius: 48rpx;
					font-family: PingFangSC-Regular;
					font-size: $uni-font-size-base;
					color: $uni-text-color;
					line-height: 96rpx;
				}
			}

			.friends-list {
				display: flex;
				align-items: center;
				height: 96rpx;
				padding: 16rpx 0;

				// border: 1px solid red;
				&:active {
					background-color: $uni-bg-color-grey;
				}

				.friends-list-l {
					// flex: 1;   // 部给 左边设置 flex 让它自动铺满
					position: relative;

					.tips {
						position: absolute;
						z-index: 10;
						top: -6rpx;
						left: 68rpx;
						min-width: 24rpx;
						padding: 0 6rpx;
						height: 36rpx;
						line-height: 36rpx;
						text-align: center;
						color: $uni-text-color-inverse;
						font-size: $uni-font-size-sm;
						background-color: $uni-color-error;
						border-radius: 18rpx;
					}

					image {
						width: 96rpx;
						height: 96rpx;
						border-radius: $uni-border-radius-base;
						background-color: $uni-color-primary;
					}
					
				}

				.friends-list-r {
					flex: 1;
					padding-left: $uni-spacing-col-base;

					.top {
						display: flex;
						align-items: center;
						justify-content: space-between;

						.name {
							flex: 1;
							font-size: 36rpx;
							font-weight: 400;
							color: $uni-text-color;
							line-height: 50rpx;
						}

						.time {
							font-size: $uni-font-size-sm;
							color: $uni-text-color-disable;
							line-height: 34rpx;
						}
					}

					.news {
						display: -webkit-box;
						-webkit-box-orient: vertical;
						-webkit-line-clamp: 1;
						overflow: hidden;
						min-width: 580rpx;
						font-size: $uni-font-size-base;
						color: $uni-text-color-grey;
						// padding-top: 10rpx;
						line-height: 40rpx;
					}
				}
			}
		}


	}
</style>
