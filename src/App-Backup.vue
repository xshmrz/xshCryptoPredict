<template>
	<div class="container py-4">
		<h2 class="text-white text-center mb-3">📈 Coin Simülasyon Paneli</h2>
		<!-- Coin ve yatırım seçimi -->
		<div class="row g-3 mb-4">
			<div class="col-md-12">
				<select v-model="selectedSymbol" class="form-select" @change="analyzeSelected">
					<option disabled value="">Bir coin seçin</option>
					<option v-for="coin in coinList" :key="coin" :value="coin">{{ coin }}</option>
				</select>
			</div>
			<div class="col-md-12">
				<input type="number"
				       min="1"
				       v-model.number="investment"
				       class="form-control"
				       placeholder="Yatırım Tutarı (USD)"
				/>
			</div>
		</div>
		<div v-if="analysisDone">
			<!-- Özet tablo -->
			<div class="table-responsive mx-auto summary-table">
				<table class="table table-dark table-bordered text-white">
					<tbody>
					<tr>
						<th>Volatilite (%)</th>
						<td class="text-end">{{ volatility.toFixed(4) }}</td>
					</tr>
					<tr>
						<th>Ortalama Kar (%)</th>
						<td class="text-end">{{ avgProfit.toFixed(4) }}</td>
					</tr>
					<tr>
						<th>En Yüksek Kar Günü (%)</th>
						<td class="text-end">{{ maxProfit.toFixed(4) }}</td>
					</tr>
					<tr>
						<th>Tavsiye Alış</th>
						<td class="text-end">${{ avgOpen.toFixed(8) }}</td>
					</tr>
					<tr>
						<th>Tavsiye Satış</th>
						<td class="text-end">${{ avgClose.toFixed(8) }}</td>
					</tr>
					</tbody>
				</table>
			</div>
			<!-- Genel değerlendirme -->
			<div class="alert" :class="avgProfit > 0 ? 'alert-success' : 'alert-danger'">
				💡 Genel Değerlendirme:
				<strong>{{ avgProfit > 0 ? 'Bu coin alınabilir.' : 'Bu coin önerilmez.' }}</strong>
			</div>
			<!-- 5 günlük tahmin tablosu -->
			<div class="table-responsive">
				<table class="table table-bordered text-white prediction-table mx-auto">
					<thead>
					<tr>
						<th>Tarih</th>
						<th class="text-end">Tavsiye Alış</th>
						<th class="text-end">Tavsiye Satış</th>
						<th class="text-end">Kar (%)</th>
						<th class="text-end">Simülasyon</th>
						<th class="text-center">Öneri</th>
					</tr>
					</thead>
					<tbody>
					<tr
							v-for="(row, index) in predictionTable"
							:key="row.date"
							:class="{
                'bg-success bg-opacity-25': index === maxProfitIndex,
                'bg-danger bg-opacity-25': index === minProfitIndex
              }"
					>
						<td>{{ row.date }}</td>
						<td class="text-end">${{ row.open.toFixed(8) }}</td>
						<td class="text-end">${{ row.close.toFixed(8) }}</td>
						<td class="text-end">{{ row.percent.toFixed(4) }}%</td>
						<td class="text-end">${{ (investment * row.percent / 100).toFixed(2) }}</td>
						<td class="text-center fw-bold">
                <span :class="row.percent > 0 ? 'text-success' : 'text-danger'">
                  {{ row.percent > 0 ? 'Alınmalı' : 'Alınmamalı' }}
                </span>
						</td>
					</tr>
					</tbody>
				</table>
			</div>
			<!-- Toplam kar/zarar -->
			<div class="alert alert-info text-end mx-auto prediction-table">
				📊 Toplam Tahmini Kar/Zarar:
				<strong>${{ totalProfit.toFixed(2) }}</strong>
			</div>
			<!-- Grafik -->
			<div class="chart-container mx-auto">
				<Bar :data="chartData" :options="chartOptions"/>
			</div>
		</div>
	</div>
