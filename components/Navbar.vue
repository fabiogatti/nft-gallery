<template>
  <b-navbar
    fixed-top
    spaced
    wrapper-class="container"
    close-on-click
    mobile-burger
    :active.sync="isBurgerMenuOpened"
    :class="{ 'navbar-shrink': !showTopNavbar }">
    <template #brand>
      <b-navbar-item tag="nuxt-link" :to="{ path: '/' }" class="logo">
        <img
          :src="logoSrc"
          alt="First NFT market explorer on Kusama and Polkadot"
          width="143"
          height="42" />
      </b-navbar-item>
      <div
        class="is-hidden-desktop is-flex is-flex-grow-1 is-align-items-center is-justify-content-flex-end"
        @click="closeBurgerMenu">
        <!-- <HistoryBrowser class="navbar-item" /> -->
        <b-button
          v-if="showSearchOnNavbar"
          icon-left="search"
          class="mr-2 mobile-nav-search-btn is-flex"
          @click="showMobileSearchBar">
        </b-button>
        <div v-show="openMobileSearchBar">
          <div
            class="fixed-stack is-flex is-align-items-center is-justify-content-space-between p-2">
            <Search
              ref="mobilSearchRef"
              hide-filter
              class="is-flex-grow-1 mt-5" />
            <b-button class="cancel-btn" @click="hideMobileSearchBar">
              {{ $t('cancel') }}
            </b-button>
          </div>
        </div>
      </div>
    </template>
    <template #start>
      <div v-if="showSearchOnNavbar" class="navbar-item is-expanded">
        <Search
          hide-filter
          class="search-navbar is-flex-grow-1 pb-0 is-hidden-touch"
          search-column-class="is-flex-grow-1" />
      </div>
    </template>
    <template #end>
      <ExploreDropdown />
      <CreateDropdown
        v-show="isCreateVisible"
        class="navbar-create custom-navbar-item"
        data-cy="create"
        :chain="chain" />
      <StatsDropdown
        class="navbar-stats custom-navbar-item"
        data-cy="stats"
        :chain="chain" />
      <ChainSelectDropdown
        id="NavChainSelect"
        class="navbar-chain custom-navbar-item"
        data-cy="chain-select" />
      <ProfileDropdown
        id="NavProfile"
        :chain="chain"
        data-cy="profileDropdown"
        @closeBurgerMenu="closeBurgerMenu" />
    </template>
  </b-navbar>
</template>

<script lang="ts">
import { Component, Ref, Watch, mixins } from 'nuxt-property-decorator'
import { get } from 'idb-keyval'

import BasicImage from '@/components/shared/view/BasicImage.vue'
import Identity from '@/components/identity/IdentityIndex.vue'
import ProfileDropdown from '~/components/navbar/ProfileDropdown.vue'
import Search from '@/components/search/Search.vue'
import ExploreDropdown from '~/components/navbar/ExploreDropdown.vue'
import CreateDropdown from '~/components/navbar/CreateDropdown.vue'
import KodaBetaDark from '@/assets/Koda_Beta_dark.svg'
import KodaBeta from '@/assets/Koda_Beta.svg'
import PrefixMixin from '@/utils/mixins/prefixMixin'

import { createVisible } from '@/utils/config/permision.config'
import { identityStore } from '@/utils/idbStore'
import AuthMixin from '@/utils/mixins/authMixin'
import ExperimentMixin from '@/utils/mixins/experimentMixin'
import ChainSelectDropdown from '~/components/navbar/ChainSelectDropdown.vue'
import StatsDropdown from '~/components/navbar/StatsDropdown.vue'

