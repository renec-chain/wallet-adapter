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
  encrypt(
    inputs: EncryptInput[],
    publicKey?: PublicKey
  ): Promise<{
    ciphertext: Uint8Array;
    nonce: Uint8Array;
    fromPublic: Uint8Array;
  }[]>;
  decrypt(inputs: DecryptInput[], publicKey?: PublicKey): Promise<(Uint8Array | null)[]>;
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
```javascript
// encrypt
try {
  const toPublic = 'Fja4due9hSbywGtJf8DPy91e3LtZfuTDaswYYh1UWR72';

  // wallet DwSmR358M7CfKasmEhgV6wHxZHR1Xxssr5Qx7kLipnJq encrypt
  const encrypt = await wallet.encrypt([
    {
      cleartext: "Test encrypt",
      toPublic,
    },
    {
      cleartext: "Test encrypt 2",
      toPublic,
    },
  ]);

  setEncrypt(encrypt);

  // Encrypt:
  // [
  //   {
  //     ciphertext: "0b4deffb97030c697f4d15b948703e4e55c84323a1ae661891cbe737",
  //     fromPublic: "DwSmR358M7CfKasmEhgV6wHxZHR1Xxssr5Qx7kLipnJq",
  //     nonce: "4ba69b97b7d55da54df4ce12b097c15b58ef4de83f7a30f4"
  //   },
  //   {
  //     ciphertext: "d6822ab539a6ef0c2f25899175c19724337c5f0c0bd3f33ed9cb3d110fd0",
  //     fromPublic: "DwSmR358M7CfKasmEhgV6wHxZHR1Xxssr5Qx7kLipnJq",
  //     nonce: "564152c0cb0feabd7b4899096ce77c3de4f473b377f9c66b"
  //   },
  // ]
} catch (error) {
  console.log("User denied sign encrypt!");
}
```
```javascript
// decrypt
try {
  if (!encrypt) return toast.error("Please encrypt before decrypt!");

  // wallet Fja4due9hSbywGtJf8DPy91e3LtZfuTDaswYYh1UWR72 decrypt
  const result = await wallet.decrypt(encrypt);

  console.log(result);
  // ["Test encrypt", "Test encrypt 2"]
} catch (error) {
  console.log("User denied sign decrypt!");
}
```
