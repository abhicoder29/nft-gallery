<template>
  <section class="wrapper no-padding-desktop gallery-item">
    <b-message type="is-primary" v-if="message">
      <div class="columns">
        <div class="column is-four-fifths">
          <p class="title is-3 has-text-black">{{ $t('mint.success') }} 🎉</p>
          <p class="subtitle is-size-5 has-text-black">
            {{ $t('mint.shareWithFriends', [nft.name]) }} △
          </p>
        </div>
        <div class="column">
          <Sharing onlyCopyLink />
        </div>
      </div>
    </b-message>

    <div class="columns">
      <div
        class="image-wrapper"
        @mouseenter="showNavigation = true"
        @mouseleave="showNavigation = false"
      >
        <button
          id="theatre-view"
          @click="toggleView"
          v-if="!isLoading && imageVisible"
        >
          {{ viewMode === 'default' ? $t('theatre') : $t('default') }}
          {{ $t('view') }}
        </button>
        <div
          class="column"
          :class="{
            'is-12 is-theatre': viewMode === 'theatre',
            'is-6 is-offset-3': viewMode === 'default',
          }"
        >
          <div
            v-orientation="
              viewMode === 'default' && !isFullScreenView && imageVisible
            "
            class="image-preview has-text-centered"
            :class="{ fullscreen: isFullScreenView }"
          >
            <b-image
              v-if="!isLoading && imageVisible && !meta.animation_url"
              :src="meta.image || '/placeholder.svg'"
              src-fallback="/placeholder.svg'"
              alt="KodaDot NFT minted multimedia"
              ratio="1by1"
              @error="onImageError"
            ></b-image>
            <img
              class="fullscreen-image"
              :src="meta.image || '/placeholder.svg'"
              alt="KodaDot NFT minted multimedia"
            />
            <b-skeleton
              height="524px"
              size="is-large"
              :active="isLoading"
            ></b-skeleton>
            <MediaResolver
              v-if="meta.animation_url"
              :class="{ withPicture: imageVisible }"
              :src="meta.animation_url"
              :poster="meta.image"
              :description="meta.description"
              :mimeType="mimeType"
            />
          </div>
          <Navigation v-if="nftsFromSameCollection && nftsFromSameCollection.length > 1" :showNavigation="showNavigation" :items="nftsFromSameCollection" :currentId="nft.id"/>
        </div>
        <button
          id="fullscreen-view"
          @keyup.esc="minimize"
          @click="toggleFullScreen"
          v-if="!isLoading && imageVisible"
          :class="{ fullscreen: isFullScreenView }"
        >
          <b-icon :icon="isFullScreenView ? 'compress-alt' : 'arrows-alt'">
          </b-icon>
        </button>
      </div>
    </div>

    <div class="columns">
      <div class="column is-6">
        <Appreciation
          :emotes="nft.emotes"
          :accountId="accountId"
          :currentOwnerId="nft.currentOwner"
          :nftId="nft.id"
          :burned="nft.burned"
        />
        <div class="nft-title">
          <Name :nft="nft" :isLoading="isLoading" />
        </div>

        <div v-if="meta.description" class="block">
          <p class="label">{{ $t('legend')}}</p>
          <b-skeleton :count="3" size="is-large" :active="isLoading"></b-skeleton>
          <DescriptionWrapper v-if="!isLoading" :text="meta.description.replaceAll('\n', '  \n')" />
        </div>
      </div>

      <div class="column is-6" v-if="detailVisible">
        <b-skeleton :count="2" size="is-large" :active="isLoading"></b-skeleton>

        <div class="columns">
          <div class="column">
            <div class="nft-title">
              <Detail :nft="nft" :isLoading="isLoading" />
            </div>
          </div>
          <div
            class="
              column
              is-flex is-flex-direction-column is-justify-content-space-between
            "
          >
            <div class="card card-actions mb-4" aria-id="contentIdForA11y3">
              <div class="card-content">
                <template v-if="hasPrice">
                  <div class="label">
                    {{ $t('price') }}
                  </div>
                  <div class="price-block__container">
                    <div class="price-block__original">
                      {{ nft.price | formatBalance(12, 'KSM') }}
                    </div>
                    <b-button
                      v-if="nft.currentOwner === accountId"
                      type="is-warning"
                      outlined
                      @click="handleUnlist"
                    >
                      {{ $t('Unlist') }}
                    </b-button>
                  </div>
                </template>

                <div class="content pt-4">
                  <p class="subtitle">
                    <IndexerGuard show-message class="pb-4">
                      <AvailableActions
                        ref="actions"
                        :account-id="accountId"
                        :current-owner-id="nft.currentOwner"
                        :price="nft.price"
                        :nft-id="nft.id"
                        :ipfs-hashes="[
                          nft.image,
                          nft.animation_url,
                          nft.metadata,
                        ]"
                        @change="handleAction"
                      />
                    </IndexerGuard>
                    <Auth />
                  </p>
                </div>

                <Sharing class="mb-4" />
              </div>
            </div>
          </div>
        </div>
        <PriceChart class="mt-4" :priceChartData="priceChartData" :openOnDefault="!compactGalleryItem" />
      </div>
    </div>

    <div class="columns">
      <div class="column">
        <History
          v-if="!isLoading"
          :events="nft.events"
          :open-on-default="!compactGalleryItem"
          @setPriceChartData="setPriceChartData"
        />
      </div>
    </div>
  </section>
