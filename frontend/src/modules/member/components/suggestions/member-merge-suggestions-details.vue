<template>
  <section v-if="props.loading">
    <div
      class="rounded p-6 transition pb-40"
      :class="{ 'bg-gray-50': props.isPrimary }"
    >
      <app-loading height="48px" width="48px" radius="50%" class="mb-6" />

      <app-loading height="16px" width="172px" radius="3px" class="mb-6" />

      <app-loading height="12px" width="100%" radius="3px" class="mb-3" />
      <app-loading height="12px" width="80%" radius="3px" class="mb-8" />

      <app-loading height="12px" width="50%" radius="3px" class="mb-3" />
      <app-loading height="12px" width="50%" radius="3px" class="mb-3" />
      <app-loading height="12px" width="50%" radius="3px" class="mb-3" />
      <app-loading height="12px" width="50%" radius="3px" class="mb-3" />
      <app-loading height="12px" width="50%" radius="3px" class="mb-3" />
    </div>
  </section>
  <section v-else class="h-full">
    <div
      class="rounded p-6 transition h-full"
      :class="{ 'bg-gray-50': props.isPrimary }"
    >
      <!-- primary member -->
      <div class="h-13 flex justify-between items-start">
        <div
          v-if="props.isPreview"
          class="bg-brand-500 rounded-full py-0.5 px-2 text-white inline-block text-xs leading-5 font-medium"
        >
          Preview
        </div>
        <div
          v-else-if="props.isPrimary"
          class="bg-brand-500 rounded-full py-0.5 px-2 text-white inline-block text-xs leading-5 font-medium"
        >
          Primary contact
        </div>
        <button
          v-else
          :disabled="isEditLockedForSampleData"
          type="button"
          class="btn btn--bordered btn--sm leading-5 !px-4 !py-1"
          @click="emit('makePrimary')"
        >
          <span class="ri-arrow-left-right-fill text-base text-gray-600 mr-2" />
          <span>Make primary</span>
        </button>
        <slot name="action" />
      </div>
      <app-avatar :entity="member" class="mb-3" />
      <div class="pb-6">
        <h6
          class="text-base text-black font-semibold leading-6"
          v-html="$sanitize(member.displayName)"
        />
        <div
          v-if="member.attributes.bio?.default"
          ref="bio"
          class="text-gray-600 leading-5 !text-xs merge-member-bio mt-2"
          :class="{ 'line-clamp-2': !more }"
          v-html="$sanitize(member.attributes.bio.default)"
        />
        <div
          v-else-if="compareMember?.attributes.bio?.default"
          ref="bio"
          class="text-transparent invisible leading-5 !text-xs merge-member-bio line-clamp-2 mt-2"
          v-html="$sanitize(compareMember?.attributes.bio.default)"
        />

        <div
          v-if="displayShowMore"
          class="text-sm text-brand-500 mt-2 cursor-pointer"
          :class="{ invisible: !member.attributes.bio?.default }"
          @click.stop="more = !more"
        >
          Show {{ more ? 'less' : 'more' }}
        </div>
      </div>

      <div>
        <article class="pb-4">
          <p class="text-2xs font-medium text-gray-500 pb-1">
            Engagement level
          </p>
          <app-community-engagement-level :member="member" />
        </article>
        <article
          v-if="
            member.attributes.location?.default
              || compareMember?.attributes.location?.default
          "
          class="pb-4"
        >
          <p class="text-2xs font-medium text-gray-500 pb-1">
            Location
          </p>
          <p class="text-xs text-gray-900 whitespace-normal">
            {{ member.attributes.location?.default || '-' }}
          </p>
        </article>
        <article
          v-if="
            member.organizations.length || compareMember?.organizations.length
          "
          class="pb-4"
        >
          <p class="text-2xs font-medium text-gray-500 pb-1">
            Organization
          </p>
          <div>
            <app-member-organizations :member="member" :show-title="false" />
          </div>
        </article>
        <article
          v-if="
            member.attributes.jobTitle?.default
              || compareMember?.attributes.jobTitle?.default
          "
          class="pb-4"
        >
          <p class="text-2xs font-medium text-gray-500 pb-1">
            Title
          </p>
          <p class="text-xs text-gray-900 whitespace-normal">
            {{ member.attributes.jobTitle?.default || '-' }}
          </p>
        </article>
        <article
          v-if="member.joinedAt || compareMember?.joinedAt"
          class="pb-4"
        >
          <p class="text-2xs font-medium text-gray-500 pb-1">
            Contact since
          </p>
          <p class="text-xs text-gray-900 whitespace-normal">
            {{ moment(member.joinedAt).format('YYYY-MM-DD') }}
          </p>
        </article>
        <article
          v-if="member.tags.length > 0 || compareMember?.tags.length > 0"
          class="pb-4"
        >
          <p class="text-2xs font-medium text-gray-500 pb-1">
            Tags
          </p>
          <app-tags
            v-if="member.tags.length > 0"
            :member="member"
            :editable="false"
            tag-classes="!bg-white !text-xs !leading-5 !py-0.5 !px-1.5"
          />
          <span v-else>-</span>
        </article>
      </div>
      <div class="pt-4">
        <h6 class="text-sm font-semibold pb-3">
          Identities
        </h6>
        <app-identities-vertical-list-members
          :member="member"
          :order="memberOrder.suggestions"
          :include-emails="true"
        />
      </div>
    </div>
  </section>
</template>

<script setup>
import {
  computed, defineProps, onMounted, ref, defineExpose,
} from 'vue';
import moment from 'moment';
import AppMemberOrganizations from '@/modules/member/components/member-organizations.vue';
import AppAvatar from '@/shared/avatar/avatar.vue';
import AppCommunityEngagementLevel from '@/modules/member/components/member-engagement-level.vue';
import AppTags from '@/modules/tag/components/tag-list.vue';
import AppLoading from '@/shared/loading/loading-placeholder.vue';
import { MemberPermissions } from '@/modules/member/member-permissions';
import { mapGetters } from '@/shared/vuex/vuex.helpers';
import memberOrder from '@/shared/modules/identities/config/identitiesOrder/member';
import AppIdentitiesVerticalListMembers from '@/shared/modules/identities/components/identities-vertical-list-members.vue';

const props = defineProps({
  member: {
    type: Object,
    default: () => null,
  },
  compareMember: {
    type: Object,
    required: false,
    default: () => null,
  },
  isPrimary: {
    type: Boolean,
    required: false,
    default: false,
  },
  isPreview: {
    type: Boolean,
    required: false,
    default: false,
  },
  loading: {
    type: Boolean,
    required: false,
    default: false,
  },
  extendBio: {
    type: Number,
    required: false,
    default: 0,
  },
});

const emit = defineEmits(['makePrimary', 'bioHeight']);

const { currentTenant, currentUser } = mapGetters('auth');

const isEditLockedForSampleData = computed(
  () => new MemberPermissions(currentTenant.value, currentUser.value)
    .editLockedForSampleData,
);

const bio = ref(null);
const displayShowMore = ref(null);
const more = ref(null);

onMounted(() => {
  setTimeout(() => {
    if (!bio.value) {
      return;
    }
    const height = bio.value.clientHeight;
    const { scrollHeight } = bio.value;
    displayShowMore.value = scrollHeight > height || false;
    emit('bioHeight', scrollHeight);
  }, 0);
});

defineExpose({
  more,
});
</script>

<script>
export default {
  name: 'AppMemberMergeSuggestionsDetails',
};
</script>

<style lang="scss">
.merge-member-bio * {
  @apply text-xs;
}
</style>
