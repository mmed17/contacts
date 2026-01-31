<template>
    <div class="drasci-roles-section">
        <div class="section-header">
            <h3 class="modern-header">{{ t('contacts', 'D-RASCI-VF Roles') }}</h3>
        </div>

        <div class="detail-card">
            <div v-if="loading" class="loading-state">
                <NcLoadingIcon :size="32" />
                <p>{{ t('contacts', 'Loading roles...') }}</p>
            </div>

            <div v-else-if="groupedByUser.length === 0" class="empty-state">
                <p>{{ t('contacts', 'No D-RASCI-VF roles configured for this project.') }}</p>
                <p class="hint">{{ t('contacts', 'Role assignments can be configured from the Deck board.') }}</p>
            </div>

            <div v-else class="roles-grid">
                <div v-for="userGroup in groupedByUser" :key="userGroup.participant" class="user-row">
                    <div class="user-info">
                        <NcAvatar :user="userGroup.participant" :size="32" />
                        <span class="user-name">{{ userGroup.displayName || userGroup.participant }}</span>
                    </div>
                    <div class="transitions-list">
                        <div v-for="transition in userGroup.transitions" :key="transition.key" class="transition-item">
                            <span class="transition-label">
                                {{ transition.fromStackTitle || getStackName(transition.fromStackId) || t('contacts', 'Any') }}
                                →
                                {{ transition.toStackTitle || getStackName(transition.toStackId) }}
                            </span>
                            <div class="role-badges">
                                <span v-for="role in transition.roles" :key="role.id"
                                      class="role-badge" :class="`role-${role.requiredRole}`"
                                      :title="getRoleDescription(role.requiredRole)">
                                    {{ getRoleLabel(role.requiredRole) }}
                                </span>
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <!-- Role Reference Guide -->
        <details class="role-guide">
            <summary>{{ t('contacts', 'D-RASCI-VF Role Guide') }}</summary>
            <div class="role-grid">
                <div v-for="role in roleOptions" :key="role.value" class="role-item">
                    <span class="role-badge" :class="`role-${role.value}`">{{ role.value }}</span>
                    <div class="role-info">
                        <strong>{{ role.label }}</strong>
                        <p>{{ role.description }}</p>
                    </div>
                </div>
            </div>
        </details>
    </div>
</template>

<script>
import { generateUrl } from '@nextcloud/router'
import axios from '@nextcloud/axios'
import { t } from '@nextcloud/l10n'

import { NcAvatar, NcLoadingIcon } from '@nextcloud/vue'

export default {
    name: 'CircleDRASCIRoles',

    components: {
        NcAvatar,
        NcLoadingIcon,
    },

    props: {
        boardId: {
            type: Number,
            required: true,
        },
        members: {
            type: Array,
            default: () => [],
        },
    },

    data() {
        return {
            loading: true,
            permissions: [],
            stacks: [],
            roleOptions: [
                { value: 'D', label: t('contacts', 'Driver'), description: t('contacts', 'Initiates and drives the work forward') },
                { value: 'R', label: t('contacts', 'Responsible'), description: t('contacts', 'Does the actual work') },
                { value: 'A', label: t('contacts', 'Accountable'), description: t('contacts', 'Ultimately answerable for completion') },
                { value: 'S', label: t('contacts', 'Support'), description: t('contacts', 'Provides support and resources') },
                { value: 'C', label: t('contacts', 'Consulted'), description: t('contacts', 'Provides input and feedback') },
                { value: 'I', label: t('contacts', 'Informed'), description: t('contacts', 'Kept informed of progress') },
                { value: 'V', label: t('contacts', 'Verify'), description: t('contacts', 'Verifies that work meets requirements') },
                { value: 'F', label: t('contacts', 'Final Approver'), description: t('contacts', 'Gives final approval') },
            ],
        }
    },

    computed: {
        groupedByUser() {
            const groups = {}

            for (const perm of this.permissions) {
                const userId = perm.participant
                if (!groups[userId]) {
                    const member = this.members.find(m => m.userId === userId)
                    groups[userId] = {
                        participant: userId,
                        displayName: member?.displayName || userId,
                        transitions: {},
                    }
                }

                const transitionKey = `${perm.fromStackId || 'any'}-${perm.toStackId}`
                if (!groups[userId].transitions[transitionKey]) {
                    groups[userId].transitions[transitionKey] = {
                        key: transitionKey,
                        fromStackId: perm.fromStackId,
                        toStackId: perm.toStackId,
                        fromStackTitle: perm.fromStackTitle, // Pre-fetched from backend
                        toStackTitle: perm.toStackTitle,     // Pre-fetched from backend
                        roles: [],
                    }
                }

                groups[userId].transitions[transitionKey].roles.push({
                    id: perm.id,
                    requiredRole: perm.requiredRole,
                })
            }

            // Convert transitions object to array for each user
            return Object.values(groups).map(user => ({
                ...user,
                transitions: Object.values(user.transitions),
            }))
        },
    },

    watch: {
        boardId: {
            handler(val) {
                if (val) {
                    this.loadData()
                }
            },
            immediate: true,
        },
    },

    methods: {
        t,

        async loadData() {
            this.loading = true
            try {
                await Promise.all([
                    this.loadPermissions(),
                    this.loadStacks(),
                ])
            } finally {
                this.loading = false
            }
        },

        async loadPermissions() {
            try {
                const url = generateUrl(`/apps/deck/boards/${this.boardId}/transition-permissions`)
                const response = await axios.get(url)
                this.permissions = response.data.ocs?.data || response.data || []
            } catch (error) {
                console.error('Error loading D-RASCI-VF permissions:', error)
                this.permissions = []
            }
        },

        async loadStacks() {
            try {
                const url = generateUrl(`/apps/deck/api/v1/boards/${this.boardId}/stacks`)
                const response = await axios.get(url)
                this.stacks = response.data.ocs?.data || response.data || []
            } catch (error) {
                console.error('Error loading stacks:', error)
                this.stacks = []
            }
        },

        getStackName(stackId) {
            if (!stackId) return null
            // Convert to string for comparison in case of type mismatch
            const stack = this.stacks.find(s => String(s.id) === String(stackId))
            return stack?.title || `Stack #${stackId}`
        },

        getRoleLabel(roleValue) {
            const role = this.roleOptions.find(r => r.value === roleValue)
            return role?.label || roleValue
        },

        getRoleDescription(roleValue) {
            const role = this.roleOptions.find(r => r.value === roleValue)
            return role?.description || ''
        },
    },
}
</script>