</template>
<script setup>
import {ref, onMounted, computed} from 'vue';
import {Bar}                      from 'vue-chartjs';
import {
	Chart as ChartJS,
	Title, Tooltip, Legend, BarElement,
	CategoryScale, LinearScale
}                                 from 'chart.js';
ChartJS.register(Title, Tooltip, Legend, BarElement, CategoryScale, LinearScale);
const coinList        = ref([]);
const selectedSymbol  = ref('');
const investment      = ref(1000);
const volatility      = ref(0);
const avgProfit       = ref(0);
const maxProfit       = ref(0);
const avgOpen         = ref(0);
const avgClose        = ref(0);
const predictionTable = ref([]);
const analysisDone    = ref(false);
const maxProfitIndex  = ref(0);
const minProfitIndex  = ref(0);
const fetchKlines = async (symbol) => {
	const res = await fetch(`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=1d&limit=30`);
	const raw = await res.json();
	return raw.map(k => parseFloat(k[4]));
};
const calculateVolatility = (closes) => {
	const mean = closes.reduce((a, b) => a + b, 0) / closes.length;
	const std  = Math.sqrt(closes.reduce((acc, val) => acc + (val - mean) ** 2, 0) / closes.length);
	return (std / mean) * 100;
};
const predict = (closes) => {
	const n     = closes.length;
	const x     = closes.map((_, i) => i);
	const xMean = x.reduce((a, b) => a + b, 0) / n;
	const yMean = closes.reduce((a, b) => a + b, 0) / n;
	let num = 0, den = 0;
	for (let i = 0; i < n; i++) {
		num += (x[i] - xMean) * (closes[i] - yMean);
		den += (x[i] - xMean) ** 2;
	}
	const slope      = num / den;
	const intercept  = yMean - slope * xMean;
	const vol        = calculateVolatility(closes);
	volatility.value = vol;
	const vary  = vol / 100 * 0.5;
	const preds = [];
	for (let i = 1; i <= 5; i++) {
		const base    = slope * (n + i - 1) + intercept;
		const open    = base * (1 + (Math.random() * 2 - 1) * vary);
		const close   = base * (1 + (Math.random() * 2 - 1) * vary);
		const percent = ((close - open) / open) * 100;
		const date    = new Date();
		date.setDate(date.getDate() + i);
		preds.push({date: date.toLocaleDateString('tr-TR'), open, close, percent});
	}
	predictionTable.value = preds;
	avgProfit.value       = preds.reduce((sum, p) => sum + p.percent, 0) / preds.length;
	maxProfit.value       = Math.max(...preds.map(p => p.percent));
	avgOpen.value         = preds.reduce((sum, p) => sum + p.open, 0) / preds.length;
	avgClose.value        = preds.reduce((sum, p) => sum + p.close, 0) / preds.length;
	let max = 0, min = 0;
	preds.forEach((p, i) => {
		if (p.percent > preds[max].percent) max = i;
		if (p.percent < preds[min].percent) min = i;
	});
	maxProfitIndex.value = max;
	minProfitIndex.value = min;
};
const totalProfit = computed(() =>
		                             predictionTable.value.reduce((sum, row) => sum + (investment.value * row.percent / 100), 0)
);
const chartData = computed(() => {
	return {
		labels  : predictionTable.value.map(p => p.date),
		datasets: [{
			label          : 'Günlük Kar/Zarar (USD)',
			data           : predictionTable.value.map(p => (investment.value * p.percent / 100).toFixed(2)),
			backgroundColor: predictionTable.value.map((p, i) => {
				if (i === maxProfitIndex.value) return 'rgba(0, 255, 0, 0.6)';
				if (i === minProfitIndex.value) return 'rgba(255, 0, 0, 0.6)';
				return 'rgba(54, 162, 235, 0.6)';
			})
		}]
	};
});
const chartOptions = {
	responsive         : true,
	maintainAspectRatio: false,
	scales             : {
		y: {
			beginAtZero: true,
			ticks      : {color: '#FFFFFF'},
			title      : {display: true, text: 'Kar/Zarar (USD)', color: '#FFFFFF'}
		},
		x: {
			ticks: {color: '#FFFFFF'},
			title: {display: true, text: 'Tarih', color: '#FFFFFF'}
		}
	},
	plugins            : {
		legend : {labels: {color: '#fff'}},
		tooltip: {
			callbacks: {
				label: ctx => ` $${ctx.raw}`
			}
		}
	}
};
const analyzeSelected = async () => {
	if (!selectedSymbol.value) return;
	const closes = await fetchKlines(selectedSymbol.value);
	predict(closes);
	analysisDone.value = true;
};
onMounted(async () => {
	const res      = await fetch('https://api.binance.com/api/v3/exchangeInfo');
	const data     = await res.json();
	coinList.value = data.symbols
	                     .filter(s => s.symbol.endsWith('USDT') && !s.symbol.includes('UP') && !s.symbol.includes('DOWN'))
	                     .map(s => s.symbol);
});
</script>
<style>
body {
	background-color : #0C1421;
	font-family      : Consolas, monospace;
	font-size        : 12px;
	}
.table th, .table td {
	font-size : 12px;
	}
.summary-table,
.prediction-table,
.chart-container {
	max-width : 600px;
	width     : 100%;
	}
.text-end {
	text-align : right !important;
	}
.chart-container {
	height  : 300px;
	padding : 1rem 0;
	}
</style>
