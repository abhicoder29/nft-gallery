<template>
  <div>
    <Loader
      v-model="isLoading"
      :status="status"
    />
    <div
      v-if="accountId"
      class="buttons"
    >
      <b-button
        v-for="action in actions"
        :key="action"
        :type="iconType(action)[0]"
        outlined
        @click="handleAction(action)"
      >
        {{ action === 'BUY' && replaceBuyNowWithYolo ? 'YOLO' : action }}
      </b-button>
    </div>
    <component
      :is="showMeta"
      v-if="showMeta"
      class="mb-4"
      empty-on-error
      @input="updateMeta"
    />
    <b-button
      v-if="showSubmit"
      type="is-primary"
      icon-left="paper-plane"
      @click="submit"
    >
      Submit {{ selectedAction }}
    </b-button>
  </div>
</template>

<script lang="ts" >
import { Component, mixins, Prop} from 'nuxt-property-decorator'
import Connector from '@vue-polkadot/vue-api'
import exec, { execResultValue, txCb } from '@/utils/transactionExecutor'
import { notificationTypes, showNotification } from '@/utils/notification'
import { unpin } from '@/utils/proxy'
import RmrkVersionMixin from '@/utils/mixins/rmrkVersionMixin'
import { somePercentFromTX } from '@/utils/support'
import shouldUpdate from '@/utils/shouldUpdate'
import nftById from '@/queries/nftById.graphql'
import PrefixMixin from '~/utils/mixins/prefixMixin'

const ownerActions = ['SEND', 'CONSUME', 'LIST']
const buyActions = ['BUY']

const needMeta: Record<string, string> = {
  SEND: 'AddressInput',
  LIST: 'BalanceInput'
}

type DescriptionTuple = [string, string] | [string];
const iconResolver: Record<string, DescriptionTuple> = {
  SEND: ['is-info is-dark'],
  CONSUME: ['is-danger'],
  LIST: ['is-light'],
  BUY: ['is-success is-dark']
}

type Action = 'SEND' | 'CONSUME' | 'LIST' | 'BUY' | '';

const components = {
  BalanceInput: () => import('@/components/shared/BalanceInput.vue'),
  AddressInput: () => import('@/components/shared/AddressInput.vue'),
  Loader: () => import('@/components/shared/Loader.vue')
}

@Component({ components })
export default class AvailableActions extends mixins(RmrkVersionMixin, PrefixMixin) {
  @Prop() public currentOwnerId!: string
  @Prop() public accountId!: string
  @Prop() public price!: string
  @Prop() public nftId!: string
  @Prop({ default: () => [] }) public ipfsHashes!: string[]
  private selectedAction: Action = ''
  private meta: string | number = ''
  protected isLoading = false
  protected status = ''

  get actions() {
    return this.isOwner
      ? ownerActions
      : this.isAvailableToBuy
        ? buyActions
        : []
  }

  get showSubmit() {
    return this.selectedAction && (!this.showMeta || this.metaValid)
  }

  get metaValid() {
    if (typeof this.meta === 'number') {
      return this.meta >= 0
    }

    return this.meta
  }

  get showMeta() {
    return needMeta[this.selectedAction]
  }

  get replaceBuyNowWithYolo(): boolean {
    return this.$store.getters['preferences/getReplaceBuyNowWithYolo']
  }

  protected iconType(value: string) {
    return iconResolver[value]
  }

  protected handleAction(action: Action) {
    if (shouldUpdate(action,  this.selectedAction)) {
      this.selectedAction = action
    } else {
      this.selectedAction = ''
      this.meta = ''
    }
  }

  get isOwner() {
    console.log(
      '{ currentOwnerId, accountId }',
      this.currentOwnerId,
      this.accountId
    )

    return (
      this.currentOwnerId &&
      this.accountId &&
      this.currentOwnerId === this.accountId
    )
  }

  get isAvailableToBuy() {
    const { price, accountId } = this
    return accountId && Number(price) > 0
  }

  private handleSelect(value: Action) {
    this.selectedAction = value
    this.meta = ''
  }

  private constructRmrk(): string {
    const { selectedAction, version, meta, nftId } = this
    return `RMRK::${selectedAction}::${version}::${nftId}${
      this.metaValid ? '::' + meta : ''
    }`
  }

  get isBuy() {
    return this.selectedAction === 'BUY'
  }

  get isConsume() {
    return this.selectedAction === 'CONSUME'
  }

  get isList() {
    return this.selectedAction === 'LIST'
  }

  get isSend() {
    return this.selectedAction === 'SEND'
  }

  protected updateMeta(value: string | number) {
    console.log(typeof value, value)
    this.meta = value
  }

  protected async checkBuyBeforeSubmit() {
    const nft = await this.$apollo.query({
      query: nftById,
      client: this.urlPrefix,
      variables: {
        id: this.nftId
      },
    })

    const { data: {nFTEntity} } = nft

    if (nFTEntity.currentOwner !== this.currentOwnerId || nFTEntity.burned || nFTEntity.price === 0 || nFTEntity.price !== this.price ) {
      showNotification(
        `[RMRK::${this.selectedAction}] Owner changed or NFT does not exist`,
        notificationTypes.warn
      )
      throw new ReferenceError('NFT has changed')
    }

  }

  protected async submit() {
    const { api } = Connector.getInstance()
    const rmrk = this.constructRmrk()
    this.isLoading = true

    try {
      showNotification(rmrk)
      console.log('submit', rmrk)
      const isBuy = this.isBuy
      const cb = isBuy ? api.tx.utility.batchAll : api.tx.system.remark
      const arg = isBuy ? [api.tx.system.remark(rmrk), api.tx.balances.transfer(this.currentOwnerId, this.price), somePercentFromTX(this.price)] : rmrk

      if (isBuy) {
        await this.checkBuyBeforeSubmit()
      }

      const tx = await exec(this.accountId, '', cb, [arg], txCb(
        async (blockHash) => {
          execResultValue(tx)
          showNotification(blockHash.toString(), notificationTypes.info)
          if (this.isConsume) {
            this.unpinNFT()
          }

          showNotification(
            `[${this.selectedAction}] ${this.nftId}`,
            notificationTypes.success
          )
          this.selectedAction = ''
          this.isLoading = false
        },
        err => {
          execResultValue(tx)
          showNotification(`[ERR] ${err.hash}`, notificationTypes.danger)
          this.selectedAction = ''
          this.isLoading = false
        },
        res => {
          if (res.status.isReady) {
            this.status = 'loader.casting'
            return
          }

          if (res.status.isInBlock) {
            this.status = 'loader.block'
            return
          }

          if (res.status.isFinalized) {
            this.status = 'loader.finalized'
            return
          }

          this.status = ''
        }
      ))

    } catch (e) {
      showNotification(`[ERR] ${e}`, notificationTypes.danger)
      console.error(e)
      this.isLoading = false
    }
  }

  protected unpinNFT() {
    this.ipfsHashes.forEach(async hash => {
      if (hash) {
        try {
          await unpin(hash)
        } catch (e) {
          console.warn(`[ACTIONS] Cannot Unpin ${hash} because: ${e}`)
        }
      }
    })
  }

  unlistNft() {
    // change the selected action to list and change meta value to 0
    this.selectedAction = 'LIST'
    this.meta = 0
    this.submit()
  }
}
</script>

