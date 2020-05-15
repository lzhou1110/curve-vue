<template>
	<div class='window white'>
		<div>
			<input type='radio' id='getwbtc' :value='0' v-model='type'>
			<label for='getwbtc'>Get WBTC</label>

			<input type='radio' id='getbtc' :value='1' v-model='type'>
			<label for='getbtc'>Get BTC</label>
		</div>

		<div class='input'>
			<label for='amount'>{{ type == 0 ? 'BTC' : 'WBTC' }} amount</label>
			<input id='amount' type='text' v-model='amount' placeholder='Amount'>
		</div>

		<div class='input'>
			<label for='address'>{{ type == 1 ? 'WBTC' : 'BTC' }} address</label>
			<input id='address' type='text' v-model='address' placeholder='Address'>
		</div>

		{{ gatewayAddress }}
		{{ btcConfirms }}
		{{ btcTxHash }}
		{{ btcTxVOut }}

		<button @click='submit'>Submit</button>

		<table class='tui-table'>
			<thead>
				<tr>
					<th>Address</th>
					<th>Confirmations</th>
					<th>Status</th>
				</tr>
			</thead>
			<tbody>
				<tr v-for='transaction in transactions'>
					<td>{{ transaction.gatewayAddress }}</td>
					<td>{{ transaction.confirmations }}</td>
					<td>{{ transaction.state }}</td>
				</tr>
			</tbody>
		</table>
	</div>	
</template>

