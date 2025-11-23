<!--
  - SPDX-FileCopyrightText: 2021 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
    <div class="circle-details">
        <DetailsHeader>
            <template #avatar="{avatarSize}">
                <Avatar :disable-tooltip="true" :display-name="circle.displayName" :is-no-user="true" :size="avatarSize" />
            </template>
            
            <template #title>
                <div v-if="loadingName" class="circle-name__loader icon-loading-small" />
                <h2>{{ circle.displayName }}</h2>
            </template>
            
            <template v-if="!circle.isOwner" #subtitle>
                {{ t('contacts', 'Team owned by {owner}', { owner: circle.owner.displayName}) }}
            </template>
            
            <template #actions>
                <Button type="tertiary" :href="circleUrl" :class="copyLinkIcon" @click.stop.prevent="copyToClipboard(circleUrl)" />
                
                <Button v-if="(circle.isOwner || circle.isAdmin) && !circle.isPersonal" @click="showSettingsModal = true">
                    <template #icon><Cog :size="20" /></template>
                    {{ t('contacts', 'Team settings') }}
                </Button>

                <Button v-if="!circle.isPendingMember && !circle.isMember && circle.canJoin" :disabled="loadingJoin" class="primary" @click="joinCircle">
                    <template #icon><Login :size="16" /></template>
                    {{ t('contacts', 'Request to join') }}
                </Button>
            </template>
        </DetailsHeader>

        <section v-if="project" class="project-details-container">
            
            <div class="section-wrapper">
                <div class="section-header">
                    <h3 class="modern-header">{{ t('projectcreatoraio', 'Project Details') }}</h3>
                    <div class="header-actions" v-if="isAdmin">
                        <template v-if="editingSection === 'project'">
                            <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                            <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">
                                {{ t('projectcreatoraio', 'Save') }}
                            </Button>
                        </template>
                        <Button v-else-if="!editingSection" type="tertiary" @click="startEditing('project')">
                            <template #icon><span class="icon-rename" /></template>
                            {{ t('projectcreatoraio', 'Edit') }}
                        </Button>
                    </div>
                </div>

                <div class="detail-card">
                    <transition name="fade" mode="out-in">
                        <div v-if="editingSection === 'project'" key="edit" class="grid-layout">
                            <div class="field-group">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Project Name') }}</label>
                                <NcTextField :value.sync="editForm.name" />
                            </div>
                            <div class="field-group">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Number') }}</label>
                                <NcTextField :value.sync="editForm.number" />
                            </div>
                            <div class="field-group">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Type') }}</label>
                                <NcSelect v-model="editFormTypeOption" :options="PROJECT_TYPES" />
                            </div>
                            <div class="field-group full-width">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Description') }}</label>
                                <NcRichContenteditable :value.sync="editForm.description" :multiline="true" class="description-box input-mode" />
                            </div>
                        </div>

                        <div v-else key="view" class="grid-layout">
                            <div class="field-group">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Project Name') }}</label>
                                <div class="value-text">{{ project.name }}</div>
                            </div>
                            <div class="field-group">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Number') }}</label>
                                <div class="value-text">{{ project.number }}</div>
                            </div>
                            <div class="field-group">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Type') }}</label>
                                <div class="value-text">{{ projectTypeLabel }}</div>
                            </div>
                            <div class="field-group full-width">
                                <label class="modern-label">{{ t('projectcreatoraio', 'Description') }}</label>
                                <NcRichContenteditable :value="project.description" :contenteditable="false" :multiline="true" class="description-box" />
                            </div>
                        </div>
                    </transition>
                </div>
            </div>

            <div class="section-wrapper mt-4">
                <div class="section-header">
                    <h3 class="modern-header">{{ t('projectcreatoraio', 'Client Information') }}</h3>
                    <div class="header-actions" v-if="isAdmin">
                        <template v-if="editingSection === 'client'">
                            <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                            <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">
                                {{ t('projectcreatoraio', 'Save') }}
                            </Button>
                        </template>
                        <Button v-else-if="!editingSection" type="tertiary" @click="startEditing('client')">
                            <template #icon><span class="icon-rename" /></template>
                            {{ t('projectcreatoraio', 'Edit') }}
                        </Button>
                    </div>
                </div>

                <div class="detail-card">
                    <transition name="fade" mode="out-in">
                        <div v-if="editingSection === 'client'" key="edit" class="grid-layout">
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Name') }}</label><NcTextField :value.sync="editForm.client_name" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Role') }}</label><NcTextField :value.sync="editForm.client_role" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Phone') }}</label><NcTextField :value.sync="editForm.client_phone" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Email') }}</label><NcTextField :value.sync="editForm.client_email" /></div>
                            <div class="field-group full-width"><label class="modern-label">{{ t('projectcreatoraio', 'Address') }}</label><NcTextField :value.sync="editForm.client_address" /></div>
                        </div>
                        <div v-else key="view" class="grid-layout">
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Name') }}</label><div class="value-text">{{ project.client_name || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Role') }}</label><div class="value-text">{{ project.client_role || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Phone') }}</label><div class="value-text">{{ project.client_phone || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Email') }}</label><div class="value-text">{{ project.client_email || '-' }}</div></div>
                            <div class="field-group full-width"><label class="modern-label">{{ t('projectcreatoraio', 'Address') }}</label><div class="value-text">{{ project.client_address || '-' }}</div></div>
                        </div>
                    </transition>
                </div>
            </div>

            <div class="section-wrapper mt-4">
                <div class="section-header">
                    <h3 class="modern-header">{{ t('projectcreatoraio', 'Location') }}</h3>
                    <div class="header-actions" v-if="isAdmin">
                        <template v-if="editingSection === 'location'">
                            <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                            <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">
                                {{ t('projectcreatoraio', 'Save') }}
                            </Button>
                        </template>
                        <Button v-else-if="!editingSection" type="tertiary" @click="startEditing('location')">
                            <template #icon><span class="icon-rename" /></template>
                            {{ t('projectcreatoraio', 'Edit') }}
                        </Button>
                    </div>
                </div>

                <div class="detail-card">
                    <transition name="fade" mode="out-in">
                        <div v-if="editingSection === 'location'" key="edit" class="grid-layout">
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Street') }}</label><NcTextField :value.sync="editForm.loc_street" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'City') }}</label><NcTextField :value.sync="editForm.loc_city" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Zip') }}</label><NcTextField :value.sync="editForm.loc_zip" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Ref') }}</label><NcTextField :value.sync="editForm.external_ref" /></div>
                        </div>
                        <div v-else key="view" class="grid-layout">
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Street') }}</label><div class="value-text">{{ project.loc_street || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'City') }}</label><div class="value-text">{{ project.loc_city || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Zip') }}</label><div class="value-text">{{ project.loc_zip || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Ref') }}</label><div class="value-text">{{ project.external_ref || '-' }}</div></div>
                        </div>
                    </transition>
                </div>
            </div>

            <div class="section-wrapper mt-4">
                <div class="section-header">
                    <h3 class="modern-header">{{ t('projectcreatoraio', 'Timeline') }}</h3>
                    <div class="header-actions" v-if="isAdmin">
                        <template v-if="editingSection === 'timeline'">
                            <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                            <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">
                                {{ t('projectcreatoraio', 'Save') }}
                            </Button>
                        </template>
                        <Button v-else-if="!editingSection" type="tertiary" @click="startEditing('timeline')">
                            <template #icon><span class="icon-rename" /></template>
                            {{ t('projectcreatoraio', 'Edit') }}
                        </Button>
                    </div>
                </div>

                <div class="detail-card">
                    <transition name="fade" mode="out-in">
                        <div v-if="editingSection === 'timeline'" key="edit" class="grid-layout">
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Start') }}</label><NcTextField type="date" :value.sync="editForm.date_start" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'End') }}</label><NcTextField type="date" :value.sync="editForm.date_end" /></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Status') }}</label><NcSelect v-model="currentStatusLabel" :options="statusOptions" :disabled="!isAdmin"/></div>
                        </div>
                        <div v-else key="view" class="grid-layout">
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Start') }}</label><div class="value-text">{{ project.date_start || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'End') }}</label><div class="value-text">{{ project.date_end || '-' }}</div></div>
                            <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Status') }}</label><div class="value-text">{{ currentStatusLabel ? currentStatusLabel.label : '-' }}</div></div>
                        </div>
                    </transition>
                </div>
            </div>
        </section>

        <div v-else class="loading-placeholder">
            <NcLoadingIcon />
        </div>

        <section v-if="circle.isMember">
            <ContentHeading>{{ t('contacts', 'Team resources') }}</ContentHeading>
             <p>{{ t('contacts', 'Anything shared with this team will show up here') }}</p>
             <div v-for="provider in resourceProviders" :key="provider.id">
                <ContentHeading>
                    <span v-show="false" class="provider__icon" v-html="provider.icon" /> {{ provider.name }}
                </ContentHeading>
                <ul>
                    <ListItem v-for="resource in resourcesForProvider(provider.id)" :key="resource.url" class="resource" :name="resource.label" @click="redirect(resource.url)">
                        <template #icon>
                            <span v-if="resource.iconEmoji" class="resource__icon">{{ resource.iconEmoji }}</span>
                            <span v-else-if="resource.iconSvg" class="resource__icon" v-html="resource.iconSvg" />
                            <span v-else-if="resource.iconURL" class="resource__icon"><img :src="resource.iconURL" alt=""></span>
                        </template>
                    </ListItem>
                </ul>
            </div>

            <div class="files-container">
                <div v-if="files" class="project-files-section">
                    <h2>{{ t('contacts', 'Root Tree') }}</h2>
                    <FileTreeNode v-for="file in files.shared" :key="file.id" :node="file"/>
                    <FileTreeNode v-for="file in files.private" :key="file.id" :node="file"/>
                </div>
                <div v-else class="empty-content">
                    {{ t('contacts', 'No project files found for this team.') }}
                </div>
            </div>
        </section>

        <template v-if="!circle.isMember">
            <NcEmptyContent v-if="circle.isPendingMember" :name="t('contacts', 'Your request to join this team is pending approval')">
                <template #icon><NcLoadingIcon :size="20" /></template>
            </NcEmptyContent>
            <NcEmptyContent v-else :name="t('contacts', 'You are not a member of {circle}', { circle: circle.displayName})">
                <template #icon><IconAccountGroup :size="20" /></template>
            </NcEmptyContent>
        </template>

        <MemberList v-if="members.length" :list="members" />

        <Modal v-if="(circle.isOwner || circle.isAdmin) && !circle.isPersonal && showSettingsModal" @close="showSettingsModal=false">
            <div class="circle-settings">
                <h2>{{ t('contacts', 'Team settings') }}</h2>
                <h3>{{ t('contacts', 'Team name') }}</h3>
                <input v-model="circle.displayName" :readonly="!circle.isOwner" type="text" @input="onNameChangeDebounce">
                <h3>{{ t('contacts', 'Settings') }}</h3>
                <CircleConfigs :circle="circle" />
                <CirclePasswordSettings :circle="circle" />
                <h3>{{ t('contacts', 'Actions') }}</h3>
                <Button v-if="circle.canLeave" type="warning" @click="confirmLeaveCircle">
                    <template #icon><Logout :size="16" /></template>{{ t('contacts', 'Leave team') }}
                </Button>
                <Button v-if="circle.canDelete" type="error" @click.prevent.stop="confirmDeleteCircle">
                    <template #icon><IconDelete :size="20" /></template>{{ t('contacts', 'Delete team') }}
                </Button>
            </div>
        </Modal>
    </div>
