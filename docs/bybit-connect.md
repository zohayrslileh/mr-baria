# Connecting a Bybit AI sub-account

Bybit authorizes agents through an OAuth flow that issues credentials for an
isolated **AI sub-account**. The scope covers creating the sub-account, creating
an API key, trading inside it, and querying positions. It excludes withdrawals,
security settings, and the main account balance. Access is revoked by deleting
the sub-account.

## What was established here

Authorization completed and the credentials are valid. Direct calls to
`api.bybit.com` from this environment are refused by CloudFront on region, so
requests are routed through a tunnel running on a machine that can reach it —
see the geo-block section. Through that tunnel the full chain is verified:
signed V5 calls return `retCode 0`, `wallet-balance` and `position/list` both
answer, and `user/query-api` reports `readOnly: 0` with `ContractTrade:
[Order, Position]`.

Only `accountType=UNIFIED` is accepted; `CONTRACT`, `SPOT` and `FUND` all
return `10001 accountType only support UNIFIED`. Margin mode is
`REGULAR_MARGIN`.

Funding the sub-account is not something the agent can do: the OAuth scope
excludes main-account access, so the transfer has to be made from the Bybit
interface.

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

### Two different 403s

CloudFront returns 403 for two unrelated reasons and they are easy to confuse:

| body says | meaning | fix |
|---|---|---|
| `configured to block access from your country` | genuine geo restriction | reach Bybit from elsewhere |
| `Bad request. We can't connect to the server` | `Host` header not recognised | rewrite Host to `api.bybit.com` |

The second is a proxy misconfiguration, not a block. A tunnel placed in front of
`api.bybit.com` must rewrite the Host header rather than forward its own:

```bash
ngrok http --host-header=rewrite https://api.bybit.com
```

```nginx
proxy_pass            https://api.bybit.com;
proxy_set_header Host api.bybit.com;
proxy_ssl_server_name on;
proxy_ssl_name        api.bybit.com;
```

Point the client at the tunnel with `BYBIT_API_BASE`. It sends
`ngrok-skip-browser-warning` so the interstitial does not intercept API calls,
and it names which of the two 403s it received.

```bash
export BYBIT_API_BASE="https://<tunnel>.ngrok-free.app"
tools/bybit-api balance      # confirms the whole chain in one call
```

A tunnel serving `api.bybit.com` carries your signed requests, which means it
sees the API key and every signature. Treat its URL as a secret and shut it
down when you are finished.

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
