Start Ganache on the default port `8545`

```bash
# Progress blocks every 1 second, and store the state so we don't need to redeploy
ganache-cli -b 1 --db ~/.ganache --mnemonic sweet
```

> Warning: Disk-space is continuously consumed using the "-b" automining and "--db" save database flags. Be sure to stop it when not in use. If it gets to large, you can always delete it with `rm -rf ~/.ganache`.

Install the contracts repo, build and deploy the contracts

```bash
npm i
npm run build
npx truffle --network ganache exec scripts/deploy-framework.js
```

Find the`Resolver address` from the previous step

```
TestResolver.new: started
TestResolver.new: done, gas used 163231, gas price 20 Gwei
Resolver address 0xF935C0947D30a2dc45B8bd99376fED0b6dC90A26
```

Then export it as `TEST_RESOLVER_ADDRESS` in the same shell:

```
export TEST_RESOLVER_ADDRESS=0xF935C0947D30a2dc45B8bd99376fED0b6dC90A26
```

Run the deployment script again. The resolver address will be used to find the upgradeable contract. If the code matches our `/build` folder, then it will not deploy again.

```bash
npx truffle --network ganache exec scripts/deploy-framework.js
```

Deploy an example token to use with the Superfluid contracts. This also registers the token in the SF resolver.

```bash
npx truffle exec --network ganache scripts/deploy-test-token.js : fDAI
# Take note of underlying token fDAI (fake DAI)
> Token fDAI address 0xdb5d87471bC53cEF0B6ea1727ec99782A6aF59ff

```

Create token wrapper for fDAI

```bash
npx truffle exec --network ganache scripts/deploy-super-token.js : fDAI
# Take note of wrapper
> SuperToken wrapper address:  0xbaF5f08c899D27D8947a1E1856D4FeE90A07c167
```

Open Truffle console to interact with the contracts

```bash
npx truffle --network ganache console
```

Then in the console

```bash
# loads the package.json "main" entrypoint js-sdk/index.js
SuperfluidSDK = require(".")

# Load the framework with version "test"
# web3 is global variable provided by Truffle console
sf = new SuperfluidSDK.Framework({version: "test", web3Provider: web3.currentProvider })
```

> Default version is "test" `const version = process.env.RELEASE_VERSION || "test";`

Initialize many objects using the Truffle artifacts and the resolver address we exported earlier. The artifacts are stored at `sf.contracts`

```bash
await sf.initialize()

Resolver at 0xF935C0947D30a2dc45B8bd99376fED0b6dC90A26
Resolving contracts with version test
Superfluid 0xC38B...
ConstantFlowAgreementV1 0x7E...
InstantDistributionAgreementV1 0xA0D..
```

Get helpers and create an alias for wallet addresses. This makes it reference each account.

```bash
const { toWad, toBN, fromWad, wad4human } = require("@decentral.ee/web3-helpers")

bob = accounts[0]
alice = accounts[1]
carol = accounts[2]
dan = accounts[3]
```

Get the fDAI token address from the SF resolver

```bash
daiAddress = await sf.resolver.get("tokens.fDAI")
```

Load the underlying token using the dai Truffle contract object

```bash
dai = await sf.contracts.TestToken.at(daiAddress)
```

Get the address of the SF wrapped fDAI token. The wrapper address is deterministic, even if the wrapper contract was not created.

```bash
daixWrapper = await sf.getERC20Wrapper(dai)

daixWrapper
 Result {
  "0": "0xbf0d4CC8D9DBab083738747aF52fccd454b3a755",
  "1": true,
  wrapperAddress: "0xbf0d4CC8D9DBab083738747aF52fccd454b3a755",
  created: true # Whether the wrapper was created or not
 }
```

Now create the Truffle contract object for the SF wrapped fDAI token- daix aka SuperDAI

```bash
daix = await sf.contracts.ISuperToken.at(daixWrapper.wrapperAddress)
```

Mint some DAI (really fDAI) for bob & check that it worked (minting is open for anyone to call in the test ERC20 contract)

```bash
dai.mint(bob, web3.utils.toWei("100", "ether"), { from: bob })

(async () => (wad4human(await dai.balanceOf(bob))))()
> '100.00000'
```

