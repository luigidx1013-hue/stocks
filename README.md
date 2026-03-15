\import React, { useMemo, useState } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Alert, AlertDescription } from "@/components/ui/alert";
import { LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, ResponsiveContainer, Legend } from "recharts";
import { TrendingUp, AlertTriangle, BarChart3 } from "lucide-react";

/**
 * Stock Projection App
 *
 * What this does:
 * - Lets a user enter a ticker and projection horizon
 * - Uses a scoring model based on historical performance, earnings growth,
 *   analyst target upside, valuation mean reversion, news sentiment, and volatility
 * - Produces bear / base / bull paths
 *
 * To use with real data:
 * 1. Replace getStockData() with fetch calls to your backend or APIs
 * 2. Recommended APIs:
 *    - Price history: Alpha Vantage, Polygon, Finnhub, Twelve Data, or your backend
 *    - Fundamentals / earnings estimates: Finnhub, Financial Modeling Prep, Polygon, or your backend
 *    - News sentiment: Finnhub sentiment, Alpha Vantage NEWS_SENTIMENT, or your own NLP service
 * 3. Deploy to GitHub + Vercel / Netlify
 *
 * Important:
 * This is an educational model, not investment advice.
 */

const MOCK_DATA = {
  AAPL: {
    currentPrice: 215,
    historical5yCagr: 0.17,
    revenueCagr: 0.08,
    earningsGrowthTarget: 0.11,
    analystTargetPrice: 240,
    currentPE: 31,
    fairPE: 28,
    beta: 1.2,
    newsSentiment: 0.22,
    qualityScore: 0.86,
  },
  MSFT: {
    currentPrice: 425,
    historical5yCagr: 0.19,
    revenueCagr: 0.12,
    earningsGrowthTarget: 0.14,
    analystTargetPrice: 470,
    currentPE: 36,
    fairPE: 32,
    beta: 0.95,
    newsSentiment: 0.19,
    qualityScore: 0.91,
  },
  NVDA: {
    currentPrice: 905,
    historical5yCagr: 0.52,
    revenueCagr: 0.39,
    earningsGrowthTarget: 0.28,
    analystTargetPrice: 980,
    currentPE: 58,
    fairPE: 44,
    beta: 1.7,
    newsSentiment: 0.15,
    qualityScore: 0.88,
  },
  VOO: {
    currentPrice: 540,
    historical5yCagr: 0.13,
    revenueCagr: 0.08,
    earningsGrowthTarget: 0.09,
    analystTargetPrice: 575,
    currentPE: 25,
    fairPE: 22,
    beta: 1.0,
    newsSentiment: 0.05,
    qualityScore: 0.93,
  },
};

function clamp(value, min, max) {
  return Math.max(min, Math.min(max, value));
}

function annualizedReturnFromTarget(currentPrice, targetPrice, years = 1) {
  if (!currentPrice || !targetPrice || currentPrice <= 0 || targetPrice <= 0) return 0;
  return Math.pow(targetPrice / currentPrice, 1 / years) - 1;
}

function valuationAdjustment(currentPE, fairPE, horizonYears) {
  if (!currentPE || !fairPE || currentPE <= 0 || fairPE <= 0) return 0;
  const totalReversion = fairPE / currentPE - 1;
  return totalReversion / Math.max(horizonYears, 1);
}

function betaPenalty(beta) {
  // Small penalty for very high beta to avoid overstating high-flyers.
  if (!beta) return 0;
  return Math.max(0, beta - 1) * 0.015;
}

function sentimentAdjustment(sentiment) {
  // sentiment expected range roughly -1 to +1
  return clamp(sentiment, -1, 1) * 0.03;
}

function qualityAdjustment(qualityScore) {
  // quality score expected range 0..1
  if (qualityScore == null) return 0;
  return (qualityScore - 0.5) * 0.04;
}

function projectionAlgorithm(stock, years) {
  const hist = stock.historical5yCagr ?? 0;
  const rev = stock.revenueCagr ?? 0;
  const earn = stock.earningsGrowthTarget ?? 0;
  const analystAnnualized = annualizedReturnFromTarget(stock.currentPrice, stock.analystTargetPrice, 1);
  const valuation = valuationAdjustment(stock.currentPE, stock.fairPE, years);
  const sentiment = sentimentAdjustment(stock.newsSentiment ?? 0);
  const quality = qualityAdjustment(stock.qualityScore);
  const riskPenalty = betaPenalty(stock.beta ?? 1);

  // Weighted blend. You can tune these weights over time using backtesting.
  let baseReturn =
    hist * 0.28 +
    rev * 0.12 +
    earn * 0.28 +
    analystAnnualized * 0.14 +
    valuation * 0.10 +
    sentiment * 0.04 +
    quality * 0.04 -
    riskPenalty;

  // Prevent unrealistically high or low expected annual returns.
  baseReturn = clamp(baseReturn, -0.25, 0.30);

  const uncertainty = clamp(0.08 + (stock.beta ?? 1) * 0.03, 0.08, 0.18);
  const bearReturn = clamp(baseReturn - uncertainty, -0.40, 0.20);
  const bullReturn = clamp(baseReturn + uncertainty, -0.10, 0.45);

  return {
    baseReturn,
    bearReturn,
    bullReturn,
    components: {
      historical: hist,
      revenue: rev,
      earnings: earn,
      analystAnnualized,
      valuation,
      sentiment,
      quality,
      riskPenalty,
    },
  };
}

