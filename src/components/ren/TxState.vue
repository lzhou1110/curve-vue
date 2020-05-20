<template>
	<div>
		<span v-show='state == 0'>
			Waiting for renVM gateway address
		</span>
		<span v-show='state == 1'>
			Waiting for deposit on BTC address
		</span>
		<span v-show='state == 2'>
			Got BTC transaction, waiting for confirmation
		</span>
		<span v-show='state == 3'>
			Waiting for renVM to do it's magic and shift
		</span>
		<span v-show='state == 4'>
			Got BTC {{ transaction.utxoAmount }}, now initiating swap
		</span>
		<span v-show='state == 5'>
			Swap ready
		</span>
		<span v-show='state == 5'>
			Swap ready
		</span>
		<span v-show='state == 6'>
			Got BTC, {{ transaction.utxoAmount }}, do you want to 
			<button @click="$emit('mint', transaction)">mint and swap now?</button>
		</span>
		<span v-show='state == 7'>
			Exchange rates expired, you'll get {{ (transaction.utxoAmount - renFee*1e18) }} BTC shifted to renBTC.
			If you want to continue swapping to WBTC, go to the <router-link to="/ren/swap">swap page</router-link>
		</span>
	</div>
</template>

<script>
	import BN from 'bignumber.js'

	export default {
		props: ['state', 'transaction'],
		computed: {
			renFee() {
				return BN(this.transaction.utxoAmount).times(1e8).times(0.001).div(1e8).toFixed()
			},
		}
	}
</script>