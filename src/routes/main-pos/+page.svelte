<script lang="ts">
	import Sidebar from '../sidebar/+page.svelte';
	import { orderedItemsStore } from '../../stores/orderedItemsStore';
	import { onMount, onDestroy } from 'svelte';
	import { handleButtonClick } from '../../utils/buttonHandler'; // Import the reusable function
	import { currentInputStore } from '../../stores/currentInputStore'; // Import the store
	import { WindowsSolid } from 'flowbite-svelte-icons';
	import { fade } from 'svelte/transition'; // Import the missing transition

	let amountPaid = '₱0.00';
	let cashierName = '';
	let staffToken: string | null = null; // Declare a variable to hold the staff_token
	let currentTime: string;
	let currentDay: string;
	let orderedItems: OrderedItem[] = []; // Declare and initialize orderedItems
	let isTakeOut = false; // Declare and initialize isTakeOut
	let isDineIn = false; // Declare and initialize isDineIn
	let queuedOrders: QueuedOrder[] = []; // Use the defined type for queuedOrders
	let orders: any[] = []; // Declare a variable to hold fetched orders
	let isReservePopupVisible = false; // State variable to control the visibility of the reserve popup
	let tableStatus: { [key: string]: boolean } = {};
	let isReservePopup2Visible = false; // State variable to control the visibility of the reserve popup
	let isReceiptPopupVisible = false;

	let reserveDate = '';
	let reserveTime = '';
	let selectedTableNumber = '';
	let orderNumber = '';
	let selectedCategory = 'All';
	let payment = '';
	let isPopupVisible = false;
	let selectedTable = ''; // Add variable for selected table in receipt

	let isCodePopupVisible = false;
	let voidIndex: number | null = null;
	let inputCode = '';
	let voidReceiptNumber = ''; // Added to store receipt number for voiding
	let voidItemIndex = -1; // Added to store item index for voiding
	let adminPassword = '122226'; // Set the admin password (this should eventually come from a secure backend)

	let reservedTables: string[] = []; // Declare an array to hold reserved table numbers

	type QueuedOrder = {
		que_order_no: string;
		receipt_number: string;
		date: string;
		time: string;
		table_number: string;
		order_status: string;
		items_ordered: string;
		total_amount: number;
		basePrice: number;
	};

	let totalOrderedItemsPrice = 0; // Variable to hold the total price of ordered items
	let voucherCode = ''; // Declare the voucherCode variable
	let voucherDiscount = 0; // Declare the voucherDiscount variable

	// Define the Voucher type
	type Voucher = {
		code: string;
		discount: number;
		voucher_discount: string;
		voucher_code: string;
		// Add other fields as necessary
	};

	// Function to calculate total price of ordered items
	function calculateTotalOrderedItemsPrice() {
		let addonsPrice = 0; // Declare addonsPrice here
		totalOrderedItemsPrice = orderedItems.reduce((total, item) => {
			// Skip voided items if they're marked as voided
			if (item.voided) return total;
			
			// Calculate total addons price
			addonsPrice +=
				parseFloat(String(item.order_addons_price || 0)) +
				parseFloat(String(item.order_addons_price2 || 0)) +
				parseFloat(String(item.order_addons_price3 || 0));

			// Calculate total for this item
			const itemTotal = parseFloat(String(item.basePrice || 0)); // Ensure this is a number

			// Return the accumulated total
			return total + itemTotal; // Sum the item total
		}, 0);

		totalOrderedItemsPrice += addonsPrice; // Add the total addons price

		// Log the total ordered items price
		console.log('Total Ordered Items Price:', totalOrderedItemsPrice.toFixed(2)); // Log the total price
	}

	// Call this function whenever orderedItems changes
	$: calculateTotalOrderedItemsPrice();

	// Add a reactive statement to log orderedItems when it changes
	$: {
		console.log('orderedItems changed:', orderedItems);
		console.log('totalOrderedItemsPrice:', totalOrderedItemsPrice);
	}

	// Function to calculate voucher discount
	async function calculateVoucherDiscount() {
		const response = await fetch(
			`http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getVouchersbyCode&voucher_code=${voucherCode}`,
			{
				method: 'GET' // Change to GET method
			}
		);
		if (response.ok) {
			const data = await response.json();
			voucherDiscount = parseFloat(data[0].voucher_discount);
		} else {
			console.error('Failed to fetch vouchers:', response.statusText);
			voucherDiscount = 0; // Reset discount on fetch failure
		}
		console.log('Voucher Code Inputted:', voucherCode); // Log the voucher code inputted
		console.log('Fetched Discount:', voucherDiscount); // Log the fetched discount
		// Log the link with the inputted voucher code
		console.log(
			`Fetch URL: http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getVouchersbyCode&voucher_code=${voucherCode}`
		);
	}

	async function fetchCashierName() {
		// Retrieve staff_token from local storage only if not already fetched
		if (!staffToken) {
			staffToken = localStorage.getItem('staff_token'); // Get the staff_token
		}
		console.log('Fetched staff_token:', staffToken); // Log the fetched staff_token
		console.log(
			`Fetching user data from: http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getUser&staff_token=${staffToken}`
		); // Log the URL with the staff_token
		const response = await fetch(
			`http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getUser&staff_token=${staffToken}`
		);
		if (response.ok) {
			const userData = await response.json();
			console.log('Fetched user data:', userData); // Log the fetched user data
			cashierName = userData || 'Unknown'; // Default to 'Unknown' if firstName is not available
		}
	}

	// Function to fetch orders
	async function fetchQueuedOrders() {
		try {
			const response = await fetch(
				'http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getQueOrders'
			);
			if (!response.ok) {
				throw new Error(`Failed to fetch queued orders: ${response.statusText}`);
			}
			queuedOrders = await response.json(); // Check the response here
			// Convert basePrice to number for each order
			queuedOrders.forEach(order => {
				order.basePrice = Number(order.basePrice); // Convert basePrice to number
				console.log('Table Number:', order.table_number); // Log the table number for each order
			});
		} catch (error) {
			console.error('Error fetching queued orders:', error); // Improved error logging
			showAlert('Failed to fetch queued orders. Please try again later.', 'error'); // Notify user
		}
	}

	// Function to fetch reserve tables
	async function fetchReserveTables() {
		const response = await fetch(
			'http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getReserveTables'
		);
		if (response.ok) {
			const data = await response.json();
			reservedTables = data.map((table: { table_number: string }) => table.table_number); // Assuming the response contains an array of reserved table objects
		} else {
			console.error('Failed to fetch reserved tables', response.statusText);
		}
	}

	async function fetchOrders() {
		const response = await fetch(
			'http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getQueOrders'
		);
		if (response.ok) {
			orders = await response.json(); // Store the fetched orders
		} else {
			console.error('Failed to fetch orders:', response.statusText);
		}
	}

	function updateTime() {
		const now = new Date();
		currentTime = now.toLocaleString('en-US', {
			year: 'numeric',
			month: 'long',
			day: 'numeric',
			hour: '2-digit',
			minute: '2-digit',
			second: '2-digit'
		});
		currentDay = now.toLocaleString('en-US', { weekday: 'long' });
	}

	onMount(() => {
		fetchQueuedOrders();
		fetchCashierName(); // Automatically fetch cashier name on mount
		fetchOrders(); // Fetch orders on component mount
		updateTime(); // Initial call to set the time
		const intervalTime = setInterval(updateTime, 1000); // Update time every second
		fetchReserveTables(); // Fetch reserved tables on component mount

		// Add event listener for keydown
		document.addEventListener('keydown', handleKeyDown);

		// Clean up the event listener on component unmount
		return () => {
			clearInterval(intervalTime); // Clear interval on component unmount
			document.removeEventListener('keydown', handleKeyDown);
		};
	});

	function handleNumberInput(num: string) {
		payment += num;
	}

	function handleBackspace() {
		payment = payment.slice(0, -1);
	}

	function handleClear() {
		payment = '';
	}

	function showAlert(message: string, type: string) {
		const alertDiv = document.createElement('div');
		alertDiv.className = `fixed top-0 left-1/2 transform -translate-x-1/2 mt-4 p-4 ${type === 'success' ? 'bg-green-500' : 'bg-red-500'} text-white rounded shadow-lg`;
		alertDiv.innerText = message;
		document.body.appendChild(alertDiv);
		setTimeout(() => {
			alertDiv.remove();
		}, 3000); // Remove alert after 3 seconds
	}

	function voidOrder(index: number) {
		voidIndex = index;
		isCodePopupVisible = true;
	}

	function confirmVoid() {
		// Check if the entered code matches the admin password
		if (inputCode === adminPassword) {
			// Password is correct, proceed with voiding the item
			if (voidReceiptNumber && voidItemIndex >= 0) {
				voidIndividualItem(voidReceiptNumber, voidItemIndex);
				closeCodePopup();
			} else {
				showAlert('Invalid void information', 'error');
			}
		} else {
			// Password is incorrect
			showAlert('Incorrect password. Void operation cancelled.', 'error');
			closeCodePopup();
		}
	}

	function closeCodePopup() {
		isCodePopupVisible = false;
		inputCode = '';
		voidReceiptNumber = '';
		voidItemIndex = -1;
	}

	// Function to show password popup before voiding
	function promptPasswordForVoid(receiptNum: string, itemIdx: number) {
		voidReceiptNumber = receiptNum;
		voidItemIndex = itemIdx;
		isCodePopupVisible = true;
	}

	let cards = Array.from({ length: 20 }, (_, index) => ({
		table: `${index + 1}`
	}));

	let outdoorCards = Array.from({ length: 15 }, (_, index) => ({
		table: `O${index + 1}`
	}));

	let gardenCards = Array.from({ length: 6 }, (_, index) => ({
		table: `G${index + 1}`
	}));

	type OrderedItem = {
		que_order_no: string;
		receipt_number: string;
		date: string;
		time: string;
		items_ordered: string;
		total_amount: number;
		amount_paid: number;
		amount_change: number;
		order_status: string;
		table_number: string;
		order_name: string;
		order_name2?: string;
		order_quantity: number;
		order_size: string;
		basePrice: number;
		order_addons?: string;
		order_addons_price?: number;
		order_addons2?: string;
		order_addons_price2?: number;
		order_addons3?: string;
		order_addons_price3?: number;
		order_price?: number;
		voided: boolean;
	};

	let totalPax = 1;
	let seniorCount = 0;
	let seniorDiscount = 0;
	let applyServiceCharge = true; // Add this variable to control service charge application

	// Function to toggle service charge
	function toggleServiceCharge() {
		applyServiceCharge = !applyServiceCharge;
		console.log('Service charge toggled:', applyServiceCharge);
	}

	// Add reactive statement to log when service charge changes
	$: {
		if (applyServiceCharge !== undefined) {
			console.log('Service charge state:', applyServiceCharge);
		}
	}

	function calculateSeniorDiscount() {
		// Ensure totalPax is a number and at least 1
		totalPax = Math.max(1, parseInt(totalPax.toString()) || 1);
		// Ensure seniorCount is a number and at least 0
		seniorCount = Math.max(0, parseInt(seniorCount.toString()) || 0);
		// Ensure seniorCount doesn't exceed totalPax
		seniorCount = Math.min(seniorCount, totalPax);

		// Calculate per person amount
		const perPersonAmount = totalOrderedItemsPrice / totalPax;
		
		// Calculate senior discount (20% of their portion)
		seniorDiscount = (perPersonAmount * 0.20) * seniorCount;
		
		// Return the value for use in template
		return seniorDiscount;
	}

	// Add reactive statement to recalculate discount when total price changes
	$: {
		if (totalOrderedItemsPrice) {
			calculateSeniorDiscount();
		}
	}

	function handlePlaceOrder() {
		// Check if there are ordered items before opening the receipt popup
		if (orderedItems.length === 0) {
			showAlert('No items ordered yet. Please add items to your order.', 'error');
			return; // Exit the function if no items are ordered
		}

		// Calculate total with all discounts
		const subtotal = Math.round(Number(totalOrderedItemsPrice)) || 0;
		const serviceCharge = applyServiceCharge ? Math.round(subtotal * 0.05) : 0;
		const voucherDiscountAmount = Math.round((subtotal * (Number(voucherDiscount) || 0)) / 100);
		const seniorDiscountAmount = Math.round(seniorDiscount);
		const totalCost = Math.round(subtotal - voucherDiscountAmount - seniorDiscountAmount + serviceCharge);
		const paymentAmount = Math.round(Number(payment)) || 0;

		console.log('Debug - Total Cost:', totalCost); // Debug log
		console.log('Debug - Payment Amount:', paymentAmount); // Debug log

		// Allow exact amount or more
		if (paymentAmount < totalCost) {
			showAlert('Invalid amount. Please enter at least the exact amount.', 'error');
			return;
		}

		isReceiptPopupVisible = true; // Show the receipt popup
	}

	async function previewReceipt() {
		try {
			const thermalData = {
				cashierName: cashierName,
				orderNumber: orderNumber,
				totalOrderedItemsPrice: totalOrderedItemsPrice,
				voucherDiscount: voucherDiscount,
				payment: 0,
				orderedItems: orderedItems
			};

			const thermalResponse = await fetch('http://localhost/kaperustiko-possystem/src/routes/thermal_printer.php', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
				},
				body: JSON.stringify(thermalData)
			});

			if (!thermalResponse.ok) {
				throw new Error('Failed to print preview receipt');
			}

			showAlert('Preview receipt printed successfully!', 'success');
		} catch (error: any) {
			console.error('Error:', error);
			showAlert(error.message || 'Failed to print preview receipt.', 'error');
		}
	}

	async function printReceipt() {
		try {
			// First send data to thermal printer
			const thermalData = {
				cashierName: cashierName,
				orderNumber: orderNumber,
				totalOrderedItemsPrice: totalOrderedItemsPrice,
				voucherDiscount: voucherDiscount,
				payment: parseFloat(payment),
				orderedItems: orderedItems,
				totalPax: totalPax,
				seniorCount: seniorCount,
				seniorDiscount: seniorDiscount,
				applyServiceCharge: applyServiceCharge,
				tableNumber: selectedTable || 'Take Out'
			};

			// Print to thermal printer
			const thermalResponse = await fetch('http://localhost/kaperustiko-possystem/src/routes/thermal_printer.php', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
				},
				body: JSON.stringify(thermalData)
			});

			if (!thermalResponse.ok) {
				throw new Error('Failed to print to thermal printer');
			}

			// Then proceed with saving receipt data
			const receiptData = {
				receiptNumber: orderNumber,
				date: new Date().toLocaleDateString(),
				time: new Date().toLocaleTimeString(),
				cashierName: cashierName,
				itemsOrdered: orderedItems,
				totalAmount: totalOrderedItemsPrice,
				amountPaid: parseFloat(payment) || 0,
				change: Math.max(0, parseFloat((parseFloat(payment) - totalOrderedItemsPrice).toFixed(2))),
				order_take: isTakeOut ? 'Take Out' : 'Dine In',
				table_number: selectedTable
			};

			const response = await fetch('http://localhost/kaperustiko-possystem/backend/modules/insert.php?action=insertReceipt', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify(receiptData)
			});

			const data = await response.json();
			if (data.error) {
				throw new Error(data.error);
			}

			showAlert('Receipt printed and saved successfully!', 'success');
			isReceiptPopupVisible = false;

			// Delete table occupancy
			const deleteResponse = await fetch(
				`http://localhost/kaperustiko-possystem/backend/modules/delete.php?action=deleteTableOccupancy&table_number=${selectedTable}`,
				{ method: 'DELETE' }
			);

			const deleteData = await deleteResponse.json();
			if (!deleteData.success) {
				console.error('Failed to delete table data:', deleteData.message);
			}

			window.location.reload();
		} catch (error: any) {
			console.error('Error:', error);
			showAlert(error.message || 'Failed to process receipt.', 'error');
		}
	}

	function openReceiptPopup() {
		isReceiptPopupVisible = true;
	}

	let isCardPopupVisible = false;
	let selectedCard: any;

	function openCardPopup(card: any) {
		isCardPopupVisible = true;
		selectedCard = card;
	}

	function closeCardPopup() {
		isCardPopupVisible = false;
	}

	// Function to handle checkout
	function handleCheckOut(table: string) {
		const ordersToCheckOut = queuedOrders.filter((order) => order.table_number === table);
		if (ordersToCheckOut.length > 0) {
			try {
				// Clear existing ordered items
				orderedItems = []; 
				
				// Loop through each order and parse items
				ordersToCheckOut.forEach(order => {
					const items = JSON.parse(order.items_ordered); // Parse items
					orderedItems.push(...items); // Add items to orderedItems
				});

				// Calculate total from the actual items - this ensures voided items aren't counted
				totalOrderedItemsPrice = 0;
				orderedItems.forEach(item => {
					// Base price for the item
					totalOrderedItemsPrice += Number(item.basePrice || 0);
					
					// Add any addon prices
					if (item.order_addons_price) totalOrderedItemsPrice += Number(item.order_addons_price || 0);
					if (item.order_addons_price2) totalOrderedItemsPrice += Number(item.order_addons_price2 || 0);
					if (item.order_addons_price3) totalOrderedItemsPrice += Number(item.order_addons_price3 || 0);
				});

				orderNumber = ordersToCheckOut[0].que_order_no; // Set the order number from the first order
				selectedTable = table; // Set the selected table
				console.log("Updated total price:", totalOrderedItemsPrice);
				console.log("Ordered items after checkout:", orderedItems);
				closeCardPopup(); // Close the popup after checking out
			} catch (error) {
				console.error("Error parsing items_ordered JSON during checkout:", error);
				showAlert('Error processing order data. Please contact support.', 'error');
			}
		} else {
			showAlert('No orders found for this table.', 'error'); // Show alert if no orders found
		}
	}

	// Function to handle table reservation
	async function handleReserveTable() {
		const response = await fetch(
			'http://localhost/kaperustiko-possystem/backend/modules/insert.php?action=reserve_date',
			{
				method: 'POST',
				headers: {
					'Content-Type': 'application/json'
				},
				body: JSON.stringify({
					reserve_date: reserveDate,
					reserve_time: reserveTime,
					table_number: selectedTableNumber
				})
			}
		);

		const result = await response.json();
		if (result.success) {
			showAlert('Table reserved successfully!', 'success');
			isReservePopupVisible = false;
			location.reload();
		} else {
			showAlert('Failed to reserve table: ' + result.error, 'error');
		}
	}

	async function openReservedTable2Popup(card: any) {
		isReservePopup2Visible = true; // Set the popup visibility to true
		selectedCard = card; // Store the selected card data

		// Fetch reserved table details
		const response = await fetch(
			`http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getInfoReserveTables&table_number=${card.table}`
		);
		if (response.ok) {
			const reservedTableDetails = await response.json();
			// You can now use reservedTableDetails to display more information in the popup if needed
			reserveDate = reservedTableDetails[0]?.reserve_date || ''; // Assuming the response contains reserve_date
			reserveTime = reservedTableDetails[0]?.reserve_time || ''; // Assuming the response contains reserve_time
		} else {
			console.error('Failed to fetch reserved table details:', response.statusText);
		}
	}

	async function handleDeleteReservation() {
		const response = await fetch(
			`http://localhost/kaperustiko-possystem/backend/modules/delete.php?action=deleteByTableNumber&table_number=${selectedCard.table}`,
			{
				method: 'DELETE'
			}
		);

		const result = await response.json();
		if (result.success) {
			showAlert('Reservation deleted successfully!', 'success');
			isReservePopup2Visible = false; // Close the popup
			location.reload(); // Optionally reload to reflect changes
		} else {
			showAlert('Failed to delete reservation: ' + result.message, 'error');
		}
	}

	let clickedKey: string | null = null; // Variable to track the clicked key

	// Function to handle keydown events
	function handleKeyDown(event: KeyboardEvent) {
		if (event.key === 'Escape') {
			// Close popups when 'Esc' is pressed
			if (isCodePopupVisible) {
				closeCodePopup();
			}
			if (isReservePopupVisible) {
				isReservePopupVisible = false;
			}
			if (isReservePopup2Visible) {
				isReservePopup2Visible = false;
			}
			if (isReceiptPopupVisible) {
				isReceiptPopupVisible = false;
			}
			if (isCardPopupVisible) {
				closeCardPopup();
			}
		} else if (event.key === 'Enter') {
			// Confirm actions when 'Enter' is pressed
			if (isCodePopupVisible) {
				confirmVoid();
			}
			if (isReservePopupVisible) {
				handleReserveTable();
			}
			if (isReceiptPopupVisible) {
				printReceipt();
			}
			// Trigger the button click functionality for placing an order
			handleButtonClick(
				'Place Order',
				0,
				orderedItems,
				payment,
				handleBackspace,
				handleClear,
				voidOrder,
				handlePlaceOrder,
				handleNumberInput,
				isDineIn,
				isTakeOut
			);
		}
	}

	async function testThermalPrint() {
		try {
			const testData = {
				cashierName: "Test Cashier",
				orderNumber: "TEST-" + Math.floor(Math.random() * 1000),
				totalOrderedItemsPrice: 299.00,
				voucherDiscount: 0,
				payment: 300.00,
				orderedItems: [
					{
						order_name: "Test Coffee",
						order_quantity: 1,
						basePrice: 199.00,
						order_addons: "Extra Shot",
						order_addons_price: 100.00
					}
				]
			};

			console.log('Sending test print data:', testData);

			const response = await fetch('http://localhost/kaperustiko-possystem/src/routes/thermal_printer.php', {
				method: 'POST',
				headers: {
					'Content-Type': 'application/json',
				},
				body: JSON.stringify(testData)
			});
			
			console.log('Response status:', response.status);
			const responseText = await response.text();
			console.log('Raw response:', responseText);

			let result;
			try {
				result = JSON.parse(responseText);
			} catch (e) {
				console.error('Failed to parse JSON response:', e);
				alert('Error: Invalid response from server. Check console for details.');
				return;
			}
			
			if (result.success) {
				alert('Test receipt printed successfully!');
			} else {
				alert('Failed to print test receipt: ' + (result.message || 'Unknown error'));
			}
		} catch (error: unknown) {
			const errorMessage = error instanceof Error ? error.message : 'Unknown error occurred';
			console.error('Print error:', error);
			alert('Error printing test receipt: ' + errorMessage);
		}
	}

	// Function to void an individual item from a queued order
	async function voidIndividualItem(receiptNumber: string, itemIndex: number) {
		try {
			console.log(`Attempting to void item index ${itemIndex} from receipt ${receiptNumber}`);
			
			// Get the current order data first
			const getResponse = await fetch(
				`http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getOrderItemsByReceipt&receipt_number=${receiptNumber}`
			);
			
			if (!getResponse.ok) {
				throw new Error(`Error fetching order: ${getResponse.statusText}`);
			}
			
			const orderData = await getResponse.json();
			console.log("Received order data:", orderData);
			
			if (!orderData || orderData.length === 0) {
				throw new Error("Order not found");
			}
			
			// Since we need to modify the original order in the database,
			// we need to get the original order object from que_orders
			const fetchOriginalOrder = await fetch(
				`http://localhost/kaperustiko-possystem/backend/modules/get.php?action=getQueOrders`
			);
			
			if (!fetchOriginalOrder.ok) {
				throw new Error(`Error fetching queued orders: ${fetchOriginalOrder.statusText}`);
			}
			
			const allOrders = await fetchOriginalOrder.json();
			const originalOrder = allOrders.find((order: any) => order.receipt_number === receiptNumber);
			
			if (!originalOrder) {
				throw new Error("Original order not found");
			}
			
			console.log("Found original order:", originalOrder);
			
			// Parse items from original order
			let items = JSON.parse(originalOrder.items_ordered);
			
			if (!Array.isArray(items) || items.length <= itemIndex) {
				throw new Error(`Invalid items array or index out of bounds. Items length: ${items?.length}, Index: ${itemIndex}`);
			}
			
			// Remove the item at the specified index
			const removedItem = items.splice(itemIndex, 1)[0];
			console.log("Removed item:", removedItem);
			
			// Create a special DELETE request for this specific action to void the item
			const voidResponse = await fetch(
				`http://localhost/kaperustiko-possystem/backend/modules/delete.php?action=voidQueuedOrder&receipt_number=${receiptNumber}`,
				{
					method: 'DELETE',
					headers: {
						'Content-Type': 'application/json'
					}
				}
			);
			
			if (!voidResponse.ok) {
				throw new Error(`Error voiding order: ${voidResponse.statusText}`);
			}
			
			// If there are still items, re-add the order with updated items
			if (items.length > 0) {
				// Re-insert the order with the updated items list
				const reinsertData = {
					saveQueOrder: true,
					receiptNumber: receiptNumber,
					date: originalOrder.date,
					time: originalOrder.time,
					cashierName: "Cashier", // Fill in with default 
					waiterName: originalOrder.waiter_name || "",
					waiterCode: originalOrder.waiter_code || "",
					itemsOrdered: items,
					totalAmount: calculateTotalFromItems(items),
					amountPaid: originalOrder.amount_paid || 0,
					change: originalOrder.amount_change || 0,
					order_take: originalOrder.order_take || "Dine In",
					table_number: originalOrder.table_number || ""
				};
				
				const reinsertResponse = await fetch(
					'http://localhost/kaperustiko-possystem/backend/modules/save_que_order.php',
					{
						method: 'POST',
						headers: {
							'Content-Type': 'application/json'
						},
						body: JSON.stringify(reinsertData)
					}
				);
				
				if (!reinsertResponse.ok) {
					throw new Error(`Error reinserting order: ${reinsertResponse.statusText}`);
				}
				
				const reinsertResult = await reinsertResponse.json();
				console.log("Reinsert result:", reinsertResult);
			}
			
			// Update UI
			showAlert("Item voided successfully", "success");
			await fetchQueuedOrders();
			
			// Re-trigger checkout logic if a card is selected
			if (selectedCard && selectedCard.table) {
				handleCheckOut(selectedCard.table);
			}
			
		} catch (error) {
			console.error('Error voiding item:', error);
			showAlert(`Error voiding item: ${error instanceof Error ? error.message : 'Unknown error'}`, 'error');
		}
	}
	
	// Helper function to calculate total from items
	function calculateTotalFromItems(items: any[]): number {
		let total = 0;
		items.forEach((item: any) => {
			// Skip voided items
			if (item.voided) return;
			
			// Base price
			const basePrice = parseFloat(item.basePrice ? String(item.basePrice) : '0');
			// Addons prices
			const addonsPrice1 = parseFloat(item.order_addons_price ? String(item.order_addons_price) : '0');
			const addonsPrice2 = parseFloat(item.order_addons_price2 ? String(item.order_addons_price2) : '0');
			const addonsPrice3 = parseFloat(item.order_addons_price3 ? String(item.order_addons_price3) : '0');
			// Add to total
			total += basePrice + addonsPrice1 + addonsPrice2 + addonsPrice3;
		});
		return total;
	}

	// Function to calculate the total bill with all discounts and charges
	function calculateTotalBill() {
		// Check if orderedItems array is empty
		if (orderedItems.length === 0) {
			console.log('Warning: orderedItems array is empty');
		}
		
		// Base subtotal - only including non-voided items
		const subtotal = totalOrderedItemsPrice;
		
		// Calculate service charge
		const serviceCharge = applyServiceCharge ? subtotal * 0.05 : 0;
		
		// Calculate discounts
		const voucherDiscountAmount = (subtotal * (parseFloat(String(voucherDiscount)) || 0)) / 100;
		const seniorDiscountAmount = seniorDiscount;
		
		// Calculate final total
		const total = subtotal - voucherDiscountAmount - seniorDiscountAmount + serviceCharge;
		
		// Add console log for debugging
		console.log('Total calculation:', {
			subtotal,
			serviceCharge,
			voucherDiscountAmount,
			seniorDiscountAmount,
			total,
			orderedItemsCount: orderedItems.length
		});
		
		return total;
	}

	// Function to calculate subtotal (for receipt)
	function calculateSubtotal() {
		// Only count non-voided items
		let subtotal = 0;
		orderedItems.forEach(item => {
			if (!item.voided) {
				subtotal += parseFloat(item.basePrice ? String(item.basePrice) : '0') + 
				           parseFloat(item.order_addons_price ? String(item.order_addons_price) : '0') +
				           parseFloat(item.order_addons_price2 ? String(item.order_addons_price2) : '0') +
				           parseFloat(item.order_addons_price3 ? String(item.order_addons_price3) : '0');
			}
		});
		return subtotal;
	}

	// Function for calculating total with discounts for receipt
	function calculateTotal() {
		const subtotal = calculateSubtotal();
		
		// Calculate discounts
		const voucherDiscountAmount = (subtotal * (parseFloat(String(voucherDiscount)) || 0)) / 100;
		const seniorDiscountAmount = seniorDiscount;
		
		// Calculate service charge
		const serviceCharge = applyServiceCharge ? subtotal * 0.05 : 0;
		
		// Calculate final total
		return subtotal - voucherDiscountAmount - seniorDiscountAmount + serviceCharge;
	}

	// Format date function for receipt
	function formatDate(date: Date): string {
		return date.toLocaleDateString('en-US', {
			year: 'numeric',
			month: 'long',
			day: 'numeric'
		});
	}
	
	// Function to close receipt popup
	function closeReceiptPopup() {
		isReceiptPopupVisible = false;
	}
