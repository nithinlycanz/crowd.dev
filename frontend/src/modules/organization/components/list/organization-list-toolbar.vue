<template>
  <div
    v-if="selectedOrganizations.length > 0"
    class="app-list-table-bulk-actions"
  >
    <span class="block text-sm font-semibold mr-4">
      {{
        pluralize('organization', selectedOrganizations.length, true)
      }}
      selected
    </span>

    <el-dropdown trigger="click" @command="handleCommand">
      <button type="button" class="btn btn--bordered btn--sm">
        <span class="mr-2">Actions</span>
        <i class="ri-xl ri-arrow-down-s-line" />
      </button>
      <template #dropdown>
        <el-dropdown-item :command="{ action: 'export' }">
          <i class="ri-lg ri-file-download-line mr-1" />
          Export to CSV
        </el-dropdown-item>

        <el-dropdown-item
          v-if="selectedOrganizations.length === 2"
          :command="{
            action: 'mergeOrganizations',
          }"
          :disabled="
            isPermissionReadOnly
              || isEditLockedForSampleData
          "
        >
          <i
            class="ri-lg mr-1 ri-shuffle-line"
          />
          Merge organizations
        </el-dropdown-item>

        <el-dropdown-item
          v-if="markAsTeamOrganizationOptions"
          :command="{
            action: 'markAsTeamOrganization',
            value: markAsTeamOrganizationOptions.value,
          }"
          :disabled="
            isPermissionReadOnly
              || isEditLockedForSampleData
          "
        >
          <i
            class="ri-lg mr-1"
            :class="markAsTeamOrganizationOptions.icon"
          />
          {{ markAsTeamOrganizationOptions.copy }}
        </el-dropdown-item>

        <hr class="border-gray-200 my-1 mx-2" />

        <el-dropdown-item
          :command="{ action: 'destroyAll' }"
          :disabled="
            isPermissionReadOnly
              || isDeleteLockedForSampleData
          "
        >
          <div
            class="flex items-center"
            :class="{
              'text-red-500': !isDeleteLockedForSampleData,
            }"
          >
            <i class="ri-lg ri-delete-bin-line mr-2" />
            <span>Delete organizations</span>
          </div>
        </el-dropdown-item>
      </template>
    </el-dropdown>
  </div>
</template>

<script setup>
import pluralize from 'pluralize';
import { computed } from 'vue';
import {
  mapGetters,
} from '@/shared/vuex/vuex.helpers';
import ConfirmDialog from '@/shared/dialog/confirm-dialog';
import Message from '@/shared/message/message';
import { useOrganizationStore } from '@/modules/organization/store/pinia';
import { storeToRefs } from 'pinia';
import Errors from '@/shared/error/errors';
import { Excel } from '@/shared/excel/excel';
import { DEFAULT_ORGANIZATION_FILTERS } from '@/modules/organization/store/constants';
import { OrganizationPermissions } from '../../organization-permissions';
import { OrganizationService } from '../../organization-service';

const props = defineProps({
  pagination: {
    type: Object,
    default: () => ({
      page: 1,
      perPage: 20,
    }),
  },
});

const { currentUser, currentTenant } = mapGetters('auth');

const organizationStore = useOrganizationStore();
const {
  selectedOrganizations,
  filters,
  mergedOrganizations,
} = storeToRefs(organizationStore);
const { fetchOrganizations } = organizationStore;

const isPermissionReadOnly = computed(
  () => new OrganizationPermissions(
    currentTenant.value,
    currentUser.value,
  ).edit === false,
);

const isEditLockedForSampleData = computed(
  () => new OrganizationPermissions(
    currentTenant.value,
    currentUser.value,
  ).editLockedForSampleData,
);
const isDeleteLockedForSampleData = computed(
  () => new OrganizationPermissions(
    currentTenant.value,
    currentUser.value,
  ).destroyLockedForSampleData,
);

const markAsTeamOrganizationOptions = computed(() => {
  const isTeamView = filters.value.settings.teamOrganization === 'filter';
  const organizationsCopy = pluralize(
    'organization',
    selectedOrganizations.value.length,
    false,
  );

  if (isTeamView) {
    return {
      icon: 'ri-bookmark-2-line',
      copy: `Unmark as team ${organizationsCopy}`,
      value: false,
    };
  }

  return {
    icon: 'ri-bookmark-line',
    copy: `Mark as team ${organizationsCopy}`,
    value: true,
  };
});

const handleDoDestroyAllWithConfirm = () => ConfirmDialog({
  type: 'danger',
  title: 'Delete organizations',
  message:
      "Are you sure you want to proceed? You can't undo this action",
  confirmButtonText: 'Confirm',
  cancelButtonText: 'Cancel',
  icon: 'ri-delete-bin-line',
})
  .then(() => {
    const ids = selectedOrganizations.value.map((m) => m.id);
    return OrganizationService.destroyAll(ids);
  })
  .then(() => fetchOrganizations({ reload: true }));

const handleMergeOrganizations = async () => {
  const [firstOrganization, secondOrganization] = selectedOrganizations.value;

  OrganizationService.mergeOrganizations(firstOrganization.id, secondOrganization.id)
    .then(() => {
      Message.closeAll();

      organizationStore
        .addMergedOrganizations(firstOrganization.id, secondOrganization.id);

      const processesRunning = Object.keys(mergedOrganizations.value).length;

      Message.info(null, {
        title: 'Organizations merging in progress',
        message: processesRunning > 1 ? `${processesRunning} processes running` : null,
      });

      fetchOrganizations({ reload: true });
    })
    .catch((error) => {
      Message.closeAll();

      if (error.response.status === 404) {
        Message.success('Organizations already merged or deleted', {
          message: `Sorry, the organizations you are trying to merge might have already been merged or deleted.
          Please refresh to see the updated information.`,
        });
      } else {
        Message.error('There was an error merging organizations');
      }
    });
};

const handleDoExport = async () => {
  try {
    const filter = {
      and: [
        ...DEFAULT_ORGANIZATION_FILTERS,
        {
          id: {
            in: selectedOrganizations.value.map((o) => o.id),
          },
        },
      ],
    };

    const response = await OrganizationService.query({
      filter,
      orderBy: `${filters.value.order.prop}_${filters.value.order.order === 'descending' ? 'DESC' : 'ASC'}`,
      offset: (props.pagination.page - 1) * props.pagination.perPage || 0,
      limit: props.pagination.perPage || 20,
    });

    Excel.exportAsExcelFile(
      response.rows.map((o) => ({
        Id: o.id,
        Name: o.name,
        Description: o.description,
        Headline: o.headline,
        Website: o.website,
        '# of contacts': o.memberCount,
        '# of activities': o.activityCount,
        Location: o.location,
        Created: o.createdAt,
        Updated: o.updatedAt,
      })),
      ['Id', 'Name', 'Description',
        'Headline', 'Headline', '# of contacts',
        '# of activities', 'Location', 'Created', 'Updated',
      ],
      `organizations_${new Date().getTime()}`,
    );

    Message.success('Organizations exported successfully');
  } catch (error) {
    Errors.handle(error);
    Message.error(
      'There was an error exporting organizations',
    );
  }
};

const handleCommand = async (command) => {
  if (command.action === 'export') {
    await handleDoExport();
  } else if (command.action === 'destroyAll') {
    await handleDoDestroyAllWithConfirm();
  } else if (command.action === 'mergeOrganizations') {
    await handleMergeOrganizations();
  } else if (command.action === 'markAsTeamOrganization') {
    Message.info(
      null,
      {
        title: 'Organizations are being updated',
      },
    );

    Promise.all(
      selectedOrganizations.value.map((row) => OrganizationService.update(row.id, {
        isTeamOrganization: command.value,
      })),
    ).then(() => {
      Message.closeAll();
      Message.success(
        `${pluralize(
          'Organization',
          selectedOrganizations.value.length,
          false,
        )} updated successfully`,
      );

      fetchOrganizations({
        reload: true,
      });
    })
      .catch(() => {
        Message.closeAll();
        Message.error('Error updating organizations');
      });
  }
};
</script>

<script>
export default {
  name: 'AppOrganizationListToolbar',
};
</script>
