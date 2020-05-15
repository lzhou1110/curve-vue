<template>
	
</template>

<script>
	import { getters, allCurrencies, contract } from '../../contract'
	import GatewayJS from "@renproject/gateway";

	export default {
		created() {
			this.$watch(() => contract.web3, val => this.mounted())
		},
		methods: {
			async mounted() {
				let gatewayjs = new GatewayJS('testnet')
				let contractAddress = 
				let amount = 0.001
				let res = await gatewayJS.open({
				    // Send BTC from the Bitcoin blockchain to the Ethereum blockchain.
				    sendToken: GatewayJS.Tokens.BTC.Btc2Eth,

				    // Amount of BTC we are sending (in Satoshis)
				    suggestedAmount: Math.floor(amount * (10 ** 8)), // Convert to Satoshis

				    // The contract we want to interact with
				    sendTo: '0x3973b2AcdFAC17171315E49eF19A0758b8B6F104',

				    // The name of the function we want to call
				    contractFn: "mintThenSwap",
				    
				    // The nonce is used to guarantee a unique deposit address.
				    nonce: GatewayJS.utils.randomNonce(),

				    // Arguments expected for calling `deposit`
				    contractParams: [
				        {
				            name: "_msg",
				            type: "bytes",
				            value: web3.utils.fromAscii(`Depositing ${amount} BTC`),
				        }
				    ],
				    
				    // Web3 provider for submitting mint to Ethereum
				    web3Provider: web3.currentProvider,
				}).result();

			}
		}
	}
</script>