</template>

<script>
import { ref } from 'vue'
import { useElementSize } from '@vueuse/core'
import debounce from 'debounce'
import { generateOcsUrl, generateUrl } from '@nextcloud/router'
import { showError } from '@nextcloud/dialogs'
import axios from '@nextcloud/axios'
import { t } from '@nextcloud/l10n';

import {
	NcAvatar as Avatar,
	NcButton as Button,
	NcEmptyContent,
	NcListItem as ListItem,
	NcLoadingIcon,
	NcModal as Modal,
	NcSelect,
	NcTextField
} from '@nextcloud/vue'

import NcRichContenteditable from '@nextcloud/vue/components/NcRichContenteditable'
import NcChip from '@nextcloud/vue/components/NcChip'
import Cog from 'vue-material-design-icons/Cog.vue'
import Login from 'vue-material-design-icons/Login.vue'
import Logout from 'vue-material-design-icons/Logout.vue'
import IconDelete from 'vue-material-design-icons/Delete.vue'
import IconAccountGroup from 'vue-material-design-icons/AccountGroup.vue'

import { CircleEdit, editCircle } from '../services/circles.ts'
import CircleActionsMixin from '../mixins/CircleActionsMixin.js'
import DetailsHeader from './DetailsHeader.vue'
import CircleConfigs from './CircleDetails/CircleConfigs.vue'
import MemberList from './MemberList/MemberList.vue'
import ContentHeading from './CircleDetails/ContentHeading.vue'
import CirclePasswordSettings from './CircleDetails/CirclePasswordSettings.vue'
import FileTreeNode from './FileTreeNode.vue'
import { getCurrentUser } from '@nextcloud/auth'

