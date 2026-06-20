import streamlit as st
import pandas as pd
import yfinance as yf
import plotly.graph_objects as go
import urllib.request
import json
from datetime import datetime
from streamlit_autorefresh import st_autorefresh

# ページの基本設定
st.set_page_config(page_title="経済モニター ULTRA v2", layout="wide")

# 30秒ごとに自動更新
st_autorefresh(interval=30000, key="datarefresh")

# スタイル改善
st.markdown("""
    <style>
    .main { background-color: #0e1117; }
    div[data-testid="stMetricValue"] { font-size: 24px; color: #00ffc8; }
    </style>
    """, unsafe_allow_html=True)

st.title("📊 個人専用 経済ダッシュボード ULTRA")
st.caption(f"最終更新 (JST): {datetime.now().strftime('%Y-%m-%d %H:%M:%S')} (30秒ごとに自動更新)")

# ==========================================
# 1. 外国為替レート (リアルタイム取得)
# ==========================================
st.subheader("💱 為替レート (USD基準)")
try:
    forex_url = "https://open.er-api.com/v6/latest/USD"
    with urllib.request.urlopen(forex_url) as response:
        forex_data = json.loads(response.read().decode())
    
    jpy_rate = forex_data["rates"]["JPY"]
    eur_rate = forex_data["rates"]["JPY"] / forex_data["rates"]["EUR"]
    gbp_rate = forex_data["rates"]["JPY"] / forex_data["rates"]["GBP"]
    
    col1, col2, col3, col4 = st.columns(4)
    col1.metric("USD / JPY", f"¥{jpy_rate:.2f}")
    col2.metric("EUR / JPY", f"¥{eur_rate:.2f}")
    col3.metric("GBP / JPY", f"¥{gbp_rate:.2f}")
    col4.metric("EUR / USD", f"${forex_data['rates']['EUR']:.4f}")
except Exception as e:
    st.error("為替データの取得に失敗しました。")

st.divider()

# ==========================================
# 2. 主要株式市場 (yfinanceで動的取得)
# ==========================================
st.subheader("📉 主要マーケット指数")

def get_market_data(ticker):
    try:
        data = yf.Ticker(ticker).history(period="2d")
        if len(data) >= 2:
            current = data['Close'].iloc[-1]
            prev = data['Close'].iloc[-2]
            diff = current - prev
            pct = (diff / prev) * 100
            return current, pct
        return None, None
    except:
        return None, None

indices = {
    "日経平均 (^N225)": "^N225",
    "S&P 500 (^GSPC)": "^GSPC",
    "米国10年債利回り (^TNX)": "^TNX",
    "NASDAQ (^IXIC)": "^IXIC"
}

idx_cols = st.columns(len(indices))
for i, (name, ticker) in enumerate(indices.items()):
    price, change = get_market_data(ticker)
    if price:
        symbol = "¥" if "^N225" in ticker else ""
        unit = "%" if "^TNX" in ticker else ""
        idx_cols[i].metric(name, f"{symbol}{price:,.2f}{unit}", f"{change:+.2f}%")
    else:
        idx_cols[i].metric(name, "取得エラー")

st.divider()

# ==========================================
# 3. GOLD & Bitcoin リアルタイムチャート
# ==========================================
st.subheader("👑 GOLD & Bitcoin マルチチャート")

col_sel1, col_sel2, col_sel3 = st.columns([2, 2, 4])
with col_sel1:
    asset = st.selectbox("分析銘柄:", ["GOLD (金先物)", "Bitcoin (BTC-USD)", "Ethereum (ETH-USD)"])
with col_sel2:
    timeframe = st.selectbox("時間足:", ["5分足", "15分足", "1時間足", "1日足"])

ticker_map = {"GOLD (金先物)": "GC=F", "Bitcoin (BTC-USD)": "BTC-USD", "Ethereum (ETH-USD)": "ETH-USD"}
interval_map = {"5分足": "5m", "15分足": "15m", "1時間足": "1h", "1日足": "1d"}
period_map = {"5分足": "1d", "15分足": "5d", "1時間足": "1mo", "1日足": "1y"}

with st.spinner("データを読み込み中..."):
    df = yf.Ticker(ticker_map[asset]).history(period=period_map[timeframe], interval=interval_map[timeframe])
    
    if not df.empty:
        # 指標計算（20本移動平均）
        df['SMA20'] = df['Close'].rolling(window=20).mean()
        
        latest_price = df['Close'].iloc[-1]
        price_change = df['Close'].iloc[-1] - df['Open'].iloc[0]
        price_change_pct = (price_change / df['Open'].iloc[0]) * 100
        
        st.metric(f"{asset} 現在値", f"${latest_price:,.2f}", f"{price_change_pct:+.2f}% (期間始値比)")
        
        fig = go.Figure()
        # ローソク足
        fig.add_trace(go.Candlestick(
            x=df.index, open=df['Open'], high=df['High'], low=df['Low'], close=df['Close'],
            name="価格", increasing_line_color='#26a69a', decreasing_line_color='#ef5350'
        ))
        # 移動平均線
        fig.add_trace(go.Scatter(x=df.index, y=df['SMA20'], line=dict(color='#ffa726', width=1.5), name="SMA20"))
        
        fig.update_layout(
            xaxis_rangeslider_visible=False,
            height=500,
            template="plotly_dark",
            margin=dict(l=0, r=0, t=0, b=0),
            legend=dict(orientation="h", yanchor="bottom", y=1.02, xanchor="right", x=1)
        )
        st.plotly_chart(fig, use_container_width=True)
    else:
        st.warning("現在、市場からデータを取得できません。")

st.divider()

# ==========================================
# 4. ヒートマップ (簡易デモ)
# ==========================================
st.subheader("🗺️ セクター別市場ヒートマップ (騰落率目安)")
heatmap_data = pd.DataFrame(
    [[0.5, 1.2, -0.3], [-0.8, -1.5, 0.2], [2.1, 0.5, 1.1], [-0.1, -0.4, 0.0]],
    index=["📱 ハイテク", "🏦 金融", "⚡ エネルギー", "💊 ヘルスケア"],
    columns=["大型株", "中型株", "小型株"]
)
st.table(heatmap_data.style.background_gradient(cmap="RdYlGn", axis=None).format("{:+.2f}%"))
