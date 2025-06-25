<template>
	<div class="file-tree-node">
		<ul>
            <NcListItem :name="node.name" :force-display-actions="true" @click="toggleExpand">
                <template #icon>
                    <component :is="iconComponent" :size="32" class="main-icon" />
                </template>

                <template #subname v-if="isFolder">
                    <span v-if="node.isEmpty">{{ t('contacts', 'Empty') }}</span>
                    <span v-else>{{ n('contacts', '%n item', '%n items', node.children.length) }}</span>
                </template>

                <template #extra-actions>
                    <NcButton variant="tertiary-no-background" @click="onPreview">
                        <template #icon>
                            <NcLoadingIcon v-if="isPreviewing" :size="20" />
                            <EyeOutline v-else :size="20" />
                        </template>
                    </NcButton>
                    <NcButton variant="tertiary-no-background" @click="onDownload">
                        <template #icon>
                            <Download :size="20" />
                        </template>
                    </NcButton>
                </template>
            </NcListItem>
        </ul>
        <div v-if="isFolderAndExpanded && isExpanded" class="file-tree-node__children">
            <FileTreeNode
                v-for="child in node.children"
                :key="child.id"
                :node="child" />
        </div>
	</div>
</template>

<script>
import { createClient } from 'webdav'
import { NcListItem, NcActionButton, NcButton } from '@nextcloud/vue';
import { generateUrl, generateRemoteUrl } from '@nextcloud/router';
import { t, n } from '@nextcloud/l10n';
import FolderIcon from 'vue-material-design-icons/Folder.vue';
import FileIcon from 'vue-material-design-icons/File.vue';
import EyeOutline from 'vue-material-design-icons/EyeOutline.vue';
import Download from 'vue-material-design-icons/Download.vue';

import { NcLoadingIcon } from '@nextcloud/vue'

const client = createClient(generateRemoteUrl('dav'));

export default {
	name: 'FileTreeNode',
	components: {
		NcListItem,
		NcActionButton,
		FolderIcon,
		FileIcon,
        EyeOutline,
        Download,
        NcButton,
        NcLoadingIcon
	},
	props: {
		node: {
			type: Object,
			required: true,
		}
	},
	data() {
		return { 
            t, 
            n,
            isExpanded: false,
            isPreviewing: false
        };
	},
	computed: {
        isFolder() {
			return this.node.type === 'folder';
		},
		isFolderAndExpanded() {
			return this.node.type === 'folder' && this.node.children?.length > 0;
		},
		iconComponent() {
			return this.node.type === 'folder' ? 'FolderIcon' : 'FileIcon';
		}
	},
	methods: {
        async onPreview() {
            if (this.node.type === 'folder') {
                this.navigateToFolder(this.node.id, this.node.name);
            } else {
                this.isPreviewing = true;
                await this.previewFile(this.node.path, this.node.mimetype);
                this.isPreviewing = false;
            }
        },
        onDownload() {
            const path = this.normalizedPath(this.node.path);
            const downloadUrl = new URL(client.getFileDownloadLink(path));

            if (this.node.type === 'folder') {
                downloadUrl.searchParams.append('accept', 'zip');
            }

            this.triggerDownload(downloadUrl.href, this.node.type !== 'folder' ? this.node.name : null);
        },
        navigateToFolder(id, name) {
            const url = generateUrl(`/apps/files/files/${id}?dir=/${name}`);
            window.location.href = url;
        },
        async previewFile(path, mimetype) {
            const normalized = this.normalizedPath(path);
            const buffer = await client.getFileContents(normalized, { format: 'binary' });
            const blob = new Blob([buffer], { type: mimetype });
            const blobUrl = URL.createObjectURL(blob);

            window.open(blobUrl, '_blank');

            setTimeout(() => URL.revokeObjectURL(blobUrl), 1000);
        },
        triggerDownload(href, filename = null) {
            const link = document.createElement('a');
            link.href = href;
            if (filename) link.download = filename;
            link.style.display = 'none';
            document.body.appendChild(link);
            link.click();
            link.remove();
        },
        normalizedPath(path) {
            const parts = path.split('/');
            if (parts.length >= 3) {
                [parts[1], parts[2]] = [parts[2], parts[1]];
            }
            return parts.join('/');
        },
        toggleExpand() {
            this.isExpanded = !this.isExpanded;
        }
	},
};
</script>

<style lang="scss" scoped>
.file-tree-node__children {
	margin-left: 40px;
	border-left: 1px solid var(--color-border);
}
</style>
