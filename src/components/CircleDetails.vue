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
            
            <div class="top-grid">
                
                <div class="section-wrapper">
                    <div class="section-header">
                        <h3 class="modern-header">{{ t('projectcreatoraio', 'Project Details') }}</h3>
                        <div class="header-actions" v-if="isAdmin">
                            <template v-if="editingSection === 'project'">
                                <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                                <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">{{ t('projectcreatoraio', 'Save') }}</Button>
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
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Project Name') }}</label><NcTextField :value.sync="editForm.name" /></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Reference') }}</label><NcTextField :value.sync="editForm.number" /></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Type') }}</label><NcSelect v-model="editFormTypeOption" :options="PROJECT_TYPES" /></div>
                                <div class="field-group full-width"><label class="modern-label">{{ t('projectcreatoraio', 'Description') }}</label><NcRichContenteditable :value.sync="editForm.description" :multiline="true" class="description-box input-mode" /></div>
                            </div>
                            <div v-else key="view" class="grid-layout">
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Project Name') }}</label><div class="value-text">{{ project.name }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Reference') }}</label><div class="value-text">{{ project.number }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Type') }}</label><div class="value-text">{{ projectTypeLabel }}</div></div>
                                <div class="field-group full-width"><label class="modern-label">{{ t('projectcreatoraio', 'Description') }}</label><div class="value-text">{{ project.description }}</div></div>
                            </div>
                        </transition>
                    </div>
                </div>

                <div class="section-wrapper">
                    <div class="section-header">
                        <h3 class="modern-header">{{ t('projectcreatoraio', 'Client Information') }}</h3>
                        <div class="header-actions" v-if="isAdmin">
                            <template v-if="editingSection === 'client'">
                                <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                                <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">{{ t('projectcreatoraio', 'Save') }}</Button>
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
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Address') }}</label><NcTextField :value.sync="editForm.client_address" /></div>
                            </div>
                            <div v-else key="view" class="grid-layout">
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Name') }}</label><div class="value-text">{{ project.client_name || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Role') }}</label><div class="value-text">{{ project.client_role || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Phone') }}</label><div class="value-text">{{ project.client_phone || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Email') }}</label><div class="value-text">{{ project.client_email || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Address') }}</label><div class="value-text">{{ project.client_address || '-' }}</div></div>
                            </div>
                        </transition>
                    </div>
                </div>

                <div class="section-wrapper">
                    <div class="section-header">
                        <h3 class="modern-header">{{ t('projectcreatoraio', 'Timeline') }}</h3>
                        <div class="header-actions" v-if="isAdmin">
                            <template v-if="editingSection === 'timeline'">
                                <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                                <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">{{ t('projectcreatoraio', 'Save') }}</Button>
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
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Registeration Date') }}</label><NcTextField type="date" :value.sync="editForm.date_start" /></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'End Date') }}</label><NcTextField type="date" :value.sync="editForm.date_end" /></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Status') }}</label><NcSelect v-model="currentStatusLabel" :options="STATUS_OPTIONS"/></div>
                            </div>
                            <div v-else key="view" class="grid-layout">
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Registeration Date') }}</label><div class="value-text">{{ project.date_start || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'End Date') }}</label><div class="value-text">{{ project.date_end || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Status') }}</label><div class="value-text">{{ currentStatusLabel.label || '-' }}</div></div>
                            </div>
                        </transition>
                        
                        <div class="mt-4">
                             <Timeline></Timeline>
                        </div>
                    </div>
                </div>

                <div class="section-wrapper stats-card">
                    <div class="section-header">
                        <h3 class="modern-header">{{ t('projectcreatoraio', 'Progress') }}</h3>
                    </div>
                    <div class="detail-card wheels-row">
                        <ProgressWheel
                            :percentage="95"
                            label="Approved"
                            color="#3b82f6" 
                            :size="130"
                        />
                        <ProgressWheel
                            :percentage="65"
                            label="Steps"
                            color="#10b981"
                            :size="130"
                            class="segmented-style"
                        />
                    </div>
                </div>

            </div>
            <div class="bottom-row">
                <div class="section-wrapper">
                    <div class="section-header">
                        <h3 class="modern-header">{{ t('projectcreatoraio', 'Location') }}</h3>
                        <div class="header-actions" v-if="isAdmin">
                            <template v-if="editingSection === 'location'">
                                <Button @click="cancelEditing" :disabled="savingProject">{{ t('projectcreatoraio', 'Cancel') }}</Button>
                                <Button type="primary" @click="saveProjectChanges" :disabled="savingProject">{{ t('projectcreatoraio', 'Save') }}</Button>
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
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Municipality') }}</label><NcTextField :value.sync="editForm.external_ref" /></div>
                            </div>
                            <div v-else key="view" class="grid-layout">
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Street') }}</label><div class="value-text">{{ project.loc_street || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'City') }}</label><div class="value-text">{{ project.loc_city || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Zip') }}</label><div class="value-text">{{ project.loc_zip || '-' }}</div></div>
                                <div class="field-group"><label class="modern-label">{{ t('projectcreatoraio', 'Municipality') }}</label><div class="value-text">{{ project.external_ref || '-' }}</div></div>
                            </div>
                        </transition>
                    </div>
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
        
        <section>
            <div class="section-wrapper mt-4">
                <div class="section-header">
                    <h2>{{ t('contacts', 'Workspace') }}</h2>
                </div>

                <div v-if="deckUrl" class="workspace-card mb-4">
                    <div class="workspace-header">
                        <span class="workspace-title">
                            <IconDeck :size="20" />
                            {{ t('projectcreatoraio', 'Project Deck') }}
                        </span>
                        <a :href="deckUrl" target="_blank" class="workspace-link">
                            {{ t('projectcreatoraio', 'Open in App') }} <span class="icon-external"></span>
                        </a>
                    </div>
                    <div class="iframe-container">
                        <div v-if="loadingDeck" class="iframe-loader">
                            <NcLoadingIcon :size="40" />
                            <p>{{ t('projectcreatoraio', 'Loading Board...') }}</p>
                        </div>
                        <iframe :src="deckUrl" class="embedded-frame" @load="onDeckIframeLoad"></iframe>
                    </div>
                </div>
                <div v-else class="empty-workspace">
                    <NcEmptyContent :name="t('projectcreatoraio', 'No Deck Board linked')" />
                </div>

                <div v-if="whiteboardUrl" class="workspace-card">
                    <div class="workspace-header">
                        <span class="workspace-title">
                            {{ t('projectcreatoraio', 'Whiteboard') }}
                        </span>
                        <a :href="whiteboardUrl" target="_blank" class="workspace-link">
                            {{ t('projectcreatoraio', 'Open Fullscreen') }} <span class="icon-external"></span>
                        </a>
                    </div>
                    <div class="iframe-container">
                        <iframe :src="whiteboardUrl" class="embedded-frame"></iframe>
                    </div>
                </div>
            </div>
        </section>

        <section>
            <div class="section-wrapper mt-4">
                <div class="section-header">
                    <h3 class="modern-header">{{ t('projectcreatoraio', 'Latest Updates') }}</h3>
                    
                    <div class="tab-pills">
                        <button 
                            class="tab-item" 
                            :class="{ 'active': activeTab === 'activity' }"
                            @click="selectTab('activity')"
                        >
                            {{ t('projectcreatoraio', 'Activity') }}
                        </button>
                        <button 
                            class="tab-item" 
                            :class="{ 'active': activeTab === 'notes' }"
                            @click="selectTab('notes')"
                        >
                            {{ t('projectcreatoraio', 'Private Notes') }}
                        </button>
                    </div>
                </div>

                <div class="detail-card communication-card">
                    
                    <div v-if="activeTab === 'activity'" class="feed-container">
                        <div v-if="loadingActivity" class="feed-loader">
                            <NcLoadingIcon :size="32" />
                        </div>
                        
                        <div v-else-if="activityList && activityList.length > 0" class="activity-list">
                            <div v-for="item in activityList" :key="item.id" class="feed-row">
                                <Avatar :user="item.actor_id" :disable-tooltip="false" :size="32" class="feed-avatar" />
                                
                                <div class="feed-bubble">
                                    <div class="feed-header">
                                        <div class="feed-meta-left">
                                            <span class="feed-user">{{ item.actor_id }}</span>
                                            <span class="feed-context">
                                                <a href="#" class="card-link" @click.prevent="openCard(item.card_id)" :name="t('projectcreatoraio', 'Go to card')">
                                                    {{ t('projectcreatoraio', 'Go to card') }}
                                                </a>
                                            </span>
                                        </div>
                                        <span class="feed-date">{{ formatDate(item.creation_timestamp) }}</span>
                                    </div>
                                    <div class="feed-message">{{ item.message }}</div>
                                </div>
                            </div>
                        </div>
                        
                        <NcEmptyContent v-else :name="t('projectcreatoraio', 'No recent activity')">
                            <template #icon>
                                <IconMessage :size="44" />
                            </template>
                            <template #desc>
                                {{ t('projectcreatoraio', 'Comments made on the Deck board will appear here.') }}
                            </template>
                        </NcEmptyContent>
                    </div>

                    <div v-else class="feed-container">
                        <div v-if="loadingNotes" class="feed-loader">
                            <NcLoadingIcon :size="32" />
                        </div>

                        <div v-else-if="notesList && notesList.length > 0" class="notes-list">
                            <div v-for="note in notesList" :key="note.id" class="note-card">
                                <div class="note-header">
                                    <div class="note-meta-left">
                                        <span class="icon-note">📝</span>
                                        <a v-if="note.cardId" href="#" class="card-link" @click.prevent="openCard(note.cardId)">
                                            {{ t('projectcreatoraio', 'Go to card') }}
                                        </a>
                                    </div>
                                    <span class="note-date">{{ formatDate(note.createdAt) }}</span>
                                </div>
                                <div class="note-body">{{ note.content }}</div>
                            </div>
                        </div>

                        <NcEmptyContent v-else :name="t('projectcreatoraio', 'No private notes found')">
                            <template #icon>
                                <IconNote :size="44" />
                            </template>
                            <template #desc>
                                {{ t('projectcreatoraio', 'Private notes you add to cards will be listed here.') }}
                            </template>
                        </NcEmptyContent>
                    </div>

                </div>
            </div>
        </section>

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
import { showError, showSuccess } from '@nextcloud/dialogs'
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
import IconMessage from 'vue-material-design-icons/Message.vue'
import IconNote from 'vue-material-design-icons/Note.vue'