function buildProjectionSeries(currentPrice, years, annualReturns) {
  const rows = [];
  let bear = currentPrice;
  let base = currentPrice;
  let bull = currentPrice;

  rows.push({ year: 0, bear, base, bull });

  for (let i = 1; i <= years; i += 1) {
    bear *= 1 + annualReturns.bearReturn;
    base *= 1 + annualReturns.baseReturn;
    bull *= 1 + annualReturns.bullReturn;
    rows.push({
      year: i,
      bear: Number(bear.toFixed(2)),
      base: Number(base.toFixed(2)),
      bull: Number(bull.toFixed(2)),
    });
  }

  return rows;
}

async function getStockData(ticker) {
  const upper = ticker.toUpperCase().trim();

  // Demo mode using mock data.
  if (MOCK_DATA[upper]) {
    await new Promise((resolve) => setTimeout(resolve, 300));
    return { ticker: upper, ...MOCK_DATA[upper] };
  }

  // Replace this with your real backend/API fetch.
  // Example shape your backend should return:
  // {
  //   ticker: "TSLA",
  //   currentPrice: 180,
  //   historical5yCagr: 0.14,
  //   revenueCagr: 0.22,
  //   earningsGrowthTarget: 0.18,
  //   analystTargetPrice: 205,
  //   currentPE: 55,
  //   fairPE: 40,
  //   beta: 1.8,
  //   newsSentiment: -0.08,
  //   qualityScore: 0.71
  // }
  throw new Error(
    `No demo data found for ${upper}. Add a backend/API integration in getStockData() or try AAPL, MSFT, NVDA, or VOO.`
  );
}

function Metric({ label, value }) {
  return (
    <div className="rounded-2xl border p-4 shadow-sm bg-white">
      <div className="text-sm text-slate-500">{label}</div>
      <div className="text-xl font-semibold mt-1">{value}</div>
    </div>
  );
}

