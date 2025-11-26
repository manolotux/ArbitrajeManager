<script setup>
import { ref, computed, watch, nextTick, onMounted } from "vue";
import {
	createIcons,
	Wallet,
	DollarSign,
	AlertCircle,
	RefreshCw,
	Calculator,
	Plus,
	Edit2,
	Save,
	History,
	CheckCircle,
	Trash2,
	Settings,
	Landmark,
	Users,
	X,
	ArrowRightLeft,
	ArrowUpRight,
	Smartphone,
	Layers,
	PieChart,
	TrendingUp,
	TrendingDown,
	Calendar,
	Search,
	LogOut,
	Lock,
} from "lucide";

// --- SWEETALERT2 IMPORTS ---
import Swal from "sweetalert2";
import "sweetalert2/dist/sweetalert2.min.css";

// --- FIREBASE IMPORTS ---
import { initializeApp } from "firebase/app";
import { getAuth, signInWithPopup, GoogleAuthProvider, signOut, onAuthStateChanged } from "firebase/auth"; // CAMBIO: Google Auth
import {
	getFirestore,
	collection,
	doc,
	addDoc,
	updateDoc,
	deleteDoc,
	onSnapshot,
	query,
	increment,
	setDoc,
	getDoc,
	where,
	orderBy,
	limit,
} from "firebase/firestore";

// --- CONFIGURACIÓN FIREBASE ---
const getEnv = (key) => {
	try {
		return import.meta.env[key];
	} catch (e) {
		return undefined;
	}
};

const envConfig = {
	apiKey: getEnv("VITE_FIREBASE_API_KEY"),
	authDomain: getEnv("VITE_FIREBASE_AUTH_DOMAIN"),
	projectId: getEnv("VITE_FIREBASE_PROJECT_ID"),
	storageBucket: getEnv("VITE_FIREBASE_STORAGE_BUCKET"),
	messagingSenderId: getEnv("VITE_FIREBASE_MESSAGING_SENDER_ID"),
	appId: getEnv("VITE_FIREBASE_APP_ID"),
	measurementId: getEnv("VITE_FIREBASE_MEASUREMENT_ID"),
};

const manualConfig = {
	apiKey: "TU_API_KEY_AQUI",
	authDomain: "TU_PROYECTO.firebaseapp.com",
	projectId: "TU_PROYECTO",
	storageBucket: "TU_PROYECTO.appspot.com",
	messagingSenderId: "000000000",
	appId: "1:000000000:web:0000000000",
	measurementId: "G-0000000000",
};

let firebaseConfig;
if (typeof __firebase_config !== "undefined") {
	firebaseConfig = JSON.parse(__firebase_config);
} else if (envConfig.apiKey) {
	firebaseConfig = envConfig;
} else {
	firebaseConfig = manualConfig;
}

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== "undefined" ? __app_id : "p2p-manager-local";

// --- STATE ---
const user = ref(null);
const authLoading = ref(true);

const accounts = ref([]);
const counterparties = ref([]);
const transactions = ref([]);
const dailyCloses = ref([]);
const activeTab = ref("dashboard");
const targetBuyRate = ref(19.5);

const snackbar = ref({ show: false, message: "", type: "success" });

// ... (Mismos estados de formularios que antes) ...
const isEditing = ref(false);
const editingId = ref(null);
const form = ref({
	type: "buy",
	date: new Date().toISOString().split("T")[0],
	reference: "",
	amountMxn: "",
	rate: "",
	amountUsdt: "",
	bankId: "",
	destinationBankId: "",
	counterpartySelect: "",
	counterparty: "",
	platform: "",
	isPending: false,
	calcMode: "rate",
	commissionPercent: 0.16,
	paymentMethod: "transfer",
});

const treasuryForm = ref({
	type: "deposit",
	date: new Date().toISOString().split("T")[0],
	assetType: "MXN",
	amount: "",
	accountId: "",
	description: "",
});
const accountForm = ref({ name: "", type: "MXN", initialBalance: 0 });
const counterpartyForm = ref({ name: "", dailyRate: "" });

// --- HELPER: SNACKBAR ---
const showSnackbar = (message, type = "success") => {
	snackbar.value = { show: true, message, type };
	nextTick(() => renderIcons());
	setTimeout(() => {
		snackbar.value.show = false;
	}, 4000);
};

// --- FIREBASE AUTH & SYNC ---

onMounted(() => {
	onAuthStateChanged(auth, (currentUser) => {
		user.value = currentUser;
		authLoading.value = false;
		if (currentUser) {
			initDataListeners(currentUser.uid);
			loadUserSettings(currentUser.uid);
		}
		renderIcons();
	});
});

// NUEVO: Función de Login con Google Mejorada
const handleLogin = async () => {
	const provider = new GoogleAuthProvider();
	try {
		await signInWithPopup(auth, provider);
		showSnackbar("Bienvenido");
	} catch (error) {
		console.error("Error Login:", error);
		let msg = "Error al conectar con Google";

		// Manejo de errores comunes
		if (error.code === "auth/popup-closed-by-user") msg = "Ventana cerrada antes de terminar";
		if (error.code === "auth/cancelled-popup-request") msg = "Solicitud cancelada";
		if (error.code === "auth/popup-blocked") msg = "El navegador bloqueó la ventana (permite popups)";
		if (error.code === "auth/unauthorized-domain") msg = "⚠️ Dominio no autorizado en Firebase Console";

		showSnackbar(msg, "error");
	}
};

const handleLogout = async () => {
	try {
		await signOut(auth);
		accounts.value = [];
		transactions.value = [];
		dailyCloses.value = [];
		counterparties.value = [];
	} catch (e) {
		console.error(e);
	}
};

const initDataListeners = (userId) => {
	onSnapshot(
		query(collection(db, "artifacts", appId, "users", userId, "accounts")),
		(s) => (accounts.value = s.docs.map((d) => ({ id: d.id, ...d.data() })))
	);
	onSnapshot(
		query(collection(db, "artifacts", appId, "users", userId, "counterparties")),
		(s) => (counterparties.value = s.docs.map((d) => ({ id: d.id, ...d.data() })))
	);
	onSnapshot(
		query(collection(db, "artifacts", appId, "users", userId, "transactions")),
		(s) =>
			(transactions.value = s.docs
				.map((d) => ({ id: d.id, ...d.data() }))
				.sort((a, b) => new Date(b.date) - new Date(a.date) || b.timestamp - a.timestamp))
	);
	onSnapshot(
		query(
			collection(db, "artifacts", appId, "users", userId, "daily_closes"),
			orderBy("timestamp", "desc"),
			limit(30)
		),
		(s) => (dailyCloses.value = s.docs.map((d) => ({ id: d.id, ...d.data() })))
	);
};

const loadUserSettings = async (userId) => {
	try {
		const docSnap = await getDoc(doc(db, "artifacts", appId, "users", userId, "settings", "general"));
		if (docSnap.exists()) targetBuyRate.value = docSnap.data().targetBuyRate || 19.5;
	} catch (e) {
		console.error(e);
	}
};

watch(targetBuyRate, async (newVal) => {
	if (!user.value) return;
	try {
		await setDoc(
			doc(db, "artifacts", appId, "users", user.value.uid, "settings", "general"),
			{ targetBuyRate: parseFloat(newVal) },
			{ merge: true }
		);
	} catch (e) {}
});

// --- COMPUTED PROPERTIES (Igual que antes) ---
const mxnAccounts = computed(() => accounts.value.filter((a) => a.type === "MXN"));
const usdtAccounts = computed(() => accounts.value.filter((a) => a.type === "USDT"));
const sortedCounterparties = computed(() => [...counterparties.value].sort((a, b) => a.name.localeCompare(b.name)));

watch(
	usdtAccounts,
	(newVal) => {
		if (newVal.length > 0 && !form.value.platform) form.value.platform = newVal[0].name;
	},
	{ immediate: true }
);

const totalMxn = computed(() => mxnAccounts.value.reduce((sum, acc) => sum + (acc.balance || 0), 0));
const totalUsdtWallet = computed(() => usdtAccounts.value.reduce((sum, acc) => sum + (acc.balance || 0), 0));
const totalEquity = computed(() => totalMxn.value + totalUsdtWallet.value * targetBuyRate.value);

const usdtOwedToClients = computed(() =>
	transactions.value
		.filter((t) => t.type === "sell" && t.status === "pending")
		.reduce((sum, t) => sum + t.amountUsdt, 0)
);
const usdtReceivable = computed(() =>
	transactions.value
		.filter((t) => t.type === "buy" && t.status === "pending")
		.reduce((sum, t) => sum + t.amountUsdt, 0)
);
const usdtDeficit = computed(() => usdtOwedToClients.value - (totalUsdtWallet.value + usdtReceivable.value));
const mxnNeededToCover = computed(() => (usdtDeficit.value > 0 ? usdtDeficit.value * targetBuyRate.value : 0));

