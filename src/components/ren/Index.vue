<template>
	<div class='window white'>
		<div>
			<button class='simplebutton' @click='use3Box'>
				<span v-show='space === null'>Use 3Box Storage</span>
				<span v-show='space !== null'>3Box Storage loaded</span>
			</button>
		</div>
		<div class='exchange'>

			<div class='input'>
				<label for='address'>{{ from_currency == 1 ? 'BTC' : 'ETH' }} address</label>
				<input id='address' type='text' v-model='address' placeholder='Address'>
			</div>
            <div class='exchangefields'>
                <fieldset class='item'>
                    <legend>From:</legend>
                    <div class='maxbalance' @click='set_max_balance'>Max: <span>{{maxBalanceText}}</span> </div>
                    <ul>
                        <li>
                            <input type="text" id="from_currency" :disabled='disabled' name="from_currency" value='0.00'
                            :style = "{backgroundColor: fromBgColor}"
                            @input='set_to_amount'
                            v-model='fromInput'>
                        </li>
                        <li v-for='(currency, i) in Object.keys(currencies)'>
                            <input type="radio" :id="'from_cur_'+i" name="from_cur" :value='i' v-model='from_currency'>
                            <label :for="'from_cur_'+i">
                                <span>{{currency | capitalize}}</span>
                            </label>
                        </label>
                        </li>
                        Amount after renVM fees: {{actualFromAmount}}
                    </ul>
                </fieldset>
                <fieldset class='item iconcontainer' @click='swapInputs'>
                    <img src='@/assets/exchange-alt-solid.svg' id='exchangeicon'/>
                </fieldset>
                <fieldset class='item'>
                    <legend>To:</legend>
                    <div class='maxbalance2'>Max: <span></span> </div>
                    <ul>
                        <li>
                            <input type="text" 
                            id="to_currency" 
                            name="to_currency" 
                            value="0.00" 
                            disabled
                            :style = "{backgroundColor: bgColor}"
                            v-model='toInput'>
