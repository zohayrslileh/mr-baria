# Connecting a Bybit AI sub-account

Bybit authorizes agents through an OAuth flow that issues credentials for an
isolated **AI sub-account**. The scope covers creating the sub-account, creating
an API key, trading inside it, and querying positions. It excludes withdrawals,
security settings, and the main account balance. Access is revoked by deleting
the sub-account.

## What was established here

Authorization completed successfully from this repository's environment and the
credentials are valid. **Trading from that environment is not possible**, for a
reason unrelated to the credentials — see the geo-block section.

## The two flows

| flow | callback | what the user pastes |
|---|---|---|
| local agent | server on `127.0.0.1:9876` | nothing, the code arrives automatically |
| headless / cloud | `https://www.bybit.com/oauth/callback` | the authorization code |

Prefer local whenever the agent runs on your own machine: nothing sensitive
crosses a chat transcript. The headless flow is the safer of the two remote
options, because what gets pasted is the single-use **authorization code**,
valid ten minutes — not a long-lived token. Once exchanged it is inert.

## Running the flow

The official module is fetched from Bybit's repository and pinned by checksum.
Verify it before running it:

```bash
curl -sS -o oauth.js \
  https://raw.githubusercontent.com/bybit-exchange/skills/main/modules/oauth.js
sha256sum oauth.js
# dc49e997e805aeb1ca4d517b114dfa72a6bdb3f0d5b609c554d2e28c26515ec2
```

Keep credentials out of the repository:

```bash
export BYBIT_CRED_DIR="$HOME/.bybit"     # never a path inside this repo
```

Local flow — the module starts the callback server, prints an authorization
URL, and captures the code itself:

```bash
node oauth.js --port 9876 --env mainnet          # or --env testnet
node oauth.js --exchange <callback_file> --env mainnet
```

Headless flow — generate the URL, open it, paste the code back:

```bash
node oauth.js --headless --env mainnet
node oauth.js --manual-code "<code>" --init-file "<init_file>" --env mainnet
node oauth.js --exchange "<output_file>" --env mainnet
```

Check that the `state` value on the return page matches the one in the URL you
opened. If it differs, paste nothing.

2FA must be enabled on the account. Without it the flow terminates with
`ret_code=20039`.

## A mismatch in the module worth knowing about

`modules/oauth.js` treats any array response from
`/oauth/v1/resource/restrict/ai_accounts` as "the user still has to choose" and
prints a selection list instead of saving anything — including when
`--sub-member-id` was supplied. But the endpoint currently returns
`result.accounts` as an array **that already contains `api_key` and
`api_secret`**, so the selection branch always wins and the credentials are
never written.

Once you have chosen an account, write the block yourself:

```python
cred["ai-account"] = {
    "sub_member_id": account["sub_member_id"],
    "api_key":       account["api_key"],
    "api_secret":    account["api_secret"],
}
```

into `oauth_token.json` at mode `0600`. That is the shape `tools/bybit-api` and
the Bybit skill both expect.

## The geo-block

Bybit's trading API is served through CloudFront and refuses whole regions:

| host | result from this environment |
|---|---|
| `api.bybit.com` | 403, CloudFront country block |
| `api.bytick.com` | 403, CloudFront country block |
| `api.bybit.nl` | 403, CloudFront country block |
| `api.bybitglobal.com` | 403 for your country |
| `api2.bybit.com` | reachable — but serves OAuth only, not `/v5/*` |

Because OAuth lives on a host that is *not* blocked, authorization can succeed
in an environment where trading is impossible. A successful connection is
therefore not evidence that orders can be placed. Test with
`tools/bybit-api balance` before assuming anything.

Run the trading side from a machine that can reach `api.bybit.com`. Market data
is unaffected — the bmo instance reaches Bybit fine, which is why every query
in `queries/` works regardless.

## Using the client

```bash
export BYBIT_CRED_DIR="$HOME/.bybit"

tools/bybit-api balance
tools/bybit-api positions

# prints the payload and exits — nothing is sent
tools/bybit-api order --symbol BTCUSDT --side Sell --qty 0.005 \
                      --price 64150 --stop 65064

# actually sends it
tools/bybit-api order --symbol BTCUSDT --side Sell --qty 0.005 \
                      --price 64150 --stop 65064 --yes
```

Secrets are read from the credential file and never printed. `--yes` is
required for anything that reaches the account.

Sizes come from `queries/bybit-executable-size.bmo`, which already rounds to the
exchange's quantity step and reports what the risk becomes when a minimum order
size forces the position larger than intended.

## Hygiene

- Credentials belong outside the repository, at mode `0600`. The `.gitignore`
  blocks the usual filenames, but the reliable fix is to keep them elsewhere.
- Access tokens last 24h; refresh tokens 30 days. After that, re-authorize.
- Fund the sub-account with only what is being risked. The default main-to-sub
  transfer limit is 5,000 USDT — that is a ceiling, not a suggestion.
- Credentials created inside an ephemeral container die with it. Delete the AI
  sub-account when finished rather than leaving an authorization outstanding.
