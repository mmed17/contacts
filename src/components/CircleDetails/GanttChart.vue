<template>
    <div class="gantt-chart-section">
        <div class="section-header">
            <h3 class="modern-header">{{ t('contacts', 'Project Timeline') }}</h3>
            <div class="header-actions">
                <!-- Navigation Controls -->
                <div class="timeline-nav">
                    <Button type="tertiary" @click="navigatePrev" :aria-label="t('contacts', 'Previous')">
                        <template #icon><ChevronLeft :size="20" /></template>
                    </Button>
                    <Button type="tertiary" @click="navigateToday" class="today-btn">
                        {{ t('contacts', 'Today') }}
                    </Button>
                    <Button type="tertiary" @click="navigateNext" :aria-label="t('contacts', 'Next')">
                        <template #icon><ChevronRight :size="20" /></template>
                    </Button>
                </div>
                <!-- Add Phase Button (Always visible for admins) -->
                <Button v-if="isAdmin" type="primary" @click="openAddModal">
                    <template #icon><Plus :size="16" /></template>
                    {{ t('contacts', 'Add Phase') }}
                </Button>
            </div>
        </div>

        <div class="detail-card">
            <div v-if="loading" class="loading-state">
                <NcLoadingIcon :size="32" />
                <p>{{ t('contacts', 'Loading timeline...') }}</p>
            </div>

            <div v-else-if="items.length === 0" class="empty-state">
                <p>{{ t('contacts', 'No timeline phases defined yet.') }}</p>
                <p class="empty-hint">{{ t('contacts', 'Click "Add Phase" in the top right to get started.') }}</p>
            </div>

            <div v-else class="gantt-container" ref="ganttContainer">
                <!-- Timeline header with adaptive scale -->
                <div class="gantt-header">
                    <div class="gantt-label-col">{{ t('contacts', 'Phase') }}</div>
                    <div class="gantt-timeline-col" ref="timelineCol">
                        <div class="timeline-content" 
                             :style="{ width: totalTimelineWidth + 'px', transform: `translateX(${scrollOffset}px)` }">
                            <!-- Year row (shown when spanning multiple years) -->
                            <div v-if="spanMultipleYears" class="year-row">
                                <span v-for="year in visibleYears" :key="year.key"
                                      class="year-label"
                                      :style="{ width: year.width + 'px' }">
                                    {{ year.label }}
                                </span>
                            </div>
                            <!-- Month row -->
                            <div class="month-row">
                                <span v-for="month in visibleMonths" :key="month.key"
                                      class="month-label"
                                      :class="{ 'compact': month.width < 60 }"
                                      :style="{ width: month.width + 'px' }">
                                    {{ month.width < 40 ? '' : month.label }}
                                </span>
                            </div>
                            <!-- Today marker -->
                            <div v-if="showTodayMarker" 
                                 class="today-marker" 
                                 :style="{ left: todayOffset + 'px' }">
                                <div class="today-line"></div>
                                <span class="today-label">{{ t('contacts', 'Today') }}</span>
                            </div>
                        </div>
                    </div>
                </div>


                <!-- Timeline rows -->
                <div v-for="item in items" :key="item.id" class="gantt-row">
                    <div class="gantt-label-col">
                        <span class="phase-name">{{ item.label }}</span>
                        <span class="phase-dates">
                            {{ formatDate(item.startDate) }} - {{ formatDate(item.endDate) }}
                        </span>
                    </div>
                    <div class="gantt-timeline-col">
                        <div class="timeline-content" 
                             :style="{ width: totalTimelineWidth + 'px', transform: `translateX(${scrollOffset}px)` }">
                            <div class="timeline-track">
                                <div class="timeline-bar"
                                     :style="getBarStyle(item)"
                                     :title="`${item.label}: ${formatDate(item.startDate)} - ${formatDate(item.endDate)}`">
                                </div>
                            </div>
                        </div>
                    </div>
                    <div v-if="isAdmin" class="gantt-actions-col">
                        <div class="reorder-buttons">
                            <Button type="tertiary" 
                                    @click="movePhaseUp(item)" 
                                    :disabled="getItemIndex(item) === 0"
                                    :aria-label="t('contacts', 'Move up')">
                                <template #icon><ChevronUp :size="16" /></template>
                            </Button>
                            <Button type="tertiary" 
                                    @click="movePhaseDown(item)" 
                                    :disabled="getItemIndex(item) === items.length - 1"
                                    :aria-label="t('contacts', 'Move down')">
                                <template #icon><ChevronDown :size="16" /></template>
                            </Button>
                        </div>
                        <Button type="tertiary" @click="openEditModal(item)" :aria-label="t('contacts', 'Edit')">
                            <template #icon><Pencil :size="16" /></template>
                        </Button>
                        <Button type="error" @click="confirmDelete(item)" :aria-label="t('contacts', 'Delete')">
                            <template #icon><Delete :size="16" /></template>
                        </Button>
                    </div>
                </div>

                <!-- Phase quick jump navigation -->
                <div v-if="items.length > 1" class="phase-jumper">
                    <span class="jumper-label">{{ t('contacts', 'Jump to:') }}</span>
                    <button v-for="(item, index) in items" 
                            :key="item.id"
                            class="phase-chip"
                            :class="{ active: currentPhaseIndex === index }"
                            :style="{ '--phase-color': item.color }"
                            @click="navigateToPhase(index)">
                        {{ item.label }}
                    </button>
                </div>
            </div>
        </div>

        <!-- Add/Edit Modal -->
        <NcModal v-if="showModal" @close="closeModal">
            <div class="phase-form">
                <h3>{{ editingItem ? t('contacts', 'Edit Phase') : t('contacts', 'Add Phase') }}</h3>
                
                <div class="form-field">
                    <label>{{ t('contacts', 'Phase Name') }}</label>
                    <NcTextField :value.sync="form.label" :placeholder="t('contacts', 'e.g., Development, QA, Production')" />
                </div>

                <div class="form-row">
                    <div class="form-field">
                        <label>{{ t('contacts', 'Start Date') }}</label>
                        <NcTextField type="date" :value.sync="form.startDate" />
                    </div>
                    <div class="form-field">
                        <label>{{ t('contacts', 'End Date') }}</label>
                        <NcTextField type="date" :value.sync="form.endDate" />
                    </div>
                </div>

                <div class="form-field">
                    <label>{{ t('contacts', 'Color') }}</label>
                    <div class="color-picker">
                        <button v-for="color in colorOptions" :key="color"
                                class="color-option"
                                :class="{ selected: form.color === color }"
                                :style="{ backgroundColor: color }"
                                @click="form.color = color">
                        </button>
                    </div>
                </div>

                <div class="form-actions">
                    <Button @click="closeModal">{{ t('contacts', 'Cancel') }}</Button>
                    <Button type="primary" @click="saveItem" :disabled="!canSave || saving">
                        {{ saving ? t('contacts', 'Saving...') : t('contacts', 'Save') }}
                    </Button>
                </div>
            </div>
        </NcModal>
    </div>
