So far we've learned how to inspect much of the information associated with the blockchain either through calling into contracts or through Parity's own APIs. We have seen how the Bond framework have made those tasks very easy. However, we have not yet had any impact on the blockchain's state---we have not yet sent a transaction.

Sending a transaction in Ethereum is a multi-stage process. The transaction is initially submitted to Parity for signing. During this stage, Parity it free to consult the user to ensure that they authorise this transaction to happen. While in some instances this may be instantaneous, this can also be a long process taking upwards of 60 seconds: depending on the type of account, password or passwords may need to be entered, a 2-factor authentication process may need to occur or the secret key required for signing may be on a different device.

The authorisation stage ends when either the user has decided to approve the transaction (and done all necessary steps to physically create the signature) or when the user has refused the request. Assuming they approved the transaction, it must now be sent to the network for finalisation (also known as "mining" or "confirming"). This is the process of burying the transaction into the block chain such that it can be considered final. Different consensus mechanisms have different means of determining finality. E.g. proof-of-work chains can never achieve true finality and rather rely on the increasingly small chance than an attacker can secretly create a longer chain buoyed by greater proof of "work". Conversely, some consensus algorithms are able to provide "finality" after only a single block.

Once the first block has been received in which the transaction appears, we consider the transaction to be "confirmed"; successive blocks may reveal further confirmations. For the present purposes, we consider only the first confirmation to be "important".

To create a new transaction in Parity, we require only a single new API: the `Transaction` class. Let's begin by importing it. Ensure you have something like the following to the top of your `app.jsx` file:

```jsx
import {Transaction, formatBalance} from 'oo7-parity';
```

The `Transaction` class is a type of `Bond`. Constructing it creates a new transaction for the user to approve and which can be pushed out on to the network. The value to which the Bond evaluates reflects the ongoing status of the transaction as it moves through the process of getting finalised. It is always an object with a single key/value. It can be:

- `{"requested": id}` The user has been asked for approval of the transaction; `id` is the (numeric) identity of the request.
- `{"signed": hash}` The user has approved the transaction and the transaction has been signed to form a final transaction hash of `hash`. It is now ready to be sent to the network for inclusion in a new block.
- `{"confirmed": receipt}` The transaction has been confirmed into a block. The receipt of the transaction is provided as `receipt`, giving information concerning its operation, including any logs.
- `{"failed": error}` The transaction has failed. Generally this is because the user did not approve the transaction or otherwise signing could not take place. `error` is a string which contains any details regarding the situation.

Creating a new transaction is a simple affair. The constructor of `Transaction` takes a single argument: an object describing the transaction. There are several keys it may contain:

- `to` The recipient of the transaction; undefined or `null` indicates this is a contract-creation transaction.
- `from` The sender of the transaction; must be an account which the user controls. Will default to `parity.bonds.defaultAccount`.
- `value` The amount of ether to send to the recipient or endow any created contract.
- `condition` A condition on which to predicate the distribution, but not the approval/signing, of the transaction. This is an object which contains one of two keys: `block` (which dictates the minimum block number before distribution) and `time` (the same but for block timestamp). 
- `gas` The amount of gas to supply with the transaction. By default, it will attempt to estimate the amount of gas to supply. Such that the transaction succeeds without exception/error.
- `gasPrice` The price (in Wei) per unit of gas. Will default to something sensible (in Parity's case, this is the median of the distribution formed from the last several hundred transactions). 
- `nonce` The sender nonce of the transaction. Will default to the latest known value.

### Giveth to gavofyork 

Let's start simple. A button to give 100 Finney (around $1 at current prices) to me. Since we want to include a button, we'll need the appropriate Material UI import for that:

```jsx
import RaisedButton from 'material-ui/RaisedButton';
```

Next, in our class, we'll maintain the address of 'gavofyork' for efficiency and brevity:

```jsx
constructor() {
	super();
	this.gavofyork = parity.bonds.registry.lookupAddress('gavofyork', 'A');
}
```

There should be nothing surprising there.

Next for the HTML, we'll have two elements: the balance of our default account (spo that we can watch it go down once the payment goes out) and the button to make a payment:

```jsx
<div>
	My balance: <Rspan>
		{parity.bonds.balance(parity.bonds.defaultAccount).map(formatBalance)}
	</Rspan>
	<br />
	<RaisedButton
		label='Give gavofyork 100 Finney'
		onClick={() => new Transaction({to: this.gavofyork, value: 100 * 1e15})}
	/>
</div>
```

The balance is nothing that we haven't seen before. The button's `onClick` prop creates our new transaction. We wish to have the user send `gavofyork` (whose "current" address is expressed by `this.gavofyork`) 100 Finney (one Finney being equivalent to 10**15 Wei). The `new Transaction` creates our transaction; our given options of `to` and `value` reflect the attributes we wish our transaction to take.

Refresh your browser and click the button! You'll see the signer window pop up asking you for approval (and maybe a password) to send the funds.

![image](https://cloud.githubusercontent.com/assets/138296/22750413/badb3b72-edfe-11e6-810b-9bcdb47a20a8.png)

Once done give it 15-30 seconds and you should see you balance reduce by around 0.1 Ether.

### Giveth to anyone

TODO