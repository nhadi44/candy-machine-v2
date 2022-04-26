<script lang="ts">
  import { candyMachineState, userState } from "../lib/store";
  import { mintOneToken } from "../lib/mint";
  import {
    getUserBalance,
    connectWallet,
    existsOwnerSPLToken,
  } from "../lib/state-helpers";
  import { web3 } from "@project-serum/anchor";
  import { awaitTransactionSignatureConfirmation } from "../lib/connection";
  import confetti from "canvas-confetti";
  import { LAMPORTS_PER_SOL } from "@solana/web3.js";
  import { onMount } from "svelte";

  const txTimeout = 30000;
  const cluster = import.meta.env.VITE_APP_SOLANA_NETWORK?.toString();
  const apiUrl = import.meta.env.VITE_APP_API_URL?.toString();
  const secretKey = import.meta.env.VITE_APP_SECRET_KEY?.toString();

  $: date = new Date($candyMachineState?.state.goLiveDate?.toNumber() * 1000);
  $: whitelistPrice =
    $candyMachineState?.state.whitelistMintSettings?.discountPrice;
  $: isActive = $candyMachineState?.state.isActive;
  $: isSoldOut = $candyMachineState?.state.isSoldOut;
  $: whitelist = $candyMachineState?.state.whitelistMintSettings;
  $: userWhitelisted = $userState.isWhiteListed;
  $: price = $candyMachineState?.state.price;
  $: nftPrice = () => {
    const discountPrice =
      $candyMachineState?.state?.whitelistMintSettings?.discountPrice?.toNumber();
    const normalPrice = $candyMachineState?.state?.price.toNumber() ?? 0;
    if ($userState.isWhiteListed && discountPrice) {
      return discountPrice;
    } else {
      return normalPrice;
    }
  };

  let isMinting = false;
  let mintSuccessful = false;
  let isWhiteListed = true;
  let { solana } = window as any;
  export let connection;

  async function connectWalletButton() {
    $userState.walletPublicKey = await connectWallet(solana);
    if ($userState.walletPublicKey) {
      // Get User Balance
      $userState.userBalance = await getUserBalance(
        $userState.walletPublicKey,
        connection
      );
      // Check if user is whitelisted (ie. check if they have token)
      if ($candyMachineState.state.whitelistMintSettings) {
        $userState.isWhiteListed = await existsOwnerSPLToken(
          $userState.walletPublicKey,
          connection,
          $candyMachineState.state.whitelistMintSettings?.mint
        );
      }
    }
  }

  async function test() {
    const res = await fetch(`${apiUrl}/whitelisted`);
    $userState.walletPublicKey = await connectWallet(solana);
    const json = await res.json();
    console.log(
      new Date($candyMachineState?.state.goLiveDate?.toNumber() * 1000)
    );
    for (let i = 0; i < json.length; i++) {
      if (json[i].member === $userState.walletPublicKey) {
        isMinting = false;
        // isActive = true;
      } else {
        // isActive = false;
      }
    }
  }

  async function mintWhitelisted() {
    try {
      $userState.walletPublicKey = await connectWallet(solana);
      const res = await fetch(
        `${apiUrl}/whitelisted/member/${$userState.walletPublicKey}`,
        { method: "GET" }
      );
      const json = await res.json();
      console.log(json.reserve);
      if (!json.member) {
        throw new Error("User is not whitelisted");
      }

      if (!json.reserve) {
        throw new Error("User does not have enough tokens");
      }

      isMinting = true;
      if ($candyMachineState?.program && $userState.walletPublicKey) {
        const mint = web3.Keypair.generate();
        const mintTxId = (
          await mintOneToken(
            $candyMachineState,
            new web3.PublicKey($userState.walletPublicKey),
            mint
          )
        )[0];
        let status: any = { err: true };
        if (mintTxId) {
          status = await awaitTransactionSignatureConfirmation(
            mintTxId,
            txTimeout,
            connection,
            "singleGossip",
            true
          );
        }
        if (!status?.err) {
          const to_send = (await json.reserve) - 1;
          await fetch(
            `${apiUrl}/whitelisted/update/${$userState.walletPublicKey}/${secretKey}`,
            {
              method: "PUT",
              mode: "cors",
              headers: {
                "Content-Type": "application/json",
              },
              body: to_send.toString(),
            }
          );
          console.log("Success");
          displaySuccess(mint.publicKey);
        } else {
          console.error("An error occurred");
        }
      }
    } catch (error: any) {
      console.error("An error occurred ", error);
    } finally {
      isMinting = false;
    }
  }

  async function mint() {
    try {
      isMinting = true;
      if ($candyMachineState?.program && $userState.walletPublicKey) {
        const mint = web3.Keypair.generate();
        const mintTxId = (
          await mintOneToken(
            $candyMachineState,
            new web3.PublicKey($userState.walletPublicKey),
            mint
          )
        )[0];
        let status: any = { err: true };
        if (mintTxId) {
          status = await awaitTransactionSignatureConfirmation(
            mintTxId,
            txTimeout,
            connection,
            "singleGossip",
            true
          );
        }
        if (!status?.err) {
          console.log("Success");
          displaySuccess(mint.publicKey);
        } else {
          console.error("An error occurred");
        }
      }
    } catch (error: any) {
      console.error("An error occurred ", error);
    } finally {
      isMinting = false;
    }
  }

  function displaySuccess(mintPublicKey: web3.PublicKey) {
    // It takes time before state change is cascaded across the network
    // so we update this variables manually upon successful completion
    $candyMachineState.state.itemsRedeemed += 1;
    mintSuccessful = true;
    $userState.solanaExplorerLink =
      cluster == "devnet"
        ? `https://explorer.solana.com/address/${mintPublicKey}?cluster=devnet`
        : `https://explorer.solana.com/address/${mintPublicKey}`;
    throwConfetti();
  }

  function throwConfetti(): void {
    confetti({
      particleCount: 200,
      spread: 70,
      origin: { y: 0.6 },
    });
  }

  function getPhantomWallet() {
    // Here we check for the solana object again.
    // If its present, reload the page so state is refreshed.
    solana = (window as any).solana;
    if (solana) {
      location.reload();
    } else {
      window.open("https://phantom.app/", "_blank");
    }
  }

  async function testPost() {
    // const to_send = (await json.reserve) - 1;
    $userState.walletPublicKey = await connectWallet(solana);
    const data = await fetch(
      `${apiUrl}/whitelisted/member/${$userState.walletPublicKey}`,
      { method: "GET" }
    );
    const json = await data.json();
    const to_send = json.reserve - 1;
    const res = await fetch(
      `${apiUrl}/whitelisted/update/${$userState.walletPublicKey}/${secretKey}`,
      {
        method: "PUT",
        mode: "cors",
        headers: {
          "Content-Type": "application/json",
        },
        body: to_send.toString(),
      }
      // { method: "GET" }
    );
    console.log(await res.json);
  }

  onMount(() => {
    solana = (window as any).solana;
    test();
  });