Approve DAIX to upgrade your DAI.

```bash
dai.approve(daix.address, "1"+"0".repeat(42), { from: bob })
```

DAIX now will "upgrade" 50 DAI from Bob's wallet, wrap it and return 50 DAIX

```bash
daix.upgrade(web3.utils.toWei("50", "ether"), { from: bob })
(async () => (wad4human(await daix.balanceOf(bob))))()
> '50.00000'
```

### Create a Constant Flow Agreement "CFA"

Tell SF that Bob wants to send 100 DAI per month to alice. This is using the cfa contract in SuperFluid. A flow is defined by the amount per second which should flow.

```bash
sf.host.callAgreement(sf.agreements.cfa.address, sf.agreements.cfa.contract.methods.createFlow(daix.address, alice, "385802469135802", "0x").encodeABI(), { from: bob })
```

The calculation here is done like. Multiply by seconds in one hour, 24 hours, 30 days, divide by 10e18 wei per DAI returns the amount of DAI per month that should flow. Superfluid uses the block timestamp when performing its calculations.

```python
>>> (385802469135802 * 3600 * 24 * 30) / 10e18
99.99999999999989
```

Now, since automining is turned on, the flow should be running automatically. alice and bobs balance are updated every second. Take note that these amounts do not add up to the original amount of 50 DAI, which will be explained later.

```bash
(await daix.balanceOf(bob)).toString() / 1e18
> 48.55092592592593
(await daix.balanceOf(alice)).toString() / 1e18
> 0.006674382716049375
```

### Inspect the flow

Check the net flow for bob. The `net flow` is the current sum of all inflow/outflows for an account. We will see the same "385802469135802" we used (in units of wei) to create the flow for 100 DAI per month.

```bash
(await sf.agreements.cfa.getNetFlow(daix.address, bob)).toString() / 10e18
> -0.0000385802469135802
```

When bob created a flow, he also made a deposit. This explains why when we used `balanceOf` to calculate the balances of alice and bob, they did not add to 50 DAI. We can see bob's "real time balance", which also includes the deposit using these commands.

```bash
rtb = await daix.realtimeBalanceOf(bob, parseInt((Date.now())/1000))

(await rtb.availableBalance).toString() / 1e18
```

So how much did bob `deposit` to create the flow? We can query this directly, using `rtb`.

```bash
(await rtb.deposit).toString() / 1e18
```

This amount will be returned when bob stops the flow. It serves as insurance in case bob starts running out of money. When this happens, the off-chain components earn the deposit as payment for submitting an on-chain transaction to stop the flow.

### Stop the Flow

Now lets stop the flow by deleting it.

```
sf.host.callAgreement(sf.agreements.cfa.address, sf.agreements.cfa.contract.methods.deleteFlow(daix.address, bob, alice, "0x").encodeABI(), { from: bob })
```

Check real time balance again. You must provide the timestamp for the current moment.

> Note: you can also provide a future time, which is useful for off-chain components to know when bob will run out of money.

```bash
rtb = await daix.realtimeBalanceOf(bob, parseInt((Date.now())/1000))
(await rtb.deposit).toString() / 1e18
> 0
```

Since the flow is stopped, the deposit has been returned, so bob's `rtb` is 0. If we check `balanceOf` and add them, we should get back the original sum of 50 DAI (and some truncation error "dust")

```bash
a=(await daix.balanceOf(bob)).toString() / 1e18
b=(await daix.balanceOf(alice)).toString() / 1e18
a + b
50.00000000000001
```

Great job!

### Update a flow

Start the flow back again

```bash
sf.host.callAgreement(sf.agreements.cfa.address, sf.agreements.cfa.contract.methods.createFlow(daix.address, alice, "385802469135802", "0x").encodeABI(), { from: bob })
```

Check the flow rate to ensure its running

```cd
(await sf.agreements.cfa.getFlow(daix.address, bob, alice)).toString()
```

TODO: add docs for updating flow. Here is the .sol interface

```sol
function updateFlow(
        ISuperToken token,
        address receiver,
        int96 flowRate,
        bytes calldata ctx
    )
        external
        virtual
        returns(bytes memory newCtx);
```
