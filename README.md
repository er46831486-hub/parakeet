# parakeet
import streamlit as st
import pandas as pd
import requests
import folium
from streamlit_folium import folium_static
from datetime import datetime
import pytz
import random

# ==========================================
# 1. 基礎設定與 PWA 配置
# ==========================================
st.set_page_config(
    page_title="Tokyo Travel Hub",
    page_icon="🗼",
    layout="centered",
    initial_sidebar_state="collapsed"
)

# 注入 iOS PWA 支援與 Cyberpunk UI 樣式
st.markdown("""
    <meta name="apple-mobile-web-app-capable" content="yes">
    <meta name="apple-mobile-web-app-status-bar-style" content="black-translucent">
    <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no">
    
    <style>
    /* 全域字體與背景 - 深色未來感 */
    .stApp {
        background-color: #0e1117;
        background-image: radial-gradient(circle at 50% 50%, #1c202b 0%, #0e1117 100%);
        font-family: 'Helvetica Neue', Helvetica, Arial, sans-serif;
    }
    
    /* 隱藏預設 Header 與 Footer */
    header {visibility: hidden;}
    footer {visibility: hidden;}
    
    /* 玻璃擬態卡片 (Glassmorphism) */
    .glass-card {
        background: rgba(255, 255, 255, 0.05);
        backdrop-filter: blur(16px);
        -webkit-backdrop-filter: blur(16px);
        border: 1px solid rgba(255, 255, 255, 0.1);
        border-radius: 16px;
        padding: 20px;
        margin-bottom: 15px;
        box-shadow: 0 4px 30px rgba(0, 0, 0, 0.5);
        color: #ffffff;
    }
    
    /* HUD 風格標題 */
    .hud-title {
        font-size: 14px;
        color: #00f2ff;
        text-transform: uppercase;
        letter-spacing: 2px;
        margin-bottom: 10px;
        border-bottom: 1px solid rgba(0, 242, 255, 0.3);
        padding-bottom: 5px;
        display: flex;
        justify-content: space-between;
    }

    /* 霓虹文字 */
    .neon-text {
        text-shadow: 0 0 5px #00f2ff, 0 0 10px #00f2ff;
    }
    
    /* 行程時間軸 */
    .timeline-item {
        border-left: 2px solid #ff0055;
        padding-left: 15px;
        margin-bottom: 15px;
        position: relative;
    }
    .timeline-item::before {
        content: '';
        position: absolute;
        left: -6px;
        top: 0;
        width: 10px;
        height: 10px;
        background: #ff0055;
        border-radius: 50%;
        box-shadow: 0 0 10px #ff0055;
    }
    
    /* 按鈕樣式 */
    .stButton>button {
        background: linear-gradient(90deg, #00f2ff 0%, #0078ff 100%);
        color: black;
        border: none;
        border-radius: 8px;
        font-weight: bold;
        transition: all 0.3s ease;
    }
    .stButton>button:hover {
        box-shadow: 0 0 15px #00f2ff;
        transform: scale(1.02);
    }
    
    /* 底部導航欄模擬 */
    .bottom-nav {
        position: fixed;
        bottom: 0;
        left: 0;
        width: 100%;
        background: rgba(14, 17, 23, 0.95);
        border-top: 1px solid #333;
        padding: 15px;
        text-align: center;
        z-index: 999;
        font-size: 12px;
        color: #888;
    }
    </style>
""", unsafe_allow_html=True)

