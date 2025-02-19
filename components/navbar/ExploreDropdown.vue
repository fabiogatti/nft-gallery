<template>
  <b-dropdown class="navbar-explore" :triggers="['hover']">
    <template #trigger>
      <nuxt-link :to="`/${urlPrefix}/explore`">
        <div class="navbar-item" data-cy="explore">
          {{ $t('explore') }}
          <svg
            class="ml-1"
            width="14"
            height="7"
            viewBox="0 0 14 7"
            fill="none"
            xmlns="http://www.w3.org/2000/svg">
            <path
              d="M13.0002 1L7.00024 6L1.00025 0.999999"
              stroke="currentColor"
              stroke-width="1.26984" />
          </svg>
        </div>
      </nuxt-link>
    </template>
    <b-dropdown-item custom aria-role="menuitem" class="is-size-6">
      <nuxt-link
        :to="`/${urlPrefix}/explore?tab=GALLERY`"
        class="menu-item mr-2">
        {{ $t('gallery') }}
      </nuxt-link>
      <nuxt-link
        :to="`/${urlPrefix}/explore?tab=COLLECTION`"
        class="menu-item mr-2">
        {{ $t('collections') }}
      </nuxt-link>
      <span class="menu-item is-disabled">
        {{ $t('users') }}
        <span class="small-size-text">
          {{ $t('soon') }}
        </span>
      </span>
      <hr class="dropdown-divider my-4" aria-role="menuitem" />
      <span
        v-for="option in options.slice(0, 3)"
        :key="option.value"
        :value="option.value"
        :class="['menu-item', 'mr-2']"
        @click="setSelectedChain(option.value)">
        {{ option.text }}
      </span>
    </b-dropdown-item>
  </b-dropdown>
</template>

<script lang="ts">
import { Component, mixins } from 'nuxt-property-decorator'
import { getChainTestList } from '~/utils/constants'
import PrefixMixin from '@/utils/mixins/prefixMixin'

import AuthMixin from '~~/utils/mixins/authMixin'

@Component({})
export default class NavbarExplore extends mixins(PrefixMixin, AuthMixin) {
  get options() {
    const availableUrlPrefixes = this.$store.getters['availableUrlPrefixes']

    if (!this.$config.dev) {
      return availableUrlPrefixes.filter(
        (urlPrefix) => !getChainTestList().includes(urlPrefix.value as string)
      )
    }
    return availableUrlPrefixes
  }

  setSelectedChain(value) {
    this.$store.dispatch('setUrlPrefix', value)
    this.$router.push({ path: `/${value}/explore?tab=COLLECTION` })
  }
}
</script>
