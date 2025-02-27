<template>
  <Alert
    v-if="showSpecNotFound"
    v-model="showSpecNotFound"
    status="error"
    :title="t('specPage.noSpecError.title')"
    class="mb-16px"
    :icon="WarningIcon"
    dismissible
  >
    <p class="mb-24px">
      {{ t('specPage.noSpecError.intro') }} <InlineCodeFragment variant="error">
        {{ route.params.unrunnable }}
      </InlineCodeFragment>
    </p>
    <p>{{ t('specPage.noSpecError.explainer') }}</p>
  </Alert>
  <Alert
    v-else-if="showOffline"
    v-model="showOffline"
    data-cy="offline-alert"
    status="warning"
    :title="t('specPage.offlineWarning.title')"
    class="mb-16px"
    :icon="WarningIcon"
    dismissible
  >
    <p class="mb-24px">
      {{ t('specPage.offlineWarning.explainer') }}
    </p>
  </Alert>
  <Alert
    v-else-if="showFetchError"
    v-model="showFetchError"
    status="warning"
    :title="t('specPage.fetchFailedWarning.title')"
    class="mb-16px"
    :icon="WarningIcon"
    dismissible
  >
    <p>
      {{ t('specPage.fetchFailedWarning.explainer1') }}
    </p>
    <p>
      <i18n-t
        scope="global"
        keypath="specPage.fetchFailedWarning.explainer2"
      >
        <ExternalLink
          href="https://www.cypressstatus.com"
          class="font-medium text-indigo-500 contents group-hocus:text-indigo-600"
        >
          Status Page
        </ExternalLink>
      </i18n-t>
    </p>
    <Button
      :prefix-icon="RefreshIcon"
      class="mt-24px"
      data-cy="refresh-button"
      @click="emit('refetchFailedCloudData')"
    >
      {{ t('specPage.fetchFailedWarning.refreshButton') }}
    </Button>
  </Alert>
  <Alert
    v-else-if="showProjectNotFound"
    v-model="showProjectNotFound"
    data-cy="project-not-found-alert"
    status="warning"
    :title="t('runs.errors.notFound.title')"
    class="mb-16px"
    :icon="WarningIcon"
    dismissible
  >
    <p class="mb-24px">
      <i18n-t
        scope="global"
        keypath="runs.errors.notFound.description"
      >
        <CodeTag
          bg
          class="bg-warning-100"
        >
          projectId: "{{ props.gql.currentProject?.projectId }}"
        </CodeTag>
      </i18n-t>
    </p>
    <Button
      :prefix-icon="ConnectIcon"
      class="mt-24px"
      data-cy="reconnect-button"
      @click="emit('reconnectProject')"
    >
      {{ t('runs.errors.notFound.button') }}
    </Button>
  </Alert>
  <Alert
    v-else-if="showProjectRequestAccess"
    v-model="showProjectRequestAccess"
    data-cy="project-request-access-alert"
    status="warning"
    :title="t('specPage.unauthorizedBannerTitle')"
    class="mb-16px"
    :icon="WarningIcon"
    dismissible
  >
    <p class="mb-24px">
      {{ props.hasRequestedAccess ? t('runs.errors.unauthorizedRequested.description') : t('runs.errors.unauthorized.description') }}
    </p>
    <RequestAccessButton :gql="props.gql" />
  </Alert>
  <RecordBanner
    v-else-if="showRecordBanner"
    v-model="showRecordBanner"
  />
  <ConnectProjectBanner
    v-else-if="showConnectBanner"
    v-model="showConnectBanner"
  />
  <CreateOrganizationBanner
    v-else-if="showCreateOrganizationBanner"
    v-model="showCreateOrganizationBanner"
  />
  <LoginBanner
    v-else-if="showLoginBanner"
    v-model="showLoginBanner"
  />
</template>

<script setup lang="ts">
import Button from '@cy/components/Button.vue'
import ExternalLink from '@cy/gql-components/ExternalLink.vue'
import { useI18n } from '@cy/i18n'
import Alert from '@packages/frontend-shared/src/components/Alert.vue'
import CodeTag from '@packages/frontend-shared/src/components/CodeTag.vue'
import InlineCodeFragment from '@packages/frontend-shared/src/components/InlineCodeFragment.vue'
import ConnectIcon from '~icons/cy/chain-link_x16.svg'
import WarningIcon from '~icons/cy/warning_x16.svg'
import RefreshIcon from '~icons/cy/action-restart_x16'
import { useRoute } from 'vue-router'
import { computed, ref, watch } from 'vue'
import RequestAccessButton from './RequestAccessButton.vue'
import { gql, useSubscription } from '@urql/vue'
import { SpecsListBannersFragment, SpecsListBanners_CheckCloudOrgMembershipDocument } from '../generated/graphql'
import interval from 'human-interval'
import { AllowedState, BannerIds } from '@packages/types'
import { LoginBanner, CreateOrganizationBanner, ConnectProjectBanner, RecordBanner } from './banners'

