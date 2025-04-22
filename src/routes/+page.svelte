<script lang="ts">
	import { onMount } from 'svelte';

	let transactions = [
		{
			date: '2025-04-08',
			description: 'Grocery Store',
			category: 'Groceries',
			amount: -89.47,
			status: 'Completed'
		},
		{
			date: '2025-04-10',
			description: 'Electric Bill',
			category: 'Utilities',
			amount: -120.3,
			status: 'Pending'
		}
		// Additional mock data can be added here
	];

	let editTransaction = null;

	interface Transaction {
		date: string;
		description: string;
		category: string;
		amount: number;
		status: string;
	}

	const openEditPopup = (transaction: Transaction) => {
		editTransaction = transaction;

		const editDate = document.getElementById('editDate') as HTMLInputElement | null;
		const editDescription = document.getElementById('editDescription') as HTMLInputElement | null;
		const editCategory = document.getElementById('editCategory') as HTMLInputElement | null;
		const editAmount = document.getElementById('editAmount') as HTMLInputElement | null;
		const editStatus = document.getElementById('editStatus') as HTMLSelectElement | null;
		const editPopup = document.getElementById('editPopup') as HTMLElement | null;

		if (editDate) editDate.value = transaction.date;
		if (editDescription) editDescription.value = transaction.description;
		if (editCategory) editCategory.value = transaction.category;
		if (editAmount) editAmount.value = transaction.amount.toString();
		if (editStatus) editStatus.value = transaction.status;
		if (editPopup) editPopup.classList.add('active');
	};

	const addFinance = (e: Event) => {
		e.preventDefault();
		const form = new FormData(e.target as HTMLFormElement);

		const date = form.get('date') as string;
		const description = form.get('description') as string;
		const category = form.get('category') as string;
		const amount = parseFloat(form.get('amount') as string);
		const status = form.get('status') as string;

		// Assuming list_id is always 1
		const list_id = 1;

		// Prepare the request body
		const requestBody = {
			date,
			description,
			category,
			amount,
			list_id
		};

		// Make a POST request to create the transaction
		fetch('http://localhost:3050/transaction/create', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify(requestBody)
		})
			.then((response) => response.json())
			.then((data) => {
				if (data.success) {
					fetch('http://localhost:3050/transaction/list/1')
						.then((response) => response.json())
						.then((data) => {
							if (data.success) {
								transactions = data.transactions;
							}
						})
						.catch((error) => {
							console.error('Error fetching transactions:', error);
						});

					alert('Transaction added successfully!');
					// Optionally, close the popup or reset the form here
				} else {
					alert('Failed to add transaction: ' + data.message);
				}
			})
			.catch((error) => {
				console.error('Error adding transaction:', error);
				alert('Failed to add transaction');
			});
	};

	onMount(() => {
		fetch('http://localhost:3050/transaction/list/1')
			.then((response) => response.json())
			.then((data) => {
				if (data.success) {
					transactions = data.transactions;
				}
			})
			.catch((error) => {
				console.error('Error fetching transactions:', error);
			});

		// Event listener for the edit button
		document.querySelectorAll('.edit-btn').forEach((button, index) => {
			button.addEventListener('click', () => {
				openEditPopup(transactions[index]);
			});
		});

		// Close the edit popup
		const closeEditPopup = document.getElementById('closeEditPopup');
		if (closeEditPopup) {
			closeEditPopup.addEventListener('click', () => {
				const editPopup = document.getElementById('editPopup');
				if (editPopup) {
					editPopup.classList.remove('active');
				}
			});
		}
	});

	let chatMessages: HTMLDivElement | null = null;
	let chatMessagesContent: string[] = [];

	function handleQuickSummary() {
        const chatMessages = document.getElementById('chatMessages');
        if (chatMessages) {
            chatMessages.innerHTML += `<div class="message user">Generating quick financial summary...</div>`;
        }

        // Generate a quick summary based on transactions
        const transactionSummary = transactions
            .map(
                (t) =>
                    `Date: ${t.date}, Description: ${t.description}, Category: ${t.category}, Amount: ${t.amount}, Status: ${t.status}`
            )
            .join('\n');

        const prompt = `
        Generate a quick financial summary based on the following transactions:

        ${transactionSummary}

        The currency is PHP. Provide a clean summary paragraph with insights such as total spending, spending by category, and suggestions for budgeting. Highlight important details.
        `;

        // Fetch the summary from the backend
        fetch('http://localhost:3050/gemini/generate', {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json'
            },
            body: JSON.stringify({
                text: prompt
            })
        })
            .then((response) => response.json())
            .then((result) => {
                if (result.success) {
                    const aiResponse =
                        result.data?.candidates?.[0]?.content?.parts?.[0]?.text || 'No response from AI.';
                    if (chatMessages) {
                        chatMessages.innerHTML += `<div class="message ai"><strong>AI:</strong> <p>${aiResponse}</p></div>`;
                    }
                } else {
                    if (chatMessages) {
                        chatMessages.innerHTML += `<div class="message error">Error: ${result.error}</div>`;
                    }
                }
            })
            .catch((error) => {
                console.error(error);
                if (chatMessages) {
                    chatMessages.innerHTML += `<div class="message error">Failed to reach the server.</div>`;
                }
            });
    }