@Component({
  components: {
    Search,
    Identity,
    BasicImage,
    ProfileDropdown,
    ExploreDropdown,
    CreateDropdown,
    ChainSelectDropdown,
    StatsDropdown,
  },
})
export default class NavbarMenu extends mixins(
  PrefixMixin,
  AuthMixin,
  ExperimentMixin
) {
  public showTopNavbar = true
  public openMobileSearchBar = false
  private fixedTitleNavAppearDistance = 85
  private lastScrollPosition = 0
  private artistName = ''
  public isBurgerMenuOpened = false
  @Ref('mobilSearchRef') readonly mobilSearchRef
  @Watch('isBurgerMenuOpened') onDisableScroll() {
    if (this.isBurgerMenuOpened) {
      return (document.body.style.overflowY = 'hidden')
    } else {
      return (document.body.style.overflowY = 'initial')
    }
  }

  get chain(): string {
    return this.urlPrefix
  }

  get inCollectionPage(): boolean {
    return this.$route.name === 'rmrk-collection-id'
  }

  get inGalleryDetailPage(): boolean {
    return this.$route.name === 'rmrk-gallery-id'
  }

  get inUserProfilePage(): boolean {
    return this.$route.name === 'rmrk-u-id'
  }

  get isCreateVisible(): boolean {
    return createVisible(this.urlPrefix)
  }

  get isTargetPage(): boolean {
    // why?
    return (
      this.inCollectionPage ||
      this.inGalleryDetailPage ||
      this.inUserProfilePage
    )
  }

  get currentCollection() {
    return this.$store.getters['history/getCurrentlyViewedCollection'] || {}
  }

  get currentGalleryItemName() {
    return this.$store.getters['history/getCurrentlyViewedItem']?.name || ''
  }

  get isLandingPage() {
    return this.$route.name === 'index'
  }

  get isDarkMode() {
    return (
      this.$colorMode.preference === 'dark' ||
      document.documentElement.className.includes('dark-mode')
    )
  }

  get logoSrc() {
    return this.isDarkMode ? KodaBetaDark : KodaBeta
  }

  get showSearchOnNavbar(): boolean {
    return !this.isLandingPage || !this.showTopNavbar || this.isBurgerMenuOpened
  }

  get navBarTitle(): string {
    let title = ''
    if (this.inCollectionPage) {
      title = this.currentCollection.name
    } else if (this.inGalleryDetailPage) {
      title = this.currentGalleryItemName
    } else if (this.inUserProfilePage) {
      const address = this.$route.params.id
      title = this.artistName ? this.artistName : address
      if (!this.artistName) {
        this.fetchArtistIdentity(address)
      }
    }
    return title
  }

  async fetchArtistIdentity(address) {
    const identity = await get(address, identityStore)
    if (identity && identity.display) {
      this.artistName = identity.display
    }
  }

  onScroll() {
    const currentScrollPosition = document.documentElement.scrollTop
    const searchBarPosition = document
      .getElementById('networkList')
      ?.getBoundingClientRect()?.top
    if (currentScrollPosition <= 0) {
      this.showTopNavbar = true
      return
    }
    if (Math.abs(currentScrollPosition - this.lastScrollPosition) < 30) {
      return
    }
    if (this.isLandingPage && searchBarPosition) {
      this.showTopNavbar = searchBarPosition > this.fixedTitleNavAppearDistance
    } else {
      this.showTopNavbar =
        currentScrollPosition < this.fixedTitleNavAppearDistance
    }
    this.lastScrollPosition = currentScrollPosition
  }

  showMobileSearchBar() {
    this.openMobileSearchBar = true
    this.$nextTick(() => {
      this.mobilSearchRef?.focusInput()
    })
  }

  hideMobileSearchBar() {
    this.openMobileSearchBar = false
  }

  closeBurgerMenu() {
    if (this.isBurgerMenuOpened) {
      this.isBurgerMenuOpened = false
    }
  }

  mounted() {
    window.addEventListener('scroll', this.onScroll)
    document.body.style.overflowY = 'initial'
  }

  beforeDestroy() {
    window.removeEventListener('scroll', this.onScroll)
  }
}
</script>