import { CircleEdit, editCircle } from '../services/circles.ts'
import CircleActionsMixin from '../mixins/CircleActionsMixin.js'
import DetailsHeader from './DetailsHeader.vue'
import CircleConfigs from './CircleDetails/CircleConfigs.vue'
import MemberList from './MemberList/MemberList.vue'
import ContentHeading from './CircleDetails/ContentHeading.vue'
import CirclePasswordSettings from './CircleDetails/CirclePasswordSettings.vue'
import FileTreeNode from './FileTreeNode.vue'
import { getCurrentUser } from '@nextcloud/auth'
import IconDeck from 'vue-material-design-icons/ViewColumn.vue'
import IconCard from 'vue-material-design-icons/CardTextOutline.vue'
import Timeline from './Timeline.vue';
import ProgressWheel from './ProgressWheel.vue';

export const PROJECT_TYPES = [
    { id: 0, label: t('projectcreatoraio', 'Combi') },
    { id: 1, label: t('projectcreatoraio', 'Solo Elektra ') },
    { id: 2, label: t('projectcreatoraio', 'Solo Water') },
    { id: 3, label: t('projectcreatoraio', 'Custom ') }
];

export const STATUS_OPTIONS = [
    { id: 0, label: t('projectcreatoraio', 'Archived') },
    { id: 1, label: t('projectcreatoraio', 'Active') },
];

