<script lang="ts">
	import { tick, untrack } from 'svelte';

	type Discount    = { label: string; type: 'percent' | 'flat'; value: number | null };
	type Offer       = { label: string; type: 'percent' | 'flat'; off: number | null; max: number | null };
	type CreditCard  = { label: string; pct: number | null };
	type Portal      = { label: string; valueType: 'percent' | 'flat'; value: number | null };
	type FutureValue = { label: string; cashValue: number | null; points: number | null };
	type ProcessorOffer = { label: string; type: 'percent' | 'flat'; off: number | null; max: number | null };
	type GiftCard    = {
		label: string; faceValue: number | null; showAdvanced: boolean;
		discounts: Discount[]; offers: Offer[]; portals: Portal[]; cards: CreditCard[]; processorOffers: ProcessorOffer[]; futures: FutureValue[];
	};
	type LayerId = 'discount' | 'offer' | 'portal' | 'giftcard' | 'card' | 'processor' | 'future';

	type SavedEntry = {
		id: string; name: string; savedAt: string;
		merchant: string; purchasePrice: number | null; taxFlat: number | null; shippingFlat: number | null;
		discounts: Discount[]; offers: Offer[]; creditCards: CreditCard[]; portals: Portal[];
		giftCards: GiftCard[]; processorOffers: ProcessorOffer[]; futureValues: FutureValue[];
	};

	// Per-type colors used by both the main sections and the gift-card sub-sections (kept in sync)
	const C_DISCOUNT = '#e76f51', C_OFFER = '#f4a261', C_PORTAL = '#2a9d8f', C_CARD = '#4f46e5', C_PROCESSOR = '#0077cc', C_FUTURE = '#b58900';

	const LAYER_DEFS: { id: LayerId; title: string; desc: string; color: string; icon: string }[] = [
		{ id: 'discount', title: 'Store discounts',    desc: 'Sale prices, promo codes, coupons',                color: C_DISCOUNT, icon: '%' },
		{ id: 'offer',    title: 'Card-linked offers', desc: 'Amex/Chase Offers — applied as statement credits', color: C_OFFER, icon: '✦' },
		{ id: 'portal',   title: 'Cashback portals',   desc: 'Rakuten, TopCashback, airline portals',            color: C_PORTAL, icon: '↗' },
		{ id: 'giftcard', title: 'Gift card boost',    desc: 'Discounted gift cards used to pay',                color: '#9b5de5', icon: '🎁' },
		{ id: 'card',      title: 'Charged to card',        desc: 'Cashback on what you actually charge',             color: C_CARD, icon: '▭' },
		{ id: 'processor', title: 'Payment processor',     desc: 'PayPal, Paze & similar checkout rewards',          color: C_PROCESSOR, icon: '⬡' },
		{ id: 'future',    title: 'Future value',           desc: 'Loyalty points, miles, bonus gift cards & credit', color: C_FUTURE, icon: '★' },
	];

	function blankGiftCard(): GiftCard {
		return {
			label: '', faceValue: null, showAdvanced: true,
			discounts: [{ label: '', type: 'percent', value: null }],
			offers: [],
			portals: [],
			cards: [{ label: '', pct: null }],
			processorOffers: [],
			futures: [],
		};
	}

	// --- State ---
	let merchant     = $state('');
	let purchasePrice = $state<number | null>(null);
	let taxFlat      = $state<number | null>(null);
	let shippingFlat = $state<number | null>(null);
	let discounts    = $state<Discount[]>([{ label: '', type: 'flat', value: null }]);
	let offers       = $state<Offer[]>([]);
	let creditCards  = $state<CreditCard[]>([{ label: '', pct: null }]);
	let portals      = $state<Portal[]>([]);
	let giftCards    = $state<GiftCard[]>([]);
	let processorOffers = $state<ProcessorOffer[]>([]);
	let futureValues    = $state<FutureValue[]>([]);
	let openLayers   = $state<Record<LayerId, boolean>>({
		discount: true, offer: false, portal: false, giftcard: false, card: true, processor: false, future: false,
	});
	let displayedPrice = $state(0);

	const SAVES_KEY = 'cashback-saves';
	function loadSaves(): SavedEntry[] {
		try { return JSON.parse(localStorage.getItem(SAVES_KEY) ?? '[]'); } catch { return []; }
	}
	function persistSaves(entries: SavedEntry[]) {
		localStorage.setItem(SAVES_KEY, JSON.stringify(entries));
	}
	let savedEntries  = $state<SavedEntry[]>(loadSaves());
	let drawerOpen    = $state(false);
	let newSaveName   = $state('');

	// --- Helpers ---
	function fmt(n: number) {
		return (Number.isFinite(n) ? n : 0).toLocaleString('en-US', { style: 'currency', currency: 'USD' });
	}

	function futureItemValue(f: FutureValue) { return f.cashValue ?? 0; }

	// A gift card purchase is a mini-transaction that mirrors the main builder:
	// face value → purchase discounts → store credit → portal & card cashback → future value.
	function gcPaidPrice(g: GiftCard) {
		let sub = g.faceValue ?? 0;
		for (const d of g.discounts.filter(d => d.type === 'percent')) sub -= sub * ((d.value ?? 0) / 100);
		for (const d of g.discounts.filter(d => d.type === 'flat'))    sub -= (d.value ?? 0);
		return Math.max(0, sub);
	}
	function gcChargedToCard(g: GiftCard) { return gcPaidPrice(g); }
	function gcPurchaseSavings(g: GiftCard) {
		const paid    = gcPaidPrice(g);
		const charged = gcChargedToCard(g);
		let offer = 0;
		for (const o of g.offers) {
			if (o.type === 'percent') { const raw = paid * ((o.off ?? 0) / 100); offer += o.max != null ? Math.min(raw, o.max) : raw; }
			else offer += o.off ?? 0;
		}
		let portal = 0;
		for (const p of g.portals)
			portal += p.valueType === 'percent' ? paid * ((p.value ?? 0) / 100) : (p.value ?? 0);
		let card = 0;
		for (const c of g.cards) card += charged * ((c.pct ?? 0) / 100);
		let processor = 0;
		for (const po of g.processorOffers) {
			if (po.type === 'percent') { const raw = charged * ((po.off ?? 0) / 100); processor += po.max != null ? Math.min(raw, po.max) : raw; }
			else processor += po.off ?? 0;
		}
		return offer + portal + card + processor;
	}
	function gcEffectiveCost(g: GiftCard) { return gcPaidPrice(g) - gcPurchaseSavings(g); }
	function gcFutureValue(g: GiftCard) {
		let v = 0; for (const f of g.futures) v += futureItemValue(f); return v;
	}

	async function focusLastLabelIn(layerId: LayerId) {
		await tick();
		const layer = document.querySelector(`[data-layer-id="${layerId}"]`);
		const inputs = layer?.querySelectorAll<HTMLInputElement>('.item-label');
		inputs?.[inputs.length - 1]?.focus();
	}
	async function focusLastInGroup(group: string) {
		await tick();
		const el = document.querySelector(`[data-group="${group}"]`);
		const inputs = el?.querySelectorAll<HTMLInputElement>('.item-label');
		inputs?.[inputs.length - 1]?.focus();
	}

	// --- Derived calculation ---
	const result = $derived.by(() => {
		const base = purchasePrice ?? 0;
		if (base <= 0) return null;

		let discountSavings = 0, discountedSubtotal = base;
		for (const d of discounts.filter(d => d.type === 'percent')) {
			const s = discountedSubtotal * ((d.value ?? 0) / 100);
			discountSavings += s; discountedSubtotal -= s;
		}
		for (const d of discounts.filter(d => d.type === 'flat')) {
			const v = d.value ?? 0; discountSavings += v; discountedSubtotal -= v;
		}

		const taxAmount      = taxFlat      ?? 0;
		const shippingAmount = shippingFlat ?? 0;

		let portalSavings = 0;
		for (const p of portals)
			portalSavings += p.valueType === 'percent'
				? discountedSubtotal * ((p.value ?? 0) / 100) : (p.value ?? 0);

		let offerSavings = 0;
		for (const o of offers) {
			if (o.type === 'percent') {
				const raw = discountedSubtotal * ((o.off ?? 0) / 100);
				offerSavings += o.max != null ? Math.min(raw, o.max) : raw;
			} else { offerSavings += o.off ?? 0; }
		}

		// Gift cards: buying a discounted gift card lowers the cash you part with at
		// checkout; any cashback earned BUYING it is a rebate that posts later.
		const gcFaceTotal = giftCards.reduce((s, g) => s + (g.faceValue ?? 0), 0);
		let gcCheckoutSavings = 0, gcRebateSavings = 0, gcFuture = 0;
		for (const g of giftCards) {
			gcCheckoutSavings += (g.faceValue ?? 0) - gcPaidPrice(g);
			gcRebateSavings   += gcPurchaseSavings(g);
			gcFuture          += gcFutureValue(g);
		}

		const chargedToCC = Math.max(0, discountedSubtotal + taxAmount + shippingAmount - gcFaceTotal);
		let creditCardSavings = 0;
		for (const cc of creditCards) creditCardSavings += chargedToCC * ((cc.pct ?? 0) / 100);

		let processorSavings = 0;
		for (const po of processorOffers) {
			if (po.type === 'percent') {
				const raw = chargedToCC * ((po.off ?? 0) / 100);
				processorSavings += po.max != null ? Math.min(raw, po.max) : raw;
			} else { processorSavings += po.off ?? 0; }
		}

		// Tier 1 — cash you actually part with at the register.
		const checkoutReductions = discountSavings + gcCheckoutSavings;
		const youPay = Math.max(0, base + taxAmount + shippingAmount - checkoutReductions);

		// Tier 2 — cash rebates that land AFTER checkout (statement credits, portal
		// payouts, card cashback, processor rewards, gift-card purchase cashback).
		const rebates = offerSavings + portalSavings + creditCardSavings + processorSavings + gcRebateSavings;
		const netCost = Math.max(0, youPay - rebates);

		// Tier 3 — non-cash future rewards (points, miles, bonus gift cards).
		let futureValue = 0;
		for (const f of futureValues) futureValue += futureItemValue(f);
		futureValue += gcFuture;

		const totalCost    = base + taxAmount + shippingAmount;
		const totalSavings = checkoutReductions + rebates;
		const savingsPct   = totalCost > 0 ? (totalSavings / totalCost) * 100 : 0;
		return {
			base, taxAmount, shippingAmount, totalCost,
			youPay, netCost, rebates, checkoutReductions, totalSavings, savingsPct, futureValue,
			discountedSubtotal, chargedToCC, gcCheckoutSavings, gcRebateSavings,
			bucket: {
				discount: discountSavings,
				giftcard: gcCheckoutSavings + gcRebateSavings,
				offer: offerSavings,
				portal: portalSavings,
				card: creditCardSavings,
				processor: processorSavings,
			},
		};
	});

	// --- Animated price ---
	$effect(() => {
		const target = result?.youPay ?? 0;
		const start  = untrack(() => displayedPrice);
		const dur = 320, t0 = performance.now();
		let raf: number;
		function step(t: number) {
			const k = Math.min(1, (t - t0) / dur);
			displayedPrice = start + (target - start) * (1 - Math.pow(1 - k, 3));
			if (k < 1) raf = requestAnimationFrame(step);
		}
		raf = requestAnimationFrame(step);
		return () => cancelAnimationFrame(raf);
	});

	const priceParts = $derived.by(() => {
		const n = Math.max(0, displayedPrice);
		const whole = Math.floor(n);
		return { dollars: whole.toLocaleString('en-US'), cents: Math.round((n - whole) * 100).toString().padStart(2, '0') };
	});

	function layerSavings(id: LayerId): number {
		return result?.bucket[id as keyof typeof result.bucket] ?? 0;
	}
	function layerCount(id: LayerId): number {
		return { discount: discounts.length, offer: offers.length, portal: portals.length, giftcard: giftCards.length, card: creditCards.length, processor: processorOffers.length, future: futureValues.length }[id];
	}

	// --- Add / remove ---
	async function addDiscount()    { discounts    = [...discounts,    { label: '', type: 'flat', value: null }]; await focusLastLabelIn('discount'); }
	function removeDiscount(i: number)  { discounts = discounts.filter((_, j) => j !== i); }
	async function addOffer()       { offers       = [...offers,       { label: '', type: 'flat', off: null, max: null }]; await focusLastLabelIn('offer'); }
	function removeOffer(i: number)     { offers = offers.filter((_, j) => j !== i); }
	async function addCreditCard()  { creditCards  = [...creditCards,  { label: '', pct: null }]; await focusLastLabelIn('card'); }
	function removeCreditCard(i: number){ creditCards = creditCards.filter((_, j) => j !== i); }
	async function addPortal()      { portals      = [...portals,      { label: '', valueType: 'percent', value: null }]; await focusLastLabelIn('portal'); }
	function removePortal(i: number)    { portals = portals.filter((_, j) => j !== i); }
	async function addGiftCard()    {
		giftCards = [...giftCards, blankGiftCard()];
		await tick();
		const layer = document.querySelector('[data-layer-id="giftcard"]');
		const faces = layer?.querySelectorAll<HTMLInputElement>('.gc-face-input');
		faces?.[faces.length - 1]?.focus();
	}
	function removeGiftCard(i: number)  { giftCards = giftCards.filter((_, j) => j !== i); }
	async function addProcessorOffer()  { processorOffers = [...processorOffers, { label: '', type: 'flat', off: null, max: null }]; await focusLastLabelIn('processor'); }
	function removeProcessorOffer(i: number){ processorOffers = processorOffers.filter((_, j) => j !== i); }
	async function addFutureValue(){ futureValues = [...futureValues, { label: '', cashValue: null, points: null }]; await focusLastLabelIn('future'); }
	function removeFutureValue(i: number){ futureValues = futureValues.filter((_, j) => j !== i); }

	// Nested gift-card sub-item add/remove (mirrors the main sections)
	function gcAdd(g: GiftCard, i: number, kind: 'discount' | 'offer' | 'portal' | 'card' | 'processor' | 'future') {
		if (kind === 'discount')        { g.discounts.push({ label: '', type: 'percent', value: null }); focusLastInGroup(`gc-${i}-discount`); }
		else if (kind === 'offer')      { g.offers.push({ label: '', type: 'flat', off: null, max: null }); focusLastInGroup(`gc-${i}-offer`); }
		else if (kind === 'portal')     { g.portals.push({ label: '', valueType: 'percent', value: null }); focusLastInGroup(`gc-${i}-portal`); }
		else if (kind === 'card')       { g.cards.push({ label: '', pct: null }); focusLastInGroup(`gc-${i}-card`); }
		else if (kind === 'processor')  { g.processorOffers.push({ label: '', type: 'flat', off: null, max: null }); focusLastInGroup(`gc-${i}-processor`); }
		else if (kind === 'future')     { g.futures.push({ label: '', cashValue: null, points: null }); focusLastInGroup(`gc-${i}-future`); }
	}
	function gcRemove<T>(arr: T[], idx: number) { arr.splice(idx, 1); }

	function addItem(id: LayerId) {
		if (id === 'discount') addDiscount();
		else if (id === 'offer') addOffer();
		else if (id === 'card') addCreditCard();
		else if (id === 'portal') addPortal();
		else if (id === 'giftcard') addGiftCard();
		else if (id === 'processor') addProcessorOffer();
		else if (id === 'future') addFutureValue();
	}

	function resetAll() {
		merchant = ''; purchasePrice = null; taxFlat = null; shippingFlat = null;
		discounts = [{ label: '', type: 'flat', value: null }]; offers = [];
		creditCards = [{ label: '', pct: null }]; portals = []; giftCards = []; processorOffers = []; futureValues = [];
	}

	// Bring older saved gift cards (flat fields) up to the current array-based shape.
	function migrateGiftCard(g: any): GiftCard {
		if (g && Array.isArray(g.discounts)) return { ...g, offers: g.offers ?? [], processorOffers: g.processorOffers ?? [] } as GiftCard;
		const discounts: Discount[] = [];
		if (g?.promoType === 'percent_off' && g.percentOff != null) discounts.push({ label: 'Discount', type: 'percent', value: g.percentOff });
		else if (g?.promoType === 'flat_off' && g.flatOff != null)  discounts.push({ label: 'Discount', type: 'flat', value: g.flatOff });
		if (discounts.length === 0) discounts.push({ label: '', type: 'percent', value: null });
		const futures: FutureValue[] = [];
		if (g?.promoType === 'bonus_gc' && g.promoBonus != null) futures.push({ label: 'Bonus GC', cashValue: g.promoBonus, points: null });
		if (g?.gcLoyaltyPoints != null) futures.push({ label: g.gcLoyaltyLabel || 'Loyalty', cashValue: ((g.gcLoyaltyPoints ?? 0) * (g.gcLoyaltyCpp ?? 0)) / 100, points: g.gcLoyaltyPoints });
		return {
			label: g?.label ?? '', faceValue: g?.faceValue ?? null, showAdvanced: false,
			discounts,
			offers: [],
			portals: g?.gcPortalValue != null ? [{ label: g.gcPortalLabel || '', valueType: g.gcPortalType ?? 'percent', value: g.gcPortalValue }] : [],
			cards: (g?.gcCardPct != null || g?.gcCardLabel) ? [{ label: g.gcCardLabel || '', pct: g.gcCardPct ?? null }] : [{ label: '', pct: null }],
			processorOffers: [],
			futures,
		};
	}

	function saveEntry() {
		const name = newSaveName.trim() || merchant || 'Untitled';
		const entry: SavedEntry = {
			id: Date.now().toString(), name,
			savedAt: new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' }),
			merchant, purchasePrice, taxFlat, shippingFlat,
			discounts: $state.snapshot(discounts) as Discount[],
			offers: $state.snapshot(offers) as Offer[],
			creditCards: $state.snapshot(creditCards) as CreditCard[],
			portals: $state.snapshot(portals) as Portal[],
			giftCards: $state.snapshot(giftCards) as GiftCard[],
			processorOffers: $state.snapshot(processorOffers) as ProcessorOffer[],
			futureValues: $state.snapshot(futureValues) as FutureValue[],
		};
		savedEntries = [entry, ...savedEntries];
		persistSaves(savedEntries);
		newSaveName = '';
	}

	function loadEntry(entry: SavedEntry) {
		merchant = entry.merchant; purchasePrice = entry.purchasePrice;
		taxFlat = entry.taxFlat; shippingFlat = entry.shippingFlat;
		discounts = entry.discounts.length ? entry.discounts : [{ label: '', type: 'flat', value: null }];
		offers = entry.offers; creditCards = entry.creditCards.length ? entry.creditCards : [{ label: '', pct: null }];
		portals = entry.portals; giftCards = (entry.giftCards ?? []).map(migrateGiftCard);
		processorOffers = entry.processorOffers ?? [];
		futureValues = (entry.futureValues ?? (entry as any).pointsEarned ?? []).map((p: any) => {
			if ('cashValue' in p) return p as FutureValue;
			const cv = p.totalValue != null ? p.totalValue : ((p.points ?? 0) * (p.cppCents ?? 0)) / 100;
			return { label: p.label ?? '', cashValue: cv || null, points: p.points ?? null };
		});
		drawerOpen = false;
	}

	function deleteEntry(id: string) {
		savedEntries = savedEntries.filter(e => e.id !== id);
		persistSaves(savedEntries);
	}

	async function copyReceipt() {
		if (!result) return;
		const lines = [
			merchant || 'Receipt',
			`Subtotal: ${fmt(result.base)}`,
			result.shippingAmount > 0 ? `Shipping: +${fmt(result.shippingAmount)}` : '',
			result.taxAmount > 0 ? `Tax: +${fmt(result.taxAmount)}` : '',
			`You pay at checkout: ${fmt(result.youPay)}`,
			result.rebates > 0 ? `Rebates back later: −${fmt(result.rebates)}` : '',
			result.rebates > 0 ? `Net cost: ${fmt(result.netCost)}` : '',
			result.futureValue > 0 ? `Future rewards: +${fmt(result.futureValue)}` : '',
			`Saved: ${fmt(result.totalSavings)} (${result.savingsPct.toFixed(1)}%)`,
		].filter(Boolean);
		try { await navigator.clipboard.writeText(lines.join('\n')); } catch {}
	}

	const dateStr = new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