</template>


<script>
import { generateUrl } from '@nextcloud/router'
import { showError, showSuccess } from '@nextcloud/dialogs'
import axios from '@nextcloud/axios'
import { t } from '@nextcloud/l10n'

import { NcButton as Button, NcLoadingIcon, NcModal, NcTextField } from '@nextcloud/vue'
import Plus from 'vue-material-design-icons/Plus.vue'
import Pencil from 'vue-material-design-icons/Pencil.vue'
import Delete from 'vue-material-design-icons/Delete.vue'
import ChevronLeft from 'vue-material-design-icons/ChevronLeft.vue'
import ChevronRight from 'vue-material-design-icons/ChevronRight.vue'
import ChevronUp from 'vue-material-design-icons/ChevronUp.vue'
import ChevronDown from 'vue-material-design-icons/ChevronDown.vue'

export default {
    name: 'GanttChart',

    components: {
        Button,
        NcLoadingIcon,
        NcModal,
        NcTextField,
        Plus,
        Pencil,
        Delete,
        ChevronLeft,
        ChevronRight,
        ChevronUp,
        ChevronDown,
    },

    props: {
        projectId: {
            type: Number,
            required: true,
        },
        isAdmin: {
            type: Boolean,
            default: false,
        },
    },

    data() {
        return {
            loading: true,
            saving: false,
            items: [],
            showModal: false,
            editingItem: null,
            form: {
                label: '',
                startDate: '',
                endDate: '',
                color: '#3b82f6',
            },
            colorOptions: [
                '#3b82f6', // Blue
                '#10b981', // Green
                '#f59e0b', // Amber
                '#ef4444', // Red
                '#8b5cf6', // Purple
                '#ec4899', // Pink
                '#06b6d4', // Cyan
                '#84cc16', // Lime
            ],
            dayWidth: 3, // pixels per day
            scrollOffset: 0, // Horizontal scroll offset in pixels
            currentPhaseIndex: 0, // Currently focused phase
        }
    },

    computed: {
        canSave() {
            return this.form.label.trim() !== '' 
                && this.form.startDate !== '' 
                && this.form.endDate !== ''
        },

        timelineRange() {
            if (this.items.length === 0) {
                const now = new Date()
                return {
                    start: new Date(now.getFullYear(), now.getMonth(), 1),
                    end: new Date(now.getFullYear(), now.getMonth() + 6, 0),
                }
            }

            let minDate = null
            let maxDate = null

            for (const item of this.items) {
                const start = new Date(item.startDate)
                const end = new Date(item.endDate)

                if (!minDate || start < minDate) minDate = start
                if (!maxDate || end > maxDate) maxDate = end
            }

            // Add padding of 1 month on each side
            const paddedStart = new Date(minDate)
            paddedStart.setMonth(paddedStart.getMonth() - 1)
            paddedStart.setDate(1)

            const paddedEnd = new Date(maxDate)
            paddedEnd.setMonth(paddedEnd.getMonth() + 2)
            paddedEnd.setDate(0)

            return { start: paddedStart, end: paddedEnd }
        },

        totalDays() {
            const { start, end } = this.timelineRange
            return Math.ceil((end - start) / (1000 * 60 * 60 * 24))
        },

        totalTimelineWidth() {
            return this.totalDays * this.dayWidth
        },

        visibleMonths() {
            const months = []
            const { start, end } = this.timelineRange
            const current = new Date(start)

            while (current <= end) {
                const monthStart = new Date(current.getFullYear(), current.getMonth(), 1)
                const monthEnd = new Date(current.getFullYear(), current.getMonth() + 1, 0)
                
                const effectiveStart = monthStart < start ? start : monthStart
                const effectiveEnd = monthEnd > end ? end : monthEnd
                
                const days = Math.ceil((effectiveEnd - effectiveStart) / (1000 * 60 * 60 * 24)) + 1
                const width = days * this.dayWidth

                // Use shorter label when in multi-year mode (year shown separately)
                const labelFormat = this.spanMultipleYears 
                    ? { month: 'short' }
                    : { month: 'short', year: 'numeric' }

                months.push({
                    key: `${current.getFullYear()}-${current.getMonth()}`,
                    label: current.toLocaleDateString('default', labelFormat),
                    width: width,
                    year: current.getFullYear(),
                })

                current.setMonth(current.getMonth() + 1)
            }

            return months
        },

        spanMultipleYears() {
            const { start, end } = this.timelineRange
            return start.getFullYear() !== end.getFullYear()
        },

        visibleYears() {
            if (!this.spanMultipleYears) return []
            
            const years = []
            const { start, end } = this.timelineRange
            const startYear = start.getFullYear()
            const endYear = end.getFullYear()

            for (let year = startYear; year <= endYear; year++) {
                // Calculate the width for this year
                const yearStart = new Date(Math.max(start.getTime(), new Date(year, 0, 1).getTime()))
                const yearEnd = new Date(Math.min(end.getTime(), new Date(year, 11, 31).getTime()))
                
                const days = Math.ceil((yearEnd - yearStart) / (1000 * 60 * 60 * 24)) + 1
                
                years.push({
                    key: `year-${year}`,
                    label: year.toString(),
                    width: days * this.dayWidth,
                })
            }

            return years
        },

        showTodayMarker() {
            if (this.items.length === 0) return false
            const { start, end } = this.timelineRange
            const today = new Date()
            return today >= start && today <= end
        },

        todayOffset() {
            const { start } = this.timelineRange
            const today = new Date()
            const days = Math.ceil((today - start) / (1000 * 60 * 60 * 24))
            return days * this.dayWidth
        },
    },

    watch: {
        projectId: {
            handler(val) {
                if (val) {
                    this.loadItems()
                }
            },
            immediate: true,
        },
    },

    methods: {
        t,

        async loadItems() {
            this.loading = true
            try {
                const url = generateUrl(`/apps/projectcreatoraio/api/v1/projects/${this.projectId}/timeline`)
                const response = await axios.get(url)
                this.items = response.data || []
            } catch (error) {
                console.error('Error loading timeline:', error)
                this.items = []
            } finally {
                this.loading = false
            }
        },

        formatDate(dateStr) {
            if (!dateStr) return '-'
            const date = new Date(dateStr)
            const day = date.getDate().toString().padStart(2, '0')
            const month = (date.getMonth() + 1).toString().padStart(2, '0')
            const year = date.getFullYear()
            return `${day}/${month}/${year}`
        },

        getBarStyle(item) {
            const { start } = this.timelineRange
            const itemStart = new Date(item.startDate)
            const itemEnd = new Date(item.endDate)

            const offsetDays = Math.ceil((itemStart - start) / (1000 * 60 * 60 * 24))
            const durationDays = Math.ceil((itemEnd - itemStart) / (1000 * 60 * 60 * 24)) + 1
            const leftPx = offsetDays * this.dayWidth

            return {
                left: `${leftPx}px`,
                width: `${durationDays * this.dayWidth}px`,
                backgroundColor: item.color || '#3b82f6',
            }
        },

        openAddModal() {
            this.editingItem = null
            this.form = {
                label: '',
                startDate: '',
                endDate: '',
                color: '#3b82f6',
            }
            this.showModal = true
        },

        openEditModal(item) {
            this.editingItem = item
            this.form = {
                label: item.label,
                startDate: item.startDate,
                endDate: item.endDate,
                color: item.color || '#3b82f6',
            }
            this.showModal = true
        },

        closeModal() {
            this.showModal = false
            this.editingItem = null
        },

        async saveItem() {
            if (!this.canSave) return

            this.saving = true
            try {
                const baseUrl = generateUrl(`/apps/projectcreatoraio/api/v1/projects/${this.projectId}/timeline`)

                if (this.editingItem) {
                    // Update
                    await axios.put(`${baseUrl}/${this.editingItem.id}`, this.form)
                    showSuccess(t('contacts', 'Phase updated'))
                } else {
                    // Create
                    await axios.post(baseUrl, this.form)
                    showSuccess(t('contacts', 'Phase added'))
                }

                this.closeModal()
                await this.loadItems()
            } catch (error) {
                console.error('Error saving phase:', error)
                showError(t('contacts', 'Failed to save phase'))
            } finally {
                this.saving = false
            }
        },

        async confirmDelete(item) {
            if (!confirm(t('contacts', 'Are you sure you want to delete "{label}"?', { label: item.label }))) {
                return
            }

            try {
                const url = generateUrl(`/apps/projectcreatoraio/api/v1/projects/${this.projectId}/timeline/${item.id}`)
                await axios.delete(url)
                showSuccess(t('contacts', 'Phase deleted'))
                await this.loadItems()
            } catch (error) {
                console.error('Error deleting phase:', error)
                showError(t('contacts', 'Failed to delete phase'))
            }
        },

        // Navigation methods
        navigatePrev() {
            // Move back by approximately 1 month (30 days worth of pixels)
            const monthPx = 30 * this.dayWidth
            this.scrollOffset = Math.min(0, this.scrollOffset + monthPx)
        },

        navigateNext() {
            // Move forward by approximately 1 month
            const monthPx = 30 * this.dayWidth
            const maxScroll = -(this.totalDays * this.dayWidth - 400) // Keep at least 400px visible
            this.scrollOffset = Math.max(maxScroll, this.scrollOffset - monthPx)
        },

        navigateToday() {
            if (!this.showTodayMarker) {
                // Today is outside the range, reset to start
                this.scrollOffset = 0
                return
            }
            // Center on today
            const containerWidth = this.$refs.timelineCol?.offsetWidth || 400
            this.scrollOffset = -(this.todayOffset - containerWidth / 2)
            // Clamp to valid range
            const maxScroll = -(this.totalDays * this.dayWidth - 400)
            this.scrollOffset = Math.max(maxScroll, Math.min(0, this.scrollOffset))
        },

        navigateToPhase(index) {
            if (index < 0 || index >= this.items.length) return

            this.currentPhaseIndex = index
            const item = this.items[index]
            const { start } = this.timelineRange
            const itemStart = new Date(item.startDate)
            const offsetDays = Math.ceil((itemStart - start) / (1000 * 60 * 60 * 24))
            const offsetPx = offsetDays * this.dayWidth

            // Center on the start of the phase
            const containerWidth = this.$refs.timelineCol?.offsetWidth || 400
            this.scrollOffset = -(offsetPx - 50) // 50px margin from left
            // Clamp to valid range
            const maxScroll = -(this.totalDays * this.dayWidth - 400)
            this.scrollOffset = Math.max(maxScroll, Math.min(0, this.scrollOffset))
        },

        // Phase reordering methods
        getItemIndex(item) {
            return this.items.findIndex(i => i.id === item.id)
        },

        async movePhaseUp(item) {
            const index = this.getItemIndex(item)
            if (index <= 0) return

            await this.swapPhaseOrder(index, index - 1)
        },

        async movePhaseDown(item) {
            const index = this.getItemIndex(item)
            if (index < 0 || index >= this.items.length - 1) return

            await this.swapPhaseOrder(index, index + 1)
        },

        async swapPhaseOrder(indexA, indexB) {
            const itemA = this.items[indexA]
            const itemB = this.items[indexB]

            if (!itemA || !itemB) return

            try {
                const baseUrl = generateUrl(`/apps/projectcreatoraio/api/v1/projects/${this.projectId}/timeline`)

                // Swap orderIndex values
                await Promise.all([
                    axios.put(`${baseUrl}/${itemA.id}`, { orderIndex: indexB }),
                    axios.put(`${baseUrl}/${itemB.id}`, { orderIndex: indexA })
                ])

                // Swap locally for immediate UI feedback
                const temp = this.items[indexA]
                this.$set(this.items, indexA, this.items[indexB])
                this.$set(this.items, indexB, temp)

                showSuccess(t('contacts', 'Phase order updated'))
            } catch (error) {
                console.error('Error reordering phases:', error)
                showError(t('contacts', 'Failed to reorder phases'))
                // Reload to get correct order
                await this.loadItems()
            }
        },
    },
}
</script>

