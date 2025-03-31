# git-clone-https-github.com-binance-grid-trader.git-cd-binance-grid-trader
import threading
from queue import Queue
from binance import Client, BinanceAPIException
from binance.enums import *
import time
import os
import pickle
import logging
from decimal import Decimal, getcontext

class SpotGridTradingBot:
    def __init__(self, api_key, api_secret, symbol='BTCUSDT',
                 take_profit=0.005, grid_step=0.0025,
                 max_levels=5, trend_multiplier=1.2,
                 counter_trend_multiplier=1.5,
                 state_file='bot_state.pkl'):
        
        # Установка точности для Decimal
        getcontext().prec = 8
        
        # Инициализация клиента Binance
        self.client = Client(api_key, api_secret)
        self.symbol = symbol
        self.is_active = True
        self.state_file = state_file
        
        # Параметры стратегии
        self.take_profit = Decimal(str(take_profit))
        self.grid_step = Decimal(str(grid_step))
        self.max_levels = max_levels
        self.trend_multiplier = Decimal(str(trend_multiplier))
        self.counter_trend_multiplier = Decimal(str(counter_trend_multiplier))
        
        # Очередь ордеров и поток обработки
        self.order_queue = Queue()
        self.order_executor = threading.Thread(target=self._process_orders)
        self.order_executor.daemon = True
        
        # Инициализация параметров рынка
        self._load_market_info()
        
        # Восстановление состояния
        self._init_grid()
        self._restore_state()
        
        # Логирование
        self._setup_logging()
        
        # Для защиты от дублирования и управления ордерами
        self.active_orders = {}  # {order_id: {'side': side, 'level': level, 'price': price, 'quantity': qty}}
        self.order_lock = threading.Lock()
        self.last_order_check = 0
        self.emergency_flag = False
        self.hedge_attempts = {}  # Для отслеживания попыток хеджирования

    def _setup_logging(self):
        """Настройка расширенного логирования"""
        logging.basicConfig(
            level=logging.INFO,
            format='%(asctime)s - %(levelname)s - %(message)s',
            handlers=[
                logging.FileHandler('grid_trading.log'),
                logging.StreamHandler()
            ]
        )
        # Дополнительный логгер для экстренных ситуаций
        self.emergency_logger = logging.getLogger('emergency')
        emergency_handler = logging.FileHandler('emergency.log')
        emergency_handler.setFormatter(logging.Formatter('%(asctime)s - EMERGENCY - %(message)s'))
        self.emergency_logger.addHandler(emergency_handler)
        self.emergency_logger.setLevel(logging.WARNING)

    def _load_market_info(self):
        """Загрузка параметров торговой пары с обработкой ошибок"""
        try:
            info = self.client.get_symbol_info(self.symbol)
            self.filters = {
                'min_qty': Decimal(info['filters'][2]['minQty']),
                'step_size': Decimal(info['filters'][2]['stepSize']),
                'min_notional': Decimal(info['filters'][3]['minNotional']),
                'tick_size': Decimal(info['filters'][0]['tickSize'])
            }
            logging.info("Market info loaded successfully")
        except Exception as e:
            self.emergency_logger.critical(f"Failed to load market info: {str(e)}")
            raise

    def _init_grid(self):
        """Инициализация сетки с Decimal"""
        self.grid_levels = {}
        self.current_level = 0
        self.base_asset = self.symbol[:-4]  # BTC
        self.quote_asset = self.symbol[-4:]  # USDT
        self.update_balances()

    def _restore_state(self):
        """Восстановление состояния после перезапуска с обработкой ошибок"""
        try:
            if os.path.exists(self.state_file):
                with open(self.state_file, 'rb') as f:
                    state = pickle.load(f)
                    self.grid_levels = {int(k): v for k, v in state.get('grid_levels', {}).items()}
                    self.current_level = state.get('current_level', 0)
                    
                    # Конвертация в Decimal при восстановлении
                    for level, data in self.grid_levels.items():
                        if 'level_price' in data:
                            self.grid_levels[level]['level_price'] = Decimal(str(data['level_price']))
                        if 'buy_price' in data:  # Для обратной совместимости
                            self.grid_levels[level]['level_price'] = Decimal(str(data['buy_price']))
                            del self.grid_levels[level]['buy_price']
                        if 'sell_price' in data:  # Для обратной совместимости
                            del self.grid_levels[level]['sell_price']
                    
                    logging.info("State restored successfully")
        except Exception as e:
            self.emergency_logger.error(f"Error restoring state: {str(e)}")
            # В случае ошибки - начинаем с чистого состояния
            self._init_grid()

    def _save_state(self):
        """Атомарное сохранение текущего состояния"""
        try:
            # Конвертация Decimal в float для сериализации
            state_to_save = {
                'grid_levels': {
                    level: {
                        k: float(v) if isinstance(v, Decimal) else v 
                        for k, v in data.items()
                    } 
                    for level, data in self.grid_levels.items()
                },
                'current_level': self.current_level
            }
            
            # Атомарная запись через временный файл
            temp_file = self.state_file + '.tmp'
            with open(temp_file, 'wb') as f:
                pickle.dump(state_to_save, f)
            
            if os.path.exists(self.state_file):
                os.remove(self.state_file)
            os.rename(temp_file, self.state_file)
            
        except Exception as e:
            self.emergency_logger.error(f"Error saving state: {str(e)}")

    def update_balances(self):
        """Обновление балансов с Decimal"""
        try:
            base_balance = self.client.get_asset_balance(self.base_asset)
            quote_balance = self.client.get_asset_balance(self.quote_asset)
            
            self.balances = {
                self.base_asset: Decimal(base_balance['free']),
                self.quote_asset: Decimal(quote_balance['free'])
            }
        except Exception as e:
            self.emergency_logger.error(f"Error updating balances: {str(e)}")
            # В случае ошибки используем последние известные значения
            if not hasattr(self, 'balances'):
                self.balances = {
                    self.base_asset: Decimal('0'),
                    self.quote_asset: Decimal('0')
                }

    def _process_orders(self):
        """Обработка ордеров в фоновом режиме с трехуровневым хеджированием"""
        while self.is_active:
            try:
                side, volume, level = self.order_queue.get(timeout=1)
                order_key = f"{side}_{level}"
                
                try:
                    target_price = self._calculate_level_price(level)
                    
                    # Проверка минимального шага цены
                    if not self._validate_price_step(target_price):
                        logging.warning(f"Price {target_price} doesn't match tick size, skipping")
                        continue
                    
                    with self.order_lock:
                        # Проверка на существующий ордер в этом ценовом диапазоне
                        if self._is_order_active_near(target_price):
                            logging.info(f"Order already exists near {target_price}, skipping")
                            continue
                            
                        # Инициализация счетчика попыток для этого ордера
                        if order_key not in self.hedge_attempts:
                            self.hedge_attempts[order_key] = 0
                        
                        # Увеличиваем счетчик попыток
                        self.hedge_attempts[order_key] += 1
                        
                        # Трехуровневая система хеджирования
                        if self.hedge_attempts[order_key] == 1:
                            # Первая попытка: Рыночный ордер
                            logging.info(f"Attempting MARKET {side} for level {level}")
                            self._execute_market_order(side, volume, level, target_price)
                            
                        elif self.hedge_attempts[order_key] == 2:
                            # Вторая попытка: Лимитный ордер по расчетной цене
                            logging.info(f"Attempting LIMIT {side} for level {level}")
                            self._execute_limit_order(side, volume, level, target_price)
                            
                        elif self.hedge_attempts[order_key] >= 3:
                            # Третья попытка: Экстренное логирование и обработка
                            self.emergency_logger.warning(
                                f"Failed to execute {side} order for level {level} after 2 attempts. "
                                f"Volume: {volume}, Target price: {target_price}"
                            )
                            self._emergency_handling(side, volume, level, target_price)
                            continue
                            
                    time.sleep(0.5)
                except BinanceAPIException as e:
                    logging.error(f"Order failed: {e.status_code} {e.message}")
                    with self.order_lock:
                        self._cleanup_canceled_orders()
                finally:
                    self.order_queue.task_done()
                    self.update_balances()
                    self._save_state()
            except:
                continue

    def _execute_market_order(self, side, volume, level, target_price):
        """Попытка исполнения рыночного ордера"""
        try:
            if side == 'BUY':
                quote_needed = volume * target_price * Decimal('1.001')
                if self.balances[self.quote_asset] >= quote_needed:
                    order = self.client.create_order(
                        symbol=self.symbol,
                        side=SIDE_BUY,
                        type=ORDER_TYPE_MARKET,
                        quantity=self._round_quantity(volume)
                    
                    logging.info(f"MARKET BUY executed: {volume} @ market price")
                    self._register_order_execution(order, side, level, target_price, volume)
                    return True
                    
            elif side == 'SELL':
                if self.balances[self.base_asset] >= volume * Decimal('1.001'):
                    order = self.client.create_order(
                        symbol=self.symbol,
                        side=SIDE_SELL,
                        type=ORDER_TYPE_MARKET,
                        quantity=self._round_quantity(volume))
                    
                    logging.info(f"MARKET SELL executed: {volume} @ market price")
                    self._register_order_execution(order, side, level, target_price, volume)
                    return True
                    
        except BinanceAPIException as e:
            logging.error(f"Market order failed: {e.status_code} {e.message}")
            # Добавляем ордер обратно в очередь для следующей попытки
            self.order_queue.put((side, volume, level))
            return False
            
        return False

    def _execute_limit_order(self, side, volume, level, target_price):
        """Попытка исполнения лимитного ордера"""
        try:
            if side == 'BUY':
                quote_needed = volume * target_price * Decimal('1.001')
                if self.balances[self.quote_asset] >= quote_needed:
                    order = self.client.create_order(
                        symbol=self.symbol,
                        side=SIDE_BUY,
                        type=ORDER_TYPE_LIMIT,
                        timeInForce=TIME_IN_FORCE_GTC,
                        quantity=self._round_quantity(volume),
                        price=self._round_price(target_price))
                    
                    logging.info(f"LIMIT BUY order placed: {volume} @ {target_price}")
                    self._register_active_order(order, side, level, target_price, volume)
                    return True
                    
            elif side == 'SELL':
                if self.balances[self.base_asset] >= volume * Decimal('1.001'):
                    order = self.client.create_order(
                        symbol=self.symbol,
                        side=SIDE_SELL,
                        type=ORDER_TYPE_LIMIT,
                        timeInForce=TIME_IN_FORCE_GTC,
                        quantity=self._round_quantity(volume),
                        price=self._round_price(target_price))
                    
                    logging.info(f"LIMIT SELL order placed: {volume} @ {target_price}")
                    self._register_active_order(order, side, level, target_price, volume)
                    return True
                    
        except BinanceAPIException as e:
            logging.error(f"Limit order failed: {e.status_code} {e.message}")
            # Добавляем ордер обратно в очередь для следующей попытки
            self.order_queue.put((side, volume, level))
            return False
            
        return False

    def _emergency_handling(self, side, volume, level, target_price):
        """Экстренная обработка неудачных ордеров"""
        self.emergency_flag = True
        self.emergency_logger.critical(
            f"EMERGENCY: Failed to execute {side} order after multiple attempts. "
            f"Level: {level}, Volume: {volume}, Price: {target_price}"
        )
        
        # Отменяем все связанные ордера
        self._cancel_related_orders(level, side)
        
        # Обновляем балансы
        self.update_balances()
        
        # Логируем текущее состояние
        self._log_full_state()
        
        # Сбрасываем флаг после обработки
        self.emergency_flag = False
        del self.hedge_attempts[f"{side}_{level}"]

    def _cancel_related_orders(self, level, side):
        """Отмена всех связанных ордеров для уровня"""
        with self.order_lock:
            orders_to_cancel = []
            for order_id, order_info in self.active_orders.items():
                if order_info['level'] == level and order_info['side'] == side:
                    orders_to_cancel.append(order_id)
            
            for order_id in orders_to_cancel:
                try:
                    self.client.cancel_order(
                        symbol=self.symbol,
                        orderId=order_id
                    )
                    logging.info(f"Cancelled related order {order_id}")
                    del self.active_orders[order_id]
                except Exception as e:
                    logging.error(f"Error cancelling order {order_id}: {str(e)}")

    def _log_full_state(self):
        """Логирование полного состояния системы"""
        state = {
            'balances': {k: str(v) for k, v in self.balances.items()},
            'grid_levels': self.grid_levels,
            'current_level': self.current_level,
            'active_orders': self.active_orders,
            'emergency_flag': self.emergency_flag
        }
        self.emergency_logger.info(f"Full system state: {state}")

    def _register_active_order(self, order, side, level, price, quantity):
        """Регистрация активного ордера"""
        with self.order_lock:
            self.active_orders[order['orderId']] = {
                'side': side,
                'level': level,
                'price': Decimal(str(price)),
                'quantity': Decimal(str(quantity)),
                'timestamp': time.time()
            }

    def _register_order_execution(self, order, side, level, price, quantity):
        """Регистрация исполненного ордера"""
        # Для рыночных ордеров получаем детали исполнения
        try:
            order_details = self.client.get_order(
                symbol=self.symbol,
                orderId=order['orderId']
            )
            
            executed_qty = Decimal(str(order_details['executedQty']))
            avg_price = Decimal(str(order_details['price']))
            
            logging.info(f"Order {order['orderId']} executed: {executed_qty} @ {avg_price}")
            
            # Обновляем уровень сетки
            if level in self.grid_levels:
                self.grid_levels[level][f"{side.lower()}_executed"] = True
                self.grid_levels[level][f"{side.lower()}_executed_qty"] = executed_qty
                self.grid_levels[level][f"{side.lower()}_avg_price"] = avg_price
            
            # Обновляем балансы
            self.update_balances()
            
        except Exception as e:
            logging.error(f"Error registering order execution: {str(e)}")

    def _validate_price_step(self, price):
        """Проверка соответствия цены минимальному шагу с Decimal"""
        tick_size = self.filters['tick_size']
        price_mod = (price / tick_size) - (price / tick_size).quantize(Decimal('1.'))
        return abs(price_mod) < Decimal('1e-8')

    def _round_price(self, price):
        """Округление цены согласно шагу цены с Decimal"""
        tick_size = self.filters['tick_size']
        return (price / tick_size).quantize(Decimal('1.')) * tick_size

    def _round_quantity(self, quantity):
        """Округление количества согласно шагу объема с Decimal"""
        step_size = self.filters['step_size']
        return (quantity / step_size).quantize(Decimal('1.')) * step_size

    def get_current_price(self):
        """Получение текущей цены с Decimal"""
        try:
            ticker = self.client.get_symbol_ticker(symbol=self.symbol)
            return Decimal(ticker['price'])
        except Exception as e:
            self.emergency_logger.error(f"Error getting price: {str(e)}")
            return Decimal('0')

    def _calculate_level_price(self, level):
        """Расчет единой цены для уровня (и для покупки, и для продажи)"""
        base_price = self.get_current_price()
        # Используем абсолютное значение grid_step
        price = base_price * (Decimal('1') - abs(self.grid_step)) ** level
        return self._round_price(price)

    def check_grid_activation(self):
        """Проверка активации уровней сетки с одинаковыми ценами"""
        current_price = self.get_current_price()
        if current_price <= Decimal('0'):
            return
            
        for level in range(1, self.max_levels+1):
            level_price = self._calculate_level_price(level)
            
            # Для buy-ордера: если цена <= уровня
            if current_price <= level_price and not self._is_order_active_near(level_price):
                self._activate_level(level, 'BUY')
            
            # Для sell-ордера: если цена >= уровня
            if current_price >= level_price and not self._is_order_active_near(level_price):
                self._activate_level(level, 'SELL')

    def _is_order_active_near(self, price):
        """Проверка наличия активного ордера вблизи цены"""
        with self.order_lock:
            tick_size = self.filters['tick_size']
            return any(
                abs(Decimal(str(order_info['price'])) - price < tick_size
                for order_info in self.active_orders.values()
            )

    def _activate_level(self, level, side):
        """Активация уровня сетки с проверкой баланса"""
        if self.current_level >= self.max_levels:
            return

        volume = self._calculate_volume(level, side)
        level_price = self._calculate_level_price(level)  # Используем общую цену
        
        if self._check_balance(side, volume):
            self.order_queue.put((side, volume, level))
            # Сохраняем данные об уровне
            if level not in self.grid_levels:
                self.grid_levels[level] = {
                    'level_price': level_price  # Общая цена для уровня
                }
            # Сохраняем данные конкретно для этого ордера
            self.grid_levels[level].update({
                f"{side.lower()}_active": True,
                f"{side.lower()}_volume": volume
            })
            self.current_level += 1
            self._save_state()

    def _calculate_volume(self, level, side):
        """Расчет объема для уровня с Decimal"""
        base_volume = max(
            self.filters['min_notional'] / self.get_current_price(),
            self.filters['min_qty']
        )
        multiplier = self.trend_multiplier if side == self._get_trend_direction() else self.counter_trend_multiplier
        return self._round_quantity(base_volume * (multiplier ** level))

    def _check_balance(self, side, volume):
        """Проверка баланса с Decimal"""
        try:
            if side == 'BUY':
                required = volume * self.get_current_price() * Decimal('1.001')
                return self.balances[self.quote_asset] >= required
            else:
                return self.balances[self.base_asset] >= volume * Decimal('1.001')
        except Exception as e:
            self.emergency_logger.error(f"Balance check error: {str(e)}")
            return False

    def _get_trend_direction(self):
        """Определение направления тренда"""
        # Здесь можно реализовать более сложную логику
        return 'BUY' if self.grid_step > Decimal('0') else 'SELL'

    def _cleanup_canceled_orders(self):
        """Очистка отмененных ордеров с улучшенным логированием"""
        try:
            canceled_orders = []
            for order_id, order_info in list(self.active_orders.items()):
                try:
                    order_status = self.client.get_order(
                        symbol=self.symbol,
                        orderId=order_id
                    )
                    if order_status['status'] in ['CANCELED', 'REJECTED', 'EXPIRED']:
                        canceled_orders.append(order_id)
                        logging.info(f"Order {order_id} was {order_status['status']}. Reason: {order_status.get('text', 'no reason provided')}")
                except BinanceAPIException as e:
                    if e.code == -2013:  # Order does not exist
                        canceled_orders.append(order_id)
                        logging.info(f"Order {order_id} no longer exists")
                    else:
                        logging.error(f"Error checking order {order_id}: {e.status_code} {e.message}")
            
            with self.order_lock:
                for order_id in canceled_orders:
                    if order_id in self.active_orders:
                        del self.active_orders[order_id]
        except Exception as e:
            self.emergency_logger.error(f"Error cleaning canceled orders: {str(e)}")

    def _sync_active_orders(self):
        """Синхронизация активных ордеров с биржей"""
        try:
            with self.order_lock:
                self.active_orders.clear()
                open_orders = self.client.get_open_orders(symbol=self.symbol)
                for order in open_orders:
                    self.active_orders[order['orderId']] = {
                        'side': order['side'],
                        'price': Decimal(order['price']),
                        'quantity': Decimal(order['origQty']),
                        'level': self._find_level_for_order(Decimal(order['price']), order['side']),
                        'timestamp': time.time()
                    }
                logging.info(f"Synced {len(open_orders)} active orders")
        except Exception as e:
            self.emergency_logger.error(f"Error syncing orders: {str(e)}")

    def _find_level_for_order(self, price, side):
        """Поиск уровня для ордера"""
        for level, data in self.grid_levels.items():
            level_price = data.get('level_price', Decimal('0'))
            if abs(level_price - price) < self.filters['tick_size']:
                return level
        return 0

    def _cleanup_filled_orders(self):
        """Очистка исполненных ордеров с улучшенной обработкой"""
        try:
            current_time = time.time()
            if current_time - self.last_order_check < 60:  # Проверяем не чаще чем раз в минуту
                return
                
            self.last_order_check = current_time
            
            with self.order_lock:
                filled_orders = []
                for order_id, order_info in list(self.active_orders.items()):
                    try:
                        order_status = self.client.get_order(
                            symbol=self.symbol,
                            orderId=order_id
                        )
                        if order_status['status'] == 'FILLED':
                            filled_orders.append(order_id)
                            logging.info(f"Order {order_id} was filled. Executed: {order_status['executedQty']} @ ~{order_status['price']}")
                    except BinanceAPIException as e:
                        if e.code == -2013:  # Order does not exist
                            filled_orders.append(order_id)
                            logging.info(f"Order {order_id} no longer exists (assumed filled)")
                        else:
                            logging.error(f"Error checking order {order_id}: {e.status_code} {e.message}")
                
                for order_id in filled_orders:
                    if order_id in self.active_orders:
                        # Обновляем информацию об исполнении в grid_levels
                        order_info = self.active_orders[order_id]
                        level = order_info['level']
                        side = order_info['side']
                        
                        if level in self.grid_levels:
                            self.grid_levels[level][f"{side.lower()}_executed"] = True
                            self.grid_levels[level][f"{side.lower()}_executed_qty"] = order_info['quantity']
                            self.grid_levels[level][f"{side.lower()}_executed_price"] = order_info['price']
                        
                        del self.active_orders[order_id]
        except Exception as e:
            self.emergency_logger.error(f"Error cleaning filled orders: {str(e)}")

    def _check_profit(self):
        """Проверка достижения тейк-профита"""
        current_price = self.get_current_price()
        if current_price <= Decimal('0'):
            return
            
        for level, data in self.grid_levels.items():
            level_price = data.get('level_price', Decimal('0'))
            
            # Для buy-ордеров: закрываем, если цена выросла на take_profit
            if data.get('buy_active', False) and current_price >= level_price * (Decimal('1') + self.take_profit):
                self._close_level(level, 'BUY')
            
            # Для sell-ордеров: закрываем, если цена упала на take_profit
            if data.get('sell_active', False) and current_price <= level_price * (Decimal('1') - self.take_profit):
                self._close_level(level, 'SELL')

    def _close_level(self, level, side):
        """Закрытие уровня с проверкой баланса"""
        volume = self.grid_levels[level].get(f"{side.lower()}_volume", Decimal('0'))
        if volume > Decimal('0'):
            self.order_queue.put(('SELL' if side == 'BUY' else 'BUY', volume, level))
            self.grid_levels[level][f"{side.lower()}_active"] = False
            self.current_level -= 1
            logging.info(f"Closed {side} level {level}")
            self._save_state()

    def start_trading(self):
        """Запуск торговли с обработкой исключений"""
        logging.info("Starting grid trading bot")
        self.order_executor.start()
        
        try:
            # Синхронизация активных ордеров
            self._sync_active_orders()
            
            while self.is_active:
                try:
                    self.check_grid_activation()
                    self._check_profit()
                    self._cleanup_filled_orders()
                    self._cleanup_canceled_orders()
                    time.sleep(5)
                except Exception as e:
                    logging.error(f"Error in main loop: {str(e)}")
                    time.sleep(10)  # Подождать перед повторной попыткой
                    
        except KeyboardInterrupt:
            logging.info("Received keyboard interrupt, stopping...")
            self.stop_trading()
        except Exception as e:
            self.emergency_logger.critical(f"Critical error in main loop: {str(e)}")
            self.stop_trading()

    def stop_trading(self):
        """Остановка торговли с улучшенной обработкой"""
        self.is_active = False
        logging.info("Stopping grid trading bot")
        
        # Отмена всех активных ордеров
        try:
            with self.order_lock:
                for order_id in list(self.active_orders.keys()):
                    try:
                        self.client.cancel_order(
                            symbol=self.symbol,
                            orderId=order_id
                        )
                        logging.info(f"Cancelled order {order_id}")
                    except Exception as e:
                        logging.error(f"Error cancelling order {order_id}: {str(e)}")
                self.active_orders.clear()