</script>

<div class="flex h-screen">
	<Sidebar />
	<div class="flex flex-grow overflow-hidden bg-gray-100">
		<div class="flex-start w-full overflow-auto p-4">
			<div class="mb-4 flex w-full space-x-4">
				{#each ['All', 'Occupied', 'Available', 'Reserved', 'Take Out'] as category}
					<button
						class="w-full rounded-md px-6 py-2 font-bold text-black"
						class:bg-cyan-950={selectedCategory === category}
						class:text-white={selectedCategory === category}
						class:bg-white={selectedCategory !== category}
						class:shadow-md={selectedCategory !== category}
						on:click={() => (selectedCategory = category)}
					>
						{category}
					</button>
				{/each}
				<button
					class="w-full rounded-md bg-cyan-950 px-6 py-2 font-bold text-white hover:bg-blue-600"
					on:click={() => (isReservePopupVisible = true)}
				>
					Reserve Table
				</button>
			</div>

			<div class="mb-4 flex items-center justify-between font-bold text-black">
				{#if selectedCategory === 'All'}
					<p>Display All Tables</p>
				{:else if selectedCategory === 'Occupied'}
					<p>Display Occupied Tables</p>
				{:else if selectedCategory === 'Available'}
					<p>Display Available Tables</p>
				{:else if selectedCategory === 'Reserved'}
					<p>Display Reserved Tables</p>
				{:else if selectedCategory === 'Take Out'}
					<p>Display Take Out Tables</p>
				{/if}
				<p class="mr-4">{currentDay} - {currentTime}</p>
			</div>

			<div class="mt-6 grid grid-cols-1 gap-6 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4">
				{#each cards as card}
					{#if selectedCategory === 'All' || 
						(selectedCategory === 'Occupied' && orders.some((order) => order.table_number === card.table)) || 
						(selectedCategory === 'Available' && !orders.some((order) => order.table_number === card.table) && !reservedTables.includes(card.table)) || 
						(selectedCategory === 'Reserved' && reservedTables.includes(card.table))}
						<button
							class={`border-2 ${orders.some((order) => order.table_number === card.table) ? 'border-white bg-cyan-950 text-white' : reservedTables.includes(card.table) ? 'border-white bg-red-950 text-white' : 'border-cyan-950 bg-white text-black'} flex flex-col items-center justify-center rounded-full p-8 shadow-lg`}
							on:click={() => {
								if (reservedTables.includes(card.table)) {
									openReservedTable2Popup(card);
								} else if (orders.some((order) => order.table_number === card.table)) {
									openCardPopup(card);
								}
							}}
							aria-label={`Open popup for table ${card.table}`}
						>
							<h3 class="text-6xl font-bold">{card.table}</h3>
							<span
								class={`mt-2 rounded-full bg-gray-200 px-4 py-2 text-xs text-gray-500 ${orders.some((order) => order.table_number === card.table) ? 'bg-red-500 text-white' : reservedTables.includes(card.table) ? 'bg-red-500 text-white' : 'bg-green-500 text-white'}`}
							>
								{orders.some((order) => order.table_number === card.table)
									? 'Occupied'
									: reservedTables.includes(card.table)
										? 'Reserved'
										: 'Available'}
							</span>
						</button>
					{/if}
				{/each}
			</div>

			<!-- Outdoor Tables Section -->
			<div class="mt-12">
				<h2 class="mb-6 text-center text-3xl font-bold text-gray-800">Outdoor Setting</h2>
				<div class="mt-6 grid grid-cols-1 gap-6 sm:grid-cols-2">
					{#each outdoorCards as card}
						<button
							class={`border-2 ${orders.some((order) => order.table_number === card.table) ? 'border-white bg-cyan-950 text-white' : reservedTables.includes(card.table) ? 'border-white bg-red-950 text-white' : 'border-cyan-950 bg-white text-black'} flex flex-col items-center justify-center rounded-full p-8 shadow-lg`}
							on:click={() => {
								if (reservedTables.includes(card.table)) {
									openReservedTable2Popup(card);
								} else if (orders.some((order) => order.table_number === card.table)) {
									openCardPopup(card);
								}
							}}
							aria-label={`Open popup for outdoor table ${card.table}`}
						>
							<h3 class="text-5xl font-bold">{card.table}</h3>
							<span
								class={`mt-2 rounded-full bg-gray-200 px-4 py-2 text-xs text-gray-500 ${orders.some((order) => order.table_number === card.table) ? 'bg-red-500 text-white' : reservedTables.includes(card.table) ? 'bg-red-500 text-white' : 'bg-green-500 text-white'}`}
							>
								{orders.some((order) => order.table_number === card.table)
									? 'Occupied'
									: reservedTables.includes(card.table)
										? 'Reserved'
										: 'Available'}
							</span>
						</button>
					{/each}
				</div>
			</div>

		

			<!-- Garden Tables Section -->
			<div class="mt-12">
				<h2 class="mb-6 text-center text-3xl font-bold text-gray-800">Garden Setting</h2>
				<div class="mt-6 grid grid-cols-1 gap-6 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-6">
					{#each gardenCards as card}
						<button
							class={`border-2 ${orders.some((order) => order.table_number === card.table) ? 'border-white bg-cyan-950 text-white' : reservedTables.includes(card.table) ? 'border-white bg-red-950 text-white' : 'border-cyan-950 bg-white text-black'} flex flex-col items-center justify-center rounded-full p-6 shadow-lg`}
							on:click={() => {
								if (reservedTables.includes(card.table)) {
									openReservedTable2Popup(card);
								} else if (orders.some((order) => order.table_number === card.table)) {
									openCardPopup(card);
								}
							}}
							aria-label={`Open popup for garden table ${card.table}`}
						>
							<h3 class="text-4xl font-bold">{card.table}</h3>
							<span
								class={`mt-2 rounded-full bg-gray-200 px-4 py-2 text-xs text-gray-500 ${orders.some((order) => order.table_number === card.table) ? 'bg-red-500 text-white' : reservedTables.includes(card.table) ? 'bg-red-500 text-white' : 'bg-green-500 text-white'}`}
							>
								{orders.some((order) => order.table_number === card.table)
									? 'Occupied'
									: reservedTables.includes(card.table)
										? 'Reserved'
										: 'Available'}
							</span>
						</button>
					{/each}
				</div>
			</div>

			{#if isCardPopupVisible}
				<div class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-70">
					<div class="w-full max-w-md rounded-lg bg-white p-6 shadow-lg max-h-[90vh] overflow-y-auto">
						<h2 class="mb-4 text-center text-2xl font-bold text-gray-800">
							Table {selectedCard.table}
						</h2>
						{#if queuedOrders.length > 0}
							{#each queuedOrders as order, itemIndex}
								{#if order.table_number === selectedCard.table}
									<div class="border-b border-gray-300 py-2">
										<p class="font-semibold">
											Order Nos: <span class="font-normal">{order.que_order_no}</span>
										</p>
										<p class="font-semibold">
											Receipt No: <span class="font-normal">{order.receipt_number}</span>
										</p>
										<p class="font-semibold">Date: <span class="font-normal">{order.date}</span></p>
										<p class="font-semibold">Time: <span class="font-normal">{order.time}</span></p>
										<p class="font-semibold">Items Ordered:</p>
										{#each (() => {
											try {
												return JSON.parse(order.items_ordered);
											} catch (error) {
												console.error("Error parsing items_ordered JSON:", error);
												console.log("Raw JSON data:", order.items_ordered);
												return []; // Return empty array in case of parsing error
											}
										})() as item, innerItemIndex}
											<div class="flex items-center justify-between border-b border-gray-200 py-2">
												<div class="flex-1">
													<p class="font-normal">Name: {item.order_name} {item.order_name2}</p>
													<p class="font-normal">Quantity: {item.order_quantity}</p>
													<p class="font-normal">Size: {item.order_size}</p>
												</div>
												<div class="flex-none text-right">
													<p class="font-normal">₱{parseFloat(item.basePrice ? String(item.basePrice) : '0').toFixed(2)}</p>
													<button
														on:click={() => promptPasswordForVoid(order.receipt_number, innerItemIndex)}
														class="bg-red-500 hover:bg-red-600 text-white font-bold py-1 px-2 rounded text-xs mt-1"
													>
														Void
													</button>
												</div>
											</div>
											<p class="font-normal">Addons:</p>
											{#if item.order_addons && item.order_addons_price != null && item.order_addons_price > 0}
												<div class="flex justify-between">
													<p class="font-normal">{item.order_addons}</p>
													<p class="text-right font-normal">₱{parseFloat(item.order_addons_price ? String(item.order_addons_price) : '0').toFixed(2)}</p>
												</div>
											{/if}
											{#if item.order_addons2}
												<div class="flex justify-between">
													<p class="font-normal">{item.order_addons2}</p>
													<p class="text-right font-normal">₱{parseFloat(item.order_addons_price2 ? String(item.order_addons_price2) : '0').toFixed(2)}</p>
												</div>
											{/if}
											{#if item.order_addons3}
												<div class="flex justify-between">
													<p class="font-normal">{item.order_addons3}</p>
													<p class="text-right font-normal">₱{parseFloat(item.order_addons_price3 ? String(item.order_addons_price3) : '0').toFixed(2)}</p>
												</div>
											{/if}
										{/each}
										<div class="flex justify-between">
											<p class="text-lg font-bold">Total Price:</p>
											<p class="text-right text-lg font-normal">₱{order.total_amount}.00</p>
										</div>
									</div>
									<p class="font-semibold">
										Status: <span class="font-normal">{order.order_status}</span>
									</p>
									
								{/if}
							{/each}
						{:else}
							<p class="text-center text-gray-600">No orders for this table.</p>
						{/if}
						<div class="mt-4 flex justify-center space-x-4">
							<button
								on:click={closeCardPopup}
								class="w-full max-w-xs rounded-md bg-red-500 px-4 py-2 text-white transition hover:bg-red-600"
								>Close</button
							>
							<button
								on:click={() => handleCheckOut(selectedCard.table)}
								class="w-full max-w-xs rounded-md bg-green-500 px-4 py-2 text-white transition hover:bg-green-600"
								>Check Out</button
							>
						</div>
					</div>
				</div>
			{/if}
		</div>
	</div>

	<div class="w-[350px]">
		<div
			class="fixed right-0 top-0 flex h-full w-[350px] flex-col items-center bg-gray-100 p-4 shadow-lg"
		>
			<div class="mb-4 w-full rounded-md bg-green-800 py-2 text-center text-white">
				<p class="text-sm font-bold">Order Number {orderNumber}</p>
			</div>

			<div class="mb-4 max-h-[400px] w-full flex-grow space-y-2 overflow-y-auto">
				{#if orderedItems.length > 0}
					{#each orderedItems as item}
						<div class="rounded-lg border bg-white p-4 shadow-md">
							<p class="font-semibold">
								{item.order_name}
								{item.order_name2} {item.order_quantity}
							</p>
							<div class="mb-2 flex w-full items-center justify-between border-b pb-1">
								<p class="font-normal">{item.order_size}</p>
								<p class="text-right font-normal">₱{parseFloat(item.basePrice ? String(item.basePrice) : '0').toFixed(2)}</p>
							</div>

							{#if item.order_addons && item.order_addons_price != null && item.order_addons_price > 0}
								<p class="font-normal">Addons:</p>
								<div class="flex flex-col">
									{#if item.order_addons}
										<div class="flex justify-between">
											<p class="font-normal">{item.order_addons}</p>
											<p class="text-right font-normal">₱{parseFloat(item.order_addons_price ? String(item.order_addons_price) : '0').toFixed(2)}</p>
										</div>
									{/if}
									{#if item.order_addons2}
										<div class="flex justify-between">
											<p class="font-normal">{item.order_addons2}</p>
											<p class="text-right font-normal">₱{parseFloat(item.order_addons_price2 ? String(item.order_addons_price2) : '0').toFixed(2)}</p>
										</div>
									{/if}
									{#if item.order_addons3}
										<div class="flex justify-between">
											<p class="font-normal">{item.order_addons3}</p>
											<p class="text-right font-normal">₱{parseFloat(item.order_addons_price3 ? String(item.order_addons_price3) : '0').toFixed(2)}</p>
										</div>
									{/if}
								</div>
							{/if}
						</div>
					{/each}
				{:else}
					<p class="text-center text-gray-600">No items ordered yet.</p>
				{/if}
			</div>

			<div class="mt-auto w-full rounded-lg p-2 shadow-md">
				<div class="mb-4 flex w-full items-center justify-between border-b pb-2">
					<p class="text-sm font-semibold text-gray-700">Voucher Code:</p>
					<input
						type="text"
						bind:value={voucherCode}
						class="text-sm font-bold text-gray-800 flex-grow mr-2 w-[60%]"
						placeholder="Enter code"
					/>
					<button
						on:click={calculateVoucherDiscount}
						class="rounded-md bg-blue-500 px-4 py-2 text-white transition hover:bg-blue-600"
					>
						Redeem
					</button>
				</div>
				<!-- Enhanced Senior/PWD Discount Fields -->
				<div class="mb-4 flex w-full items-center justify-between border-b pb-2">
					<div class="flex items-center w-1/2 mr-2">
						<p class="text-sm font-semibold text-gray-700 mr-2">Total PAX:</p>
						<input
							type="text"
							bind:value={totalPax}
							class="text-sm font-bold text-gray-800 w-16 border rounded px-2 py-1"
							on:input={() => {
								// Capture the input even if it's not a number yet
								calculateSeniorDiscount();
							}}
							on:blur={() => {
								// On blur, ensure it's at least 1
								if (!totalPax || totalPax < 1) totalPax = 1;
								calculateSeniorDiscount();
							}}
							on:keydown={(e) => {
								if (e.key === 'Enter') {
									if (!totalPax || totalPax < 1) totalPax = 1;
									calculateSeniorDiscount();
								}
							}}
						/>
					</div>
					<div class="flex items-center w-1/2">
						<p class="text-sm font-semibold text-gray-700 mr-2">Senior/PWD:</p>
						<input
							type="text"
							bind:value={seniorCount}
							class="text-sm font-bold text-gray-800 w-16 border rounded px-2 py-1"
							on:input={() => {
								// Capture the input even if it's not a number yet
								calculateSeniorDiscount();
							}}
							on:blur={() => {
								// On blur, ensure it's at least 0
								if (!seniorCount || seniorCount < 0) seniorCount = 0;
								calculateSeniorDiscount();
							}}
							on:keydown={(e) => {
								if (e.key === 'Enter') {
									if (!seniorCount || seniorCount < 0) seniorCount = 0;
									calculateSeniorDiscount();
								}
							}}
						/>
					</div>
				</div>
				<div class="flex w-full items-center justify-between border-b pb-1">
					<button 
						class="flex items-center text-left focus:outline-none" 
						on:click={toggleServiceCharge}
					>
						<div class={`w-5 h-5 mr-2 border rounded flex items-center justify-center ${applyServiceCharge ? 'bg-blue-600 border-blue-600' : 'bg-white border-gray-300'}`}>
							{#if applyServiceCharge}
								<svg xmlns="http://www.w3.org/2000/svg" class="h-4 w-4 text-white" viewBox="0 0 20 20" fill="currentColor">
									<path fill-rule="evenodd" d="M16.707 5.293a1 1 0 010 1.414l-8 8a1 1 0 01-1.414 0l-4-4a1 1 0 011.414-1.414L8 12.586l7.293-7.293a1 1 0 011.414 0z" clip-rule="evenodd" />
								</svg>
							{/if}
						</div>
						<span class="text-sm font-semibold text-gray-700">Service Charge (5%)</span>
					</button>
					<p class="text-sm font-bold text-gray-800">₱{applyServiceCharge ? (totalOrderedItemsPrice * 0.05).toFixed(2) : '0.00'}</p>
				</div>
				<div class="flex w-full items-center justify-between border-b pb-1">
					<p class="text-sm font-semibold text-gray-700">Voucher Discount:</p>
					<p class="text-sm font-bold text-gray-800">₱{(totalOrderedItemsPrice * voucherDiscount / 100).toFixed(2)}</p>
				</div>
				<div class="flex w-full items-center justify-between border-b pb-1">
					<p class="text-sm font-semibold text-gray-700">Senior/PWD Discount ({seniorCount}):</p>
					<p class="text-sm font-bold text-gray-800">₱{seniorDiscount.toFixed(2)}</p>
				</div>
				<div class="flex w-full items-center justify-between border-b pb-1">
					<p class="text-sm font-semibold text-gray-700">Total:</p>
					<p class="text-sm font-bold text-gray-800">
						₱{(()=>{
							// Direct calculation for debugging
							let directTotal = 0;
							orderedItems.forEach(item => {
								if (!item.voided) {
									directTotal += parseFloat(String(item.basePrice || 0)) + 
										parseFloat(String(item.order_addons_price || 0)) +
										parseFloat(String(item.order_addons_price2 || 0)) +
										parseFloat(String(item.order_addons_price3 || 0));
								}
							});
							
							// Apply service charge
							const serviceCharge = applyServiceCharge ? directTotal * 0.05 : 0;
							
							// Apply discounts
							const voucherAmount = (directTotal * (parseFloat(String(voucherDiscount)) || 0)) / 100;
							
							// Get final total
							const finalTotal = directTotal - voucherAmount - seniorDiscount + serviceCharge;
							
							console.log("Direct calculation:", {
								directTotal,
								serviceCharge,
								voucherAmount,
								seniorDiscount,
								finalTotal
							});
							
							return finalTotal.toFixed(2);
						})()}
					</p>
				</div>
				<div class="flex justify-between">
					<p class="text-sm">Amount Paid:</p>
					<span class="text-sm">₱{(parseFloat(payment) || 0).toFixed(2)}</span>
				</div>
				<div class="flex justify-between">
					<p class="text-sm">Change:</p>
					<span class="text-sm">
						₱{(() => {
							// Calculate total with all discounts
							const totalCost = Math.round(calculateTotalBill());
							const paymentAmount = parseFloat(payment) || 0;
							
							// If payment and total are equal (or very close), return 0
							if (Math.abs(paymentAmount - totalCost) <= 1) {
								return '0.00';
							}
							
							// Otherwise calculate regular change
							return Math.max(0, paymentAmount - totalCost).toFixed(2);
						})()}
					</span>
				</div>
			</div>

			<div class="w-full">
				<div class="mb-4">
					<input
						id="payment-input"
						type="text"
						bind:value={payment}
						placeholder="Enter amount"
						class="w-full rounded-md border border-gray-300 p-3 text-gray-800 focus:border-blue-500 focus:ring-blue-500"
						on:input={() => {
							// Store in localStorage to maintain compatibility with other functions
							localStorage.setItem('payment', payment);
							
							// Calculate total with all discounts
							const totalCost = Math.round(calculateTotalBill());
							const paymentAmount = parseFloat(payment) || 0;

							console.log('Debug - Total Cost:', totalCost); // Debug log
							console.log('Debug - Payment Amount:', paymentAmount); // Debug log
							
							// Allow exact amount or more
							if (paymentAmount < totalCost) {
								showAlert('Invalid amount. Please enter at least the exact amount.', 'error');
							}
						}}
					/>
				</div>
				
				<div class="grid grid-cols-1 gap-4">
					<button
						on:click={() => {
							// Place Order functionality
							handleButtonClick(
								'Place Order',
								0,
								orderedItems,
								payment,
								handleBackspace,
								handleClear,
								voidOrder,
								handlePlaceOrder,
								handleNumberInput,
								isDineIn,
								isTakeOut
							);
							
							currentInputStore.update((store) => {
								return {
									...store,
									currentInput: 'Place Order',
									amountPaid: parseFloat(amountPaid.replace('₱', '').replace(',', ''))
								};
							});
						}}
						class="rounded py-3 font-bold text-white bg-blue-600 hover:bg-blue-700 w-full"
					>
						Checkout Bill
					</button>
					<button
						on:click={() => previewReceipt()}
						class="rounded py-3 font-bold text-white bg-green-600 hover:bg-green-700 w-full"
					>
						Preview Receipt
					</button>
				</div>
			</div>
		</div>
	</div>
</div>

{#if isCodePopupVisible}
	<div class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-70">
		<div class="w-full max-w-md rounded-lg bg-white p-8 shadow-lg">
			<h2 class="mb-4 text-center text-2xl font-bold">Input 6-Digit Code</h2>
			<input
				type="text"
				bind:value={inputCode}
				maxlength="6"
				class="w-full rounded border border-gray-300 p-2 text-center"
				placeholder="Enter 6-digit code"
			/>
			<div class="mt-4 flex justify-between">
				<button on:click={closeCodePopup} class="rounded-md bg-red-500 px-4 py-2 text-white"
					>Cancel</button
				>
				<button on:click={confirmVoid} class="rounded-md bg-blue-500 px-4 py-2 text-white"
					>Confirm</button
				>
			</div>
		</div>
	</div>
{/if}

{#if isReservePopupVisible}
	<div class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-70">
		<div class="w-full max-w-md rounded-lg bg-white p-8 shadow-lg">
			<h2 class="mb-6 text-center text-2xl font-bold text-gray-800">Reserve a Table</h2>
			<input
				type="date"
				bind:value={reserveDate}
				class="mb-4 w-full rounded border border-gray-300 p-3 shadow-sm focus:outline-none focus:ring focus:ring-blue-200"
				min={new Date().toISOString().split('T')[0]}
			/>
			<input
				type="time"
				bind:value={reserveTime}
				class="mb-4 w-full rounded border border-gray-300 p-3 shadow-sm focus:outline-none focus:ring focus:ring-blue-200"
			/>
			<select
				bind:value={selectedTableNumber}
				class="mt-1 block w-full rounded-md border-gray-300 shadow-sm focus:border-blue-500 focus:ring focus:ring-blue-200"
			>
				<option value="">Select Table Number</option>
				<!-- Indoor Tables -->
				<optgroup label="Indoor Tables">
					{#each Array(20) as _, index}
						<option
							value={(index + 1).toString()}
							class={queuedOrders.some((order) => order.table_number === (index + 1).toString())
								? 'bg-red-500 text-white'
								: reservedTables.includes((index + 1).toString())
									? 'bg-orange-950 text-white'
									: ''}
							disabled={tableStatus[index + 1]}
						>
							Table {index + 1}
						</option>
					{/each}
				</optgroup>
				
				<!-- Outdoor Tables -->
				<optgroup label="Outdoor Tables">
					{#each Array(15) as _, index}
						<option
							value={`O${index + 1}`}
							class={queuedOrders.some((order) => order.table_number === `O${index + 1}`)
								? 'bg-red-500 text-white'
								: reservedTables.includes(`O${index + 1}`)
									? 'bg-orange-950 text-white'
									: ''}
							disabled={tableStatus[`O${index + 1}`]}
						>
							Table O{index + 1}
						</option>
					{/each}
				</optgroup>
				
				<!-- Garden Tables -->
				<optgroup label="Garden Tables">
					{#each Array(6) as _, index}
						<option
							value={`G${index + 1}`}
							class={queuedOrders.some((order) => order.table_number === `G${index + 1}`)
								? 'bg-red-500 text-white'
								: reservedTables.includes(`G${index + 1}`)
									? 'bg-orange-950 text-white'
									: ''}
							disabled={tableStatus[`G${index + 1}`]}
						>
							Table G{index + 1}
						</option>
					{/each}
				</optgroup>
			</select>
			<div class="mt-6 flex justify-between">
				<button
					on:click={() => (isReservePopupVisible = false)}
					class="rounded-md bg-red-600 px-4 py-2 text-white transition hover:bg-red-700"
					>Cancel</button
				>
				<button
					on:click={handleReserveTable}
					class="rounded-md bg-blue-600 px-4 py-2 text-white transition hover:bg-blue-700"
					>Confirm</button
				>
			</div>
		</div>
	</div>
{/if}

{#if isReservePopup2Visible}
	<div class="fixed inset-0 flex items-center justify-center bg-black bg-opacity-70">
		<div class="w-full max-w-md rounded-lg bg-white p-8 shadow-lg">
			<h2 class="mb-4 text-center text-2xl font-bold">Reserved Table Details</h2>
			<p>Table Number: {selectedCard.table}</p>
			<p>Reserved Date: {reserveDate}</p>
			<p>Reserved Time: {reserveTime}</p>
			<div class="mt-4 flex justify-center space-x-4">
				<button
					on:click={() => (isReservePopup2Visible = false)}
					class="w-full max-w-xs rounded-md bg-red-500 px-4 py-2 text-white transition hover:bg-red-600"
					>Close</button
				>
				<button
					on:click={handleDeleteReservation}
					class="w-full max-w-xs rounded-md bg-red-600 px-4 py-2 text-white transition hover:bg-red-700"
					>Delete Reservation</button
				>
			</div>
		</div>
	</div>
{/if}

{#if isReceiptPopupVisible}
	<div class="receipt-popup" transition:fade={{ duration: 150 }}>
		<div class="receipt-popup-overlay" on:click={closeReceiptPopup}></div>
		<div class="receipt-popup-content">
			<div class="receipt-popup-header">
				<h3>Receipt Preview</h3>
				<button class="close-button" on:click={closeReceiptPopup}>×</button>
			</div>
			<div class="receipt-popup-body">
				<div class="receipt">
					<div class="receipt-header text-center">
						<h3>Kaperustiko</h3>
						<p>Kaperustiko Cafe & Restaurant</p>
						<p>33 Purok 1, Gugo, Santa Maria, Bulacan</p>
						<p>Receipt #: {orderNumber}</p>
						<p>Date: {formatDate(new Date())}</p>
						<p>Time: {new Date().toLocaleTimeString()}</p>
						<p>Table Number: {selectedTable || 'Take-Out'}</p>
					</div>
					
					<div class="receipt-body mt-4">
						<table class="w-full">
							<thead>
								<tr>
									<th class="text-left">Item</th>
									<th class="text-right">Price</th>
								</tr>
							</thead>
							<tbody>
								{#each orderedItems as item}
									{#if !item.voided}
										<tr>
											<td class="text-left">
												{item.order_name}
												{item.order_name2}
												{#if item.order_addons}
													<br />
													<span class="text-sm">+ {item.order_addons}</span>
												{/if}
												{#if item.order_addons2}
													<br />
													<span class="text-sm">+ {item.order_addons2}</span>
												{/if}
												{#if item.order_addons3}
													<br />
													<span class="text-sm">+ {item.order_addons3}</span>
												{/if}
											</td>
											<td class="text-right">
												₱{parseFloat(item.basePrice ? String(item.basePrice) : '0').toFixed(2)}
												{#if item.order_addons_price && parseFloat(String(item.order_addons_price)) > 0}
													<br />
													<span class="text-sm">₱{parseFloat(item.order_addons_price ? String(item.order_addons_price) : '0').toFixed(2)}</span>
												{/if}
												{#if item.order_addons_price2 && parseFloat(String(item.order_addons_price2)) > 0}
													<br />
													<span class="text-sm">₱{parseFloat(item.order_addons_price2 ? String(item.order_addons_price2) : '0').toFixed(2)}</span>
												{/if}
												{#if item.order_addons_price3 && parseFloat(String(item.order_addons_price3)) > 0}
													<br />
													<span class="text-sm">₱{parseFloat(item.order_addons_price3 ? String(item.order_addons_price3) : '0').toFixed(2)}</span>
												{/if}
											</td>
										</tr>
									{/if}
								{/each}
							</tbody>
						</table>

						<div class="summary mt-4">
							<div class="flex justify-between">
								<span>Subtotal:</span>
								<span>₱{calculateSubtotal().toFixed(2)}</span>
							</div>

							{#if voucherDiscount > 0}
								<div class="flex justify-between">
									<span>Voucher Discount ({voucherDiscount}%):</span>
									<span>-₱{((calculateSubtotal() * voucherDiscount) / 100).toFixed(2)}</span>
								</div>
							{/if}

							{#if seniorCount > 0}
								<div class="flex justify-between">
									<span>Senior Discount ({seniorCount} pax):</span>
									<span>-₱{calculateSeniorDiscount().toFixed(2)}</span>
								</div>
							{/if}

							{#if applyServiceCharge}
								<div class="flex justify-between">
									<span>Service Charge (5%):</span>
									<span>₱{(calculateSubtotal() * 0.05).toFixed(2)}</span>
								</div>
							{/if}

							<div class="flex justify-between font-bold mt-2">
								<span>Total:</span>
								<span>₱{calculateTotal().toFixed(2)}</span>
							</div>
						</div>
					</div>

					<div class="receipt-footer text-center mt-4">
						<p>Thank you for dining with us!</p>
						<p>Please come again.</p>
					</div>
				</div>
			</div>
			<div class="receipt-popup-actions">
				<button class="print-button" on:click={printReceipt}>
					<i class="fas fa-print"></i> Print Receipt
				</button>
				<button class="cancel-button ml-2" on:click={closeReceiptPopup}>
					<i class="fas fa-times"></i> Close
				</button>
			</div>
		</div>
	</div>
{/if}