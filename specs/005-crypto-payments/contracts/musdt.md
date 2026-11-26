# Contract: mUSDT token

Source: `shop-operator/contracts/MockUSDT.sol`, OpenZeppelin Contracts ERC20 (pinned version),
compiled with a pinned `solc`; ABI and bytecode committed in `contracts/build/`, CI recompiles
and fails on any difference.

| Item | Value |
|------|-------|
| Name / symbol / decimals | `Mock USDT` / `mUSDT` / `6` |
| `mint(uint256 amount)` | Mints `amount` to `msg.sender`; reverts `MintTooLarge()` above `1_000 * 10**6` per call; anyone may call |
| ERC20 | `transfer`, `balanceOf`, `Transfer(address indexed from, address indexed to, uint256 value)` etc., unchanged |
| Owner, pause, blacklist | none |

Selectors used by the shop page: `transfer(address,uint256)` `0xa9059cbb`,
`balanceOf(address)` `0x70a08231`, `mint(uint256)` `0xa0712d68`. Transfer topic:
`0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a4df523b3ef`.

**Deployment (manual, maintainer)**: Remix, compiler and settings as pinned in
`contracts/README.md`, from the maintainer's own MetaMask on Sepolia; no constructor arguments.
Record the address in `kube-state/clusters/local/shop-operator/values.yaml`
(`payments.tokenAddress`) and in `contracts/README.md`, and verify the source on the explorer.
Tests deploy the committed bytecode on Anvil.
