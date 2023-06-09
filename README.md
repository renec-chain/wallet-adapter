## Demon Wallet Adapter for DApp

This is a wallet adapter for DApps that connects to the Demon wallet. It provides functionality for connecting, disconnecting, and signing transactions and messages using the Demon wallet.

### Installation

```bash
npm install @solana/wallet-adapter-base @solana/web3.js eventemitter3
```

### Usage

Import the necessary modules and classes:

```javascript
import {
  BaseMessageSignerWalletAdapter,
  WalletName,
  WalletNotReadyError,
  WalletSignMessageError,
  WalletSignTransactionError,
} from "@solana/wallet-adapter-base";
import {
  scopePollingDetectionStrategy,
  WalletAccountError,
  WalletDisconnectionError,
  WalletPublicKeyError,
  WalletReadyState,
} from "@solana/wallet-adapter-base";
import { Connection, Keypair, PublicKey, VersionedTransaction } from "@solana/web3.js";
import { EventEmitter } from "eventemitter3";
```

Define the interface for the Demon wallet:

```javascript
interface DemonWallet extends EventEmitter {
  isDemon?: boolean;
  connect(): Promise<string>;
  disconnect(): Promise<void>;
  signTransaction<T extends Transaction | VersionedTransaction>(
    tx: T,
    publicKey?: PublicKey
  ): Promise<T>;
  signAllTransaction<T extends Transaction | VersionedTransaction>(
    txs: T[],
    publicKey?: PublicKey
  ): Promise<T[]>;
  signMessage(msg: Uint8Array, publicKey?: PublicKey): Promise<Uint8Array>;
  sendTransaction<T extends Transaction | VersionedTransaction>(
    tx: T,
    connection: Connection,
    signers?: Keypair[],
    publicKey?: PublicKey
  ): Promise<string>;
  isConnected: boolean;
}
```

Define the interface for the Demon window object:

```javascript
interface DemonWindow extends Window {
  demon: {
    sol: DemonWallet,
  };
}
```

Declare the `window` object as `DemonWindow`:

```javascript
declare const window: DemonWindow;
```

Define the configuration interface for the Demon wallet adapter:

```javascript
export interface DemonWalletAdapterConfig {}
```

Define the `DemonWalletName` constant:

```javascript
export const DemonWalletName = "Demon" as WalletName<"Demon">;
```

Define the `DemonWalletAdapter` class that extends `BaseMessageSignerWalletAdapter`:

```javascript
export class DemonWalletAdapter extends BaseMessageSignerWalletAdapter {
  name = DemonWalletName;
  url = "https://renec.foundation/";
  icon = "..."; // Base64 encoded icon image
  readonly supportedTransactionVersions = null;
  private _connecting: boolean;
  private _wallet: DemonWallet | null;
  private _publicKey: PublicKey | null;
  private _readyState: WalletReadyState =
    typeof window === "undefined" || typeof document === "undefined"
      ? WalletReadyState.Unsupported
      : WalletReadyState.NotDetected;
  constructor(config: DemonWalletAdapterConfig = {}) {
    super();
    this._connecting = false;
    this._wallet = null;
    this._publicKey = null;
    if (this._readyState !== WalletReadyState.Unsupported) {
      scopePollingDetectionStrategy(() => {
        if (window.demon?.sol) {
          this._readyState = WalletReadyState.Installed;
          this.emit("readyStateChange", this._readyState);
          return true;
        }
        return false;
      });
    }
  }
  // Rest of the class implementation...
}
```
### Usage Example
```javascript
// Create an instance of the Demon wallet adapter
const wallet = new DemonWalletAdapter();
// Connect to the Demon wallet
await wallet.connect().then((walletAddress) => {
  // do something
});
```
```javascript
// sign message
try {
  const message = "Test sign message";
  const encodedMessage = new TextEncoder().encode(message);
  const signature = await wallet.signMessage(encodedMessage);
  console.log("signature", signature);
} catch (error) {
  console.log("User denied sign transaction!");
}
```
```javascript
// sign transaction
const fromPubkey = new PublicKey(walletAddress);
const transaction = new Transaction().add(
  SystemProgram.transfer({
    fromPubkey,
    toPubkey: new PublicKey(toAddress),
    lamports: 100000,
  })
);
let blockhash = await (await connection.getLatestBlockhash("finalized")).blockhash;
transaction.recentBlockhash = blockhash;
transaction.feePayer = fromPubkey;
try {
  await wallet.signTransaction(transaction);
} catch (error) {
  console.log("User denied sign transaction!");
}
try {
  const connection = new Connection(networkUrl, "recent");
  await connection.sendRawTransaction(transaction.serialize());
  console.log("Send transaction success!");
} catch (error) {
  console.log("Send transaction fail!");
}
```
```javascript
// send mint transaction with signers
import { ASSOCIATED_TOKEN_PROGRAM_ID, MintLayout, Token } from "@solana/spl-token";
const mintAccount = Keypair.generate();
const walletTokenAddress = await Token.getAssociatedTokenAddress(
  ASSOCIATED_TOKEN_PROGRAM_ID,
  TOKEN_PROGRAM_ID,
  mintAccount.publicKey,
  customWallet.publicKey!
);
const balanceNeeded = await Token.getMinBalanceRentForExemptMint(customWallet.connection);
const instructions = [
  SystemProgram.createAccount({
    fromPubkey: customWallet.publicKey!,
    newAccountPubkey: mintAccount.publicKey,
    space: MintLayout.span,
    lamports: balanceNeeded,
    programId: TOKEN_PROGRAM_ID,
  }),
  Token.createInitMintInstruction(
    TOKEN_PROGRAM_ID,
    mintAccount.publicKey,
    9,
    customWallet.publicKey!,
    null
  ),
  Token.createAssociatedTokenAccountInstruction(
    ASSOCIATED_TOKEN_PROGRAM_ID,
    TOKEN_PROGRAM_ID,
    mintAccount.publicKey,
    walletTokenAddress,
    customWallet.publicKey!,
    customWallet.publicKey!
  ),
  Token.createMintToInstruction(
    TOKEN_PROGRAM_ID,
    mintAccount.publicKey,
    walletTokenAddress,
    customWallet.publicKey!,
    [],
    100 * LAMPORTS_PER_SOL
  ),
];
const transaction = new Transaction().add(...instructions);
let blockhash = await (await connection.getLatestBlockhash("finalized")).blockhash;
transaction.recentBlockhash = blockhash;
transaction.feePayer = wallet.publicKey!;
transaction.partialSign(mintAccount);
try {
  const txid = await wallet.sendTransactionWithSigners(
    transaction,
    walletContext?.wallet?.connection!,
    [mintAccount]
  );
  console.log("txid", txid);
} catch (error) {
  console.log("User denied sign transaction!");
}
```
