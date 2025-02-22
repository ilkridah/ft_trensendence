<template>
	<div class="chat" @click="showMenu = false" v-if="selectedChannel.name">
		<Teleport to="body">
			<ModalChat :show="modalChat" @close="modalChat = false" :connected_user="connected_user" :Friend_id="Friend_id"
				:username="username" :selectedChannel="selectedChannel" />
		</Teleport>
		<div class="top_chat_container">
			<div class="channel_name" v-if="selectedChannel.name">{{ channelName }}</div>
			<ChannelOptionsMenu v-if="selectedChannel.name && selectedChannel.name.length <= 16"
				:isOwner="selectedChannel.owner === sender.id" :isAdmin="selectedChannel.admins.includes(sender.id)"
				:showMenu="showMenu" :isProtected="selectedChannel.is_protected" @openMenu="toggleMenu"
				@leave_Channel="leave_Channel(sender.id)" @delete_Password="delete_Password()"
				@displayChannelOption="emitToModalManager"></ChannelOptionsMenu>
		</div>
		<div class="message_box">
			<div v-for="(msg, index) in selectedChannel.messages" class="single_message"
				:class="sender.id === msg.sender ? 'sent' : 'received'">
				<img draggable="false" v-if="sender.id !== msg.sender" alt="avatar" @click="showChatModal(msg)"
					:src="msg.sender_img">
				<div class="msg_txt_box">
					<span v-if="sender.id !== msg.sender" class="sender_name">{{ msg.sender_name }}</span>
					<div :class="sender.id === msg.sender ? 'sent_txt' : 'received_txt'" :ref="`message-${index}`"
						class="message">{{ msg.text }}</div>
				</div>
			</div>
		</div>
		<div class="send_container" v-if="selectedChannel.name">
			<form v-on:submit.prevent="sendMessage">
				<div class="sendbox">
					<input v-if="!selectedChannel.muted" type="text" maxlength="280" v-model="message"
						:placeholder="'send a message ' + inputString">
					<input v-else disabled type="text" placeholder="You have been mute from this channel.">
					<button type="button" @click="sendMessage()">
						<font-awesome-icon icon="fa-solid fa-paper-plane" />
					</button>
				</div>
			</form>
		</div>
	</div>

</template>

<script lang="ts">

import { SocketService } from '@/services/SocketService';
import ModalChat from './ModalChat.vue';
import { useNotification } from '@kyvg/vue3-notification';
import router from '@/router';
import ChannelOptionsMenu from './ChannelOptionsMenu.vue'

export default {
	components: {
		ModalChat,
		ChannelOptionsMenu,
	},

	data() {
		return {
			message: '' as string,
			modalChat: false as boolean,
			connected_user: -1 as number,
			Friend_id: -1 as number,
			username: '' as string,
			showMenu: false,
			channelName: "",
			inputString: "",
			lastUpdate: 0,
		};
	},

	async updated() {
		if (this.selectedChannel.messages && !this.lastUpdate) {
			const lastMessage = this.$refs[`message-${this.selectedChannel.messages.length - 1}`] as any;
			if (!lastMessage)
				return;
			setTimeout(() => {
				lastMessage[0].scrollIntoView();
			}, 20);
		}
	},

	mounted() {
		this.connected_user = SocketService.getUser;
		this.channelName = "";
		this.inputString = "";
	},

	props: ['selectedChannel', 'sender'],

	watch: {
		selectedChannel: {
			deep: true,
			async handler() {
				if (this.selectedChannel.messages) {
					await this.getChanName();
					const lastMessage = this.$refs[`message-${this.selectedChannel.messages.length - 1}`] as any;
					if (lastMessage) {
						lastMessage[0].scrollIntoView({ behavior: !this.lastUpdate || ((Date.now() - this.lastUpdate) < 60) ? 'instant' : 'smooth' });
						this.lastUpdate = Date.now();
					}
				}
			}
		}
	},

	methods: {

		navigateToProfile(name: any) {
      		router.push({ path: '/profile', query: { user: name } });
      		this.$emit('close');
    	},

		toggleMenu() {
			this.showMenu = !this.showMenu;
		},
		emitToModalManager(modalToggled: string) {
			this.$emit('displayChannelOption', modalToggled);
		},
		sendMessage() {
			if (SocketService.getStatus && this.message) {
				if (/^\s+$/i.test(this.message))
					return;
				this.message = this.message.trimStart().trimEnd();
				const data = {
					channel_id: this.selectedChannel.id,
					text: this.message,
					sender: this.sender.id,
					sender_name: this.sender.name,
					sender_img: this.sender.img,
				};
				SocketService.getInstance.emit('message', data);
				this.message = '';
			}
		},

		async getChanName() {
			if (this.selectedChannel.name.length <= 16) {
				this.channelName = this.selectedChannel.name;
				this.inputString = "in " + this.selectedChannel.name;
				return;
			}
			const json = await (await fetch("http://" + import.meta.env.VITE_HOST + ":3000/chat/" + this.selectedChannel.id + '/getUsers',
				{ method: "get", credentials: "include" })).json();
			if (json[0].id === SocketService.getUser.id) {
				this.channelName = json[1].name;
				this.inputString = "à @" + json[1].name;
			}
			else {
				this.channelName = json[0].name;
				this.inputString = "à @" + json[0].name;
			}
		},

		async leave_Channel(id: number) {
			const response = await fetch('http://' + import.meta.env.VITE_HOST + ':3000/user/' + id + '/channels/' + this.selectedChannel.id + '/remove', {
				credentials: 'include',
				method: 'POST'
			});
			const response_json = await response.json();
			if (response_json['ok'])
				this.$emit('removeChannel', id);
			this.lastUpdate = 0;
		},

		showChatModal(msg: any) {
			if (msg.sender === this.sender.id) {
				this.navigateToProfile(msg.send_name);
				return;
			}
			this.Friend_id = msg.sender;
			this.username = msg.sender_name;
			this.modalChat = true;
		},

		async delete_Password() {
			const response = (await (await fetch('http://' + import.meta.env.VITE_HOST + ':3000/chat/' + this.$props.selectedChannel.id + '/' + this.$props.sender.id + '/delete_Password', {
				credentials: 'include',
				method: 'POST',
			})).json());
			const notif = useNotification();
			if (!response['ok']) {
				notif.notify({
					title: 'Error',
					text: response['message'],
					type: 'error',
					group: 'notif-center',
				});
				return;
			}
			notif.notify({
				text: response['message'],
				type: 'success',
				group: 'notif-center',
			});
			this.$props.selectedChannel.is_protected = false;
		},
	}
}
</script>