<style lang="scss" scoped>
.gantt-chart-section {
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
    overflow-x: auto;
}

.loading-state,
.empty-state {
    text-align: center;
    padding: 32px;
    color: var(--color-text-maxcontrast);

    p {
        margin: 0 0 16px;
    }
}

.gantt-container {
    min-width: 100%;
}

.gantt-header,
.gantt-row {
    display: flex;
    align-items: stretch;
}

.gantt-label-col {
    width: 180px;
    min-width: 180px;
    padding: 12px;
    display: flex;
    flex-direction: column;
    justify-content: center;
    border-right: 1px solid var(--color-border);

    .phase-name {
        font-weight: 600;
        font-size: 14px;
    }

    .phase-dates {
        font-size: 11px;
        color: var(--color-text-maxcontrast);
        margin-top: 4px;
    }
}

.gantt-timeline-col {
    flex: 1;
    overflow: hidden;
}

.gantt-actions-col {
    display: flex;
    gap: 4px;
    padding: 8px;
    align-items: center;
}

.reorder-buttons {
    display: flex;
    flex-direction: column;
    gap: 2px;
    margin-right: 4px;

    button {
        padding: 0;
        min-height: 24px;
        min-width: 24px;

        &:disabled {
            opacity: 0.3;
            cursor: not-allowed;
        }
    }
}

