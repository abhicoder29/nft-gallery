<template>
  <div
    class="card nft-card"
    :class="{ 'is-current-owner': accountIsCurrentOwner() }"
  >
    <LinkResolver
      class="nft-card__skeleton"
      :route="route"
      :link="link"
      :param="id"
      tag="a"
    >
      <div
        v-if="image"
        class="card-image"
      >
        <span
          v-if="emoteCount"
          class="card-image__emotes"
        >
          <b-icon icon="heart" />
          <span class="card-image__emotes__count">{{ emoteCount }}</span>
        </span>
        <BasicImage
          :src="image"
          :alt="title"
          custom-class="gallery__image-wrapper"
        />
        <span
          v-if="price > 0 && !hidePriceValue"
          class="card-image__price"
        >
          <Money
            :value="price"
            inline
          />
        </span>
      </div>

      <div
        v-else
        class="card-image"
      >
        <span
          v-if="emoteCount"
          class="card-image__emotes"
        >
          <b-icon icon="heart" />
          <span class="card-image__emotes__count">{{ emoteCount }}</span>
        </span>

        <b-image
          :src="placeholder"
          alt="Simple image"
          ratio="1by1"
        />

        <span
          v-if="price > 0"
          class="card-image__price"
        >
          <Money
            :value="price"
            inline
          />
        </span>
      </div>

      <div class="card-content">
        <span
          class="title mb-0 is-4 has-text-centered has-text-primary"
          :title="name"
        >
          <div class="has-text-overflow-ellipsis">
            {{ nftName }}
          </div>
        </span>
      </div>
    </LinkResolver>
  </div>
</template>

<script lang="ts" >
import { Component, mixins, Prop, Vue, Watch } from 'nuxt-property-decorator'
import { get, update } from 'idb-keyval'
import shouldUpdate from '@/utils/shouldUpdate'
import { fetchNFTMetadata, getSanitizer } from '../utils'
import { NFT } from '../service/scheme'
import AuthMixin from '@/utils/mixins/authMixin'

const components = {
  LinkResolver: () => import('@/components/shared/LinkResolver.vue'),
  Money: () => import('@/components/shared/format/Money.vue'),
  BasicImage: () => import('@/components/shared/view/BasicImage.vue'),
}

@Component({ components })
export default class GalleryCard extends mixins(AuthMixin) {
  @Prop({ default: '/rmrk/gallery' }) public route!: string
  @Prop({ default: 'rmrk/gallery' }) public link!: string
  @Prop() public id!: string
  @Prop() public name!: string
  protected image = ''
  protected title = ''
  @Prop() public emoteCount!: string | number
  @Prop() public imageType!: string
  @Prop() public price!: string
  @Prop() public metadata!: string
  @Prop() public currentOwner!: string

  private placeholder = '/placeholder.webp'

  async mounted() {
    if (this.metadata) {
      const meta = await get(this.metadata)
      if (meta) {
        this.image = getSanitizer(meta.image || '')(meta.image || '')
        this.title = meta.name
      } else {
        const m = await fetchNFTMetadata(
          { metadata: this.metadata } as NFT,
          getSanitizer(this.metadata, undefined, 'permafrost')
        )
        this.image = getSanitizer(m.image || '')(m.image || '')
        this.title = m.name
        update(this.metadata, () => m)
      }
    }
  }

  @Watch('accountId', { immediate: true })
  hasAccount(value: string, oldVal: string) {
    if (shouldUpdate(value, oldVal)) {
      this.accountIsCurrentOwner()
    }
  }

  get hidePriceValue(): boolean {
    return this.$store.getters['preferences/getHidePriceValue']
  }

  get nftName(): string {
    return this.name || this.title
  }

  public accountIsCurrentOwner() {
    return this.accountId === this.currentOwner
  }
}
</script>

<style lang="scss">
@import '@/styles/variables';

.nft-card {
  border-radius: 8px;
  position: relative;
  overflow: hidden;
  box-shadow: 0px 2px 5px 0.5px $primary-light;

  &.is-current-owner {
    box-shadow: 0px 2px 5px 0.5px #41b883;
  }

  .has-text-overflow-ellipsis {
    overflow: hidden;
    white-space: nowrap;
    text-overflow: ellipsis;
    color: #fff;
  }

  &__skeleton {
    .ff-canvas {
      border-radius: 8px;
    }
    transition: all 0.3s;

    .card-image {
      &__emotes {
        position: absolute;
        background-color: $primary-light;
        border-radius: 4px;
        padding: 3px 8px;
        color: #fff;
        top: 10px;
        right: 10px;
        font-size: 14px;
        z-index: 3;
        transition: all 0.3s;
      }

      &__price {
        position: absolute;
        background-color: #363636;
        border-radius: 4px;
        padding: 3px 8px;
        color: #fff;
        bottom: 10px;
        left: 10px;
        font-size: 14px;
        z-index: 3;
        transition: all 0.3s ease;
      }
    }
  }

  .card-image__emotes__count {
    vertical-align: text-bottom;
  }

  @media screen and (min-width: 1024px) {
    .card-content {
      position: absolute;
      bottom: -45px;
      left: 0;
      width: 100%;
      background: #fff;
      opacity: 0;
    }

    .card-image img {
      transition: all 0.3s ease;
    }

    &:hover .card-content {
      bottom: 0;
      opacity: 1;
      z-index: 2;
      background: #000 !important;
    }

    &:hover .card-image img {
      transform: scale(1.1) translateY(-5%);
    }

    &:hover .ff-canvas {
      transform: scale(1.1) translateY(-50%);
    }

    &:hover .card-image__emotes {
      top: 15px;
      right: 15px;
    }

    &:hover .card-image__price {
      bottom: 62px;
    }
  }
}
</style>
