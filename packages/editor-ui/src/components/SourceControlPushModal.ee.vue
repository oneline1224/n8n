<script lang="ts" setup>
import Modal from './Modal.vue';
import { SOURCE_CONTROL_PUSH_MODAL_KEY, VIEWS } from '@/constants';
import { computed, onMounted, ref } from 'vue';
import type { EventBus } from 'n8n-design-system/utils';
import { useI18n } from '@/composables/useI18n';
import { useLoadingService } from '@/composables/useLoadingService';
import { useToast } from '@/composables/useToast';
import { useSourceControlStore } from '@/stores/sourceControl.store';
import { useUIStore } from '@/stores/ui.store';
import { useRoute } from 'vue-router';
import dateformat from 'dateformat';
import { RecycleScroller } from 'vue-virtual-scroller';
import 'vue-virtual-scroller/dist/vue-virtual-scroller.css';
import type { BaseTextKey } from '@/plugins/i18n';
import { refDebounced } from '@vueuse/core';
import {
	N8nHeading,
	N8nText,
	N8nLink,
	N8nCheckbox,
	N8nInput,
	N8nIcon,
	N8nButton,
	N8nBadge,
	N8nNotice,
	N8nPopover,
	N8nSelect,
	N8nOption,
	N8nInputLabel,
} from 'n8n-design-system';
import {
	SOURCE_CONTROL_FILE_STATUS,
	SOURCE_CONTROL_FILE_TYPE,
	SOURCE_CONTROL_FILE_LOCATION,
	type SourceControlledFileStatus,
	type SourceControlAggregatedFile,
} from '@/types/sourceControl.types';
import { orderBy } from 'lodash-es';

const props = defineProps<{
	data: { eventBus: EventBus; status: SourceControlAggregatedFile[] };
}>();

const loadingService = useLoadingService();
const uiStore = useUIStore();
const toast = useToast();
const i18n = useI18n();
const sourceControlStore = useSourceControlStore();
const route = useRoute();

type Changes = {
	tags: SourceControlAggregatedFile[];
	variables: SourceControlAggregatedFile[];
	credentials: SourceControlAggregatedFile[];
	workflows: SourceControlAggregatedFile[];
	currentWorkflow?: SourceControlAggregatedFile;
};

const classifyFilesByType = (
	files: SourceControlAggregatedFile[],
	currentWorkflowId?: string,
): Changes =>
	files.reduce<Changes>(
		(acc, file) => {
			// do not show remote workflows that are not yet created locally during push
			if (
				file.location === SOURCE_CONTROL_FILE_LOCATION.REMOTE &&
				file.type === SOURCE_CONTROL_FILE_TYPE.WORKFLOW &&
				file.status === SOURCE_CONTROL_FILE_STATUS.CREATED
			) {
				return acc;
			}

			if (file.type === SOURCE_CONTROL_FILE_TYPE.VARIABLES) {
				acc.variables.push(file);
				return acc;
			}

			if (file.type === SOURCE_CONTROL_FILE_TYPE.TAGS) {
				acc.tags.push(file);
				return acc;
			}

			if (file.type === SOURCE_CONTROL_FILE_TYPE.WORKFLOW && currentWorkflowId === file.id) {
				acc.currentWorkflow = file;
			}

			if (file.type === SOURCE_CONTROL_FILE_TYPE.WORKFLOW) {
				acc.workflows.push(file);
				return acc;
			}

			if (file.type === SOURCE_CONTROL_FILE_TYPE.CREDENTIAL) {
				acc.credentials.push(file);
				return acc;
			}

			return acc;
		},
		{ tags: [], variables: [], credentials: [], workflows: [], currentWorkflow: undefined },
	);

const workflowId = computed(
	() =>
		([VIEWS.WORKFLOW].includes(route.name as VIEWS) && route.params.name?.toString()) || undefined,
);

const changes = computed(() => classifyFilesByType(props.data.status, workflowId.value));

const selectedChanges = ref<Set<string>>(new Set());
const toggleSelected = (id: string) => {
	if (selectedChanges.value.has(id)) {
		selectedChanges.value.delete(id);
	} else {
		selectedChanges.value.add(id);
	}
};