</template>

<script lang="ts" >
import { Component, mixins, Vue, Watch } from 'nuxt-property-decorator'
import { NFT, NFTMetadata, Emote } from '../service/scheme'
import { sanitizeIpfsUrl, resolveMedia, getSanitizer } from '../utils'
import { emptyObject } from '@/utils/empty'

import AvailableActions from './AvailableActions.vue'
import { notificationTypes, showNotification } from '@/utils/notification'
// import Money from '@/components/shared/format/Money.vue';
// import/ Sharing from '@/components/rmrk/Gallery/Item/Sharing.vue';
// import Facts from '@/components/rmrk/Gallery/Item/Facts.vue';
// import Name from '@/components/rmrk/Gallery/Item/Name.vue';
// import VueMarkdown from 'vue-markdown-render'

import isShareMode from '@/utils/isShareMode'
import nftById from '@/queries/nftById.graphql'
import nftListIdsByCollection from '@/queries/nftListIdsByCollection.graphql'
import { fetchNFTMetadata } from '../utils'
import { get, set } from 'idb-keyval'
import { MediaType } from '../types'
import axios from 'axios'
import { exist } from './Search/exist'
import Orientation from '@/directives/DeviceOrientation'
import PrefixMixin from '~/utils/mixins/prefixMixin'

@Component<GalleryItem>({
  metaInfo() {
    const image = `https://og-image-green-seven.vercel.app/${encodeURIComponent(
      this.nft.name as string
    )}.png?price=${
      Number(this.nft.price)
        ? Vue.filter('formatBalance')(this.nft.price, 12, 'KSM')
        : ''
    }&image=${this.meta.image as string}&mime=${this.mimeType}`
    return {
      title: this.nft.name,
      titleTemplate: '%s | Low Carbon NFTs',
      meta: [
        { name: 'description', content: this.meta.description as string },
        { property: 'og:title', content: this.nft.name as string },
        {
          property: 'og:description',
          content: this.meta.description as string,
        },
        { property: 'og:image', content: image },
        { property: 'og:video', content: this.meta.image as string },
        { property: 'og:author', content: this.nft.currentOwner as string },
        { property: 'twitter:title', content: this.nft.name as string },
        {
          property: 'twitter:description',
          content: this.meta.description as string,
        },
        { property: 'twitter:image', content: image },
      ],
    }
  },
  components: {
    Auth: () => import('@/components/shared/Auth.vue'),
    AvailableActions: () => import('./AvailableActions.vue'),
    Facts: () => import('@/components/rmrk/Gallery/Item/Facts.vue'),
    // MarkdownItVueLight: MarkdownItVueLight as VueConstructor<Vue>,
    History: () => import('@/components/rmrk/Gallery/History.vue'),
    Money: () => import('@/components/shared/format/Money.vue'),
    Name: () => import('@/components/rmrk/Gallery/Item/Name.vue'),
    Navigation: () => import('@/components/rmrk/Gallery/Item/Navigation.vue'),
    Sharing: () => import('@/components/rmrk/Gallery/Item/Sharing.vue'),
    Appreciation: () => import('@/components/rmrk/Gallery/Appreciation.vue'),
    MediaResolver: () => import('@/components/rmrk/Media/MediaResolver.vue'),
    // PackSaver: () => import('../Pack/PackSaver.vue'),
    IndexerGuard: () => import('@/components/shared/wrapper/IndexerGuard.vue'),
    DescriptionWrapper: () => import('@/components/shared/collapse/DescriptionWrapper.vue'),
    Detail: () => import('@/components/rmrk/Gallery/Item/Detail.vue'),
    PriceChart: () => import('@/components/rmrk/Gallery/PriceChart.vue'),
  },
  directives: {
    orientation: Orientation,
  },
})
export default class GalleryItem extends mixins(PrefixMixin) {
  private id = ''
  // private accountId: string = '';
  private passsword = ''
  private nft: NFT = emptyObject<NFT>()
  private nftsFromSameCollection: NFT[] = []
  private imageVisible = true
  private viewMode = this.$store.getters['preferences/getTheatreView']
  private isFullScreenView = false
  public isLoading = true
  public mimeType = ''
  public meta: NFTMetadata = emptyObject<NFTMetadata>()
  public emotes: Emote[] = []
  public message = ''
  public priceChartData: [Date, number][][] = []
  public showNavigation = false

