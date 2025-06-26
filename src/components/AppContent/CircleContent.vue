<!--
  - SPDX-FileCopyrightText: 2018 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->

<template>
	<AppContent>
		<EmptyContent v-if="!circle" :name="t('contacts', 'Please select a team')">
			<template #icon>
				<AccountGroup :size="20" />
			</template>
		</EmptyContent>

		<EmptyContent v-else-if="loading" class="empty-content" :name="t('contacts', 'Loading team…')">
			<template #icon>
				<IconLoading :size="20" />
			</template>
		</EmptyContent>

		<CircleDetails v-else 
			:circle="circle" 
			:project="project" 
			:file-tree="fileTree" />
	</AppContent>
</template>
<script>
import { showError } from '@nextcloud/dialogs'
import {
	NcAppContent as AppContent,
	NcEmptyContent as EmptyContent,
	NcLoadingIcon as IconLoading,
	isMobile,
} from '@nextcloud/vue'
import AccountGroup from 'vue-material-design-icons/AccountGroup.vue'
import CircleDetails from '../CircleDetails.vue'
import RouterMixin from '../../mixins/RouterMixin.js'
import { generateUrl } from '@nextcloud/router'
import axios from 'axios'

export default {
	name: 'CircleContent',

	components: {
		AppContent,
		CircleDetails,
		EmptyContent,
		AccountGroup,
		IconLoading,
	},

	mixins: [isMobile, RouterMixin],

	props: {
		loading: {
			type: Boolean,
			default: true,
		},
	},

	data() {
		return {
			loadingList: false,
			fileTree: null,
			project: null,
		}
	},

	computed: {
		// store variables
		circles() {
			return this.$store.getters.getCircles
		},
		circle() {
			return this.$store.getters.getCircle(this.selectedCircle)
		},
		members() {
			return Object.values(this.circle?.members || [])
		},

		/**
		 * Is the current circle empty
		 *
		 * @return {boolean}
		 */
		isEmptyCircle() {
			return this.members.length === 0
		},
	},

	watch: {
		circle(newCircle) {
			if (newCircle?.id) {
				this.fetchCircleMembers(newCircle.id);
				this.fetchProjectOfCircle(this.circle.id);
			}
		},
	},

	beforeMount() {
		if (this.circle?.id) {
			this.fetchCircleMembers(this.circle.id);
			this.fetchProjectOfCircle(this.circle.id);
		}
	},

	methods: {
		async fetchCircleMembers(circleId) {
			this.loadingList = true
			this.logger.debug('Fetching members for', { circleId })

			try {
				await this.$store.dispatch('getCircleMembers', circleId)
			} catch (error) {
				console.error(error)
				showError(t('contacts', 'There was an error fetching the member list'))
			} finally {
				this.loadingList = false
			}
		},
		async fetchProjectOfCircle(circleId) {
			this.project = null;
			this.fileTree = null;
			
			const url = generateUrl(`/apps/projectcreatoraio/api/v1/projects/circle/${circleId}/files`);
			const response = await axios.get(url, {
				headers: {
					'OCS-APIRequest': 'true',
					'Content-Type': 'application/json'
				}
			});

			this.fileTree = response.data.tree;
			this.project = response.data.project;
		},
	},
}
</script>

<style lang="scss" scoped>
// TODO: replace my button component when available
button {
	height: 44px;
	display: flex;
	justify-content: center;
	align-items: center;
	span {
		margin-right: 10px;
	}
}

.empty-content {
	height: 100%;
}
</style>