const monthlyStats = computed(() => {
	const now = new Date();
	const currentMonth = now.getMonth();
	const relevantTxs = transactions.value.filter((tx) => {
		const d = new Date(tx.date);
		return (
			d.getMonth() === currentMonth &&
			d.getFullYear() === now.getFullYear() &&
			(tx.type === "deposit" || tx.type === "withdrawal")
		);
	});
	let funded = 0,
		expenses = 0;
	relevantTxs.forEach((tx) => {
		let valMxn = tx.amountMxn;
		if (!valMxn && tx.amountUsdt) valMxn = tx.amountUsdt * (tx.rate || targetBuyRate.value);
		if (tx.type === "deposit") funded += valMxn;
		if (tx.type === "withdrawal") expenses += valMxn;
	});
	return { funded, expenses };
});

const dailyProfit = computed(() => {
	const lastClose = dailyCloses.value.length > 0 ? dailyCloses.value[0] : null;
	const lastEquity = lastClose ? lastClose.totalEquity : 0;
	const todayStr = new Date().toISOString().split("T")[0];
	const todayCapitalTxs = transactions.value.filter(
		(tx) => tx.date === todayStr && (tx.type === "deposit" || tx.type === "withdrawal")
	);
	let todayFunding = 0,
		todayExpenses = 0;
	todayCapitalTxs.forEach((tx) => {
		let val = tx.amountMxn;
		if (!val && tx.amountUsdt) val = tx.amountUsdt * targetBuyRate.value;
		if (tx.type === "deposit") todayFunding += val;
		if (tx.type === "withdrawal") todayExpenses += val;
	});
	if (!lastClose) return 0;
	return totalEquity.value - lastEquity - todayFunding + todayExpenses;
});

const groupedPending = computed(() => {
	const pending = transactions.value.filter((t) => t.status === "pending");
	const groups = {};
	pending.forEach((tx) => {
		const key = `${tx.counterparty}-${tx.type}`;
		if (!groups[key])
			groups[key] = {
				id: key,
				counterparty: tx.counterparty,
				type: tx.type,
				count: 0,
				totalUsdt: 0,
				totalMxn: 0,
				transactions: [],
			};
		groups[key].count++;
		groups[key].totalUsdt += parseFloat(tx.amountUsdt) || 0;
		groups[key].totalMxn += parseFloat(tx.amountMxn) || 0;
		groups[key].transactions.push(tx);
	});
	return Object.values(groups).sort((a, b) => b.totalUsdt - a.totalUsdt);
});

// --- LOGIC ACTIONS (Resumido para brevedad, misma lógica que versión anterior) ---
const handleAccountSubmit = async () => {
	if (!user.value) return;
	try {
		await addDoc(collection(db, "artifacts", appId, "users", user.value.uid, "accounts"), {
			name: accountForm.value.name.trim(),
			type: accountForm.value.type,
			balance: parseFloat(accountForm.value.initialBalance) || 0,
			timestamp: Date.now(),
		});
		showSnackbar("Cuenta creada");
		accountForm.value.name = "";
	} catch (e) {
		showSnackbar(e.message, "error");
	}
};
const handleCounterpartySubmit = async () => {
	if (!user.value) return;
	try {
		await addDoc(collection(db, "artifacts", appId, "users", user.value.uid, "counterparties"), {
			name: counterpartyForm.value.name.trim(),
			dailyRate: parseFloat(counterpartyForm.value.dailyRate) || 0,
			timestamp: Date.now(),
		});
		showSnackbar("Agregado");
		counterpartyForm.value.name = "";
	} catch (e) {
		showSnackbar(e.message, "error");
	}
};
const updateCounterpartyRate = async (id, newRate) => {
	try {
		await updateDoc(doc(db, "artifacts", appId, "users", user.value.uid, "counterparties", id), {
			dailyRate: parseFloat(newRate) || 0,
		});
	} catch (e) {}
};
const deleteAccount = async (id) => {
	if (
		(
			await Swal.fire({
				title: "¿Eliminar?",
				icon: "warning",
				showCancelButton: true,
				confirmButtonColor: "#ef4444",
				confirmButtonText: "Sí",
			})
		).isConfirmed
	) {
		try {
			await deleteDoc(doc(db, "artifacts", appId, "users", user.value.uid, "accounts", id));
		} catch (e) {}
	}
};
const deleteCounterparty = async (id) => {
	if (
		(
			await Swal.fire({
				title: "¿Eliminar?",
				icon: "warning",
				showCancelButton: true,
				confirmButtonColor: "#ef4444",
				confirmButtonText: "Sí",
			})
		).isConfirmed
	) {
		try {
			await deleteDoc(doc(db, "artifacts", appId, "users", user.value.uid, "counterparties", id));
		} catch (e) {}
	}
};

const updateAccountBalance = async (id, amount, op) => {
	if (!id) return;
	await updateDoc(doc(db, "artifacts", appId, "users", user.value.uid, "accounts", id), {
		balance: increment(op === "add" ? Math.abs(amount) : -Math.abs(amount)),
	});
};
const findUsdtAccountByName = (name) =>
	name ? usdtAccounts.value.find((a) => a.name.trim().toLowerCase() === name.trim().toLowerCase()) : null;

const applyBalanceEffect = async (tx, reverse = false) => {
	if (tx.type === "transfer_own") {
		const sourceOp = reverse ? "add" : "subtract";
		const destOp = reverse ? "subtract" : "add";
		if (tx.bankId) await updateAccountBalance(tx.bankId, tx.amountMxn, sourceOp);
		if (tx.destinationBankId) await updateAccountBalance(tx.destinationBankId, tx.amountMxn, destOp);
		return;
	}
	let mxnSign = 0;
	if (tx.type === "buy" || tx.type === "withdrawal") mxnSign = -1;
	if (tx.type === "sell" || tx.type === "deposit") mxnSign = 1;
	if (reverse) mxnSign *= -1;
	if (tx.bankId && mxnSign !== 0)
		await updateAccountBalance(tx.bankId, tx.amountMxn, mxnSign === 1 ? "add" : "subtract");

	let usdtSign = 0;
	if (tx.status === "completed") {
		if (tx.type === "buy") usdtSign = 1;
		if (tx.type === "sell") usdtSign = -1;
	}
	if (reverse) usdtSign *= -1;
	if (tx.platform && usdtSign !== 0) {
		const w = findUsdtAccountByName(tx.platform);
		if (w) await updateAccountBalance(w.id, tx.amountUsdt, usdtSign === 1 ? "add" : "subtract");
	}
};

const handleTransactionSubmit = async () => {
	if (!user.value) return;
	if (
		(form.value.type === "buy" || form.value.type === "sell") &&
		form.value.platform &&
		!findUsdtAccountByName(form.value.platform)
	) {
		showSnackbar(`Error: No existe wallet ${form.value.platform}`, "error");
		return;
	}

	let finalCp = form.value.counterparty;
	if (form.value.type === "transfer_own") {
		if (!form.value.bankId || !form.value.destinationBankId || form.value.bankId === form.value.destinationBankId) {
			showSnackbar("Verifica cuentas", "error");
			return;
		}
		form.value.counterparty = "Movimiento Interno";
	} else {
		finalCp = form.value.counterpartySelect !== "__OTHER__" ? form.value.counterpartySelect : form.value.counterparty;
		if (!finalCp && form.value.type === "sell") finalCp = "Por Identificar";
		if (!finalCp) {
			showSnackbar("Falta contraparte", "error");
			return;
		}
		form.value.counterparty = finalCp;
	}

	if (isEditing.value && editingId.value) {
		const oldTx = transactions.value.find((t) => t.id === editingId.value);
		if (oldTx) await applyBalanceEffect(oldTx, true);
	}

	const txData = {
		...form.value,
		amountMxn: parseFloat(form.value.amountMxn) || 0,
		rate: form.value.type === "transfer_own" ? 0 : parseFloat(form.value.rate),
		amountUsdt: form.value.type === "transfer_own" ? 0 : parseFloat(form.value.amountUsdt),
		status: form.value.isPending ? "pending" : "completed",
		commissionPercent: parseFloat(form.value.commissionPercent || 0),
		timestamp: Date.now(),
	};
	delete txData.counterpartySelect;

	try {
		if (isEditing.value) {
			await updateDoc(doc(db, "artifacts", appId, "users", user.value.uid, "transactions", editingId.value), txData);
			showSnackbar("Actualizado");
		} else {
			await addDoc(collection(db, "artifacts", appId, "users", user.value.uid, "transactions"), txData);
			applyBalanceEffect(txData, false);
			showSnackbar("Registrado");
		}
		if (form.value.type === "sell") resetForm("partial");
		else resetForm("full");
	} catch (e) {
		showSnackbar("Error", "error");
	}
};