  get accountId() {
    return this.$store.getters.getAuthAddress
  }

  public async created() {
    this.checkId()
    exist(this.$route.query.message, (val) => {
      this.message = val === 'congrats' ? val : ''
      this.$router.replace({ query: null } as any)
    })

    try {
      // const nft = await rmrkService.getNFT(this.id);
      this.$apollo.addSmartQuery('nft', {
        client: this.urlPrefix,
        query: nftById,
        variables: {
          id: this.id,
        },
        update: ({ nFTEntity }) => ({
          ...nFTEntity,
          emotes: nFTEntity?.emotes?.nodes,
        }),
        result: () => {
          Promise.all([
            this.fetchMetadata(),
            this.fetchCollectionItems()
          ])
        },
        pollInterval: 5000,
      })
    } catch (e) {
      showNotification(`${e}`, notificationTypes.warn)
    }

    this.isLoading = false
  }

  onImageError(e: any) {
    console.warn('Image error', e)
  }

  public setPriceChartData(data: [Date, number][][]) {
    this.priceChartData = data
  }

  public async fetchCollectionItems() {
    const collectionId = (this.nft as any)?.collectionId
    if(collectionId) {
      // cancel request and get ids from store in case we already fetched collection data before
      if(this.$store.state.history?.currentCollection?.id === collectionId) {
        this.nftsFromSameCollection = this.$store.state.history.currentCollection?.nftIds || []
        return
      }
      try {
        const nfts = await this.$apollo.query({
          query: nftListIdsByCollection,
          client: this.urlPrefix,
          variables: {
            id: collectionId,
          },
        })

        const {
          data: {
            nFTEntities
          }
        } =  nfts

        this.nftsFromSameCollection = nFTEntities?.nodes.map((n: { id: string }) => n.id) || []
        this.$store.dispatch('history/setCurrentCollection', { id: collectionId, nftIds: this.nftsFromSameCollection })
      } catch (e) {
        showNotification(`${e}`, notificationTypes.warn)
      }
    }
  }

  public async fetchMetadata() {
    if (this.nft['metadata'] && !this.meta['image']) {
      const m = await get(this.nft.metadata)

      const meta = m
        ? m
        : await fetchNFTMetadata(
          this.nft,
          getSanitizer(this.nft.metadata, undefined, 'permafrost')
        )
      console.log(meta)

      const imageSanitizer = getSanitizer(meta.image)
      this.meta = {
        ...meta,
        image: imageSanitizer(meta.image),
        animation_url: sanitizeIpfsUrl(
          meta.animation_url || meta.image,
          'pinata'
        ),
      }

      // console.log(this.meta)
      if (this.meta.animation_url && !this.mimeType) {
        const { headers } = await axios.head(this.meta.animation_url)
        this.mimeType = headers['content-type']
        // console.log(this.mimeType)
        const mediaType = resolveMedia(this.mimeType)
        this.imageVisible = ![
          MediaType.VIDEO,
          MediaType.MODEL,
          MediaType.IFRAME,
          MediaType.OBJECT,
        ].some((t) => t === mediaType)
      }

      if (!m) {
        set(this.nft.metadata, meta)
      }
    }
  }

