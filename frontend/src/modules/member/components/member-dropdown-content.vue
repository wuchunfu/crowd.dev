<template>
  <router-link
    v-if="!props.hideEdit"
    :to="{
      name: 'memberEdit',
      params: {
        id: member.id,
      },
    }"
    :class="{
      'pointer-events-none cursor-not-allowed': isEditLockedForSampleData,
    }"
  >
    <button
      class="h-10 el-dropdown-menu__item w-full mb-1"
      :disabled="isEditLockedForSampleData"
      type="button"
    >
      <i class="ri-pencil-line text-base mr-2" />
      <span class="text-xs">Edit contact</span>
    </button>
  </router-link>
  <button
    v-if="isFindGitHubFeatureEnabled"
    class="h-10 el-dropdown-menu__item w-full mb-1"
    type="button"
    :disabled="isFindingGitHubDisabled"
    @click="handleCommand({
      action: Actions.FIND_GITHUB,
      member,
    })"
  >
    <span
      class="max-w-[16px]"
      color="#9CA3AF"
    ><i class="ri-github-fill" /></span>
    <span class="ml-2 text-xs"> Find GitHub </span>
  </button>
  <button
    v-if="!props.hideMerge"
    class="h-10 el-dropdown-menu__item w-full"
    :disabled="isEditLockedForSampleData"
    type="button"
    @click="
      handleCommand({
        action: Actions.MERGE_CONTACT,
        member,
      })
    "
  >
    <i class="ri-group-line text-base mr-2" /><span class="text-xs">Merge contact</span>
  </button>

  <!-- Hubspot -->
  <el-tooltip
    placement="top"
    content="Upgrade your plan to create a 2-way sync with HubSpot"
    :disabled="isHubspotFeatureEnabled"
    popper-class="max-w-[260px]"
  >
    <span>
      <button
        v-if="!isSyncingWithHubspot"
        class="h-10 el-dropdown-menu__item w-full"
        :disabled="isHubspotActionDisabled"
        type="button"
        @click="
          handleCommand({
            action: Actions.SYNC_HUBSPOT,
            member,
          })
        "
      >
        <app-svg name="hubspot" class="h-4 w-4 text-current" />
        <span class="text-xs pl-2">Sync with HubSpot</span>
      </button>
      <button
        v-else
        class="h-10 el-dropdown-menu__item w-full"
        :disabled="isHubspotActionDisabled"
        type="button"
        @click="
          handleCommand({
            action: Actions.STOP_SYNC_HUBSPOT,
            member,
          })
        "
      >
        <app-svg name="hubspot" class="h-4 w-4 text-current" />
        <span class="text-xs pl-2">Stop sync with HubSpot</span>
      </button>
    </span>
  </el-tooltip>

  <el-tooltip
    placement="top"
    content="Mark as team contact if they belong to your own organization"
    popper-class="max-w-[260px]"
  >
    <span>
      <button
        v-if="!member.attributes.isTeamMember?.default"
        class="h-10 el-dropdown-menu__item w-full"
        :disabled="isEditLockedForSampleData"
        type="button"
        @click="
          handleCommand({
            action: Actions.MARK_CONTACT_AS_TEAM_CONTACT,
            member,
            value: true,
          })
        "
      >
        <i class="ri-bookmark-line text-base mr-2" /><span class="text-xs">Mark as team contact</span>
      </button>
    </span>
  </el-tooltip>

  <button
    v-if="member.attributes.isTeamMember?.default"
    class="h-10 el-dropdown-menu__item w-full"
    :disabled="isEditLockedForSampleData"
    type="button"
    @click="
      handleCommand({
        action: Actions.MARK_CONTACT_AS_TEAM_CONTACT,
        member,
        value: false,
      })
    "
  >
    <i class="ri-bookmark-2-line text-base mr-2" /><span class="text-xs">Unmark as team contact</span>
  </button>
  <button
    v-if="!member.attributes.isBot?.default"
    class="h-10 el-dropdown-menu__item w-full"
    :disabled="isEditLockedForSampleData"
    type="button"
    @click="
      handleCommand({
        action: Actions.MARK_CONTACT_AS_BOT,
        member,
      })
    "
  >
    <i class="ri-robot-line text-base mr-2" /><span class="text-xs">Mark as bot</span>
  </button>
  <button
    v-if="member.attributes.isBot?.default"
    class="h-10 el-dropdown-menu__item w-full"
    :disabled="isEditLockedForSampleData"
    type="button"
    @click="
      handleCommand({
        action: Actions.UNMARK_CONTACT_AS_BOT,
        member,
      })
    "
  >
    <i class="ri-robot-line text-base mr-2" /><span class="text-xs">Unmark as bot</span>
  </button>
  <el-divider class="border-gray-200" />
  <button
    class="h-10 el-dropdown-menu__item w-full"
    :disabled="isDeleteLockedForSampleData"
    type="button"
    @click="
      handleCommand({
        action: Actions.DELETE_CONTACT,
        member,
      })
    "
  >
    <i
      class="ri-delete-bin-line text-base mr-2"
      :class="{
        'text-red-500': !isDeleteLockedForSampleData,
      }"
    /><span
      class="text-xs"
      :class="{
        'text-red-500': !isDeleteLockedForSampleData,
      }"
    >Delete contact</span>
  </button>
