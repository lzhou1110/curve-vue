<template>
	<div class='window white'>
		<div>
			<button class='simplebutton' @click='use3Box'>
				<span v-show='space === null'>Use 3Box Storage</span>
				<span v-show='space !== null'>3Box Storage loaded</span>
			</button>
		</div>
		<div>
			<input type='radio' id='getwbtc' :value='0' v-model='type'>
			<label for='getwbtc'>Get WBTC</label>

			<input type='radio' id='getbtc' :value='1' v-model='type'>
			<label for='getbtc'>Get BTC</label>
		</div>

		<div class='input'>
			<label for='amount'>{{ type == 0 ? 'BTC' : 'WBTC' }} amount</label>
			<input id='amount' type='text' v-model='amount' placeholder='Amount' @input = 'setAmount'>
		</div>

		<div class='input'>
			<label for='address'>{{ type == 1 ? 'BTC' : 'ETH' }} address</label>
			<input id='address' type='text' v-model='address' placeholder='Address'>
		</div>

		<div id='result'>
			<div>Exchange amount: {{ exchangeAmount }}</div>
			<div>Ren fees: {{ renFee }}</div>
			<div>BTC miner's fee: {{ this.minersFee / 1e8 }}</div>
			<div>Total received amount: {{ receivedAmount }}</div>
			<div>Exchange rate: {{ exchangeRate }}</div>
		</div>


		<button @click='submit'>Submit</button>

		<table class='tui-table'>
			<thead>
				<tr>
					<th>BTC Address</th>
					<th>Confirmations</th>
					<th>Status</th>
					<th>Progress</th>
				</tr>
			</thead>
			<tbody>
				<tr v-for='transaction in transactions'>
					<td>
						<span :class="{'loading line': !transaction.gatewayAddress }"></span>
						{{ transaction.gatewayAddress }}
					</td>
					<td>{{ transaction.confirmations }}</td>
					<td>
						<tx-state 
							:state='transaction.state' 
							:transaction='transaction'
							@mint='mintThenSwap'/>
					</td>
					<td>
						{{ transaction.state == 6 ? 100 : (transaction.state / 5 * 100 | 0) }}%
						<span :class="{'loading line': transaction.state != 6}"></span>
					</td>
				</tr>
			</tbody>
		</table>
	</div>	
</template>

