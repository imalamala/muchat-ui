<template>
	<el-container class="ibox">
		<el-aside width="80px" class="navi-bar">
			<div class="user-head-image">
				<head-image :url="$store.state.userStore.userInfo.headImageThumb" :size="60" @click.native="showSettingDialog=true">
				</head-image>
			</div>

			<el-menu background-color="#333333" text-color="#ddd" style="margin-top: 10px;">
				<el-menu-item title="聊天">
					<router-link v-bind:to="'/home/chat'">
            <i class="operate-icon icon-message-list" />
						<div v-show="unreadCount>0" class="unread-text">{{unreadCount}}</div>
					</router-link>
				</el-menu-item>
				<el-menu-item title="好友">
					<router-link v-bind:to="'/home/friend'">
            <i class="operate-icon icon-friend-list" />
					</router-link>
				</el-menu-item>
				<el-menu-item title="群聊">
					<router-link v-bind:to="'/home/group'">
            <i class="operate-icon icon-group-list" />
					</router-link>
				</el-menu-item>

				<el-menu-item title="Gitee">

          <div class="operate-icons">
            <a
                href="https://github.com/pisces-hub/muchat"
                target="_blank"
                title="Github"
            >
              <i class="operate-icon icon-github" />
            </a>
          </div>

				</el-menu-item>
			</el-menu>
			<div class="exit-box" @click="handleExit()" title="退出">
        <i class="operate-icon icon-exit" />
			</div>
		</el-aside>
		<el-main class="content-box">
			<router-view></router-view>
		</el-main>
		<setting :visible="showSettingDialog" @close="closeSetting()"></setting>
		<user-info v-show="uiStore.userInfo.show" :pos="uiStore.userInfo.pos" :user="uiStore.userInfo.user" @close="$store.commit('closeUserInfoBox')"></user-info>
		<full-image :visible="uiStore.fullImage.show" :url="uiStore.fullImage.url" @close="$store.commit('closeFullImageBox')"></full-image>
		<chat-private-video ref="privateVideo" :visible="uiStore.chatPrivateVideo.show" 
		:friend="uiStore.chatPrivateVideo.friend" 
		:master="uiStore.chatPrivateVideo.master"
		:offer="uiStore.chatPrivateVideo.offer"
		@close="$store.commit('closeChatPrivateVideoBox')" >
		</chat-private-video>
		<chat-video-acceptor ref="videoAcceptor"
		v-show="uiStore.videoAcceptor.show"
		:friend="uiStore.videoAcceptor.friend" 
		@close="$store.commit('closeVideoAcceptorBox')" >
		</chat-video-acceptor>
	</el-container>
</template>

