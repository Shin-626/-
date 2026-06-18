# import streamlit as st
import yfinance as yf
import pandas as pd
import pandas_datareader.data as web
from datetime import datetime, timedelta
import plotly.graph_objects as go

# ページの設定
st.set_page_config(page_title="Global Economic Dashboard", layout="wide")
st.title("🌐 個人専用：世界経済・市場監視ダッシュボード")

# ----------------------------------------------------
# 1. 監視対象の定義（主要国・アセット）
# ----------------------------------------------------
market_symbols = {
    "株価指数": {
        "米国 (S&P 500)": "^GSPC",
        "日本 (日経225)": "^N225",
        "欧州 (Euro Stoxx 50)": "^STOXX50E",
        "中国 (上海総合)": "000001.SS",
        "英国 (FTSE 100)": "^FTSE"
    },
    "為替 (対日本円)": {
        "米ドル / 円 (USD/JPY)": "USDJPY=X",
        "ユーロ / 円 (EUR/JPY)": "EURJPY=X",
        "ポンド / 円 (GBP/JPY)": "GBPJPY=X",
        "人民元 / 円 (CNY/JPY)": "CNYJPY=X"
    }
}

# FREDから取得する主要経済指標 (Series ID)
economic_series = {
    "米国政策金利 (Effective Federal Funds Rate)": "FEDFUNDS",
    "米国消費者物価指数 (CPI YoY % Change)": "CPIAUCSL",
    "日本の政策金利 (Short-Term Policy Interest Rate)": "INTDSRJPM193N",
    "米10年物国債利回り (10-Year Treasury Constant Maturity Rate)": "DGS10"
}

# ----------------------------------------------------
# 2. データ取得関数
# ----------------------------------------------------
@st.cache_data(ttl=60) # 1分間キャッシュしてリロードを高速化
def get_market_data(ticker_symbol):
    ticker = yf.Ticker(ticker_symbol)
    # 本日と前日のデータを取得して前日比を計算
    df = ticker.history(period="2d")
    if len(df) >= 2:
        current_price = df['Close'].iloc[-1]
        prev_price = df['Close'].iloc[-2]
        change = current_price - prev_price
        pct_change = (change / prev_price) * 100
        return current_price, change, pct_change
    elif len(df) == 1:
        return df['Close'].iloc[-0], 0.0, 0.0
    return None, None, None

@st.cache_data(ttl=3600) # 経済指標は1時間キャッシュ
def get_economic_indicator(series_id):
    start = datetime.now() - timedelta(days=365 * 3) # 過去3年分
    end = datetime.now()
    try:
        df = web.DataReader(series_id, 'fred', start, end)
        return df
    except Exception as e:
        return None

# ----------------------------------------------------
# 3. UIの構築
# ----------------------------------------------------

# --- セクション1: 為替・株式（リアルタイム風） ---
st.header("📈 主要市場レート（為替・株価指数）")

for category, items in market_symbols.items():
    st.subheader(f"■ {category}")
    cols = st.columns(len(items))
    
    for idx, (name, symbol) in enumerate(items.items()):
        price, change, pct_change = get_market_data(symbol)
        with cols[idx]:
            if price is not None:
                # 通貨や指数に応じた小数点以下の桁数調整
                fmt = ".4f" if "JPY=X" in symbol else ".2f"
                st.metric(
                    label=name,
                    value=f"{price:{fmt}}",
                    delta=f"{change:{fmt}} ({pct_change:.2f}%)"
                )
            else:
                st.metric(label=name, value="データ取得エラー")

st.markdown("---")

# --- セクション2: 経済指標の精査 ---
st.header("📊 主要国 経済指標アナリティクス")

selected_indicator = st.selectbox(
    "分析する経済指標を選択してください:",
    list(economic_series.keys())
)

series_id = economic_series[selected_indicator]
eco_data = get_economic_indicator(series_id)

if eco_data is not None and not eco_data.empty:
    # 直近の値を表示
    latest_date = eco_data.index[-1].strftime('%Y-%m-%d')
    latest_value = eco_data.iloc[-1, 0]
    
    # CPIなどの場合は前年比計算が必要な場合もありますが、FREDのデータソース自体が調整済みのものを指定しています
    st.write(f"**最新発表値:** {latest_value:.2f} （発表日: {latest_date}）")
    
    # Plotlyでインタラクティブなチャートを描画
    fig = go.Figure()
    fig.add_trace(go.Scatter(
        x=eco_data.index, 
        y=eco_data.iloc[:, 0], 
        mode='lines', 
        name=selected_indicator,
        line=dict(color='#00b4d8', width=2)
    ))
    fig.update_layout(
        title=f"{selected_indicator} の推移 (過去3年)",
        xaxis_title="日付",
        yaxis_title="値 (%)",
        template="plotly_dark",
        margin=dict(l=40, r=40, t=40, b=40)
    )
    st.plotly_chart(fig, use_container_width=True)
else:
    st.error("経済指標データの取得に失敗しました。時間をおいて再度お試しください。")
    
