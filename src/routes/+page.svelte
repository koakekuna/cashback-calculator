<script lang="ts">
	import { tick, untrack } from 'svelte';

	type Discount   = { label: string; type: 'percent' | 'flat'; value: number | null };
	type Offer      = { label: string; type: 'percent' | 'flat'; off: number | null; max: number | null };
	type CreditCard = { label: string; pct: number | null };
	type Portal     = { label: string; valueType: 'percent' | 'flat'; value: number | null };
	type GiftCard   = {
		label: string; faceValue: number | null;
		promoType: 'percent_off' | 'flat_off' | 'bonus_gc';
		percentOff: number | null; flatOff: number | null; promoBonus: number | null;
		storeCredit: number | null;
		gcCardLabel: string; gcCardPct: number; showAdvanced: boolean;
		gcPortalLabel: string; gcPortalType: 'percent' | 'flat'; gcPortalValue: number | null;
		gcLoyaltyLabel: string; gcLoyaltyPoints: number | null; gcLoyaltyCpp: number | null;
	};
	type Points = { label: string; points: number | null; cppCents: number | null; totalValue: number | null };
	type ProcessorOffer = { label: string; type: 'percent' | 'flat'; off: number | null; max: number | null };
	type LayerId = 'discount' | 'offer' | 'portal' | 'giftcard' | 'card' | 'processor' | 'points';

	type SavedEntry = {
		id: string; name: string; savedAt: string;
		merchant: string; purchasePrice: number | null; taxFlat: number | null; shippingFlat: number | null;
		discounts: Discount[]; offers: Offer[]; creditCards: CreditCard[]; portals: Portal[];
		giftCards: GiftCard[]; processorOffers: ProcessorOffer[]; pointsEarned: Points[];
	};

	const LAYER_DEFS: { id: LayerId; title: string; desc: string; color: string; icon: string }[] = [
		{ id: 'discount', title: 'Store discounts',    desc: 'Sale prices, promo codes, coupons',                color: '#e76f51', icon: '%' },
		{ id: 'offer',    title: 'Card-linked offers', desc: 'Amex/Chase Offers — applied as statement credits', color: '#f4a261', icon: '✦' },
		{ id: 'portal',   title: 'Cashback portals',   desc: 'Rakuten, TopCashback, airline portals',            color: '#2a9d8f', icon: '↗' },
		{ id: 'giftcard', title: 'Gift card boost',    desc: 'Discounted gift cards used to pay',                color: '#9b5de5', icon: '🎁' },
		{ id: 'card',      title: 'Charged to card',        desc: 'Cashback on what you actually charge',             color: '#4f46e5', icon: '▭' },
		{ id: 'processor', title: 'Payment processor',     desc: 'PayPal, Paze & similar checkout rewards',          color: '#0077cc', icon: '⬡' },
		{ id: 'points',    title: 'Points earned',         desc: 'Loyalty points & miles (future value)',            color: '#b58900', icon: '★' },
	];

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
	let pointsEarned    = $state<Points[]>([]);
	let openLayers   = $state<Record<LayerId, boolean>>({
		discount: true, offer: false, portal: false, giftcard: false, card: true, processor: false, points: false,
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

	function gcPaidPrice(g: GiftCard) {
		const fv = g.faceValue ?? 0;
		if (g.promoType === 'percent_off') return fv * (1 - (g.percentOff ?? 0) / 100);
		if (g.promoType === 'flat_off')    return Math.max(0, fv - (g.flatOff ?? 0));
		return fv;
	}
	function gcPurchaseSavings(g: GiftCard) {
		const paid           = gcPaidPrice(g);
		const credit         = g.storeCredit ?? 0;
		const chargedToCard  = Math.max(0, paid - credit);
		const cc             = chargedToCard * (g.gcCardPct / 100);
		const portal         = g.gcPortalType === 'percent'
			? paid * ((g.gcPortalValue ?? 0) / 100) : (g.gcPortalValue ?? 0);
		const loyalty        = ((g.gcLoyaltyPoints ?? 0) * (g.gcLoyaltyCpp ?? 0)) / 100;
		const promo          = g.promoType === 'bonus_gc' ? (g.promoBonus ?? 0) : 0;
		return cc + portal + loyalty + promo + credit;
	}
	function gcEffectiveCost(g: GiftCard) { return gcPaidPrice(g) - gcPurchaseSavings(g); }

	async function focusLastLabelIn(layerId: LayerId) {
		await tick();
		const layer = document.querySelector(`[data-layer-id="${layerId}"]`);
		const inputs = layer?.querySelectorAll<HTMLInputElement>('.item-label');
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

		const gcFaceTotal = giftCards.reduce((s, g) => s + (g.faceValue ?? 0), 0);
		let giftCardSavings = 0;
		for (const g of giftCards) giftCardSavings += (g.faceValue ?? 0) - gcEffectiveCost(g);

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

		let futureValue = 0;
		for (const p of pointsEarned)
			futureValue += p.totalValue != null ? p.totalValue : ((p.points ?? 0) * (p.cppCents ?? 0)) / 100;

		const totalSavings = discountSavings + offerSavings + portalSavings + giftCardSavings + creditCardSavings + processorSavings;
		const effective    = Math.max(0, base + taxAmount + shippingAmount - totalSavings);
		const savingsPct   = totalSavings > 0 ? (totalSavings / base) * 100 : 0;
		return {
			base, taxAmount, shippingAmount, effective, totalSavings, savingsPct, futureValue,
			discountedSubtotal, chargedToCC,
			bucket: { discount: discountSavings, offer: offerSavings, portal: portalSavings, giftcard: giftCardSavings, card: creditCardSavings, processor: processorSavings },
		};
	});

	// --- Animated price ---
	$effect(() => {
		const target = result?.effective ?? 0;
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
		return { discount: discounts.length, offer: offers.length, portal: portals.length, giftcard: giftCards.length, card: creditCards.length, processor: processorOffers.length, points: pointsEarned.length }[id];
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
		giftCards = [...giftCards, {
			label: '', faceValue: null, promoType: 'percent_off', percentOff: null, flatOff: null, promoBonus: null,
			storeCredit: null,
			gcCardLabel: '', gcCardPct: 1, showAdvanced: false,
			gcPortalLabel: '', gcPortalType: 'percent', gcPortalValue: null,
			gcLoyaltyLabel: '', gcLoyaltyPoints: null, gcLoyaltyCpp: null,
		}];
		await focusLastLabelIn('giftcard');
	}
	function removeGiftCard(i: number)  { giftCards = giftCards.filter((_, j) => j !== i); }
	async function addProcessorOffer()  { processorOffers = [...processorOffers, { label: '', type: 'flat', off: null, max: null }]; await focusLastLabelIn('processor'); }
	function removeProcessorOffer(i: number){ processorOffers = processorOffers.filter((_, j) => j !== i); }
	async function addPointsEarned(){ pointsEarned = [...pointsEarned, { label: '', points: null, cppCents: null, totalValue: null }]; await focusLastLabelIn('points'); }
	function removePointsEarned(i: number){ pointsEarned = pointsEarned.filter((_, j) => j !== i); }

	function addItem(id: LayerId) {
		if (id === 'discount') addDiscount();
		else if (id === 'offer') addOffer();
		else if (id === 'card') addCreditCard();
		else if (id === 'portal') addPortal();
		else if (id === 'giftcard') addGiftCard();
		else if (id === 'processor') addProcessorOffer();
		else if (id === 'points') addPointsEarned();
	}

	function resetAll() {
		merchant = ''; purchasePrice = null; taxFlat = null; shippingFlat = null;
		discounts = [{ label: '', type: 'flat', value: null }]; offers = [];
		creditCards = [{ label: '', pct: null }]; portals = []; giftCards = []; processorOffers = []; pointsEarned = [];
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
			pointsEarned: $state.snapshot(pointsEarned) as Points[],
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
		portals = entry.portals; giftCards = entry.giftCards;
		processorOffers = entry.processorOffers ?? [];
		pointsEarned = entry.pointsEarned;
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
			`Saved: ${fmt(result.totalSavings)} (${result.savingsPct.toFixed(1)}%)`,
			`You pay: ${fmt(result.effective)}`,
		].filter(Boolean);
		try { await navigator.clipboard.writeText(lines.join('\n')); } catch {}
	}

	const dateStr = new Date().toLocaleDateString('en-US', { month: 'short', day: 'numeric', year: 'numeric' });
</script>

<div class="shell">

	<!-- Top bar -->
	<div class="topbar">
		<div class="brand">
			<span class="brand-mark">S</span>
			Stack <small>cashback calculator</small>
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
						<div class="inp lg">
							<span class="prefix">$</span>
							<input type="number" min="0" step="0.01" placeholder="0.00" bind:value={purchasePrice} />
						</div>
					</div>
					<div class="field">
						<span class="field-label">Tax · Ship</span>
						<div class="tax-ship-row">
							<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="tax" bind:value={taxFlat} /></div>
							<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="ship" bind:value={shippingFlat} /></div>
						</div>
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
								<p class="hero-label">You pay (true out-of-pocket)</p>
								<p class="hero-price mono">
									<span class="hero-dollar">$</span>{priceParts.dollars}<span class="hero-cents">.{priceParts.cents}</span>
								</p>
							</div>
							<div class="hero-right">
								<span class="hero-sticker mono">{fmt(totalCost)}</span>
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
									<div class="stack-bar-seg you-pay" style="width: {Math.min(100, (result.effective / totalCost) * 100)}%;"></div>
								</div>
								<div class="stack-bar-legend">
									{#each segments as seg}
										{@const layerDef = LAYER_DEFS.find(l => l.id === seg.id)}
										<span class="legend-dot" style="--c: {seg.color}">
											{layerDef?.title} <span class="val mono">−{fmt(seg.val)}</span>
										</span>
									{/each}
									<span class="legend-dot" style="--c: var(--ink)">
										You pay <span class="val mono">{fmt(result.effective)}</span>
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
						<div class="layer-head" class:open={openLayers[layer.id]} onclick={() => openLayers[layer.id] = !openLayers[layer.id]}>
							<span class="chev">▶</span>
							<span class="dot" style="--c: {layer.color}"></span>
							<span class="layer-title">{layer.title}<small>{layer.desc}</small></span>
							<span class="layer-amount" class:has-savings={savings > 0.005}>
								{#if savings > 0.005}−{fmt(savings)}{:else if count > 0}{count} item{count > 1 ? 's' : ''}{:else}Add{/if}
							</span>
						</div>

						{#if openLayers[layer.id]}
							<div class="layer-body">

								{#if layer.id === 'discount'}
									{#each discounts as d, i}
										{@const saving = d.type === 'percent' ? (purchasePrice ?? 0) * ((d.value ?? 0) / 100) : (d.value ?? 0)}
										<div class="item-row">
											<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">%</div>
											<div class="item-main">
												<input class="item-label" type="text" placeholder="Promo code / sale" bind:value={d.label} />
												<span class="item-meta">{d.type === 'percent' ? 'Percent off' : 'Flat amount'}</span>
											</div>
											<div class="seg">
												<button class:active={d.type === 'percent'} onclick={() => discounts[i].type = 'percent'}>%</button>
												<button class:active={d.type === 'flat'} onclick={() => discounts[i].type = 'flat'}>$</button>
											</div>
											<div class="item-value-input">
												{#if d.type === 'percent'}
													<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="10" bind:value={d.value} /><span class="suffix">%</span></div>
												{:else}
													<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0.00" bind:value={d.value} /></div>
												{/if}
											</div>
											<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
											<button class="item-remove" onclick={() => removeDiscount(i)}>×</button>
										</div>
									{/each}

								{:else if layer.id === 'offer'}
									{#each offers as o, i}
										{@const saving = o.type === 'percent' ? Math.min(o.max != null ? o.max : Infinity, (purchasePrice ?? 0) * ((o.off ?? 0) / 100)) : (o.off ?? 0)}
										<div class="item-row" class:offer-percent={o.type === 'percent'}>
											<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">✦</div>
											<div class="item-main">
												<input class="item-label" type="text" placeholder="Amex / Chase Offer" bind:value={o.label} />
												<span class="item-meta">
													{o.type === 'flat' ? 'Statement credit' : `${o.off ?? 0}% back${o.max != null ? ` · capped ${fmt(o.max)}` : ''}`}
												</span>
											</div>
											<div class="seg">
												<button class:active={o.type === 'flat'} onclick={() => { offers[i].type = 'flat'; offers[i].max = null; }}>$</button>
												<button class:active={o.type === 'percent'} onclick={() => offers[i].type = 'percent'}>%</button>
											</div>
											<div class="item-value-input">
												{#if o.type === 'flat'}
													<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="10" bind:value={o.off} /></div>
												{:else}
													<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="5" bind:value={o.off} /><span class="suffix">%</span></div>
												{/if}
											</div>
											{#if o.type === 'percent'}
												<div class="item-value-input">
													<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="cap" bind:value={o.max} /></div>
												</div>
											{/if}
											<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
											<button class="item-remove" onclick={() => removeOffer(i)}>×</button>
										</div>
									{/each}

								{:else if layer.id === 'portal'}
									{#each portals as p, i}
										{@const saving = p.valueType === 'percent' ? (purchasePrice ?? 0) * ((p.value ?? 0) / 100) : (p.value ?? 0)}
										<div class="item-row">
											<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">↗</div>
											<div class="item-main">
												<input class="item-label" type="text" placeholder="Portal (e.g. Rakuten)" bind:value={p.label} />
												<span class="item-meta">{p.valueType === 'percent' ? 'Percent back' : 'Flat amount'}</span>
											</div>
											<div class="seg">
												<button class:active={p.valueType === 'percent'} onclick={() => portals[i].valueType = 'percent'}>%</button>
												<button class:active={p.valueType === 'flat'} onclick={() => portals[i].valueType = 'flat'}>$</button>
											</div>
											<div class="item-value-input">
												{#if p.valueType === 'percent'}
													<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="3" bind:value={p.value} /><span class="suffix">%</span></div>
												{:else}
													<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="5" bind:value={p.value} /></div>
												{/if}
											</div>
											<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
											<button class="item-remove" onclick={() => removePortal(i)}>×</button>
										</div>
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
												<button class="item-remove" onclick={() => removeGiftCard(i)}>×</button>
											</div>
											{#if gc.showAdvanced}
												<div class="gc-detail">
													<div class="gc-detail-row">
														<label>Face value</label>
														<div class="inp" style="width:100px;"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="100" bind:value={gc.faceValue} /></div>
													</div>
													<div class="gc-detail-row">
														<label>Purchase discount</label>
														<div class="seg">
															<button class:active={gc.promoType === 'percent_off'} onclick={() => giftCards[i].promoType = 'percent_off'}>% off</button>
															<button class:active={gc.promoType === 'flat_off'}    onclick={() => giftCards[i].promoType = 'flat_off'}>$ off</button>
															<button class:active={gc.promoType === 'bonus_gc'}    onclick={() => giftCards[i].promoType = 'bonus_gc'}>Bonus GC</button>
														</div>
													</div>
													{#if gc.promoType === 'percent_off'}
														<div class="gc-detail-row">
															<label>Discount %</label>
															<div class="inp" style="width:100px;"><input type="number" min="0" max="100" step="0.1" placeholder="10" bind:value={gc.percentOff} /><span class="suffix">%</span></div>
														</div>
													{:else if gc.promoType === 'flat_off'}
														<div class="gc-detail-row">
															<label>Discount $</label>
															<div class="inp" style="width:100px;"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="10" bind:value={gc.flatOff} /></div>
														</div>
													{:else}
														<div class="gc-detail-row">
															<label>Bonus GC value</label>
															<div class="inp" style="width:100px;"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="10" bind:value={gc.promoBonus} /></div>
														</div>
													{/if}
													<div class="gc-detail-row">
														<label>Store credit used</label>
														<div style="display:flex;align-items:center;gap:6px;">
															<div class="inp" style="width:100px;"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0" bind:value={gc.storeCredit} /></div>
															<span class="gc-hint">reward GC / platform credit</span>
														</div>
													</div>
													<div class="gc-detail-row">
														<label>Portal cashback</label>
														<div style="display:flex;gap:4px;align-items:center;">
															<div class="seg">
																<button class:active={gc.gcPortalType === 'percent'} onclick={() => giftCards[i].gcPortalType = 'percent'}>%</button>
																<button class:active={gc.gcPortalType === 'flat'} onclick={() => giftCards[i].gcPortalType = 'flat'}>$</button>
															</div>
															{#if gc.gcPortalType === 'percent'}
																<div class="inp" style="width:80px;"><input type="number" min="0" max="100" step="0.1" placeholder="3.5" bind:value={gc.gcPortalValue} /><span class="suffix">%</span></div>
															{:else}
																<div class="inp" style="width:80px;"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="0" bind:value={gc.gcPortalValue} /></div>
															{/if}
															<input class="gc-card-name" placeholder="Portal name" bind:value={gc.gcPortalLabel} />
														</div>
													</div>
													<div class="gc-detail-row">
														<label>Card used to buy GC</label>
														<div style="display:flex;gap:6px;align-items:center;">
															<input class="gc-card-name" placeholder="Card name" bind:value={gc.gcCardLabel} />
															<div class="inp" style="width:80px;"><input type="number" min="0" max="100" step="0.01" placeholder="1" bind:value={gc.gcCardPct} /><span class="suffix">%</span></div>
														</div>
													</div>
													<div class="gc-detail-row">
														<label>Loyalty earned</label>
														<div style="display:flex;gap:6px;align-items:center;">
															<input class="gc-card-name" placeholder="Program" bind:value={gc.gcLoyaltyLabel} />
															<div class="inp" style="width:70px;"><input type="number" min="0" step="1" placeholder="pts" bind:value={gc.gcLoyaltyPoints} /></div>
															<div class="inp" style="width:72px;"><input type="number" min="0" step="0.01" placeholder="¢/pt" bind:value={gc.gcLoyaltyCpp} /><span class="suffix">¢</span></div>
														</div>
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
										{@const saving = (result?.chargedToCC ?? 0) * ((cc.pct ?? 0) / 100)}
										<div class="item-row">
											<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">▭</div>
											<div class="item-main">
												<input class="item-label" type="text" placeholder="Card name (e.g. Citi Double Cash)" bind:value={cc.label} />
												<span class="item-meta">Cashback on <strong>{fmt(result?.chargedToCC ?? 0)}</strong> charged</span>
											</div>
											<div class="item-value-input">
												<div class="inp"><input type="number" min="0" max="100" step="0.01" placeholder="2" bind:value={cc.pct} /><span class="suffix">%</span></div>
											</div>
											<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
											<button class="item-remove" onclick={() => removeCreditCard(i)}>×</button>
										</div>
									{/each}

								{:else if layer.id === 'processor'}
									{#each processorOffers as po, i}
										{@const saving = po.type === 'percent' ? Math.min(po.max != null ? po.max : Infinity, (result?.chargedToCC ?? 0) * ((po.off ?? 0) / 100)) : (po.off ?? 0)}
										<div class="item-row" class:offer-percent={po.type === 'percent'}>
											<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">⬡</div>
											<div class="item-main">
												<input class="item-label" type="text" placeholder="e.g. PayPal Honey, Paze" bind:value={po.label} />
												<span class="item-meta">
													{po.type === 'flat' ? 'Statement credit / cashback' : `${po.off ?? 0}% back on ${fmt(result?.chargedToCC ?? 0)} charged${po.max != null ? ` · capped ${fmt(po.max)}` : ''}`}
												</span>
											</div>
											<div class="seg">
												<button class:active={po.type === 'flat'} onclick={() => { processorOffers[i].type = 'flat'; processorOffers[i].max = null; }}>$</button>
												<button class:active={po.type === 'percent'} onclick={() => processorOffers[i].type = 'percent'}>%</button>
											</div>
											<div class="item-value-input">
												{#if po.type === 'flat'}
													<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="10" bind:value={po.off} /></div>
												{:else}
													<div class="inp"><input type="number" min="0" max="100" step="0.1" placeholder="5" bind:value={po.off} /><span class="suffix">%</span></div>
												{/if}
											</div>
											{#if po.type === 'percent'}
												<div class="item-value-input">
													<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="cap" bind:value={po.max} /></div>
												</div>
											{/if}
											<span class="item-saves" class:zero={saving <= 0}>−{fmt(saving)}</span>
											<button class="item-remove" onclick={() => removeProcessorOffer(i)}>×</button>
										</div>
									{/each}

								{:else if layer.id === 'points'}
									{#each pointsEarned as pts, i}
										{@const value = pts.totalValue != null ? pts.totalValue : ((pts.points ?? 0) * (pts.cppCents ?? 0)) / 100}
										{@const derivedCpp = pts.totalValue != null && (pts.points ?? 0) > 0 ? (pts.totalValue * 100 / pts.points!).toFixed(2) : null}
										<div class="item-row">
											<div class="item-icon" style="background: {layer.color}22; color: {layer.color}">★</div>
											<div class="item-main">
												<input class="item-label" type="text" placeholder="Loyalty program" bind:value={pts.label} />
												<span class="item-meta">
													{pts.points ?? 0} pts
													{#if derivedCpp}× {derivedCpp}¢{:else}× {pts.cppCents ?? 0}¢{/if}
													≈ <strong>{fmt(value)}</strong> future
												</span>
											</div>
											<div class="item-value-input">
												<div class="inp"><input type="number" min="0" step="1" placeholder="pts" bind:value={pts.points} /></div>
											</div>
											{#if pts.totalValue == null}
												<div class="item-value-input">
													<div class="inp"><input type="number" min="0" step="0.01" placeholder="¢/pt" bind:value={pts.cppCents} /><span class="suffix">¢</span></div>
												</div>
											{/if}
											<div class="item-value-input">
												<div class="inp"><span class="prefix">$</span><input type="number" min="0" step="0.01" placeholder="value" bind:value={pts.totalValue} /></div>
											</div>
											<button class="item-remove" onclick={() => removePointsEarned(i)}>×</button>
										</div>
									{/each}
								{/if}

								<div class="add-section">
									<button class="add-chip custom" onclick={() => addItem(layer.id)}>
										<span class="chip-plus">+</span> Add
									</button>
								</div>

							</div>
						{/if}
					</div>
				{/each}
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

					{#if result.shippingAmount > 0}
						<div class="receipt-row"><span class="label">Shipping</span><span class="amt">+{fmt(result.shippingAmount)}</span></div>
					{/if}
					{#if result.taxAmount > 0}
						<div class="receipt-row"><span class="label">Tax</span><span class="amt">+{fmt(result.taxAmount)}</span></div>
					{/if}

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

					{#each giftCards as gc}
						{@const v = (gc.faceValue ?? 0) - gcEffectiveCost(gc)}
						{#if v > 0}
							<div class="receipt-row save">
								<span class="label">{gc.label || 'Gift card'}</span>
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

					<div class="receipt-total">
						<div class="label">You pay</div>
						<div class="amt mono">{fmt(result.effective)}</div>
					</div>

					{#if result.futureValue > 0}
						<div class="receipt-future">
							<span>+ Future value (points)</span>
							<strong>{fmt(result.futureValue)}</strong>
						</div>
					{/if}

					<div class="receipt-foot">
						<button onclick={() => window.print()}>Print</button>
						<button onclick={copyReceipt}>Copy</button>
					</div>
				</div>

				<div class="wallet-hint">
					<strong>Stack order:</strong> Discounts → Portals → Offers → then card cashback on the remaining balance. Gift card savings reduce the amount charged to your card.
				</div>
			{:else}
				<div class="receipt">
					<div class="receipt-empty">
						<div class="emo">🧾</div>
						Enter a subtotal to print<br />your stacked receipt.
					</div>
				</div>
				<div class="wallet-hint">
					<strong>Tip ·</strong> Discounts apply first, then portals and card offers, and finally cashback on your card charge. Stack them all to find your true cost.
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
		grid-template-columns: 1fr 120px 180px;
		gap: 10px;
	}
	@media (max-width: 600px) {
		.purchase-row { grid-template-columns: 1fr 1fr; }
		.purchase-row > :first-child { grid-column: 1 / -1; }
	}
	.tax-ship-row { display: flex; gap: 6px; }

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
	.inp.lg input { padding: 10px 4px; font-size: 16px; font-weight: 500; }
	.inp.lg .prefix { font-size: 15px; }
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
	.chev { color: var(--ink-4); font-size: 11px; transition: transform .15s; flex-shrink: 0; }
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

	.item-saves {
		font-family: 'JetBrains Mono', monospace;
		font-size: 12px; color: var(--green); font-weight: 600;
		min-width: 56px; text-align: right; font-variant-numeric: tabular-nums;
		flex-shrink: 0;
	}
	.item-saves.zero { color: var(--ink-4); }

	.item-remove {
		width: 22px; height: 22px; border-radius: 5px;
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

	/* Add section */
	.add-section { margin-top: 8px; }
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
	.add-chip.custom { border-style: dashed; }
	.chip-plus { font-size: 14px; line-height: 1; }

	/* Gift card detail */
	.gc-detail {
		background: var(--bg);
		border: 1px solid var(--line);
		border-radius: 9px;
		margin-top: 8px;
		padding: 10px 12px;
		font-size: 12px;
		color: var(--ink-2);
	}
	.gc-detail-row {
		display: flex; justify-content: space-between; align-items: center;
		gap: 10px; padding: 5px 0;
	}
	.gc-detail-row + .gc-detail-row { border-top: 1px dashed var(--line); }
	.gc-detail-row label { font-size: 11px; color: var(--ink-3); font-weight: 500; }
	.gc-card-name {
		border: 0; background: transparent; outline: none;
		font-size: 12px; width: 100px; color: var(--ink-2);
		font-family: inherit;
	}
	.gc-hint { font-size: 10.5px; color: var(--ink-4); white-space: nowrap; }
	.gc-advantage {
		margin-top: 6px; padding-top: 8px;
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
	.receipt-future {
		background: rgba(255,255,255,.04);
		border-radius: 8px; padding: 8px 10px; margin-top: 12px;
		font-size: 11.5px; color: #c4c3be;
		display: flex; justify-content: space-between; align-items: baseline;
	}
	.receipt-future :global(strong) { color: #a5b4fc; font-family: 'JetBrains Mono', monospace; font-weight: 600; }
	.receipt-empty { text-align: center; padding: 30px 12px; color: #555; font-size: 13px; }
	.receipt-empty .emo { font-size: 26px; margin-bottom: 8px; opacity: .6; }
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
