# Merkl Camp Example — AIT rewards data

Branch `ait-rewards-data`: per-user Merkle proof files in the format the AIT
backend's `GET /v1/users/:wallet/rewards` endpoint expects.

## Layout

```
{rewardToken}/{chainId}/{walletAddress}.json
```

- `rewardToken` is a folder name (not an on-chain address). Two folders
  here, one per AIT track:
  - `AIT-vault/` — vault track campaigns (all epochs)
  - `AIT-refer/` — referral track campaigns (all epochs)
- `chainId` = `8453` (Base mainnet).
- `walletAddress` is EIP-55-checksummed.

**One file per `(wallet, track, chain)`, holding every epoch the wallet has
claims for** — the file is _not_ split per epoch.

## File format

```json
{
  "proof":   [["0x...", "0x..."], ["0x...", "0x..."]],
  "id":      ["0", "1"],
  "amounts": ["1000000", "1500000"]
}
```

Parallel arrays — `proof[i]`, `id[i]`, `amounts[i]` belong to the same claim.

**`id[i]` is the epoch number** for that claim. A wallet that earned in epoch 0
and epoch 2 (skipping 1) has `id: ["0", "2"]`. The on-chain distributor's
`claim(epoch, amount, proof)` call takes the epoch as the leaf key, so
`id[i]` doubles as the on-chain claim id.

A 404 means the wallet has no rewards for that (track, chain) at all. A file
existing but missing an entry for some epoch means no claim for that specific
epoch.

## ⚠️ Proofs are placeholders

The `proof[]` values in these fixture files are **deterministic placeholders**
(e.g. `0x1111...1111`), not real Merkle proofs. They exercise the backend's
response shape end-to-end but won't verify against the distributor on-chain.
Replace with real proofs from an actual Merkle tree builder before going live.

## Derivation from source samples

`sample.json` and `sample2.json` use a per-wallet `{reason → {amount,
timestamp}}` shape. Each `reason` was bucketed into a `(track, epoch)` pair and
folded into the wallet's file:

| Source reason key  | Track  | Epoch source |
|--------------------|--------|--------------|
| `epoch-N`          | vault  | `N` |
| `referral:eN:*`    | refer  | `N` |
| `vaults`           | vault  | `1` (current) |
| `refers`           | refer  | `1` (current) |

For coverage, every wallet in these fixtures has claims at multiple epochs so
the dev server's `?status=active|upcoming|ended` views are populated:

Amounts below are shown as **human AIT** (the on-chain values in the JSON files
are `× 10^18` — AIT has 18 decimals). The `0x5ef0…` refer entry intentionally
skips e1 to exercise the "wallet has no claim for this epoch" path.

### Vault track (`AIT-vault/8453/`)

| Wallet | e0 | e1 | e2 | e3 | e4 |
|---|---|---|---|---|---|
| `0x077EeF2934Db480826326d880788eCc12d131C6a` | 0.5 AIT | 1.0 AIT | 2.0 AIT |   |   |
| `0x95E111E87847Cdb3E3e9Bf16607A36099115dEC7` | 0.7 AIT | 1.2 AIT | 1.5 AIT | 1.8 AIT |   |
| `0x9df7C98C933A0cB409606A3A24B1660a70283542` | 2.0 AIT | 4.0 AIT | 5.0 AIT | 6.0 AIT | 8.0 AIT |
| `0xDE6D6f23AabBdC9469C8907eCE7c379F98e4Cb75` | 1.0 AIT | 2.0 AIT | 3.0 AIT |   |   |

### Refer track (`AIT-refer/8453/`)

| Wallet | e0 | e1 | e2 | e3 |
|---|---|---|---|---|
| `0x077EeF2934Db480826326d880788eCc12d131C6a` | 0.8 AIT | 1.5 AIT  | 2.2 AIT |   |
| `0x5ef01a9aB62f700BB0BCC0F11f9CF7aa8fc543fd` | 5.0 AIT | *(skipped)* | 8.0 AIT |   |
| `0x95E111E87847Cdb3E3e9Bf16607A36099115dEC7` | 0.4 AIT | 0.8 AIT  | 1.1 AIT |   |
| `0x9df7C98C933A0cB409606A3A24B1660a70283542` | 3.0 AIT | 6.0 AIT  | 7.0 AIT |   |
| `0xDE6D6f23AabBdC9469C8907eCE7c379F98e4Cb75` | 1.8 AIT | 3.5 AIT  | 4.5 AIT | 5.5 AIT |

## Using with the backend

```bash
# In AIT-backend/.env
GITHUB_RAW_BASE=https://raw.githubusercontent.com/MaxShotLab/Merkl_Camp_Example/ait-rewards-data
```

In `AIT-backend/config/merkl-campaigns.yaml`, every vault campaign uses
`rewardToken: AIT-vault` and every refer campaign uses `rewardToken: AIT-refer`
— the per-campaign `epoch` field selects which entry inside the wallet's file
to return.