export const PROJECT_TYPES = [
	{ id: 0, label: t('projectcreatoraio', 'Combi') },
	{ id: 1, label: t('projectcreatoraio', 'Solo Elektra ') },
	{ id: 2, label: t('projectcreatoraio', 'Solo Water') },
	{ id: 3, label: t('projectcreatoraio', 'Custom ') }
];

export default {
	name: 'CircleDetails',

	components: {
		Avatar,
		Button,
		CircleConfigs,
		CirclePasswordSettings,
		ContentHeading,
		DetailsHeader,
		ListItem,
		Cog,
		IconAccountGroup,
		IconDelete,
		Login,
		Logout,
		MemberList,
		Modal,
		NcEmptyContent,
		NcLoadingIcon,
		NcRichContenteditable,
		FileTreeNode,
		NcSelect,
		NcChip,
		NcTextField
	},

	mixins: [CircleActionsMixin],

	setup() {
		const avatarList = ref()
		const { width } = useElementSize(avatarList)
		return { avatarList, width }
	},

	data() {
		return {
			PROJECT_TYPES, // exposing types 

			loadingDescription: false,
			loadingName: false,
			showSettingsModal: false,
			showMembersModal: false,
			resources: null,

			// NEW DATA FOR EDIT MODE
			editingSection: null, // 'project', 'client', 'location', 'timeline'            
			savingProject: false,
            editForm: {
                name: '',
                number: '',
                type: 0,
                description: '',
                // Client
                client_name: '',
                client_role: '',
                client_phone: '',
                client_email: '',
                client_address: '',
                // Location
                loc_street: '',
                loc_city: '',
                loc_zip: '',
                external_ref: '',
                // Dates
                date_start: '',
                date_end: ''
            }
		}
	},
	props: {
		project: {
			type: Object,
			required: false
		},
		files: {
			type: Object,
			required: false
		}
	},
	computed: {
		isAdmin() {
			return !!getCurrentUser()?.isAdmin;
		},

		projectTypeLabel() {
			if (!this.project || !PROJECT_TYPES) {
				return '';
			}
			const typeInfo = PROJECT_TYPES[this.project.type];
			return typeInfo ? typeInfo.label : 'Unknown';
		},

		statusOptions() {
			return [
				{ id: 0, label: this.t('projectcreatoraio', 'Archived') },
				{ id: 1, label: this.t('projectcreatoraio', 'Active') },
			];
		},

		// Helper for Select component in Edit Mode
        editFormTypeOption: {
            get() {
                return PROJECT_TYPES.find(opt => opt.id === this.editForm.type) || PROJECT_TYPES[0];
            },
            set(option) {
                this.editForm.type = option ? option.id : 0;
            }
		},

		currentStatusLabel: {
			get() {
				return this.statusOptions.find(opt => opt.id === this.project.status);
			},
			set(option) {
				this.project.status = option ? option.id : null;
			}
		},

		statusClass() {
			return {
				'status--active': this.project.status === 1,
				'status--archived': this.project.status === 0,
			};
		},
		
		descriptionPlaceholder() {
			if (this.circle.description.trim() === '') {
				return t('contacts', 'There is no description for this team')
			}
			return t('contacts', 'Enter a description for the team')
		},

		isEmptyDescription() {
			return this.circle.description.trim() === ''
		},

		showDescription() {
			if (this.circle.isOwner) {
				return true
			}
			return !this.isEmptyDescription
		},

		members() {
			return Object.values(this.$store.getters.getCircle(this.circle.id)?.members || [])
		},

		maxMembers() {
			// How many avatars (default-clickable-area + 12px gap) fit?
			const avatarWidth = parseInt(window.getComputedStyle(document.body).getPropertyValue('--default-clickable-area')) + 12
			const maxMembers = Math.floor(this.width / avatarWidth)
			return (this.members.length > maxMembers)
				? maxMembers - 1
				: maxMembers
		},

		memberLimit() {
			return Math.min(this.members.length, this.maxMembers)
		},

		membersLimited() {
			return this.members.slice(0, this.memberLimit)
		},

		hasExtraMembers() {
			return this.members.length > this.maxMembers
		},

		resourceProviders() {
			return this.resources?.reduce((acc, res) => {
				if (!acc.find(p => p.id === res.provider.id)) {
					acc.push(res.provider)
				}
				return acc
			}, []) ?? []
		},

		resourcesForProvider() {
			return (providerId) => {
				return this.resources?.filter(res => res.provider.id === providerId) ?? []
			}
		},
	},

	watch: {
		'circle.id': {
			handler() {
				this.fetchTeamResources()
			},
			immediate: true,
		},
	},

	methods: {
		async fetchTeamResources() {
			const response = await axios.get(generateOcsUrl(`/teams/${this.circle.id}/resources`))
			this.resources = response.data.ocs.data.resources
		},
		/**
		 * Autocomplete @mentions on the description
		 *
		 * @param {string} search the search term
		 * @param {Function} callback callback to be called with results array
		 */
		onAutocomplete(search, callback) {
			// TODO: implement autocompletion. Disabled for now
			// eslint-disable-next-line n/no-callback-literal
			callback([])
		},

		onDescriptionChangeDebounce: debounce(function(...args) {
			this.onDescriptionChange(...args)
		}, 500),
		async onDescriptionChange(description) {
			this.loadingDescription = true
			try {
				await editCircle(this.circle.id, CircleEdit.Description, description)
			} catch (error) {
				console.error('Unable to edit team description', description, error)
				showError(t('contacts', 'An error happened during description sync'))
			} finally {
				this.loadingDescription = false
			}
		},

		onNameChangeDebounce: debounce(function(event) {
			this.onNameChange(event.target.value)
		}, 500),
		async onNameChange(name) {
			this.loadingName = true
			try {
				await editCircle(this.circle.id, CircleEdit.Name, name)
			} catch (error) {
				console.error('Unable to edit name', name, error)
				showError(t('contacts', 'An error happened during name sync'))
			} finally {
				this.loadingName = false
			}
		},

		redirect(url) {
			window.open(url, '_self');
		},

		/**
         * START EDITING A SPECIFIC SECTION
         * @param {string} section - 'project', 'client', 'location', or 'timeline'
         */
        startEditing(section) {
            // 1. Copy current data to editForm
            this.editForm = JSON.parse(JSON.stringify(this.project));
            
            // 2. Ensure nulls become strings
            Object.keys(this.editForm).forEach(k => {
                if (this.editForm[k] === null) this.editForm[k] = '';
            });

            // 3. Set active section
            this.editingSection = section;
        },

		/**
         * CANCEL EDIT
         * Discards changes and closes inputs
         */
        cancelEditing() {
            this.editingSection = null;
            this.editForm = {};
        },

        /**
         * SAVE CHANGES
         * Sends one single request to the server
         */
        async saveProjectChanges() {
            this.savingProject = true;
            try {
                const url = generateUrl(`/apps/projectcreatoraio/api/v1/projects/${this.project.id}`);
                
                // Send the PUT request
                const response = await axios.put(url, this.editForm);
                
                // Update the local 'project' prop with the new values so the View updates
                Object.assign(this.project, this.editForm);
                
                showSuccess(t('projectcreatoraio', 'Project details saved'));
                this.isEditing = false;
            } catch (error) {
                console.error(error);
                showError(t('projectcreatoraio', 'Could not save project details'));
            } finally {
                this.savingProject = false;
            }
        }
 	},
}
</script>

