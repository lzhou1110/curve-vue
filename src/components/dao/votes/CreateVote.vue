<template>
	<div class='window white'>
		<div id='modal' class='modal rootmodal' v-if='showRootModal' @click.self='hideRootModal'>

			<div class='modal-content window white'>
				<fieldset>
					<div class='legend2 hoverpointer' @click='hideRootModal'>
						[<span class='greentext'>■</span>]
					</div>
					<legend v-show='!executeVote'>Create a vote on {{ appName }}</legend>
					<legend v-show='executeVote'>Execute a vote on {{ appName }}</legend>
					<div class='content'>
						<div>
							<span> {{ voteDescription }} </span>
							<div>
								{{ textdescription }}
							</div>
							<p v-html = 'description'></p>
						</div>
						<hr>
						<p class='explanation' v-show='!executeVote'>
							This vote requires {{ getSupportText }}% acceptance and {{ getQuorumText }}% quorum to be passed
						</p>
						<p class='simple-error' v-show='!willSucceed'>
							The transaction may fail, you may not have the required permissions to make the transaction
						</p>
					</div>
					<button @click='createVote' v-show='!executeVote'>Create Vote</button>
					<button @click='createVote' v-show='executeVote'>Vote</button>
				</fieldset>
			</div>
		</div>

		<div id='modal' class='modal rootmodal' v-if='showDescriptionModal' @click.self='hideDescriptionModal'>
			<div class='modal-content window white'>
				<fieldset>
					<div class='legend2 hoverpointer' @click='hideDescriptionModal'>
						[<span class='greentext'>■</span>]
					</div>
					<legend>Add description</legend>
					<div class='content'>
						<label for='textdescription'>Description:</label>
						<textarea id='textdescription' v-model='textdescription'></textarea>
						<button @click='submitVote'>Create Vote <span class='loading line' v-show='loadingVote'></span></button>
					</div>
				</fieldset>
			</div>
		</div>

		<router-link to='/dao'>← Back</router-link>

		<div class='votetypes'>
			<button @click='type=0' :class="{'simplebutton': type==0}">Pool Vote</button>
			<button @click='type=1' :class="{'simplebutton': type==1}">Gauge Vote</button>
			<button @click='type=2' :class="{'simplebutton': type==2}">Emergency Member</button>
			<button @click='type=3' :class="{'simplebutton': type==3}">VotingEscrow</button>
			<button @click='type=4' :class="{'simplebutton': type==4}">PoolProxy</button>
			<button @click='type=5' :class="{'simplebutton': type==5}">Registry</button>
			<button @click='type=6' :class="{'simplebutton': type==6}">Vesting</button>
			<button @click='type=7' :class="{'simplebutton': type==7}">SmartWallet Checker</button>
		</div>

		<component :is='voteComponent' class='votecomponent' @showRootModal='emitShowRootModal' @makeCall='makeCall'></component>

		<gas-price></gas-price>

	</div>
</template>