const maybeSelectCurrentWorkflow = (workflow?: SourceControlAggregatedFile) =>
	workflow && selectedChanges.value.add(workflow.id);
onMounted(() => maybeSelectCurrentWorkflow(changes.value.currentWorkflow));

const filters = ref<{ status?: SourceControlledFileStatus }>({
	status: undefined,
});
const statusFilterOptions: Array<{ label: string; value: SourceControlledFileStatus }> = [
	{
		label: 'New',
		value: SOURCE_CONTROL_FILE_STATUS.CREATED,
	},
	{
		label: 'Modified',
		value: SOURCE_CONTROL_FILE_STATUS.MODIFIED,
	},
	{
		label: 'Deleted',
		value: SOURCE_CONTROL_FILE_STATUS.DELETED,
	},
] as const;

const search = ref('');
const debouncedSearch = refDebounced(search, 250);

const filterCount = computed(() =>
	Object.values(filters.value).reduce((acc, item) => (item ? acc + 1 : acc), 0),
);

const filteredWorkflows = computed(() => {
	const searchQuery = debouncedSearch.value.toLocaleLowerCase();

	return changes.value.workflows.filter((workflow) => {
		if (!workflow.name.toLocaleLowerCase().includes(searchQuery)) {
			return false;
		}

		if (filters.value.status && filters.value.status !== workflow.status) {
			return false;
		}

		return true;
	});
});

const statusPriority: Partial<Record<SourceControlledFileStatus, number>> = {
	[SOURCE_CONTROL_FILE_STATUS.MODIFIED]: 1,
	[SOURCE_CONTROL_FILE_STATUS.RENAMED]: 2,
	[SOURCE_CONTROL_FILE_STATUS.CREATED]: 3,
	[SOURCE_CONTROL_FILE_STATUS.DELETED]: 4,
} as const;
const getPriorityByStatus = (status: SourceControlledFileStatus): number =>
	statusPriority[status] ?? 0;

const sortedWorkflows = computed(() => {
	const sorted = orderBy(
		filteredWorkflows.value,
		[
			// keep the current workflow at the top of the list
			({ id }) => id === changes.value.currentWorkflow?.id,
			({ status }) => getPriorityByStatus(status),
			'updatedAt',
		],
		['desc', 'asc', 'desc'],
	);

	return sorted;
});

const commitMessage = ref('');
const isSubmitDisabled = computed(() => {
	if (!commitMessage.value.trim()) {
		return true;
	}

	const toBePushed =
		changes.value.credentials.length +
		changes.value.tags.length +
		changes.value.variables.length +
		selectedChanges.value.size;
	if (toBePushed <= 0) {
		return true;
	}

	return false;
});

const selectAll = computed(
	() =>
		selectedChanges.value.size > 0 && selectedChanges.value.size === sortedWorkflows.value.length,
);

const selectAllIndeterminate = computed(
	() => selectedChanges.value.size > 0 && selectedChanges.value.size < sortedWorkflows.value.length,
);

function onToggleSelectAll() {
	if (selectAll.value) {
		selectedChanges.value.clear();
	} else {
		selectedChanges.value = new Set(changes.value.workflows.map((file) => file.id));
	}
}

function close() {
	uiStore.closeModal(SOURCE_CONTROL_PUSH_MODAL_KEY);
}

function renderUpdatedAt(file: SourceControlAggregatedFile) {
	const currentYear = new Date().getFullYear().toString();

	return i18n.baseText('settings.sourceControl.lastUpdated', {
		interpolate: {
			date: dateformat(
				file.updatedAt,
				`d mmm${file.updatedAt?.startsWith(currentYear) ? '' : ', yyyy'}`,
			),
			time: dateformat(file.updatedAt, 'HH:MM'),
		},
	});
}

async function onCommitKeyDownEnter() {
	if (!isSubmitDisabled.value) {
		await commitAndPush();
	}
}