async function handleSubmit(e: Event) {
	e.preventDefault();

	const userPromptInput = document.getElementById('userPrompt') as HTMLInputElement;
	const userPrompt = userPromptInput?.value || '';
	const chatMessages = document.getElementById('chatMessages');

	if (chatMessages) {
		chatMessages.innerHTML += `<div class="message user"><strong>User:</strong> ${userPrompt}</div>`;
		chatMessages.innerHTML += `<div class="message user">Generating financial report...</div>`;
	}

	const transactionSummary = transactions
		.map(
			(t) =>
				`Date: ${t.date}, Description: ${t.description}, Category: ${t.category}, Amount: ${t.amount}, Status: ${t.status}`
		)
		.join('\n');

	const prompt = `
	${userPrompt}

	Here are the transactions:
	${transactionSummary}

	The currency is PHP. Provide a clean summary paragraph with insights such as total spending, spending by category, and suggestions for budgeting. Highlight important details.
	`;

	try {
		const response = await fetch('http://localhost:3050/gemini/generate', {
			method: 'POST',
			headers: {
				'Content-Type': 'application/json'
			},
			body: JSON.stringify({
				text: prompt
			})
		});

		const result = await response.json();

		if (result.success) {
			const aiResponse =
				result.data?.candidates?.[0]?.content?.parts?.[0]?.text || 'No response from AI.';

			// Format the response as a clean paragraph
			const formattedResponse = aiResponse
				.replace(/"([^"]+)":/g, '<strong>$1:</strong>') // Highlight keys
				.replace(/\n/g, '<br>'); // Add line breaks for readability

			if (chatMessages) {
				chatMessages.innerHTML += `<div class="message ai"><strong>AI:</strong> <p>${formattedResponse}</p></div>`;
			}
		} else {
			if (chatMessages) {
				chatMessages.innerHTML += `<div class="message error">Error: ${result.error}</div>`;
			}
		}
	} catch (error) {
		console.error(error);
		if (chatMessages) {
			chatMessages.innerHTML += `<div class="message error">Failed to reach the server.</div>`;
		}
	}
}
</script>

<header>
	<p>Finance Tracker</p>
</header>
<div class="container">
	<!-- Data Overview -->
	<div class="data-overview">
		<h2>Data Overview</h2>
		<div class="actions">
			<select id="categoryFilter" class="category-filter">
				<option value="">All Categories</option>
				<option value="Groceries">Groceries</option>
				<option value="Utilities">Utilities</option>
				<!-- Categories will be dynamically added here -->
			</select>
			<button>Export</button>
			<button>Refresh</button>
		</div>
		<table>
			<thead>
				<tr>
					<th>Date</th>
					<th>Description</th>
					<th>Category</th>
					<th>Amount</th>
					<th>Actions</th>
				</tr>
			</thead>
			<tbody>
				{#each transactions as transaction}
					<tr>
						<td>{transaction.date}</td>
						<td>{transaction.description}</td>
						<td>{transaction.category}</td>
						<td>{transaction.amount}</td>
						<td><button class="edit-btn">Edit</button></td>
					</tr>
				{/each}
			</tbody>
		</table>
	</div>

	<!-- Add Finance Popup -->
	<div class="popup">
		<h2>Add Item</h2>
		<form id="addFinanceForm" on:submit={addFinance}>
			<div>
				<label for="date">Date</label>
				<input type="date" id="date" name="date" required />
			</div>
			<div>
				<label for="description">Description</label>
				<input type="text" id="description" name="description" required />
			</div>
			<div>
				<label for="category">Category</label>
				<input type="text" id="category" name="category" required />
			</div>
			<div>
				<label for="amount">Amount</label>
				<input type="number" id="amount" name="amount" required />
			</div>
			<div>
				<label for="status">Status</label>
				<select id="status" name="status" required>
					<option value="Completed">Completed</option>
					<option value="Pending">Pending</option>
				</select>
			</div>
			<button type="submit">Add</button>
		</form>
	</div>

	<!-- Edit Popup -->
	<div class="edit-popup" id="editPopup">
		<button class="close-btn" id="closeEditPopup">x</button>
		<h2>Edit</h2>
		<form id="editFinanceForm" class="editFinance">
			<div>
				<label for="editDate">Date</label>
				<input type="date" id="editDate" name="date" required />
			</div>
			<div>
				<label for="editDescription">Description</label>
				<input type="text" id="editDescription" name="description" required />
			</div>
			<div>
				<label for="editCategory">Category</label>
				<input type="text" id="editCategory" name="category" required />
			</div>
			<div>
				<label for="editAmount">Amount</label>
				<input type="number" id="editAmount" name="amount" required />
			</div>
			<div>
				<label for="editStatus">Status</label>
				<select id="editStatus" name="status" required>
					<option value="Completed">Completed</option>
					<option value="Pending">Pending</option>
				</select>
			</div>
			<button type="submit">Save</button>
		</form>
	</div>
</div>

<div class="ai-container">
    <div class="ai-integration">
        <div class="top-actions">
			<h2>fAInance</h2>
            <button type="button" class="quick-summary-btn" on:click={handleQuickSummary}>
                Quick Summary
            </button>
        </div>
        <div class="chat-box">
            <div id="chatMessages" class="messages">
				{#if chatMessagesContent.length === 0}
				<p class="placeholder-text">No messages yet. Start by entering a prompt or click "Quick Summary".</p>
			{/if}
			{@html chatMessagesContent.join('')}

			</div>
            <form id="chatForm" on:submit={handleSubmit}>
                <input
                    type="text"
                    id="userPrompt"
                    placeholder="Enter your custom prompt here..."
                    class="prompt-input"
                    required
                />
                <button type="submit">Send</button>
            </form>
        </div>
    </div>
</div>