.gantt-header {
    background: var(--color-background-dark);
    border-bottom: 1px solid var(--color-border);

    .gantt-label-col {
        font-weight: 600;
        font-size: 13px;
        color: var(--color-text-maxcontrast);
    }
}

/* Shared timeline content wrapper for header and rows */
.timeline-content {
    display: flex;
    flex-direction: column;
    transition: transform 0.3s ease;
    position: relative;
}

.year-row {
    display: flex;
    border-bottom: 1px solid var(--color-border);
    background: var(--color-background-darker, #e5e7eb);
}

.year-label {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 6px 8px;
    font-size: 13px;
    font-weight: 700;
    color: var(--color-main-text);
    border-right: 1px solid var(--color-border);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.month-row {
    display: flex;
}

.month-label {
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 6px 4px;
    font-size: 11px;
    font-weight: 500;
    color: var(--color-text-maxcontrast);
    border-right: 1px solid var(--color-border);
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
    min-height: 28px;

    &.compact {
        font-size: 10px;
        padding: 4px 2px;
    }
}

.gantt-row {
    border-bottom: 1px solid var(--color-border);

    &:last-child {
        border-bottom: none;
    }

    &:hover {
        background: var(--color-background-hover);
    }
}

.timeline-track {
    position: relative;
    height: 32px;
    margin: 8px 0;
    transition: transform 0.3s ease;
}

.timeline-bar {
    position: absolute;
    top: 0;
    height: 100%;
    border-radius: 4px;
    min-width: 6px;
    cursor: pointer;
    transition: transform 0.2s ease;

    &:hover {
        transform: scaleY(1.1);
    }
}

.phase-form {
    padding: 20px;
    min-width: 400px;

    h3 {
        margin: 0 0 20px;
        font-size: 18px;
    }
}

.form-field {
    margin-bottom: 16px;

    label {
        display: block;
        font-size: 13px;
        font-weight: 500;
        margin-bottom: 6px;
        color: var(--color-text-maxcontrast);
    }
}

.form-row {
    display: flex;
    gap: 16px;

    .form-field {
        flex: 1;
    }
}

.color-picker {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
}

.color-option {
    width: 32px;
    height: 32px;
    border-radius: 6px;
    border: 2px solid transparent;
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover {
        transform: scale(1.1);
    }

    &.selected {
        border-color: var(--color-main-text);
        box-shadow: 0 0 0 2px var(--color-main-background), 0 0 0 4px var(--color-primary-element);
    }
}

.form-actions {
    display: flex;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 20px;
}

/* Navigation Controls */
.header-actions {
    display: flex;
    align-items: center;
    gap: 12px;
}

.timeline-nav {
    display: flex;
    align-items: center;
    gap: 4px;
    background: var(--color-background-dark);
    border-radius: 8px;
    padding: 4px;

    .today-btn {
        min-width: auto;
        padding: 6px 12px;
        font-weight: 500;
    }
}

/* Today Marker */
.today-marker {
    position: absolute;
    top: 0;
    bottom: -100%;
    z-index: 10;
    pointer-events: none;
    transition: left 0.3s ease;
}

.today-line {
    width: 2px;
    height: 100%;
    background: var(--color-error);
    position: absolute;
    left: 0;
}

.today-label {
    position: absolute;
    top: -20px;
    left: 50%;
    transform: translateX(-50%);
    font-size: 10px;
    font-weight: 600;
    color: var(--color-error);
    background: var(--color-main-background);
    padding: 2px 6px;
    border-radius: 4px;
    white-space: nowrap;
}

/* Phase Jumper */
.phase-jumper {
    display: flex;
    align-items: center;
    gap: 8px;
    padding: 16px 12px;
    border-top: 1px solid var(--color-border);
    flex-wrap: wrap;
}

.jumper-label {
    font-size: 12px;
    font-weight: 500;
    color: var(--color-text-maxcontrast);
}

.phase-chip {
    padding: 6px 12px;
    border-radius: 16px;
    font-size: 12px;
    font-weight: 500;
    border: 1px solid var(--color-border);
    background: var(--color-main-background);
    color: var(--color-main-text);
    cursor: pointer;
    transition: all 0.2s ease;

    &:hover {
        border-color: var(--phase-color, var(--color-primary-element));
        background: var(--color-background-hover);
    }

    &.active {
        background: var(--phase-color, var(--color-primary-element));
        color: white;
        border-color: var(--phase-color, var(--color-primary-element));
    }
}

/* Empty state enhancements */
.empty-hint {
    font-size: 13px;
    color: var(--color-text-lighter);
    margin-top: 8px;
}

/* Timeline scrolling */
.month-labels {
    transition: transform 0.3s ease;
}

.gantt-timeline-col {
    position: relative;
    overflow: hidden;
}

.timeline-bar {
    transition: left 0.3s ease;
}
</style>

