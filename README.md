# ETH-Burn

import streamlit as st
import yfinance as yf
import pandas as pd
import time
from datetime import datetime
import pytz
import requests # Added for API calls

# --- DATA UPDATES ---
SHARES = 435_666_174
CASH = 1_000_000_000
BTC_HELD = 193
ETH_HELD = 4_110_525 
EIGHT_STOCK_VALUE = 23_000_000
ETH_STAKED = 408_627  
ANNUAL_STAKING_APR = 0.03
ETH_DAILY_ISSUANCE = 2700 # Approximate PoS daily issuance

st.set_page_config(page_title="BMNR NAV Tracker", page_icon="📈", layout="wide")

# --- CUSTOM UI STYLING ---
st.markdown("""
    <style>
    [data-testid="stMetricLabel"] p { color: #ADD8E6 !important; font-size: 1.1rem !important; font-weight: bold !important; }
    [data-testid="stMetricValue"] div { color: #ADD8E6 !important; font-size: 2.2rem !important; }
    .timestamp { color: #888888; font-size: 0.9rem; margin-top: -20px; margin-bottom: 20px; }
    </style>
    """, unsafe_allow_html=True)

@st.cache_data(ttl=60)
def fetch_burn_rate():
    """Fetches real-time burn data (using ultrasound.money or similar public API concepts)"""
    try:
        # Example using a public data endpoint for burn rate (ETH/day)
        # Note: In a production app, you'd use a dedicated API key from Etherscan or Alchemy
        response = requests.get("https://api.ultrasound.money/v1/burn-rate", timeout=5)
        data = response.json()
        return data.get('burn_rate_24h', 2100) # Fallback to 2100 if API fails
    except:
        return 2100 # Estimated historical average

def fetch_prices():
    try:
        bmnr = yf.Ticker("BMNR").fast_info.last_price
        eth = yf.Ticker("ETH-USD").fast_info.last_price
        btc = yf.Ticker("BTC-USD").fast_info.last_price
        return bmnr, eth, btc
    except:
        return 0.0, 0.0, 0.0

bmnr_p, eth_p, btc_p = fetch_prices()
current_burn_rate = fetch_burn_rate()

# --- CALCULATIONS ---
if bmnr_p > 0 and eth_p > 0:
    val_eth = ETH_HELD * eth_p
    val_btc = BTC_HELD * btc_p
    total_nav = val_eth + val_btc + CASH + EIGHT_STOCK_VALUE
    nav_per_share = total_nav / SHARES
    mnav = (bmnr_p * SHARES) / total_nav
    eth_per_share = ETH_HELD / SHARES
    
    # Burn Analytics
    is_deflationary = current_burn_rate > ETH_DAILY_ISSUANCE
    burn_deficit = max(0, ETH_DAILY_ISSUANCE - current_burn_rate)
    
    # --- HEADER SECTION ---
    st.title("BMNR mNAV Tracker")
    est_time = datetime.now(pytz.timezone('US/Eastern')).strftime('%Y-%m-%d %I:%M:%S %p')
    st.markdown(f'<p class="timestamp">Last Updated: {est_time} EST</p>', unsafe_allow_html=True)

    # TOP METRICS ROW
    m1, m2, m3, m4, m5 = st.columns(5)
    with m1: st.metric("BMNR Price", f"${bmnr_p:,.2f}")    
    with m2: st.metric("NAV/Share", f"${nav_per_share:,.2f}")
    with m3: st.metric("mNAV (Total)", f"{mnav:.3f}x")
    with m4: st.metric("ETH/Share", f"{eth_per_share:.6f}")
    with m5: st.metric("Burn Rate (24h)", f"{current_burn_rate:,.0f} ETH")

    st.divider()

    # --- ETH BURN & DEFLATION SECTION ---
    st.subheader("Ethereum Supply Dynamics")
    col1, col2, col3 = st.columns(3)
    
    with col1:
        st.write("**Current Daily Burn**")
        st.write(f"🔥 {current_burn_rate:,.2f} ETH / day")
        
    with col2:
        st.write("**Target for Deflation**")
        st.write(f"🎯 > {ETH_DAILY_ISSUANCE:,.0f} ETH / day")
        
    with col3:
        status_color = "green" if is_deflationary else "orange"
        status_text = "DEFLATIONARY" if is_deflationary else "INFLATIONARY"
        st.markdown(f"**Status:** <span style='color:{status_color}; font-weight:bold;'>{status_text}</span>", unsafe_allow_html=True)
        if not is_deflationary:
            st.write(f"Deficit: {burn_deficit:,.0f} ETH/day")

    st.divider()

    # --- TREASURY BREAKDOWN ---
    st.subheader("Treasury Breakdown")
    # ... (Rest of your treasury dataframe code from previous steps) ...
    assets_data = {
        "Asset": ["Ethereum (ETH)", "Bitcoin (BTC)", "Cash", "Eightco Stake"],
        "Total Quantity": [ETH_HELD, BTC_HELD, 0, 0],
        "Live Price": [eth_p, btc_p, 0, 0],
        "Staked Amount": [ETH_STAKED, 0, 0, 0],
        "Total Value": [val_eth, val_btc, CASH, EIGHT_STOCK_VALUE]
    }
    st.dataframe(pd.DataFrame(assets_data), use_container_width=True, hide_index=True)

    time.sleep(60)
    st.rerun()