export const DEFAULT_BASE_URL = 'https://excalidraw.loket.site';

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
        NcTextField,
        IconDeck,
        IconCard,
        IconMessage,
        IconNote,
        Timeline,
        ProgressWheel,
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
            STATUS_OPTIONS, // exposing options

            loadingDescription: false,
            loadingName: false,
            showSettingsModal: false,
            showMembersModal: false,
            resources: null,

            // NEW DATA FOR EDIT MODE
            loadingDeck: true,
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
            },

            // COMMUNICATION HUB DATA
            activeTab: 'activity', // 'activity' or 'notes'
            
            activityList: [],
            loadingActivity: false,
            
            notesList: [],
            loadingNotes: false,

            commentsPollingObj: null,
            notesPollingObj: null,
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
            const typeInfo = PROJECT_TYPES.find(type => type.id === this.project.type);
            return typeInfo ? typeInfo.label : 'Unknown';
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
                return STATUS_OPTIONS.find(opt => opt.id === this.editForm.status) || STATUS_OPTIONS[0];
            },
            set(option) {
                if(option) {
                    this.editForm.status = option.id;
                }
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

        deckUrl() {
            if (!this.project || !this.project.boardId) return null;
            // Standard Nextcloud Deck URL. Adjust if your setup is different.
            return generateUrl(`/apps/deck/board/${this.project.boardId}`);
        },

        whiteboardUrl() {
            console.log("this.project", this.project);
            if (!this.project || !this.project.white_board_id) return null;

            return `${DEFAULT_BASE_URL}/#room=${this.project.white_board_id}`;
        }
    },

    watch: {
        'circle.id': {
            handler() {
                this.fetchTeamResources()
            },
            immediate: true,
        },
        // When project loads, fetch the data
        'project.id': {
            handler(val) {
                if (val) {
                    this.fetchLatestComments();
                    this.fetchLatestPrivateNotes();

                    this.selectTab('activity');
                }
            },
            immediate: true
        }
    },

    methods: {
        selectTab(tabName) {
            this.activeTab = tabName;

            if (this.activeTab === 'activity') {
                this.stopNotesPolling();
                this.startCommentsPolling();
            } else if (this.activeTab === 'notes') {
                this.stopCommentsPolling();
                this.startNotesPolling();
            }
        },
        stopCommentsPolling() {
            this.commentsPollingObj && clearInterval(this.commentsPollingObj);
            this.commentsPollingObj = null;
        },
        stopNotesPolling() {
            this.notesPollingObj && clearInterval(this.notesPollingObj);
            this.notesPollingObj = null;
        },
        startNotesPolling() {
            if(this.notesPollingObj) return;

            this.notesPollingObj = setInterval(() => this.fetchLatestPrivateNotes(), 5000);
        },
        startCommentsPolling() {
            if(this.commentsPollingObj) return;

            this.commentsPollingObj = setInterval(() => this.fetchLatestComments(), 5000);
        },

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
         * Sends only the allowed editable fields to the server
         */
        async saveProjectChanges() {
            this.savingProject = true;
            try {
                const url = generateUrl(`/apps/projectcreatoraio/api/v1/projects/${this.project.id}`);
                
                // 1. Define the whitelist of allowed fields
                const allowedFields = [
                    // Project Details
                    'name', 'number', 'type', 'description', 
                    // Client Info
                    'client_name', 'client_role', 'client_phone', 'client_email', 'client_address',
                    // Location
                    'loc_street', 'loc_city', 'loc_zip', 'external_ref',
                    // Timeline
                    'date_start', 'date_end', 'status'
                ];

                // 2. Construct the payload dynamically
                const payload = {};
                allowedFields.forEach(field => {
                    if (this.editForm[field] !== undefined) {
                        payload[field] = this.editForm[field];
                    }
                });

                // 3. Send only the clean payload
                const response = await axios.put(url, payload);
                Object.assign(this.project, response.data);
                
                showSuccess(t('projectcreatoraio', 'Section saved successfully'));
                this.editingSection = null;
            } catch (error) {
                console.error(error);
                showError(t('projectcreatoraio', 'Could not save details'));
            } finally {
                this.savingProject = false;
            }
        },
        onDeckIframeLoad(event) {
            const iframe = event.target;
            try {
                // 1. Access the document inside the iframe
                const innerDoc = iframe.contentDocument || iframe.contentWindow.document;
                
                // 2. Create a style element
                const style = innerDoc.createElement('style');
                
                // 3. Write CSS to hide the header and fix spacing
                // #header: The top blue/black bar
                // #content: The main container (usually has padding-top for the header)
                style.textContent = `
                    header, #header, .header-bar { display: none !important; }
                    .app-navigation-toggle-wrapper { display: none !important; }
                    #content-vue { margin: 0 !important; height: 100vh; width: 100%; border-radius: 0; }
                    .board-wrapper { max-height: 100vh !important; }
                    .app-navigation { display: none !important; }
                `;
                
                // 4. Append it to the iframe's head
                innerDoc.head.appendChild(style);
                this.loadingDeck = false;
            } catch (e) {
                console.warn('Could not hide iframe header. Likely a cross-origin restriction.', e);
            }
        },

        /**
         * 1. FETCH LATEST PRIVATE NOTES OF USER in the project
         */
        async fetchLatestPrivateNotes() {
            try {
                // Standard Deck Endpoint for Board Activity
                const url = generateUrl(`/apps/deck/boards/${this.project.boardId}/notes/latest`);
                const response = await axios.get(url);
                this.notesList = response.data || [];
            } catch (e) {
                console.error("Could not fetch activity", e);
                this.notesList = [];
            }
        },

        /**
         * 2. FETCH Latest Comments in the Project (Custom API)
         */
        async fetchLatestComments() {
            try {
                // We assume you will create this endpoint in your controller
                const url = generateUrl(`/apps/deck/boards/${this.project.boardId}/comments/latest`);
                const response = await axios.get(url);
                this.activityList = response.data || [];
            } catch (e) {
                console.error("Could not fetch notes", e);
                this.activityList = [];
            }
        },

        /**
         * Helper: Format Date (e.g. "Nov 24, 10:00 AM")
         */
        formatDate(timestamp) {
            if (!timestamp) return '';
            // Handle both Unix timestamp (Deck) and ISO string (Custom Note)
            const date = typeof timestamp === 'number' 
                ? new Date(timestamp * 1000) 
                : new Date(timestamp);
                
            return date.toLocaleDateString(undefined, { 
                month: 'short', day: 'numeric', hour: '2-digit', minute: '2-digit' 
            });
        },
        /**
         * Opens the specific card in a new tab
         */
        openCard(card_id) {
            const url = generateUrl(`/apps/deck/board/${this.project.boardId}/card/${card_id}`);
            window.open(url, '_blank');
        },
    },
}
</script>

