<script>
	import { tick } from "svelte";
	import JSZip from "jszip";
	import FileSaver from "file-saver";
	import { Connection, clusterApiUrl, PublicKey } from "@solana/web3.js";

	// Setup the connection.
	let cnx = new Connection(clusterApiUrl("mainnet-beta"));

	// Toggle the loader modal.
	const toggleLoader = () => {
		document.getElementById(`modal`).classList.toggle(`hidden`);
	};

	// Just a little thing to update the modal text when needed.
	const updateLoader = (message) => {
		document.getElementById(`loader-message`).innerText = message;
	};

	// Sleep timer.
	const sleep = (ms) => {
		return new Promise((resolve) => setTimeout(resolve, ms));
	};

	// Method to split a list into chunks. We'll need this since we
	// can only fetch 200 signatures at a time.
	const splitArrayIntoChunksOfLen = (arr, len) => {
		let chunks = [],
			i = 0,
			n = arr.length;
		while (i < n) {
			chunks.push(arr.slice(i, (i += len)));
		}
		return chunks;
	};

	// Method to fetch all transaction IDs for a given account.
	let i = 0;
	const fetchTransactions = async (address) => {
		let before = null;
		let accountHistory = [];
		updateLoader(`\nFetching transactions for ${address}`);
		while (true) {
			let startingValue = i * 1000 + 1;
			let endingValue = i * 1000 + 1000;
			updateLoader(
				`Fetching transaction IDs ${startingValue} - ${endingValue}`
			);
			let accountHistoryBlock =
				await cnx.getConfirmedSignaturesForAddress2(
					new PublicKey(address),
					{ before: before }
				);
			if (accountHistoryBlock.length === 0) {
				break;
			}
			accountHistory = accountHistory.concat(
				accountHistoryBlock.map((i) => i.signature)
			);
			before =
				accountHistoryBlock[accountHistoryBlock.length - 1][
					"signature"
				];
			i += 1;
			await sleep(1000);
		}
		updateLoader(
			`Finished. Fetched ${accountHistory.length} transaction IDs.`
		);
		await sleep(1000);
		return accountHistory;
	};

	// Method to fetch bulk transaction data for given transaction IDs.
	let j = 1;
	const fetchTransactionsInfo = async (signatures) => {
		let transactions = [];
		const signatureChunks = splitArrayIntoChunksOfLen(signatures, 200);
		for (const signatureChunk of signatureChunks) {
			updateLoader(
				`Fetching transaction chunk ${j}/${signatureChunks.length}`
			);
			const transactionSet = await cnx.getParsedConfirmedTransactions(
				signatureChunk
			);
			transactions = transactions.concat(transactionSet);
			j++;
		}
		return transactions;
	};

	// Method to compile and compress the extracted data, then prompt user for download.
	const prepareData = async (address, data) => {
		updateLoader(`Compressing extracted data`);
		const zip = new JSZip();
		zip.file(`tx_${address}.json`, JSON.stringify(data));
		zip.generateAsync({ type: "blob" }).then(function (content) {
			toggleLoader();
			FileSaver.saveAs(content, `tx_${address}.zip`);
		});
	};

	// Here's the main method.
	const sendToExtract = async () => {
		toggleLoader();
		let address = document.getElementById(`address`).value;
		let txIds = await fetchTransactions(address);
		let txs = await fetchTransactionsInfo(txIds);
		await prepareData(address, txs);
	};

	// Add a listener to the input box, so when a user hits enter,
	// the button is pressed.
	const addEnterListener = async () => {
		await tick();
		const input = document.getElementById(`address`);
		input.addEventListener("keyup", function (event) {
			// Number 13 is the "Enter" key on the keyboard
			if (event.key === `Enter`) {
				// Cancel the default action, if needed
				event.preventDefault();
				// Trigger the button element with a click
				document.getElementById("btn-extract").click();
			}
		});
	};

	addEnterListener();
</script>

