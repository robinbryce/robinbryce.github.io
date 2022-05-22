# Problem

How to enable use of different wallet providers for web3 apps for test
automation using [playwright](https://playwright.dev/) and
[svelte-kit](https://kit.svelte.dev/).  And more generally, how to enable
end-to-end test specific behaviour for svelte-kit based web apps. 

# How

* Configure the localStorage used for the browser context either globally in
    playwright.config.js or in the options for the specific test
    Eg:
        "{\"ethprovider\":\"http://127.0.0.1:8545/\"}"
* Use svelte-local-storage-store to make the variable default off and be
  available as a $var in .svelte templates
* Have test specific logic in the .svelte routes, pages or components that
  switch on the storage variable
* Run `npx hardhat node` from any project which has hardhat installed
* The node runs at 'http://127.0.0.0:8545/'

Using playwright, it is possible to configure the localstorage for specific
suites of tests and also individual tests. Giving quite a lot of control over
provider interactions in tests. For the details
[see](https://playwright.dev/docs/api/class-testoptions)

The config & code below illustrates the use of ethers JsonRpcProvider. Other
providers would require different configuration and setup, but the trick for
switching the behaviour on browser local storage remains the same.

# Ingredients

A good general article on if/why/how much of this kind of testing to do for web3
can be found
[here](https://mirror.xyz/jjpa.eth/Nt6oojn6WcnmDzV97CL1XFGXkNkyWekAGaCEO5SYpXE)

For end to end testing to be representative for a web3 app it is necessary to
deal with the interactions between the app and the wallet provider. Making this
possible involves quite a few moving parts. The approach described above
required all of the following:


[svelte-kit](https://kit.svelte.dev/) is an opinionated way to use svelte.
svelte avoids the shadow dom and the typical framework run time footprint by
pre-compiling a minimal and optimised javascript (and css etc) bundle.

[playwright](https://playwright.dev/) is an end to end testing framework for
javascript apps that supports a powerful model for browser context separation
and reuse.

[svelte-local-storage-store](https://www.npmjs.com/package/svelte-local-storage-store) gives svelte based web apps a [svelte/store](https://svelte.dev/docs#run-time-svelte-store) integration with browser local storage
[svelte-ethers-store](https://www.npmjs.com/package/svelte-ethers-store/v/0.0.2) gives svelte based web apps a clean way to subscribe to provider changes - network change, wallet chainge and so on

[hardhat](https://hardhat.org/) is a popular integration testing framework for
web3 apps. Amongst many other facilities, it supports running

[hardhat-ganache](https://hardhat.org/guides/ganache-tests.html) is a CI/CD friendly
ethereum node.

[ethers.js](https://docs.ethers.io/v5/) is a popular client library for
interacting with smart contracts from javascript using the defacto standard
JSON-RPC interfaces offered by most ethereum compatible blockchain platforms. It
makes it particularly easy to work with different kinds of wallet
[providers](https://docs.ethers.io/v5/api/providers/)

# Examples & Snippets


## playwrite.config.js

```const config = {
	testDir: './tests/playwright',
	use: {
		browserName: "firefox",
		headless: true,
		storageState: {
			origins: [
				{
					origin: "http://localhost:3000",
					localStorage: [
						{
						name: "playwright",
						value: '{"ethprovider":"http://127.0.0.1:8545/"}'
						}
					]
				}
			]
		},
	},
	webServer: {
		command: 'npm run build && npm run preview',
		port: 3000,
		timeout: 120000
	}
};
```

## svelte-local-storage-store

```
import { writable } from 'svelte-local-storage-store';

export const playwright = writable('playwright', "null");
```

## provider creation with svelte-ethers-store

```
<script>
    import { onMount } from 'svelte';
    import { get as storeGet } from 'svelte/store';
    import { defaultEvmStores } from 'svelte-ethers-store';
    import { contracts, connected, chainId, signer, signerAddress, provider} from 'svelte-ethers-store';

    onMount(async () => {
        if ($playwright) {
            const ethprovider = JSON.parse(storeGet(playwright))?.ethprovider;
            await defaultEvmStores.setProvider(ethprovider).catch(console.error);
        } else {
            await defaultEvmStores.setProvider();
            const [signer] = await $provider.send("eth_requestAccounts");
            console.log("signer: ", JSON.stringify(signer));
        }

        // window.ethereum.pollingInterval = 1500;
        console.log("pollingInterval", $provider.pollingInterval);
        await defaultEvmStores.attachContract('Arena', arenaAddress, abi);
    });
</script>
```
