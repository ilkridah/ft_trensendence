<template>
	<div class="social_buttons">
		<button class="spe" @click="handleClick(true, 1)" v-bind:class="{ 'focused': focusedButton === 1 }">Channels</button>
		<button class="spe" @click="handleClick(false, 2)" v-bind:class="{ 'focused': focusedButton === 2 }">Friendships</button>
		<ButtonAdd tooltip_text="Add Friend" icon="fa-solid fa-user-plus" @click="showModalFriend = true"></ButtonAdd>
		<ButtonAdd tooltip_text="Enter a channel" icon="fa-comment-medical" @click="showModal = true"></ButtonAdd>
		<!-- <ButtonAdd tooltip_text="Classement" icon="fa-trophy" @click="showLeaderboard = true"></ButtonAdd> -->
		<Teleport to="body">
			<ModalAddFriend :show="showModalFriend" @close="showModalFriend = false"></ModalAddFriend>
			<ModalAdd :show="showModal" @close="showModal = false" @newChannel="joinChannelForwarder"></ModalAdd>
			<KickUserModal :show="showKickModal" :channel_id="selectedChannel.id" @close="showKickModal = false;"
				@kick="notifyKickForwarder"></KickUserModal>
			<UnBanUserModal :show="showUnBanModal" :channel_id="selectedChannel.id" @close="showUnBanModal = false;">
			</UnBanUserModal>
			<BanUserModal :show="showBanModal" :channel_id="selectedChannel.id"
				@close="showBanModal = false;" @kick="notifyKickForwarder"></BanUserModal>
			<MuteModal :show="showMuteModal" :channel_id="selectedChannel.id" @close="showMuteModal = false" />
			<AddAdminModal :show="showAddAdminModal" :channel_id="selectedChannel.id" @close="showAddAdminModal = false;"/>
			<RemoveAdminModal :show="showRemoveAdminModal" :channel_id="selectedChannel.id" :sender="sender" @close='showRemoveAdminModal = false;' />
			<AddPasswordModal :show="showAddPasswordModal" :channel_id="selectedChannel.id" :sender="sender" @close="showAddPasswordModal = false;"
				@updateButton="updateButtonForwarder" />
		</Teleport>
	</div>
</template>
<script lang="ts">
import ButtonAdd from '../ButtonAdd.vue';
import ModalAddFriend from './ModalAddFriend.vue';
import ModalAdd from './ModalAdd.vue';
import UnBanUserModal from './UnBanUserModal.vue';
import AddAdminModal from './AddAdminModal.vue';
import RemoveAdminModal from './RemoveAdminModal.vue';
import AddPasswordModal from './AddPasswordModal.vue';
import BanUserModal from './BanUserModal.vue';

export default {
	components: {
		ButtonAdd,
		ModalAddFriend,
		ModalAdd,
		UnBanUserModal,
		AddAdminModal,
		RemoveAdminModal,
		AddPasswordModal,
		BanUserModal,
	},

	props: ['selectedChannel', 'sender'],

	data() {
		return {
			listView: false,
			showModalFriend: false,
			showModal: false,
			showUnBanModal: false,
			showBanModal: false,
			showMuteModal: false,
			showAddAdminModal: false,
			showRemoveAdminModal: false,
			showAddPasswordModal: false,
			showKickModal: false,
			focusedButton: 2,
		}
	},

	methods: {
		joinChannelForwarder(channel: any, channel_password: any) {
			this.$emit('joinChannel', channel, channel_password);
		},

		notifyKickForwarder(channel_id: number, User_id: number, ban: boolean) {
			this.$emit('kick', channel_id, User_id, ban);
		},

		updateButtonForwarder() {
			this.$emit('updateButton');
		},

		handleClick(listviewState: boolean, buttonId: number){
			this.listView = listviewState;
			this.focusedButton = buttonId;
		},
	},
}
</script>
<style>
.social_buttons {
	display: flex;
	position: relative;
	align-items: center;
	justify-content: center;
	gap: 1%;
	height: 7%;
	padding: 3px;
}

.spe {
	display: flex;
	align-items: center;
	justify-content: center;
	color: white;
	background-color: #0b2d39;
	height: 90%;
	flex-shrink: 0;
	width: 28%;
	overflow: hidden;
	border-radius: 20px;
	border: none;
	cursor: pointer;
	font-family: 'Poppins', sans-serif;
	font-weight: bold;
	font-size: clamp(0.5rem, 0.4118rem + 0.4706vw, 0.75rem);
}

.spe:hover {
	background-color: #2596be;
}

.spe.focused {
	background-color: #2596be;
}
</style>