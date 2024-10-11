import { config as dotenv } from "dotenv";
import {
  createWalletClient,
  http,
  getContract,
  erc20Abi,
  parseUnits,
  maxUint256,
  publicActions,
  concat,
  numberToHex,
  size,
} from "viem";
import type { Hex } from "viem";
import { privateKeyToAccount } from "viem/accounts";
import { scroll } from "viem/chains";
import { wethAbi } from "./abi/weth-abi";
import qs from "qs";

// Load environment variables
dotenv();
const { PRIVATE_KEY, ZERO_EX_API_KEY, ALCHEMY_HTTP_TRANSPORT_URL } =
  process.env;

// Validate requirements
if (!PRIVATE_KEY) throw new Error("missing PRIVATE_KEY.");
if (!ZERO_EX_API_KEY) throw new Error("missing ZERO_EX_API_KEY.");
if (!ALCHEMY_HTTP_TRANSPORT_URL)
  throw new Error("missing ALCHEMY_HTTP_TRANSPORT_URL.");

// Setup wallet client
const client = createWalletClient({
  account: privateKeyToAccount(`0x${PRIVATE_KEY}` as `0x${string}`),
  chain: scroll,
  transport: http(ALCHEMY_HTTP_TRANSPORT_URL),
}).extend(publicActions);

const [address] = await client.getAddresses();

// Set up contracts
const weth = getContract({
  address: "0x5300000000000000000000000000000000000004",
  abi: wethAbi,
  client,
});
const wsteth = getContract({
  address: "0xf610A9dfB7C89644979b4A0f27063E9e7d7Cda32",
  abi: erc20Abi,
  client,
});

// Function to display the percentage breakdown of liquidity sources
function displayLiquiditySources(route: any) {
  const fills = route.fills;
  const totalBps = fills.reduce((acc: number, fill: any) => acc + parseInt(fill.proportionBps), 0);

  console.log(`${fills.length} Sources`);
  fills.forEach((fill: any) => {
    const percentage = (parseInt(fill.proportionBps) / 100).toFixed(2);
    console.log(`${fill.source}: ${percentage}%`);
  });
}

// Function to display the buy/sell taxes for tokens
function displayTokenTaxes(tokenMetadata: any) {
  const buyTokenBuyTax = (parseInt(tokenMetadata.buyToken.buyTaxBps) / 100).toFixed(2);
  const buyTokenSellTax = (parseInt(tokenMetadata.buyToken.sellTaxBps) / 100).toFixed(2);
  const sellTokenBuyTax = (parseInt(tokenMetadata.sellToken.buyTaxBps) / 100).toFixed(2);
  const sellTokenSellTax = (parseInt(tokenMetadata.sellToken.sellTaxBps) / 100).toFixed(2);

  if (buyTokenBuyTax > 0 || buyTokenSellTax > 0) {
    console.log(`Buy Token Buy Tax: ${buyTokenBuyTax}%`);
    console.log(`Buy Token Sell Tax: ${buyTokenSellTax}%`);
  }

  if (sellTokenBuyTax > 0 || sellTokenSellTax > 0) {
    console.log(`Sell Token Buy Tax: ${sellTokenBuyTax}%`);
    console.log(`Sell Token Sell Tax: ${sellTokenSellTax}%`);
  }
}

// Fetch liquidity sources from Scroll
async function getLiquiditySources() {
  const chainId = client.chain.id.toString();
  const sourcesParams = new URLSearchParams({ chainId });

  try {
    const response = await fetch(
      `https://api.0x.org/swap/v1/sources?${sourcesParams.toString()}`,
      { headers: getHeaders() }
    );
    const sourcesData = await response.json();
    const sources = Object.keys(sourcesData.sources);
    console.log("Liquidity sources for Scroll chain:", sources.join(", "));
  } catch (error) {
    console.error("Error fetching liquidity sources:", error);
  }
}