<script>
	import { contract } from '../../../contract'
	import allabis from '../../../allabis'

	import * as daoabis from '../allabis'

	import { getVote, state, getters } from '../voteStore'

	import { helpers as voteHelpers, OWNERSHIP_APP_ADDRESS, PARAMETER_APP_ADDRESS, OWNERSHIP_AGENT, PARAMETER_AGENT } from '../voteStore'

	import * as radspec from 'radspec'

	import * as gasPriceStore from '../../common/gasPriceStore'
    import GasPrice from '../../common/GasPrice.vue'

	const radspecFormat = {
		userHelpers: {
			address: () => async (address, prefix) => {
				if(!prefix) prefix = ''
				return {
					type: 'string',
					value: '<br> ' + prefix + ' ' + voteHelpers.shortenAddress(address),
				}
			},
			param: () => async (param, name) => {
				if(!name) name = ''
				return {
					type: 'string',
					value: '<br> ' + name + ':' + param,
				}
			},
		}
	}

	import RootModal from '../common/RootModal'

	import { Fragment } from 'vue-fragment'

	import PoolVote from '../createvote/PoolVote'
	import GaugeVote from '../createvote/GaugeVote'
	import EmergencyVote from '../createvote/EmergencyVote'
	import VotingEscrowVote from '../createvote/VotingEscrowVote'
	import PoolProxyVote from '../createvote/PoolProxyVote'
	import RegistryVote from '../createvote/RegistryVote'
	import VestingVote from '../createvote/VestingVote'
	import SmartWalletChecker from '../createvote/SmartWalletChecker'

	let ownership_actions = ['unkill_me', 'commit_transfer_ownership', 'revert_transfer_ownership', 'set_aave_referral', 'donate_admin_fees']

	let parameter_actions = ['commit_new_parameters', 'revert_new_parameters', 'commit_new_fee', 'apply_new_fee', 'ramp_A', 'stop_ramp_A']

	import RootModalMixin from '../common/RootModalMixin'

	export default {
		components: {
			RootModal,
			Fragment,
			PoolVote,
			GaugeVote,
			EmergencyVote,
			VotingEscrowVote,
			PoolProxyVote,
			RegistryVote,
			VestingVote,
			GasPrice,
			SmartWalletChecker,
		},

		mixins: [RootModalMixin],

		data: () => ({
			poolProxy: null,
			votingEscrow: null,
			gaugeController: null,

			type: 0,

			description: '',

			call: {},

			showDescriptionModal: false,
			textdescription: '',

			loadingVote: false,

		}),

		async created() {
			this.$watch(() => contract.default_account && state.initialized, val => {
				if(val) this.mounted()
			}, {
				immediate: true,
			})
		},

		computed: {
			allPools() {
				return Object.keys(allabis)
						.filter(pool => pool != 'y' && pool != 'susd' && pool != 'tbtc')
						.map(pool => ({ pool: pool, address: allabis[pool].swap_address, abi: allabis[pool].swap_abi }))
			},
			doesRampUp() {
				if(!this.selectedPool)
					return false
				console.log(Object.values(this.selectedPool.abi).map(abi => abi.name), "ABI")
				return Object.values(this.selectedPool.abi).map(abi => abi.name).includes('ramp_A')
			},
			voteComponent() {
				if(this.type == 0) return 'PoolVote'
				if(this.type == 1) return 'GaugeVote'
				if(this.type == 2) return 'EmergencyVote'
				if(this.type == 3) return 'VotingEscrowVote'
				if(this.type == 4) return 'PoolProxyVote'
				if(this.type == 5) return 'RegistryVote'
				if(this.type == 6) return 'VestingVote'
				if(this.type == 7) return 'SmartWalletChecker'
				return 'PoolVote'
			},
			gasPrice() {
                return gasPriceStore.state.gasPrice
            },
            gasPriceWei() {
                return gasPriceStore.state.gasPriceWei
            },
		},

		methods: {
			async mounted() {
				this.selectedPool = this.allPools[0]

				this.poolProxy = new web3.eth.Contract(daoabis.poolproxy_abi, daoabis.poolproxy_address)
				this.votingEscrow = new web3.eth.Contract(daoabis.votingescrow_abi, daoabis.votingescrow_address)
				this.gaugeController = new web3.eth.Contract(daoabis.gaugecontroller_abi, daoabis.gaugecontroller_address)
			},
			emitShowRootModal() {
				this.showRootModal = true
			},
			async describeCall(method, params, calldata, abi, expression) {

				console.log(method, abi, calldata, expression, "RADSPEC")

				let desc = await radspec.evaluate(expression, {
					abi: [abi],
					transaction: {
						data: calldata,
					},
				}, radspecFormat)

				this.description = desc
				this.showRootModal = true
			},
			async hideDescriptionModal() {
				this.showDescriptionModal = false
				this.textdescription = ''
			},
			async submitVote() {
				this.loadingVote = true

				state.proposeLoading = method


				let ipfshash = await fetch('https://pushservice.curve.fi/pinipfs', {
					method: 'POST',
					headers: {
						'Content-type': 'application/json',
					},
					body: JSON.stringify({
						text: this.textdescription,
					})
				})

				ipfshash = await ipfshash.json()
				ipfshash = 'ipfs:' + ipfshash.ipfsHash

				let { abiname, method, params, contractAddress, agent, voteApp } = this.call

				let abi = daoabis[abiname+'_abi'].find(abi => abi.name == method)
				console.log(abiname, method)
				let natspeckey = Object.keys(daoabis[abiname+'_natspec'].methods).find(key => key.includes(method))
				console.log(natspeckey, "NATSPEC")
				let expression = daoabis[abiname+'_natspec'].methods[natspeckey].notice
				console.log(params, "PARAMS")
				console.log(abi, "ABI CALL")
				console.log(method)
				let call = web3.eth.abi.encodeFunctionCall(abi, params)
				console.log(call, "CALL")

				let agent_abi = daoabis.agent_abi.find(abi => abi.name == 'execute')
				let agentcall = web3.eth.abi.encodeFunctionCall(agent_abi, [contractAddress, 0, call])


				agent = agent.toLowerCase()
				voteApp = voteApp.toLowerCase()


				console.log(agent, "THE AGENT")
				console.log(contractAddress, call, "CONTRACT ADDRESS CALL DATA")
				console.log(agentcall, "THE AGENT CALL")

				// let agentcalls = [
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000008485273a22000000000000000000000000a2b47e3d5c44877cca798226b7b8118f9bfb7a5600000000000000000000000000000000000000000000000000000000000001c200000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	// '0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb7000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000064cfca0bdb00000000000000000000000052EA46506B9CC5Ef470C5bf89f17Dc28bB35D85C00000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000008485273a2200000000000000000000000045f783cce6b7ff23b2ab2d70e416cdb7d6055f5100000000000000000000000000000000000000000000000000000000000003e800000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000008485273a2200000000000000000000000079a8c46dea5ada233abaffd40f3a0a2b1e5a4f2700000000000000000000000000000000000000000000000000000000000001f400000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb700000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000060000000000000000000000000000000000000000000000000000000000000008485273a22000000000000000000000000a5407eae9ba41422680e2e00537571bcc53efbfd000000000000000000000000000000000000000000000000000000000000006400000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb7000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000064cfca0bdb00000000000000000000000006364f10b501e868329afbc005b3492902d6c76300000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb7000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000064cfca0bdb00000000000000000000000093054188d876f558f4a66b2ef1d97d16edf0895b00000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb7000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000064cfca0bdb0000000000000000000000007fc77b5c7614e1533320ea6ddc2eb61fa00a971400000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// 	'0xb61d27f60000000000000000000000006e8f6d1da6232d5e40b0b8758a0145d6c5123eb7000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000600000000000000000000000000000000000000000000000000000000000000064cfca0bdb0000000000000000000000004CA9b3063Ec5866A4B82E437059D2C43d1be596F00000000000000000000000000000000000000000000000000000000003d0900000000000000000000000000000000000000000000000000000000012a05f20000000000000000000000000000000000000000000000000000000000',
				// ]

				let encodeData = [{ to: agent, data: agentcall}]

				// encodeData = []
				// for(let agentcall of agentcalls) {
				// 	encodeData.push({ to: agent, data: agentcall })
				// }

				console.log(encodeData, "COMMIT ADMIN FEE ALL POOLS DATA")

				if(method == 'commit_smart_wallet_checker') {
					let applyabi = daoabis[abiname+'_abi'].find(abi => abi.name == 'apply_smart_wallet_checker')
					let applycall = web3.eth.abi.encodeFunctionCall(applyabi, [])
					let applyagentcall = web3.eth.abi.encodeFunctionCall(agent_abi, [contractAddress, 0, applycall])
					encodeData.push({ to: agent, data: applyagentcall })
				}

				let calldata = voteHelpers.encodeCallsScript(encodeData)


				let intent
				try {
					intent = await state.org.appIntent(voteApp, 'newVote(bytes,string,bool,bool)', [calldata, ipfshash, false, false])
				}
				catch(err) {
					console.error(err)
				}
				let paths = await intent.paths(contract.default_account)

				this.loadingVote = false
				this.showDescriptionModal = false
				state.transactionIntent = paths

				let desc = await radspec.evaluate(expression, {
					abi: [abi],
					transaction: {
						data: call,
					},
				}, radspecFormat)

				this.description = ''
				this.showRootModal = true

				state.proposeLoading = null


				console.log(paths, "THE PATHS")
			},
			async makeCall(abiname, method, params, contractAddress, agent, voteApp, ipfshash = '') {
				this.call = {
					abiname,
					method,
					params,
					contractAddress,
					agent,
					voteApp,
					ipfshash,
				}
				this.showDescriptionModal = true
			},
		},
	}
</script>

<style scoped>
	.votetypes button {
		margin-top: 1em;
		margin-right: 1em;
	}
	.votecomponent {
		margin-top: 1em;
	}
	.modal-content .content {
		text-align: left;
		color: black;
	}
</style>