<!--                             <p class='actualvalue' v-show='swapwrapped'>
                                ≈ {{actualToValue}} {{Object.keys(currencies)[this.to_currency] | capitalize}}
                            </p> -->
                        </li>
                        <li v-for='(currency, i) in Object.keys(currencies)'>
                            <input type="radio" :id="'to_cur_'+i" name="to_cur" :value='i' v-model='to_currency'>
                            <label :for="'to_cur_'+i">
                                <span>{{currency | capitalize}}</span>
                            </label>
                        </label>
                        </li>
                    </ul>
                </fieldset>
            </div>
	     	<ul>
	            <li>
	                <input id="inf-approval" type="checkbox" name="inf-approval" v-model='inf_approval'>
	                <label for="inf-approval">Infinite approval - trust this contract forever
	                    <span class='tooltip'>[?]
	                        <span class='tooltiptext long'>
	                            Preapprove the contract to to be able to spend any amount of your coins. You will not need to approve again.
	                        </span>
	                    </span>
	                </label>
	            </li>
	        </ul>
            <p class='exchange-rate'>Exchange rate (including fees and renVM fee): 
            	<span id="exchange-rate">{{ exchangeRate && exchangeRate.toFixed(4) }}</span>
            </p>
            <p class='exchange-rate'>Exchange rate (including fees): 
            	<span id="exchange-rate">{{ exchangeRateOriginal && exchangeRateOriginal.toFixed(4) }}</span>
            </p>
            <p class='simple-error' v-show='from_currency == 1 && lessThanMinOrder'>
            	Minimum order size is {{ minOrderSize / 1e8 }} 
            </p>
        </div>

		<div>
			<input type='radio' id='getwbtc' :value='0' v-model='type'>
			<label for='getwbtc'>Get WBTC</label>

			<input type='radio' id='getbtc' :value='1' v-model='type'>
			<label for='getbtc'>Get BTC</label>
		</div>

		<!-- <div class='input'>
			<label for='amount'>{{ type == 0 ? 'BTC' : 'WBTC' }} amount</label>
			<input id='amount' type='text' v-model='amount' placeholder='Amount' @input = 'setAmount'>
		</div>

 -->
		<div id='result'>
			<div>Exchange rate: {{ exchangeRate }}</div>
			<div>Ren fees: {{ renFee }}</div>
			<div>BTC miner's fee: {{ this.minersFee / 1e8 }}</div>
			<div>Total received amount: {{ (toInput / 1e8).toFixed(8) }}</div>
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

	const swap_address = '0x8474c1236F0Bc23830A23a41aBB81B2764bA9f4F'
	const swap_abi = [{"name":"TokenExchange","inputs":[{"type":"address","name":"buyer","indexed":true},{"type":"int128","name":"sold_id","indexed":false},{"type":"uint256","name":"tokens_sold","indexed":false},{"type":"int128","name":"bought_id","indexed":false},{"type":"uint256","name":"tokens_bought","indexed":false}],"anonymous":false,"type":"event"},{"name":"AddLiquidity","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256[2]","name":"token_amounts","indexed":false},{"type":"uint256[2]","name":"fees","indexed":false},{"type":"uint256","name":"invariant","indexed":false},{"type":"uint256","name":"token_supply","indexed":false}],"anonymous":false,"type":"event"},{"name":"RemoveLiquidity","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256[2]","name":"token_amounts","indexed":false},{"type":"uint256[2]","name":"fees","indexed":false},{"type":"uint256","name":"token_supply","indexed":false}],"anonymous":false,"type":"event"},{"name":"RemoveLiquidityOne","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256","name":"token_amount","indexed":false},{"type":"uint256","name":"coin_amount","indexed":false}],"anonymous":false,"type":"event"},{"name":"RemoveLiquidityImbalance","inputs":[{"type":"address","name":"provider","indexed":true},{"type":"uint256[2]","name":"token_amounts","indexed":false},{"type":"uint256[2]","name":"fees","indexed":false},{"type":"uint256","name":"invariant","indexed":false},{"type":"uint256","name":"token_supply","indexed":false}],"anonymous":false,"type":"event"},{"name":"CommitNewAdmin","inputs":[{"type":"uint256","name":"deadline","indexed":true,"unit":"sec"},{"type":"address","name":"admin","indexed":true}],"anonymous":false,"type":"event"},{"name":"NewAdmin","inputs":[{"type":"address","name":"admin","indexed":true}],"anonymous":false,"type":"event"},{"name":"CommitNewFee","inputs":[{"type":"uint256","name":"deadline","indexed":true,"unit":"sec"},{"type":"uint256","name":"fee","indexed":false},{"type":"uint256","name":"admin_fee","indexed":false}],"anonymous":false,"type":"event"},{"name":"NewFee","inputs":[{"type":"uint256","name":"fee","indexed":false},{"type":"uint256","name":"admin_fee","indexed":false}],"anonymous":false,"type":"event"},{"name":"RampA","inputs":[{"type":"uint256","name":"old_A","indexed":false},{"type":"uint256","name":"new_A","indexed":false},{"type":"uint256","name":"initial_time","indexed":false,"unit":"sec"},{"type":"uint256","name":"future_time","indexed":false,"unit":"sec"}],"anonymous":false,"type":"event"},{"name":"StopRampA","inputs":[{"type":"uint256","name":"A","indexed":false},{"type":"uint256","name":"t","indexed":false,"unit":"sec"}],"anonymous":false,"type":"event"},{"outputs":[],"inputs":[{"type":"address[2]","name":"_coins"},{"type":"address","name":"_pool_token"},{"type":"uint256","name":"_A"},{"type":"uint256","name":"_fee"}],"constant":false,"payable":false,"type":"constructor"},{"name":"A","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":5227},{"name":"get_virtual_price","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":967716},{"name":"calc_token_amount","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"uint256[2]","name":"amounts"},{"type":"bool","name":"deposit"}],"constant":true,"payable":false,"type":"function","gas":3810860},{"name":"add_liquidity","outputs":[],"inputs":[{"type":"uint256[2]","name":"amounts"},{"type":"uint256","name":"min_mint_amount"}],"constant":false,"payable":false,"type":"function","gas":5858197},{"name":"get_dy","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"}],"constant":true,"payable":false,"type":"function","gas":2327087},{"name":"get_dy_underlying","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"}],"constant":true,"payable":false,"type":"function","gas":2326882},{"name":"exchange","outputs":[],"inputs":[{"type":"int128","name":"i"},{"type":"int128","name":"j"},{"type":"uint256","name":"dx"},{"type":"uint256","name":"min_dy"}],"constant":false,"payable":false,"type":"function","gas":4783971},{"name":"remove_liquidity","outputs":[],"inputs":[{"type":"uint256","name":"_amount"},{"type":"uint256[2]","name":"min_amounts"}],"constant":false,"payable":false,"type":"function","gas":153460},{"name":"remove_liquidity_imbalance","outputs":[],"inputs":[{"type":"uint256[2]","name":"amounts"},{"type":"uint256","name":"max_burn_amount"}],"constant":false,"payable":false,"type":"function","gas":5857985},{"name":"calc_withdraw_one_coin","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"uint256","name":"_token_amount"},{"type":"int128","name":"i"}],"constant":true,"payable":false,"type":"function","gas":10920},{"name":"remove_liquidity_one_coin","outputs":[],"inputs":[{"type":"uint256","name":"_token_amount"},{"type":"int128","name":"i"},{"type":"uint256","name":"min_amount"}],"constant":false,"payable":false,"type":"function","gas":3677379},{"name":"ramp_A","outputs":[],"inputs":[{"type":"uint256","name":"_future_A"},{"type":"uint256","unit":"sec","name":"_future_time"}],"constant":false,"payable":false,"type":"function","gas":151937},{"name":"stop_ramp_A","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":148697},{"name":"commit_new_fee","outputs":[],"inputs":[{"type":"uint256","name":"new_fee"},{"type":"uint256","name":"new_admin_fee"}],"constant":false,"payable":false,"type":"function","gas":110521},{"name":"apply_new_fee","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":97220},{"name":"revert_new_parameters","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":21955},{"name":"commit_transfer_ownership","outputs":[],"inputs":[{"type":"address","name":"_owner"}],"constant":false,"payable":false,"type":"function","gas":74632},{"name":"apply_transfer_ownership","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":60688},{"name":"revert_transfer_ownership","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":22045},{"name":"withdraw_admin_fees","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":12424},{"name":"kill_me","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":37998},{"name":"unkill_me","outputs":[],"inputs":[],"constant":false,"payable":false,"type":"function","gas":22135},{"name":"coins","outputs":[{"type":"address","name":""}],"inputs":[{"type":"int128","name":"arg0"}],"constant":true,"payable":false,"type":"function","gas":2310},{"name":"balances","outputs":[{"type":"uint256","name":""}],"inputs":[{"type":"int128","name":"arg0"}],"constant":true,"payable":false,"type":"function","gas":2340},{"name":"fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2171},{"name":"admin_fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2201},{"name":"owner","outputs":[{"type":"address","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2231},{"name":"initial_A","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2261},{"name":"future_A","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2291},{"name":"initial_A_time","outputs":[{"type":"uint256","unit":"sec","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2321},{"name":"future_A_time","outputs":[{"type":"uint256","unit":"sec","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2351},{"name":"admin_actions_deadline","outputs":[{"type":"uint256","unit":"sec","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2381},{"name":"transfer_ownership_deadline","outputs":[{"type":"uint256","unit":"sec","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2411},{"name":"future_fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2441},{"name":"future_admin_fee","outputs":[{"type":"uint256","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2471},{"name":"future_owner","outputs":[{"type":"address","name":""}],"inputs":[],"constant":true,"payable":false,"type":"function","gas":2501}]

	const adapterABI = [{"inputs":[{"internalType":"contract ICurveExchange","name":"_exchange","type":"address"},{"internalType":"contract IGatewayRegistry","name":"_registry","type":"address"},{"internalType":"contract IERC20","name":"_wbtc","type":"address"},{"internalType":"address","name":"_owner","type":"address"}],"payable":false,"stateMutability":"nonpayable","type":"constructor"},{"anonymous":false,"inputs":[{"indexed":true,"internalType":"address","name":"oldRelayHub","type":"address"},{"indexed":true,"internalType":"address","name":"newRelayHub","type":"address"}],"name":"RelayHubChanged","type":"event"},{"payable":true,"stateMutability":"payable","type":"fallback"},{"constant":true,"inputs":[{"internalType":"address","name":"","type":"address"},{"internalType":"address","name":"","type":"address"},{"internalType":"bytes","name":"","type":"bytes"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"bytes","name":"","type":"bytes"},{"internalType":"uint256","name":"","type":"uint256"}],"name":"acceptRelayedCall","outputs":[{"internalType":"uint256","name":"","type":"uint256"},{"internalType":"bytes","name":"","type":"bytes"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"exchange","outputs":[{"internalType":"contract ICurveExchange","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"getHubAddr","outputs":[{"internalType":"address","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"internalType":"uint256","name":"_minWbtcAmount","type":"uint256"},{"internalType":"address payable","name":"_wbtcDestination","type":"address"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"bytes32","name":"_nHash","type":"bytes32"},{"internalType":"bytes","name":"_sig","type":"bytes"}],"name":"mintThenSwap","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"context","type":"bytes"},{"internalType":"bool","name":"success","type":"bool"},{"internalType":"uint256","name":"actualCharge","type":"uint256"},{"internalType":"bytes32","name":"preRetVal","type":"bytes32"}],"name":"postRelayedCall","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"context","type":"bytes"}],"name":"preRelayedCall","outputs":[{"internalType":"bytes32","name":"","type":"bytes32"}],"payable":false,"stateMutability":"nonpayable","type":"function"},{"constant":true,"inputs":[],"name":"registry","outputs":[{"internalType":"contract IGatewayRegistry","name":"","type":"address"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":true,"inputs":[],"name":"relayHubVersion","outputs":[{"internalType":"string","name":"","type":"string"}],"payable":false,"stateMutability":"view","type":"function"},{"constant":false,"inputs":[{"internalType":"bytes","name":"_btcDestination","type":"bytes"},{"internalType":"uint256","name":"_amount","type":"uint256"},{"internalType":"uint256","name":"_minRenbtcAmount","type":"uint256"}],"name":"swapThenBurn","outputs":[],"payable":false,"stateMutability":"nonpayable","type":"function"}]

	const adapterAddress = '0x2407750D9E57990559b6be9C17Ea200Cbdf17b93'

	const wbtcAddress = '0x2260FAC5E5542a773Aa44fBCfeDf7C193bc2C599'

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
			amount: '0.001',
			toInput: '0.00',
			address: '',
			sdk: new RenSDK('testnet'),
			transactions: [],


			gatewayAddress: '',
			btcConfirms: 0,
			btcTxHash: '',
			btcTxVOut: '',

			renResponse: '',
			renSignature: '',
			utxoAmount: 0,
			minersFee: 35000,

			default_account: '',
			adapterContract: null,
			wbtccontract: null,

			confirmations: 0,
			// 1 - getting btc deposit address, 2 - waiting to confirm on btc network, 3 - 
			state: null,
			box: null,
			space: null,
			swap: null,


			maxBalance: 0,
			disabled: false,
			fromInput: '0.001',
			from_currency: 1,
			to_currency: 0,
			get_dy_original: '',
			fromBgColor: '',
			bgColor: '',
			swapwrapped: false,
			currencies: {
				btc: 'BTC',
				wbtc: 'WBTC',
			},
			inf_approval: false,
			promise: helpers.makeCancelable(Promise.resolve()),
		}),
		computed: {
			exchangeAmount() {
				return BN(this.amount).div(1e8).toFixed()
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
			maxBalanceText() {
				if(this.from_currency == 0) return 'N/A'
				return BN(this.maxBalance).div(1e8).toFixed(8)
			},
			exchangeRate() {
            	return this.toInput / this.fromInput
            },
            exchangeRateOriginal() {
            	return this.get_dy_original / 1e8  / this.fromInput
            },
            renVMfee() {
            	return BN(this.fromInput).times(0.001).div(1e8).toFixed(0)
            },
            actualFromAmount() {
            	if(this.from_currency == 0)
            		return BN(this.fromInput).times(0.999).toFixed(8)
            	return BN(this.fromInput).times(1e8).times(0.999).minus(this.minersFee).toFixed(0)
            },
            actualToAmount() {
            	return BN(this.fromInput).times()
            },
            minOrderSize() {
            	return 35001
            },
            lessThanMinOrder() {
            	if(this.from_currency == 0) return false
            	if(this.from_currency == 1 && BN(this.fromInput).times(1e8).lt(BN(this.minOrderSize))) return true	
            },
		},
		watch: {
			type(val) {
				this.address = ''
				if(val == 0) this.address = this.default_account
			},
			from_currency(val, oldval) {
                if(val == this.to_currency) {
                    this.to_currency = oldval;
                }
                
                this.from_cur_handler()
            },
            async to_currency(val, oldval) {
            	if(val == this.from_currency) {
            		if (this.to_currency == 0) {
                        this.from_currency = 1;
                    } else {
                        this.from_currency = 0;
                    }
            	}
            	await this.from_cur_handler()
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
				this.wbtccontract = new contract.web3.eth.Contract(ERC20_abi, wbtcAddress)
				this.address = this.default_account = contract.default_account
				if(this.from_currency == 1) this.address = this.default_account
				console.log(contract.default_account, "DEFAULT ACCOUNT")
				console.log(await this.sdk.renVM.sendMessage('ren_queryFees', {}))
				this.from_cur_handler()
				//resume only transactions submitted to btc network
				this.loadTransactions()

				// //test wait for burn
				// const burn = await this.sdk.burnAndRelease({
				//     // Send BTC from the Ethereum blockchain to the Bitcoin blockchain.
				//     // This is the reverse of shitIn.
				//     sendToken: RenJS.Tokens.BTC.Eth2Btc,

				//     // The web3 provider to talk to Ethereum
				//     web3Provider: web3.currentProvider,

				//     // The transaction hash of our contract call
				//     ethereumTxHash: '0x22b007a000dfa003a49fa3bcdbfb73e796050e913d73826a9ad45a5b6fb7e0c6',
				// }).readFromEthereum();
				// console.log(burn)
				// console.log(await burn.queryTx())
				// let promiEvent = await burn.submit()
				// console.log(promiEvent, "PROMI EVENT")
				// //promiEvent.on('txHash', hash => console.log(hash)).on('status', status => console.log(status));


			},

			set_max_balance() {
				if(this.from_currency == 1) {
					this.fromInput = 0
					return;
				}
				this.fromInput = BN(this.maxBalance).div(1e8).toFixed(8)
			},

			set_from_amount() {

			},

			async set_to_amount() {
				// console.log(+this.totalFee, "REN FEE")
				// let amount = (BN(this.amount).minus(BN(this.totalFee))).times(1e8).toFixed(0, 1)
				// console.log(i, j, amount, "I J AMOUNT")
				let i = this.from_currency
				let j = this.to_currency
				let dx = BN(this.fromInput).times(1e8).toFixed(0,1)
				let original_dx = dx
				if(this.from_currency == 0) {
					dx = BN(this.fromInput).times(0.999).times(1e8).toFixed(0,1)
				}
				if(this.from_currency == 1) {
					dx = BN(this.fromInput).times(1e8).minus(this.minersFee).toFixed(0,1)
				}
				console.log(dx, "THE DX")
				let get_dys = [this.swap.methods.get_dy(i, j, original_dx).call(), this.swap.methods.get_dy(i, j, dx).call()]
				console.log(get_dys)
				this.promise.cancel()
				let promise = Promise.all(get_dys)
				this.promise = helpers.makeCancelable(promise)
				let [get_dy_original, get_dy] = await promise
				console.log(get_dy_original, get_dy, "THE DY")
				this.get_dy_original = get_dy_original
				this.toInput = BN(get_dy).div(1e8).toFixed(8);
			},

			swapInputs() {

			},

			async from_cur_handler() {
				this.address = ''
				if(this.from_currency == 0) this.address = this.default_account

                let currentAllowance = BN(await this.wbtccontract.methods.allowance(this.default_account, adapterAddress).call())
                let maxAllowance = contract.max_allowance.div(BN(2))
                if (currentAllowance.gt(maxAllowance))
                    this.inf_approval = true;
                else
                    this.inf_approval = false;

                await this.setMaxBalance();
                await this.set_to_amount();
            },

			async setMaxBalance() {
				console.log(this.address, "THE ADDRESS")
				let balance = await this.wbtccontract.methods.balanceOf(this.default_account).call()
				this.maxBalance = this.default_account ? balance : 0
				console.log(this.maxBalance)
			},

			async setAmount() {
				// console.log(+this.totalFee, "REN FEE")
				// let amount = (BN(this.amount).minus(BN(this.totalFee))).times(1e8).toFixed(0, 1)
				// console.log(i, j, amount, "I J AMOUNT")
				let i = this.from_currency
				let j = this.to_currency
				let get_dy = await this.swap.methods.get_dy(i, j, amount).call()
				console.log(get_dy, "THE DY")
				this.toInput = BN(get_dy).div(1e8).toFixed(8);
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
				console.log(BN(this.toInput).times(0.99).toFixed(0, 1), "MIN VALUE")
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
				let min_amount = params.contractCalls[0].contractParams[0].value
				
				let amount = BN(this.amount).times(1e8).minus(BN(this.renFee(0, this.amount))).toFixed(0, 1)
				let get_dy = await this.swap.methods.get_dy(i, j, amount).call()

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

				if(get_dy < min_amount) transaction.state = 7
				this.upsertTx(transaction)
				this.transactions = this.transactions.filter(t => t.id != id)
				
			},

			async burn() {
				await common.approveAmount(this.wbtccontract, 
					BN(this.amount).times(1e8), 
					this.default_account, adapterAddress)
				console.log(RenJS.utils.BTC.addressToHex(this.address),
					BN(this.amount).times(1e8),
					BN(this.toInput).times(0.99).toFixed(0, 1))
				await this.adapterContract.methods.swapThenBurn(
					RenJS.utils.BTC.addressToHex(this.address),
					BN(this.amount).times(1e8),
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
	.exchange-rate {
		text-align: left;
	}
</style>