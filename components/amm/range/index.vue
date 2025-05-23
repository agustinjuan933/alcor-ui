<template lang="pug">
client-only
  .div

    // TODO This things
    p.infoBox(v-if="isUninitialized") {{ $t('Your position will appear here.') }}
    p.infoBox(v-if="isLoading") {{ $t('Loading') }}
    p.infoBox(v-if="error") {{ $t('Liquidity data not available.') }}
    //

    .chartWrapper(ref="chartWrapper")
      Chart(
        :current="price"
        :series="series"
        :width="isMobile ? 300 : 400"
        :height="240"
        :margins="{ top: 10, right: 2, bottom: 20, left: 0 }"
        :interactive="interactive"
        :brushLabel="brushLabel"
        :brushDomain="brushDomain"
        @onBrushDomainChange="onBrushDomainChangeEnded"
        :zoomLevels="zoomLevels"
        :title="chartTitle"
        :ticksAtLimit="ticksAtLimit")

        template(slot="header")
          .fs-18.current-price(v-if="price && tokenA && tokenB")
            span.disable {{ $t('Current Price') }}:&nbsp;
            span {{ price }} {{ tokenB.symbol }} per {{ tokenA.symbol }}
        template(#afterZoomIcons)
          slot(name="afterZoomIcons")
</template>

<script>
import { format } from 'd3'
import { FeeAmount } from '@alcorexchange/alcor-swap-sdk'

import Chart from './Chart.vue'

import { ZOOM_LEVELS } from './constants'

import { parseToken } from '~/utils/amm'

export default {
  components: { Chart },
  props: ['tokenA', 'tokenB', 'feeAmount', 'ticksAtLimit', 'price', 'priceLower', 'priceUpper', 'interactive', 'chartTitle'],

  data() {
    return {
      error: false,
      ZOOM_LEVELS,
      FeeAmount,
      width: 400,
      series: [],
      // series: [
      //   {
      //     x: 1834304665020282,
      //     y: 7082829
      //   },
      //   {
      //     x: 0.94743466,
      //     y: 7082829
      //   },
      //   {
      //     x: 0.46955859,
      //     y: 57359
      //   },
      //   {
      //     x: 0,
      //     y: 0
      //   }]
    }
  },

  computed: {
    isSorted() {
      return this.tokenA.sortsBefore(this.tokenB)
    },

    isUninitialized() {
      return !this.tokenA || !this.tokenB || (this.series === undefined && !this.isLoading)
    },

    isLoading: () => false,

    zoomLevels() {
      return ZOOM_LEVELS[this.feeAmount || FeeAmount.MEDIUM]
    },

    brushDomain() {
      // Все ок
      const { isSorted, priceLower, priceUpper } = this

      const leftPrice = isSorted ? priceLower : priceUpper?.invert()
      const rightPrice = isSorted ? priceUpper : priceLower?.invert()

      return leftPrice && rightPrice
        ? [parseFloat(leftPrice?.toSignificant(6)), parseFloat(rightPrice?.toSignificant(6))]
        : undefined
    }
  },

  watch: {
    feeAmount() {
      this.fetchSeries()
    },

    isSorted() {
      this.fetchSeries()
    }
  },

  mounted() {
    this.fetchSeries()
  },

  methods: {
    async fetchSeries() {
      this.series = []
      const { tokenA, tokenB, feeAmount } = this

      const pool = this.$store.state.amm.pools.find(p => {
        return (parseToken(p.tokenA).equals(tokenA) && parseToken(p.tokenB).equals(tokenB) && p.fee == feeAmount) ||
        (parseToken(p.tokenA).equals(tokenB) && parseToken(p.tokenB).equals(tokenA) && p.fee == feeAmount)
      })

      const { data: series } = await this.$axios.get('/v2/swap/pools/' + pool.id + '/liquidityChartSeries', { params: { inverted: !this.isSorted } })

      this.series = series
    },

    brushLabel(d, x) {
      const { price, ticksAtLimit, isSorted } = this

      if (!price) return ''

      if (d === 'w' && ticksAtLimit[isSorted ? 'LOWER' : 'UPPER']) return '0'
      if (d === 'e' && ticksAtLimit[isSorted ? 'UPPER' : 'LOWER']) return '∞'

      const percent = (x < price ? -1 : 1) * ((Math.max(x, price) - Math.min(x, price)) / price) * 100

      return price ? `${format(Math.abs(percent) > 1 ? '.2~s' : '.2~f')(percent)}%` : ''
    },

    onBrushDomainChangeEnded({ domain, mode, initialInfinty }) {
      const { ticksAtLimit, isSorted } = this

      let leftRangeValue = Number(domain[0])
      const rightRangeValue = Number(domain[1])

      if (initialInfinty) {
        this.$emit('onPreDefinedRangeSelect', { lowerValue: 'infinity' })
        return
      }

      if (leftRangeValue <= 0) leftRangeValue = 1 / 10 ** 6

      // Вызывает изначальные занчения для заполнения инпутов
      if (
        (!ticksAtLimit[isSorted ? 'LOWER' : 'UPPER'] || mode === 'handle' || mode === 'reset') &&
        leftRangeValue > 0
      ) {
        this.$emit('onLeftRangeInput', leftRangeValue.toFixed(20))
      }

      if ((!ticksAtLimit[isSorted ? 'UPPER' : 'LOWER'] || mode === 'reset') && rightRangeValue > 0) {
        // todo: remove this check. Upper bound for large numbers
        // sometimes fails to parse to tick.
        if (rightRangeValue < 1e35) {
          this.$emit('onRightRangeInput', rightRangeValue.toFixed(20))
        }
      }
    }
  }
}
</script>

<style scoped>
.chartWrapper {
  position: relative;
  justify-content: center;
  align-content: center;
}
.current-price{
  text-align: center;
}
</style>