<script>
	import Vue from 'vue'
	import { getters, allCurrencies, contract } from '../../contract'
	import RenSDK from '@renproject/ren'
	import BN from 'bignumber.js'
	import TxState from './TxState.vue'
	import * as helpers from '../../utils/helpers'
	import * as common from '../../utils/common'
	import allabis, { ERC20_abi } from '../../allabis'
	//import Box from '3box'

	const swap_address = '0x62869F49ea8b6c3EEdEcA8b8b1c6731090aD7A3D'
	const swap_abi = [{"name":"TokenExchange","inputs":[{"type":"address","name":"buyer","indexed":true},{"type":"int128","name":"sold_id","indexed":false},{"type":"uint256","name":"tokens_sold","indexed":false},{"type":"int128","name":"bought_id","indexed":false},{"type":"uint256","name":"tokens_bought","indexed":false}],"anonymous":false,"type":"event"},{"name":"TokenExchangeUnderlying","inputs":[{"type":"address","name":"buyer","indexed":true},{"type":"int128","name":"sold_id","indexed":false},{"type":"uint256","name":"tokens_sold","indexed":false},{"type":"int128","name":"bought_id","indexed":false},{"type":"uint256","name":"tokens_bought","indexed":false}],"anonymous":false,"type":"event"},{"name":"AddLiquidity","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256[2]","name":"token_amounts","indexed":false},{"type":"uint256[2]","name":"fees","indexed":false},{"type":"uint256","name":"invariant","indexed":false},{"type":"uint256","name":"token_supply","indexed":false}],"anonymous":false,"type":"event"},{"name":"RemoveLiquidity","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256[2]","name":"token_amounts","indexed":false},{"type":"uint256[2]","name":"fees","indexed":false},{"type":"uint256","name":"token_supply","indexed":false}],"anonymous":false,"type":"event"},{"name":"RemoveLiquidityImbalance","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256[2]","name":"token_amounts","indexed":false},{"type":"uint256[2]","name":"fees","indexed":false},{"type":"uint256","name":"invariant","indexed":false},{"type":"uint256","name":"token_supply","indexed":false}],"anonymous":false,"type":"event"},{"name":"CommitNewAdmin","inputs":[{"type":"uint256","name":"deadline","indexed":true,"unit":"sec"},{"type":"address","name":"admin","indexed":true}],"anonymous":false,"type":"event"},{"name":"NewAdmin","inputs":[{"type":"address","name":"admin","indexed":true}],"anonymous":false,"type":"event"},{"name":"CommitNewParameters","inputs":[{"type":"uint256","name":"deadline","indexed":true,"unit":"sec"},{"type":"uint256","name":"A","indexed":false},{"type":"uint256","name":"fee","indexed":false},{"type":"uint256","name":"admin_fee","indexed":false}],"anonymous":false,"type":"event"},{"name":"NewParameters","inputs":[{"type":"uint256","name":"A","indexed":false},{"type":"uint256","name":"fee","indexed":false},{"type":"uint256","name":"admin_fee","indexed":false}],"anonymous":false,"type":"event"},{"outputs":[],"inputs":[{"type":"address[2]","name":"_coins"},{"type":"address[2]","name":"_underlying_coins"},{"type":"address","name":"_pool_token"},{"type":"uint256","name":"_A"},{"type":"uint256","name":"_fee"}],"constant":false,"payable":false,"type":"constructor"},{"name":"get_virtual_price","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":1084167},{"name":"calc_token_amount","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"uint256[2]","name":"amounts"},{"type":"bool","name":"deposit"}],"constant":true,"payable":false,"type":"function","gas":4239939},{"name":"add_liquidity","outputs":[],"inputs":[{"type":"uint256[2]","name":"amounts"},{"type":"uint256","name":"min_mint_amount"}],"constant":false,"payable":false,"type":"function","gas":6479997},{"name":"get_dy","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"}],"constant":true,"payable":false,"type":"function","gas":2543681},{"name":"get_dy_underlying","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"}],"constant":true,"payable":false,"type":"function","gas":2543476},{"name":"exchange","outputs":[],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"},{"type":"uint256","name":"min_dy"}],"constant":false,"payable":false,"type":"function","gas":5184453},{"name":"exchange_underlying","outputs":[],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"},{"type":"uint256","name":"min_dy"}],"constant":false,"payable":false,"type":"function","gas":5200697},{"name":"remove_liquidity","outputs":[],"inputs":[{"type":"uint256","name":"_amount"},{"type":"uint256[2]","name":"min_amounts"}],"constant":false,"payable":false,"type":"function","gas":153838},{"name":"remove_liquidity_imbalance","outputs":[],"inputs":[{"type":"uint256[2]","name":"amounts"},{"type":"uint256","name":"max_burn_amount"}],"constant":false,"payable":false,"type":"function","gas":6479648},{"name":"commit_new_parameters","outputs":[],"inputs":[{"type":"uint256","name":"amplification"},{"type":"uint256","name":"new_fee"},{"type":"uint256","name":"new_admin_fee"}],"constant":false,"payable":false,"type":"function","gas":146045},{"name":"apply_new_parameters","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":133452},{"name":"revert_new_parameters","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":21775},{"name":"commit_transfer_ownership","outputs":[],"inputs":[{"type":"address","name":"_owner"}],"constant":false,"payable":false,"type":"function","gas":74452},{"name":"apply_transfer_ownership","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":60508},{"name":"revert_transfer_ownership","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":21865},{"name":"withdraw_admin_fees","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":12771},{"name":"kill_me","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":37818},{"name":"unkill_me","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":21955},{"name":"coins","outputs":[{"type":"address","name":""}],"inputs":[{"type":"int128","name":"arg0"}],"constant":true,"payable":false,"type":"function","gas":2130},{"name":"underlying_coins","outputs":[{"type":"address","name":""}],"inputs":[{"type":"int128","name":"arg0"}],"constant":true,"payable":false,"type":"function","gas":2160},{"name":"balances","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"int128","name":"arg0"}],"constant":true,"payable":false,"type":"function","gas":2190},{"name":"A","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2021},{"name":"fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2051},{"name":"admin_fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2081},{"name":"owner","outputs":[{"type":"address","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2111},{"name":"admin_actions_deadline","outputs":[{"type":"uint256","unit":"sec","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2141},{"name":"transfer_ownership_deadline","outputs":[{"type":"uint256","unit":"sec","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2171},{"name":"future_A","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2201},{"name":"future_fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2231},{"name":"future_admin_fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2261},{"name":"future_owner","outputs":[{"type":"address","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2291}]

	const adapterABI = [{"inputs":[{"internalType":"contract ICurveExchange","name":"_exchange","type":"address"},{"internalType":"contract IGatewayRegistry","name":"_registry","type":"address"},{"internalType":"contract IERC20","name":"_wbtc","type":"address"},{"internalType":"address","name":"_owner","type":"address"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"oldRelayHub","type":"address"},{"indexed":true,"internalType":"address","name":"newRelayHub","type":"address"}],"name":"RelayHubChanged","type":"event"},{"payable":true,"stateMutability":"payable","type":"fallback"},{"constant":true,"inputs":[{"internalType":"address","name":"","type":"address"},{"internalType":"address","name":"","type":"address"},{"internalType":"bytes","name":"","type":"bytes"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"bytes","name":"","type":"bytes"},{"internalType":"uint256","name":"","type":"uint256"}],"name":"acceptRelayedCall","outputs":[{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"bytes","name":"","type":"bytes"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"exchange","outputs":[{"internalType":"contract ICurveExchange","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"getHubAddr","outputs":[{"internalType":"address","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"internalType":"uint256","name":"_minWbtcAmount","type":"uint256"},{"internalType":"address payable","name":"_wbtcDestination","type":"address"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"bytes32","name":"_nHash","type":"bytes32"},{"internalType":"bytes","name":"_sig","type":"bytes"}],"name":"mintDontSwap","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"uint256","name":"_minWbtcAmount","type":"uint256"},{"internalType":"address payable","name":"_wbtcDestination","type":"address"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"bytes32","name":"_nHash","type":"bytes32"},{"internalType":"bytes","name":"_sig","type":"bytes"}],"name":"mintThenSwap","outputs":[{"internalType":"uint256","name":"wbtcBought","type":"uint256"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"context","type":"bytes"},{"internalType":"bool","name":"success","type":"bool"},{"internalType":"uint256","name":"actualCharge","type":"uint256"},{"internalType":"bytes32","name":"preRetVal","type":"bytes32"}],"name":"postRelayedCall","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"context","type":"bytes"}],"name":"preRelayedCall","outputs":[{"internalType":"bytes32","name":"","type":"bytes32"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"registry","outputs":[{"internalType":"contract IGatewayRegistry","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"relayHubVersion","outputs":[{"internalType":"string","name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"_btcDestination","type":"bytes"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"uint256","name":"_minRenbtcAmount","type":"uint256"}],"name":"swapThenBurn","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"}]

	const adapterAddress = '0x3973b2acdfac17171315e49ef19a0758b8b6f104'

	const wbtcAddress = '0x8Cc301A58c03FF01b83116FCa618560414eC2A97'

	const txObject = () => ({
		id: '',
		amount: '',
		toAmount: 0,
		address: '',
		params: '',
		gatewayAddress: '',
		confirmations: 0,
		// 0 - waiting for renVM gateway address, 1 - waiting for deposit on BTC address, 2 - got BTC transaction, waiting for confirmation
		// 3 - waiting for renVM to do it's magic and shift, 4 - got renBTC, now initiating swap, 5 - swap ready
		state: 0,
		btcTxHash: '',
		btcTxVOut: '',
		renResponse: '',
		signature: '',
		box: null,
		space: null,
	})

	export default {
		components: {
			TxState,
		},
		data: () => ({
			type: 0,
			amount: '0.0001',
			toInput: '0.00',
			address: '0x309b8Da6166eDaa47664b595041899325B859fBe',
			sdk: new RenSDK('testnet'),
			transactions: [],


			gatewayAddress: '',
			btcConfirms: 0,
			btcTxHash: '',
			btcTxVOut: '',

			renResponse: '',
			renSignature: '',
			utxoAmount: 0,
			minersFee: 5000,

			default_account: '',
			adapterContract: null,
			wbtccontract: null,

			confirmations: 0,
			// 1 - getting btc deposit address, 2 - waiting to confirm on btc network, 3 - 
			state: null,
			box: null,
			space: null,
			swap: null,
		}),
		computed: {
			exchangeAmount() {
				return BN(this.toInput).div(1e8).toFixed()
			},
			receivedAmount() {
				return BN(this.exchangeAmount).minus(this.totalFee).toFixed(8)
			},
			exchangeRate() {
				return BN(this.exchangeAmount).div(this.amount).toFixed(8)
			},
			totalFee() {
				let amount = BN(this.amount).times(1e8)
				return BN(this.minersFee).plus(amount.times(0.001)).div(1e8).toFixed()
			},
			renFee() {
				return BN(this.amount).times(1e8).times(0.001).div(1e8).toFixed()
			},
		},
		watch: {
			type(val) {
				this.amount = ''
				this.address = ''
				if(val == 0) this.address = this.default_account
			},
		},
		created() {
			this.$watch(() => contract.web3 && contract.default_account, val => {
				if(!val) return;
				this.mounted()
			})
		},
		mounted() {
			contract.web3 && contract.default_account && this.mounted()
		},
		methods: {
			async mounted() {

				this.swap = new contract.web3.eth.Contract(swap_abi, swap_address)
				this.adapterContract = new contract.web3.eth.Contract(adapterABI, adapterAddress)
				this.wbtccontract = new contract.web3.eth.Contract(ERC20_abi, this.wbtcAddress)
				this.address = this.default_account = contract.default_account
				this.setAmount()
				//resume only transactions submitted to btc network
				this.loadTransactions()
			},

			async setAmount() {
				let i = 0;
				let j = 1;
				if(this.type == 1) [i, j] = [j, i]
				console.log(+this.totalFee, "REN FEE")
				let amount = (BN(this.amount).minus(BN(this.totalFee))).times(1e8).toFixed(0, 1)
				console.log(i, j, amount, "I J AMOUNT")
				let get_dy = await this.swap.methods.get_dy(i, j, amount).call()
				this.toInput = get_dy;
			},

			async loadTransactions() {
				let items = await this.getAllItems()
				this.transactions = Object.values(await this.getAllItems()).reverse()
				await Promise.all(this.transactions.filter(t => t.btcTxHash).map(t=>this.sendMint(t)))
			},

			async use3Box() {
				// if(this.box !== null) return;(1e8-500)*0.999
				// this.box = await Box.openBox(contract.default_account, contract.web3.currentProvider)
				// this.space = await this.box.openSpace('curvebtc')
				// await this.space.syncDone
				// this.loadTransactions();
			},

			async setItem(key, item) {
				if(this.space !== null) {
					return await this.space.private.set(key, JSON.stringify(item))
				}
				return localStorage.setItem(key, JSON.stringify(item))
			},

			async getItem(key) {
				if(this.space !== null) return await this.space.private.get(key)
				return localStorage.getItem(key)
			},

			async getAllItems() {
				let storage = localStorage
				if(this.space !== null) {
					storage = await this.space.private.all();
				}
				return Object.keys(storage).filter(key => key.startsWith('curvebtc_')).map(k => JSON.parse(storage[k]))
			},

			upsertTx(transaction) {
				let key = 'curvebtc_' + transaction.id
				transaction.web3Provider = null;
				if(transaction.params && transaction.params.web3Provider)
					transaction.params.web3Provider = null;
				transaction.box = null;
				transaction.space = null
				this.setItem(key, transaction)
			},

			async mint() {
				let mint = await this.initMint();
				let transaction = this.transactions[0]
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
			                value: BN(this.toInput).times(0.99).toFixed(0, 1),
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
					transfer.toInput = this.toInput;
					let tx = {...txObject(), ...transfer}
					this.upsertTx(tx)
					this.transactions.unshift(tx)
				}

				return mint;
			},

			async sendMint(transfer, mint) {

				let transaction = this.transactions.find(t => t.id == transfer.id)
				//transaction is ready to be sent to eth network
				if(transaction.renResponse && transaction.signature) {
					transaction.state = 6
					transaction.confirmations = 'Confirmed'
					this.upsertTx(transaction)
					//await this.mintThenSwap(transfer)
				}
				else {
					mint = mint || await this.initMint(transfer);

					//transaction initated, but didn't get an address, so updating
					if(!transaction.params) {
						transaction.params = mint.params;
						transaction.gatewayAddress = await mint.gatewayAddress()
						transaction.state = 1
						this.upsertTx(transaction)
					}

					let deposit
					//transaction was submitted to btc network
					if(transaction.btcTxHash && String(transaction.btcTxVOut) !== 'undefined') {
						deposit = await mint.wait(this.confirmations, {
							txHash: transaction.btcTxHash,
							vOut: transaction.btcTxVOut,
						})
						.on('deposit', deposit => {
							console.log(deposit, "DEPOSIT EVENT submitted")
							if(deposit.utxo) {
								transaction.state = 3;
								transaction.confirmations = deposit.utxo.confirmations
								transaction.btcTxHash = deposit.utxo.txHash
								transaction.btcTxVOut = deposit.utxo.vOut
								this.upsertTx(transaction)
							}
						})
					}
					//trasaction not submitted to btc network
					else {
						deposit = await mint.wait(this.confirmations)
										.on('deposit', deposit => {
											console.log(deposit, "DEPOSIT EVENT")
											if(deposit.utxo) {
												if(transaction.state == 1) {
													transaction.state = 2
												}
												transaction.confirmations = deposit.utxo.confirmations
												transaction.btcTxHash = deposit.utxo.txHash
												transaction.btcTxVOut = deposit.utxo.vOut
												this.upsertTx(transaction)
											}
										})
					}

					transaction.state = 3

					let signature = await deposit.submit()
					transaction.state = 4
					transaction.renResponse = signature.renVMResponse;
					transaction.signature = signature.signature
					transaction.utxoAmount = transaction.renResponse.autogen.amount
					this.upsertTx(transaction)
					this.mintThenSwap(transaction)
				}
			},

			async mintThenSwap({ id, params, utxoAmount, renResponse, signature }) {
				let transaction = this.transactions.find(t => t.id == id);
				this.submitMint(transaction)
				// let min_amount = params.contractCalls[0].contractParams[0].value
				
				// let amount = BN(this.amount).times(1e8).minus(BN(this.renFee(0, this.amount))).toFixed(0, 1)
				// let get_dy = await this.swap.methods.get_dy(i, j, amount).call()

				// if(get_dy < min_amount) {
				// 	transaction.state = 100
				// }
				// else {
				// 	transaction.params.contractCalls[0].contractParams[1].value = min_amount
				// 	this.submitMint(transaction)
				// }
				
			},

			async submitMint({ id, params, utxoAmount, renResponse, signature }) {
				let transaction = this.transactions.find(t => t.id == id);

				await this.adapterContract.methods.mintThenSwap(
					params.contractCalls[0].contractParams[0].value,
					params.contractCalls[0].contractParams[1].value,
					utxoAmount,
					renResponse.autogen.nhash,
					signature,
				).send({
					from: this.default_account
				})

				transaction.state = 5
				this.upsertTx(transaction)

				this.transactions = this.transactions.filter(t => t.id != id)
			},

			async burn() {
				await common.approveAmount(this.wbtccontract, 
					RenJS.utils.value(this.amount, "btc").sats().toNumber(), 
					this.default_account, this.wbtcAddress)
				await this.adapterContract.methods.swapThenBurn(
					RenJS.utils.BTC.addressToHex(this.address),
					RenJS.utils.value(this.amount, "btc").sats().toNumber(),
					BN(this.toInput).times(0.99).toFixed(0, 1),
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
		width: 100%;
		margin-top: 1em;
	}
	.simplebutton {
		margin-bottom: 1em;
	}
	#result {
		margin-top: 1em;
	}
	#result > div {
		margin-top: 0.3em;
	}
</style>