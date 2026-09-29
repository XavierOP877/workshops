# Merkle-tree Airdrops

## Overview

You build the airdrop pattern many token launches use: a list of thousands of `(address, amount)` pairs is compressed into a single 32-byte **Merkle root** stored on chain, and every recipient claims their own share by presenting a short proof. The project owner pays for one deployment and funding, not thousands of transfers, and the contract never stores the list.

The workshop covers the whole pipeline:

```
allowlist.csv ──► generate-tree.mjs ──► tree.json ──► /api/proof ──► Claim button ──► MerkleAirdrop.claim()
                         │                                                                  ▲
                         └──────────── MERKLE_ROOT ──► Deploy.s.sol ────────────────────────┘
```

The same contract pays out a **new ERC-20** you deploy in the workshop, **any existing ERC-20**, or **native AVAX**. You pick which at deploy time.

You build everything yourself, starting from an empty folder. By the end it looks like this:

```
merkle-airdrop/
├── contracts/          forge init
│   ├── src/            AirdropToken.sol, MerkleAirdrop.sol
│   ├── test/           MerkleAirdrop.t.sol
│   └── script/         Deploy.s.sol
└── app/                create-rainbowkit (Next.js + RainbowKit + wagmi)
    ├── data/           allowlist.csv, tree.json (generated)
    ├── scripts/        generate-tree.mjs
    └── src/            airdrop.ts, wagmi.ts, components/Claim.tsx, pages/index.tsx, pages/api/proof.ts
```

## Learning objectives

- Explain what a Merkle root commits to and why a proof is only `log₂(n)` hashes long.
- Build a tree off chain with OpenZeppelin's `StandardMerkleTree` and verify its proofs on chain with `MerkleProof`.
- Explain why the leaf is hashed twice and why it must contain the claimer's address.
- Serve proofs from a small backend so the full allowlist never ships to the browser.
- Wire a claim flow with RainbowKit and wagmi: look up, read state, write, wait for the receipt.

## Prerequisites

