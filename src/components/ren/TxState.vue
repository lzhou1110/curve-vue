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
			<span @click="$emit('mint', transaction)">mint and swap now?</span>
		</span>
		<span v-show='state == 100'>
			Exchange rates expired, do you want to continue with the swap?
		</span>
	</div>
</template>

<script>
	// 0 - waiting for renVM gateway address, 1 - waiting for deposit on BTC address, 2 - got BTC transaction, waiting for confirmation
	// 3 - waiting for renVM to do it's magic and shift, 4 - got renBTC, now initiating swap, 5 - swap ready
	// 6 - has renBTC, left the page, so need to confirm eth swap tx
	export default {
		props: ['state', 'transaction'],
	}
</script>