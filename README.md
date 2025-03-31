# git-clone-https-github.com-binance-grid-trader.git-cd-binance-grid-trader
import json
import hmac
import hashlib
import requests
import signal
import sys
import os
import time
from datetime import datetime

class BinanceGridTrader:
    def __init__(self, config_file='config.json'):
        # Инициализация флагов и обработчиков сигналов
        self.shutdown_flag = False
        signal.signal(signal.SIGINT, self.handle_signal)
        signal.signal(signal.SIGTERM, self.handle_signal)
        
        # Загрузка конфигурации
        self.load_config(config_file)
        
        # Настройки API
        self.base_url = 'https://api.binance.com'
        self.headers = {'X-MBX-APIKEY': self.api_key}
        
        # Параметры торговли
        self.symbol = 'BTCUSDT'
        self.timeframe = '1h'
        self.base_amount = 0.001
        self.trend_multiplier = 1.5
        self.counter_multiplier = 3
        self.max_orders = 10
        self.profit_target = 1.005
        self.drawdown_limit = 0.01
        self.order_spread = 0.005
        
        # Получение информации о символе и tick size
        self.symbol_info = self.get_symbol_info()
        if not self.symbol_info:
            raise Exception("Failed to get symbol info")
        self.tick_size = float(self.symbol_info['filters'][0]['tickSize'])
        
        # Состояние системы
        self.active_orders = []
        self.trade_history = []
        self.last_candle_time = None
        self.current_trend = None
        self.initial_balance = self.get_account_balance()
        self.current_balance = self.initial_balance
        self.retry_count = 3
        self.retry_delay = 5

    def handle_signal(self, signum, frame):
        """Обработчик сигналов для graceful shutdown"""
        print(f"\nReceived signal {signum}, initiating graceful shutdown...")
        self.shutdown_flag = True

    def load_config(self, config_file):
        """Загрузка конфигурации"""
        try:
            if os.path.exists(config_file):
                with open(config_file) as f:
                    config = json.load(f)
                self.api_key = config['api_key']
                self.api_secret = config['api_secret']
            else:
                self.api_key = os.environ['BINANCE_API_KEY']
                self.api_secret = os.environ['BINANCE_API_SECRET']
        except Exception as e:
            raise Exception(f"Configuration load error: {str(e)}")

    def get_symbol_info(self):
        """Получение информации о символе (включая tick size)"""
        try:
            response = requests.get(
                f"{self.base_url}/api/v3/exchangeInfo",
                params={'symbol': self.symbol}
            )
            response.raise_for_status()
            data = response.json()
            for symbol_data in data['symbols']:
                if symbol_data['symbol'] == self.symbol:
                    return symbol_data
            return None
        except Exception as e:
            print(f"Symbol info error: {str(e)}")
            return None

    def validate_price(self, price):
        """Корректировка цены под tick size"""
        try:
            return round(price / self.tick_size) * self.tick_size
        except Exception as e:
            print(f"Price validation error: {str(e)}")
            return None

    def make_api_request(self, endpoint, params=None, method='GET', signed=False):
        """Улучшенный API-запрос с обработкой ошибок"""
        url = f"{self.base_url}{endpoint}"
        
        for attempt in range(self.retry_count):
            try:
                if signed:
                    params = params or {}
                    params['timestamp'] = int(time.time() * 1000)
                    query_string = '&'.join([f"{k}={v}" for k, v in params.items()])
                    signature = hmac.new(
                        self.api_secret.encode('utf-8'),
                        query_string.encode('utf-8'),
                        hashlib.sha256
                    ).hexdigest()
                    params['signature'] = signature
                
                if method == 'GET':
                    response = requests.get(url, headers=self.headers, params=params, timeout=10)
                else:
                    response = requests.post(url, headers=self.headers, data=params, timeout=10)
                
                response.raise_for_status()
                return response.json()
                
            except requests.exceptions.ConnectionError:
                print(f"Connection error (attempt {attempt + 1}), retrying...")
                if attempt == self.retry_count - 1:
                    raise
                time.sleep(self.retry_delay * (attempt + 1))
                
            except requests.exceptions.RequestException as e:
                print(f"API request failed (attempt {attempt + 1}): {str(e)}")
                if attempt == self.retry_count - 1:
                    raise
                time.sleep(self.retry_delay)
        return None

    def place_order(self, side, price, amount):
        """Размещение ордера с проверкой tick size"""
        validated_price = self.validate_price(price)
        if validated_price is None:
            print(f"Invalid price {price} after tick size validation")
            return None
            
        try:
            order = self.make_api_request(
                '/api/v3/order',
                method='POST',
                signed=True,
                params={
                    'symbol': self.symbol,
                    'side': side,
                    'type': 'LIMIT',
                    'timeInForce': 'GTC',
                    'quantity': round(amount, 6),
                    'price': round(validated_price, 2),
                    'recvWindow': 5000
                }
            )
            return order
        except Exception as e:
            print(f"Order placement error: {str(e)}")
            return None

    def cancel_all_orders(self):
        """Отмена всех активных ордеров"""
        try:
            orders = self.make_api_request(
                '/api/v3/openOrders',
                signed=True,
                params={'symbol': self.symbol}
            )
            for order in orders:
                self.cancel_order(order['orderId'])
            self.active_orders = []
            return True
        except Exception as e:
            print(f"Cancel all orders error: {str(e)}")
            return False

    def add_recovery_orders(self, drawdown):
        """Добавление ордеров с проверкой лимита"""
        if len(self.active_orders) >= self.max_orders:
            print(f"Max orders limit reached ({self.max_orders})")
            return
            
        current_price = self.get_current_price()
        if current_price is None:
            return

        multiplier = min(2, 1 + (drawdown / self.drawdown_limit))
        
        buy_price = self.validate_price(current_price * (1 - self.order_spread * multiplier))
        buy_amount = round(self.base_amount * (self.counter_multiplier ** multiplier), 6)
        
        sell_price = self.validate_price(current_price * (1 + self.order_spread * multiplier))
        sell_amount = round(self.base_amount * (self.trend_multiplier ** multiplier), 6)
        
        if buy_price is None or sell_price is None:
            return
            
        buy_order = self.place_order('BUY', buy_price, buy_amount)
        sell_order = self.place_order('SELL', sell_price, sell_amount)
        
        if buy_order and sell_order:
            self.active_orders.extend([buy_order, sell_order])
            self.record_trade('recovery', buy_order, sell_order)

    def cleanup(self):
        """Очистка перед завершением работы"""
        print("\nPerforming cleanup...")
        self.cancel_all_orders()
        print("Cleanup completed. Exiting.")

    def run(self):
        """Основной цикл с обработкой graceful shutdown"""
        print(f"Starting Binance Grid Trader for {self.symbol}")
        print(f"Initial balance: {self.initial_balance:.2f} USDT")
        
        try:
            while not self.shutdown_flag:
                try:
                    if self.check_new_candle():
                        # ... остальная торговая логика ...
                        pass
                        
                    time.sleep(60)