<style lang="scss" scoped>

/* --- THE GRID MAGIC --- */
.project-details-container {
  width: 100%;
  margin: 20px auto 60px auto;
  font-family: var(--font-family, -apple-system, BlinkMacSystemFont, sans-serif);
}

.top-grid {
  display: grid;
  /* - minmax(350px, 1fr): Cards will shrink to 350px.
     - If screen is wide (>1500px), all 4 cards fit on one line.
     - If screen is smaller, they wrap automatically.
  */
  grid-template-columns: repeat(auto-fit, minmax(600px, 1fr));
  gap: 20px;
  width: 100%;
}

.bottom-row {
  margin-top: 20px;
  /* "Always separate line with only half page space" */
  width: 50%;
  min-width: 400px; /* Prevent it from getting too skinny on small screens */
}

@media (max-width: 768px) {
  .bottom-row {
    width: 100%; /* On mobile, let it take full width */
  }
}

/* Ensure individual cards take full height of the row */
.section-wrapper {
  background: white; /* or your theme color */
  border-radius: 8px; /* Optional rounded corners */
  display: flex;
  flex-direction: column;
  height: 100%; /* Important for grid height matching */
}

.detail-card {
  flex-grow: 1; /* Ensures cards align nicely in height */
}

.wheels-row {
  display: flex;
  justify-content: space-around;
  align-items: center;
  padding: 20px 0;
  height: 100%;
}