<main>
	<div id="modal" class="modal hidden">
		<div class="modal-window">
			<div class="loader">
				<div class="lds-ring">
					<div />
					<div />
					<div />
					<div />
				</div>
			</div>
			<div id="loader-message">Loading...</div>
		</div>
		<div class="modal-exit">X</div>
	</div>
	<div class="container">
		<div class="title">
			<h1>solextract.io</h1>
		</div>
		<input
			id="address"
			class="address"
			type="text"
			required
			placeholder="Your Solana account address..."
			autocorrect="off"
			autocapitalize="off"
			spellcheck="false"
		/>
		<div id="btn-extract" class="btn" on:click={sendToExtract}>
			extract me daddy
		</div>
	</div>
	<div class="disclaimer">
		This is an experimental tool. Use at your own risk.<br /><br />
		Created with ♡ by
		<a href="https://github.com/verata-veritatis/sol-extract"
			>verata-veritatis</a
		>.
	</div>
	<div class="logo" />
</main>

<style>
	@import url("https://fonts.googleapis.com/css2?family=DM+Mono:wght@300;500&display=swap");
	::placeholder {
		color: rgb(47, 54, 70);
	}
	main {
		height: 100%;
		text-align: center;
		margin: 0 auto;
	}
	h1 {
		color: rgb(0,255,163);
		font-family: "DM Mono", monospace;
		font-size: 4em;
		font-weight: 500;
		margin: 50px;
	}
	input {
		appearance: none;
		-webkit-appearance: none;
		border: none;
		outline: none;
		color: rgb(0,255,163);
	}
	.modal {
		position: fixed;
		display: flex;
		height: 100vh;
		width: 100vw;
		z-index: 999;
		background: rgba(0, 0, 0, 0.5);
		justify-content: center;
		align-items: center;
	}
	.modal-window {
		background: rgb(0,255,163);
		width: 550px;
		height: 300px;
		display: flex;
		align-items: center;
		justify-content: center;
		flex-direction: column;
		color: rgb(20 24 34);
		text-transform: uppercase;
		font-family: "DM Mono", monospace;
		font-size: 1.1em;
		font-weight: 500;
	}
	.modal-exit {
		position: absolute;
		top: 25px;
		right: 30px;
		color: rgb(0,255,163);
		font-size: 30px;
		cursor: pointer;
	}
	.hidden {
		pointer-events: none;
		opacity: 0;
	}
	.loader {
		margin: -5px 0 50px 0;
	}
	.title {
		margin-bottom: 100px;
		border: 2px solid rgb(0,255,163);
	}
	.container {
		display: flex;
		align-items: center;
		justify-content: center;
		flex-direction: column;
		height: 100%;
		font-family: "DM Mono", monospace;
		font-weight: 500;
	}
	.address {
		background: rgb(10, 13, 17);
		width: 650px;
		height: 40px;
		padding: 30px 30px;
	}
	.disclaimer {
		position: fixed;
		bottom: 50px;
		width: 100%;
		text-align: center;
		font-family: "DM Mono", monospace;
		color: rgba(255, 255, 255, 0.3);
		font-weight: 300;
		font-size: 14px;
	}
	.btn {
		display: flex;
		justify-content: center;
		align-items: center;
		cursor: pointer;
		margin: 50px;
		height: 50px;
		width: 250px;
		background-color: rgb(0,255,163);
		color: rgb(20 24 34);
		text-transform: uppercase;
		font-family: "DM Mono", monospace;
		font-size: 1.1em;
		transition: background-color 0.25s;
		font-weight: 500;
	}
	.btn:hover {
		background-color: rgb(3,225,255);
	}
	.lds-ring {
		display: inline-block;
		position: relative;
		width: 80px;
		height: 80px;
	}
	.lds-ring div {
		box-sizing: border-box;
		display: block;
		position: absolute;
		width: 64px;
		height: 64px;
		margin: 8px;
		border: 8px solid rgb(20 24 34);
		border-radius: 50%;
		animation: lds-ring 1.2s cubic-bezier(0.5, 0, 0.5, 1) infinite;
		border-color: rgb(20 24 34) transparent transparent transparent;
	}
	.lds-ring div:nth-child(1) {
		animation-delay: -0.45s;
	}
	.lds-ring div:nth-child(2) {
		animation-delay: -0.3s;
	}
	.lds-ring div:nth-child(3) {
		animation-delay: -0.15s;
	}
	@keyframes lds-ring {
		0% {
			transform: rotate(0deg);
		}
		100% {
			transform: rotate(360deg);
		}
	}
</style>
