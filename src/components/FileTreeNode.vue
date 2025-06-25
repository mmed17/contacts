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
                            <EyeOutline :size="20" />
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
import { getCurrentUser } from '@nextcloud/auth';
import { NcListItem, NcActionButton, NcButton } from '@nextcloud/vue';
import { generateUrl, generateRemoteUrl } from '@nextcloud/router';
import { t, n } from '@nextcloud/l10n';
import FolderIcon from 'vue-material-design-icons/Folder.vue';
import FileIcon from 'vue-material-design-icons/File.vue';
import EyeOutline from 'vue-material-design-icons/EyeOutline.vue';
import Download from 'vue-material-design-icons/Download.vue';

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
        NcButton
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
            isExpanded: false
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
		onPreview() {
            const url = `/apps/files/files/${this.node.id}?dir=/${this.node.name}`;
            window.location.href = generateUrl(url);
		},
		onDownload() {
            const apath = this.node.path;
            const parts = apath.split('/');
            
            [parts[1], parts[2]] = [parts[2], parts[1]];
            const newPath = parts.join('/');

            const url = new URL(client.getFileDownloadLink(newPath));
            const hiddenElement = document.createElement('a');

            if(this.node.type === 'folder') {
                url.searchParams.append('accept', 'zip');
            } else {
                hiddenElement.download = this.node.name;
            }
            
            hiddenElement.href = url.href;
            hiddenElement.click();
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
