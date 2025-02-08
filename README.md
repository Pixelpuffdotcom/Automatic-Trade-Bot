# Automatic-Trade-Bot
import os
import time
import sqlite3
import logging
import smtplib
import pandas as pd
import requests
import numpy as np
from datetime import datetime, timedelta
from dotenv import load_dotenv
from email.message import EmailMessage

# Load environment variables
load_dotenv()

# Configuration
CONFIG = {
    "DHAN_CLIENT_ID": os.getenv("DHAN_CLIENT_ID"),
    "DHAN_ACCESS_TOKEN": os.getenv("DHAN_ACCESS_TOKEN"),
    "DB_FILE": "trading_db.sqlite",
    "MAX_DAILY_LOSS": 2.0,  # 2% of capital
    "POSITION_SIZE": 0.2,    # 20% per trade
    "NOTIFICATION_EMAIL": os.getenv("NOTIFICATION_EMAIL"),
    "EMAIL_PASSWORD": os.getenv("EMAIL_PASSWORD"),
    "NIFTY_25": ["RELIANCE", "TATAMOTORS", "HDFCBANK", "ICICIBANK", "INFY",
                "TCS", "KOTAKBANK", "AXISBANK", "LT", "SBIN", 
                "ITC", "HINDUNILVR", "ONGC", "NTPC", "POWERGRID",
                "SUNPHARMA", "MARUTI", "TITAN", "ULTRACEMCO", "BAJAJ-AUTO",
                "BAJFINANCE", "HCLTECH", "WIPRO", "ADANIPORTS", "JSWSTEEL"]
}

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("trading_bot.log"),
        logging.StreamHandler()
    ]
)

class EnhancedDhanAPI:
    def __init__(self):
        self.base_url = "https://api.dhan.co"
        self.headers = {
            "access-token": CONFIG["DHAN_ACCESS_TOKEN"],
            "Content-Type": "application/json"
        }
        self.session = requests.Session()
        self.session.headers.update(self.headers)
        
    def _make_request(self, method, endpoint, **kwargs):
        """Generic request handler with retry logic"""
        retries = 3
        for attempt in range(retries):
            try:
                response = self.session.request(method, self.base_url + endpoint, **kwargs)
                response.raise_for_status()
                return response.json()
            except requests.exceptions.RequestException as e:
                logging.error(f"Attempt {attempt+1} failed: {str(e)}")
                time.sleep(2 ** attempt)
        return None

    def get_historical_data(self, symbol, interval, duration):
        """Get historical data with caching"""
        cache_key = f"{symbol}_{interval}_{duration}"
        try:
            return pd.read_pickle(f"cache/{cache_key}.pkl")
        except FileNotFoundError:
            pass
            
        params = {
            "instrument": symbol,
            "segment": "NSE",
            "interval": interval,
            "count": duration
        }
        data = self._make_request("GET", "/historical/chart/"+symbol, params=params)
        if data:
            df = pd.DataFrame(data['data'])
            df['timestamp'] = pd.to_datetime(df['timestamp'], unit='ms')
            df.set_index('timestamp', inplace=True)
            os.makedirs("cache", exist_ok=True)
            df.to_pickle(f"cache/{cache_key}.pkl")
            return df
        return None

    def place_order(self, symbol, action, quantity):
        """Enhanced order placement with confirmation"""
        payload = {
            "symbol": symbol,
            "exchange": "NSE",
            "segment": "EQ",
            "orderType": "MARKET",
            "type": action.upper(),
            "quantity": quantity,
            "product": "CNC",
            "validity": "DAY"
        }
        response = self._make_request("POST", "/orders", json=payload)
        if response and 'orderId' in response:
            return self.confirm_order_execution(response['orderId'])
        return None

    def confirm_order_execution(self, order_id):
        """Check order status until confirmation"""
        for _ in range(5):
            time.sleep(2)
            status = self._make_request("GET", f"/orders/{order_id}")
            if status and status['status'] == 'EXECUTED':
                return order_id
        return None

class RiskManager:
    def __init__(self):
        self.conn = sqlite3.connect(CONFIG["DB_FILE"])
        self.max_daily_loss = CONFIG["MAX_DAILY_LOSS"]
        self.position_size = CONFIG["POSITION_SIZE"]
        
    def check_daily_loss(self):
        query = """SELECT SUM(profit) FROM trades 
                   WHERE date(timestamp) = date('now')"""
        daily_pnl = self.conn.execute(query).fetchone()[0] or 0
        return daily_pnl < -self.max_daily_loss
    
    def calculate_position_size(self, portfolio_value):
        return portfolio_value * self.position_size
    
    def circuit_breaker(self):
        return self.check_daily_loss()

class PerformanceTracker:
    def __init__(self):
        self.conn = sqlite3.connect(CONFIG["DB_FILE"])
        self._init_db()
        
    def _init_db(self):
        self.conn.execute("""CREATE TABLE IF NOT EXISTS trades (
                            id INTEGER PRIMARY KEY,
                            timestamp DATETIME,
                            symbol TEXT,
                            action TEXT,
                            quantity INTEGER,
                            price REAL,
                            order_id TEXT)""")
        
        self.conn.execute("""CREATE TABLE IF NOT EXISTS performance (
                            date DATE PRIMARY KEY,
                            returns REAL,
                            volatility REAL,
                            max_drawdown REAL)""")
        self.conn.commit()
        
    def update_trade(self, trade_data):
        query = """INSERT INTO trades 
                   (timestamp, symbol, action, quantity, price, order_id)
                   VALUES (?, ?, ?, ?, ?, ?)"""
        self.conn.execute(query, trade_data)
        self.conn.commit()
        
    def calculate_metrics(self):
        # Implement advanced performance metrics
        pass

class NotificationSystem:
    @staticmethod
    def send_alert(subject, message):
        msg = EmailMessage()
        msg.set_content(message)
        msg['Subject'] = subject
        msg['From'] = CONFIG["NOTIFICATION_EMAIL"]
        msg['To'] = CONFIG["NOTIFICATION_EMAIL"]
        
        try:
            with smtplib.SMTP_SSL('smtp.gmail.com', 465) as smtp:
                smtp.login(CONFIG["NOTIFICATION_EMAIL"], 
                          CONFIG["EMAIL_PASSWORD"])
                smtp.send_message(msg)
        except Exception as e:
            logging.error(f"Email send failed: {str(e)}")

class TradingEngine:
    def __init__(self):
        self.api = EnhancedDhanAPI()
        self.risk_manager = RiskManager()
        self.performance_tracker = PerformanceTracker()
        self.portfolio_value = 1000000  # Initial capital
        
    def market_hours(self):
        now = datetime.now().time()
        return datetime.strptime("09:15", "%H:%M").time() < now < \
               datetime.strptime("15:30", "%H:%M").time()
               
    def select_stocks(self):
        # Implement enhanced fundamental+technical screening
        return CONFIG["NIFTY_25"][:
