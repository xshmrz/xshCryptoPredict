<template>
	<div id="loader">
		<img src="/src/assets/img/loading-icon.png" alt="icon" class="loading-icon"/>
	</div>
	<div class="appHeader">
		<div class="left"></div>
		<div v-if="!analysisDone" class="pageTitle">{{ t('appTitle') }}</div>
		<div v-else class="pageTitle">{{ selectedSymbol }}</div>
		<div class="right"></div>
	</div>
	<div id="appCapsule">
		<!-- Coin Listesi -->
		<ul v-if="!analysisDone" class="listview link-listview inset mt-2 mb-2">
			<li v-for="coin in coinList" :key="coin.symbol">
				<a class="fw-bold" @click="selectAndAnalyze(coin.symbol)">
					<div class="me-auto">{{ coin.symbol }}</div>
					<div class="fw-bold me-2 text-end">{{ formatPrice(coin.price) }}</div>
					<div style="min-width: 60px;" class="text-end" :class="coin.percent > 0 ? 'text-success' : (coin.percent < 0 ? 'text-danger' : '')">
						{{ (coin.percent >= 0 ? '+' : '') + coin.percent.toFixed(2).replace('.', ',') + '%' }}
					</div>
				</a>
			</li>
		</ul>
		<!-- Genel Değerlendirme (Toplam Öngörülen % bazlı) -->
		<div v-if="analysisDone" class="section mt-2">
			<div class="text-center alert" :class="totalForecastPercent >= 0 ? 'alert-success' : 'alert-danger'">
				<strong>
					{{ totalForecastPercent >= 0 ? t('evaluationBuy') : t('evaluationAvoid') }}
				</strong>
			</div>
		</div>
		<!-- Volatilite, MACD ve Kar Hedefi -->
		<ul v-if="analysisDone" class="listview link-listview not-arrow inset mt-2">
			<li><a class="fw-bold"><div>{{ t('volatility') }}</div><div class="fw-bold text-end">{{ volatility.toFixed(2) }}</div></a></li>
			<li><a class="fw-bold"><div>{{ macdLabel }}</div><div class="fw-bold text-end">{{ macdHistogram.toFixed(4).replace('.', ',') }}</div></a></li>
			<li><a class="fw-bold"><div>{{ profitTargetLabel }}</div><div class="fw-bold text-end">{{ profitTarget }}%</div></a></li>
			<li><a class="fw-bold"><div>Toplam Öngörülen %</div><div class="fw-bold text-end">{{ totalForecastPercent.toFixed(2) }}%</div></a></li>
		</ul>
		<!-- 5 Günlük Strateji Listesi -->
		<ul v-if="analysisDone && strategyList.length" class="listview link-listview not-arrow inset mt-2 mb-2">
			<li v-for="(s, i) in strategyList" :key="i">
				<a>
					<div class="fw-bold">{{ s.date }}</div>
					<div class="fw-bold small text-end">
						<div :class="{'text-success': s.suggestion === 'BUY','text-danger' : s.suggestion === 'AVOID','text-secondary': s.suggestion === 'HOLD'}">
							{{ t('suggestions.' + s.suggestion) }}
							({{ s.percent.toFixed(2).replace('.', ',') }}%)
						</div>
						<div>
							{{ s.open.toFixed(4).replace('.', ',') }} /
							{{ s.close.toFixed(4).replace('.', ',') }}
						</div>
					</div>
				</a>
			</li>
		</ul>
	</div>
	<!-- Geri Butonu -->
	<div class="appBottomMenu" v-if="analysisDone">
		<a @click="resetAnalysis" href="javascript:void(0)" class="item">
			<div class="col">
				<button class="btn btn-primary w-100">{{ t('back') }}</button>
			</div>
		</a>
	</div>