const handleTreasurySubmit = async () => {
	if (!user.value || !treasuryForm.value.accountId || !treasuryForm.value.amount) {
		showSnackbar("Faltan datos", "error");
		return;
	}
	const amount = parseFloat(treasuryForm.value.amount);
	const isDeposit = treasuryForm.value.type === "deposit";
	const txData = {
		type: treasuryForm.value.type,
		date: treasuryForm.value.date,
		amountMxn: treasuryForm.value.assetType === "MXN" ? amount : 0,
		amountUsdt: treasuryForm.value.assetType === "USDT" ? amount : 0,
		rate: 0,
		bankId: treasuryForm.value.assetType === "MXN" ? treasuryForm.value.accountId : null,
		platform:
			treasuryForm.value.assetType === "USDT"
				? accounts.value.find((a) => a.id === treasuryForm.value.accountId)?.name
				: null,
		counterparty: isDeposit ? "Fondeo Capital" : "Gasto/Retiro",
		reference: treasuryForm.value.description,
		status: "completed",
		timestamp: Date.now(),
	};
	try {
		await addDoc(collection(db, "artifacts", appId, "users", user.value.uid, "transactions"), txData);
		await updateAccountBalance(treasuryForm.value.accountId, amount, isDeposit ? "add" : "subtract");
		showSnackbar("Registrado");
		treasuryForm.value.amount = "";
		treasuryForm.value.description = "";
	} catch (e) {
		showSnackbar("Error", "error");
	}
};

const deleteTransaction = async (id) => {
	if (
		(
			await Swal.fire({
				title: "¿Eliminar?",
				icon: "warning",
				showCancelButton: true,
				confirmButtonColor: "#ef4444",
				confirmButtonText: "Sí",
			})
		).isConfirmed
	) {
		const tx = transactions.value.find((t) => t.id === id);
		if (!tx) return;
		try {
			if (tx.type === "deposit" || tx.type === "withdrawal") {
				const isDeposit = tx.type === "deposit";
				const accId = tx.amountMxn > 0 ? tx.bankId : accounts.value.find((a) => a.name === tx.platform)?.id;
				const amount = tx.amountMxn > 0 ? tx.amountMxn : tx.amountUsdt;
				if (accId) await updateAccountBalance(accId, amount, isDeposit ? "subtract" : "add");
			} else {
				await applyBalanceEffect(tx, true);
			}
			await deleteDoc(doc(db, "artifacts", appId, "users", user.value.uid, "transactions", id));
			showSnackbar("Eliminado");
		} catch (e) {
			showSnackbar("Error", "error");
		}
	}
};

const closeDay = async () => {
	if (
		(
			await Swal.fire({
				title: "¿Cerrar Día?",
				html: `Capital: <strong>${formatCurrency(totalEquity.value)}</strong>`,
				icon: "info",
				showCancelButton: true,
				confirmButtonColor: "#0f172a",
				confirmButtonText: "Cerrar",
			})
		).isConfirmed
	) {
		try {
			await addDoc(collection(db, "artifacts", appId, "users", user.value.uid, "daily_closes"), {
				date: new Date().toISOString().split("T")[0],
				timestamp: Date.now(),
				totalMxn: totalMxn.value,
				totalUsdt: totalUsdtWallet.value,
				targetRate: targetBuyRate.value,
				totalEquity: totalEquity.value,
			});
			showSnackbar("Día cerrado");
		} catch (e) {
			showSnackbar("Error", "error");
		}
	}
};

// --- CONFIRMACIONES ---
const confirmSinglePending = async (tx) => {
	try {
		let usdtSign = 0;
		if (tx.type === "buy") usdtSign = 1;
		if (tx.type === "sell") usdtSign = -1;
		const w = findUsdtAccountByName(tx.platform);
		if (usdtSign !== 0 && w) await updateAccountBalance(w.id, tx.amountUsdt, usdtSign === 1 ? "add" : "subtract");
		await updateDoc(doc(db, "artifacts", appId, "users", user.value.uid, "transactions", tx.id), {
			status: "completed",
		});
	} catch (e) {
		showSnackbar("Error", "error");
	}
};
const confirmGroup = async (group) => {
	if (
		(
			await Swal.fire({
				title: "¿Confirmar todo?",
				icon: "question",
				showCancelButton: true,
				confirmButtonColor: "#10b981",
				confirmButtonText: "Sí",
			})
		).isConfirmed
	) {
		try {
			await Promise.all(group.transactions.map((tx) => confirmSinglePending(tx)));
			showSnackbar("Liquidado");
		} catch (e) {}
	}
};
const confirmPartial = async (group) => {
	const { value: amountStr } = await Swal.fire({
		title: `Pago Parcial a ${group.counterparty}`,
		html: `Deuda Total: ${formatUSDT(group.totalUsdt)}`,
		input: "number",
		inputAttributes: { step: "0.01" },
		showCancelButton: true,
		confirmButtonText: "Pagar",
	});
	if (amountStr) {
		const pay = parseFloat(amountStr);
		let rem = pay;
		try {
			const sorted = [...group.transactions].sort((a, b) => a.timestamp - b.timestamp);
			for (const tx of sorted) {
				if (rem <= 0.001) break;
				const txAmt = parseFloat(tx.amountUsdt);
				if (rem >= txAmt) {
					await confirmSinglePending(tx);
					rem -= txAmt;
				} else {
					const newPend = txAmt - rem;
					const newMxn = (tx.amountMxn / txAmt) * newPend;
					await updateDoc(doc(db, "artifacts", appId, "users", user.value.uid, "transactions", tx.id), {
						amountUsdt: newPend,
						amountMxn: newMxn,
					});
					const paidMxn = tx.amountMxn - newMxn;
					const comp = {
						...tx,
						amountUsdt: rem,
						amountMxn: paidMxn,
						status: "completed",
						timestamp: Date.now(),
						reference: `${tx.reference || ""} (P)`,
						notes: "Pago parcial",
					};
					delete comp.id;
					await addDoc(collection(db, "artifacts", appId, "users", user.value.uid, "transactions"), comp);
					let sign = 0;
					if (tx.type === "buy") sign = 1;
					if (tx.type === "sell") sign = -1;
					const w = findUsdtAccountByName(tx.platform);
					if (w && sign !== 0) await updateAccountBalance(w.id, comp.amountUsdt, sign === 1 ? "add" : "subtract");
					rem = 0;
				}
			}
			showSnackbar("Pago aplicado");
		} catch (e) {
			showSnackbar("Error", "error");
		}
	}
};

const startEdit = (tx) => {
	if (tx.type === "deposit" || tx.type === "withdrawal") {
		showSnackbar("Edita tesorería borrando y creando.", "info");
		return;
	}
	isEditing.value = true;
	editingId.value = tx.id;
	const isInCatalog = counterparties.value.some((c) => c.name === tx.counterparty);
	form.value = {
		...tx,
		isPending: tx.status === "pending",
		counterpartySelect: isInCatalog ? tx.counterparty : "__OTHER__",
		counterparty: tx.counterparty,
		calcMode: tx.calcMode || "rate",
		commissionPercent: tx.commissionPercent !== undefined ? tx.commissionPercent : 0.16,
		paymentMethod: tx.paymentMethod || "transfer",
		destinationBankId: tx.destinationBankId || "",
	};
	activeTab.value = "operations";
};

const resetForm = (mode = "full") => {
	isEditing.value = false;
	editingId.value = null;
	const defPlat = usdtAccounts.value.length > 0 ? usdtAccounts.value[0].name : "";
	if (mode === "partial") {
		form.value.amountMxn = "";
		form.value.amountUsdt = "";
		form.value.reference = "";
		form.value.counterpartySelect = "";
		form.value.counterparty = "";
		nextTick(() => document.getElementById("amountMxnInput")?.focus());
	} else {
		form.value = {
			type: "buy",
			date: new Date().toISOString().split("T")[0],
			reference: "",
			amountMxn: "",
			rate: "",
			amountUsdt: "",
			bankId: "",
			destinationBankId: "",
			counterpartySelect: "",
			counterparty: "",
			platform: defPlat,
			isPending: false,
			calcMode: "rate",
			commissionPercent: 0.16,
			paymentMethod: "transfer",
		};
	}
};

watch(
	() => form.value.counterpartySelect,
	(newVal) => {
		if (newVal && newVal !== "__OTHER__") {
			form.value.counterparty = newVal;
			const cp = counterparties.value.find((c) => c.name === newVal);
			if (cp && cp.dailyRate > 0 && !(form.value.type === "buy" && form.value.calcMode === "amount")) {
				form.value.rate = cp.dailyRate;
				showSnackbar(`TC: ${cp.dailyRate}`);
			}
		} else if (newVal === "__OTHER__" && !isEditing.value) form.value.counterparty = "";
	}
);