const route = useRoute()
const { t } = useI18n()

gql`
fragment SpecsListBanners on Query {
  ...RequestAccessButton
  cloudViewer {
    id
    firstOrganization: organizations(first: 1) {
      nodes {
        id
      }
    }
  }
  cachedUser {
    id
  }
  currentProject {
    id
    projectId
    savedState
    cloudProject {
      __typename
      ... on CloudProject {
        id
        runs(first: 1) {
          nodes {
            id
          }
        }
      }
    }
  }
}
`

gql`
subscription SpecsListBanners_CheckCloudOrgMembership {
  cloudViewerChange {
    ...SpecsListBanners
  }
}
`

const props = withDefaults(defineProps<{
  gql: SpecsListBannersFragment
  hasRequestedAccess?: boolean
  isSpecNotFound?: boolean
  isOffline?: boolean
  isFetchError?: boolean
  isProjectNotFound?: boolean
  isProjectUnauthorized?: boolean
}>(), {
  isSpecNotFound: undefined,
  isOffline: undefined,
  isFetchError: undefined,
  isProjectNotFound: undefined,
  isProjectUnauthorized: undefined,
})

const emit = defineEmits<{
  (e: 'refetchFailedCloudData'): void
  (e: 'reconnectProject'): void
}>()

useSubscription({ query: SpecsListBanners_CheckCloudOrgMembershipDocument })

const showSpecNotFound = ref(props.isSpecNotFound)
const showOffline = ref(props.isOffline)
const showFetchError = ref(props.isFetchError)
const showProjectNotFound = ref(props.isProjectNotFound)
const showProjectRequestAccess = ref(props.isProjectUnauthorized)
const showRecordBanner = ref(false)
const showConnectBanner = ref(false)
const showCreateOrganizationBanner = ref(false)
const showLoginBanner = ref(false)

watch(
  () => ([props.isSpecNotFound, props.isOffline, props.isFetchError, props.isProjectNotFound, props.isProjectUnauthorized]),
  () => {
    showSpecNotFound.value = props.isSpecNotFound
    showOffline.value = props.isOffline
    showFetchError.value = props.isFetchError
    showProjectNotFound.value = props.isProjectNotFound
    showProjectRequestAccess.value = props.isProjectUnauthorized
  },
)

const cloudData = computed(() => ([props.gql.cloudViewer, props.gql.cachedUser, props.gql.currentProject] as const))

watch(
  cloudData,
  ([cloudViewer, cachedUser, currentProject]) => {
    // Cached user covers state where we're authenticated but data isn't loaded yet
    const isLoggedIn = !!cachedUser?.id || !!cloudViewer?.id
    // Need to be able to tell whether the lack of `firstOrganization` means they don't have an org or whether it just hasn't loaded yet
    // Not having this check can cause a brief flicker of the 'Create Org' banner while org data is loading
    const isOrganizationLoaded = !!cloudViewer?.firstOrganization
    const isMemberOfOrganization = (cloudViewer?.firstOrganization?.nodes?.length ?? 0) > 0
    const isProjectConnected = !!currentProject?.projectId && currentProject.cloudProject?.__typename === 'CloudProject'
    const hasNoRecordedRuns = currentProject?.cloudProject?.__typename === 'CloudProject' && (currentProject.cloudProject?.runs?.nodes?.length ?? 0) === 0
    const hasFourDaysOfCypressUse = (Date.now() - currentProject?.savedState?.firstOpened) > interval('4 days')

    showRecordBanner.value = !hasBannerBeenDismissed(BannerIds.ACI_082022_RECORD) && isLoggedIn && isProjectConnected && isMemberOfOrganization && isProjectConnected && hasNoRecordedRuns && hasFourDaysOfCypressUse
    showConnectBanner.value = !hasBannerBeenDismissed(BannerIds.ACI_082022_CONNECT_PROJECT) && isLoggedIn && isMemberOfOrganization && !isProjectConnected && hasFourDaysOfCypressUse
    showCreateOrganizationBanner.value = !hasBannerBeenDismissed(BannerIds.ACI_082022_CREATE_ORG) && isLoggedIn && isOrganizationLoaded && !isMemberOfOrganization && hasFourDaysOfCypressUse
    showLoginBanner.value = !hasBannerBeenDismissed(BannerIds.ACI_082022_LOGIN) && !isLoggedIn && hasFourDaysOfCypressUse
  },
  { immediate: true },
)

function hasBannerBeenDismissed (bannerId: string) {
  const bannersState = (props.gql.currentProject?.savedState as AllowedState)?.banners

  return !!bannersState?._disabled || !!bannersState?.[bannerId]?.dismissed
}

</script>