</template>

<script setup lang="ts">
import { mapActions, mapGetters } from '@/shared/vuex/vuex.helpers';
import { MemberService } from '@/modules/member/member-service';
import Message from '@/shared/message/message';
import { MemberPermissions } from '@/modules/member/member-permissions';
import ConfirmDialog from '@/shared/dialog/confirm-dialog';
import AppSvg from '@/shared/svg/svg.vue';
import { useMemberStore } from '@/modules/member/store/pinia';
import { CrowdIntegrations } from '@/integrations/integrations-config';
import { HubspotEntity } from '@/integrations/hubspot/types/HubspotEntity';
import { HubspotApiService } from '@/integrations/hubspot/hubspot.api.service';
import { FeatureFlag, FEATURE_FLAGS } from '@/utils/featureFlag';
import { useStore } from 'vuex';
import { useRoute } from 'vue-router';
import { computed } from 'vue';
import { Member } from '../types/Member';

enum Actions {
  DELETE_CONTACT = 'deleteContact',
  SYNC_HUBSPOT = 'syncHubspot',
  STOP_SYNC_HUBSPOT = 'stopSyncHubspot',
  MARK_CONTACT_AS_TEAM_CONTACT = 'markContactAsTeamContact',
  MARK_CONTACT_AS_BOT = 'markContactAsBot',
  UNMARK_CONTACT_AS_BOT = 'unmarkContactAsBot',
  MERGE_CONTACT = 'mergeContact',
  FIND_GITHUB = 'findGithub'
}

const emit = defineEmits<{(e: 'merge'): void, (e: 'closeDropdown'): void, (e: 'findGithub'): void }>();
const props = defineProps<{
  member: Member;
  hideMerge: boolean;
  hideEdit: boolean;
}>();

const store = useStore();
const route = useRoute();

const { currentUser, currentTenant } = mapGetters('auth');
const { doFind } = mapActions('member');

const memberStore = useMemberStore();

const isEditLockedForSampleData = computed(
  () => new MemberPermissions(currentTenant.value, currentUser.value)
    .editLockedForSampleData,
);

const isDeleteLockedForSampleData = computed(
  () => new MemberPermissions(currentTenant.value, currentUser.value)
    .destroyLockedForSampleData,
);

const isSyncingWithHubspot = computed(
  () => props.member.attributes?.syncRemote?.hubspot || false,
);

const isHubspotConnected = computed(() => {
  const hubspot = CrowdIntegrations.getMappedConfig('hubspot', store);
  const enabledFor = hubspot.settings?.enabledFor || [];

  return (
    hubspot.status === 'done' && enabledFor.includes(HubspotEntity.MEMBERS)
  );
});

const isHubspotDisabledForMember = computed(
  () => props.member.emails.length === 0,
);

const isHubspotFeatureEnabled = computed(() => FeatureFlag.isFlagEnabled(FEATURE_FLAGS.hubspot));

const isHubspotActionDisabled = computed(
  () => !isHubspotConnected.value
    || isHubspotDisabledForMember.value
    || !isHubspotFeatureEnabled.value,
);

const isFindingGitHubDisabled = computed(() => (
  !!props.member.username?.github
));

const isFindGitHubFeatureEnabled = computed(() => FeatureFlag.isFlagEnabled(
  FEATURE_FLAGS.findGitHub,
));