export default function StockProjectionApp() {
  const [ticker, setTicker] = useState("AAPL");
  const [years, setYears] = useState(5);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState("");
  const [stock, setStock] = useState(null);
  const [results, setResults] = useState(null);

  const handleAnalyze = async () => {
    try {
      setLoading(true);
      setError("");
      const data = await getStockData(ticker);
      const annualReturns = projectionAlgorithm(data, Number(years));
      const series = buildProjectionSeries(data.currentPrice, Number(years), annualReturns);
      setStock(data);
      setResults({ annualReturns, series });
    } catch (err) {
      setError(err.message || "Unable to analyze this ticker.");
      setStock(null);
      setResults(null);
    } finally {
      setLoading(false);
    }
  };

  const summary = useMemo(() => {
    if (!stock || !results) return null;
    const last = results.series[results.series.length - 1];
    return {
      baseAnnual: `${(results.annualReturns.baseReturn * 100).toFixed(2)}%`,
      bearAnnual: `${(results.annualReturns.bearReturn * 100).toFixed(2)}%`,
      bullAnnual: `${(results.annualReturns.bullReturn * 100).toFixed(2)}%`,
      currentPrice: `$${stock.currentPrice.toFixed(2)}`,
      baseEnd: `$${last.base.toLocaleString()}`,
      bearEnd: `$${last.bear.toLocaleString()}`,
      bullEnd: `$${last.bull.toLocaleString()}`,
    };
  }, [stock, results]);

  return (
    <div className="min-h-screen bg-slate-50 p-6 md:p-10">
      <div className="max-w-6xl mx-auto space-y-6">
        <div className="flex items-center gap-3">
          <div className="rounded-2xl bg-white p-3 shadow-sm border">
            <TrendingUp className="h-6 w-6" />
          </div>
          <div>
            <h1 className="text-3xl font-bold tracking-tight">Stock Projection Analyzer</h1>
            <p className="text-slate-600 mt-1">
              Enter a ticker to project bear, base, and bull outcomes using historical performance,
              earnings growth, analyst targets, valuation, sentiment, and risk.
            </p>
          </div>
        </div>

        <Alert>
          <AlertTriangle className="h-4 w-4" />
          <AlertDescription>
            This model is for education and experimentation. It is not financial advice and should not be used as the sole basis for investment decisions.
          </AlertDescription>
        </Alert>

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-6">
          <Card className="rounded-2xl shadow-sm lg:col-span-1">
            <CardHeader>
              <CardTitle>Inputs</CardTitle>
            </CardHeader>
            <CardContent className="space-y-4">
              <div className="space-y-2">
                <Label htmlFor="ticker">Ticker</Label>
                <Input
                  id="ticker"
                  value={ticker}
                  onChange={(e) => setTicker(e.target.value.toUpperCase())}
                  placeholder="AAPL"
                />
              </div>

              <div className="space-y-2">
                <Label htmlFor="years">Projection Years</Label>
                <Input
                  id="years"
                  type="number"
                  min={1}
                  max={30}
                  value={years}
                  onChange={(e) => setYears(e.target.value)}
                />
              </div>

              <Button className="w-full rounded-2xl" onClick={handleAnalyze} disabled={loading}>
                {loading ? "Analyzing..." : "Analyze Stock"}
              </Button>

              <div className="text-sm text-slate-500 pt-2">
                Demo tickers built in: <span className="font-medium">AAPL, MSFT, NVDA, VOO</span>
              </div>
            </CardContent>
          </Card>

          <div className="lg:col-span-2 space-y-6">
            {error && (
              <Alert className="border-red-200 bg-red-50">
                <AlertDescription>{error}</AlertDescription>
              </Alert>
            )}

            {summary && (
              <>
                <div className="grid grid-cols-2 md:grid-cols-3 gap-4">
                  <Metric label="Current Price" value={summary.currentPrice} />
                  <Metric label="Base Annual Return" value={summary.baseAnnual} />
                  <Metric label="Bear Annual Return" value={summary.bearAnnual} />
                  <Metric label="Bull Annual Return" value={summary.bullAnnual} />
                  <Metric label={`Base ${years}-Year Value`} value={summary.baseEnd} />
                  <Metric label={`Bull ${years}-Year Value`} value={summary.bullEnd} />
                </div>

                <Card className="rounded-2xl shadow-sm">
                  <CardHeader>
                    <CardTitle className="flex items-center gap-2">
                      <BarChart3 className="h-5 w-5" /> Projection Paths
                    </CardTitle>
                  </CardHeader>
                  <CardContent>
                    <div className="h-[360px] w-full">
                      <ResponsiveContainer width="100%" height="100%">
                        <LineChart data={results.series}>
                          <CartesianGrid strokeDasharray="3 3" />
                          <XAxis dataKey="year" />
                          <YAxis />
                          <Tooltip formatter={(value) => `$${Number(value).toLocaleString()}`} />
                          <Legend />
                          <Line type="monotone" dataKey="bear" strokeWidth={2} dot={false} />
                          <Line type="monotone" dataKey="base" strokeWidth={2} dot={false} />
                          <Line type="monotone" dataKey="bull" strokeWidth={2} dot={false} />
                        </LineChart>
                      </ResponsiveContainer>
                    </div>
                  </CardContent>
                </Card>

                <Card className="rounded-2xl shadow-sm">
                  <CardHeader>
                    <CardTitle>Model Inputs and Contributions</CardTitle>
                  </CardHeader>
                  <CardContent>
                    <div className="grid grid-cols-1 md:grid-cols-2 gap-4 text-sm">
                      <div className="rounded-2xl border p-4">
                        <div className="font-medium mb-2">Underlying Data</div>
                        <div>Ticker: {stock.ticker}</div>
                        <div>Current Price: ${stock.currentPrice.toFixed(2)}</div>
                        <div>Historical 5Y CAGR: {(stock.historical5yCagr * 100).toFixed(2)}%</div>
                        <div>Revenue CAGR: {(stock.revenueCagr * 100).toFixed(2)}%</div>
                        <div>Earnings Growth Target: {(stock.earningsGrowthTarget * 100).toFixed(2)}%</div>
                        <div>Analyst Target Price: ${stock.analystTargetPrice.toFixed(2)}</div>
                        <div>Current P/E: {stock.currentPE.toFixed(2)}</div>
                        <div>Fair P/E: {stock.fairPE.toFixed(2)}</div>
                        <div>Beta: {stock.beta.toFixed(2)}</div>
                        <div>News Sentiment: {(stock.newsSentiment * 100).toFixed(2)}%</div>
                        <div>Quality Score: {(stock.qualityScore * 100).toFixed(0)} / 100</div>
                      </div>

                      <div className="rounded-2xl border p-4">
                        <div className="font-medium mb-2">Annual Return Components</div>
                        <div>Historical contribution input: {(results.annualReturns.components.historical * 100).toFixed(2)}%</div>
                        <div>Revenue contribution input: {(results.annualReturns.components.revenue * 100).toFixed(2)}%</div>
                        <div>Earnings contribution input: {(results.annualReturns.components.earnings * 100).toFixed(2)}%</div>
                        <div>Analyst annualized input: {(results.annualReturns.components.analystAnnualized * 100).toFixed(2)}%</div>
                        <div>Valuation adjustment: {(results.annualReturns.components.valuation * 100).toFixed(2)}%</div>
                        <div>Sentiment adjustment: {(results.annualReturns.components.sentiment * 100).toFixed(2)}%</div>
                        <div>Quality adjustment: {(results.annualReturns.components.quality * 100).toFixed(2)}%</div>
                        <div>Risk penalty: -{(results.annualReturns.components.riskPenalty * 100).toFixed(2)}%</div>
                      </div>
                    </div>
                  </CardContent>
                </Card>
              </>
            )}
          </div>
        </div>
      </div>
    </div>
  );
}