<script>
	import { getters, allCurrencies, contract } from '../../contract'
	import RenSDK from '@renproject/ren'
	import BigNumber from 'bignumber.js'
	import * as helpers from '../../utils/helpers'

	const adapterABI = [{"inputs":[{"internalType":"contract ICurveExchange","name":"_exchange","type":"address"},{"internalType":"contract IGatewayRegistry","name":"_registry","type":"address"},{"internalType":"contract IERC20","name":"_wbtc","type":"address"},{"internalType":"address","name":"_owner","type":"address"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"oldRelayHub","type":"address"},{"indexed":true,"internalType":"address","name":"newRelayHub","type":"address"}],"name":"RelayHubChanged","type":"event"},{"payable":true,"stateMutability":"payable","type":"fallback"},{"constant":true,"inputs":[{"internalType":"address","name":"","type":"address"},{"internalType":"address","name":"","type":"address"},{"internalType":"bytes","name":"","type":"bytes"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"bytes","name":"","type":"bytes"},{"internalType":"uint256","name":"","type":"uint256"}],"name":"acceptRelayedCall","outputs":[{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"bytes","name":"","type":"bytes"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"exchange","outputs":[{"internalType":"contract ICurveExchange","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"getHubAddr","outputs":[{"internalType":"address","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"internalType":"uint256","name":"_minWbtcAmount","type":"uint256"},{"internalType":"address payable","name":"_wbtcDestination","type":"address"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"bytes32","name":"_nHash","type":"bytes32"},{"internalType":"bytes","name":"_sig","type":"bytes"}],"name":"mintDontSwap","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"uint256","name":"_minWbtcAmount","type":"uint256"},{"internalType":"address payable","name":"_wbtcDestination","type":"address"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"bytes32","name":"_nHash","type":"bytes32"},{"internalType":"bytes","name":"_sig","type":"bytes"}],"name":"mintThenSwap","outputs":[{"internalType":"uint256","name":"wbtcBought","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"context","type":"bytes"},{"internalType":"bool","name":"success","type":"bool"},{"internalType":"uint256","name":"actualCharge","type":"uint256"},{"internalType":"bytes32","name":"preRetVal","type":"bytes32"}],"name":"postRelayedCall","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"context","type":"bytes"}],"name":"preRelayedCall","outputs":[{"internalType":"bytes32","name":"","type":"bytes32"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"registry","outputs":[{"internalType":"contract IGatewayRegistry","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"relayHubVersion","outputs":[{"internalType":"string","name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"_btcDestination","type":"bytes"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"uint256","name":"_minRenbtcAmount","type":"uint256"}],"name":"swapThenBurn","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"}]

	const adapterAddress = '0x3973b2acdfac17171315e49ef19a0758b8b6f104'

	export default {
		data: () => ({
			type: 0,
			amount: '0.0001',
			address: '0x309b8Da6166eDaa47664b595041899325B859fBe',
			sdk: new RenSDK('testnet'),
			transactions: [{
				id: 1,
				gatewayAddress: '1',
				confirmations: 3,
				state: 3,
			}],


			gatewayAddress: '',
			btcConfirms: 0,
			btcTxHash: '',
			btcTxVOut: '',

			renResponse: '',
			renSignature: '',
			default_account: '',
			adapterContract: null,

			confirmations: 0,
			// 1 - getting btc deposit address, 2 - waiting to confirm on btc network, 3 - 
			state: null,
		}),
		watch: {
			type() {
				this.amount = ''
				this.address = ''
			},
			transactions(val) {
				//use 3box
				localStorage.setItem('transactions', JSON.stringify(val))
			}
		},
		created() {
			this.$watch(() => contract.web3, val => {
				this.mounted()
			})
		},
		mounted() {
			contract.web3 && this.mounted()
		},
		methods: {
			async mounted() {
				this.adapterContract = new web3.eth.Contract(adapterABI, adapterAddress)
				this.default_account = (await web3.eth.getAccounts())[0]
				//resume only transactions submitted to btc network
				await Promise.all(this.transactions.filter(t => t.btcTxHash).map(t=>this.sendMint(t)))
			},

			async mint() {
				let mint = await this.initMint();
				let transaction = this.transactions[this.transactions.length - 1]
				await this.sendMint(transaction, mint)
			},

			async initMint(transaction) {
				if(transaction) {
					var { id, amount, nonce, address } = transaction
				}
				else {
					amount = this.amount
					address = this.address
				}
				
				let transfer = {
				    // Send BTC from the Bitcoin blockchain to the Ethereum blockchain.
				    sendToken: RenJS.Tokens.BTC.Btc2Eth,

				    // The contract we want to interact with
				    sendTo: adapterAddress,

				    suggestedAmount: RenJS.utils.value(amount, "btc").sats().toNumber(),

				    // The name of the function we want to call
				    contractFn: "mintThenSwap",
				    
				    nonce: nonce || RenJS.utils.randomNonce(),

				    // Arguments expected for calling `deposit`
				    contractParams: [
				        {
			                name: "_minWbtcAmount",
			                type: "uint256",
			                value: 0
			            },
			            {
			                name: "_wbtcDestination",
			                type: "address",
			                value: address,
			            },
				    ],
				    
				    // Web3 provider for submitting mint to Ethereum
				    web3Provider: web3.currentProvider,
				}
				const mint = this.sdk.lockAndMint(transfer);

				if(!id) {
					transfer.id = helpers.generateID();
					transfer.amount = this.amount;
					transfer.address = this.address;
					this.transactions.push(transfer)
				}

				return mint;
			},

			async sendMint(transfer, mint) {

				let transaction = this.transcations.find(t => t.id == transfer.id)
				//transaction is ready to be sent to eth network
				if(transaction.renResponse && transaction.renSignature) {
					await this.mintThenSwap(transfer)
				}
				else {
					mint = mint || await initMint(transfer);

					//transaction initated, but didn't get an address, so updating
					if(!transaction.params) {
						transaction.params = mint.params;
						transaction.gatewayAddress = await mint.gatewayAddress();
					}

					let deposit
					//transaction was submitted to btc network
					if(transaction.btcTxHash && String(transaction.btcTxVOut) !== 'undefined') {
						deposit = await mint.wait(this.confirmations, {
							txHash: transaction.btcTxHash,
							vOut: transaction.btcTxVOut,
						})
					}
					//trasaction not submitted to btc network
					else {
						deposit = await mint.wait(this.confirmations)
										.on('deposit', deposit => {
											if(deposit.utxo) {
												if(transaction.state == 3) {
													transaction.state = 4
												}
												transaction.confirmations = deposit.utxo.confirmations
												transaction.btcTxHash = deposit.utxo.txHash
												transaction.btcTxVOut = deposit.utxo.vOut
											}
										})
					}

					transaction.state = 5

					let signature = await deposit.submit()
					transaction.renResponse = signature.renVMResponse;
					transaction.signature = signature.signature
				}
			},

			async mintThenSwap({ id, params, utxoAmount, renHash, renSignature }) {
				await this.adapterContract.methods.mintThenSwap(
					params.contractCalls[0].contractParams[0].value,
					params.contractCalls[0].contractParams[1].value,
					utxoAmount,
					renHash,
					renSignature,
				).send({
					from: this.default_account
				})

				this.transactions = this.transactions.filter(t => t.id != id)
			},

			async burn() {
				await this.adapterContract.methods.swapThenBurn(
					RenJS.utils.BTC.addressToHex(this.address),
					RenJS.utils.value(this.amount, "btc").sats().toNumber(),
					0
				).send({
					from: this.default_account
				})
			},

			async submit() {
				if(this.type == 0) this.mint()
				if(this.type == 1) this.burn()
			}
		}
	}
</script>

<style scoped>
	label[for='getbtc'] {
		margin-left: 1em;
	}
	.input {
		margin-top: 1em;
	}
	.input input {
		display: block;
	}
	#amount {
		width: 100px;
	}
	#address {
		width: 600px;
	}
	button {
		margin-top: 1em;
	}
	.tui-table {
		margin-top: 1em;
	}
</style>