<script>
	import HeadImage from '../components/common/HeadImage.vue';
	import Setting from '../components/setting/Setting.vue';
	import UserInfo from '../components/common/UserInfo.vue';
	import FullImage from '../components/common/FullImage.vue';
	import ChatPrivateVideo from '../components/chat/ChatPrivateVideo.vue';
	import ChatVideoAcceptor from '../components/chat/ChatVideoAcceptor.vue';
	
	
	export default {
		components: {
			HeadImage,
			Setting,
			UserInfo,
			FullImage,
			ChatPrivateVideo,
			ChatVideoAcceptor
		},
		data() {
			return {
				showSettingDialog: false,
				
			}
		},
		methods: {
			init(userInfo) {
				this.$store.commit("setUserInfo", userInfo);
				this.$store.commit("setUserState", this.$enums.USER_STATE.FREE);
				this.$store.commit("initStore");
				this.$wsApi.createWebSocket(userInfo.id);
				this.$wsApi.onopen(() => {
					this.pullUnreadMessage();
				});
				this.$wsApi.onmessage((cmd,msgInfo) => {
					if (cmd == 2) {
						// 异地登录，强制下线
						this.$message.error("您已在其他地方登陆，将被强制下线");
						setTimeout(() => {
							location.href = "/";
						}, 1000)

					} else if (cmd == 3) {
						// 插入私聊消息
						this.handlePrivateMessage(msgInfo);
					} else if (cmd == 4) {
						// 插入群聊消息
						this.handleGroupMessage(msgInfo);
					} 
				})
			},
			pullUnreadMessage() {
				// 拉取未读私聊消息
				this.$http({
					url: "/message/private/pullUnreadMessage",
					method: 'post'
				});
				// 拉取未读群聊消息
				this.$http({
					url: "/message/group/pullUnreadMessage",
					method: 'post'
				});
			},
			handlePrivateMessage(msg) {
				// 好友列表存在好友信息，直接插入私聊消息
				let friend = this.$store.state.friendStore.friends.find((f) => f.id == msg.sendId);
				if (friend) {
					this.insertPrivateMessage(friend, msg);
					return;
				}
				// 好友列表不存在好友信息，则发请求获取好友信息
				this.$http({
					url: `/friend/find/${msg.sendId}`,
					method: 'get'
				}).then((friend) => {
					this.insertPrivateMessage(friend, msg);
					this.$store.commit("addFriend", friend);
				})
			},
			insertPrivateMessage(friend, msg) {
				// webrtc 信令
				if(msg.type >= this.$enums.MESSAGE_TYPE.RTC_CALL 
				&& msg.type <= this.$enums.MESSAGE_TYPE.RTC_CANDIDATE){
					// 呼叫
					console.log(msg)
					if(msg.type == this.$enums.MESSAGE_TYPE.RTC_CALL
					|| msg.type == this.$enums.MESSAGE_TYPE.RTC_CANCEL){
						this.$store.commit("showVideoAcceptorBox",friend);
						this.$refs.videoAcceptor.handleMessage(msg)
					}else {
						this.$refs.privateVideo.handleMessage(msg)
					}
					return ;
				}
				
				let chatInfo = {
					type: 'PRIVATE',
					targetId: friend.id,
					showName: friend.nickName,
					headImage: friend.headImage
				};
				// 打开会话
				this.$store.commit("openChat", chatInfo);
				// 插入消息
				this.$store.commit("insertMessage", msg);
				// 播放提示音
				this.playAudioTip();
			},
			handleGroupMessage(msg) {
				// 群聊缓存存在，直接插入群聊消息
				let group = this.$store.state.groupStore.groups.find((g) => g.id == msg.groupId);
				if (group) {
					this.insertGroupMessage(group, msg);
					return;
				}
				// 群聊缓存存在，直接插入群聊消息
				this.$http({
					url: `/group/find/${msg.groupId}`,
					method: 'get'
				}).then((group) => {
					this.insertGroupMessage(group, msg);
					this.$store.commit("addGroup", group);
				})
			},
			insertGroupMessage(group, msg) {
				let chatInfo = {
					type: 'GROUP',
					targetId: group.id,
					showName: group.remark,
					headImage: group.headImageThumb
				};
				// 打开会话
				this.$store.commit("openChat", chatInfo);
				// 插入消息
				this.$store.commit("insertMessage", msg);
				// 播放提示音
				this.playAudioTip();
			},
			handleExit() {
        this.$confirm('确认要退出登录吗?', '提示', {
          confirmButtonText: '确定',
          cancelButtonText: '取消',
          type: 'warning'
        }).then(() => {
          this.$wsApi.closeWebSocket();
          sessionStorage.removeItem("token");
          location.href = "/";
        });

			},
			playAudioTip(){
				let audio = new Audio();
				let url = require(`@/assets/audio/tip.wav`);
				audio.src = url;
				audio.play();
			},
			showSetting() {
				this.showSettingDialog = true;
			},
			closeSetting() {
				this.showSettingDialog = false;
			}
		},
		computed: {
			uiStore() {
				return this.$store.state.uiStore;
			},
			unreadCount() {
				let unreadCount = 0;
				let chats = this.$store.state.chatStore.chats;
				chats.forEach((chat) => {
					unreadCount += chat.unreadCount
				});
				return unreadCount;
			}
		},
		watch: {
			unreadCount: {
				handler(newCount, oldCount) {
					let tip = newCount > 0 ? `${newCount}条未读` : "";
					this.$elm.setTitleTip(tip);
				},
				immediate: true
			}
		},
		mounted() {
			this.$http({
				url: "/user/self",
				methods: 'get'
			}).then((userInfo) => {
				this.init(userInfo);
			})
		},
		unmounted() {
			this.$wsApi.closeWebSocket();
		}
	}
</script>

<style scoped lang="scss">
	.navi-bar {
		background: #333333;
		padding: 10px;
		padding-top: 20px;

		.user-head-image {
			position: relative;
			width: 50px;
			height: 50px;
		}

		.el-menu {
			border: none;
			flex: 1;

			.el-menu-item {
				margin: 25px 0;

				.router-link-exact-active span {
					color: white !important;
				}



				span {
					font-size: 24px !important;
					color: #aaaaaa;

					&:hover {
						color: white !important;
					}
				}

				.unread-text {
					position: absolute;
					line-height: 20px;
					background-color: #f56c6c;
					left: 36px;
					top: 7px;
					color: white;
					border-radius: 30px;
					padding: 0 5px;
					font-size: 10px;
					text-align: center;
					white-space: nowrap;
					border: 1px solid #f1e5e5;
				}
			}
		}

		.exit-box {
			position: absolute;
			width: 60px;
			bottom: 40px;
			color: #aaaaaa;
			font-size: 24px;
			text-align: center;
			cursor: pointer;

			&:hover {
				color: white !important;
			}
		}
	}

	.content-box {
		padding: 0;
		background-color: #f3e9ee;
		color: #333;
		text-align: center;

	}

  .ibox{
    position: relative;
    flex-direction: row;
    width: 80vw;
    height: 80vh;
    margin: 0 auto;
    margin-top: 10vh;
    border-radius: 16px;
  }
  .icon-github {
    background-image: url('~@/assets/icons/github.png');
  }
  .icon-friend-list {
    background-image: url('~@/assets/icons/friend-list.png');
  }
  .icon-message-list {
    background-image: url('~@/assets/icons/message-list.png');
  }
  .icon-group-list {
    background-image: url('~@/assets/icons/group-list.png');
  }
  .icon-exit {
    background-image: url('~@/assets/icons/exit.png');
  }
  .operate-icon {
    display: inline-block;
    width: 28px;
    height: 28px;
    background-repeat: no-repeat;
    background-position: center;
    background-size: cover;
  }

</style>