# ==========================================
# 2. 資料處理 (行程資料庫)
# ==========================================
# 將 CSV 資料整合為結構化 Data
ITINERARY = {
    "2025-12-28": {
        "title": "DAY 1: 降臨東京",
        "loc": [35.6895, 139.6917], # 新宿
        "events": [
            {"time": "06:30", "loc": "成田機場", "act": "抵達 (Skyline -> 日暮里 -> 新宿)"},
            {"time": "14:00", "loc": "新宿", "act": "午餐: Gyu Tongue Lemon"},
            {"time": "16:00", "loc": "新宿/池袋", "act": "逛街 / Check-in"},
            {"time": "19:00", "loc": "池袋", "act": "晚餐: Steak Rice & Curry 或 銀座篝"},
        ]
    },
    "2025-12-29": {
        "title": "DAY 2: 澀谷事變",
        "loc": [35.6580, 139.7016], # 澀谷
        "events": [
            {"time": "07:00", "loc": "築地市場", "act": "海鮮早餐"},
            {"time": "10:00", "loc": "東京鐵塔", "act": "赤羽橋/芝公園散步"},
            {"time": "14:00", "loc": "澀谷", "act": "逛街 Shopping"},
            {"time": "15:00", "loc": "Shibuya Sky", "act": "展望台 (需預約!)"},
            {"time": "21:00", "loc": "澀谷", "act": "晚餐: BISTRO MEATMAN / 居酒屋"},
        ]
    },
    "2025-12-30": {
        "title": "DAY 3: 湘南海岸",
        "loc": [35.3190, 139.5506], # 鐮倉
        "events": [
            {"time": "08:00", "loc": "鐮倉高校前", "act": "灌籃高手平交道 / 七里濱"},
            {"time": "11:00", "loc": "小町通", "act": "午餐 & 逛街"},
            {"time": "14:00", "loc": "江之島", "act": "江島神社 / 邊津宮洗錢"},
            {"time": "19:30", "loc": "鐮倉 -> 新宿", "act": "返程 (橫須賀線)"},
        ]
    },
    "2025-12-31": {
        "title": "DAY 4: 跨年百鬼夜行",
        "loc": [35.7147, 139.7963], # 淺草
        "events": [
            {"time": "09:30", "loc": "淺草", "act": "梨花和服 (已預約)"},
            {"time": "11:00", "loc": "淺草寺", "act": "雷門 / 仲見世通 / 隅田公園"},
            {"time": "18:00", "loc": "上野/阿美橫町", "act": "晚餐 / 買蕎麥麵"},
            {"time": "22:30", "loc": "增上寺", "act": "聽 108 下鐘聲跨年"},
        ]
    },
    "2026-01-01": {
        "title": "DAY 5: 富士朝聖",
        "loc": [35.4856, 138.8028], # 下吉田
        "events": [
            {"time": "07:35", "loc": "新宿 BUSTA", "act": "巴士前往中央道下吉田"},
            {"time": "09:30", "loc": "新倉山淺間神社", "act": "展望台看富士山"},
            {"time": "14:00", "loc": "河口湖", "act": "大池公園 / LAWSON"},
            {"time": "20:00", "loc": "東京車站", "act": "丸之內聖誕燈飾"},
        ]
    },
    "2026-01-02": {
        "title": "DAY 6: 歸途",
        "loc": [35.7719, 140.3929], # 成田
        "events": [
            {"time": "10:00", "loc": "日暮里", "act": "Citywalk / 最後補貨"},
            {"time": "13:05", "loc": "日暮里 -> 成田", "act": "搭乘 Skyline"},
            {"time": "16:35", "loc": "成田一航", "act": "起飛回家"},
        ]
    }
}

# ==========================================
# 3. 功能模組函數
# ==========================================

def get_weather(lat, lon):
    """取得 Open-Meteo 即時天氣"""
    try:
        url = f"https://api.open-meteo.com/v1/forecast?latitude={lat}&longitude={lon}&current=temperature_2m,relative_humidity_2m,apparent_temperature,is_day,precipitation,weather_code,wind_speed_10m&timezone=Asia%2FTokyo"
        r = requests.get(url).json()
        current = r['current']
        
        # 簡單的天氣代碼轉換
        code = current['weather_code']
        icon = "☁️"
        if code <= 3: icon = "☀️"
        elif code <= 49: icon = "🌫️"
        elif code <= 69: icon = "🌧️"
        elif code >= 70: icon = "❄️"
        
        return {
            "temp": f"{current['temperature_2m']}°C",
            "feel": f"{current['apparent_temperature']}°C",
            "rain": f"{current['precipitation']} mm",
            "wind": f"{current['wind_speed_10m']} km/h",
            "icon": icon
        }
    except:
        return {"temp": "--", "feel": "--", "rain": "--", "wind": "--", "icon": "⚠️"}

def get_exchange_rate():
    """模擬匯率 API (避免 Key 失效，使用靜態或簡單爬蟲概念)"""
    # 這裡為了演示穩定性，設定一個動態變化的假數值，實際可換成 twd.rter.info API
    base_rate = 0.215
    variation = random.uniform(-0.002, 0.002)
    return f"{base_rate + variation:.4f}"

# ==========================================
# 4. 主畫面 UI 建構
# ==========================================

# --- Header: 日期與狀態 ---
tokyo_tz = pytz.timezone('Asia/Tokyo')
now_tokyo = datetime.now(tokyo_tz)
date_str = now_tokyo.strftime("%Y-%m-%d")
time_str = now_tokyo.strftime("%H:%M")

# 判斷今天是行程的哪一天
current_key = date_str
if current_key not in ITINERARY:
    # 如果不在行程日期內，預設顯示第一天或當作測試
    current_key = "2025-12-28" 
    status_msg = "⚠️ 非行程日期 (預覽模式)"
else:
    status_msg = "🟢 正在執行行程"

day_data = ITINERARY[current_key]

# --- UI: Weather HUD ---
weather = get_weather(day_data['loc'][0], day_data['loc'][1])
rate = get_exchange_rate()

