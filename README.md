# Merkl Camp Example — AIT rewards data

Branch `ait-rewards-data`: per-user Merkle proof files derived from
`sample.json` and `sample2.json`, in the format the AIT backend's
`GET /v1/users/:wallet/rewards` endpoint expects.

## Layout

```
{rewardToken}/{chainId}/{walletAddress}.json
```

- `rewardToken` is a folder name (not an on-chain address). Two folders
  here, one per AIT track:
  - `AIT-vault/` — vault track campaigns
  - `AIT-refer/` — referral track campaigns
- `chainId` = `8453` (Base mainnet), matching the source samples.
- `walletAddress` is EIP-55-checksummed.

## File format

```json
{
  "proof":   [["0x...", "0x..."], ...],
  "id":      ["0", "1", ...],
  "amounts": ["1000000", "1500000", ...]
}
```

Parallel arrays — `proof[i]`, `id[i]`, `amounts[i]` belong to the same
claim. A 404 means the wallet has no rewards for that token/chain.

## ⚠️ Proofs are placeholders

The `proof[]` values in these fixture files are **deterministic
placeholders** (e.g. `0x1111...1111`), not real Merkle proofs. They
exist so the backend can exercise the response shape end-to-end.
Replace with real proofs generated from an actual Merkle tree before
using on-chain.

## Derivation from source samples

`sample.json` and `sample2.json` use a per-wallet `{reason → {amount,
timestamp}}` shape. Conversion to the per-user file format:

| Source reason key                          | Track  |
|--------------------------------------------|--------|
| `epoch-N`                                  | vault  |
| `vaults`                                   | vault  |
| `refers`                                   | refer  |
| `referral:*`                               | refer  |

Per-wallet amounts collapse into `amounts[]` entries with sequential
`id[]` values starting at `"0"`.

### Wallet → claims map

**`AIT-vault/8453/`:**
| Wallet | id | amount | source |
|--------|----|--------|--------|
| `0x9df7C98C933A0cB409606A3A24B1660a70283542` | 0 | 4000000  | sample.json `epoch-1` |
| `0x077EeF2934Db480826326d880788eCc12d131C6a` | 0 | 1000000  | sample2.json `vaults` |
| `0xDE6D6f23AabBdC9469C8907eCE7c379F98e4Cb75` | 0 | 2000000  | sample2.json `vaults` |
| `0x95E111E87847Cdb3E3e9Bf16607A36099115dEC7` | 0 | 1200000  | sample2.json `vaults` |

**`AIT-refer/8453/`:**
| Wallet | id | amount | source |
|--------|----|--------|--------|
| `0x9df7C98C933A0cB409606A3A24B1660a70283542` | 0 | 6000000  | sample.json `referral:e0:hyperliquid:...` |
| `0x5ef01a9aB62f700BB0BCC0F11f9CF7aa8fc543fd` | 0 | 5000000  | sample.json `referral:e0:hyperliquid:...` |
| `0x5ef01a9aB62f700BB0BCC0F11f9CF7aa8fc543fd` | 1 | 10000000 | sample.json `referral:e7:polymarket-referral:...` |
| `0x077EeF2934Db480826326d880788eCc12d131C6a` | 0 | 1500000  | sample2.json `refers` |
| `0xDE6D6f23AabBdC9469C8907eCE7c379F98e4Cb75` | 0 | 3500000  | sample2.json `refers` |
| `0x95E111E87847Cdb3E3e9Bf16607A36099115dEC7` | 0 | 800000   | sample2.json `refers` |

## Using with the backend

```bash
# In AIT-backend/.env
GITHUB_RAW_BASE=https://raw.githubusercontent.com/MaxShotLab/Merkl_Camp_Example/ait-rewards-data
```

The campaign registry (`AIT-backend/config/merkl-campaigns.yaml`) must
have entries whose `rewardToken` matches a folder name (`AIT-vault` or
`AIT-refer`), `chainId` = `8453`, and `rewardTokenAddress` set to the
on-chain reward token (USDC on Base = `0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913`,
6 decimals).