/* =========================================
   1. MODERN DASHBOARD STYLES (Project Details)
   ========================================= */

/* HEADERS */
.section-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
    padding: 0 4px;
}

.header-actions {
    display: flex;
    gap: 8px;
}

/* CARDS */
.detail-card {
    background-color: var(--color-main-background, #fff);
    border-radius: 12px;
    padding: 24px;
    // transition: box-shadow 0.2s ease, transform 0.2s ease;
}

// .detail-card:hover {
//     box-shadow: 0 4px 12px rgba(0,0,0,0.05);
//     border-color: #dcdcdc;
// }

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

/* WORKSPACE STYLES */
.workspace-card {
    background-color: var(--color-main-background, #fff);
    border: 1px solid var(--color-border, #ededed);
    border-radius: 12px;
    overflow: hidden; /* Ensures iframe corners match border-radius */
    margin-bottom: 24px;
}

.workspace-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 16px 24px;
    background-color: var(--color-background-hover); /* Slight contrast for header */
    border-bottom: 1px solid var(--color-border);
}

.workspace-title {
    font-weight: 600;
    font-size: 15px;
    color: var(--color-main-text);
    display: flex;
    align-items: center;
    gap: 8px;
}

.workspace-link {
    font-size: 13px;
    color: var(--color-primary);
    text-decoration: none;
    font-weight: 500;
}

.iframe-container {
    position: relative;
    width: 100%;
    height: 600px; /* Fixed height for the board view */
}

.iframe-loader {
    position: absolute;
    top: 0; left: 0; width: 100%; height: 100%;
    display: flex; flex-direction: column; align-items: center; justify-content: center;
    color: var(--color-text-maxcontrast);
    gap: 10px;
    z-index: 10; /* Sits above iframe */
    background-color: var(--color-main-background);
}

.embedded-frame {
    width: 100%;
    height: 100%;
    border: none;
    display: block;
}

.icon-deck {
    /* You can use a Nextcloud icon class here or an SVG */
    display: inline-block;
    width: 20px;
    height: 20px;
    background-image: var(--icon-deck-000); /* Requires deck app css loaded, or use generic icon */
    background-size: contain;
    background-repeat: no-repeat;
}

/* --- TAB SWITCHER (Pills) --- */
.tab-pills {
    display: flex;
    background-color: #f4f4f4;
    padding: 4px;
    border-radius: 8px;
    gap: 4px;
}

.tab-item {
    border: none;
    background: transparent;
    padding: 6px 16px;
    font-size: 13px;
    font-weight: 600;
    color: #666;
    border-radius: 6px;
    cursor: pointer;
    transition: all 0.2s ease;
}

.tab-item:hover {
    background-color: rgba(0,0,0,0.05);
    color: #333;
}

.tab-item.active {
    background-color: #ffffff;
    color: var(--color-primary);
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

/* --- MAIN CARD & CONTAINER --- */
.communication-card {
    padding: 0; /* Edge to edge */
    overflow: hidden;
    height: 450px; /* Fixed height for scrolling */
    display: flex;
    flex-direction: column;
}

.feed-container {
    flex-grow: 1;
    overflow-y: auto;
    padding: 20px;
    background-color: #ffffff;
}

.feed-loader {
    height: 100%;
    display: flex;
    align-items: center;
    justify-content: center;
}

/* --- NOTES FEED STYLES --- */
.note-card {
    background-color: #fffde7; /* Sticky Note Yellow */
    border: 1px solid #f9f1a5;
    padding: 14px;
    border-radius: 8px;
    margin-bottom: 12px;
    box-shadow: 0 1px 2px rgba(0,0,0,0.03);
    transition: transform 0.2s;
}

.note-card:hover {
    transform: translateY(-1px);
    box-shadow: 0 3px 6px rgba(0,0,0,0.05);
}

.note-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 8px;
    padding-bottom: 6px;
    border-bottom: 1px solid rgba(0,0,0,0.05);
}

.note-date {
    font-size: 11px;
    color: #888;
    font-weight: 600;
}

.note-body {
    font-size: 14px;
    color: #333;
    line-height: 1.5;
}

.icon-note {
    font-size: 14px;
}

/* --- ACTIVITY FEED STYLES --- */
.feed-row {
    display: flex;
    gap: 12px;
    margin-bottom: 16px;
    align-items: flex-start;
    width: 100%; /* Ensure the row itself is full width */
}

.feed-avatar {
    margin-top: 4px;
    flex-shrink: 0;
}

.feed-bubble {
    background-color: #f5f7f9;
    padding: 12px 16px;
    border-radius: 4px 12px 12px 12px;
    font-size: 14px;
    color: var(--color-main-text);
    
    /* FULL WIDTH FIX */
    flex-grow: 1;    /* Takes up all remaining space */
    width: 100%;     /* Forces full width */
    max-width: none; /* Removes the previous 85% limit */
}

.feed-header {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    margin-bottom: 6px;
    font-size: 12px;
    gap: 12px;
}

.feed-meta-left {
    display: flex;
    flex-wrap: wrap;
    align-items: center;
    gap: 6px;
}

.feed-user {
    font-weight: 700;
    color: #222;
}

/* Navigation Context (e.g. "on Card Name") */
.feed-context {
    color: #666;
    display: inline-flex;
    align-items: center;
    gap: 4px;
}

.card-link {
    color: var(--color-primary);
    font-weight: 600;
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    gap: 4px;
    background-color: rgba(0,0,0,0.04);
    padding: 2px 6px;
    border-radius: 4px;
    transition: background-color 0.2s;
}

.card-link:hover {
    background-color: rgba(0,0,0,0.08);
    text-decoration: none;
}

.feed-date {
    color: #888;
    font-size: 11px;
    white-space: nowrap;
    flex-shrink: 0;
}

.feed-message {
    font-size: 14px;
    color: #444;
    line-height: 1.6;
    white-space: pre-wrap;
    word-break: break-word; /* Prevents long words from overflowing */
}

.inline-spaced {
    display: flex;
    flex-direction: row;
    justify-content: space-between;
    gap: 15px;
}

.flex-1 {
    flex: 1;
}

.modern-header {
    margin: 10px !important;
}

</style>