</template>
<script setup>
import {ref, onMounted, computed} from 'vue';
// Dil sistemi
const locale               = ref('tr');
const translations         = {
	tr: {
		back           : 'Geri',
		appTitle       : 'Kripto Uygulaması',
		volatility     : 'Volatilite (%)',
		evaluationBuy  : 'Coin ALIM için uygun görünüyor.',
		evaluationAvoid: 'Coin önerilmiyor.',
		totalProfit    : 'Toplam Tahmini Kâr/Zarar',
		suggestions    : {BUY: 'AL', AVOID: 'ÖNERİLMEZ', HOLD: 'BEKLE'}
	},
	en: {
		back           : 'Back',
		appTitle       : 'Crypto App',
		volatility     : 'Volatility (%)',
		evaluationBuy  : 'This coin is potentially a BUY.',
		evaluationAvoid: 'This coin is not recommended.',
		totalProfit    : 'Total Estimated P&L',
		suggestions    : {BUY: 'BUY', AVOID: 'AVOID', HOLD: 'HOLD'}
	}
};
const t                    = key => {
	const keys = key.split('.');
	let res    = translations[locale.value];
	for (const k of keys) res = res?.[k];
	return res || key;
};
// Temel veriler
const coinList             = ref([]);
const selectedSymbol       = ref('');
const investment           = ref(100);
const volatility           = ref(0);
const macdHistogram        = ref(0);
const profitTarget         = ref(25);
const strategyList         = ref([]);
const analysisDone         = ref(false);
const totalForecastPercent = ref(0);
const macdLabel            = 'MACD Hist. (Son)';
const profitTargetLabel    = 'Kar Hedefi (%)';
// Göstergeler (EMA, RSI, MACD)
function calculateEMA(prices, period) {
	const k            = 2 / (period + 1);
	const emaArr       = [];
	let ema            = prices.slice(0, period).reduce((a, b) => a + b, 0) / period;
	emaArr[period - 1] = ema;
	for (let i = period; i < prices.length; i++) {
		ema       = prices[i] * k + ema * (1 - k);
		emaArr[i] = ema;
	}
	return emaArr;
}
function calculateRSI(prices, period = 14) {
	let gains = 0, losses = 0;
	for (let i = 1; i <= period; i++) {
		const d = prices[i] - prices[i - 1];
		if (d >= 0) gains += d;
		else losses -= d;
	}
	let avgG     = gains / period, avgL = losses / period, rs = avgG / avgL;
	const rsiArr = [100 - 100 / (1 + rs)];
	for (let i = period + 1; i < prices.length; i++) {
		const d = prices[i] - prices[i - 1], g = d > 0 ? d : 0, l = d < 0 ? -d : 0;
		avgG    = (avgG * (period - 1) + g) / period;
		avgL    = (avgL * (period - 1) + l) / period;
		rs      = avgG / avgL;
		rsiArr.push(100 - 100 / (1 + rs));
	}
	return rsiArr;
}
function calculateMACD(prices, shortP = 12, longP = 26, signalP = 9) {
	const fast     = calculateEMA(prices, shortP);
	const slow     = calculateEMA(prices, longP);
	const macdLine = fast.map((v, i) => v - (slow[i] || 0));
	const signal   = calculateEMA(macdLine.slice(longP - 1), signalP);
	return macdLine.slice(longP - 1).map((v, i) => v - (signal[i] || 0));
}
// Veri çekme + volatilite
const fetchKlines         = async symbol => {
	const res  = await fetch(
			`https://api.binance.com/api/v3/klines?symbol=${symbol}&interval=1d&limit=365`
	);
	const data = await res.json();
	return data.map(k => parseFloat(k[4]));
};
const calculateVolatility = closes => {
	const mean = closes.reduce((a, b) => a + b, 0) / closes.length;
	const varr = closes.reduce((s, v) => s + Math.pow(v - mean, 2), 0) / closes.length;
	return (Math.sqrt(varr) / mean) * 100;
};
// Tahmin + günlük öneri, toplam göstergesi
const predict             = closes => {
	macdHistogram.value = calculateMACD(closes).at(-1) || 0;
	const ema7          = calculateEMA(closes, 7);
	const ema30         = calculateEMA(closes, 30);
	const rsi           = calculateRSI(closes);
	const changes       = closes
			.slice(-30)
			.map((c, i, a) => (i === 0 ? 0 : ((c - a[i - 1]) / a[i - 1]) * 100))
			.slice(1);
	const meanCh        = changes.reduce((a, b) => a + b, 0) / changes.length;
	const stdCh         = Math.sqrt(
			changes.reduce((s, v) => s + Math.pow(v - meanCh, 2), 0) / changes.length
	);
	// Günlük tahminler ve toplam
	const temp          = [];
	for (let i = 1; i <= 5; i++) {
		const factor = (i - 3) / 2;
		temp.push(meanCh + factor * stdCh);
	}
	const sumPct               = temp.reduce((a, b) => a + b, 0);
	totalForecastPercent.value = sumPct;
	const preds                = [];
	let price                  = closes.at(-1);
	const base                 = new Date();
	base.setDate(base.getDate() - 1);
	temp.forEach((pct, idx) => {
		const open  = price;
		const close = open * (1 + pct / 100);
		let score   = 0;
		if (ema7.at(-1) > ema30.at(-1)) score += 40;
		if (rsi.at(-1) < 70) score += 30;
		if (pct >= 2) score += 30;
		if (macdHistogram.value > 0) score += 10;
		// Günlük öneri
		let suggestion = 'HOLD';
		if (score >= 70 && pct > 0) suggestion = 'BUY';
		else if (pct < 0 || rsi.at(-1) > 70 || ema7.at(-1) < ema30.at(-1))
			suggestion = 'AVOID';
		// gg/aa/yyyy formatlı tarih
		const d = new Date(base);
		d.setDate(base.getDate() + idx + 1);
		const day           = String(d.getDate()).padStart(2, '0');
		const month         = String(d.getMonth() + 1).padStart(2, '0');
		const year          = d.getFullYear();
		const formattedDate = `${day}/${month}/${year}`;
		preds.push({date: formattedDate, open, close, percent: pct, suggestion, confidence: score});
		price = close;
	});
	strategyList.value = preds;
};
// Toplam kar/zarar (tüm günlere göre)
const totalProfit         = computed(() =>
		                                     strategyList.value.reduce(
				                                     (sum, p) => sum + (p.close - p.open) * (investment.value / p.open),
				                                     0
		                                     )
);
// Analiz akışı
const analyzeSelected     = async () => {
	if (!selectedSymbol.value) return;
	const closes = await fetchKlines(selectedSymbol.value);
	predict(closes.slice(0, -1));
	volatility.value   = calculateVolatility(closes.slice(0, -1));
	analysisDone.value = true;
};
const selectAndAnalyze    = async sym => {
	selectedSymbol.value = sym;
	await analyzeSelected();
};
const resetAnalysis       = () => {
	analysisDone.value   = false;
	strategyList.value   = [];
	selectedSymbol.value = '';
};
// Formatlama
const formatPrice         = (v, d = 6) => {
	const n = parseFloat(v);
	return n.toLocaleString(locale.value === 'tr' ? 'tr-TR' : 'en-US', {
		minimumFractionDigits: n < 1 ? d : 2,
		maximumFractionDigits: n < 1 ? d : 8
	});
};
// Başlangıçta coin listesi çekme
onMounted(async () => {
	const res      = await fetch('https://api.binance.com/api/v3/ticker/24hr');
	const data     = await res.json();
	coinList.value = data
			.filter(t => t.symbol.endsWith('TRY') && !/UP|DOWN/.test(t.symbol))
			.map(t => ({
				symbol : t.symbol,
				price  : parseFloat(t.lastPrice),
				percent: parseFloat(t.priceChangePercent)
			}));
	const ld       = document.getElementById('loader');
	if (ld) {
		ld.style.pointerEvents = 'none';
		ld.style.opacity       = '0';
		setTimeout(() => (ld.style.display = 'none'), 1000);
	}
});
</script>
