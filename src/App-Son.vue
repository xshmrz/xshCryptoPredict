<template>
	<div id="loader">
		<img src="/src/assets/img/loading-icon.png" alt="icon" class="loading-icon">
	</div>
	<div class="appHeader">
		<div class="left"></div>
		<div class="pageTitle">Crypto App (Binance)</div>
		<div class="right"></div>
	</div>
	<!-- Coin Listesi -->
	<div id="appCapsule">
		<ul v-if="!selectedSymbol" class="listview link-listview inset mt-2">
			<li v-for="coin in coins" :key="coin.symbol">
				<a class="fw-bold" href="javascript:void(0)" @click="selectCoin(coin.symbol)">
					{{ coin.symbol }}
					<div class="d-flex justify-content-between align-items-center">
						<div class="text-muted">{{ coin.lastPrice }} ₺</div>
						<div :class="parseFloat(coin.priceChangePercent) >= 0 ? 'text-success' : 'text-danger'" class="text-end" style="min-width: 75px !important;">
							{{ parseFloat(coin.priceChangePercent).toFixed(2) }}%
						</div>
					</div>
				</a>
			</li>
		</ul>
		<div v-if="selectedSymbol && predictionTable.length" class="section mt-2">
			<div class="alert" :class="avgProfit > 0 ? 'alert-success' : 'alert-danger'" role="alert">
				{{ avgProfit > 0 ? 'Bu coin alınabilir.' : 'Bu coin önerilmez.' }}
			</div>
		</div>
		<div v-if="selectedSymbol && predictionTable.length" class="section mt-2">
			<div class="card">
				<div class="table-responsive">
					<table class="table table-striped">
						<thead>
						<tr>
							<th>Tarih</th>
							<th class="text-end">Balance</th>
						</tr>
						</thead>
						<tbody>
						<tr v-for="(row, index) in predictionTable" :key="row.date">
							<td>
								<div class="fw-bold">{{ row.date }}</div>
								<div class="fw-bold" :class="row.percent > 0 ? 'text-success' : 'text-danger'">{{ row.percent > 0 ? 'RECOMMENDED' : 'DO NOT BUY' }}</div>
							</td>
							<td class="text-end">
								<div>{{ row.open.toFixed(8) }}</div>
								<div>{{ row.close.toFixed(8) }}</div>
							</td>
						</tr>
						</tbody>
					</table>
				</div>
			</div>
		</div>
		
	</div>
</template>
<script setup>
import {ref, computed, onMounted} from 'vue';
const coins           = ref([]);
const selectedSymbol  = ref('');
const investment      = ref(1000);
const predictionTable = ref([]);
const maxProfitIndex  = ref(0);
const minProfitIndex  = ref(0);
const totalProfit     = computed(() => predictionTable.value.reduce((sum, row) => sum + (investment.value * row.percent / 100), 0)
);
onMounted(async () => {
	const res    = await fetch('https://api.binance.com/api/v3/ticker/24hr');
	const data   = await res.json();
	coins.value  = data.filter(item => item.symbol.endsWith('USDT'));
	const loader = document.getElementById('loader');
	if (loader) {
		loader.setAttribute('style', 'pointer-events: none; opacity: 0; transition: 0.2s ease-in-out;');
		setTimeout(() => {
			loader.setAttribute('style', 'display: none;');
		}, 1000);
	}
});
async function selectCoin(symbol) {
	selectedSymbol.value = symbol;
	const closes         = await fetchKlines(symbol);
	predict(closes);
}
async function fetchKlines(symbol) {
	const res = await fetch(`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=1d&limit=30`);
	const raw = await res.json();
	return raw.map(k => parseFloat(k[4]));
}
function predict(closes) {
	const n     = closes.length;
	const x     = closes.map((_, i) => i);
	const xMean = x.reduce((a, b) => a + b, 0) / n;
	const yMean = closes.reduce((a, b) => a + b, 0) / n;
	let num     = 0, den = 0;
	for (let i = 0; i < n; i++) {
		num += (x[i] - xMean) * (closes[i] - yMean);
		den += (x[i] - xMean) ** 2;
	}
	const slope     = num / den;
	const intercept = yMean - slope * xMean;
	const vary      = 0.015; // %1.5 dalgalanma
	const preds     = [];
	for (let i = 1; i <= 5; i++) {
		const base    = slope * (n + i - 1) + intercept;
		const open    = base * (1 + (Math.random() * 2 - 1) * vary);
		const close   = base * (1 + (Math.random() * 2 - 1) * vary);
		const percent = ((close - open) / open) * 100;
		const date    = new Date();
		date.setDate(date.getDate() + i);
		preds.push({
			           date: date.toLocaleDateString('tr-TR'),
			           open,
			           close,
			           percent
		           });
	}
	predictionTable.value = preds;
	let max               = 0, min = 0;
	preds.forEach((p, i) => {
		if (p.percent > preds[max].percent) max = i;
		if (p.percent < preds[min].percent) min = i;
	});
	maxProfitIndex.value = max;
	minProfitIndex.value = min;
}
</script>
<style>
</style>