<style scoped>
input:placeholder-shown {
	text-overflow: ellipsis;
}

.empty_chat h1 {
	font-size: 1.35em;
}

.paragraphs {
	width: 95%;
	height: 80%;
	display: flex;
	gap: 10px;
	flex-direction: column;
	align-items: center;
}

.empty_chat {
	font-size: clamp(0.625rem, 0.3365rem + 0.9231vw, 1rem);
	justify-content: center;
	text-align: justify;
	display: flex;
	flex-direction: column;
	align-items: center;
	width: 60%;
	height: 100%;
	background-color: white;
	border: 3px solid #515151;
	border-radius: 10px;

}

.creators_names {
	align-self: end;
	margin-right: 10px;
}

.chat {
	display: flex;
	flex-direction: column;
	align-items: center;
	justify-content: space-between;
	background: white;
	width: 60%;
	height: 100%;
	border: 3px solid #515151;
	border-radius: 10px;

}

.top_chat_container {
	display: flex;
	align-items: center;
	justify-content: center;
	width: 100%;
	height: 40px;
	position: relative;
	border-bottom: 3px solid #515151;
	border-radius: 0 0 10px 10px;

}
.channel_name {
	width: 100%;
	padding-left: 10px;
	font-weight: bold;
	font-size: clamp(1rem, 0.4231rem + 1.8462vw, 1.75rem);
}

.message {
	text-align: left;
	word-wrap: break-word;
	padding: 10px;
	max-width: 100%;
	font-size: clamp(0.75rem, 0.5577rem + 0.6154vw, 1rem);
	overflow-wrap: anywhere;
	white-space: break-spaces
}

.msg_txt_box {
	display: flex;
	flex-direction: column;
	align-items: start;
	max-width: 70%;
}

.sent_txt {
	background: rgb(187, 214, 255);
	border-radius: 20px 20px 0px 20px;
}

.received_txt {
	background-color: #d4eefd;
	border-radius: 20px 20px 20px 0px;
}

.message_box {
	display: flex;
	flex-direction: column;
	width: 95%;
	margin-bottom: 10px;
	height: 100%;
	max-height: 100%;
	overflow-y: scroll;
	scrollbar-width: none;
	min-height: min-content;
}

.message_box .single_message:first-child {
	margin-top: auto;
}

::-webkit-scrollbar {
	width: 0;
	background-color: transparent;
}

.single_message {
	display: flex;
	align-items: end;
	margin-top: 10px;
	overflow-wrap: break-word;
}

.single_message img {
	padding-left: 5px;
	padding-right: 5px;
	width: 40px;
	height: 40px;
	border-radius: 50%;
}

.single_message img:hover {
	cursor: pointer;
	opacity: 0.5;
}

.sender_name {
	font-size: 10px;
	margin-left: 10px;
}

.sent {
	text-align: right;
	display: flex;
	flex-direction: row-reverse;
}

.send_container {
	display: flex;
	flex-direction: column;
	align-items: center;
	width: 100%;
	margin-bottom: 10px;
}

.send_container form {
	width: 95%;
	border-radius: 10px;
}

.sendbox {
	width: 100%;
	text-align: center;
	display: flex;
	flex-direction: row;
	justify-content: center;
	align-items: center;
	gap: 10px;
}

.sendbox input {
	width: 100%;
	height: 50px;
	border: none;
	border-radius: 500px;
	background-color: #ebebeb;
	padding-left: 15px;
	font-size: clamp(0.6875rem, 0.4471rem + 0.7692vw, 1rem);
}

.sendbox input:focus {
	outline: none;
}

.sendbox button {
	padding: 20px;
	text-align: center;
	color: white;
	font-weight: bold;
	border: none;
	border-radius: 500px;
	background-color: #036280;
	cursor: pointer;
}

.sendbox button:hover {
	background-color: #004d64;
}
</style>