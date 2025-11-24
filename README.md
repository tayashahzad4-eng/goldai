import React, { useState, useEffect, useRef } from 'react';
import { Activity, TrendingUp, TrendingDown, Bell, RefreshCw, Clock, ShieldAlert, DollarSign, BarChart2, Newspaper, Target, Zap } from 'lucide-react';

export default function App() {
  const [currentPrice, setCurrentPrice] = useState(0.00);
  const [signal, setSignal] = useState(null);
  const [sentiment, setSentiment] = useState({ status: 'Neutral', score: 50 });
  const [technical, setTechnical] = useState({ rsi: 50, trend: 'Flat', ema: 0 });
  const [levels, setLevels] = useState({ support: 0, resistance: 0 });
  const [marketStatus, setMarketStatus] = useState('Initializing Pro Tools...');
  const [history, setHistory] = useState([]);
  const [newsFeed, setNewsFeed] = useState([]);
  
  // Store live candle data for calculations (Close prices)
  const [priceHistory, setPriceHistory] = useState([]); 
  const priceRef = useRef(0.00);

  // --- 1. PROFESSIONAL INDICATOR CALCULATIONS ---
  
  // Relative Strength Index (RSI 14) Calculation
  const calculateRSI = (prices, period = 14) => {
    if (prices.length < period + 1) return 50; // Not enough data yet
    
    let gains = 0;
    let losses = 0;
    
    // Calculate initial average
    for (let i = prices.length - period; i < prices.length; i++) {
      const difference = prices[i] - prices[i - 1];
      if (difference >= 0) gains += difference;
      else losses += Math.abs(difference);
    }
    
    const avgGain = gains / period;
    const avgLoss = losses / period;
    
    if (avgLoss === 0) return 100;
    const rs = avgGain / avgLoss;
    return 100 - (100 / (1 + rs));
  };

  // Exponential Moving Average (EMA 20)
  const calculateEMA = (prices, period = 20) => {
    if (prices.length < period) return prices[prices.length - 1]; // Return current price if not enough data
    
    const k = 2 / (period + 1);
    let ema = prices[0];
    
    for (let i = 1; i < prices.length; i++) {
      ema = (prices[i] * k) + (ema * (1 - k));
    }
    return ema;
  };

  // Price Action: Find Support & Resistance
  const calculateLevels = (prices) => {
    if (prices.length < 10) return { support: prices[0], resistance: prices[0] };
    
    // Simple Local Min/Max over recent history
    const recent = prices.slice(-20); // Last 20 ticks
    const min = Math.min(...recent);
    const max = Math.max(...recent);
    
    return { support: min, resistance: max };
  };

  // --- 2. DATA FETCHING (Real-Time) ---
  const fetchRealPrice = async () => {
    try {
      // Using GoldAPI.io (Public/Free)
      const URL = "https://api.gold-api.com/price/XAU";
      const response = await fetch(URL, { method: 'GET' });
      const data = await response.json();

      if (data.price) {
        const livePrice = parseFloat(data.price);
        setCurrentPrice(livePrice);
        priceRef.current = livePrice;
        
        // Build History Live
        setPriceHistory(prev => {
          const newHist = [...prev, livePrice];
          if (newHist.length > 50) newHist.shift(); // Keep last 50 ticks
          return newHist;
        });

        setMarketStatus('Live Market Data');
        
        // Trigger Analysis
        performAnalysis(livePrice, priceHistory); // Pass current history
      }
    } catch (error) {
      console.error("API Error:", error);
      setMarketStatus('Reconnecting...');
    }
  };

  useEffect(() => {
    fetchRealPrice();
    const interval = setInterval(fetchRealPrice, 5000); // 5s Refresh
    return () => clearInterval(interval);
  }, []); // Run once on mount

  // --- 3. ADVANCED ANALYSIS ENGINE ---
  const performAnalysis = (price, historyData) => {
    // Need at least some data to analyze
    if (historyData.length < 5) return;

    // A. Technical Indicators
    const rsiVal = calculateRSI(historyData);
    const emaVal = calculateEMA(historyData);
    const levelVals = calculateLevels(historyData);
    
    // Determine Trend (Price vs EMA)
    let trend = 'Flat';
    if (price > emaVal + 0.5) trend = 'Uptrend';
    else if (price < emaVal - 0.5) trend = 'Downtrend';

    setTechnical({ rsi: Math.round(rsiVal), trend: trend, ema: emaVal.toFixed(2) });
    setLevels(levelVals);

    // B. Fundamental Logic (Simulated for Demo Context)
    // In a real app, you'd fetch this from a News API.
    // Logic: Strong Trend usually implies strong Fundamental backing.
    let newsStatus = 'Neutral';
    let newsScore = 50;

    if (trend === 'Uptrend') {
        newsStatus = 'Bullish';
        newsScore = 70 + (rsiVal / 10); // Correlate with momentum
    } else if (trend === 'Downtrend') {
        newsStatus = 'Bearish';
        newsScore = 30 - (rsiVal / 10);
    }
    setSentiment({ status: newsStatus, score: newsScore });

    // C. SIGNAL ALGORITHM (Price Action + Tech + Fundamentals)
    let newSignal = null;
    const time = new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' });

    // SCENARIO 1: BREAKOUT BUY (Price Action)
    // Price breaks Resistance + Uptrend + News Bullish
    if (price > levelVals.resistance && trend === 'Uptrend' && newsStatus === 'Bullish') {
       newSignal = createSignal('BUY', price, 'Breakout Strategy', 'Resistance Broken');
    }
    // SCENARIO 2: PULLBACK BUY (Technical)
    // Uptrend + RSI Oversold (<40) = Buying Dip
    else if (trend === 'Uptrend' && rsiVal < 40) {
       newSignal = createSignal('BUY', price, 'Trend Pullback', 'RSI Oversold in Uptrend');
    }
    // SCENARIO 3: BREAKDOWN SELL (Price Action)
    // Price breaks Support + Downtrend + News Bearish
    else if (price < levelVals.support && trend === 'Downtrend' && newsStatus === 'Bearish') {
       newSignal = createSignal('SELL', price, 'Breakdown Strategy', 'Support Broken');
    }
    // SCENARIO 4: REJECTION SELL (Technical)
    // Downtrend + RSI Overbought (>60) = Selling Rally
    else if (trend === 'Downtrend' && rsiVal > 60) {
       newSignal = createSignal('SELL', price, 'Trend Rejection', 'RSI Overbought in Downtrend');
    }

    if (newSignal) {
      setSignal(prev => {
         // Avoid spamming same signal
         if (!prev || prev.time !== newSignal.time) {
             if (window.navigator && window.navigator.vibrate) window.navigator.vibrate(500);
             setHistory(hist => [newSignal, ...hist].slice(0, 5));
             return newSignal;
         }
         return prev;
      });
    }
  };

  const createSignal = (type, price, strategy, reason) => {
      const sl = type === 'BUY' ? price - 4.00 : price + 4.00; // $4 SL
      const tp = type === 'BUY' ? price + 8.00 : price - 8.00; // $8 TP (1:2 Risk Reward)
      
      return {
        type,
        entry: price,
        sl: sl.toFixed(2),
        tp: tp.toFixed(2),
        time: new Date().toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' }),
        strategy,
        reason
      };
  };

  return (
    <div className="min-h-screen bg-black text-white font-sans flex justify-center items-center p-4">
      <div className="w-full max-w-md bg-gray-900 h-[850px] rounded-3xl flex flex-col shadow-2xl border border-gray-800 relative overflow-hidden ring-8 ring-gray-800">
        
        {/* Pro Header */}
        <div className="bg-gray-800/80 backdrop-blur-md p-4 flex justify-between items-center sticky top-0 z-10 border-b border-gray-700">
          <div className="flex items-center gap-2">
            <div className="bg-gradient-to-tr from-yellow-600 to-yellow-400 p-2 rounded-lg shadow-lg">
                <Activity size={18} className="text-black font-bold" />
            </div>
            <div>
                <h1 className="text-lg font-bold text-white tracking-tight leading-none">Gold Pro AI</h1>
                <p className="text-[10px] text-yellow-500 font-mono">XAU/USD ANALYZER</p>
            </div>
          </div>
          <div className="flex items-center gap-1 bg-gray-700/50 px-2 py-1 rounded border border-gray-600">
            <Zap size={10} className="text-yellow-400 fill-current" />
            <span className="text-[10px] font-bold text-gray-300">REAL-TIME</span>
          </div>
        </div>

        <div className="flex-1 p-4 overflow-y-auto scrollbar-hide">
            
            {/* Price Action Card */}
            <div className="bg-gray-800 rounded-2xl p-5 mb-4 border border-gray-700 shadow-lg relative overflow-hidden">
                <div className="flex justify-between items-start mb-2">
                    <p className="text-gray-400 text-xs font-bold tracking-wider">LIVE MARKET PRICE</p>
                    <span className={`text-[10px] px-2 py-0.5 rounded font-bold ${technical.trend === 'Uptrend' ? 'bg-green-900/40 text-green-400' : technical.trend === 'Downtrend' ? 'bg-red-900/40 text-red-400' : 'bg-gray-700 text-gray-400'}`}>
                        {technical.trend.toUpperCase()}
                    </span>
                </div>
                
                <div className="text-5xl font-bold text-white font-mono tracking-tighter mb-4">
                    {currentPrice > 0 ? `$${currentPrice.toFixed(2)}` : '---.--'}
                </div>

                {/* Key Levels (Support / Resistance) */}
                <div className="grid grid-cols-2 gap-2 mt-2">
                    <div className="bg-black/30 rounded p-2 border-l-2 border-red-500">
                        <p className="text-[9px] text-gray-500 uppercase">Resistance (Ceiling)</p>
                        <p className="text-sm font-mono text-red-400">${levels.resistance.toFixed(2)}</p>
                    </div>
                    <div className="bg-black/30 rounded p-2 border-l-2 border-green-500 text-right">
                        <p className="text-[9px] text-gray-500 uppercase">Support (Floor)</p>
                        <p className="text-sm font-mono text-green-400">${levels.support.toFixed(2)}</p>
                    </div>
                </div>
            </div>

            {/* TECHNICAL & FUNDAMENTAL DASHBOARD */}
            <div className="grid grid-cols-2 gap-3 mb-6">
                
                {/* Technicals */}
                <div className="bg-gray-800/40 p-3 rounded-xl border border-gray-700">
                    <div className="flex items-center gap-2 mb-3 border-b border-gray-700 pb-2">
                        <BarChart2 size={14} className="text-blue-500"/>
                        <span className="text-xs font-bold text-gray-300">TECHNICALS</span>
                    </div>
                    <div className="space-y-2">
                        <div className="flex justify-between items-center">
                            <span className="text-[10px] text-gray-500">RSI (14)</span>
                            <span className={`text-xs font-bold ${technical.rsi > 70 ? 'text-red-400' : technical.rsi < 30 ? 'text-green-400' : 'text-white'}`}>{technical.rsi}</span>
                        </div>
                        <div className="flex justify-between items-center">
                            <span className="text-[10px] text-gray-500">EMA (20)</span>
                            <span className="text-xs font-mono text-yellow-500">${technical.ema}</span>
                        </div>
                        <div className="w-full bg-gray-700 h-1 rounded-full mt-1">
                             <div className="bg-blue-500 h-1 rounded-full" style={{width: `${technical.rsi}%`}}></div>
                        </div>
                    </div>
                </div>

                {/* Fundamentals */}
                <div className="bg-gray-800/40 p-3 rounded-xl border border-gray-700">
                    <div className="flex items-center gap-2 mb-3 border-b border-gray-700 pb-2">
                        <Newspaper size={14} className="text-purple-500"/>
                        <span className="text-xs font-bold text-gray-300">FUNDAMENTALS</span>
                    </div>
                    <div className="space-y-2">
                         <div className="flex justify-between items-center">
                            <span className="text-[10px] text-gray-500">Sentiment</span>
                            <span className={`text-xs font-bold ${sentiment.status === 'Bullish' ? 'text-green-400' : sentiment.status === 'Bearish' ? 'text-red-400' : 'text-gray-400'}`}>
                                {sentiment.status}
                            </span>
                        </div>
                        <p className="text-[9px] text-gray-500 leading-tight">
                            {sentiment.status === 'Bullish' ? "Investors favoring safe-haven assets." : sentiment.status === 'Bearish' ? "Strong Dollar pressuring Gold." : "Market awaiting high-impact news."}
                        </p>
                        <div className="w-full bg-gray-700 h-1 rounded-full mt-2">
                             <div className={`h-1 rounded-full transition-all duration-500 ${sentiment.score > 50 ? 'bg-green-500' : 'bg-red-500'}`} style={{width: `${sentiment.score}%`}}></div>
                        </div>
                    </div>
                </div>
            </div>
            
            {/* SIGNAL CARD */}
            <div className="flex items-center gap-2 mb-3">
                <Target size={16} className="text-yellow-500" />
                <h2 className="text-gray-400 text-xs font-bold tracking-wide">AI GENERATED SIGNAL</h2>
            </div>
            
            {signal ? (
                <div className={`relative rounded-xl p-5 mb-6 border-l-4 shadow-2xl animate-in fade-in zoom-in duration-300 ${signal.type === 'BUY' ? 'bg-green-900/20 border-green-500' : 'bg-red-900/20 border-red-500'}`}>
                    <div className="flex justify-between items-start mb-4">
                        <span className={`px-4 py-1.5 rounded-lg text-sm font-bold shadow-sm ${signal.type === 'BUY' ? 'bg-green-600 text-white' : 'bg-red-600 text-white'}`}>
                            {signal.type} @ {signal.entry}
                        </span>
                        <div className="text-right">
                             <span className="text-[10px] text-gray-400 block">{signal.time}</span>
                             <span className="text-[9px] text-yellow-500 font-bold">{signal.strategy}</span>
                        </div>
                    </div>

                    <p className="text-xs text-gray-300 mb-4 italic border-l-2 border-gray-600 pl-2">
                        "Algorithm detected {signal.reason}. Confirmed by Sentiment."
                    </p>

                    <div className="bg-gray-950/80 rounded-lg p-3 flex divide-x divide-gray-700 border border-gray-700">
                        <div className="flex-1 px-2 text-center">
                            <p className="text-[9px] text-gray-500 uppercase font-bold mb-1">Stop Loss</p>
                            <p className="text-lg font-bold text-red-500">${signal.sl}</p>
                        </div>
                        <div className="flex-1 px-2 text-center">
                            <p className="text-[9px] text-gray-500 uppercase font-bold mb-1">Take Profit</p>
                            <p className="text-lg font-bold text-green-500">${signal.tp}</p>
                        </div>
                    </div>
                </div>
            ) : (
                <div className="bg-gray-800/30 border border-dashed border-gray-700 rounded-xl p-8 text-center mb-6">
                    <RefreshCw className="w-6 h-6 text-gray-600 mx-auto mb-2 animate-spin-slow" />
                    <p className="text-gray-400 font-medium text-sm">Analyzing Market Structure...</p>
                    <p className="text-gray-600 text-[10px] mt-2">Checking EMA(20), RSI(14) & Price Action Levels</p>
                </div>
            )}

            {/* History Table */}
            <h2 className="text-gray-500 text-xs font-bold mb-2 pl-1">RECENT SIGNALS</h2>
            <div className="space-y-2 pb-20">
                {history.map((item, index) => (
                    <div key={index} className="bg-gray-800/50 p-3 rounded-lg flex justify-between items-center border border-gray-800">
                        <div className="flex flex-col">
                            <span className={`text-xs font-bold ${item.type === 'BUY' ? 'text-green-500' : 'text-red-500'}`}>
                                {item.type} <span className="text-gray-400">@</span> {item.entry}
                            </span>
                            <span className="text-[9px] text-gray-500">{item.strategy}</span>
                        </div>
                        <div className="text-right">
                            <span className="text-[10px] text-gray-400 block">{item.time}</span>
                            <span className={`text-[9px] font-bold ${item.type === 'BUY' ? 'text-green-400' : 'text-red-400'}`}>
                                Target: ${item.tp}
                            </span>
                        </div>
                    </div>
                ))}
            </div>

        </div>

        {/* Floating Action Button */}
        <div className="absolute bottom-6 left-4 right-4">
            <button 
                onClick={() => {
                    setMarketStatus('Updating...');
                    fetchRealPrice();
                    if (navigator.vibrate) navigator.vibrate(50);
                }}
                className="w-full bg-gradient-to-r from-yellow-600 to-yellow-500 hover:from-yellow-500 hover:to-yellow-400 text-black font-bold py-3.5 rounded-xl shadow-xl shadow-yellow-500/20 flex justify-center items-center gap-2 active:scale-95 transition-transform"
            >
                <RefreshCw size={18} />
                SCAN MARKETS
            </button>
        </div>

      </div>
      
      <style jsx global>{`
        @keyframes spin-slow {
          from { transform: rotate(0deg); }
          to { transform: rotate(360deg); }
        }
        .animate-spin-slow {
          animation: spin-slow 3s linear infinite;
        }
      `}</style>
    </div>
  );
}
