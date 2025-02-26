# Contributing

Thank you for your interest in improving the Chainlink External Adapter codebase! The steps below support development of original adapters, but you are more than welcome to contribute to existing adapters as well. When opening a PR, please invite `smartcontractkit/solutions-engineering` to review the changes.

## Table of Contents

1. [Creating A New Adapter](#Creating-A-New-Adapter)
2. [Input](#Input)
3. [Output](#Output)
4. [Adding Provider API Rate Limits](#Adding-Provider-API-Rate-Limits)
5. [Mock Integration Testing](#Mock-Integration-Testing)
6. [Running Integration Tests](#Running-Integration-Tests)
7. [Generating Changesets](#Generating-Changesets)
8. [Common Patterns](#Common-Patterns)
9. [Soak Testing (Chainlink Labs)](<#Soak-Testing-(Chainlink-Labs)>)
10. [Logging Censorship](#logging-censorship)

## Creating A New Adapter

To get started use the `new` script with one argument:

```bash
$ yarn new [template-type]
```

This will start interactive command line interface, where you can provide additional information.

|    Parameter    |           Description           |             Options             |
| :-------------: | :-----------------------------: | :-----------------------------: |
| `template-type` | the name of the template to use | `composite`, `source`, `target` |

For example

```bash
$ yarn new source
```

_If on a Mac, this requires `gnu-sed` to be installed and set as the default for the command `sed`._

You can open a PR with the [New EA PR Template](./.github/PULL_REQUEST_TEMPLATE/new_ea_pr_template.md) by replacing `<branch>` in this URL: [https://github.com/smartcontractkit/external-adapters-js/compare/develop...<branch>?quick_pull=1&template=new_ea_pr_template.md](https://github.com/smartcontractkit/external-adapters-js/compare/develop...<branch>?quick_pull=1&template=new_ea_pr_template.md)

## Input

When flux monitor or OCR jobs from the core Chainlink node post to external adapters, the request body looks as follows:

```js
{
  "id": "2cae6a10e5184aa685c3428964b02418",
  "data": { "from": "ETH", "to": "USD" },
  "meta": {
    "latestAnswer": 39307000000,
    "updatedAt": 1616448197
  }
}
```

The `updatedAt` field is a unix timestamp representing when the `latestAnswer` was computed.

Optionally `data` parameters can also be passed via a query string added to the [Bridge](https://docs.chain.link/docs/node-operators) URL like: `{ENDPOINT}?from=ETH&to=USD`. This is useful when trying to conform to unified input parameters.

## Output

The External Adapter will do some processing, often request data from an API, and return the following response structure:

```js
  {
    "jobRunID": "2cae6a10e5184aa685c3428964b02418",
    "statusCode": 200,
    "data": {
      "result": 3000 // Result for Flux Monitor jobs.
      // ... Response data
    },
    "result": 3000, // Result for OCR jobs.
    "maxAge": 222, // [OPTIONAL] metadata for how long the request is cached for
    "debug": { // [OPTIONAL]
      // Additional metadata from the EA framework used for tracking metrics
    }
  }
```

## Adding Provider API Rate Limits

When adding a new adapter the tiers from that provider will need to be added to the [static configurations](packages/core/bootstrap/src/lib/provider-limits/limits.json) under the `NAME` given to the adapter.

## Mock Integration Testing

We use [Nock](https://github.com/nock/nock) for intercepting HTTP requests in integration tests and returning mock data.
The [recording](https://github.com/nock/nock#recording) functionality of nock is used when first writing the test to automatically generate accurate fixture data.

For example, take a look at the [synth-index](./packages/composites/synth-index/test/integration/adapter.test.ts) test to see it in usage. When the `RECORD` environment variable is truthy, nock will proxy HTTP requests and generate fixture data that can be used to contruct the integration test.

### Testing HTTP Requests

1. Setup nock to record HTTP requests, see the [synth-index](./packages/composites/synth-index/test/integration/adapter.test.ts) test for a code sample.
2. Run the test, using live API endpoints for the external adapter under test to hit, with nock recording on (`export RECORD=true`).
3. Using the generated fixture data from step 2, write a `fixtures.ts` file to return the mock data instead.
4. Run the tests again with nock recording disabled (`unset RECORD`). API requests should now be intercepted and mocked using the fixture data. Be sure to run tests with the `--updateSnapshot` flags to update the integration snapshot if necessary.

### Testing WebSocket Messages

1. Run your tests, using live API endpoints with recording on (`export RECORD=true`) and with a TTL for the connections long enough for the normal requests to go through, but lower than the jest timeout (this will depend on the test, but one example would be `export WS_SUBSCRIPTION_TTL=3000`)
2. You will see a log statement with the message "Recorded WebSocketMessages: {JSON Object}". This JSON object contains all the WebSocket messages sent and received, but they are not printed as code as in the case of the Nock features, due to their asynchronous nature.
3. Using the recorded messages, write a `fixtures.ts` file with "Exchanges", i.e. request and response(s) pairs that will be asserted as part of your test (see this [ncfx test fixtures example](./packages/sources/ncfx/test/integration/fixtures.ts)).
4. Write your tests (example in [ncfx adapter test](./packages/sources/ncfx/test/integration/adapter.test.ts)) using the helper functions from the [test-helpers](./packages/core/test-helpers/src/websocket.ts) package. Note the necessary setup performed in the `beforeAll` function. Also note the path for the `WebSocketClassProvider` import, it has to be that one due to the way Singleton patterns work (or rather don't work) across dependencies.
5. Finally, run your tests with recording disabled (`unset RECORD`). The WebSocket connection should be replaced and mocked.

For more information on Jest, see the [Jest docs](https://jestjs.io/docs/cli).

## Running Integration Tests

When running integration tests (for example `yarn test packages/sources/binance/test/integration`) make sure that metrics are disabled (`export METRICS_ENABLED=false`) and EA server is running on random available port (`export EA_PORT=0`).

## Generating Changesets

When making changes to an adapter, a changeset should be added for each feature or bug fix introduced. For each "unit" of changes, follow these steps to determine the packages affected, the version upgrade, and finally the changelog text. If you make a mistake, simply delete the files created in `.changeset/` and try again:

1. Run `yarn changeset` (from the root directory) to open a list of packages. You can filter packages by typing a string to match part of a package name (ex. type `coinm` to match `@chainlink/coinmarketcap-adapter` and `@chainlink/coinmetrics-adapter`). Use the `up` and `down` arrows to traverse the list and use `space` to select and unselect packages.
2. After selecting all packages affected by your change, press `enter` to determine the version level change for each package. Use `up`, `down`, and `space` to select the packages for each level of change, using `enter` to move through each level. This starts with `Major`, then goes to `Minor`, then any remaining unselected packages will have `Patch` applied.
3. In the final step, add a text summary that will be added to the `CHANGELOG.md` for every package when changesets are consumed.
4. Once the files in `.changeset/` have been created, add them to your branch to include them in the final PR.

## Common Patterns

- Use [BigNumber](https://github.com/MikeMcl/bignumber.js/) when operating on large integers
- Use [Decimal.js](https://github.com/MikeMcl/decimal.js/) for all floating point operations. `Decimal.js` uses a precision of 20 by default, but we may lose some precision with really large numbers, so please update to a higher precision before usage:

```js
Decimal.set({ precision: 100 })
```

- Handling "includes" in the request should be done with the following priority:
  1. Full-featured "includes" array (in the format of [presetIncludes.json](packages/core/bootstrap/src/lib/external-adapter/overrides/presetIncludes.json))
  2. Pre-set includes from the EA (set in [presetIncludes.json](packages/core/bootstrap/src/lib/external-adapter/overrides/presetIncludes.json))
  3. String array as "includes"

## Soak Testing (Chainlink Labs)

In order to soak test adapters we need to create and push the adapter out to the sdlc cluster. From there we can use the Flux Emulator or K6 to send traffic to it for the amount of time you need.

Prerequisites to starting an external adapter in the sdlc cluster

1. You must be on the vpn to access the k8s cluster.
2. You must have your kubeconfig set to the sdlc cluster which also requires you be logged into the aws secure-sdlc account as a user with k8s permissions. To do so it would look something like this but with your specific profile name `aws sso login --profile sdlc`. Instructions to set this up and set your kubeconfig can be found in [QA Kubernetes Cluster](https://www.notion.so/chainlink/QA-Kubernetes-Cluster-ca3f1a64e6fd4476ac5a76c8bfcd8624)
3. In order to pull the external adapter helm chart you need to have a GitHub PAT and add the chainlik helm repo using the instructions in [smartcontractkit/charts](https://github.com/smartcontractkit/charts)

To spin up an adapter in the sdlc cluster:

```bash
# Build all packages
yarn install
yarn setup

# Build the docker-compose
# The uniqueName can be your name or something unique to you, for example in ci it will use the PR number
# Change the adapter name to the adapter you are testing
export AWS_PROFILE=sdlc
export AWS_REGION=us-west-2
export UNIQUE_NAME=unique-name
export ADAPTER_NAME=coingecko
export IMAGE_PREFIX=795953128386.dkr.ecr.us-west-2.amazonaws.com/adapters/
export IMAGE_TAG=pr${UNIQUE_NAME}
IMAGE_TAG=${IMAGE_TAG} IMAGE_PREFIX=${IMAGE_PREFIX} yarn generate:docker-compose

# Build the docker image
docker-compose -f docker-compose.generated.yaml build ${ADAPTER_NAME}-adapter

# Push adapter image to private ecr
# If you haven't logged into the docker repository you may need to do this before the push will work
# aws ecr get-login-password --region ${AWS_REGION} --profile ${AWS_PROFILE} | docker login --username AWS --password-stdin ${IMAGE_PREFIX}
# If you need to create a repository for a new adapter it can be done like so:
# aws ecr create-repository --region ${AWS_REGION} --profile ${AWS_PROFILE} --repository-name adapters/${ADAPTER_NAME} || true
docker push ${IMAGE_PREFIX}${ADAPTER_NAME}-adapter:${IMAGE_TAG}

# Start the adapter in the sdlc cluster
yarn qa:adapter start ${ADAPTER_NAME} ${UNIQUE_NAME} ${IMAGE_TAG}
```

To tear down the deployment made above after you are done testing:

```bash
yarn qa:adapter stop ${ADAPTER_NAME} ${UNIQUE_NAME} ${UNIQUE_NAME}
```

To start running a test via Flux Emulator:

```bash
# Use the same unique and adapter name from when you started the adapter
export UNIQUE_NAME=unique-name
export ADAPTER_NAME=coingecko
yarn qa:flux:configure start ${ADAPTER_NAME} ${UNIQUE_NAME}
```

To stop running a test via Flux Emulator:

```bash
yarn qa:flux:configure stop ${ADAPTER_NAME} ${UNIQUE_NAME}
```

To build a K6 payload file from the Flux Emulator config on WeiWatchers:

```bash
yarn qa:flux:configure k6payload ${ADAPTER_NAME} ${UNIQUE_NAME}
```

To start a test using k6 and the generated payload. Note read the k6 readme ./packages/k6/README.md It contains more information on how to configure the test to point to the adapter you have deployed among other things.

```bash
export UNIQUE_NAME=unique
export ADAPTER_NAME=coingecko
# create the config
yarn qa:flux:configure k6payload ${ADAPTER_NAME} ${UNIQUE_NAME}

# Move to the k6 package and build/push
UNIQUE_NAME=${UNIQUE_NAME} ./packages/k6/buildAndPushImage.sh

# start the test pod
UNIQUE_NAME=${UNIQUE_NAME} ADAPTER=${ADAPTER_NAME} ./packages/k6/start.sh
```

To stop or tear down a test using k6 in the cluster do the below.

```bash
UNIQUE_NAME=${UNIQUE_NAME} ADAPTER=${ADAPTER} ./packages/k6/stop.sh
```

When you are done testing please remember to tear down any adapters and k6 deployments in the cluster. If you used the same UNIQUE_NAME for all of the above you can clean up both the adapters and the k6 tests with this:

```bash
PR_NUMBER=${UNIQUE_NAME} ./packages/scripts/src/ephemeral-adapters/cleanup.sh
```

### Output testing

Soak testing additionally can test the responses output. The output testing runs against assertions placed in `./packages/k6/src/config/assertions`. Common assertions are in `assertions.json`, adapter-specific ones will be loaded from `${adapterName}-assertions.json`. Assertions can be applied for all the requests or specific set of parameters. See examples in the folder.

The output testing checks the variety of input parameters, should be at least 10. For new adapters various parameters should be defined in `test-payload.json`.

Input parameters in `test-payload.json` should include the following pairs:

**High/low volume pairs**

```
[{"from": "OGN", "to": "ETH"}, {"from": "CSPR", "to": "USD"}, {"from": "CTSI", "to": "ETH"}, {"from": "BADGER", "to": "ETH"}, {"from": "BADGER", "to": "USD"}, {"from": "BSW", "to": "USD"}, {"from": "KNC", "to": "USD"}, {"from": "KNC", "to": "USD"}, {"from": "QUICK", "to": "ETH"}, {"from": "QUICK", "to": "USD"}, {"from": "FIS", "to": "USD"}, {"from": "MBOX", "to": "USD"}, {"from": "FOR", "to": "USD"}, {"from": "DGB", "to": "USD"}, {"from": "SUSD", "to": "ETH"}, {"from": "SUSD", "to": "USD"}, {"from": "WIN", "to": "USD"}, {"from": "SRM", "to": "ETH"}, {"from": "SRM", "to": "USD"}, {"from": "ALCX", "to": "ETH"}, {"from": "ALCX", "to": "USD"}, {"from": "ADX", "to": "USD"}, {"from": "NEXO", "to": "USD"}, {"from": "ANT", "to": "ETH"}, {"from": "ANT", "to": "USD"}, {"from": "LOOKS", "to": "USD"}, {"from": "WNXM", "to": "USD"}, {"from": "MDX", "to": "USD"}, {"from": "ALPHA", "to": "BNB"}, {"from": "ALPHA", "to": "USD"}, {"from": "NMR", "to": "ETH"}, {"from": "NMR", "to": "USD"}, {"from": "PHA", "to": "USD"}, {"from": "OHM", "to": "ETH"}, {"from": "BAL", "to": "ETH"}, {"from": "BAL", "to": "USD"}, {"from": "CVX", "to": "USD"}, {"from": "MOVR", "to": "USD"}, {"from": "DEGO", "to": "USD"}, {"from": "USDD", "to": "USD"}, {"from": "QI", "to": "USD"}, {"from": "MIM", "to": "USD"}, {"from": "KP3R", "to": "ETH"}, {"from": "REP", "to": "ETH"}, {"from": "REP", "to": "USD"}, {"from": "WING", "to": "USD"}, {"from": "XVS", "to": "BNB"}, {"from": "XVS", "to": "USD"}, {"from": "BOND", "to": "ETH"}, {"from": "BOND", "to": "USD"}, {"from": "REQ", "to": "USD"}, {"from": "LEO", "to": "USD"}, {"from": "ORN", "to": "ETH"}, {"from": "ALPACA", "to": "USD"}, {"from": "AGEUR", "to": "USD"}, {"from": "VAI", "to": "USD"}, {"from": "BIFI", "to": "USD"}, {"from": "AUTO", "to": "USD"}, {"from": "CEL", "to": "ETH"}, {"from": "CEL", "to": "USD"}, {"from": "HT", "to": "USD"}, {"from": "GLM", "to": "USD"}, {"from": "OM", "to": "USD"}, {"from": "BIT", "to": "USD"}, {"from": "ERN", "to": "USD"}, {"from": "PLA", "to": "USD"}, {"from": "FARM", "to": "ETH"}, {"from": "FARM", "to": "USD"}, {"from": "FORTH", "to": "USD"}, {"from": "OXT", "to": "USD"}, {"from": "MLN", "to": "ETH"}, {"from": "MLN", "to": "USD"}, {"from": "ONG", "to": "USD"}, {"from": "GUSD", "to": "ETH"}, {"from": "GUSD", "to": "USD"}, {"from": "DFI", "to": "USD"}, {"from": "SWAP", "to": "ETH"}, {"from": "ALUD", "to": "USD"}, {"from": "CREAM", "to": "BNB"}, {"from": "CREAM", "to": "USD"}, {"from": "GNO", "to": "ETH"}, {"from": "XAVA", "to": "USD"}, {"from": "BOO", "to": "USD"}, {"from": "AMPL", "to": "USD"}, {"from": "AMPL", "to": "USD"}, {"from": "RAI", "to": "ETH"}, {"from": "RAI", "to": "USD"}, {"from": "FEI", "to": "ETH"}, {"from": "FEI", "to": "USD"}, {"from": "DPX", "to": "USD"}, {"from": "EURT", "to": "USD"}, {"from": "LON", "to": "ETH"}, {"from": "BORING", "to": "BNB"}, {"from": "BORING", "to": "USD"}, {"from": "RARI", "to": "ETH"}, {"from": "DNT", "to": "ETH"}, {"from": "FOX", "to": "USD"}, {"from": "XHV", "to": "USD"}, {"from": "TRIBE", "to": "ETH"}, {"from": "UMEE", "to": "ETH"}, {"from": "UST", "to": "ETH"}, {"from": "UST", "to": "USD"}, {"from": "ZCN", "to": "USD"}, {"from": "DPI", "to": "USD"}]
```

**Collision risks**

```
[{"from": "MIM", "to": "USD"}, {"from": "LRC", "to": "USD"}, {"from": "OHM", "to": "USD"}, {"from": "QUICK", "to": "USD"}]
```

**Forex feeds**

```
[{"from": "AED", "to": "USD"}, {"from": "AUD", "to": "USD"}, {"from": "BRL", "to": "USD"}, {"from": "CAD", "to": "USD"}, {"from": "CHF", "to": "USD"}, {"from": "CNY", "to": "USD"}, {"from": "COP", "to": "USD"}, {"from": "CZK", "to": "USD"}, {"from": "EUR", "to": "USD"}, {"from": "GBP", "to": "USD"}, {"from": "HKD", "to": "USD"}, {"from": "IDR", "to": "USD"}, {"from": "ILS", "to": "USD"}, {"from": "INR", "to": "USD"}, {"from": "JPY", "to": "USD"}, {"from": "KRW", "to": "USD"}, {"from": "MXN", "to": "USD"}, {"from": "NZD", "to": "USD"}, {"from": "PHP", "to": "USD"}, {"from": "PLN", "to": "USD"}, {"from": "SEK", "to": "USD"}, {"from": "SGD", "to": "USD"}, {"from": "THB", "to": "USD"}, {"from": "TRY", "to": "USD"}, {"from": "TZS", "to": "USD"}, {"from": "VND", "to": "USD"}, {"from": "XAG", "to": "USD"}, {"from": "XAU", "to": "USD"}, {"from": "XPT", "to": "USD"}, {"from": "ZAR", "to": "USD"}, {"from": "ZEC", "to": "USD"}, {"from": "ZIL", "to": "USD"}, {"from": "ZRX", "to": "ETH"}, {"from": "ZRX", "to": "USD"}]
```

Available types of assertions:

- `minPrecision` - minimum precision for a numeric value
- `greaterThan` - minimum numeric value
- `lessThan` - maximum numeric value
- `minItems` - list contains at least the required number of items
- `contains` - list contains a specific item (string or number)
- `hasKey` - an object contains a specific key

## Logging Censorship

If you are introducing a new env var that contains sensitive data, ensure that it is added to our logging configurations to help censor it in the logs. Follow the steps below to do so.

In the case that the env var's value it not altered by the adapter such as Base64 encoding, add the env var to the `configRedactEnvVars` list in packages/core/bootstrap/src/lib/config/logging.ts

In the case that the env var's value is altered in the adapter prior to use, add the JSON path of the key containing the sensitive value (i.e. `config.api.apiKey`) to the `redactPaths` list in packages/core/bootstrap/src/lib/config/logging.ts

As an example, the path `config.api.apiKey` would alter logs in the following way:

```js
{
  "config": {
    ...
    "api": {
      ...
      "apiKey": "[REDACTED]"
    }
  }
}
```
