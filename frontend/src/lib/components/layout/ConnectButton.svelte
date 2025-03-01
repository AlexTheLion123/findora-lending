<script lang="ts" context="module">
	import MetaMaskOnboarding from '@metamask/onboarding';

	let isInit = false;
	let button: HTMLButtonElement; // html selector
	let accounts: string[] = [];
	let isWrongNetwork = false;

	let onboarding: MetaMaskOnboarding;

	declare global {
		interface Window {
			ethereum: any;
		}
	}

	declare let ethereum: any;
</script>

<script lang="ts">
	// TODO use @metamask/detect-provider to detect mobile properly as well

	import { onMount } from 'svelte/internal';
	import { isConnected, currentAccount, chainId, buttonText } from '$lib/stores';

	function initialize() {
		onboarding = new MetaMaskOnboarding();
		updateButton();

		// listen to relevant events
		if (MetaMaskOnboarding.isMetaMaskInstalled()) {
			ethereum.on('chainChanged', (_chainId: string) => {
				handleChainChanged(_chainId);
				updateButton();
			}); // listen to chain changes

			ethereum.on('accountsChanged', (newAccounts: string[]) => {
				accounts = newAccounts;
				updateButton();
			});
		}
	}

	function isMetaMaskInstalled(): boolean {
		const { ethereum } = window;
		return ethereum && ethereum.isMetaMask;
	}

	async function updateButton() {
		if (!MetaMaskOnboarding.isMetaMaskInstalled()) {
			// Metamask is not installed
			
			setButtonText('Click here to install metmask', false);
			
			button.onclick = () => {
				setButtonText('Onboarding in progress', true);
				onboarding.startOnboarding();
			};
		} else {
			// Metamask is installed, and therefore ethereum exists
			
			if (!isInit) {
				// If first time getting here, set chainId store
				$chainId = await getChain();
				isInit = true;
			}
			
			if (!checkNetwork($chainId)) {
				// If wrong chain, prevent connection attempt
				setButtonText('Wrong network', true);
				return;
			}
			
			if (accounts && accounts.length > 0) {
				// accounts have been retrieved

				setButtonText(accounts[0], true);

				onboarding.stopOnboarding();
			} else {
				// no accounts retrieved yet

				setButtonText('Connect Wallet', false);
				button.onclick = onClickConnect;
			}
		}
		accounts[0] ? updateStores(accounts[0]) : updateStores(undefined);
	}

	async function onClickConnect() {
		await ethereum.request({ method: 'eth_requestAccounts' }).then((_accounts: string[]) => {
			accounts = _accounts;
			updateButton();
		});
	}

	async function getChain() {
		return ethereum.request({ method: 'eth_chainId' }).catch(console.log);
	}


	function updateStores(_curAccount: string | undefined) {
		if (_curAccount) {
			$isConnected = true;
			$currentAccount = _curAccount;
		} else {
			$isConnected = false;
			// @ts-ignore
			$currentAccount = null;
		}
	}

	function handleChainChanged(_chainId: string) {
		$chainId = _chainId;
		return checkNetwork(_chainId);
	}

	function checkNetwork(_chainId: string | null) {
		if (_chainId !== '0x1' && _chainId !== '0x539') { // TODO remove dev chainId
			// is not ethereum mainnet
			// not ethereum
			alert('We only support ethereum mainnet, please change the network');
			setButtonText('Wrong network', true);
			isWrongNetwork = true;
			return false;
		}

		// is ethereum
		isWrongNetwork = false;
		changeSymbol(); // TODO change symbol to the valid network
		return true;
	}

	function changeSymbol() {
		// TODO change symbol or removed, think handled in other components
	}

	function setButtonText(text: string, isDisabled: boolean) {
		$buttonText = text;
		button.disabled = isDisabled;
	}

	onMount(initialize);
</script>

<button
	id="connect"
	bind:this={button}
	class:wrongNetwork={isWrongNetwork === true}
	on:click={() => updateButton()}
>
	{$buttonText}
</button>

<style lang="scss">
	$btn-color: rgba(105, 105, 206, 0.3);
	$wrong-network-btn-color: rgba(255, 0, 0, 0.3);
	#connect {
		background: $btn-color;
		overflow: hidden;
		width: 200px;
		padding: 15px 40px;
		margin: 0 50px;
		border-radius: 7px;
		border: 1px solid rgba(255, 255, 255, 0.2);

		&:hover {
			background: lighten($btn-color, 10%);
			cursor: pointer;
		}
	}

	.wrongNetwork {
		background: $wrong-network-btn-color !important;
	}
</style>