const doManualAction = async ({
  loadingMessage,
  actionFn,
  successMessage,
  errorMessage,
}: {
  loadingMessage?: string;
  successMessage?: string;
  errorMessage?: string;
  actionFn: Promise<any>;
}) => {
  emit('closeDropdown');

  if (loadingMessage) {
    Message.info(null, {
      title: loadingMessage,
    });
  }

  return actionFn
    .then(() => {
      if (successMessage) {
        Message.closeAll();
        Message.success(successMessage);
      }
      Promise.resolve();
    })
    .catch(() => {
      if (errorMessage) {
        Message.closeAll();
        Message.error(errorMessage);
      }
      Promise.reject();
    });
};

const handleCommand = async (command: {
  action: Actions;
  member: Member;
  value?: boolean;
}) => {
  // Delete contact
  if (command.action === Actions.DELETE_CONTACT) {
    ConfirmDialog({
      type: 'danger',
      title: 'Delete contact',
      message: "Are you sure you want to proceed? You can't undo this action",
      confirmButtonText: 'Confirm',
      cancelButtonText: 'Cancel',
      icon: 'ri-delete-bin-line',
    }).then(() => {
      doManualAction({
        loadingMessage: 'Contact is being deleted',
        successMessage: 'Contact successfully deleted',
        errorMessage: 'Something went wrong',
        actionFn: MemberService.destroyAll([command.member.id]),
      }).then(() => {
        memberStore.fetchMembers({ reload: true });
      });
    });

    return;
  }

  // Sync with hubspot
  if (
    command.action === Actions.SYNC_HUBSPOT
    || command.action === Actions.STOP_SYNC_HUBSPOT
  ) {
    const isSyncing = command.action === Actions.SYNC_HUBSPOT;

    doManualAction({
      loadingMessage: isSyncing
        ? 'Contact is being synced with Hubspot'
        : 'Contact syncing with Hubspot is being stopped',
      successMessage: isSyncing
        ? 'Contact is now syncing with HubSpot'
        : 'Contact syncing stopped',
      errorMessage: 'Something went wrong',
      actionFn: isSyncing
        ? HubspotApiService.syncMember(command.member.id)
        : HubspotApiService.stopSyncMember(command.member.id),
    }).then(() => {
      if (route.name === 'member') {
        memberStore.fetchMembers({ reload: true });
      } else {
        doFind(command.member.id);
      }
    });

    return;
  }

  // Mark as team contact
  if (command.action === Actions.MARK_CONTACT_AS_TEAM_CONTACT) {
    doManualAction({
      loadingMessage: 'Contact is being updated',
      successMessage: 'Contact updated successfully',
      errorMessage: 'Something went wrong',
      actionFn: MemberService.update(command.member.id, {
        attributes: {
          ...command.member.attributes,
          isTeamMember: {
            default: command.value,
          },
        },
      }),
    }).then(() => {
      if (route.name === 'member') {
        memberStore.fetchMembers({ reload: true });
      } else {
        doFind(command.member.id);
      }
    });

    return;
  }

  // Mark as bot
  if (
    command.action === Actions.MARK_CONTACT_AS_BOT
    || command.action === Actions.UNMARK_CONTACT_AS_BOT
  ) {
    doManualAction({
      loadingMessage: 'Contact is being updated',
      successMessage: 'Contact updated successfully',
      errorMessage: 'Something went wrong',
      actionFn: MemberService.update(command.member.id, {
        attributes: {
          ...command.member.attributes,
          isBot: {
            default: command.action === Actions.MARK_CONTACT_AS_BOT,
          },
        },
      }),
    }).then(() => {
      if (route.name === 'member') {
        memberStore.fetchMembers({ reload: true });
      } else {
        doFind(command.member.id);
      }
    });

    return;
  }

  // Merge contact
  if (command.action === Actions.MERGE_CONTACT) {
    emit('closeDropdown');
    emit('merge');

    return;
  }

  if (command.action === Actions.FIND_GITHUB) {
    emit('closeDropdown');
    emit('findGithub');

    return;
  }

  emit('closeDropdown');
};
</script>

<style lang="scss" scoped>
.el-dropdown__popper .el-dropdown__list {
  @apply p-2;
}

// Override divider margin
.el-divider--horizontal {
  @apply my-2;
}

.el-dropdown-menu__item:disabled {
  @apply cursor-not-allowed text-gray-400;
}

.el-dropdown-menu__item:disabled:hover {
  @apply bg-white;
}
</style>