<style lang="scss" scoped>
/* =========================================
   1. MODERN DASHBOARD STYLES (Project Details)
   ========================================= */

.project-details-container {
    max-width: 900px;
    margin: 20px auto 60px auto;
    font-family: var(--font-family, -apple-system, BlinkMacSystemFont, sans-serif);
}

.section-wrapper {
    margin-bottom: 24px;
}

/* HEADERS */
.section-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
    padding: 0 4px;
}

.modern-header {
    font-size: 16px;
    font-weight: 600;
    color: #444444;
    margin: 0;
}

.header-actions {
    display: flex;
    gap: 8px;
}

/* CARDS */
.detail-card {
    background-color: var(--color-main-background, #fff);
    border: 1px solid var(--color-border, #ededed);
    border-radius: 12px;
    padding: 24px;
    transition: box-shadow 0.2s ease, transform 0.2s ease;
}

.detail-card:hover {
    box-shadow: 0 4px 12px rgba(0,0,0,0.05);
    border-color: #dcdcdc;
}

/* GRID */
.grid-layout {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
    gap: 20px 32px;
}

.full-width {
    grid-column: 1 / -1;
}

/* LABELS & VALUES */
.field-group {
    display: flex;
    flex-direction: column;
}

.modern-label {
    display: block;
    font-size: 11px;
    text-transform: uppercase;
    font-weight: 700;
    color: #888888;
    letter-spacing: 0.05em;
    margin-bottom: 8px;
}

.value-text {
    font-size: 15px;
    color: var(--color-main-text, #222);
    line-height: 1.5;
    min-height: 20px;
}

/* DESCRIPTION */
.description-box {
    display: block;
    width: 100%;
    min-height: 60px;
    white-space: pre-wrap;
    line-height: 1.6;
    color: var(--color-main-text, #222);
    border: none !important;
    background: transparent !important;
    padding: 0 !important;
}

.fade-enter-active, .fade-leave-active {
    transition: opacity 0.2s ease;
}
.fade-enter-from, .fade-leave-to {
    opacity: 0;
}

.field-group :deep(.nc-textfield-wrapper),
.field-group :deep(.nc-select-wrapper) {
    width: 100%;
}


/* =========================================
   2. RESTORED LEGACY STYLES (Resources, Files, Members)
   ========================================= */

/* Main Layout Wrappers for non-project sections */
.app-content-details header,
.app-content-details section:not(.project-details-container) {
    max-width: 800px;
    margin: auto;
    margin-bottom: 36px;
    @media screen and (max-width: 1024px) {
        padding: 0 20px;
    }
}

/* Restore Avatar Header sizing */
.app-content-details header :deep(.contact-header__avatar) {
    width: 75px !important;
}

.app-content-details header :deep(.contact-header__no-wrap) {
    flex-grow: 1;
}

.app-content-details header :deep(.contact-header__actions) {
    flex-grow: 0;
}

.circle-name__loader {
    margin-left: 8px;
}

.circle-details {
    padding-inline: 20px;
}

/* Restore Resource List Styles & Icons */
.provider__icon {
    display: inline-block;
    width: 24px;
    height: 24px;
}

.resource {
    &__icon {
        width: 44px;
        height: 44px;
        display: flex;
        align-items: center;
        justify-content: center;
        text-align: center;
        svg {
            width: 20px;
            height: 20px;
        }
        img {
            border-radius: var(--border-radius-pill);
            overflow: hidden;
            width: 32px;
            height: 32px;
        }
    }

    &:deep(.line-one__name) {
        font-weight: normal;
    }
}

/* Restore File Tree & Lists */
:deep(.app-content-list) {
    max-width: 100%;
    border: 0;
}

.files-container {
    margin-top: 24px;
}

/* Restore Member List Layout */
.avatar-box {
    display: flex;
    justify-content: space-between;
    align-items: center;
}

.avatar-list {
    display: flex;
    flex-wrap: wrap;
    flex-grow: 1;
    gap: 12px;
}

/* =========================================
   3. SETTINGS MODAL STYLES
   ========================================= */
.circle-settings {
    margin: 20px;
    padding-bottom: 20px;
}

.circle-settings h2 {
    font-size: 20px;
    font-weight: bold;
    margin-bottom: 20px;
    color: var(--color-main-text);
}

.circle-settings h3 {
    font-size: 16px;
    font-weight: 600;
    margin-top: 20px;
    margin-bottom: 10px;
    color: var(--color-text-maxcontrast);
}

.circle-settings input[type="text"] {
    width: 100%;
    padding: 10px;
    border: 1px solid var(--color-border);
    border-radius: var(--border-radius);
}

.circle-settings button,
.circle-settings .button-vue {
    margin-right: 10px;
    margin-top: 10px;
}

.circle-details-section__configs {
    margin-bottom: 15px;
}
</style>