</script>

<div class="flex flex-col">
  {#if !solana}
    <button
      class=" px-3 py-2 rounded-md  bg-sky-600  hover:bg-sky-700 text-white font-bold"
      on:click={() => getPhantomWallet()}>Get Phantom Wallet</button
    >
  {:else if !$userState.walletPublicKey}
    <button
      class=" px-3 py-2 rounded-md  bg-sky-600  hover:bg-sky-700 text-white font-bold"
      on:click={connectWalletButton}>Connect</button
    >
  {:else if isSoldOut}
    <button
      class=" px-3 py-2 rounded-md  bg-sky-600  hover:bg-sky-700 text-white font-bold "
      >Sold Out!</button
    >
  {:else if !isActive && whitelist?.presale}
    <!-- Mint not active, presale enabled but user not whitelisted -->
    <button class=" btn-black" disabled={true}
      >Whitelist Presale Access Only</button
    >
  {:else if !isActive && !whitelist?.presale}
    <!-- Mint is not active and not a presale -->
    <button class=" btn-black" disabled={true}
      >Mint Live @ {date.toUTCString()}</button
    >
  {:else if $userState.userBalance < nftPrice()}
    <button
      class=" px-3 py-2 rounded-md  bg-sky-600  hover:bg-sky-700 text-white font-bold disabled:bg-gray-400"
      disabled={true}
      >Insufficient Funds ({(nftPrice() / LAMPORTS_PER_SOL).toFixed(2)} SOL required)</button
    >
    <div />
    <!-- {:else if isWhiteListed}
    <button
      class=" px-3 py-2 rounded-md  bg-sky-600  hover:bg-sky-700 text-white font-bold disabled:bg-gray-400"
      disabled={isMinting}
      on:click={mint}
    >
      {#if isMinting}
        <span>Minting ...</span>
      {:else if mintSuccessful}
        <span>Mint succesful! Mint another?</span>
      {:else}
        <span
          >Whitelist Mint ({(nftPrice() / LAMPORTS_PER_SOL).toFixed(2)} SOL)</span
        >
      {/if}
    </button> -->
  {:else}
    <button
      class=" px-3 py-2 rounded-md  bg-sky-600  hover:bg-sky-700 text-white font-bold disabled:bg-gray-400"
      disabled={isMinting}
      on:click={mintWhitelisted}
    >
      {#if isMinting}
        <span>Minting ...</span>
      {:else if mintSuccessful}
        <span>Mint succesful! Mint another?</span>
      {:else}
        <span>Mint ({(nftPrice() / LAMPORTS_PER_SOL).toFixed(2)} SOL)</span>
      {/if}
    </button>
  {/if}
  <button class="px-3 bg-red-500 text-white my-4" on:click={testPost}
    >Test post</button
  >
</div>