  public checkId() {
    if (this.$route.params.id) {
      this.id = this.$route.params.id
    }
  }

  public toggleView(): void {
    this.viewMode = this.viewMode === 'default' ? 'theatre' : 'default'
  }

  public toggleFullScreen(): void {
    this.isFullScreenView = !this.isFullScreenView
  }

  public minimize(): void {
    this.isFullScreenView = false
  }

  public toast(message: string): void {
    this.$buefy.toast.open(message)
  }

  get hasPrice() {
    return Number(this.nft.price) > 0
  }

  get nftId() {
    const { id } = this.nft
    return id
  }

  get detailVisible() {
    return !isShareMode
  }

  get compactGalleryItem(): boolean {
    return this.$store.state.preferences.compactGalleryItem
  }

  protected handleAction(deleted: boolean) {
    if (deleted) {
      showNotification('INSTANCE REMOVED', notificationTypes.warn)
    }
  }

  protected handleUnlist() {
    // call unlist function from the AvailableActions component
    (this.$refs.actions as AvailableActions).unlistNft()
  }

  @Watch('meta.image')
  handleNFTPopulationFinished(newVal) {
    if(newVal) {
    // save visited detail page to history
      this.$store.dispatch('history/addHistoryItem', { id: this.id, name: this.nft.name, image: this.meta.image, collection: (this.nft.collection as any).name, date: new Date() })
    }
  }
}
</script>

<style lang="scss">
@import '@/styles/variables';

hr.comment-divider {
  border-top: 1px solid lightpink;
  border-bottom: 1px solid lightpink;
}

.gallery-item {
  .nft-title {
    margin-bottom: 24px;
  }

  .gallery-item__skeleton {
    width: 95%;
    margin: auto;
  }

  .image-wrapper {
    position: relative;
    margin: 30px auto;
    width: 100%;

    .image {
      border: 2px solid $primary;
    }

    .fullscreen-image {
      display: none;
    }

    .image-preview {
      &.fullscreen {
        position: fixed;
        width: 100%;
        height: 100%;
        top: 0;
        left: 0;
        z-index: 999998;
        background: #000;

        img.fullscreen-image {
          display: block;
          object-fit: contain;
          width: 100%;
          height: 100%;
          overflow: auto;
          position: absolute;
          top: 0;
          left: 50%;
          transform: translate(-50%, 0);
          overflow-y: hidden;
        }

        .image {
          visibility: hidden;
        }
      }
    }

    .column {
      transition: 0.3s all;
    }

    button {
      border: 2px solid $primary;
      color: #fff;
      font-weight: bold;
      text-transform: uppercase;
      padding: 7px 16px;
      font-size: 20px;
      background: $scheme-main;
      z-index: 2;

      &:hover {
        background: $primary;
        cursor: pointer;
      }
    }
  }

  button#theatre-view {
    position: absolute;
    top: 13px;
    left: 13px;
    color: $light-text;
    @media screen and (max-width: 768px) {
      display: none;
    }
  }

  button#fullscreen-view {
    position: absolute;
    bottom: 13px;
    right: 13px;

    &.fullscreen {
      position: fixed;
      z-index: 999998;
      bottom: 0;
      right: 0;
    }
  }

  .price-block {
    border: 2px solid $primary;
    padding: 14px;

    &__original {
      font-size: 24px;
      text-transform: uppercase;
      font-weight: 500;
    }

    &__container {
      display: flex;
      justify-content: space-between;
      align-items: center;
    }

    &__exchange {
      opacity: 0.6;
      color: $dark;
      margin: 0;
    }
  }

  .card-actions {
    border-radius: 0 !important;
    box-shadow: none;
    border: 2px solid $primary;

    &-content {
      padding-left: 1rem;
      padding-top: 1rem;
    }

    &-footer {
      &-item {
        padding: 0.75rem !important;
      }
    }
  }

  &.no-padding-desktop {
    @media screen and (min-width: 1023px) {
      padding: 0;
    }
  }
}
</style>