- Completed [Getting Started with Avalanche](../../getting-started/en/README.md): a wallet (Core or MetaMask) on Fuji with test AVAX.
- Comfortable reading Solidity and TypeScript/React.
- [Foundry](https://book.getfoundry.sh): `curl -L https://getfoundry.sh/install | bash && foundryup`
- [Node.js](https://nodejs.org) 20.9 or newer (required by Next.js 16).
- For the Fuji part: a **throwaway** wallet funded from the [Fuji faucet](https://go.team1.network/faucet). Never use a wallet that holds real funds in a workshop.

## Workshop

### Part 1: Merkle trees in ten minutes

You want to send tokens to 10,000 addresses. Three options:

| Approach | Who pays gas | What you store up front |
|---|---|---|
| Loop over 10,000 transfers | You, for all 10,000 | nothing |
| Store the list in a mapping, let people claim | You, for 10,000 storage writes | 10,000 storage slots |
| Store a Merkle root, let people claim with a proof | Each claimer, for their own claim | **one 32-byte root** |

With the Merkle root, each claim still writes a little storage (a "claimed" flag, plus the claimer's token balance for an ERC-20), but the claimer pays for it, and only people who actually claim do.

A Merkle tree hashes every entry into a **leaf**, then hashes pairs of hashes upward until one hash is left: the **root**.

```
                     root = H(n01, n23)
                    /                  \
          n01 = H(A, B)              n23 = H(C, D)
          /         \                /         \
   A = leaf(alice)  B = leaf(bob)  C = leaf(carol)  D = leaf(dave)
```

Change a single bit of any entry and the root changes. That is what "the root commits to the list" means.

To prove Alice is on the list, you do not need the whole tree. You need only the **siblings** on her path to the root: `B`, then `n23`. The contract recomputes `H(H(A, B), n23)` and checks the result equals the stored root. With 10,000 entries a proof is at most 14 hashes (`⌈log₂ 10,000⌉`). With a million entries it is at most 20.

Two details matter for security, and both are handled by OpenZeppelin's libraries:

1. **The leaf binds the address.** A leaf is `hash(account, amount)`. Proofs are public (they sit in every claim transaction), but Alice's proof is useless to Eve because Eve's address produces a different leaf.
2. **The leaf is hashed twice.** `StandardMerkleTree` uses `keccak256(keccak256(abi.encode(account, amount)))`. An inner node is the hash of exactly 64 bytes (its two children), and so is a single-hashed leaf, because `abi.encode(address, uint256)` is also 64 bytes. Without the second hash, the two children of an inner node could be decoded as an `(account, amount)` pair and claimed as if they were a leaf (a *second-preimage attack*). With an `address` as the first field the attack happens to be impractical, since the left child would need 12 leading zero bytes, but the double hash rules it out for any leaf type.

OpenZeppelin also **sorts each pair** before hashing, so a proof is a plain list of hashes with no left or right flags.

### Part 2: Create the contracts project

Start from an empty folder. `forge init` creates a Foundry project, initializes git, and installs `forge-std`:

```sh
mkdir merkle-airdrop && cd merkle-airdrop
forge init contracts
cd contracts
rm src/Counter.sol test/Counter.t.sol script/Counter.s.sol
forge install OpenZeppelin/openzeppelin-contracts
```

Foundry picks up the `@openzeppelin/contracts/` import path from the installed library on its own. Check it with `forge remappings`.

Replace `foundry.toml` to pin the compiler. Solidity 0.8.28 with the Cancun EVM version compiles to opcodes the Avalanche C-Chain supports:

```toml
[profile.default]
src = "src"
out = "out"
libs = ["lib"]
solc_version = "0.8.28"
evm_version = "cancun"
optimizer = true
optimizer_runs = 200
```

#### The token

Create `src/AirdropToken.sol`. It is a plain OpenZeppelin ERC-20 with a fixed supply and nothing airdrop-specific in it:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

/// @title  AirdropToken
/// @notice A plain OpenZeppelin ERC-20 with a fixed supply minted to the deployer. Nothing airdrop-specific lives
///         here: the airdrop contract works with any ERC-20, this one just gives the workshop something to send.
contract AirdropToken is ERC20 {
    constructor(uint256 initialSupply) ERC20("Workshop Drop", "DROP") {
        _mint(msg.sender, initialSupply);
    }
}
```

#### The airdrop contract

Create `src/MerkleAirdrop.sol`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {MerkleProof} from "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";

/// @title  MerkleAirdrop
/// @notice Pays out a fixed allowlist of (account, amount) pairs. Only the Merkle root of the list is stored on
///         chain; each claimer brings their own proof. Pays either an ERC-20 or native AVAX.
/// @dev    Leaves use OpenZeppelin's StandardMerkleTree encoding: keccak256(bytes.concat(keccak256(abi.encode(
///         account, amount)))). The double hash stops a 64-byte inner node from being passed off as a leaf.
contract MerkleAirdrop {
    using SafeERC20 for IERC20;

    /// @notice Token paid out, or address(0) for native AVAX.
    address public immutable TOKEN;

    /// @notice Root of the allowlist tree. Immutable: a new list needs a new airdrop.
    bytes32 public immutable MERKLE_ROOT;

    /// @notice Whether an account already claimed. One claim per account, for its full amount.
    mapping(address account => bool) public hasClaimed;

    event Claimed(address indexed account, uint256 amount);

    error AlreadyClaimed();
    error InvalidProof();
    error NativeTransferFailed();

    /// @param token ERC-20 to distribute, or address(0) for native AVAX.
    /// @param merkleRoot Root printed by `generate-tree.mjs`.
    /// @dev Payable so a native airdrop can be funded in the deploy transaction.
    constructor(address token, bytes32 merkleRoot) payable {
        TOKEN = token;
        MERKLE_ROOT = merkleRoot;
    }

    /// @notice Accepts AVAX top-ups for a native airdrop.
    receive() external payable {}

    /// @notice Claim `amount` for `account`. Anyone may submit the transaction; funds always go to `account`.
    /// @param proof Sibling hashes from the leaf up to the root, served by the app's /api/proof route.
    function claim(address account, uint256 amount, bytes32[] calldata proof) external {
        if (hasClaimed[account]) revert AlreadyClaimed();

        if (!MerkleProof.verify(proof, MERKLE_ROOT, _leaf(account, amount))) revert InvalidProof();

        // Effects before interactions: a re-entering receiver sees hasClaimed == true.
        hasClaimed[account] = true;
        emit Claimed(account, amount);

        if (TOKEN == address(0)) {
            (bool ok,) = account.call{value: amount}("");
            if (!ok) revert NativeTransferFailed();
        } else {
            IERC20(TOKEN).safeTransfer(account, amount);
        }
    }

    /// @dev keccak256(bytes.concat(keccak256(abi.encode(account, amount)))), written in assembly to hash in the
    ///      free scratch space (0x00-0x3f) instead of allocating memory for abi.encode.
    function _leaf(address account, uint256 amount) private pure returns (bytes32 leaf) {
        assembly ("memory-safe") {
            mstore(0x00, account)
            mstore(0x20, amount)
            mstore(0x00, keccak256(0x00, 0x40)) // inner hash of the 64-byte abi.encode(account, amount)
            leaf := keccak256(0x00, 0x20) // outer hash of that 32-byte result
        }
    }
}
```

Things to point out:

- **The leaf is rebuilt on chain.** `claim` never trusts a leaf from the caller. `_leaf` rebuilds `keccak256(keccak256(abi.encode(account, amount)))` from the arguments, so the proof only works for the exact account and amount in the list. It is written in assembly because the plain Solidity version allocates new memory for `abi.encode` and `bytes.concat` on every call, while the assembly version hashes in the 64 bytes of scratch space Solidity reserves for exactly this. `test_ScriptGeneratedTreeVerifies` proves the two produce the same hash.
- **`TOKEN == address(0)` means native AVAX.** One contract covers both cases. For an ERC-20, `SafeERC20` handles tokens that return no boolean.
- **Anyone can submit a claim for `account`**, but the funds always go to `account`. A project can pay gas for its users (a relayer), and a stranger who front-runs you only pays your gas for you. Your own transaction then reverts with `AlreadyClaimed`.
- **`hasClaimed` is set before the transfer.** A malicious receiver that re-enters `claim` hits `AlreadyClaimed`.
- **`MERKLE_ROOT` is `immutable`.** No admin can swap the list after launch. Correcting a list means deploying a new airdrop.

Compile:

```sh
forge build
```

#### The tests

Create `test/MerkleAirdrop.t.sol`. The tests build the four-leaf tree from Part 1 by hand, so you can see that a proof really is just `[sibling, uncle]`:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Test} from "forge-std/Test.sol";
import {MerkleAirdrop} from "../src/MerkleAirdrop.sol";
import {AirdropToken} from "../src/AirdropToken.sol";

/// @dev A receiver that refuses AVAX, to prove a failed native payout reverts the whole claim.
contract Rejecter {
    receive() external payable {
        revert("no thanks");
    }
}

contract MerkleAirdropTest is Test {
    address alice = makeAddr("alice");
    address bob = makeAddr("bob");
    address carol = makeAddr("carol");
    address relayer = makeAddr("relayer");
    address rejecter = address(new Rejecter());

    //            root
    //          /      \
    //       n01        n23
    //      /   \      /   \
    //   alice  bob  carol  rejecter
    bytes32[4] leaves;
    bytes32 n01;
    bytes32 n23;
    bytes32 root;

    function setUp() public {
        leaves[0] = _leaf(alice, 100 ether);
        leaves[1] = _leaf(bob, 200 ether);
        leaves[2] = _leaf(carol, 300 ether);
        leaves[3] = _leaf(rejecter, 400 ether);
        n01 = _hashPair(leaves[0], leaves[1]);
        n23 = _hashPair(leaves[2], leaves[3]);
        root = _hashPair(n01, n23);
    }

    function test_ClaimERC20() public {
        (MerkleAirdrop drop, AirdropToken token) = _erc20Airdrop();

        vm.prank(alice);
        drop.claim(alice, 100 ether, _proof(leaves[1], n23));

        assertEq(token.balanceOf(alice), 100 ether);
        assertTrue(drop.hasClaimed(alice));
    }

    function test_ClaimNative() public {
        MerkleAirdrop drop = new MerkleAirdrop{value: 1000 ether}(address(0), root);

        vm.prank(carol);
        drop.claim(carol, 300 ether, _proof(leaves[3], n01));

        assertEq(carol.balance, 300 ether);
    }

    function test_RelayedClaimPaysAccountNotSender() public {
        (MerkleAirdrop drop, AirdropToken token) = _erc20Airdrop();

        vm.prank(relayer);
        drop.claim(bob, 200 ether, _proof(leaves[0], n23));

        assertEq(token.balanceOf(bob), 200 ether);
        assertEq(token.balanceOf(relayer), 0);
    }

    function test_RevertWhen_ClaimedTwice() public {
        (MerkleAirdrop drop,) = _erc20Airdrop();
        bytes32[] memory proof = _proof(leaves[1], n23);

        drop.claim(alice, 100 ether, proof);
        vm.expectRevert(MerkleAirdrop.AlreadyClaimed.selector);
        drop.claim(alice, 100 ether, proof);
    }

    function test_RevertWhen_AmountInflated() public {
        (MerkleAirdrop drop,) = _erc20Airdrop();

        vm.expectRevert(MerkleAirdrop.InvalidProof.selector);
        drop.claim(alice, 101 ether, _proof(leaves[1], n23));
    }

    function test_RevertWhen_ProofStolen() public {
        (MerkleAirdrop drop,) = _erc20Airdrop();
        address eve = makeAddr("eve");

        // Alice's proof is public once she claims. It is useless to anyone else: the leaf is bound to her address.
        vm.expectRevert(MerkleAirdrop.InvalidProof.selector);
        drop.claim(eve, 100 ether, _proof(leaves[1], n23));
    }

    function test_RevertWhen_NativeReceiverRejects() public {
        MerkleAirdrop drop = new MerkleAirdrop{value: 1000 ether}(address(0), root);

        vm.expectRevert(MerkleAirdrop.NativeTransferFailed.selector);
        drop.claim(rejecter, 400 ether, _proof(leaves[2], n01));
        assertFalse(drop.hasClaimed(rejecter)); // the revert rolled the flag back too
    }

    /// @dev Values printed by `node scripts/generate-tree.mjs` for the sample data/allowlist.csv. If this fails,
    ///      the script and the contract disagree on the leaf encoding.
    function test_ScriptGeneratedTreeVerifies() public {
        bytes32 scriptRoot = SCRIPT_ROOT;
        MerkleAirdrop drop = new MerkleAirdrop{value: 1000 ether}(address(0), scriptRoot);

        drop.claim(SCRIPT_ACCOUNT, SCRIPT_AMOUNT, _scriptProof());
        assertEq(SCRIPT_ACCOUNT.balance, SCRIPT_AMOUNT);
    }

    // ---------------------------------------------------------------------------------------------------------------

    function _erc20Airdrop() internal returns (MerkleAirdrop drop, AirdropToken token) {
        token = new AirdropToken(1000 ether);
        drop = new MerkleAirdrop(address(token), root);
        assertTrue(token.transfer(address(drop), 1000 ether));
    }

    /// @dev Same encoding as OpenZeppelin's StandardMerkleTree.of(values, ["address", "uint256"]).
    function _leaf(address account, uint256 amount) internal pure returns (bytes32) {
        return keccak256(bytes.concat(keccak256(abi.encode(account, amount))));
    }

    /// @dev OpenZeppelin sorts each pair before hashing, so a proof never has to say "left" or "right".
    function _hashPair(bytes32 a, bytes32 b) internal pure returns (bytes32) {
        return a < b ? keccak256(abi.encode(a, b)) : keccak256(abi.encode(b, a));
    }

    function _proof(bytes32 sibling, bytes32 uncle) internal pure returns (bytes32[] memory p) {
        p = new bytes32[](2);
        p[0] = sibling;
        p[1] = uncle;
    }

    // Anvil dev account #1's entry in the sample allowlist (250 tokens, 18 decimals).
    bytes32 constant SCRIPT_ROOT = 0x3370f46db9a7094b71229fb2ad0a0eeaf1478670b0f270b106873d1e2b2194d8;
    address constant SCRIPT_ACCOUNT = 0x70997970C51812dc3A010C7d01b50e0d17dc79C8;
    uint256 constant SCRIPT_AMOUNT = 250 ether;

    function _scriptProof() internal pure returns (bytes32[] memory p) {
        p = new bytes32[](2);
        p[0] = 0x91ca955de9f6cc63db9e7369302acfbbd2c9c7ca7c6d3cc8cf9c9acaead52c6c;
        p[1] = 0x4e868e39005be45583f2aa1f7140f4479057bfd3fcd009d6713a21faaba8f3f5;
    }
}
```

```sh
forge test
```

You should see 8 passing tests. `test_RevertWhen_ProofStolen` and `test_RevertWhen_AmountInflated` show the leaf binding at work. `test_ScriptGeneratedTreeVerifies` uses a root and proof produced by the tree script you write in Part 3. If the script and the contract ever disagreed on how a leaf is hashed, this test would fail.

### Part 3: Create the app and generate the tree

Go back to the `merkle-airdrop` folder and scaffold a RainbowKit app next to `contracts/`. The template is a Next.js app with RainbowKit, wagmi and viem already wired up:

```sh
cd ..
npm init @rainbow-me/rainbowkit@latest app
cd app
npm install @openzeppelin/merkle-tree
mkdir -p data scripts
```

#### The allowlist

Create `data/allowlist.csv`. Amounts are in human units (`100` = 100 tokens). The rows are anvil's public dev accounts #0 to #3:

```csv
address,amount
0xf39Fd6e51aad88F6F4ce6aB8827279cffFb92266,100
0x70997970C51812dc3A010C7d01b50e0d17dc79C8,250
0x3C44CdDdB6a900fa2b585dd299e03d12FA4293BC,50.5
0x90F79bf6EB2c4f870365E785982E1f101E93b906,1000
```

#### The tree script

Create `scripts/generate-tree.mjs`:

```js
// Builds the Merkle tree from data/allowlist.csv and writes data/tree.json for the /api/proof route.
// Prints MERKLE_ROOT and TOTAL to stdout as KEY=value lines, ready for the deploy script's .env.
// Amounts in the CSV are human units; set DECIMALS if your token does not use 18 (e.g. DECIMALS=6 for USDC).
import { readFileSync, writeFileSync } from "node:fs";
import { StandardMerkleTree } from "@openzeppelin/merkle-tree";
import { getAddress, isAddress, parseUnits } from "viem";

const decimals = Number(process.env.DECIMALS ?? 18);
const lines = readFileSync("data/allowlist.csv", "utf8").trim().split(/\r?\n/).slice(1); // skip header

const seen = new Set();
const values = lines.map((line, i) => {
  const cols = line.split(",").map((s) => s.trim());
  // An extra column usually means a decimal comma ("1,5"), which would otherwise silently pay 1.
  if (cols.length !== 2) throw new Error(`line ${i + 2}: expected "address,amount", got ${cols.length} columns`);
  const [rawAddress, rawAmount] = cols;
  // parseUnits rounds excess decimals instead of failing; refuse them so nobody gets a rounded amount.
  if ((rawAmount.split(".")[1] ?? "").length > decimals) {
    throw new Error(`line ${i + 2}: "${rawAmount}" has more than ${decimals} decimals`);
  }
  // Default strict mode checks the EIP-55 checksum on mixed-case input, so a typo fails here instead of locking funds.
  if (!isAddress(rawAddress)) throw new Error(`line ${i + 2}: invalid address "${rawAddress}"`);
  const account = getAddress(rawAddress);
  // A second row for the same account would be unclaimable: the contract allows one claim per account.
  if (seen.has(account)) throw new Error(`line ${i + 2}: duplicate address ${account}`);
  seen.add(account);
  const amount = parseUnits(rawAmount, decimals);
  if (amount <= 0n) throw new Error(`line ${i + 2}: amount must be positive`);
  return [account, amount.toString()];
});

// Leaf = keccak256(keccak256(abi.encode(address, uint256))), exactly what MerkleAirdrop.claim recomputes.
const tree = StandardMerkleTree.of(values, ["address", "uint256"]);
writeFileSync("data/tree.json", JSON.stringify(tree.dump(), null, 2));

const total = values.reduce((sum, [, amount]) => sum + BigInt(amount), 0n);
console.error(`Wrote data/tree.json with ${values.length} entries.`);
console.log(`MERKLE_ROOT=${tree.root}`);
console.log(`TOTAL=${total}`);
```

The script validates the list before it hashes anything. It rejects invalid addresses (including mixed-case addresses with a wrong EIP-55 checksum), rows that are not exactly `address,amount`, amounts that are not positive or have more decimals than the token, and **duplicates**, because the contract allows one claim per account and a second row for the same address could never be claimed. Every value goes through `StandardMerkleTree.of(values, ["address", "uint256"])`, which is the encoding `claim` recomputes.

Run it:

```sh
node scripts/generate-tree.mjs
```

```
Wrote data/tree.json with 4 entries.
MERKLE_ROOT=0x3370f46db9a7094b71229fb2ad0a0eeaf1478670b0f270b106873d1e2b2194d8
TOTAL=1400500000000000000000
```

This root is `SCRIPT_ROOT` in your test file: the script and the contract agree.

Now **add yourself to `data/allowlist.csv`**, using the address of the wallet you will claim with, and run the script again. The root changes, because the list changed. This time, write the output straight into the deploy environment:

```sh
node scripts/generate-tree.mjs > ../contracts/.env
```

Only the two `KEY=value` lines go to stdout, so this is safe. The status message goes to stderr.

`TOTAL` is in base units (wei for an 18-decimal token). If you airdrop an existing token with different decimals, run `DECIMALS=6 node scripts/generate-tree.mjs`.

### Part 4: Deploy

Create `script/Deploy.s.sol` in `contracts/`. It deploys the airdrop and funds it in the same run:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity 0.8.28;

import {Script, console} from "forge-std/Script.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {MerkleAirdrop} from "../src/MerkleAirdrop.sol";
import {AirdropToken} from "../src/AirdropToken.sol";

/// @notice Deploys and funds an airdrop in one go. Reads from the environment:
///         MERKLE_ROOT  root printed by generate-tree.mjs
///         TOTAL        sum of all amounts in base units, also printed by generate-tree.mjs
///         ASSET        "new" (default): deploy AirdropToken and fund from it
///                      "native":        fund with AVAX sent in the deploy transaction
///                      0x…:             an existing ERC-20 the deployer already holds at least TOTAL of
contract Deploy is Script {
    using SafeERC20 for IERC20;

    function run() external {
        bytes32 root = vm.envBytes32("MERKLE_ROOT");
        uint256 total = vm.envUint("TOTAL");
        string memory asset = vm.envOr("ASSET", string("new"));

        vm.startBroadcast();
        MerkleAirdrop drop;
        if (_eq(asset, "native")) {
            drop = new MerkleAirdrop{value: total}(address(0), root);
        } else {
            address token = _eq(asset, "new") ? address(new AirdropToken(total)) : vm.parseAddress(asset);
            drop = new MerkleAirdrop(token, root);
            IERC20(token).safeTransfer(address(drop), total);
            // Catches fee-on-transfer tokens, which would leave the last claimers unpaid.
            require(IERC20(token).balanceOf(address(drop)) >= total, "airdrop underfunded");
            console.log("Token:", token);
        }
        vm.stopBroadcast();

        console.log("MerkleAirdrop deployed at:", address(drop));
    }

    function _eq(string memory a, string memory b) internal pure returns (bool) {
        return keccak256(bytes(a)) == keccak256(bytes(b));
    }
}
```

It reads `MERKLE_ROOT`, `TOTAL` and `ASSET` from `contracts/.env`, which Foundry loads automatically:

| `ASSET` | What happens |
|---|---|
| `new` (default) | Deploys `AirdropToken` with exactly `TOTAL` supply and moves all of it into the airdrop |
| `native` | Funds the airdrop with `TOTAL` AVAX sent in the deploy transaction |
| `0x…` | Uses an existing ERC-20. Your wallet must hold at least `TOTAL` of it |

#### Locally on anvil first

In a new terminal, start a local chain:

```sh
anvil
```

Back in your first terminal:

```sh
cd ../contracts
forge script script/Deploy.s.sol --rpc-url http://127.0.0.1:8545 --broadcast \
  --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

That key is anvil's public dev account #0. It is fine locally and must never be used anywhere else. The script prints `MerkleAirdrop deployed at: 0x…`.

Try the native variant as well: `ASSET=native forge script …` with the same flags.

#### Then on Fuji

First, **remove the four anvil rows** from `data/allowlist.csv` and keep only real addresses, such as your own. Their private keys are public, so on Fuji anyone could use them to claim those allocations and drain the airdrop. Then regenerate the tree and the deploy environment:

```sh
cd ../app
node scripts/generate-tree.mjs > ../contracts/.env
cd ../contracts
```

From now on `data/tree.json` matches your Fuji airdrop, not the anvil one.

Import your throwaway key into an encrypted keystore, so it never sits in plain text on disk:

```sh
cast wallet import workshop --interactive      # paste the key, choose a password
cast wallet address --account workshop         # your deployer address
```

Deploy with the default, a new ERC-20. The script deploys `AirdropToken` (symbol `DROP`), mints `TOTAL` to your wallet, and moves it into the airdrop. Your wallet only pays gas, so any amounts work:

```sh
forge script script/Deploy.s.sol \
  --rpc-url https://api.avax-test.network/ext/bc/C/rpc \
  --account workshop --sender $(cast wallet address --account workshop) --broadcast
```

It prints `Token: 0x…` and `MerkleAirdrop deployed at: 0x…`. Keep both addresses.

**Alternative: native AVAX.** Put `ASSET=native` in front of the same command. The airdrop is then funded with `TOTAL` AVAX from your wallet in the deploy transaction, so the wallet needs `TOTAL` AVAX plus gas. The faucet only gives a little test AVAX, so first set small amounts in the CSV (for example `0.01`) and regenerate the tree and `.env`.

Optionally, verify the source on Snowtrace so claimers can read what they are signing. Verification needs the constructor arguments. The airdrop exposes both as public getters, so the command reads them back from the chain and you only paste the airdrop address:

```sh
AIRDROP=0xYourMerkleAirdropAddress
RPC=https://api.avax-test.network/ext/bc/C/rpc
forge verify-contract $AIRDROP src/MerkleAirdrop.sol:MerkleAirdrop \
  --chain 43113 --verifier etherscan \
  --verifier-url 'https://api.routescan.io/v2/network/testnet/evm/43113/etherscan' \
  --etherscan-api-key verifyContract \
  --constructor-args $(cast abi-encode "constructor(address,bytes32)" \
    $(cast call $AIRDROP "TOKEN()(address)" --rpc-url $RPC) \
    $(cast call $AIRDROP "MERKLE_ROOT()(bytes32)" --rpc-url $RPC))
```

### Part 5: The backend

Back in `app/`, create the API route `src/pages/api/proof.ts`. It loads `tree.json` once at startup, indexes every entry by address, and answers each lookup from memory:

```ts
import { StandardMerkleTree } from "@openzeppelin/merkle-tree";
import type { NextApiRequest, NextApiResponse } from "next";
import { getAddress, isAddress } from "viem";
import treeData from "../../../data/tree.json";

// The full allowlist stays on the server. The API answers one address at a time, so nobody downloads the whole list.
const tree = StandardMerkleTree.load(treeData as Parameters<typeof StandardMerkleTree.load<[string, string]>>[0]);

// One in-memory map built at startup, fine for ~100k entries. Past that, move proofs to a key-value store or database.
const byAccount = new Map<string, { amount: string; proof: string[] }>();
Array.from(tree.entries()).forEach(([i, [account, amount]]) => {
  byAccount.set(account, { amount, proof: tree.getProof(i) });
});

export default function handler(req: NextApiRequest, res: NextApiResponse) {
  const address = String(req.query.address ?? "");
  if (!isAddress(address, { strict: false })) return res.status(400).json({ error: "invalid address" });

  const entry = byAccount.get(getAddress(address));
  if (!entry) return res.status(404).json({ error: "not eligible" });

  res.status(200).json(entry);
}
```

Start the app:

```sh
cd ../app
npm run dev                                    # http://localhost:3000
```

The dev server keeps running in that terminal. Query it from another one:

```sh
curl "localhost:3000/api/proof?address=0xYourWalletAddress"
```

```
GET /api/proof?address=0xYou…    → 200 {"amount":"500000000000000000000","proof":[…]}
GET /api/proof?address=0xdead…   → 404 {"error":"not eligible"}
GET /api/proof?address=nope      → 400 {"error":"invalid address"}
```

Why a backend at all? You could import `tree.json` straight into the frontend, but then every visitor downloads the entire allowlist: every address and every amount. With an API route, nobody gets the list in one download. Anyone can still look up a specific address, because allocations are public on chain once claimed anyway.

The backend is **not trusted**. If it served a wrong proof, the contract would simply reject the claim. The worst a broken backend can do is fail to serve a proof. Anyone holding `tree.json` can rebuild every proof, which is why projects often publish the file (on IPFS, for example) as a fallback.

### Part 6: Claim

Four files turn the template into the claim page. First, create `src/airdrop.ts`. It holds the contract address and the part of the ABI the app calls:

```ts
import { parseAbi } from "viem";

export const AIRDROP_ADDRESS = process.env.NEXT_PUBLIC_AIRDROP_ADDRESS as `0x${string}`;

export const airdropAbi = parseAbi([
  "function claim(address account, uint256 amount, bytes32[] proof)",
  "function hasClaimed(address account) view returns (bool)",
  "function TOKEN() view returns (address)",
]);
```

Replace `src/wagmi.ts` so the app talks to Fuji and your local anvil instead of the template's chains:

```ts
import { getDefaultConfig } from "@rainbow-me/rainbowkit";
import { avalancheFuji, foundry } from "wagmi/chains";

export const config = getDefaultConfig({
  appName: "Merkle Airdrop",
  // Free at https://cloud.reown.com. Injected wallets (Core, MetaMask) work without it; WalletConnect QR does not.
  projectId: process.env.NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID ?? "YOUR_PROJECT_ID",
  chains: [avalancheFuji, foundry],
  ssr: true,
});
```

Create `src/components/Claim.tsx`:

```tsx
import { useQuery } from "@tanstack/react-query";
import { useEffect } from "react";
import { type BaseError, erc20Abi, formatUnits, zeroAddress } from "viem";
import { useAccount, useReadContract, useWaitForTransactionReceipt, useWriteContract } from "wagmi";
import { AIRDROP_ADDRESS, airdropAbi } from "../airdrop";

type Allocation = { amount: string; proof: `0x${string}`[] };

export function Claim() {
  const { address, chain } = useAccount();

  // 1. Ask the backend for this wallet's leaf and proof. null = not on the list.
  const allocation = useQuery({
    queryKey: ["proof", address],
    enabled: !!address,
    queryFn: async (): Promise<Allocation | null> => {
      const res = await fetch(`/api/proof?address=${address}`);
      if (res.status === 404) return null;
      if (!res.ok) throw new Error(`proof API returned ${res.status}`);
      return res.json();
    },
  });

  // 2. Read what the chain knows: already claimed? which asset? (address(0) = native AVAX)
  const claimed = useReadContract({
    address: AIRDROP_ADDRESS,
    abi: airdropAbi,
    functionName: "hasClaimed",
    args: address && [address],
    query: { enabled: !!address },
  });
  const { data: token } = useReadContract({ address: AIRDROP_ADDRESS, abi: airdropAbi, functionName: "TOKEN" });
  const isNative = token === zeroAddress;
  const erc20 = { address: token, abi: erc20Abi, query: { enabled: !!token && !isNative } } as const;
  const { data: symbol } = useReadContract({ ...erc20, functionName: "symbol" });
  const { data: decimals } = useReadContract({ ...erc20, functionName: "decimals" });

  // 3. Send the claim and wait for it to land.
  const { writeContract, data: hash, isPending, error } = useWriteContract();
  const receipt = useWaitForTransactionReceipt({ hash });
  useEffect(() => {
    if (receipt.isSuccess) claimed.refetch();
  }, [receipt.isSuccess, claimed.refetch]);

  if (!address) return <p>Connect your wallet to check eligibility.</p>;
  if (allocation.isPending || claimed.isPending) return <p>Checking…</p>;
  if (allocation.isError) return <p>Could not load your proof: {allocation.error.message}</p>;
  if (allocation.data === null) return <p>This address is not on the airdrop list.</p>;
  if (claimed.data) {
    // Fuji's chain config carries Snowtrace; local anvil has no explorer, so the hash stays plain text there.
    // The hash is only known in the session that sent the claim. After a reload this shows just "Claimed".
    const explorer = chain?.blockExplorers?.default.url;
    return (
      <div>
        <p>Claimed. Check your wallet.</p>
        {hash && (
          <p>
            Transaction:{" "}
            {explorer ? (
              <a href={`${explorer}/tx/${hash}`} target="_blank" rel="noreferrer">
                {hash}
              </a>
            ) : (
              hash
            )}
          </p>
        )}
      </div>
    );
  }

  const { amount, proof } = allocation.data;
  const label = `${formatUnits(BigInt(amount), isNative ? 18 : (decimals ?? 18))} ${isNative ? "AVAX" : (symbol ?? "")}`;
  const busy = isPending || receipt.isLoading;

  return (
    <div>
      <button
        type="button"
        disabled={busy}
        onClick={() =>
          writeContract({
            address: AIRDROP_ADDRESS,
            abi: airdropAbi,
            functionName: "claim",
            args: [address, BigInt(amount), proof],
          })
        }
      >
        {isPending ? "Confirm in wallet…" : receipt.isLoading ? "Claiming…" : `Claim ${label}`}
      </button>
      {error && <p>{(error as BaseError).shortMessage ?? error.message}</p>}
    </div>
  );
}
```

It goes through four steps:

1. **Look up.** `useQuery` fetches `/api/proof` for the connected address. A 404 shows "not on the list".
2. **Read state.** `useReadContract` reads `hasClaimed(address)` and `TOKEN()`. For an ERC-20 it also reads `symbol()` and `decimals()`, so the button can say `Claim 50.5 DROP` or `Claim 50.5 AVAX`.
3. **Write.** `useWriteContract` sends `claim(address, amount, proof)`.
4. **Confirm.** `useWaitForTransactionReceipt` waits for the block, then `hasClaimed` is read again and the button is replaced by "Claimed. Check your wallet." and the transaction hash. On Fuji the hash links to Snowtrace, using the explorer URL from wagmi's chain config. Local anvil has no explorer, so there it stays plain text.

Finally, replace `src/pages/index.tsx` with the connect button and the claim component:

```tsx
import { ConnectButton } from "@rainbow-me/rainbowkit";
import type { NextPage } from "next";
import Head from "next/head";
import { Claim } from "../components/Claim";
import styles from "../styles/Home.module.css";

const Home: NextPage = () => (
  <div className={styles.container}>
    <Head>
      <title>Merkle Airdrop</title>
    </Head>
    <main className={styles.main}>
      <ConnectButton />
      <h1 className={styles.title}>Merkle Airdrop</h1>
      <Claim />
    </main>
  </div>
);

export default Home;
```

Point the app at your airdrop. Next.js reads `NEXT_PUBLIC_*` variables when the dev server starts, so stop `npm run dev` (Ctrl+C) and start it again:

```sh
echo "NEXT_PUBLIC_AIRDROP_ADDRESS=0xYourMerkleAirdropAddress" > .env.local   # the airdrop on the chain your wallet uses
npm run dev                                    # http://localhost:3000
```

Open http://localhost:3000 in your browser. Click **Connect Wallet**, the RainbowKit button, choose your wallet, and approve the connection. If the wallet is on another network, RainbowKit shows **Wrong network**: click it and switch to Avalanche Fuji. The page looks up your address and shows a button such as **Claim 500 DROP**. Click it and confirm the transaction in your wallet. About two seconds later the page shows "Claimed. Check your wallet." and the transaction hash, linked to Snowtrace.

For a local run instead, `tree.json` must match the airdrop you deployed on anvil: put the anvil rows back into the CSV, run `node scripts/generate-tree.mjs > ../contracts/.env`, redeploy on anvil, and use that address in `.env.local`. Then add the network `http://127.0.0.1:8545` (chain ID `31337`) to your wallet, import one of the dev accounts from the CSV, and select **Foundry** in RainbowKit.

Now add the token to your wallet (the `Token:` address the deploy script printed) and look at your balance. Then try to claim a second time from the command line. Take `amount` and `proof` from `curl "localhost:3000/api/proof?address=0xYou"`:

```sh
# Fuji
cast send 0xYourMerkleAirdropAddress "claim(address,uint256,bytes32[])" 0xYou <amount> "[<proof0>,<proof1>,…]" \
  --rpc-url https://api.avax-test.network/ext/bc/C/rpc --account workshop

# Local anvil: anyone may submit a claim, so anvil's dev account #0 can send it
cast send 0xYourMerkleAirdropAddress "claim(address,uint256,bytes32[])" 0xYou <amount> "[<proof0>,<proof1>,…]" \
  --rpc-url http://127.0.0.1:8545 --private-key 0xac0974bec39a17e36ba4a6b4d238ff944bacb478cbed5efcae784d7bf4f2ff80
```

Both revert with `custom error 0x646cf558`, which is `AlreadyClaimed()` (check with `cast sig "AlreadyClaimed()"`). Use the RPC of the chain you claimed on. An address with no contract behind it accepts the call without an error and proves nothing.

## Next steps

- Read the OpenZeppelin `MerkleProof` source. `multiProofVerify` proves many leaves at once for less gas.
- Compare Merkle airdrops with signature-based claims, where a backend signs `(account, amount)` with a key. Which is easier to update, and which needs less trust?
- Try [Sealed-bid Auctions](../../sealed-bid-auctions/en/README.md), which builds another commitment scheme: commit-reveal.

## Resources

- [OpenZeppelin merkle-tree library](https://github.com/OpenZeppelin/merkle-tree)
- [OpenZeppelin MerkleProof](https://docs.openzeppelin.com/contracts/5.x/api/utils/cryptography#MerkleProof)
- [Uniswap MerkleDistributor](https://github.com/Uniswap/merkle-distributor)
- [Foundry book](https://book.getfoundry.sh)
- [RainbowKit documentation](https://rainbowkit.com/docs/introduction)
- [wagmi documentation](https://wagmi.sh)
- [Snowtrace testnet](https://testnet.snowtrace.io)
- [Fuji faucet](https://go.team1.network/faucet)
- [Avalanche documentation](https://go.team1.network/docs)