st.markdown(f"""
    <div style="text-align:center; margin-bottom: 20px;">
        <h1 style="margin:0; font-size: 2.5em; font-weight: 800; background: -webkit-linear-gradient(#eee, #333); -webkit-background-clip: text; -webkit-text-fill-color: transparent;">TOKYO HUB</h1>
        <div style="color: #00f2ff; font-family: monospace;">{date_str} <span style="color: #ff0055">{time_str}</span> JST</div>
    </div>
""", unsafe_allow_html=True)

# 天氣卡片
st.markdown(f"""
    <div class="glass-card">
        <div class="hud-title">
            <span>WEATHER SYSTEM</span>
            <span>TOKYO</span>
        </div>
        <div style="display: flex; justify-content: space-between; align-items: center;">
            <div style="font-size: 3em;">{weather['icon']}</div>
            <div style="text-align: right;">
                <div style="font-size: 2em; font-weight: bold;">{weather['temp']}</div>
                <div style="color: #ccc; font-size: 0.8em;">體感 {weather['feel']}</div>
            </div>
        </div>
        <div style="display: flex; justify-content: space-between; margin-top: 10px; font-size: 0.9em; color: #888;">
            <span>💧 降雨: {weather['rain']}</span>
            <span>💨 風速: {weather['wind']}</span>
        </div>
    </div>
""", unsafe_allow_html=True)

# 匯率與交通狀態
col1, col2 = st.columns(2)
with col1:
    st.markdown(f"""
        <div class="glass-card" style="text-align: center;">
            <div class="hud-title">JPY/TWD</div>
            <div style="font-size: 1.5em; color: #00ff9d;">{rate}</div>
            <div style="font-size: 0.7em; color: #666;">即時匯率</div>
        </div>
    """, unsafe_allow_html=True)
with col2:
    st.markdown(f"""
        <div class="glass-card" style="text-align: center;">
            <div class="hud-title">TRAIN</div>
            <div style="font-size: 1.5em; color: #ff0055;">Normal</div>
            <div style="font-size: 0.7em; color: #666;">山手線/地鐵</div>
        </div>
    """, unsafe_allow_html=True)

# --- UI: 行程地圖 (Interactive Map) ---
st.markdown('<div class="hud-title">LIVE LOCATION TRACKING</div>', unsafe_allow_html=True)

m = folium.Map(location=day_data['loc'], zoom_start=13, tiles="CartoDB dark_matter")

# 標記當日主要地點
folium.Marker(
    day_data['loc'], 
    popup=day_data['title'],
    icon=folium.Icon(color="red", icon="info-sign")
).add_to(m)

# 模擬車站位置 (範例)
folium.CircleMarker(
    location=[day_data['loc'][0]+0.002, day_data['loc'][1]+0.002],
    radius=5,
    color="#00f2ff",
    fill=True,
    fill_color="#00f2ff",
    popup="Nearest Station"
).add_to(m)

folium_static(m, height=250)

# --- UI: 今日任務 (Timeline) ---
st.markdown(f"""
    <div class="glass-card">
        <div class="hud-title">MISSION LOG: {day_data['title']}</div>
        <div style="margin-top: 15px;">
""", unsafe_allow_html=True)

for event in day_data['events']:
    # 判斷任務狀態 (假設過去時間為完成)
    event_hour = int(event['time'].split(":")[0])
    is_done = now_tokyo.hour > event_hour
    
    color = "#444" if is_done else "#fff"
    icon = "✅" if is_done else "💠"
    
    st.markdown(f"""
        <div class="timeline-item" style="border-color: {('#333' if is_done else '#ff0055')}">
            <div style="color: #00f2ff; font-size: 0.8em; font-family: monospace;">{event['time']}</div>
            <div style="font-weight: bold; color: {color}; font-size: 1.1em;">{event['loc']}</div>
            <div style="color: #aaa; font-size: 0.9em;">{event['act']}</div>
        </div>
    """, unsafe_allow_html=True)

st.markdown("</div></div>", unsafe_allow_html=True)

# --- UI: 工具箱 (Expanders) ---
with st.expander("🛠️ 戰術支援 (Tools)"):
    tab1, tab2, tab3 = st.tabs(["翻譯", "緊急", "提示"])
    
    with tab1:
        st.text_input("輸入中文...", placeholder="想吃的餐廳怎麼說？")
        st.info("此功能需連接 OpenAI API (目前為展示模式)")
        
    with tab2:
        st.markdown("""
        - **警察**: 110
        - **救護車**: 119
        - **遺失物中心**: 03-3814-4151
        """)
        
    with tab3:
        st.warning("天氣寒冷，請務必攜帶圍巾與手套。")
        st.success("記得攜帶護照免稅 (Tax Free) QR Code。")

# 底部空間
st.markdown("<br><br><br>", unsafe_allow_html=True)

# 底部 Fake Nav
st.markdown("""
    <div class="bottom-nav">
        TOKYO TRAVEL HUB v1.0 | SYSTEM ONLINE
    </div>
""", unsafe_allow_html=True)