</script>

<!-- Shared item-row snippets — used by both the main sections and the gift-card sub-sections -->
{#snippet discountRow(d: Discount, onRemove: () => void, base: number)}
	{@const saving = d.type === 'percent' ? base * ((d.value ?? 0) / 100) : (d.value ?? 0)}
	<div class="item-row">
		<div class="item-icon" style="background: {C_DISCOUNT}22; color: {C_DISCOUNT}">%</div>
		<div class="item-main">
			<input class="item-label" type="text" placeholder="Promo code / sale" bind:value={d.label} />
			<span class="item-meta">{d.type === 'percent' ? 'Percent off' : 'Flat amount'}</span>
		</div>
		<div class="seg">
			<button class:active={d.type === 'percent'} onclick={() => d.type = 'percent'}>%</button>
			<button class:active={d.type === 'flat'} onclick={() => d.type = 'flat'}>$</button>
		</div>
		<div class="item-value-input">
			{#if d.type === 'percent'}
				<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="10" aria-label="Discount percentage" bind:value={d.value} /><span class="suffix">%</span></div>
			{:else}
				<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0.00" aria-label="Discount amount" bind:value={d.value} /></div>
			{/if}
		</div>
		<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
		<button class="item-remove" aria-label="Remove" onclick={onRemove}>×</button>
	</div>
{/snippet}

{#snippet portalRow(p: Portal, onRemove: () => void, base: number)}
	{@const saving = p.valueType === 'percent' ? base * ((p.value ?? 0) / 100) : (p.value ?? 0)}
	<div class="item-row">
		<div class="item-icon" style="background: {C_PORTAL}22; color: {C_PORTAL}">↗</div>
		<div class="item-main">
			<input class="item-label" type="text" placeholder="Portal (e.g. Rakuten)" bind:value={p.label} />
			<span class="item-meta">{p.valueType === 'percent' ? 'Percent back' : 'Flat amount'}</span>
		</div>
		<div class="seg">
			<button class:active={p.valueType === 'percent'} onclick={() => p.valueType = 'percent'}>%</button>
			<button class:active={p.valueType === 'flat'} onclick={() => p.valueType = 'flat'}>$</button>
		</div>
		<div class="item-value-input">
			{#if p.valueType === 'percent'}
				<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="3" aria-label="Portal cashback rate" bind:value={p.value} /><span class="suffix">%</span></div>
			{:else}
				<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="5" aria-label="Portal cashback amount" bind:value={p.value} /></div>
			{/if}
		</div>
		<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
		<button class="item-remove" aria-label="Remove" onclick={onRemove}>×</button>
	</div>
{/snippet}

{#snippet cardRow(c: CreditCard, onRemove: () => void, charged: number)}
	{@const saving = charged * ((c.pct ?? 0) / 100)}
	<div class="item-row">
		<div class="item-icon" style="background: {C_CARD}22; color: {C_CARD}">▭</div>
		<div class="item-main">
			<input class="item-label" type="text" placeholder="Card name (e.g. Citi Double Cash)" bind:value={c.label} />
			<span class="item-meta">Cashback on <strong>{fmt(charged)}</strong> charged</span>
		</div>
		<div class="item-value-input">
			<div class="inp"><input type="number" min="0" max="100" step="0.01" placeholder="2" aria-label="Cashback rate" bind:value={c.pct} /><span class="suffix">%</span></div>
		</div>
		<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
		<button class="item-remove" aria-label="Remove" onclick={onRemove}>×</button>
	</div>
{/snippet}

{#snippet offerRow(o: Offer, onRemove: () => void, base: number)}
	{@const saving = o.type === 'percent' ? Math.min(o.max != null ? o.max : Infinity, base * ((o.off ?? 0) / 100)) : (o.off ?? 0)}
	<div class="item-row" class:offer-percent={o.type === 'percent'}>
		<div class="item-icon" style="background: {C_OFFER}22; color: {C_OFFER}">✦</div>
		<div class="item-main">
			<input class="item-label" type="text" placeholder="Amex / Chase Offer" bind:value={o.label} />
			<span class="item-meta">{o.type === 'flat' ? 'Statement credit' : `${o.off ?? 0}% back${o.max != null ? ` · capped ${fmt(o.max)}` : ''}`}</span>
		</div>
		<div class="seg">
			<button class:active={o.type === 'flat'} onclick={() => { o.type = 'flat'; o.max = null; }}>$</button>
			<button class:active={o.type === 'percent'} onclick={() => o.type = 'percent'}>%</button>
		</div>
		<div class="item-value-input">
			{#if o.type === 'flat'}
				<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="10" aria-label="Offer amount" bind:value={o.off} /></div>
			{:else}
				<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="5" aria-label="Offer rate" bind:value={o.off} /><span class="suffix">%</span></div>
			{/if}
		</div>
		{#if o.type === 'percent'}
			<div class="item-value-input"><div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="cap" aria-label="Maximum cashback cap" bind:value={o.max} /></div></div>
		{/if}
		<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
		<button class="item-remove" aria-label="Remove" onclick={onRemove}>×</button>
	</div>
{/snippet}

{#snippet processorRow(po: ProcessorOffer, onRemove: () => void, charged: number)}
	{@const saving = po.type === 'percent' ? Math.min(po.max != null ? po.max : Infinity, charged * ((po.off ?? 0) / 100)) : (po.off ?? 0)}
	<div class="item-row" class:offer-percent={po.type === 'percent'}>
		<div class="item-icon" style="background: {C_PROCESSOR}22; color: {C_PROCESSOR}">⬡</div>
		<div class="item-main">
			<input class="item-label" type="text" placeholder="e.g. PayPal Honey, Paze" bind:value={po.label} />
			<span class="item-meta">{po.type === 'flat' ? 'Statement credit / cashback' : `${po.off ?? 0}% back on ${fmt(charged)} charged${po.max != null ? ` · capped ${fmt(po.max)}` : ''}`}</span>
		</div>
		<div class="seg">
			<button class:active={po.type === 'flat'} onclick={() => { po.type = 'flat'; po.max = null; }}>$</button>
			<button class:active={po.type === 'percent'} onclick={() => po.type = 'percent'}>%</button>
		</div>
		<div class="item-value-input">
			{#if po.type === 'flat'}
				<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="10" aria-label="Processor offer amount" bind:value={po.off} /></div>
			{:else}
				<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="5" aria-label="Processor offer rate" bind:value={po.off} /><span class="suffix">%</span></div>
			{/if}
		</div>
		{#if po.type === 'percent'}
			<div class="item-value-input"><div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="cap" aria-label="Maximum cashback cap" bind:value={po.max} /></div></div>
		{/if}
		<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
		<button class="item-remove" aria-label="Remove" onclick={onRemove}>×</button>
	</div>
{/snippet}

{#snippet futureRow(f: FutureValue, onRemove: () => void)}
	{@const value = futureItemValue(f)}
	{@const cpp = (f.points && f.cashValue) ? ((f.cashValue / f.points) * 100).toFixed(2) : null}
	<div class="item-row">
		<div class="item-icon" style="background: {C_FUTURE}22; color: {C_FUTURE}">★</div>
		<div class="item-main">
			<input class="item-label" type="text" placeholder="Loyalty program / bonus GC" bind:value={f.label} />
			<span class="item-meta">
				{#if cpp}≈ {cpp}¢/pt{:else if value > 0}future value{:else}enter cash value below{/if}
			</span>
		</div>
		<div class="item-value-input item-value-pts"><div class="inp"><input type="number" min="0" step="1" placeholder="pts" aria-label="Number of points (optional)" bind:value={f.points} /><span class="suffix">pts</span></div></div>
		<div class="item-value-input"><div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="value" aria-label="Cash value" bind:value={f.cashValue} /></div></div>
		<span class="item-future" class:zero={value <= 0}>+{fmt(value)}</span>
		<button class="item-remove" aria-label="Remove" onclick={onRemove}>×</button>
	</div>
{/snippet}

<div class="shell">

	<!-- Top bar -->
	<div class="topbar">
		<div class="brand">
			<span class="brand-mark">G</span>
			Good Deals <small>cashback calculator</small>
		</div>
		<div class="topbar-right">
			<button class="icon-btn" onclick={() => drawerOpen = true}>
				<svg width="13" height="13" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6"><rect x="2" y="3" width="12" height="10" rx="1.5"/><path d="M5 7h6M5 10h4"/></svg>
				Saves {#if savedEntries.length > 0}<span class="saves-badge">{savedEntries.length}</span>{/if}
			</button>
			<button class="icon-btn" onclick={resetAll}>
				<svg width="13" height="13" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="1.6"><path d="M3 8a5 5 0 1 0 1.5-3.5"/><path d="M3 3v3h3"/></svg>
				Reset
			</button>
		</div>
	</div>

	<!-- Saved entries drawer -->
	{#if drawerOpen}
		<div class="drawer-backdrop" onclick={() => drawerOpen = false}></div>
		<div class="drawer">
			<div class="drawer-head">
				<span class="drawer-title">Saved entries</span>
				<button class="icon-btn" onclick={() => drawerOpen = false}>
					<svg width="13" height="13" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="2"><path d="M4 4l8 8M12 4l-8 8"/></svg>
					Close
				</button>
			</div>

			<div class="drawer-save-row">
				<div class="inp" style="flex:1;">
					<input type="text" placeholder={merchant || 'Entry name…'} bind:value={newSaveName} onkeydown={(e) => e.key === 'Enter' && saveEntry()} />
				</div>
				<button class="save-btn" onclick={saveEntry} disabled={!purchasePrice}>Save current</button>
			</div>

			{#if savedEntries.length === 0}
				<div class="drawer-empty">No saved entries yet. Fill in a purchase and click "Save current".</div>
			{:else}
				<div class="drawer-list">
					{#each savedEntries as entry}
						<div class="drawer-entry">
							<div class="drawer-entry-info">
								<span class="drawer-entry-name">{entry.name}</span>
								<span class="drawer-entry-meta">{entry.merchant || ''}{entry.merchant && entry.purchasePrice ? ' · ' : ''}{entry.purchasePrice != null ? fmt(entry.purchasePrice) : ''} · {entry.savedAt}</span>
							</div>
							<div class="drawer-entry-actions">
								<button class="de-load" onclick={() => loadEntry(entry)}>Load</button>
								<button class="de-del" onclick={() => deleteEntry(entry.id)}>×</button>
							</div>
						</div>
					{/each}
				</div>
			{/if}
		</div>
	{/if}

	<!-- Main grid -->
	<div class="grid">

		<!-- LEFT column: builder -->
		<div class="builder">

			<!-- Purchase card -->
			<div class="purchase-card">
				<h3>What you're buying</h3>
				<div class="purchase-row">
					<div class="field">
						<span class="field-label">Merchant / item</span>
						<div class="inp merchant">
							<input type="text" placeholder="e.g. PetSmart cat food" bind:value={merchant} />
						</div>
					</div>
					<div class="field">
						<span class="field-label">Subtotal</span>
						<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0.00" aria-label="Subtotal" bind:value={purchasePrice} /></div>
					</div>
					<div class="field">
						<span class="field-label">Tax</span>
						<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0.00" aria-label="Tax" bind:value={taxFlat} /></div>
					</div>
					<div class="field">
						<span class="field-label">Shipping</span>
						<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0.00" aria-label="Shipping" bind:value={shippingFlat} /></div>
					</div>
				</div>

				{#if result}
					{@const totalCost = result.base + result.taxAmount + result.shippingAmount}
					{@const segments = (Object.entries(result.bucket) as [string, number][])
						.map(([id, val]) => ({ id, val, color: LAYER_DEFS.find(l => l.id === id)?.color ?? '#888' }))
						.filter(s => s.val > 0.005)}
					<div class="hero">
						<div class="hero-row">
							<div>
								<p class="hero-label">You pay at checkout</p>
								<p class="hero-price mono">
									<span class="hero-dollar">$</span>{priceParts.dollars}<span class="hero-cents">.{priceParts.cents}</span>
								</p>
								{#if result.rebates > 0.005}
									<p class="hero-net">Net <strong class="mono">{fmt(result.netCost)}</strong> after <span class="mono">{fmt(result.rebates)}</span> back in rebates</p>
								{/if}
							</div>
							<div class="hero-right">
								{#if result.totalSavings > 0}
									<span class="hero-sticker mono">{fmt(totalCost)}</span>
								{/if}
								{#if result.totalSavings > 0}
									<span class="hero-savings">
										<svg width="11" height="11" viewBox="0 0 16 16" fill="none" stroke="currentColor" stroke-width="2.2"><path d="M4 8l3 3 5-6"/></svg>
										Saved {fmt(result.totalSavings)} · {result.savingsPct.toFixed(1)}% off
									</span>
								{:else}
									<span class="hero-savings muted">No savings stacked yet</span>
								{/if}
							</div>
						</div>

						{#if segments.length > 0}
							<div class="stack-bar-wrap">
								<div class="stack-bar">
									{#each segments as seg}
										<div class="stack-bar-seg" style="width: {Math.min(100, (seg.val / totalCost) * 100)}%; background: {seg.color};"></div>
									{/each}
									<div class="stack-bar-seg you-pay" style="width: {Math.min(100, (result.netCost / totalCost) * 100)}%;"></div>
								</div>
								<div class="stack-bar-legend">
									{#each segments as seg}
										{@const layerDef = LAYER_DEFS.find(l => l.id === seg.id)}
										<span class="legend-dot" style="--c: {seg.color}">
											{layerDef?.title} <span class="val mono">−{fmt(seg.val)}</span>
										</span>
									{/each}
									<span class="legend-dot" style="--c: var(--ink)">
										{result.rebates > 0.005 ? 'Net cost' : 'You pay'} <span class="val mono">{fmt(result.netCost)}</span>
									</span>
								</div>
							</div>
						{/if}
					</div>
				{/if}
			</div>

			<!-- Stack layers -->
			<div class="stack">
				{#each LAYER_DEFS as layer}
					{@const savings = layerSavings(layer.id)}
					{@const count   = layerCount(layer.id)}
					<div class="stack-layer" data-layer-id={layer.id}>
						<div class="layer-head" class:open={openLayers[layer.id]} class:static={count === 0} onclick={() => { if (count > 0) openLayers[layer.id] = !openLayers[layer.id]; }}>
							{#if count > 0}
								<svg class="chev" width="10" height="10" viewBox="0 0 10 10" fill="none" aria-hidden="true"><path d="M3 2l4 3-4 3" stroke="currentColor" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>
							{:else}
								<span class="chev-spacer"></span>
							{/if}
							<span class="dot" style="--c: {layer.color}"></span>
							<span class="layer-title">{layer.title}<small>{layer.desc}</small></span>
							{#if count > 0}
								{#if layer.id === 'future'}
									<span class="layer-amount" class:has-future={(result?.futureValue ?? 0) > 0.005}>
										{#if (result?.futureValue ?? 0) > 0.005}+{fmt(result?.futureValue ?? 0)}{:else}{count} item{count > 1 ? 's' : ''}{/if}
									</span>
								{:else}
									<span class="layer-amount" class:has-savings={savings > 0.005}>
										{#if savings > 0.005}−{fmt(savings)}{:else}{count} item{count > 1 ? 's' : ''}{/if}
									</span>
								{/if}
							{/if}
							<button class="layer-add" aria-label="Add to {layer.title}" onclick={(e) => { e.stopPropagation(); openLayers[layer.id] = true; addItem(layer.id); }}>+ Add</button>
						</div>

						{#if openLayers[layer.id] && count > 0}
							<div class="layer-body">

								{#if layer.id === 'discount'}
									{#each discounts as d, i}
										{@render discountRow(d, () => removeDiscount(i), purchasePrice ?? 0)}
									{/each}

								{:else if layer.id === 'offer'}
									{#each offers as o, i}
										{@render offerRow(o, () => removeOffer(i), result?.discountedSubtotal ?? (purchasePrice ?? 0))}
									{/each}

								{:else if layer.id === 'portal'}
									{#each portals as p, i}
										{@render portalRow(p, () => removePortal(i), result?.discountedSubtotal ?? (purchasePrice ?? 0))}
									{/each}

								{:else if layer.id === 'giftcard'}
									{#each giftCards as gc, i}
										{@const paid      = gcPaidPrice(gc)}
										{@const effCost   = gcEffectiveCost(gc)}
										{@const advantage = (gc.faceValue ?? 0) > 0 ? (gc.faceValue ?? 0) - effCost : 0}
										<div class="item-row item-row-block">
											<div class="item-row-inner">
												<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">🎁</div>
												<div class="item-main">
													<input class="item-label" type="text" placeholder="Discounted gift card" bind:value={gc.label} />
													<span class="item-meta">
														Face <strong>{fmt(gc.faceValue ?? 0)}</strong> · You pay <strong>{fmt(paid)}</strong> · Advantage <strong>{fmt(advantage)}</strong>
													</span>
												</div>
												<button class="add-chip" onclick={() => giftCards[i].showAdvanced = !giftCards[i].showAdvanced}>
													{gc.showAdvanced ? 'Hide' : 'Details'}
												</button>
												<button class="item-remove" aria-label="Remove" onclick={() => removeGiftCard(i)}>×</button>
											</div>
											{#if gc.showAdvanced}
												<div class="gc-detail">
													<!-- Face value + store credit (scalars specific to the GC) -->
													<div class="gc-fields">
														<div class="gc-field">
															<span class="field-label">Face value</span>
															<div class="inp"><span class="prefix">$</span><input class="gc-face-input" type="number" min="0" step="0.01" placeholder="100" aria-label="Gift card face value" bind:value={gc.faceValue} /></div>
														</div>
						
													</div>

													<!-- Purchase discounts (mirrors Store discounts) -->
													<div class="gc-sub" data-group="gc-{i}-discount">
														<div class="gc-sub-head"><span class="dot" style="--c: {C_DISCOUNT}"></span><span class="gc-sub-title">Store discounts<small>Sale prices, promo codes, coupons</small></span><button class="sub-add" aria-label="Add store discount" onclick={() => gcAdd(gc, i, 'discount')}>+ Add</button></div>
														{#each gc.discounts as d, di}
															{@render discountRow(d, () => gcRemove(gc.discounts, di), gc.faceValue ?? 0)}
														{/each}
													</div>

													<!-- Card-linked offers (mirrors Card-linked offers) -->
													<div class="gc-sub" data-group="gc-{i}-offer">
														<div class="gc-sub-head"><span class="dot" style="--c: {C_OFFER}"></span><span class="gc-sub-title">Card-linked offers<small>Amex/Chase offers on the gift card purchase</small></span><button class="sub-add" aria-label="Add card-linked offer" onclick={() => gcAdd(gc, i, 'offer')}>+ Add</button></div>
														{#each gc.offers as o, oi}
															{@render offerRow(o, () => gcRemove(gc.offers, oi), paid)}
														{/each}
													</div>

													<!-- Cashback portals (mirrors Cashback portals) -->
													<div class="gc-sub" data-group="gc-{i}-portal">
														<div class="gc-sub-head"><span class="dot" style="--c: {C_PORTAL}"></span><span class="gc-sub-title">Cashback portals<small>Portal used to buy the gift card</small></span><button class="sub-add" aria-label="Add cashback portal" onclick={() => gcAdd(gc, i, 'portal')}>+ Add</button></div>
														{#each gc.portals as p, pi}
															{@render portalRow(p, () => gcRemove(gc.portals, pi), paid)}
														{/each}
													</div>

													<!-- Card used to buy GC (mirrors Charged to card) -->
													<div class="gc-sub" data-group="gc-{i}-card">
														<div class="gc-sub-head"><span class="dot" style="--c: {C_CARD}"></span><span class="gc-sub-title">Charged to card<small>Cashback on what you charge to buy the GC</small></span><button class="sub-add" aria-label="Add card" onclick={() => gcAdd(gc, i, 'card')}>+ Add</button></div>
														{#each gc.cards as c, ci}
															{@render cardRow(c, () => gcRemove(gc.cards, ci), gcChargedToCard(gc))}
														{/each}
													</div>

													<!-- Payment processor (mirrors Payment processor) -->
													<div class="gc-sub" data-group="gc-{i}-processor">
														<div class="gc-sub-head"><span class="dot" style="--c: {C_PROCESSOR}"></span><span class="gc-sub-title">Payment processor<small>PayPal, Paze & similar checkout rewards</small></span><button class="sub-add" aria-label="Add payment processor reward" onclick={() => gcAdd(gc, i, 'processor')}>+ Add</button></div>
														{#each gc.processorOffers as po, poi}
															{@render processorRow(po, () => gcRemove(gc.processorOffers, poi), gcChargedToCard(gc))}
														{/each}
													</div>

													<!-- Future value (mirrors Future value — bonus GC, loyalty) -->
													<div class="gc-sub" data-group="gc-{i}-future">
														<div class="gc-sub-head"><span class="dot" style="--c: {C_FUTURE}"></span><span class="gc-sub-title">Future value<small>Bonus gift cards & loyalty earned buying the GC</small></span><button class="sub-add" aria-label="Add future value" onclick={() => gcAdd(gc, i, 'future')}>+ Add</button></div>
														{#each gc.futures as f, fi}
															{@render futureRow(f, () => gcRemove(gc.futures, fi))}
														{/each}
													</div>

													<div class="gc-advantage">
														<span>Total advantage</span>
														<span>{fmt(advantage)}{(gc.faceValue ?? 0) > 0 && advantage > 0 ? ` · ${((advantage / (gc.faceValue ?? 1)) * 100).toFixed(1)}% off face` : ''}</span>
													</div>
												</div>
											{/if}
										</div>
									{/each}

								{:else if layer.id === 'card'}
									{#each creditCards as cc, i}
										{@render cardRow(cc, () => removeCreditCard(i), result?.chargedToCC ?? 0)}
									{/each}

								{:else if layer.id === 'processor'}
									{#each processorOffers as po, i}
										{@render processorRow(po, () => removeProcessorOffer(i), result?.chargedToCC ?? 0)}
									{/each}

								{:else if layer.id === 'future'}
									{#each futureValues as f, i}
										{@render futureRow(f, () => removeFutureValue(i))}
									{/each}
								{/if}

							</div>
						{/if}
					</div>
				{/each}
			</div>

			<div class="wallet-hint">
				<strong>How savings stack:</strong> discounts first, then portals and offers, then card cashback on what's left. Gift card savings lower your card charge.
			</div>
		</div>

		<!-- RIGHT column: receipt -->
		<div class="receipt-wrap">
			{#if result}
				<div class="receipt">
					<div class="receipt-head">
						<div class="receipt-merchant">{merchant || 'Receipt'}</div>
						<div class="receipt-date">{dateStr}</div>
					</div>
					<div class="receipt-divider"></div>

					<!-- Tier 1 — cash you part with at the register -->
					<div class="receipt-row">
						<span class="label">Subtotal</span>
						<span class="amt">{fmt(result.base)}</span>
					</div>

					{#each discounts as d}
						{@const v = d.type === 'percent' ? result.base * ((d.value ?? 0) / 100) : (d.value ?? 0)}
						{#if v > 0}
							<div class="receipt-row save">
								<span class="label">{d.label || 'Discount'}{#if d.type === 'percent'} <small>{d.value}%</small>{/if}</span>
								<span class="amt">{fmt(v)}</span>
							</div>
						{/if}
					{/each}

					{#each giftCards as gc}
						{@const v = (gc.faceValue ?? 0) - gcPaidPrice(gc)}
						{#if v > 0}
							<div class="receipt-row save">
								<span class="label">{gc.label || 'Gift card'} <small>gift card</small></span>
								<span class="amt">{fmt(v)}</span>
							</div>
						{/if}
					{/each}

					{#if result.shippingAmount > 0}
						<div class="receipt-row"><span class="label">Shipping</span><span class="amt">+{fmt(result.shippingAmount)}</span></div>
					{/if}
					{#if result.taxAmount > 0}
						<div class="receipt-row"><span class="label">Tax</span><span class="amt">+{fmt(result.taxAmount)}</span></div>
					{/if}

					<div class="receipt-total" class:has-more={result.rebates > 0.005}>
						<div class="label">You pay at checkout</div>
						<div class="amt mono">{fmt(result.youPay)}</div>
					</div>

					<!-- Tier 2 — cash rebates that post after checkout -->
					{#if result.rebates > 0.005}
						<div class="receipt-section">Rebates · back later</div>

						{#each offers as o}
							{@const v = o.type === 'percent' ? Math.min(o.max != null ? o.max : Infinity, result.discountedSubtotal * ((o.off ?? 0) / 100)) : (o.off ?? 0)}
							{#if v > 0}
								<div class="receipt-row save">
									<span class="label">{o.label || 'Card offer'}</span>
									<span class="amt">{fmt(v)}</span>
								</div>
							{/if}
						{/each}

						{#each portals as p}
							{@const v = p.valueType === 'percent' ? result.discountedSubtotal * ((p.value ?? 0) / 100) : (p.value ?? 0)}
							{#if v > 0}
								<div class="receipt-row save">
									<span class="label">{p.label || 'Portal'}{#if p.valueType === 'percent'} <small>{p.value}%</small>{/if}</span>
									<span class="amt">{fmt(v)}</span>
								</div>
							{/if}
						{/each}

						{#each creditCards as cc}
							{@const v = result.chargedToCC * ((cc.pct ?? 0) / 100)}
							{#if v > 0}
								<div class="receipt-row save">
									<span class="label">{cc.label || 'Card cashback'} <small>{cc.pct}%</small></span>
									<span class="amt">{fmt(v)}</span>
								</div>
							{/if}
						{/each}

						{#each processorOffers as po}
							{@const v = po.type === 'percent' ? Math.min(po.max != null ? po.max : Infinity, result.chargedToCC * ((po.off ?? 0) / 100)) : (po.off ?? 0)}
							{#if v > 0}
								<div class="receipt-row save">
									<span class="label">{po.label || 'Processor cashback'}{#if po.type === 'percent'} <small>{po.off}%</small>{/if}</span>
									<span class="amt">{fmt(v)}</span>
								</div>
							{/if}
						{/each}

						{#each giftCards as gc}
							{@const v = gcPurchaseSavings(gc)}
							{#if v > 0}
								<div class="receipt-row save">
									<span class="label">{gc.label || 'Gift card'} rebates</span>
									<span class="amt">{fmt(v)}</span>
								</div>
							{/if}
						{/each}

						<div class="receipt-total">
							<div class="label">Net cost</div>
							<div class="amt mono">{fmt(result.netCost)}</div>
						</div>
					{/if}

					<!-- Tier 3 — non-cash future rewards -->
					{#if result.futureValue > 0}
						<div class="receipt-future">
							<span>+ Future rewards</span>
							<strong>{fmt(result.futureValue)}</strong>
						</div>
					{/if}

					<div class="receipt-foot">
						<button onclick={() => window.print()}>Print</button>
						<button onclick={copyReceipt}>Copy</button>
					</div>
				</div>
			{:else}
				<div class="receipt">
					<div class="receipt-empty">
						<div class="emo">🧾</div>
						<strong>Your receipt is empty</strong><br />Enter a subtotal above to see your true out-of-pocket cost.
					</div>
				</div>
			{/if}
		</div>

	</div>
</div>

<style>
	:root {
		--bg: #f0f0eb;
		--bg-2: #e8e8e1;
		--paper: #ffffff;
		--paper-warm: #fafaf6;
		--ink: #1a1a1a;
		--ink-2: #4a4a47;
		--ink-3: #8a8a85;
		--ink-4: #c0c0b8;
		--line: #e4e4dc;
		--line-2: #d6d6cc;
		--green: #16a34a;
		--green-bg: #e9f7ed;
		--dark: #111111;
		--dark-2: #1f1f1d;
		--gold: #b58900;
	}

	.shell {
		max-width: 1200px;
		margin: 0 auto;
		padding: 28px 24px 80px;
		font-family: 'Geist', system-ui, -apple-system, sans-serif;
		-webkit-font-smoothing: antialiased;
	}
	@media (max-width: 720px) { .shell { padding: 16px 14px 60px; } }

	/* Topbar */
	.topbar {
		display: flex;
		align-items: center;
		justify-content: space-between;
		margin-bottom: 24px;
		gap: 12px;
	}
	.brand {
		display: flex;
		align-items: center;
		gap: 10px;
		font-weight: 700;
		font-size: 15px;
		letter-spacing: -0.01em;
		color: var(--ink);
	}
	.brand-mark {
		width: 26px; height: 26px;
		border-radius: 7px;
		background: var(--ink);
		color: #f0f0eb;
		display: grid; place-items: center;
		font-weight: 800; font-size: 14px;
		letter-spacing: -0.02em;
	}
	.brand :global(small) { font-weight: 500; color: var(--ink-3); margin-left: 2px; font-size: 15px; }
	.topbar-right { display: flex; gap: 6px; }
	.icon-btn {
		display: inline-flex; align-items: center; gap: 6px;
		padding: 7px 11px;
		border-radius: 8px;
		color: var(--ink-2);
		font-size: 13px; font-weight: 500;
		border: 1px solid transparent;
		background: none;
		cursor: pointer;
		transition: background .12s, border-color .12s, color .12s;
		font-family: inherit;
	}
	.icon-btn:hover { background: var(--paper-warm); border-color: var(--line); color: var(--ink); }

	/* Layout */
	.grid {
		display: grid;
		grid-template-columns: 1fr 340px;
		gap: 28px;
		align-items: start;
	}
	@media (max-width: 980px) {
		.grid { grid-template-columns: 1fr; gap: 20px; }
		.receipt-wrap { order: -1; }
	}
	.builder { display: flex; flex-direction: column; gap: 18px; }

	/* Purchase card */
	.purchase-card {
		background: var(--paper);
		border: 1px solid var(--line);
		border-radius: 14px;
		padding: 14px 16px 16px;
	}
	.purchase-card h3 {
		font-size: 11px; font-weight: 600; color: var(--ink-3);
		text-transform: uppercase; letter-spacing: 0.08em;
		margin-bottom: 10px;
	}
	.purchase-row {
		display: grid;
		grid-template-columns: 1fr 120px 110px 110px;
		gap: 10px;
	}
	@media (max-width: 600px) {
		.purchase-row { grid-template-columns: 1fr 1fr; }
		.purchase-row > :first-child { grid-column: 1 / -1; }
	}

	.field { display: flex; flex-direction: column; gap: 5px; }
	.field-label { font-size: 11px; color: var(--ink-3); font-weight: 500; }

	/* Input wrapper */
	.inp {
		display: flex; align-items: center;
		background: var(--paper-warm);
		border: 1px solid var(--line);
		border-radius: 8px;
		overflow: hidden;
		transition: border-color .12s, background .12s;
		width: 100%;
	}
	.inp:focus-within { border-color: var(--ink); background: var(--paper); }
	.inp .prefix, .inp .suffix {
		color: var(--ink-3); font-size: 13px; padding: 0 8px;
		user-select: none; font-weight: 500; flex-shrink: 0;
	}
	.inp input {
		flex: 1; min-width: 0; border: 0; outline: none; background: transparent;
		padding: 8px 4px; font-size: 14px; font-variant-numeric: tabular-nums;
		font-family: inherit; color: var(--ink);
	}
	.inp input::placeholder { color: var(--ink-4); }
	.inp.merchant input { font-weight: 500; }
	:global(input[type="number"]) { -moz-appearance: textfield; }
	:global(input[type="number"]::-webkit-outer-spin-button),
	:global(input[type="number"]::-webkit-inner-spin-button) { -webkit-appearance: none; margin: 0; }

	/* Hero (inside purchase card) */
	.hero {
		background: var(--paper);
		border: 1px solid var(--line);
		border-radius: 14px;
		padding: 20px 22px 18px;
		margin-top: 16px;
		background: var(--paper-warm);
	}
	.hero-row {
		display: flex;
		align-items: baseline;
		justify-content: space-between;
		gap: 16px;
		flex-wrap: wrap;
	}
	.hero-label {
		font-size: 11px; font-weight: 600; color: var(--ink-3);
		text-transform: uppercase; letter-spacing: 0.08em;
		margin-bottom: 2px;
	}
	.hero-price {
		font-weight: 700; font-size: 52px;
		letter-spacing: -0.035em; line-height: 1.02;
		font-variant-numeric: tabular-nums;
		color: var(--ink);
	}
	.hero-dollar { color: var(--ink-3); font-weight: 600; }
	.hero-net {
		margin-top: 6px; font-size: 12.5px; color: var(--ink-3);
	}
	.hero-net :global(strong) { color: var(--green); font-weight: 600; }
	.hero-cents {
		font-size: 0.52em; color: var(--ink-3); font-weight: 600;
		margin-left: 2px; vertical-align: 0.32em;
	}
	.hero-right { display: flex; flex-direction: column; align-items: flex-end; gap: 6px; }
	.hero-sticker {
		font-size: 18px; color: var(--ink-3); font-weight: 500;
		text-decoration: line-through; text-decoration-thickness: 1.5px;
		font-variant-numeric: tabular-nums;
	}
	.hero-savings {
		font-size: 13px; color: var(--green); font-weight: 600;
		background: var(--green-bg); padding: 4px 10px; border-radius: 6px;
		display: inline-flex; gap: 6px; align-items: center;
		font-variant-numeric: tabular-nums;
	}
	.hero-savings.muted { color: var(--ink-3); background: var(--bg-2); }

	/* Stack bar */
	.stack-bar-wrap { margin-top: 18px; }
	.stack-bar {
		display: flex; height: 12px; width: 100%;
		border-radius: 6px; overflow: hidden;
		background: var(--bg-2); border: 1px solid var(--line);
	}
	.stack-bar-seg { height: 100%; transition: width .28s ease; }
	.stack-bar-seg + .stack-bar-seg { border-left: 1px solid rgba(255,255,255,0.5); }
	.stack-bar-seg.you-pay { background: var(--ink); }
	.stack-bar-legend {
		display: flex; flex-wrap: wrap; gap: 10px 16px;
		margin-top: 10px; font-size: 12px; color: var(--ink-3); font-weight: 500;
	}
	.legend-dot { display: inline-flex; align-items: center; gap: 5px; }
	.legend-dot::before {
		content: ''; width: 8px; height: 8px; border-radius: 2px;
		background: var(--c, var(--ink-3));
	}
	.legend-dot .val { color: var(--ink); font-weight: 600; font-variant-numeric: tabular-nums; }

	/* Stack layers container */
	.stack {
		background: var(--paper);
		border: 1px solid var(--line);
		border-radius: 14px;
		overflow: hidden;
	}
	.stack-layer { border-bottom: 1px solid var(--line); }
	.stack-layer:last-child { border-bottom: 0; }

	.layer-head {
		display: flex; align-items: center; gap: 10px;
		padding: 14px 18px;
		cursor: pointer; user-select: none;
		transition: background .1s;
	}
	.layer-head:hover { background: var(--paper-warm); }
	.layer-head.static { cursor: default; }
	.layer-head.static:hover { background: none; }
	.chev { color: var(--ink-4); transition: transform .15s, color .15s; flex-shrink: 0; display: block; }
	.chev-spacer { width: 10px; flex-shrink: 0; }
	.layer-head.open .chev { transform: rotate(90deg); color: var(--ink-2); }
	.dot {
		width: 8px; height: 8px; border-radius: 50%;
		background: var(--c, var(--ink-4)); flex-shrink: 0;
	}
	.layer-title { flex: 1; font-weight: 600; font-size: 14px; color: var(--ink); }
	.layer-title :global(small) {
		font-weight: 500; color: var(--ink-3); margin-left: 8px; font-size: 12px;
	}
	.layer-amount {
		font-family: 'JetBrains Mono', monospace;
		font-size: 13px; color: var(--ink-3); font-weight: 600;
		font-variant-numeric: tabular-nums; flex-shrink: 0;
	}
	.layer-amount.has-savings { color: var(--green); }
	.layer-amount.has-future { color: var(--gold); }

	.layer-add {
		font-size: 12px; font-weight: 500; color: var(--ink-3);
		border: 1px solid var(--line); border-radius: 7px;
		padding: 4px 9px; background: var(--paper);
		cursor: pointer; font-family: inherit; flex-shrink: 0; white-space: nowrap;
		transition: background .1s, border-color .1s, color .1s;
	}
	.layer-add:hover { background: var(--bg); border-color: var(--ink-3); color: var(--ink); }

	.layer-body { padding: 4px 18px 16px; }

	/* Item rows */
	.item-row {
		display: flex; align-items: center; gap: 10px;
		padding: 8px 10px;
		border-radius: 9px;
		background: var(--paper-warm);
		border: 1px solid var(--line);
		margin-bottom: 8px;
	}
	.item-row-block { display: block; padding: 10px 12px; }
	.item-row-inner { display: flex; align-items: center; gap: 10px; }

	.item-icon {
		width: 28px; height: 28px; border-radius: 7px;
		display: grid; place-items: center;
		font-size: 13px; font-weight: 700;
		flex-shrink: 0;
	}
	.item-main { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 1px; }
	.item-label {
		font-size: 13px; font-weight: 600; color: var(--ink);
		border: 0; background: transparent; outline: none;
		padding: 0; width: 100%; font-family: inherit;
	}
	.item-label::placeholder { color: var(--ink-4); font-weight: 500; }
	.item-meta { font-size: 11px; color: var(--ink-3); font-variant-numeric: tabular-nums; }
	.item-meta :global(strong) { color: var(--ink-2); font-weight: 600; }

	.item-value-input { width: 90px; flex-shrink: 0; }
	.item-value-input .inp { width: 100%; }
	.item-value-pts { width: 80px; }

	.item-saves {
		font-family: 'JetBrains Mono', monospace;
		font-size: 12px; color: var(--green); font-weight: 600;
		min-width: 56px; text-align: right; font-variant-numeric: tabular-nums;
		flex-shrink: 0;
	}
	.item-saves.zero { color: var(--ink-4); }

	.item-future {
		font-family: 'JetBrains Mono', monospace;
		font-size: 12px; color: var(--gold); font-weight: 600;
		min-width: 56px; text-align: right; font-variant-numeric: tabular-nums;
		flex-shrink: 0;
	}
	.item-future.zero { color: var(--ink-4); }

	.item-remove {
		width: 36px; height: 36px; border-radius: 8px;
		color: var(--ink-4); display: grid; place-items: center;
		font-size: 14px; flex-shrink: 0;
		background: none; border: none; cursor: pointer;
		transition: background .1s, color .1s;
		font-family: inherit;
	}
	.item-remove:hover { background: #fde2e2; color: #c0392b; }

	/* Segmented toggle */
	.seg {
		display: inline-flex;
		border: 1px solid var(--line);
		border-radius: 6px;
		overflow: hidden;
		background: var(--bg);
		flex-shrink: 0;
	}
	.seg button {
		padding: 4px 8px; font-size: 11px; font-weight: 600;
		color: var(--ink-3); background: none; border: none; cursor: pointer;
		font-family: inherit; transition: background .1s, color .1s;
	}
	.seg button.active { background: var(--ink); color: #fff; }
	.seg button + button { border-left: 1px solid var(--line); }

	/* Add buttons */
	.add-chip {
		display: inline-flex; align-items: center; gap: 6px;
		padding: 6px 10px; border-radius: 7px;
		border: 1px dashed var(--line);
		background: transparent;
		font-size: 12px; font-weight: 500; color: var(--ink-3);
		cursor: pointer; font-family: inherit;
		transition: background .1s, border-color .1s, color .1s;
	}
	.add-chip:hover { background: var(--paper-warm); border-color: var(--ink-3); color: var(--ink); }

	.sub-add {
		font-size: 11.5px; font-weight: 500; color: var(--ink-3);
		border: 1px dashed var(--line); border-radius: 6px;
		padding: 3px 8px; background: transparent;
		cursor: pointer; font-family: inherit; flex-shrink: 0; white-space: nowrap;
		transition: background .1s, border-color .1s, color .1s;
	}
	.sub-add:hover { background: var(--paper-warm); border-color: var(--ink-3); color: var(--ink); }

	/* Gift card detail — full mirror of the main builder */
	.gc-detail {
		background: var(--bg);
		border: 1px solid var(--line);
		border-radius: 9px;
		margin-top: 8px;
		padding: 12px;
		font-size: 12px;
		color: var(--ink-2);
	}
	.gc-fields { display: flex; gap: 10px; margin-bottom: 12px; }
	.gc-field { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 5px; }
	.gc-field .inp { width: 100%; }

	.gc-sub { padding-top: 12px; margin-top: 4px; border-top: 1px dashed var(--line); }
	.gc-sub-head { display: flex; align-items: center; gap: 8px; margin-bottom: 8px; }
	.gc-sub-title { flex: 1; min-width: 0; font-weight: 600; font-size: 12.5px; color: var(--ink); }
	.gc-sub-title :global(small) { font-weight: 500; color: var(--ink-3); margin-left: 6px; font-size: 11px; }

	.gc-advantage {
		margin-top: 12px; padding-top: 10px;
		border-top: 1px solid var(--line);
		display: flex; justify-content: space-between;
		font-weight: 600; color: var(--green);
		font-size: 12.5px; font-variant-numeric: tabular-nums;
	}

	/* Receipt */
	.receipt-wrap { position: sticky; top: 16px; }
	.receipt {
		background: var(--dark);
		color: #efeee9;
		border-radius: 16px;
		padding: 20px 20px 18px;
		box-shadow: 0 10px 30px rgba(0,0,0,.08), 0 2px 6px rgba(0,0,0,.05);
	}
	.receipt-head {
		display: flex; justify-content: space-between; align-items: baseline;
		margin-bottom: 4px;
	}
	.receipt-merchant { font-size: 13px; font-weight: 600; color: #efeee9; }
	.receipt-date { font-size: 11px; color: #777; font-family: 'JetBrains Mono', monospace; }
	.receipt-divider { border-top: 1px dashed #2e2e2c; margin: 12px 0 10px; }
	.receipt-row {
		display: flex; justify-content: space-between; align-items: baseline;
		font-size: 12.5px; margin-bottom: 4px; color: #c4c3be;
		font-variant-numeric: tabular-nums;
	}
	.receipt-row .label { display: flex; align-items: center; gap: 5px; }
	.receipt-row .label :global(small) { color: #777; font-size: 10.5px; }
	.receipt-row .amt { font-family: 'JetBrains Mono', monospace; font-weight: 500; }
	.receipt-row.save .amt { color: #4ade80; }
	.receipt-row.save .label::before { content: '−'; color: #4ade80; margin-right: 1px; }
	.receipt-total {
		display: flex; justify-content: space-between; align-items: baseline;
		margin-top: 12px; padding-top: 12px;
		border-top: 1px solid #2e2e2c;
	}
	.receipt-total .label { font-size: 11px; color: #888; text-transform: uppercase; letter-spacing: 0.08em; font-weight: 600; }
	.receipt-total .amt { font-size: 26px; font-weight: 700; letter-spacing: -0.02em; font-variant-numeric: tabular-nums; }
	/* When rebates follow, the checkout total is an intermediate figure — de-emphasize it. */
	.receipt-total.has-more { margin-top: 10px; padding-top: 10px; }
	.receipt-total.has-more .amt { font-size: 18px; color: #c4c3be; font-weight: 600; }
	.receipt-section {
		font-size: 10px; font-weight: 600; color: #777;
		text-transform: uppercase; letter-spacing: 0.09em;
		margin: 12px 0 6px;
	}
	.receipt-future {
		background: rgba(255,255,255,.04);
		border-radius: 8px; padding: 8px 10px; margin-top: 12px;
		font-size: 11.5px; color: #c4c3be;
		display: flex; justify-content: space-between; align-items: baseline;
	}
	.receipt-future :global(strong) { color: #a5b4fc; font-family: 'JetBrains Mono', monospace; font-weight: 600; }
	.receipt-empty { text-align: center; padding: 30px 12px; color: #555; font-size: 13px; line-height: 1.55; }
	.receipt-empty .emo { font-size: 26px; margin-bottom: 8px; opacity: .6; }
	.receipt-empty :global(strong) { color: #c4c3be; font-weight: 600; }
	.receipt-foot { margin-top: 14px; display: flex; gap: 6px; }
	.receipt-foot button {
		flex: 1; padding: 8px;
		background: rgba(255,255,255,.06);
		border-radius: 8px; border: none;
		font-size: 12px; font-weight: 500;
		color: #c4c3be; cursor: pointer;
		font-family: inherit;
		transition: background .12s;
	}
	.receipt-foot button:hover { background: rgba(255,255,255,.11); color: #fff; }

	.wallet-hint {
		margin-top: 14px; background: var(--paper);
		border: 1px solid var(--line); border-radius: 12px;
		padding: 12px 14px; font-size: 12px; color: var(--ink-3); line-height: 1.5;
	}
	.wallet-hint :global(strong) { color: var(--ink); font-weight: 600; }

	.mono { font-family: 'JetBrains Mono', monospace; font-feature-settings: 'tnum', 'zero'; }

	/* Saves badge */
	.saves-badge {
		display: inline-flex; align-items: center; justify-content: center;
		min-width: 16px; height: 16px; padding: 0 4px;
		border-radius: 8px; background: var(--ink); color: #f0f0eb;
		font-size: 10px; font-weight: 700; line-height: 1;
		margin-left: 2px;
	}

	/* Drawer */
	.drawer-backdrop {
		position: fixed; inset: 0; background: rgba(0,0,0,.25);
		z-index: 100; backdrop-filter: blur(2px);
	}
	.drawer {
		position: fixed; top: 0; left: 0; bottom: 0;
		width: 380px; max-width: 92vw;
		background: var(--paper);
		border-right: 1px solid var(--line);
		z-index: 101;
		display: flex; flex-direction: column;
		box-shadow: 4px 0 24px rgba(0,0,0,.10);
		overflow: hidden;
	}
	.drawer-head {
		display: flex; align-items: center; justify-content: space-between;
		padding: 18px 20px 14px;
		border-bottom: 1px solid var(--line);
		flex-shrink: 0;
	}
	.drawer-title { font-weight: 700; font-size: 15px; color: var(--ink); }

	.drawer-save-row {
		display: flex; gap: 8px; align-items: center;
		padding: 14px 16px;
		border-bottom: 1px solid var(--line);
		flex-shrink: 0;
	}
	.save-btn {
		padding: 8px 14px; border-radius: 8px;
		background: var(--ink); color: #f0f0eb;
		font-size: 12px; font-weight: 600;
		border: none; cursor: pointer; white-space: nowrap;
		font-family: inherit;
		transition: opacity .12s;
		flex-shrink: 0;
	}
	.save-btn:disabled { opacity: .4; cursor: not-allowed; }
	.save-btn:not(:disabled):hover { opacity: .85; }

	.drawer-empty {
		padding: 28px 20px; font-size: 13px; color: var(--ink-3); line-height: 1.5;
	}
	.drawer-list { flex: 1; overflow-y: auto; padding: 8px 12px 20px; }

	.drawer-entry {
		display: flex; align-items: center; gap: 10px;
		padding: 10px 10px;
		border-radius: 10px;
		border: 1px solid var(--line);
		background: var(--paper-warm);
		margin-bottom: 6px;
	}
	.drawer-entry-info { flex: 1; min-width: 0; display: flex; flex-direction: column; gap: 2px; }
	.drawer-entry-name { font-size: 13px; font-weight: 600; color: var(--ink); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
	.drawer-entry-meta { font-size: 11px; color: var(--ink-3); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
	.drawer-entry-actions { display: flex; gap: 4px; flex-shrink: 0; }
	.de-load {
		padding: 5px 10px; border-radius: 6px; font-size: 12px; font-weight: 600;
		background: var(--ink); color: #f0f0eb; border: none; cursor: pointer;
		font-family: inherit; transition: opacity .1s;
	}
	.de-load:hover { opacity: .8; }
	.de-del {
		width: 26px; height: 26px; border-radius: 6px; font-size: 15px;
		background: none; border: 1px solid var(--line); color: var(--ink-4);
		cursor: pointer; display: grid; place-items: center;
		transition: background .1s, color .1s;
	}
	.de-del:hover { background: #fde2e2; border-color: #fca5a5; color: #c0392b; }
</style>