async function commitAndPush() {
	const files = changes.value.tags
		.concat(changes.value.variables)
		.concat(changes.value.credentials)
		.concat(changes.value.workflows.filter((file) => selectedChanges.value.has(file.id)));

	loadingService.startLoading(i18n.baseText('settings.sourceControl.loading.push'));
	close();

	try {
		await sourceControlStore.pushWorkfolder({
			force: true,
			commitMessage: commitMessage.value,
			fileNames: files,
		});

		toast.showToast({
			title: i18n.baseText('settings.sourceControl.modals.push.success.title'),
			message: i18n.baseText('settings.sourceControl.modals.push.success.description'),
			type: 'success',
		});
	} catch (error) {
		toast.showError(error, i18n.baseText('error'));
	} finally {
		loadingService.stopLoading();
	}
}

const getStatusText = (status: SourceControlledFileStatus) =>
	i18n.baseText(`settings.sourceControl.status.${status}` as BaseTextKey);
const getStatusTheme = (status: SourceControlledFileStatus) => {
	const statusToBadgeThemeMap: Partial<
		Record<SourceControlledFileStatus, 'success' | 'danger' | 'warning'>
	> = {
		[SOURCE_CONTROL_FILE_STATUS.CREATED]: 'success',
		[SOURCE_CONTROL_FILE_STATUS.DELETED]: 'danger',
		[SOURCE_CONTROL_FILE_STATUS.MODIFIED]: 'warning',
	} as const;
	return statusToBadgeThemeMap[status];
};
</script>