// Helper function to get request headers
function getHeaders() {
  return new Headers({
    "Content-Type": "application/json",
    "0x-api-key": ZERO_EX_API_KEY,
    "0x-version": "v2",
  });
}

// Main function to fetch price, quote, and handle the transaction
async function main() {
  await getLiquiditySources();

  const decimals = (await weth.read.decimals()) as number;
  const sellAmount = parseUnits("0.1", decimals);
  const affiliateFeeBps = "100"; // 1%
  const surplusCollection = "true";

  // Fetch price
  const price = await fetchPrice(sellAmount, affiliateFeeBps, surplusCollection);
  if (price?.issues?.allowance) {
    await handleAllowance(price);
  }

  const quote = await fetchQuote(priceParams);
  displayLiquiditySources(quote.route);
  displayTokenTaxes(quote.tokenMetadata);

  await handleTransaction(quote);
}

// Fetch price with monetization parameters
async function fetchPrice(sellAmount, affiliateFeeBps, surplusCollection) {
  const priceParams = new URLSearchParams({
    chainId: client.chain.id.toString(),
    sellToken: weth.address,
    buyToken: wsteth.address,
    sellAmount: sellAmount.toString(),
    taker: client.account.address,
    affiliateFee: affiliateFeeBps,
    surplusCollection: surplusCollection,
  });

  try {
    const response = await fetch(
      `https://api.0x.org/swap/permit2/price?${priceParams.toString()}`,
      { headers: getHeaders() }
    );
    return await response.json();
  } catch (error) {
    console.error("Error fetching price:", error);
  }
}

// Handle setting allowance for Permit2 if necessary
async function handleAllowance(price) {
  try {
    const { request } = await weth.simulate.approve([price.issues.allowance.spender, maxUint256]);
    console.log("Approving Permit2 to spend WETH...");
    const hash = await weth.write.approve(request.args);
    console.log("Approved Permit2 to spend WETH.", await client.waitForTransactionReceipt({ hash }));
  } catch (error) {
    console.error("Error approving Permit2:", error);
  }
}

// Fetch quote for the swap
async function fetchQuote(priceParams) {
  const quoteParams = new URLSearchParams(priceParams);
  try {
    const response = await fetch(
      `https://api.0x.org/swap/permit2/quote?${quoteParams.toString()}`,
      { headers: getHeaders() }
    );
    return await response.json();
  } catch (error) {
    console.error("Error fetching quote:", error);
  }
}

// Handle the transaction with permit2 signature
async function handleTransaction(quote) {
  if (quote.permit2?.eip712) {
    try {
      const signature = await client.signTypedData(quote.permit2.eip712);
      console.log("Signed permit2 message");
      quote.transaction.data = appendSignatureToData(quote.transaction.data, signature);
      await sendTransaction(quote.transaction);
    } catch (error) {
      console.error("Error signing permit2:", error);
    }
  } else {
    console.error("Failed to obtain a signature, transaction not sent.");
  }
}

// Append signature to transaction data
function appendSignatureToData(transactionData, signature) {
  const signatureLengthInHex = numberToHex(size(signature), { signed: false, size: 32 });
  return concat([transactionData, signatureLengthInHex, signature]);
}

// Send the transaction
async function sendTransaction(transaction) {
  const nonce = await client.getTransactionCount({ address: client.account.address });
  const signedTransaction = await client.signTransaction({
    account: client.account,
    chain: client.chain,
    gas: BigInt(transaction.gas || 0),
    to: transaction.to,
    data: transaction.data,
    value: transaction.value ? BigInt(transaction.value) : undefined,
    gasPrice: transaction.gasPrice ? BigInt(transaction.gasPrice) : undefined,
    nonce: nonce,
  });

  const hash = await client.sendRawTransaction({ serializedTransaction: signedTransaction });
  console.log(`Transaction hash: ${hash}`);
  console.log(`See tx details at https://scrollscan.com/tx/${hash}`);
}

main();