watch(
	() => [
		form.value.amountMxn,
		form.value.rate,
		form.value.amountUsdt,
		form.value.type,
		form.value.calcMode,
		form.value.commissionPercent,
	],
	([mxnS, rateS, usdtS, type, mode, comm], [oldMxn, oldRate, oldUsdt]) => {
		if (type !== "buy" && type !== "sell") return;
		const mxn = parseFloat(mxnS),
			rate = parseFloat(rateS),
			usdt = parseFloat(usdtS),
			commDec = parseFloat(comm) / 100 || 0;
		if (type === "buy") {
			if (mode === "rate") {
				if (mxn && rate) form.value.amountUsdt = ((mxn / rate) * (1 - commDec)).toFixed(2);
			} else if (mode === "amount") {
				if (mxn && usdt > 0) form.value.rate = (mxn / usdt).toFixed(4);
			}
		} else if (type === "sell") {
			if (mxn && rate) form.value.amountUsdt = (mxn / rate).toFixed(2);
		}
	}
);

const renderIcons = () =>
	nextTick(() =>
		createIcons({
			icons: {
				Wallet,
				DollarSign,
				AlertCircle,
				RefreshCw,
				Calculator,
				Plus,
				Edit2,
				Save,
				History,
				CheckCircle,
				Trash2,
				Settings,
				Landmark,
				Users,
				X,
				ArrowRightLeft,
				ArrowUpRight,
				Smartphone,
				Layers,
				PieChart,
				TrendingUp,
				TrendingDown,
				Calendar,
				Search,
				LogOut,
				Lock,
			},
			attrs: { class: "w-5 h-5" },
		})
	);
watch(activeTab, renderIcons);

const formatCurrency = (val) => new Intl.NumberFormat("es-MX", { style: "currency", currency: "MXN" }).format(val);
const formatUSDT = (val) => parseFloat(val).toFixed(2) + " ₮";
const getBankName = (id) => accounts.value.find((a) => a.id === id)?.name || "N/A";
const getTypeName = (type) =>
	({ buy: "COMPRA", sell: "VENTA", deposit: "FONDEO", withdrawal: "GASTO", transfer_own: "TRASPASO" }[type] ||
	type.toUpperCase());
const getBadgeColor = (type) =>
	({
		buy: "bg-indigo-100 text-indigo-700",
		sell: "bg-emerald-100 text-emerald-700",
		transfer_own: "bg-purple-100 text-purple-700",
		deposit: "bg-teal-100 text-teal-700",
		withdrawal: "bg-rose-100 text-rose-700",
	}[type] || "bg-gray-100 text-gray-600");
</script>