<template>
	<Modal
		width="812px"
		:event-bus="data.eventBus"
		:name="SOURCE_CONTROL_PUSH_MODAL_KEY"
		max-height="80%"
		:custom-class="$style.sourceControlPush"
	>
		<template #header>
			<N8nHeading tag="h1" size="xlarge">
				{{ i18n.baseText('settings.sourceControl.modals.push.title') }}
			</N8nHeading>
			<div class="mb-l mt-l">
				<N8nText tag="div">
					{{ i18n.baseText('settings.sourceControl.modals.push.description') }}
					<N8nLink :to="i18n.baseText('settings.sourceControl.docs.using.pushPull.url')">
						{{ i18n.baseText('settings.sourceControl.modals.push.description.learnMore') }}
					</N8nLink>
				</N8nText>

				<N8nNotice v-if="!changes.workflows.length" class="mt-xs">
					<i18n-t keypath="settings.sourceControl.modals.push.noWorkflowChanges">
						<template #link>
							<N8nLink size="small" :to="i18n.baseText('settings.sourceControl.docs.using.url')">
								{{ i18n.baseText('settings.sourceControl.modals.push.noWorkflowChanges.moreInfo') }}
							</N8nLink>
						</template>
					</i18n-t>
				</N8nNotice>
			</div>

			<div :class="[$style.filers]">
				<N8nCheckbox
					:class="$style.selectAll"
					:indeterminate="selectAllIndeterminate"
					:model-value="selectAll"
					data-test-id="source-control-push-modal-toggle-all"
					@update:model-value="onToggleSelectAll"
				>
					<N8nText bold tag="strong">
						{{ i18n.baseText('settings.sourceControl.modals.push.workflowsToCommit') }}
					</N8nText>
					<N8nText tag="strong">
						({{ selectedChanges.size }}/{{ sortedWorkflows.length }})
					</N8nText>
				</N8nCheckbox>
				<N8nPopover trigger="click" width="304" style="align-self: normal">
					<template #reference>
						<N8nButton
							icon="filter"
							type="tertiary"
							style="height: 100%"
							:active="Boolean(filterCount)"
							data-test-id="source-control-filter-dropdown"
						>
							<N8nBadge v-show="filterCount" theme="primary" class="mr-4xs">
								{{ filterCount }}
							</N8nBadge>
							{{ i18n.baseText('forms.resourceFiltersDropdown.filters') }}
						</N8nButton>
					</template>
					<N8nInputLabel
						:label="i18n.baseText('workflows.filters.status')"
						:bold="false"
						size="small"
						color="text-base"
						class="mb-3xs"
					/>
					<N8nSelect v-model="filters.status" data-test-id="source-control-status-filter" clearable>
						<N8nOption
							v-for="option in statusFilterOptions"
							:key="option.label"
							data-test-id="source-control-status-filter-option"
							v-bind="option"
						>
						</N8nOption>
					</N8nSelect>
				</N8nPopover>
				<N8nInput
					v-model="search"
					data-test-id="source-control-push-search"
					:placeholder="i18n.baseText('workflows.search.placeholder')"
					clearable
				>
					<template #prefix>
						<N8nIcon icon="search" />
					</template>
				</N8nInput>
			</div>
		</template>
		<template #content>
			<RecycleScroller
				:class="[$style.scroller]"
				:items="sortedWorkflows"
				:item-size="69"
				key-field="id"
			>
				<template #default="{ item: file }">
					<N8nCheckbox
						:class="['scopedListItem', $style.listItem]"
						data-test-id="source-control-push-modal-file-checkbox"
						:model-value="selectedChanges.has(file.id)"
						@update:model-value="toggleSelected(file.id)"
					>
						<span>
							<N8nText v-if="file.status === SOURCE_CONTROL_FILE_STATUS.DELETED" color="text-light">
								<span v-if="file.type === SOURCE_CONTROL_FILE_TYPE.WORKFLOW">
									Deleted Workflow:
								</span>
								<span v-if="file.type === SOURCE_CONTROL_FILE_TYPE.CREDENTIAL">
									Deleted Credential:
								</span>
								<strong>{{ file.name || file.id }}</strong>
							</N8nText>
							<N8nText v-else bold> {{ file.name }} </N8nText>
							<N8nText v-if="file.updatedAt" tag="p" class="mt-0" color="text-light" size="small">
								{{ renderUpdatedAt(file) }}
							</N8nText>
						</span>
						<span :class="[$style.badges]">
							<N8nBadge
								v-if="changes.currentWorkflow && file.id === changes.currentWorkflow.id"
								class="mr-2xs"
							>
								Current workflow
							</N8nBadge>
							<N8nBadge :theme="getStatusTheme(file.status)">
								{{ getStatusText(file.status) }}
							</N8nBadge>
						</span>
					</N8nCheckbox>
				</template>
			</RecycleScroller>
		</template>

		<template #footer>
			<N8nText bold tag="p" class="mb-2xs">
				{{ i18n.baseText('settings.sourceControl.modals.push.commitMessage') }}
			</N8nText>
			<N8nInput
				v-model="commitMessage"
				data-test-id="source-control-push-modal-commit"
				:placeholder="i18n.baseText('settings.sourceControl.modals.push.commitMessage.placeholder')"
				@keydown.enter="onCommitKeyDownEnter"
			/>

			<div :class="$style.footer">
				<N8nButton type="tertiary" class="mr-2xs" @click="close">
					{{ i18n.baseText('settings.sourceControl.modals.push.buttons.cancel') }}
				</N8nButton>
				<N8nButton
					data-test-id="source-control-push-modal-submit"
					type="primary"
					:disabled="isSubmitDisabled"
					@click="commitAndPush"
				>
					{{ i18n.baseText('settings.sourceControl.modals.push.buttons.save') }}
				</N8nButton>
			</div>
		</template>
	</Modal>
</template>

<style module lang="scss">
.filers {
	display: flex;
	align-items: center;
	gap: 8px;
}

.selectAll {
	flex-shrink: 0;
	margin-bottom: 0;
}

.scroller {
	height: 380px;
	max-height: 100%;
}

.listItem {
	align-items: center;
	padding: var(--spacing-xs);
	transition: border 0.3s ease;
	border-radius: var(--border-radius-large);
	border: var(--border-base);

	&:hover {
		border-color: var(--color-foreground-dark);
	}

	:global(.el-checkbox__label) {
		display: flex;
		width: 100%;
		justify-content: space-between;
		align-items: center;
	}

	:global(.el-checkbox__inner) {
		transition: none;
	}
}

.badges {
	display: flex;
}

.footer {
	display: flex;
	flex-direction: row;
	justify-content: flex-end;
	margin-top: 20px;
}

.sourceControlPush {
	:global(.el-dialog__header) {
		padding-bottom: var(--spacing-xs);
	}
}
</style>