<style lang="scss" scoped>
.drasci-roles-section {
    margin-top: 24px;
}

.section-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 12px;

    .modern-header {
        margin: 0;
        font-size: 18px;
        font-weight: 600;
    }
}

.detail-card {
    background: var(--color-main-background);
    border: 1px solid var(--color-border);
    border-radius: 12px;
    padding: 20px;
}

.loading-state,
.empty-state {
    text-align: center;
    padding: 32px;
    color: var(--color-text-maxcontrast);

    p {
        margin: 0;
    }

    .hint {
        margin-top: 8px;
        font-size: 13px;
    }
}

.roles-grid {
    display: flex;
    flex-direction: column;
    gap: 16px;
}

.user-row {
    display: flex;
    gap: 16px;
    padding: 12px;
    background: var(--color-background-hover);
    border-radius: 8px;

    .user-info {
        display: flex;
        align-items: center;
        gap: 12px;
        min-width: 180px;

        .user-name {
            font-weight: 500;
        }
    }

    .transitions-list {
        flex: 1;
        display: flex;
        flex-wrap: wrap;
        gap: 12px;
    }

    .transition-item {
        display: flex;
        flex-direction: column;
        gap: 6px;
        padding: 8px 12px;
        background: var(--color-main-background);
        border-radius: 6px;
        border: 1px solid var(--color-border-dark);

        .transition-label {
            font-size: 12px;
            color: var(--color-text-maxcontrast);
        }

        .role-badges {
            display: flex;
            flex-wrap: wrap;
            gap: 4px;
        }
    }
}

.role-badge {
    display: inline-flex;
    align-items: center;
    padding: 4px 10px;
    border-radius: 4px;
    font-size: 12px;
    font-weight: 600;

    &.role-D { background: #e91e63; color: white; }
    &.role-R { background: #9c27b0; color: white; }
    &.role-A { background: #3f51b5; color: white; }
    &.role-S { background: #00bcd4; color: white; }
    &.role-C { background: #4caf50; color: white; }
    &.role-I { background: #8bc34a; color: white; }
    &.role-V { background: #ff9800; color: white; }
    &.role-F { background: #f44336; color: white; }
}

.role-guide {
    margin-top: 16px;

    summary {
        cursor: pointer;
        font-size: 13px;
        color: var(--color-text-maxcontrast);
        padding: 8px 0;

        &:hover {
            color: var(--color-main-text);
        }
    }

    .role-grid {
        display: grid;
        grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
        gap: 12px;
        padding: 12px;
        background: var(--color-background-dark);
        border-radius: 8px;
        margin-top: 8px;
    }

    .role-item {
        display: flex;
        align-items: flex-start;
        gap: 8px;

        .role-info {
            strong {
                display: block;
                font-size: 13px;
            }

            p {
                margin: 4px 0 0;
                font-size: 11px;
                color: var(--color-text-maxcontrast);
            }
        }
    }
}
</style>