<template>
	<div class="bg-slate-50 min-h-screen text-slate-800 font-sans">
		<!-- LOGIN SCREEN -->
		<div v-if="!user" class="flex items-center justify-center min-h-screen bg-slate-900 p-4">
			<div class="bg-white p-8 rounded-2xl shadow-2xl w-full max-w-md text-center">
				<div class="mb-6 flex justify-center">
					<div class="bg-emerald-500 w-16 h-16 rounded-xl flex items-center justify-center shadow-lg">
						<span class="text-3xl font-bold text-slate-900">P2P</span>
					</div>
				</div>
				<h2 class="text-2xl font-bold text-gray-800 mb-2">Bienvenido</h2>
				<p class="text-gray-500 mb-6">Accede a tu dashboard de capital</p>

				<button
					@click="handleLogin"
					class="w-full bg-white border border-gray-300 hover:bg-gray-50 text-gray-700 font-bold py-3 rounded-lg shadow-sm transform transition active:scale-95 flex justify-center items-center gap-3"
				>
					<img src="https://www.svgrepo.com/show/475656/google-color.svg" class="w-6 h-6" alt="Google" />
					<span>Entrar con Google</span>
				</button>
			</div>
		</div>

		<!-- APP CONTENT -->
		<div v-else class="pb-20">
			<!-- NAVBAR -->
			<nav class="bg-slate-900 text-white p-4 sticky top-0 z-50 shadow-xl">
				<div class="container mx-auto flex flex-wrap justify-between items-center gap-4">
					<div class="flex items-center gap-2">
						<div
							class="w-8 h-8 bg-emerald-500 rounded text-slate-900 flex items-center justify-center font-bold shadow-[0_0_10px_rgba(16,185,129,0.5)]"
						>
							P2P
						</div>
						<span class="font-bold text-lg tracking-tight hidden md:block">Control Capital Cloud</span>
					</div>
					<div class="flex gap-2 text-sm overflow-x-auto items-center no-scrollbar">
						<button
							@click="activeTab = 'dashboard'"
							:class="[
								'px-3 py-2 rounded-lg transition-all flex items-center gap-1',
								activeTab === 'dashboard'
									? 'bg-emerald-600 text-white shadow-lg'
									: 'hover:bg-slate-800 text-slate-300',
							]"
						>
							<i data-lucide="pie-chart" class="w-4 h-4"></i> <span class="hidden sm:inline">Dash</span>
						</button>
						<button
							@click="
								resetForm();
								activeTab = 'operations';
							"
							:class="[
								'px-3 py-2 rounded-lg transition-all flex items-center gap-1',
								activeTab === 'operations'
									? 'bg-emerald-600 text-white shadow-lg'
									: 'hover:bg-slate-800 text-slate-300',
							]"
						>
							<i data-lucide="plus" class="w-4 h-4"></i> <span class="hidden sm:inline">Operar</span>
						</button>
						<button
							@click="activeTab = 'treasury'"
							:class="[
								'px-3 py-2 rounded-lg transition-all flex items-center gap-1',
								activeTab === 'treasury' ? 'bg-emerald-600 text-white shadow-lg' : 'hover:bg-slate-800 text-slate-300',
							]"
						>
							<i data-lucide="landmark" class="w-4 h-4"></i> <span class="hidden sm:inline">Tesorería</span>
						</button>
						<button
							@click="activeTab = 'history'"
							:class="[
								'px-3 py-2 rounded-lg transition-all flex items-center gap-1',
								activeTab === 'history' ? 'bg-emerald-600 text-white shadow-lg' : 'hover:bg-slate-800 text-slate-300',
							]"
						>
							<i data-lucide="history" class="w-4 h-4"></i> <span class="hidden sm:inline">Historial</span>
						</button>
						<button
							@click="activeTab = 'accounts'"
							:class="[
								'px-3 py-2 rounded-lg transition-all flex items-center gap-1',
								activeTab === 'accounts' ? 'bg-emerald-600 text-white shadow-lg' : 'hover:bg-slate-800 text-slate-300',
							]"
						>
							<i data-lucide="settings" class="w-4 h-4"></i> <span class="hidden sm:inline">Catálogos</span>
						</button>
						<div class="w-px h-6 bg-slate-700 mx-1"></div>
						<button @click="handleLogout" class="text-slate-400 hover:text-red-400 p-2" title="Cerrar Sesión">
							<i data-lucide="log-out" class="w-5 h-5"></i>
						</button>
					</div>
				</div>
			</nav>

			<main class="container mx-auto p-4 md:p-6 space-y-6">
				<!-- ALERT SI NO HAY CUENTAS -->
				<div
					v-if="accounts.length === 0 && activeTab !== 'accounts'"
					class="bg-blue-50 border border-blue-200 p-4 rounded-lg flex items-center gap-3 animate-fade-in"
				>
					<i data-lucide="alert-circle" class="text-blue-500"></i>
					<div>
						<h3 class="font-bold text-blue-800">¡Bienvenido!</h3>
						<p class="text-sm text-blue-600">
							Empieza configurando tus cuentas y clientes en la pestaña <strong>Catálogos</strong>.
						</p>
					</div>
				</div>

				<!-- ================= DASHBOARD ================= -->
				<div v-if="activeTab === 'dashboard'" class="space-y-6">
					<!-- 1. KPI CARDS PRINCIPALES -->
					<div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-4 gap-4">
						<div class="bg-white p-6 rounded-xl shadow-sm border border-gray-100 flex items-start justify-between">
							<div>
								<p class="text-gray-500 text-sm font-medium mb-1">MXN en Bancos</p>
								<h3 class="text-2xl font-bold text-gray-800">{{ formatCurrency(totalMxn) }}</h3>
							</div>
							<div class="p-3 rounded-lg bg-blue-100 text-blue-600"><i data-lucide="wallet" class="w-6 h-6"></i></div>
						</div>
						<div class="bg-white p-6 rounded-xl shadow-sm border border-gray-100 flex items-start justify-between">
							<div>
								<p class="text-gray-500 text-sm font-medium mb-1">USDT en Wallets</p>
								<h3 class="text-2xl font-bold text-gray-800">{{ formatUSDT(totalUsdtWallet) }}</h3>
							</div>
							<div class="p-3 rounded-lg bg-emerald-100 text-emerald-600">
								<i data-lucide="dollar-sign" class="w-6 h-6"></i>
							</div>
						</div>
						<div class="bg-white p-6 rounded-xl shadow-sm border-l-4 border-orange-500 flex flex-col justify-between">
							<div class="flex justify-between items-start">
								<div>
									<p class="text-gray-500 text-sm font-medium">Deuda Clientes</p>
									<h3 class="text-2xl font-bold text-gray-800">{{ formatUSDT(usdtOwedToClients) }}</h3>
								</div>
								<i data-lucide="alert-circle" class="text-orange-500 w-6 h-6"></i>
							</div>
						</div>
						<div class="bg-white p-6 rounded-xl shadow-sm border-l-4 border-indigo-500 flex flex-col justify-between">
							<div class="flex justify-between items-start">
								<div>
									<p class="text-gray-500 text-sm font-medium">Por Cobrar</p>
									<h3 class="text-2xl font-bold text-gray-800">{{ formatUSDT(usdtReceivable) }}</h3>
								</div>
								<i data-lucide="refresh-cw" class="text-indigo-500 w-6 h-6"></i>
							</div>
						</div>
					</div>

					<!-- 2. BARRA DE GANANCIA Y CIERRE -->
					<div
						class="bg-slate-900 rounded-xl p-4 text-white shadow-lg border border-slate-700 grid grid-cols-1 md:grid-cols-3 gap-4 items-center"
					>
						<!-- Ganancia -->
						<div class="flex flex-col">
							<span class="text-xs text-slate-400 uppercase font-bold flex items-center gap-1"
								><i data-lucide="trending-up" class="w-3 h-3"></i> Ganancia Estimada Hoy</span
							>
							<div class="text-3xl font-bold" :class="dailyProfit >= 0 ? 'text-emerald-400' : 'text-rose-400'">
								{{ dailyProfit >= 0 ? "+" : "" }}{{ formatCurrency(dailyProfit) }}
							</div>
						</div>

						<!-- Tesorería Mensual -->
						<div class="flex gap-6 border-l border-slate-700 pl-6">
							<div>
								<span class="text-xs text-slate-400 uppercase">Fondeo Mes</span>
								<div class="text-lg font-bold text-teal-400">{{ formatCurrency(monthlyStats.funded) }}</div>
							</div>
							<div>
								<span class="text-xs text-slate-400 uppercase">Gastos Mes</span>
								<div class="text-lg font-bold text-rose-400">{{ formatCurrency(monthlyStats.expenses) }}</div>
							</div>
						</div>

						<!-- Botón Cierre -->
						<div class="text-right">
							<div class="text-sm text-slate-400 mb-1">
								Capital Total: <span class="font-bold text-white">{{ formatCurrency(totalEquity) }}</span>
							</div>
							<button
								@click="closeDay"
								class="bg-indigo-600 hover:bg-indigo-700 text-white px-4 py-2 rounded-lg text-sm font-bold flex items-center justify-center gap-2 transition-all w-full md:w-auto ml-auto"
							>
								<i data-lucide="save" class="w-4 h-4"></i> Cerrar Día
							</button>
						</div>
					</div>

					<!-- 3. CALCULADORA DE COBERTURA -->
					<div
						class="bg-gradient-to-r from-slate-800 to-slate-700 rounded-xl shadow-xl p-6 text-white border border-slate-600"
					>
						<div class="flex items-center gap-2 mb-4 border-b border-slate-600 pb-2">
							<i data-lucide="calculator" class="text-emerald-400 w-5 h-5"></i>
							<h3 class="font-bold text-lg">Calculadora de Cobertura</h3>
						</div>
						<div class="grid grid-cols-1 md:grid-cols-3 gap-8 items-center">
							<div class="space-y-1">
								<div class="flex justify-between text-sm">
									<span>Debo a clientes:</span
									><span class="font-mono text-red-400">-{{ formatUSDT(usdtOwedToClients) }}</span>
								</div>
								<div class="flex justify-between text-sm">
									<span>Tengo en Wallet:</span
									><span class="font-mono text-emerald-400">+{{ formatUSDT(totalUsdtWallet) }}</span>
								</div>
								<div class="flex justify-between text-sm">
									<span>Viene en camino:</span
									><span class="font-mono text-indigo-400">+{{ formatUSDT(usdtReceivable) }}</span>
								</div>
								<div class="border-t border-slate-500 mt-2 pt-2 flex justify-between font-bold text-lg">
									<span>Déficit:</span>
									<span :class="usdtDeficit > 0 ? 'text-yellow-400' : 'text-emerald-400'">{{
										usdtDeficit > 0 ? formatUSDT(usdtDeficit) : "Cubierto"
									}}</span>
								</div>
							</div>
							<div class="bg-slate-600/50 p-4 rounded-lg border border-slate-500">
								<label class="text-xs text-slate-300 block mb-1">TC Esperado de Compra</label>
								<div class="flex items-center gap-2">
									<span class="text-emerald-400 font-bold">$</span
									><input
										type="number"
										v-model="targetBuyRate"
										class="bg-transparent border-b border-emerald-500 text-white font-bold text-xl w-full focus:outline-none"
									/>
								</div>
							</div>
							<div class="text-right">
								<p class="text-slate-400 text-xs uppercase font-bold">Necesitas Depositar</p>
								<h2 class="text-4xl font-bold text-emerald-400 tracking-tighter">
									{{ formatCurrency(mxnNeededToCover) }}
								</h2>
							</div>
						</div>
					</div>

					<!-- 4. TABLAS: SALDOS Y PENDIENTES -->
					<div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
						<!-- Saldos Bancos -->
						<div class="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden">
							<div class="bg-blue-600 px-6 py-3 border-b border-gray-100 flex justify-between items-center">
								<h3 class="font-bold text-white text-sm">Saldos MXN</h3>
								<span class="text-white font-mono font-bold">{{ formatCurrency(totalMxn) }}</span>
							</div>
							<table class="w-full text-sm">
								<tbody class="divide-y divide-gray-100">
									<tr v-for="acc in mxnAccounts" :key="acc.id" class="hover:bg-gray-50">
										<td class="px-6 py-3">{{ acc.name }}</td>
										<td class="px-6 py-3 text-right font-medium">{{ formatCurrency(acc.balance) }}</td>
									</tr>
								</tbody>
							</table>
						</div>

						<!-- Pendientes Agrupados -->
						<div class="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden">
							<div class="bg-orange-500 px-6 py-3 flex justify-between items-center">
								<h3 class="font-bold text-white">Pendientes</h3>
								<span class="bg-white/20 text-white text-xs px-2 py-1 rounded"
									>{{ groupedPending.length }} Grupos</span
								>
							</div>
							<div class="p-0 max-h-80 overflow-y-auto divide-y divide-gray-100">
								<div v-if="groupedPending.length === 0" class="p-8 text-center text-gray-400">Todo conciliado.</div>

								<div v-for="group in groupedPending" :key="group.id" class="p-4 hover:bg-orange-50 transition-colors">
									<div class="flex justify-between items-start mb-2">
										<div>
											<h4 class="font-bold text-gray-800 text-lg flex items-center gap-2">
												{{ group.counterparty }}
												<!-- Botón para Editar/Identificar rápido -->
												<button
													@click="activeTab = 'history'"
													title="Ir al historial para editar"
													class="text-gray-400 hover:text-indigo-600"
												>
													<i data-lucide="search" class="w-3 h-3"></i>
												</button>
											</h4>
											<div class="flex items-center gap-2 text-xs text-gray-500">
												<span
													:class="
														group.type === 'buy'
															? 'text-indigo-600 bg-indigo-50 px-2 rounded'
															: 'text-emerald-600 bg-emerald-50 px-2 rounded'
													"
												>
													{{ group.type === "buy" ? "POR COBRAR" : "POR PAGAR" }}
												</span>
												<span>• {{ group.count }} movs</span>
											</div>
										</div>
										<div class="text-right">
											<div class="font-mono text-xl font-bold text-gray-800">{{ formatUSDT(group.totalUsdt) }}</div>
											<div class="text-xs text-gray-400">{{ formatCurrency(group.totalMxn) }} MXN</div>
										</div>
									</div>

									<div class="flex gap-2 mt-3">
										<button
											@click="confirmGroup(group)"
											class="flex-1 bg-slate-800 hover:bg-slate-700 text-white py-2 rounded text-sm flex items-center justify-center gap-2"
										>
											<i data-lucide="check-circle" class="w-4 h-4"></i> OK TODO
										</button>
										<button
											@click="confirmPartial(group)"
											class="flex-1 bg-white border border-slate-300 hover:bg-slate-50 text-slate-700 py-2 rounded text-sm flex items-center justify-center gap-2"
										>
											<i data-lucide="pie-chart" class="w-4 h-4"></i> PARCIAL
										</button>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>

				<!-- ================= TESORERÍA ================= -->
				<div v-if="activeTab === 'treasury'" class="max-w-2xl mx-auto space-y-6">
					<div class="bg-white rounded-xl shadow-md overflow-hidden border border-gray-100">
						<div class="bg-slate-800 px-6 py-4">
							<h2 class="text-xl font-bold text-white flex items-center gap-2">
								<i data-lucide="landmark" class="w-5 h-5"></i> Movimientos de Capital
							</h2>
						</div>
						<form @submit.prevent="handleTreasurySubmit" class="p-6 space-y-6">
							<div class="grid grid-cols-2 gap-4">
								<label
									class="border rounded-lg p-3 cursor-pointer hover:bg-teal-50 flex flex-col items-center gap-2 transition-colors"
									:class="{ 'border-teal-500 bg-teal-50 text-teal-700': treasuryForm.type === 'deposit' }"
								>
									<input type="radio" value="deposit" v-model="treasuryForm.type" class="hidden" />
									<i data-lucide="trending-up" class="w-6 h-6"></i>
									<span class="font-bold text-sm">FONDEO (Ingreso)</span>
								</label>
								<label
									class="border rounded-lg p-3 cursor-pointer hover:bg-rose-50 flex flex-col items-center gap-2 transition-colors"
									:class="{ 'border-rose-500 bg-rose-50 text-rose-700': treasuryForm.type === 'withdrawal' }"
								>
									<input type="radio" value="withdrawal" v-model="treasuryForm.type" class="hidden" />
									<i data-lucide="trending-down" class="w-6 h-6"></i>
									<span class="font-bold text-sm">GASTO (Retiro)</span>
								</label>
							</div>
							<div class="grid grid-cols-2 gap-4">
								<div class="space-y-2">
									<label class="text-xs font-bold text-gray-500 uppercase">Fecha</label>
									<input
										type="date"
										v-model="treasuryForm.date"
										class="w-full p-2 border border-gray-300 rounded-lg"
									/>
								</div>
								<div class="space-y-2">
									<label class="text-xs font-bold text-gray-500 uppercase">Moneda</label>
									<select
										v-model="treasuryForm.assetType"
										class="w-full p-2 border border-gray-300 rounded-lg bg-white"
									>
										<option value="MXN">MXN</option>
										<option value="USDT">USDT</option>
									</select>
								</div>
							</div>
							<div class="space-y-2">
								<label class="text-xs font-bold text-gray-500 uppercase">Cuenta Afectada</label>
								<select
									v-model="treasuryForm.accountId"
									required
									class="w-full p-2 border border-gray-300 rounded-lg bg-white"
								>
									<option value="" disabled>Seleccionar...</option>
									<template v-if="treasuryForm.assetType === 'MXN'">
										<option v-for="acc in mxnAccounts" :key="acc.id" :value="acc.id">
											{{ acc.name }} ({{ formatCurrency(acc.balance) }})
										</option>
									</template>
									<template v-else>
										<option v-for="acc in usdtAccounts" :key="acc.id" :value="acc.id">
											{{ acc.name }} ({{ formatUSDT(acc.balance) }})
										</option>
									</template>
								</select>
							</div>
							<div class="space-y-2">
								<label class="text-xs font-bold text-gray-500 uppercase">Monto</label>
								<input
									type="number"
									step="0.01"
									v-model="treasuryForm.amount"
									required
									class="w-full p-2 border border-gray-300 rounded-lg font-bold text-xl"
									placeholder="0.00"
								/>
							</div>
							<div class="space-y-2">
								<label class="text-xs font-bold text-gray-500 uppercase">Descripción</label>
								<input
									type="text"
									v-model="treasuryForm.description"
									class="w-full p-2 border border-gray-300 rounded-lg"
									placeholder="Ej. Inyección socio, Pago luz, Retiro personal"
								/>
							</div>
							<button
								type="submit"
								class="w-full bg-slate-900 hover:bg-slate-800 text-white font-bold py-3 rounded-lg flex justify-center gap-2 transition-all"
							>
								<i data-lucide="save" class="w-5 h-5"></i> Registrar Movimiento
							</button>
						</form>
					</div>
				</div>

				<!-- ================= OPERAR ================= -->
				<div
					v-if="activeTab === 'operations'"
					class="max-w-2xl mx-auto bg-white rounded-xl shadow-md overflow-hidden border border-gray-100"
				>
					<div :class="[isEditing ? 'bg-amber-500' : 'bg-emerald-600', 'px-6 py-4 transition-colors']">
						<h2 class="text-xl font-bold text-white flex items-center gap-2">
							<i :data-lucide="isEditing ? 'edit-2' : 'plus'" class="w-5 h-5"></i>
							{{ isEditing ? "Editando Movimiento" : "Nueva Operación" }}
						</h2>
					</div>
					<form @submit.prevent="handleTransactionSubmit" class="p-6 space-y-6">
						<!-- (Formulario de Operaciones P2P Estándar) -->
						<div class="grid grid-cols-2 gap-4">
							<div class="space-y-2">
								<label class="text-xs font-bold text-gray-500 uppercase">Tipo</label>
								<select v-model="form.type" class="w-full p-2 border border-gray-300 rounded-lg">
									<option value="buy">Comprar USDT</option>
									<option value="sell">Vender USDT</option>
									<option value="transfer_own" class="bg-indigo-50 font-bold text-indigo-800">
										Mover Saldo entre Mis Cuentas
									</option>
								</select>
							</div>
							<div class="space-y-2">
								<label class="text-xs font-bold text-gray-500 uppercase">Fecha</label
								><input
									type="date"
									v-model="form.date"
									required
									class="w-full p-2 border border-gray-300 rounded-lg"
								/>
							</div>
						</div>

						<!-- MODO COMPRA: TIPO DE SALIDA -->
						<div v-if="form.type === 'buy'" class="space-y-2">
							<label class="text-xs font-bold text-gray-500 uppercase">Método de Salida</label>
							<div class="grid grid-cols-2 gap-4">
								<label
									class="flex items-center gap-2 border rounded-lg p-2 cursor-pointer hover:bg-gray-50"
									:class="{ 'border-indigo-500 bg-indigo-50': form.paymentMethod === 'transfer' }"
								>
									<input type="radio" value="transfer" v-model="form.paymentMethod" class="text-indigo-600" />
									<div class="flex items-center gap-2 text-sm font-medium text-gray-700">
										<i data-lucide="arrow-up-right" class="w-4 h-4"></i> Transferencia
									</div>
								</label>
								<label
									class="flex items-center gap-2 border rounded-lg p-2 cursor-pointer hover:bg-gray-50"
									:class="{ 'border-red-500 bg-red-50': form.paymentMethod === 'cash_withdrawal' }"
								>
									<input type="radio" value="cash_withdrawal" v-model="form.paymentMethod" class="text-red-600" />
									<div class="flex items-center gap-2 text-sm font-medium text-gray-700">
										<i data-lucide="smartphone" class="w-4 h-4"></i> Retiro Efectivo
									</div>
								</label>
							</div>
						</div>

						<!-- MODO TRASPASO -->
						<div
							v-if="form.type === 'transfer_own'"
							class="bg-purple-50 p-4 rounded-lg border border-purple-200 space-y-4"
						>
							<div class="flex items-center gap-2 text-purple-700 font-bold border-b border-purple-200 pb-2 mb-2">
								<i data-lucide="arrow-right-left" class="w-5 h-5"></i> Movimiento Interno
							</div>
							<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
								<div class="space-y-1">
									<label class="text-xs font-bold text-gray-500 uppercase">De Cuenta (Origen)</label>
									<select v-model="form.bankId" required class="w-full p-2 border border-gray-300 rounded-lg bg-white">
										<option value="" disabled>Seleccionar...</option>
										<option v-for="acc in mxnAccounts" :key="acc.id" :value="acc.id">
											{{ acc.name }} ({{ formatCurrency(acc.balance) }})
										</option>
									</select>
								</div>
								<div class="space-y-1">
									<label class="text-xs font-bold text-gray-500 uppercase">A Cuenta (Destino)</label>
									<select
										v-model="form.destinationBankId"
										required
										class="w-full p-2 border border-gray-300 rounded-lg bg-white"
									>
										<option value="" disabled>Seleccionar...</option>
										<option v-for="acc in mxnAccounts" :key="acc.id" :value="acc.id">
											{{ acc.name }} ({{ formatCurrency(acc.balance) }})
										</option>
									</select>
								</div>
							</div>
							<div class="space-y-1">
								<label class="text-xs font-bold text-gray-500 uppercase">Monto a Mover</label>
								<input
									id="amountMxnInput"
									type="number"
									step="0.01"
									v-model="form.amountMxn"
									required
									class="w-full p-2 border border-gray-300 rounded-lg font-bold text-purple-800"
								/>
							</div>
						</div>

						<!-- FORMULARIO STANDARD -->
						<div v-else>
							<div
								v-if="form.type === 'buy'"
								class="bg-indigo-50 p-3 rounded-lg border border-indigo-100 flex gap-4 mb-4"
							>
								<div class="flex items-center gap-2">
									<input
										type="radio"
										id="modeRate"
										value="rate"
										v-model="form.calcMode"
										class="text-indigo-600 focus:ring-indigo-500"
									/>
									<label for="modeRate" class="text-sm font-medium text-gray-700 cursor-pointer"
										>Por Tasa (Binance)</label
									>
								</div>
								<div class="flex items-center gap-2">
									<input
										type="radio"
										id="modeAmount"
										value="amount"
										v-model="form.calcMode"
										class="text-indigo-600 focus:ring-indigo-500"
									/>
									<label for="modeAmount" class="text-sm font-medium text-gray-700 cursor-pointer"
										>Por Cantidad (Meda)</label
									>
								</div>
							</div>

							<div class="grid grid-cols-1 md:grid-cols-2 gap-6 mb-4">
								<div class="space-y-1">
									<label class="text-xs font-bold text-gray-500 uppercase">Referencia</label
									><input type="text" v-model="form.reference" class="w-full p-2 border border-gray-300 rounded-lg" />
								</div>
								<div class="space-y-1">
									<label class="text-xs font-bold text-gray-500 uppercase">Contraparte</label>
									<select
										v-model="form.counterpartySelect"
										class="w-full p-2 border border-gray-300 rounded-lg bg-white"
									>
										<option value="" disabled>Selecciona...</option>
										<option v-for="cp in sortedCounterparties" :key="cp.id" :value="cp.name">{{ cp.name }}</option>
										<option value="__OTHER__" class="font-bold text-indigo-600">+ OTRO</option>
									</select>
									<input
										v-if="form.counterpartySelect === '__OTHER__'"
										v-model="form.counterparty"
										type="text"
										placeholder="Nombre..."
										class="w-full p-2 mt-2 border border-indigo-300 rounded-lg bg-indigo-50"
										required
									/>
								</div>
							</div>
							<div class="bg-slate-50 p-4 rounded-lg border border-slate-200 space-y-4">
								<div class="grid grid-cols-1 md:grid-cols-2 gap-4">
									<div class="space-y-1">
										<label class="text-xs font-bold text-gray-500 uppercase">Cuenta MXN</label>
										<select
											v-model="form.bankId"
											required
											class="w-full p-2 border border-gray-300 rounded-lg bg-white"
										>
											<option value="">Seleccionar...</option>
											<option v-for="acc in mxnAccounts" :key="acc.id" :value="acc.id">
												{{ acc.name }} - {{ formatCurrency(acc.balance) }}
											</option>
										</select>
									</div>
									<div class="space-y-1">
										<label class="text-xs font-bold text-gray-500 uppercase">Monto MXN</label
										><input
											id="amountMxnInput"
											type="number"
											step="0.01"
											v-model="form.amountMxn"
											required
											class="w-full p-2 border border-gray-300 rounded-lg font-bold"
										/>
									</div>
								</div>

								<div
									v-if="form.type === 'buy' || form.type === 'sell'"
									class="grid grid-cols-1 md:grid-cols-2 gap-4 pt-4 border-t border-slate-200"
								>
									<div class="space-y-1">
										<label class="text-xs font-bold text-gray-500 uppercase">Wallet Destino</label>
										<select v-model="form.platform" class="w-full p-2 border border-gray-300 rounded-lg bg-white">
											<option v-for="acc in usdtAccounts" :key="acc.id" :value="acc.name">{{ acc.name }}</option>
										</select>
									</div>
									<div class="space-y-1">
										<label class="text-xs font-bold text-gray-500 uppercase">TC</label>
										<input
											type="number"
											step="0.0001"
											v-model="form.rate"
											:readonly="form.type === 'buy' && form.calcMode === 'amount'"
											:class="[
												'w-full p-2 border rounded-lg',
												form.type === 'buy' && form.calcMode === 'amount'
													? 'bg-gray-100 text-gray-500'
													: 'border-gray-300',
											]"
										/>
									</div>
									<div v-if="form.type === 'buy' && form.calcMode === 'rate'" class="space-y-1">
										<label class="text-xs font-bold text-gray-500 uppercase">Comisión %</label>
										<div class="relative">
											<input
												type="number"
												step="0.01"
												v-model="form.commissionPercent"
												class="w-full p-2 border border-gray-300 rounded-lg"
											/>
											<span class="absolute right-3 top-2 text-gray-400">%</span>
										</div>
									</div>
									<div :class="['space-y-1', form.type === 'buy' && form.calcMode === 'rate' ? '' : 'md:col-span-2']">
										<label class="text-xs font-bold text-gray-500 uppercase"
											>USDT {{ form.type === "buy" && form.calcMode === "amount" ? "Recibidos" : "Calculado" }}</label
										>
										<input
											type="number"
											step="0.01"
											v-model="form.amountUsdt"
											:readonly="(form.type === 'buy' && form.calcMode === 'rate') || form.type === 'sell'"
											:class="[
												'w-full p-2 border rounded-lg font-bold',
												(form.type === 'buy' && form.calcMode === 'rate') || form.type === 'sell'
													? 'border-emerald-200 bg-emerald-50 text-emerald-800'
													: 'border-gray-300',
											]"
										/>
									</div>
								</div>
							</div>
							<div
								v-if="form.type === 'buy' || form.type === 'sell'"
								class="bg-yellow-50 p-3 rounded-lg border border-yellow-200 mt-4"
							>
								<div class="flex items-center gap-3">
									<input
										type="checkbox"
										id="isPending"
										v-model="form.isPending"
										class="w-5 h-5 text-orange-500 rounded border-gray-300"
									/>
									<label for="isPending" class="text-sm text-gray-700"
										><strong>Dejar como Pendiente</strong>
										<p class="text-xs text-gray-500">El dinero sale/entra del banco, pero el USDT espera.</p></label
									>
								</div>
							</div>
						</div>

						<button
							type="submit"
							class="w-full bg-slate-900 hover:bg-slate-800 text-white font-bold py-3 rounded-lg flex justify-center gap-2"
						>
							<i data-lucide="save" class="w-5 h-5"></i> {{ isEditing ? "Guardar" : "Registrar" }}
						</button>
					</form>
				</div>

				<!-- ================= HISTORIAL ================= -->
				<div
					v-if="activeTab === 'history'"
					class="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden"
				>
					<div class="bg-gray-100 px-6 py-4 border-b border-gray-200">
						<h3 class="font-bold text-gray-700 flex items-center gap-2">
							<i data-lucide="history" class="w-5 h-5"></i> Historial
						</h3>
					</div>
					<div class="overflow-x-auto">
						<table class="w-full text-sm">
							<thead class="bg-gray-50 text-xs uppercase text-gray-500">
								<tr>
									<th class="px-6 py-3 text-left">Ref</th>
									<th class="px-6 py-3 text-left">Tipo</th>
									<th class="px-6 py-3 text-left">Detalles</th>
									<th class="px-6 py-3 text-right">MXN</th>
									<th class="px-6 py-3 text-right">USDT</th>
									<th class="px-6 py-3 text-center">Estado</th>
									<th class="px-6 py-3 text-center">Acciones</th>
								</tr>
							</thead>
							<tbody class="divide-y divide-gray-100">
								<tr v-for="tx in transactions" :key="tx.id" class="hover:bg-gray-50">
									<td class="px-6 py-3">
										<div class="text-gray-800">{{ tx.date }}</div>
										<div class="text-xs text-gray-500 font-mono">{{ tx.reference || "-" }}</div>
									</td>
									<td class="px-6 py-3">
										<span :class="['px-2 py-1 rounded-full text-xs font-bold', getBadgeColor(tx.type)]">{{
											getTypeName(tx.type)
										}}</span>
										<div
											v-if="tx.type === 'buy' && tx.paymentMethod === 'cash_withdrawal'"
											class="mt-1 text-[10px] text-red-500 font-bold flex items-center gap-1"
										>
											<i data-lucide="smartphone" class="w-3 h-3"></i> RETIRO
										</div>
									</td>
									<td class="px-6 py-3 font-medium">
										<div v-if="tx.type === 'transfer_own'" class="flex items-center gap-2">
											<span>{{ getBankName(tx.bankId) }}</span>
											<i data-lucide="arrow-right-left" class="w-3 h-3 text-gray-400"></i>
											<span>{{ getBankName(tx.destinationBankId) }}</span>
										</div>
										<div v-else>
											{{ tx.counterparty }}
											<div class="text-xs text-gray-400 font-normal">{{ getBankName(tx.bankId) }}</div>
										</div>
									</td>
									<td class="px-6 py-3 text-right font-medium">{{ formatCurrency(tx.amountMxn) }}</td>
									<td class="px-6 py-3 text-right font-mono">{{ tx.amountUsdt ? formatUSDT(tx.amountUsdt) : "-" }}</td>
									<td class="px-6 py-3 text-center">
										<span
											v-if="tx.status === 'pending'"
											class="text-orange-500 font-bold text-xs flex items-center justify-center gap-1"
											><i data-lucide="alert-circle" class="w-3 h-3"></i> Pendiente</span
										>
										<span v-else class="text-emerald-500 font-bold text-xs flex items-center justify-center gap-1"
											><i data-lucide="check-circle" class="w-3 h-3"></i> OK</span
										>
									</td>
									<td class="px-6 py-3 text-center">
										<div class="flex justify-center gap-2">
											<button @click="startEdit(tx)" class="text-blue-500 hover:bg-blue-50 p-1 rounded">
												<i data-lucide="edit-2" class="w-4 h-4"></i>
											</button>
											<button @click="deleteTransaction(tx.id)" class="text-red-500 hover:bg-red-50 p-1 rounded">
												<i data-lucide="trash-2" class="w-4 h-4"></i>
											</button>
										</div>
									</td>
								</tr>
							</tbody>
						</table>
					</div>
				</div>

				<!-- ================= GESTIÓN DE CUENTAS (RESTAURADO) ================= -->
				<div v-if="activeTab === 'accounts'" class="max-w-4xl mx-auto space-y-8">
					<!-- GESTIÓN DE CONTRAPARTES -->
					<div class="bg-white rounded-xl shadow-sm border border-gray-200 overflow-hidden">
						<div class="bg-indigo-600 px-6 py-4 flex justify-between items-center">
							<h3 class="font-bold text-white flex items-center gap-2">
								<i data-lucide="users" class="w-5 h-5"></i> Catálogo de Contrapartes
							</h3>
						</div>
						<div class="p-6">
							<p class="text-sm text-gray-500 mb-4">
								Agrega aquí tus clientes o proveedores frecuentes. Puedes establecer un <strong>TC Hoy</strong> para
								agilizar la captura.
							</p>
							<form @submit.prevent="handleCounterpartySubmit" class="flex gap-4 mb-6 items-end">
								<div class="flex-1">
									<input
										v-model="counterpartyForm.name"
										type="text"
										placeholder="Nombre (Ej. Rojas)"
										class="w-full p-2 border border-gray-300 rounded-lg"
										required
									/>
								</div>
								<div class="w-32">
									<input
										v-model="counterpartyForm.dailyRate"
										type="number"
										step="0.01"
										placeholder="TC Hoy"
										class="w-full p-2 border border-gray-300 rounded-lg"
									/>
								</div>
								<button
									type="submit"
									class="bg-indigo-600 hover:bg-indigo-700 text-white font-bold py-2 px-6 rounded-lg"
								>
									Agregar
								</button>
							</form>
							<div class="grid grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-3">
								<div
									v-for="cp in sortedCounterparties"
									:key="cp.id"
									class="bg-gray-50 p-3 rounded-lg border border-gray-200 relative group"
								>
									<div class="flex justify-between items-start mb-2">
										<span class="font-medium text-gray-800 truncate pr-2" :title="cp.name">{{ cp.name }}</span>
										<button @click="deleteCounterparty(cp.id)" class="text-gray-400 hover:text-red-500">
											<i data-lucide="trash-2" class="w-4 h-4"></i>
										</button>
									</div>
									<div class="flex items-center gap-2 bg-white border border-gray-200 rounded px-2 py-1">
										<span class="text-[10px] font-bold text-gray-400 uppercase">TC</span>
										<input
											type="number"
											step="0.01"
											:value="cp.dailyRate"
											@change="updateCounterpartyRate(cp.id, $event.target.value)"
											class="w-full text-sm font-mono bg-transparent focus:outline-none text-indigo-600 font-bold"
											placeholder="-"
										/>
									</div>
								</div>
								<div v-if="counterparties.length === 0" class="col-span-full text-center text-gray-400 text-sm py-4">
									No hay contrapartes registradas.
								</div>
							</div>
						</div>
					</div>

					<!-- GESTIÓN DE CUENTAS -->
					<div class="bg-white rounded-xl shadow-sm border border-gray-200 p-6">
						<h3 class="font-bold text-gray-700 mb-4 flex items-center gap-2">
							<i data-lucide="plus" class="w-5 h-5"></i> Cuentas Bancarias / Wallets
						</h3>
						<form @submit.prevent="handleAccountSubmit" class="flex flex-col md:flex-row gap-4 items-end">
							<div class="flex-1 w-full">
								<label class="text-xs font-bold text-gray-500 uppercase">Nombre</label>
								<input
									v-model="accountForm.name"
									type="text"
									placeholder="Ej. BBVA, Binance"
									required
									class="w-full p-2 border border-gray-300 rounded-lg"
								/>
							</div>
							<div class="w-full md:w-32">
								<label class="text-xs font-bold text-gray-500 uppercase">Moneda</label>
								<select v-model="accountForm.type" class="w-full p-2 border border-gray-300 rounded-lg bg-white">
									<option value="MXN">MXN</option>
									<option value="USDT">USDT</option>
								</select>
							</div>
							<div class="w-full md:w-40">
								<label class="text-xs font-bold text-gray-500 uppercase">Saldo Inicial</label>
								<input
									v-model="accountForm.initialBalance"
									type="number"
									step="0.01"
									class="w-full p-2 border border-gray-300 rounded-lg"
								/>
							</div>
							<button
								type="submit"
								class="bg-emerald-600 hover:bg-emerald-700 text-white font-bold py-2 px-6 rounded-lg"
							>
								Crear
							</button>
						</form>

						<div class="grid grid-cols-1 md:grid-cols-2 gap-6 mt-6">
							<div class="bg-gray-50 rounded-lg border border-gray-200 overflow-hidden">
								<div class="bg-blue-100 px-4 py-2 border-b border-blue-200 text-blue-800 font-bold text-sm">MXN</div>
								<div class="divide-y divide-gray-200">
									<div v-for="acc in mxnAccounts" :key="acc.id" class="p-3 flex justify-between items-center">
										<span class="text-sm font-medium"
											>{{ acc.name }} <span class="text-gray-500">({{ formatCurrency(acc.balance) }})</span></span
										>
										<button @click="deleteAccount(acc.id)" class="text-red-400 hover:text-red-600">
											<i data-lucide="trash-2" class="w-4 h-4"></i>
										</button>
									</div>
								</div>
							</div>
							<div class="bg-gray-50 rounded-lg border border-gray-200 overflow-hidden">
								<div class="bg-emerald-100 px-4 py-2 border-b border-emerald-200 text-emerald-800 font-bold text-sm">
									USDT
								</div>
								<div class="divide-y divide-gray-200">
									<div v-for="acc in usdtAccounts" :key="acc.id" class="p-3 flex justify-between items-center">
										<span class="text-sm font-medium"
											>{{ acc.name }} <span class="text-gray-500">({{ formatUSDT(acc.balance) }})</span></span
										>
										<button @click="deleteAccount(acc.id)" class="text-red-400 hover:text-red-600">
											<i data-lucide="trash-2" class="w-4 h-4"></i>
										</button>
									</div>
								</div>
							</div>
						</div>
					</div>
				</div>
			</main>

			<!-- SNACKBAR NOTIFICATIONS -->
			<transition
				enter-active-class="transform ease-out duration-300 transition"
				enter-from-class="translate-y-2 opacity-0 sm:translate-y-0 sm:translate-x-2"
				enter-to-class="translate-y-0 opacity-100 sm:translate-x-0"
				leave-active-class="transition ease-in duration-100"
				leave-from-class="opacity-100"
				leave-to-class="opacity-0"
			>
				<div
					v-if="snackbar.show"
					:class="[
						'fixed bottom-4 right-4 px-6 py-4 rounded-lg shadow-xl text-white font-medium z-50 flex items-center gap-3 min-w-[300px]',
						snackbar.type === 'error' ? 'bg-red-600' : 'bg-slate-800 border-l-4 border-emerald-500',
					]"
				>
					<i :data-lucide="snackbar.type === 'error' ? 'alert-circle' : 'check-circle'" class="w-6 h-6 shrink-0"></i>
					<span class="flex-1">{{ snackbar.message }}</span>
				</div>
			</transition>
		</div>
	</div>
</template>
