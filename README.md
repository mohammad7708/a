      
import datetime
import time
import pandas as pd
import MetaTrader5 as mt5
import numpy as np
import statistics
import pytz
import requests
from bs4 import BeautifulSoup
import investpy
import ta
import pandas_ta
import openpyxl
import talib
import math
import telebot
import timedelta

# TOKEN = '7717525197:AAH_Jwqsh-u8-a-N762I-qU5jclKdMzqZQo'

# bot = telebot.TeleBot(TOKEN)


# def send_message_to_channel(message):

#      channel_id = '@fg_signal'
#      bot.send_message(channel_id, message)


# @bot.message_handler(func=lambda message: True)
# def echo_all(message):
  
#      send_message_to_channel(message.text)
 
def total_lot_with_type():
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = (datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference)
    deals = mt5.history_deals_get(start_of_day, mt5_now)
    if deals is None:
        return None
    else:
        return sum(deal.volume for deal in deals)

# Total positions
def total_positons():
    positions_total=mt5.positions_total()
    return positions_total

def total_positons_comment(comments):
    positions = mt5.positions_get()

    if isinstance(comments, str):
        return sum(1 for position in positions if position.comment == comments)

    elif isinstance(comments, list):
        for comment in comments:
            if any(position.comment == comment for position in positions):
                return False
        return True
    
# Total positions
def total_orders():
    orders_total=mt5.orders_get()
    return orders_total

# balance
def balance():
    balance = mt5.account_info()._asdict()['balance']
    return balance

# profit
def profit():
    positions = mt5.positions_get()
    profit = 0
    for position in positions:
        profit += position._asdict()['profit']
    return profit

# kandel
def kandel(timeframe='30m', limit=10 , symbol = 'BTCUSD.'):
       
    symbol = symbol
    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1
    candles = mt5.copy_rates_from_pos(symbol, time, 0, limit)
    df = pd.DataFrame(candles, columns=['time', 'open', 'high', 'low', 'close'])
    return df.iloc

#rsi
def rsi(timeframe,symbol):

    ohlc = kandel(timeframe , 14*10 , symbol)
    candles = pd.DataFrame(ohlc[:])
    candles['rsi'] = ta.momentum.RSIIndicator(candles['close'], window=14).rsi()
    rsi= candles['rsi'].tolist()
    return rsi[-1]

# lot
def qty(myBalance):
    if myBalance < 300:
        lot =  0.01
        return lot
    elif myBalance >= 300 and myBalance <= 499:
        lot =  0.02
        return lot
    elif myBalance >= 500 and myBalance <= 999:
        lot = 0.03
        return lot
    elif myBalance >= 1000 and myBalance <= 1499:
        lot = 0.04
        return lot
    elif myBalance >= 1500 and myBalance <= 1999:
        lot = 0.05
        return lot
    elif myBalance >= 2000 and myBalance <= 2499:
        lot = 0.06
        return lot
    elif myBalance >= 2500 and myBalance <= 2999:
        lot = 0.07
        return lot
    elif myBalance >= 3000 and myBalance <= 3999:
        lot = 0.08
        return lot
    elif myBalance >= 4000 and myBalance <= 5000:
        lot = 0.09
        return lot
    elif myBalance > 5000:
        lot = 0.1
        return lot
    
# create_order
def create_order(symbol , lot , order_type , price , sl , tp , comment):
    symbol_info = mt5.symbol_info(symbol)
    filling_mode = symbol_info.filling_mode
    if filling_mode == 1:
        filling_mode = mt5.ORDER_FILLING_FOK
    elif filling_mode == 2:
        filling_mode = mt5.ORDER_FILLING_IOC
    elif filling_mode != 1 and filling_mode != 2:
        filling_mode = mt5.ORDER_FILLING_FOK or mt5.ORDER_FILLING_IOC

    request={
        "action": mt5.TRADE_ACTION_DEAL,
        "symbol": symbol,
        "volume": lot,
        "type": order_type,
        "price": price,
        "sl" : sl,
        "tp" : tp,
        "comment": comment,
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling":filling_mode,
        }
    order = mt5.order_send(request)
    return order

#close_order
def close_order(symbol , lot , order_type , price , ticket):
    filling_modes = [mt5.ORDER_FILLING_FOK, mt5.ORDER_FILLING_IOC, mt5.ORDER_FILLING_RETURN]

    for filling_mode in filling_modes:
        request = {
            "action": mt5.TRADE_ACTION_DEAL,
            "symbol": symbol,
            "volume": lot,
            "type": order_type,
            "position": ticket,
            "price": price,
            "deviation": 20,
            "magic": 0,
            "comment": "Close position",
            "type_time": mt5.ORDER_TIME_GTC,
            "type_filling": filling_mode,
        }

        result = mt5.order_send(request)

# pending_order
def pending_order(symbol , lot , order_type , price , sl , tp , comment):
    request={
        "action": mt5.TRADE_ACTION_PENDING,
        "symbol": symbol,
        "volume": lot,
        "type": order_type,
        "price": price,
        "sl" : sl,
        "tp" : tp,
        "comment": comment,
        "type_time": mt5.ORDER_TIME_GTC,
        "type_filling": mt5.ORDER_FILLING_IOC,
        }
    order = mt5.order_send(request)
    return order

#remove pending order
def remove_order(symbol , ticket):
    request={
        "action": mt5.TRADE_ACTION_REMOVE,
        "symbol": symbol,
        "order": ticket,
    }
    order = mt5.order_send(request)
    return order

#close_all_positions
def close_all_positions():
    positions = mt5.positions_get()
    if positions is None:
        pass

    for position in positions:

        p_symbol = position._asdict()['symbol']
        p_ticket = position._asdict()['ticket']
        p_lot = position._asdict()['volume']
        if position._asdict()['type'] == 0 :
            #buy
            order_type = mt5.ORDER_TYPE_SELL
            p = mt5.symbol_info_tick(p_symbol).bid
        else:
            order_type = mt5.ORDER_TYPE_BUY
            p = mt5.symbol_info_tick(p_symbol).ask

        close_order(p_symbol , p_lot , order_type , p , p_ticket)

#close_half_positions
def close_half_positions():
    positions = mt5.positions_get()
    if positions is None or len(positions) == 0:
        return

    # نصف تعداد پوزیشن‌ها را محاسبه می‌کنیم
    half_count = len(positions) // 2

    for i, position in enumerate(positions):
        if i >= half_count:
            break
        
        p_symbol = position._asdict()['symbol']
        p_ticket = position._asdict()['ticket']
        p_lot = position._asdict()['volume']
        if position._asdict()['type'] == 0 :
            #buy
            order_type = mt5.ORDER_TYPE_SELL
            p = mt5.symbol_info_tick(p_symbol).bid
        else:
            order_type = mt5.ORDER_TYPE_BUY
            p = mt5.symbol_info_tick(p_symbol).ask

        close_order(p_symbol , p_lot , order_type , p , p_ticket)

def close_half_with_comment(comment):
  
    positions = mt5.positions_get()
    
    positions_with_comment = [pos for pos in positions if pos.comment in comment]
    
    half_count = len(positions_with_comment) // 2

    for i, position in enumerate(positions_with_comment):
        if i >= half_count:
            break
         
        p_symbol = position._asdict()['symbol']
        p_ticket = position._asdict()['ticket']
        p_lot = position._asdict()['volume']
        if position._asdict()['type'] == 0 :
            #buy
            order_type = mt5.ORDER_TYPE_SELL
            p = mt5.symbol_info_tick(p_symbol).bid
        else:
            order_type = mt5.ORDER_TYPE_BUY
            p = mt5.symbol_info_tick(p_symbol).ask

        close_order(p_symbol , p_lot , order_type , p , p_ticket)

def close_all_with_comment(comment):
  
    positions = mt5.positions_get()
    
    positions_with_comment = [pos for pos in positions if pos.comment in comment]
    
    for position in positions_with_comment :
         
        p_symbol = position._asdict()['symbol']
        p_ticket = position._asdict()['ticket']
        p_lot = position._asdict()['volume']
        if position._asdict()['type'] == 0 :
            #buy
            order_type = mt5.ORDER_TYPE_SELL
            p = mt5.symbol_info_tick(p_symbol).bid
        else:
            order_type = mt5.ORDER_TYPE_BUY
            p = mt5.symbol_info_tick(p_symbol).ask

        close_order(p_symbol , p_lot , order_type , p , p_ticket)

# moving average 26
def average26(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=26 , symbol = symbol)
    average = statistics.mean(item['low'] for item in ohlc)
    return average

# moving average 12
def average12(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=12 , symbol = symbol)
    average = statistics.mean(item['close'] for item in ohlc)
    return average

# moving average 50
def average50(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=50 , symbol = symbol)
    average = statistics.mean(item['close'] for item in ohlc)
    return average

# moving average 60
def average60(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=60 , symbol = symbol)
    average = statistics.mean(item['close'] for item in ohlc)
    return average

# moving average 162
def average162(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=162 , symbol = symbol)
    average = statistics.mean(item['close'] for item in ohlc)
    return average

# moving average 100
def average100(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=100 , symbol = symbol)
    average = statistics.mean(item['close'] for item in ohlc)
    return average

# moving average 200
def average200(timeframe , symbol ='BTCUSD.'):
        
    ohlc = kandel(timeframe , limit=200 , symbol = symbol)
    average = statistics.mean(item['close'] for item in ohlc)
    return average

# long or short
def whatKandel(timeframe = '30m' , candle = -1 , symbol ='BTCUSD.'):
    ohlc = kandel(timeframe , limit=10 , symbol = symbol)
    if ohlc[candle]['open'] > ohlc[candle]['close']:
        return 'short'
    else:
        return 'long'
    
def isBeta(timeframe , candel , symbol ='BTCUSD.' , m = 50):
    kandels = kandel(timeframe , limit=10 , symbol = symbol)
    res = kandels[candel]
    if res['open'] > res['close']:
        # short kandel
        if res['open'] == res['high'] and (res['close'] - res['low']) <= res['open'] - res['close'] and body(timeframe , candel ,symbol ) >= m :
            return True
        else:
            return False
    elif res['open'] < res['close']:
        # long kandel
        if res['open'] == res['low']  and (res['high'] - res['close']) <= res['close'] - res['open'] and body(timeframe , candel ,symbol ) >= m :
            return True
        else:
            return False
    else:
        return False
    
def gap(timeframe , symbol ='BTCUSD.'):
    kandels = kandel(timeframe , limit=5 , symbol = symbol)
    #long
    if kandels[-1]['open'] > kandels[-2]['close'] and whatKandel(timeframe , -1 , symbol) == 'long':
        return True
    #short
    if kandels[-1]['open'] < kandels[-2]['close'] and whatKandel(timeframe , -1 , symbol) == 'short':
        return True
    else:
        return False
    
def isBack(timeframe , candel , upOrDown , symbol ='BTCUSD.'):
    kandels = kandel(timeframe , limit=10 , symbol = symbol)
    res = kandels[candel]
    if res['open'] > res['close']:
        # short kandel
        if upOrDown == 'up'and (res['open'] - res['close'])*3 < res['high'] - res['open'] and (res['close'] - res['low']) * 3 <= res['high'] - res['open'] :
            return True
        
        elif upOrDown == 'down'and (res['open'] - res['close'])*4 < res['close'] - res['low'] and (res['high'] - res['open'])*3 <= res['close'] - res['low'] :
            return True
        else:
            return False
        
    elif res['open'] < res['close']:
        # long kandel
        if upOrDown == 'up'and (res['close'] - res['open'])*4 < res['high'] - res['close'] and res['high'] - res['close'] > (res['open'] - res['low'])*3 :
            return True
        
        elif upOrDown == 'down'and (res['close'] - res['open'])*3 < res['open'] - res['low'] and (res['high'] - res['close'])*3 < res['open'] - res['low']:
            return True
        
        else:
            return False
    else:
        return False

def body(timeframe , candel , symbol ='BTCUSD.'):
    kandels = kandel(timeframe , limit=10 , symbol = symbol)
    res = kandels[candel]
    if res['open'] > res['close']:
        # short kandel
        body = res['open'] - res['close']
        return body
        
    elif res['open'] < res['close']:
        # long kandel
        body = res['close'] - res['open']
        return body
    else:
        return 0

def check_time(start_hour, end_hour):
    current_time = datetime.datetime.now(datetime.UTC)
    if current_time.hour >= start_hour and current_time.hour <= end_hour:
        return True
    else:
        return False
    
def check_time_min(start_hour, start_minute, end_hour, end_minute):

    current_time = datetime.datetime.now(datetime.UTC).time()
   
    start_time = datetime.time(start_hour, start_minute)
    end_time = datetime.time(end_hour, end_minute)
    
    if start_time <= current_time <= end_time:
        return True
    else:
        return False

def hemayat(symbol):
    kandeld = kandel('1d' , 5 , symbol )
    kandelw = kandel('1w' , 5 , symbol )
    kande4h = kandel('4h' , 5 , symbol )
    kande1h = kandel('1h' , 5 , symbol )
    lines = [kandeld[-2]['high'] , kandeld[-2]['low'] , kandelw[-1]['high'] , kandelw[-1]['low'] , kandelw[-2]['high'] , kandelw[-2]['low'] , kande4h[-2]['high'] , kande4h[-2]['low'] , kande1h[-2]['high'] , kande1h[-2]['low']]
    price = mt5.symbol_info_tick(symbol).ask
    line = []
    for i in lines:
        if i < price:
            line.append(i)
    if len(line) == 0:
        return False
    else:
        return max(line)
        
def moghavemat(symbol):
    kandeld = kandel('1d' , 5 , symbol )
    kandelw = kandel('1w' , 5 , symbol )
    kande4h = kandel('4h' , 5 , symbol )
    kande1h = kandel('1h' , 5 , symbol )
    lines = [kandeld[-2]['high'] , kandeld[-2]['low'] , kandelw[-1]['high'] , kandelw[-1]['low'] , kandelw[-2]['high'] , kandelw[-2]['low'] , kande4h[-2]['high'] , kande4h[-2]['low'] , kande1h[-2]['high'] , kande1h[-2]['low']]
    price = mt5.symbol_info_tick(symbol).ask
    line = []
    for i in lines:
        if i > price:
            line.append(i)
    if len(line) == 0:
        return False
    else:
        return min(line)
        
def session_hemayat(symbol) :

    def get_session(dt):
        if datetime.time(21, 0) <= dt.time() < datetime.time(5, 59):
            return 'sydney'
        elif datetime.time(0, 0) <= dt.time() < datetime.time(8, 59):
            return 'tokyo'
        elif datetime.time(7, 0) <= dt.time() < datetime.time(15, 59):
            return 'londen'
        elif datetime.time(13, 0) <= dt.time() < datetime.time(20, 59):
            return 'new'
        return None

    timezone = mt5.TIMEFRAME_H1
    date_from = datetime.datetime.now(pytz.UTC) - datetime.timedelta(days=2)
    date_to = datetime.datetime.now(pytz.UTC)

    rates = mt5.copy_rates_range(symbol, timezone, date_from, date_to)
    if rates is None or len(rates) == 0:
        return False

    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s', utc=True)

    df['session'] = df['time'].apply(get_session)
    lines = []

    for session in ['sydney', 'tokyo', 'londen' , 'new']:
        session_data = df[df['session'] == session]
        if not session_data.empty:
            session_high = session_data['high'].max()
            session_low = session_data['low'].min()
            lines.extend([session_high, session_low])

    price = mt5.symbol_info_tick(symbol).ask
    line = []
    for i in lines:
        if i < price:
            line.append(i)
    if len(line) == 0:
        return False
    else:
        return max(line)

def session_moghavemat(symbol) :

    def get_session(dt):
        if datetime.time(21, 0) <= dt.time() < datetime.time(5, 59):
            return 'sydney'
        elif datetime.time(0, 0) <= dt.time() < datetime.time(8, 59):
            return 'tokyo'
        elif datetime.time(7, 0) <= dt.time() < datetime.time(15, 59):
            return 'londen'
        elif datetime.time(13, 0) <= dt.time() < datetime.time(20, 59):
            return 'new'
        return None

    timezone = mt5.TIMEFRAME_H1
    date_from = datetime.datetime.now(pytz.UTC) - datetime.timedelta(days=2)
    date_to = datetime.datetime.now(pytz.UTC)

    rates = mt5.copy_rates_range(symbol, timezone, date_from, date_to)
    if rates is None or len(rates) == 0:
        return False

    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s', utc=True)

    df['session'] = df['time'].apply(get_session)
    lines = []

    for session in ['sydney', 'tokyo', 'londen' , 'new']:
        session_data = df[df['session'] == session]
        if not session_data.empty:
            session_high = session_data['high'].max()
            session_low = session_data['low'].min()
            lines.extend([session_high, session_low])

    price = mt5.symbol_info_tick(symbol).ask
    line = []
    for i in lines:
        if i > price:
            line.append(i)
    if len(line) == 0:
        return False
    else:
        return min(line)

def fvg(symbol , timeframe):
    kandels = kandel(timeframe , limit=10 , symbol = symbol)
    if kandels[-2]['high'] < kandels[-4]['low'] and whatKandel(timeframe , -2 , symbol) == 'short' and whatKandel(timeframe , -3 , symbol) == 'short' :
        return True
    elif kandels[-2]['low'] > kandels[-4]['high'] and whatKandel(timeframe , -2 , symbol) == 'long' and whatKandel(timeframe , -3 , symbol) == 'long' :
        return True
    else:
        return False
    
def sharp(symbol , timeframe):
    kandels = kandel(timeframe , limit=10 , symbol = symbol)
    if fvg(symbol , timeframe) == True and whatKandel(timeframe , -2 , symbol) == 'short':
        return 'short'
    elif fvg(symbol , timeframe) == True and whatKandel(timeframe , -2 , symbol) == 'long':
        return 'long'
    else: 
        return False

def kijun_sen(symbol ,timeframe ,num):
    kandels = kandel(timeframe =timeframe , limit=num , symbol = symbol)
    high = []
    low = []
    i = -1
    for n in range(num) :
        high.append(kandels[i]['high'])
        low.append(kandels[i]['low'])
        i -= 1
    mini = min(low)
    maxi = max(high)
    sen = (maxi + mini ) / 2 
    return sen

def kijun_sen_befor(symbol ,timeframe ,num):
    x = num + 2
    kandels = kandel(timeframe =timeframe , limit=x , symbol = symbol)
    high = []
    low = []
    i = -3
    for n in range(num) :
        high.append(kandels[i]['high'])
        low.append(kandels[i]['low'])
        i -= 1
    mini = min(low)
    maxi = max(high)
    sen = (maxi + mini ) / 2 
    return sen

def ichi_cross(symbol ,timeframe ,num1 , num2):
    if kijun_sen_befor(symbol ,timeframe ,num1) < kijun_sen_befor(symbol ,timeframe ,num2) and kijun_sen(symbol ,timeframe ,num1) >  kijun_sen(symbol ,timeframe ,num2):
        return 'down to up'
    elif kijun_sen_befor(symbol ,timeframe ,num1) > kijun_sen_befor(symbol ,timeframe ,num2) and kijun_sen(symbol ,timeframe ,num1) <  kijun_sen(symbol ,timeframe ,num2):
        return 'up to down'
    else:
        return False
    
def ema20(timeframe,symbol):
    ohlc = kandel(timeframe , 20*10 , symbol = symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ema'] = prices['close'].ewm(span = 20).mean()
    ema = prices['ema'].values.tolist()
    return (ema)[-1]

def ema50(timeframe,symbol):
    ohlc = kandel(timeframe , 50*10 , symbol = symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ema'] = prices['close'].ewm(span = 50).mean()
    ema = prices['ema'].values.tolist()
    return (ema)[-1]

def ema100(timeframe,symbol):
    ohlc = kandel(timeframe , 100*10 , symbol = symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ema'] = prices['close'].ewm(span = 100).mean()
    ema = prices['ema'].values.tolist()
    return (ema)[-1]

def ema200(timeframe,symbol):
    ohlc = kandel(timeframe , 200*10 , symbol = symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ema'] = prices['close'].ewm(span = 200).mean()
    ema = prices['ema'].values.tolist()
    return (ema)[-1]

def ema(timeframe, window , symbol):
    ohlc = kandel(timeframe , window*10 , symbol = symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ema'] = prices['close'].ewm(span = window).mean()
    ema = prices['ema'].values.tolist()
    return (ema)[-1]

def ema_all(timeframe, window , symbol):
    ohlc = kandel(timeframe , window*10 , symbol = symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ema'] = prices['close'].ewm(span = window).mean()
    ema = prices['ema'].values.tolist()
    return ema

#--------------------------------------------------------------------------------
def ema_cross(timeframe, symbol , ema1 , ema2): 
    if \
        ema_all(timeframe , ema1 , symbol)[-2] < ema_all(timeframe , ema2 , symbol)[-2] \
        and ema_all(timeframe , ema1 , symbol)[-1] > ema_all(timeframe , ema2 , symbol)[-1]  :
        
        return "down to up"
    
    elif \
        ema_all(timeframe , ema1 , symbol)[-2] > ema_all(timeframe , ema2 , symbol)[-2] \
        and ema_all(timeframe , ema1 , symbol)[-1] <ema_all(timeframe , ema2 , symbol)[-1] :

        return "up to down"
    
    else : 
            return False
    


def fibo_long(symbol, timeframe, num):
   
    kandels = kandel(timeframe, num, symbol)
    
    high = [kandels[i]['high'] for i in range(num)]
    low = [kandels[i]['low'] for i in range(num)]
    
    high = max(high)
    low = min(low)
    diff = high - low
    
    levels = {
        "0%": low,
        "23.6%": high - 0.236 * diff,
        "38.2%": high - 0.382 * diff,
        "50%": high - 0.5 * diff,
        "61.8%": high - 0.618 * diff,
        "78.6%": high - 0.786 * diff,
        "100%": high
    }
    
    return levels

def fibo_short(symbol, timeframe, num):
   
    kandels = kandel(timeframe, num, symbol)
    
    high = [kandels[i]['high'] for i in range(num)]
    low = [kandels[i]['low'] for i in range(num)]
    
    high = max(high)
    low = min(low)
    diff = high - low
    
    levels = {
        "0%":low ,
        "78.6%": high - 0.236 * diff,
        "61.8%": high - 0.382 * diff,
        "50%": high - 0.5 * diff,
        "38.2%": high - 0.618 * diff,
        "23.6%": high - 0.786 * diff,
        "100%": high
    }
    
    return levels

def order_book(symbol, limit=100 , volume=10 , bidsOrasks = 'bids'):
    url = f'https://api.binance.com/api/v3/depth?symbol={symbol}&limit={limit}'
    response = requests.get(url)
    
    if response.status_code == 200:
        data = response.json()
        bids = [[float(order[0]), float(order[1])] for order in data['bids']]
        asks = [[float(order[0]), float(order[1])] for order in data['asks']]

        def filter_large_orders(orders, volume):
            return [order for order in orders if order[1] >= volume]
        
        large_bids = filter_large_orders(bids, volume)
        large_asks = filter_large_orders(asks, volume)

        if bidsOrasks == 'bids' and large_bids != []:
            return max(large_bids)[0]
        elif bidsOrasks == 'asks' and large_asks != []:
            return min(large_asks)[0]
        else:
            pass
    else:
        
        return None

def order_book_signal(symbol):
    limit = 2000
    url = f'https://api.binance.com/api/v3/depth?symbol={symbol}&limit={limit}'
    response = requests.get(url)
    
    if response.status_code == 200:
        data = response.json()
        bids = data['bids']
        asks = data['asks']
        bids_vol = []
        asks_vol = []
        for bid in bids :
            bids_vol.append(float(bid[1]))
           
        for ask in asks :
            asks_vol.append(float(ask[1]))
           
        if sum(asks_vol) > sum(bids_vol):
            return 'short'
        elif sum(bids_vol) > sum(asks_vol):
            return 'long'
        else:
            return None
    else:
        
        return None

#Bollinger Band Algorithm
def BB(timeframe, window , symbol, num_std_dev=2):
    
    ohlc = kandel(timeframe , limit=window , symbol = symbol)  
    
    
    SMA = statistics.mean(item['low'] for item in ohlc)
 
    SD = statistics.stdev(item['low'] for item in ohlc)
   
    UB = SMA + (num_std_dev * SD)
   
    LB = SMA - (num_std_dev * SD)
    
    
    return [SMA,UB,LB]

def time_high_low(symbol, start_hour, start_minute, timeframe_str, num_candles, time_offset_minutes=0):

    today = datetime.datetime.now(datetime.timezone.utc).date()

    start_time = datetime.datetime(today.year, today.month, today.day, start_hour, start_minute, 0, tzinfo=datetime.timezone.utc)
    start_time = start_time + datetime.timedelta(minutes=time_offset_minutes)

    def parse_timeframe(timeframe_str):
        tf_map = {
            '1m': (mt5.TIMEFRAME_M1, 1),
            '3m': (mt5.TIMEFRAME_M3, 3),
            '5m': (mt5.TIMEFRAME_M5, 5),
            '15m': (mt5.TIMEFRAME_M15, 15),
            '30m': (mt5.TIMEFRAME_M30, 30),
            '1h': (mt5.TIMEFRAME_H1, 60),
            '4h': (mt5.TIMEFRAME_H4, 240),
            '1d': (mt5.TIMEFRAME_D1, 1440),
            '1w': (mt5.TIMEFRAME_W1, 10080),
            '1mn': (mt5.TIMEFRAME_MN1, 43200)
        }

        return tf_map.get(timeframe_str.lower(), (None, None))

    timeframe, timeframe_minutes = parse_timeframe(timeframe_str)

    end_time = start_time + datetime.timedelta(minutes=timeframe_minutes * num_candles)

    rates = mt5.copy_rates_range(symbol, timeframe, start_time, end_time)

    if rates is None or len(rates) == 0:
        return False

    data = pd.DataFrame(rates)
    data['time'] = pd.to_datetime(data['time'], unit='s')
    data = data.sort_values(by='time')

    selected_candles = data.head(num_candles)

    highest_point = selected_candles['high'].max()
    lowest_point = selected_candles['low'].min()


    return {
        'high': highest_point,
        'low': lowest_point
    }

def modify_position(ticket, new_stop_loss):

    position = mt5.positions_get(ticket=ticket)
    if not position:
        return False
    
    position = position[0]

    request = {
        "action": mt5.TRADE_ACTION_SLTP,
        "position": position.ticket,
        "sl": new_stop_loss,
        "tp": position.tp,
        "symbol": position.symbol,
        "type": position.type,
        "volume": position.volume,
    }

    result = mt5.order_send(request)

def modify_position2(ticket, new_tp):

    position = mt5.positions_get(ticket=ticket)
    if not position:
        return False
    
    position = position[0]

    request = {
        "action": mt5.TRADE_ACTION_SLTP,
        "position": position.ticket,
        "sl": position.sl,
        "tp": new_tp,
        "symbol": position.symbol,
        "type": position.type,
        "volume": position.volume,
    }

    result = mt5.order_send(request)

def sar(symbol, timeframe, step=0.02, max_step=0.2):
    ohlc = kandel(timeframe, 50, symbol)
    data = pd.DataFrame(ohlc[:])

    high = data['high']
    low = data['low']

    sar = talib.SAR(high, low, acceleration=step, maximum=max_step)

    return sar.tolist()

def cci (symbol , timeframe , period = 20): 

    ohlc = kandel(timeframe , period*10 , symbol)
    candles = pd.DataFrame(ohlc[:])
    candles['cci'] = ta.trend.CCIIndicator(high=candles['high'], low=candles['low'], close=candles['close'], window=period).cci()
    cci= candles['cci'].tolist()
    return cci

def atr(symbol , timeframe , period = 14) : 
    
    ohlc = kandel(timeframe , period*5 , symbol)
    candles = pd.DataFrame(ohlc[:])
    candles['H-L'] = abs(candles['high'] - candles['low'])
    candles['H-PC'] = abs(candles['high'] - candles['close'].shift(1))
    candles['L-PC'] = abs(candles['low'] - candles['close'].shift(1))
    candles['TR'] = candles[['H-L' , 'H-PC' , 'L-PC']].max(axis=1 , skipna=False)
    candles['ATR'] = candles['TR'].rolling(period).mean()
    atr = candles['ATR'].tolist()
    return atr

def mfi (symbol ,timeframe , period = 14) :

    ohlc = kandel(timeframe , period*5 , symbol)
    candles = pd.DataFrame(ohlc[:]) 
    candles['mfi'] = ta.volume.MFIIndicator(high=candles['high'], low=candles['low'], close=candles['close'], volume=candles['tick_volume'], window=period).money_flow_index()
    mfi = candles['mfi'].tolist()
    return mfi

def smma(symbol , timeframe , period):

    ohlc = kandel(timeframe , period*10 , symbol)
    candles = ohlc[:]


    smma = [np.nan] * (period - 1)
    initial_smma = candles['close'][:period].mean()
    smma.append(initial_smma)
    
    for price in candles['close'][period:]:
        smma_value = (smma[-1] * (period - 1) + price) / period
        smma.append(smma_value)
    return smma

def wma(symbol , timeframe , period) : 
    ohlc = kandel(timeframe , period*10 , symbol)
    candles = ohlc[:]

    weights = np.arange(1, period + 1)
    candles['wma'] = candles['close'].rolling(window = period).apply(lambda prices: np.dot(prices, weights)/weights.sum(), raw=True)
    wma = candles['wma'].tolist()

    return wma

def macd (symbol , timeframe , line ,short_period = 12 , long_period = 26 , signal_period = 9 ) : 
    ohlc = kandel(timeframe , long_period * 10 , symbol)
    candles = ohlc[:]

    candles['EMA_short'] = candles['close'].ewm(span=short_period, adjust=False).mean()
    candles['EMA_long'] = candles['close'].ewm(span=long_period, adjust=False).mean()
    candles['MACD'] = candles['EMA_short'] - candles['EMA_long']
    candles['Signal'] = candles['MACD'].rolling(window=signal_period).mean()
    candles['Histogram'] = candles['MACD'] - candles['Signal']

    
    macd = candles['MACD'].tolist()
    signall = candles['Signal'].tolist()
    histogram = candles['Histogram'].tolist()

    if line == 'macd' :
        return macd
    
    if line == 'signal' : 
        return signall
    
    if line == 'histogram' : 
        return histogram

def dema(symbol , timeframe , period = 14):
    ohlc = kandel(timeframe , period*5 , symbol)
    candles = ohlc[:]

    candles['ema1'] = candles['close'].ewm(span=period, adjust=False).mean()
    candles['ema2'] = candles['ema1'].ewm(span=period, adjust=False).mean()
    candles['dema'] = 2 * candles['ema1'] - candles['ema2']
    dema = candles['dema'].tolist()

    return dema

def tema(symbol , timeframe , period = 14) : 

    ohlc = kandel(timeframe , period * 10 , symbol)
    candles = ohlc[:]

    candles['ema1'] = candles['close'].ewm(span=period, adjust=False).mean()
    candles['ema2'] = candles['ema1'].ewm(span=period, adjust=False).mean()
    candles['ema3'] = candles['ema2'].ewm(span=period, adjust=False).mean()
    candles['tema'] = (3 * (candles['ema1'] - candles['ema2'])) + candles['ema3']
    tema = candles['tema'].tolist()

    return tema

def donchain_channel(symbol , timeframe ,line ,  period = 20) : 

    ohlc = kandel(timeframe , period*2 , symbol)
    candles = pd.DataFrame(ohlc[:])

    candles['upper'] = candles['high'].rolling(window=period).max()
    candles['lower'] = candles['low'].rolling(window=period).min()
    candles['middle'] = (candles['upper'] + candles['lower']) / 2

    upper = candles['upper'].shift(+1).tolist()
    lower = candles['lower'].shift(+1).tolist()
    middle = candles['middle'].shift(+1).tolist()

    if line == 'upper' : 
        return upper
    
    if line == 'lower' : 
        return lower

    if line == 'middle' : 
        return middle

def swing(symbol , timeframe , swing , window = 5 , lookback = 250)  : 

    ohlc = kandel(timeframe , lookback , symbol)
    candles = pd.DataFrame(ohlc[:])


    high_rolling_max = candles['high'].rolling(window=window, center=True).max()
    low_rolling_min = candles['low'].rolling(window=window, center=True).min()

    candles['swing_high'] = candles['high'][(candles['high'] == high_rolling_max) & (candles['high'].shift(window) < candles['high']) & (candles['high'].shift(-window) < candles['high'])]
    candles['swing_low'] = candles['low'][(candles['low'] == low_rolling_min) & (candles['low'].shift(window) > candles['low']) & (candles['low'].shift(-window) > candles['low'])]
    
    untouched_swing_highs = []
    untouched_swing_lows = []
    
    for i in range(len(candles)):
        if pd.notna(candles['swing_high'].iloc[i]):
            swing_high_touched = any(candles['high'].iloc[i+1:].ge(candles['swing_high'].iloc[i]))
            if not swing_high_touched:
                untouched_swing_highs.append(candles['swing_high'].iloc[i])
        
        if pd.notna(candles['swing_low'].iloc[i]):
            swing_low_touched = any(candles['low'].iloc[i+1:].le(candles['swing_low'].iloc[i]))
            if not swing_low_touched:
                untouched_swing_lows.append( candles['swing_low'].iloc[i])
    
    if swing == "high" : 
        return untouched_swing_highs

    if swing == "low" : 
        return untouched_swing_lows
    
def Avrage(symbol , timeframe, window) : 
    ohlc = kandel(timeframe, window*2 , symbol)
    prices = pd.DataFrame(ohlc[:])
    prices['ma'] = prices['close'].rolling(window= window).mean()
    ma = prices['ma'].values.tolist()
    return ma

def ravand(symbol, timeframe):

    if timeframe == '1m':
        i = '5m'
    elif timeframe == '3m':
        i = '15m'
    elif timeframe == '5m':
        i = '15m'
    elif timeframe == '15m':
        i = '30m'
    elif timeframe == '30m':
        i = '1h'
    elif timeframe == '1h':
        i = '4h'
    elif timeframe == '4h':
        i = '1d'
    elif timeframe == '1d':
        i = '1w'
    else:
        return False
    
    price = mt5.symbol_info_tick(symbol).ask
    sar_values = sar(symbol, i)
    if sar_values[-1] < price :
        return 'long'
    else :
        return 'short'

def sar_signal(symbol, timeframe, step=0.02, max_step=0.2 , increment=0.01):

    sar_values = sar(symbol, timeframe, step, max_step)
    kandels = kandel(timeframe, 10, symbol)
    #sell signal 
    if kandels[-2]['open'] > sar_values[-2] and kandels[-1]['open'] < sar_values[-1] :
        return 'short'
    #long signal
    elif kandels[-2]['open'] < sar_values[-2] and kandels[-1]['open'] > sar_values[-1] :
        return 'long'
    else:
        return False

def keltner_channel(symbol , timeframe ,line ,  period = 20 , atr = 10 , multiplier = 2) : 

    ohlc = kandel(timeframe , period*10 , symbol)
    candles = pd.DataFrame(ohlc[:])

    candles['EMA'] = candles['close'].ewm(span=period, adjust=False).mean()
    candles['TR'] = np.maximum(candles['high'] - candles['low'], 
                          np.maximum(abs(candles['high'] - candles['close'].shift(1)),
                                     abs(candles['low'] - candles['close'].shift(1))))
    candles['ATR'] = candles['TR'].rolling(window=atr).mean()
    candles['Upper_Channel'] = (candles['EMA'] + (multiplier * candles['ATR'])).shift(1)
    candles['Lower_Channel'] = (candles['EMA'] - (multiplier * candles['ATR'])).shift(1)

    if line == "up" : 
        return candles['Upper_Channel'].tolist()
    
    if line == 'mid' : 
        return candles['EMA'].shift(1).tolist()
    
    if line == 'low' :
        return candles['Lower_Channel'].tolist()

def line(symbol, timeframe, upOrdown):
    kandels = kandel(timeframe, 5, symbol)
    leg = (kandels[-2]['high'] - kandels[-2]['low']) / 8
    candle_color = whatKandel(timeframe, -1, symbol)
    lines = []

    if candle_color == 'long':
        num1 = kandels[-1]['low']
        lines.append(kandels[-1]['low'])
        for _ in range(50):
            num1 += leg
            lines.append(num1)
    else:
        num1 = kandels[-1]['high']
        lines.append(kandels[-1]['high'])
        for _ in range(50):
            num1 -= leg
            lines.append(num1)

    x = []
    if upOrdown == 'up':
        price = mt5.symbol_info_tick(symbol).bid
        for i in lines:
            if i > price:
                x.append(i)

        if len(x) == 0:
            return False
        else:
            return min(x)
    else:
        price = mt5.symbol_info_tick(symbol).ask
        for i in lines:
            if i < price:
                x.append(i)

        if len(x) == 0:
            return False
        else:
            return max(x)

def trend_alert(long_term='1d', mid_term='4h', symbol='BTCUSD.'):

    def candle(timeframe='30m', limit=500, symbol='BTCUSD'):
        timeframes = {
            '1m': mt5.TIMEFRAME_M1,
            '3m': mt5.TIMEFRAME_M3,
            '5m': mt5.TIMEFRAME_M5,
            '15m': mt5.TIMEFRAME_M15,
            '30m': mt5.TIMEFRAME_M30,
            '1h': mt5.TIMEFRAME_H1,
            '4h': mt5.TIMEFRAME_H4,
            '1d': mt5.TIMEFRAME_D1,
            '1w': mt5.TIMEFRAME_W1,
            '1mn': mt5.TIMEFRAME_MN1
        }
        
        time = timeframes.get(timeframe, mt5.TIMEFRAME_M30)  # Default to '30m' if not found
        candles = mt5.copy_rates_from_pos(symbol, time, 0, limit)
        
        if candles is None or len(candles) == 0:
            print(f"No data received for {symbol} on {timeframe}")
            return pd.DataFrame()  # Return empty DataFrame
        
        df = pd.DataFrame(candles, columns=['time', 'open', 'high', 'low', 'close'])
        df['time'] = pd.to_datetime(df['time'], unit='s')
        return df

    def heiken_ashi(df):
        if len(df) < 2:
            return False
        
        ha_close = (df['open'] + df['high'] + df['low'] + df['close']) / 4.0
        ha_open = pd.Series(index=df.index, data=np.nan)
        ha_open.iloc[0] = (df['open'].iloc[0] + df['close'].iloc[0]) / 2.0
        
        for i in range(1, len(df)):
            ha_open.iloc[i] = (ha_open.iloc[i-1] + ha_close.iloc[i-1]) / 2.0
        
        ha_high = np.maximum(df['high'], np.maximum(ha_open, ha_close))
        ha_low = np.minimum(df['low'], np.minimum(ha_open, ha_close))
        
        ha_df = pd.DataFrame({
            'open': ha_open,
            'high': ha_high,
            'low': ha_low,
            'close': ha_close
        })
        return ha_df


    lt_df = candle(timeframe=long_term, limit=500, symbol=symbol)
    mt_df = candle(timeframe=mid_term, limit=500, symbol=symbol)

    if len(lt_df) < 2 or len(mt_df) < 2:
        return False

    lt_ha = heiken_ashi(lt_df)
    mt_ha = heiken_ashi(mt_df)
    mt_df['ema20'] = mt_df['close'].ewm(span=20, adjust=False).mean()
    lt_trend = lt_ha['close'] > lt_ha['open']
    mt_trend = (mt_ha['close'] > mt_ha['open']) & (mt_ha['close'] > mt_df['ema20']) & (mt_df['ema20'].diff() > 0)

    long_signal = lt_trend.iloc[-1] and mt_trend.iloc[-1]
    short_signal = not lt_trend.iloc[-1] and not mt_trend.iloc[-1]

    if long_signal:
        return 'long'
    elif short_signal:
        return 'short'
    else:
        return False

def nadaraya_signals(symbol, timeframe, mult=2.0, h=8.0):

    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1

    def gauss(x, h):
        return np.exp(-(x**2) / (2 * h**2))

    def nadaraya_watson(src, h):
        n = len(src)
        out = np.zeros(n)
        for i in range(n):
            weights = gauss(np.arange(i+1)[::-1], h)
            out[i] = np.sum(src[:i+1] * weights) / np.sum(weights)
        return out

    rates = mt5.copy_rates_from_pos(symbol, time, 0, 500)
    if rates is None:
        print("Failed to get rates, error code =", mt5.last_error())
        return []

    
    data = pd.DataFrame(rates)
    data['time'] = pd.to_datetime(data['time'], unit='s')
    data.set_index('time', inplace=True)

  
    src = data['close'].values
    out = nadaraya_watson(src, h)
    mae = np.mean(np.abs(src - out)) * mult
    upper = out + mae
    lower = out - mae


    signals = []
    for i in range(1, len(src)):
        if src[i] > upper[i-1] and src[i-1] <= upper[i-1]:
            signals.append(('short'))
        elif src[i] < lower[i-1] and src[i-1] >= lower[i-1]:
            signals.append(('long'))

    return signals

def signal(symbol , timeframe) :

    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1
    num_bars = 1000

    rates = mt5.copy_rates_from_pos(symbol, time, 0, num_bars)

    df = pd.DataFrame(rates)


    df['close'] = df['close'].astype(float)
    df['high'] = df['high'].astype(float)
    df['low'] = df['low'].astype(float)
    df['open'] = df['open'].astype(float)

    df['time'] = pd.to_datetime(df['time'], unit='s')
    df.set_index('time', inplace=True)

    df['ema_9'] = df['close'].ewm(span=9, adjust=False).mean()
    df['ema_12'] = df['close'].ewm(span=12, adjust=False).mean()
    df['sma_5'] = df['close'].rolling(window=5).mean()
    df['sma_9'] = df['close'].rolling(window=9).mean()

    df['signal'] = 0

    df.loc[df['ema_9'] > df['ema_12'], 'signal'] = 1  
    df.loc[df['ema_9'] < df['ema_12'], 'signal'] = -1  


    last_signal = df.iloc[-1] 

    if last_signal['signal'] == 1:
        return("long")
        
    elif last_signal['signal'] == -1:
        return("short")
        
    else:
        return "هیچ سیگنالی صادر نشده است."

def nadaraya_lower(symbol, timeframe, mult=2.0, h=8.0):

    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1

    def gauss(x, h):
        return np.exp(-(x**2) / (2 * h**2))

    def nadaraya_watson(src, h):
        n = len(src)
        out = np.zeros(n)
        for i in range(n):
            weights = gauss(np.arange(i+1)[::-1], h)
            out[i] = np.sum(src[:i+1] * weights) / np.sum(weights)
        return out

    rates = mt5.copy_rates_from_pos(symbol, time, 0, 500)
    if rates is None:
        print("Failed to get rates, error code =", mt5.last_error())
        return []

    
    data = pd.DataFrame(rates)
    data['time'] = pd.to_datetime(data['time'], unit='s')
    data.set_index('time', inplace=True)

  
    src = data['close'].values
    out = nadaraya_watson(src, h)
    mae = np.mean(np.abs(src - out)) * mult
    upper = out + mae
    lower = out - mae



    return lower

def nadaraya(symbol, timeframe, mult=2.0, h=8.0 , updown='up'):

    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1

    def gauss(x, h):
        return np.exp(-(x**2) / (2 * h**2))

    def nadaraya_watson(src, h):
        n = len(src)
        out = np.zeros(n)
        for i in range(n):
            weights = gauss(np.arange(i+1)[::-1], h)
            out[i] = np.sum(src[:i+1] * weights) / np.sum(weights)
        return out

    rates = mt5.copy_rates_from_pos(symbol, time, 0, 500)
    if rates is None:
        print("Failed to get rates, error code =", mt5.last_error())
        return []

    
    data = pd.DataFrame(rates)
    data['time'] = pd.to_datetime(data['time'], unit='s')
    data.set_index('time', inplace=True)

  
    src = data['close'].values
    out = nadaraya_watson(src, h)
    mae = np.mean(np.abs(src - out)) * mult
    upper = out + mae
    lower = out - mae
    if updown == 'up':
        return upper
    else :
        return lower

def ravand_signal(symbol, timeframe):

    uper = nadaraya(symbol, timeframe, mult=2.5, h=4.0 , updown='up')
    down = nadaraya(symbol, timeframe, mult=2.5, h=4.0 , updown='down')
    Avrage18 = Avrage(symbol ,timeframe,18)[-1]
    if down[-1] > Avrage18 and down[-2]>= Avrage18 and down[-3] <= Avrage18 and uper[-1] > Avrage18 :
        return 'long'
    elif uper[-1] < Avrage18 and uper[-2]<= Avrage18 and uper[-3] >= Avrage18 and down[-1] < Avrage18 :
        return 'short'
    elif  uper[-1] > Avrage18 and uper[-2] > Avrage18 and uper[-3] > Avrage18  and uper[-4] > Avrage18 and uper[-5] > Avrage18 and down[-1] < Avrage18 and  down[-2] < Avrage18 and down[-3] < Avrage18  and down[-4] < Avrage18  and down[-4] < Avrage18:
        return 'reng'
    else :
        return False
    
def nadaraya_signals2(symbol, timeframe, mult=2.0, h=8.0):
    ask_price = mt5.symbol_info_tick(symbol).ask
    bid_price = mt5.symbol_info_tick(symbol).bid
    Avrage18 = Avrage(symbol , timeframe , 18)[-1]
    if ravand_signal(symbol, timeframe) == 'reng' and bid_price > Avrage18:
        return 'short' 
    elif ravand_signal(symbol, timeframe) == 'reng' and ask_price < Avrage18:
        return 'long' 

def ott_signal(symbol ,timeframe, length=1, percent=1.4):

    ohlc = kandel(timeframe , 100 , symbol)
    candles = ohlc[:]
    src = candles['close']
    valpha = 2 / (length + 1)

    vud1 = pd.Series(np.where(src > src.shift(1), src - src.shift(1), 0))
    vdd1 = pd.Series(np.where(src < src.shift(1), src.shift(1) - src, 0))
    vUD = vud1.rolling(window=9).sum()
    vDD = vdd1.rolling(window=9).sum()
    vCMO = (vUD - vDD) / (vUD + vDD)
    vCMO = vCMO.fillna(0)  # Replace NaNs with 0

    VAR = np.zeros(len(src))
    for i in range(1, len(src)):
        VAR[i] = valpha * abs(vCMO[i]) * src[i] + (1 - valpha * abs(vCMO[i])) * VAR[i-1]

    MAvg = VAR
    fark = MAvg * percent * 0.01
    longStop = MAvg - fark
    longStopPrev = pd.Series(longStop).shift(1).fillna(longStop[0])

    longStop = np.where(MAvg > longStopPrev, np.maximum(longStop, longStopPrev), longStop)
    shortStop = MAvg + fark
    shortStopPrev = pd.Series(shortStop).shift(1).fillna(shortStop[0])

    shortStop = np.where(MAvg < shortStopPrev, np.minimum(shortStop, shortStopPrev), shortStop)
    dir = np.ones(len(src))
    dir[0] = 1

    for i in range(1, len(src)):
        if dir[i-1] == -1 and MAvg[i] > shortStopPrev[i]:
            dir[i] = 1
        elif dir[i-1] == 1 and MAvg[i] < longStopPrev[i]:
            dir[i] = -1
        else:
            dir[i] = dir[i-1]

    MT = np.where(dir == 1, longStop, shortStop)
    OTT = np.where(MAvg > MT, MT * (200 + percent) / 200, MT * (200 - percent) / 200)
    candles['OTT'] = OTT

    otts = candles['OTT'].tolist()
    if round(otts[-10]) == round(otts[-5]) and round(otts[-5]) == round(otts[-1]) :
        return 'reng'
    elif round(otts[-1]) < round(otts[-5]):
        return 'short'
    elif round(otts[-1]) > round(otts[-5]):
        return 'long'
    else:
        return False

def supertrend(symbol , timeframe , atr_period=9, multiplier=3.9, change_atr=True):
    ohlc = kandel(timeframe, atr_period*10 , symbol)
    df = ohlc[:]
    df['hl2'] = (df['high'] + df['low']) /2
    
    df['TR'] = np.maximum(df['high'] - df['low'], 
                          np.maximum(abs(df['high'] - df['close'].shift(1)), 
                                     abs(df['low'] - df['close'].shift(1))))
    

    if change_atr:
        df['ATR'] = df['TR'].rolling(window=atr_period).mean()
    else:
        df['ATR'] = df['TR'].ewm(span=atr_period, adjust=False).mean()


    df['UpperBand'] = df['hl2'] - (multiplier * df['ATR'])
    df['LowerBand'] = df['hl2'] + (multiplier * df['ATR'])


    df['Supertrend'] = np.nan
    df['Trend'] = 1
    
    for i in range(1, len(df)):
        if df['close'].iloc[i - 1] > df.at[i - 1, 'UpperBand']:
            df.at[i, 'UpperBand'] = max(df.at[i, 'UpperBand'], df.at[i - 1, 'UpperBand'])
        else:
            df.at[i, 'UpperBand'] = df.at[i, 'UpperBand']

        if df['close'].iloc[i - 1] < df.at[i - 1, 'LowerBand']:
            df.at[i, 'LowerBand'] = min(df.at[i, 'LowerBand'], df.at[i - 1, 'LowerBand'])
        else:
            df.at[i, 'LowerBand'] = df.at[i, 'LowerBand']


        if df.at[i - 1, 'Trend'] == 1:
            df.at[i, 'Trend'] = -1 if df.at[i, 'close'] < df.at[i - 1, 'UpperBand'] else 1
        else:
            df.at[i, 'Trend'] = 1 if df.at[i, 'close'] > df.at[i - 1, 'LowerBand'] else -1


        df.at[i, 'Supertrend'] = df.at[i, 'UpperBand'] if df.at[i, 'Trend'] == 1 else df.at[i, 'LowerBand']
        df['Supertrend_Position'] = np.where(df['Supertrend'] > df['close'], 'short', 'long')

        supertrend = []
        for _, row in df.iterrows():
            row_dict = {'value': row['Supertrend'], 'position': row['Supertrend_Position']}
            supertrend.append(row_dict)
        
    return supertrend

def supertrend_signal(symbol , timeframe , atr_period=9, multiplier=3.9, change_atr=True):
    SuperT = supertrend(symbol , timeframe , atr_period=9, multiplier=3.9, change_atr=True)
    if SuperT[-2]['position'] == 'long' and SuperT[-3]['position']=='short' :
        return 'long'
    elif SuperT[-2]['position'] =='short' and SuperT[-3]['position'] =='long' :
        return 'short'
    else : 
        return False

def supertrend_hi(symbol, timeframe, atr_period=9, multiplier=3.9, change_atr=True, source='hl2', ema_value=200):
    ohlc = candle(timeframe, atr_period * 10, symbol)  # گرفتن داده‌های کندل
    ha_df = heiken_ashi(ohlc)  # تبدیل داده‌ها به کندل‌های Heiken Ashi

    df = ha_df.copy()
    
    # محاسبه EMA با مقدار مشخص‌شده توسط کاربر
    df['ema'] = df['close'].ewm(span=ema_value, adjust=False).mean()

    # محاسبه hl2
    df['hl2'] = (df['high'] + df['low']) / 2
    
    # انتخاب منبع (hl2 یا ema)
    if source == 'hl2':
        df['selected_source'] = df['hl2']
    elif source == 'ema':
        df['selected_source'] = df['ema']
    else:
        raise ValueError("Source must be either 'hl2' or 'ema'")

    df['TR'] = np.maximum(df['high'] - df['low'],
                          np.maximum(abs(df['high'] - df['close'].shift(1)),
                                     abs(df['low'] - df['close'].shift(1))))

    if change_atr:
        df['ATR'] = df['TR'].rolling(window=atr_period).mean()
    else:
        df['ATR'] = df['TR'].ewm(span=atr_period, adjust=False).mean()

    # استفاده از منبع انتخاب‌شده برای محاسبه UpperBand و LowerBand
    df['UpperBand'] = df['selected_source'] - (multiplier * df['ATR'])
    df['LowerBand'] = df['selected_source'] + (multiplier * df['ATR'])

    df['Supertrend'] = np.nan
    df['Trend'] = 1

    for i in range(1, len(df)):
        if df['close'].iloc[i - 1] > df.at[i - 1, 'UpperBand']:
            df.at[i, 'UpperBand'] = max(df.at[i, 'UpperBand'], df.at[i - 1, 'UpperBand'])
        else:
            df.at[i, 'UpperBand'] = df.at[i, 'UpperBand']

        if df['close'].iloc[i - 1] < df.at[i - 1, 'LowerBand']:
            df.at[i, 'LowerBand'] = min(df.at[i, 'LowerBand'], df.at[i - 1, 'LowerBand'])
        else:
            df.at[i, 'LowerBand'] = df.at[i, 'LowerBand']

        if df.at[i - 1, 'Trend'] == 1:
            df.at[i, 'Trend'] = -1 if df.at[i, 'close'] < df.at[i - 1, 'UpperBand'] else 1
        else:
            df.at[i, 'Trend'] = 1 if df.at[i, 'close'] > df.at[i - 1, 'LowerBand'] else -1

        df.at[i, 'Supertrend'] = df.at[i, 'UpperBand'] if df.at[i, 'Trend'] == 1 else df.at[i, 'LowerBand']

    df['Supertrend_Position'] = np.where(df['Supertrend'] > df['close'], 'Above', 'Below')

    supertrend_hi = []
    for _, row in df.iterrows():
        row_dict = {'value': row['Supertrend'], 'position': row['Supertrend_Position']}
        supertrend_hi.append(row_dict)

    return supertrend_hi

def SMA_RSI(timeframe , symbol , period):
    ohlc = kandel(timeframe, period*10, symbol)  
    
    candles = pd.DataFrame(ohlc[:], columns=['time', 'open', 'high', 'low', 'close', 'tick_volume', 'spread', 'real_volume'])

    candles['rsi'] = ta.momentum.RSIIndicator(candles['close'], window=period).rsi()
    
    valid_rsi = candles['rsi'].dropna()
    
    average_rsi = statistics.mean(valid_rsi)
    
    return average_rsi

def cross_macd(symbol , timeframe ,short_period = 12 , long_period = 26 , signal_period = 9 ): 
    mcd = macd(symbol , timeframe , 'macd' ,short_period = 12 , long_period = 26 , signal_period = 9 ) 
    signals = macd(symbol , timeframe , 'signal' ,short_period = 12 , long_period = 26 , signal_period = 9 ) 
    if  mcd[-1]> signals[-1] and mcd[-2]>=signals[-2] and mcd[-3]<signals[-3] and mcd[-4]<signals[-2] and mcd[-5]<signals[-2]: 
        return 'long' 
    elif mcd[-1]< signals[-1] and mcd[-2]<=signals[-2] and mcd[-3]>signals[-3] and mcd[-4]>signals[-2] and mcd[-5]>signals[-2]: 
        return 'short' 
    else: 
        return False

def count_sl():
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference
    orders = mt5.history_deals_get(start_of_day, mt5_now)
    stop_count = sum(1 for order in orders if order.profit < 0 )
    return stop_count

def count_tp():
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference
    orders = mt5.history_deals_get(start_of_day, mt5_now)
    profit_count = sum(1 for order in orders if order.profit > 0)
    return profit_count

def profit_today():
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference
    orders = mt5.history_deals_get(start_of_day, mt5_now)
    profit_today = sum(order.profit for order in orders)    
    return profit_today

def count_sl_in_hours(hours=0):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_time = mt5_now - datetime.timedelta(hours=hours)
    orders = mt5.history_deals_get(start_time, mt5_now)
    stop_count = sum(1 for order in orders if order.profit < 0 )
    return stop_count

def count_tp_in_hours(hours=0):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_time = mt5_now - datetime.timedelta(hours=hours)
    orders = mt5.history_deals_get(start_time, mt5_now)
    stop_count = sum(1 for order in orders if order.profit > 0 )
    return stop_count

def count_sl_in_hours_with_comment(comment, hours=0):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_time = mt5_now - datetime.timedelta(hours=hours)
    deals = mt5.history_deals_get(start_time, mt5_now)
    if deals is None:
        print("No deals found.")
        return 0
    position_ids = set(deal.position_id for deal in deals if deal.comment in comment and deal.type in (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL))
    stop_count = sum(1 for deal in deals if deal.position_id in position_ids and deal.profit < 0)

    return stop_count

def count_tp_in_hours_with_comment(comment, hours=0):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_time = mt5_now - datetime.timedelta(hours=hours)
    deals = mt5.history_deals_get(start_time, mt5_now)
    if deals is None:
        print("No deals found.")
        return 0
    position_ids = set(deal.position_id for deal in deals if deal.comment in comment and deal.type in (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL))
    stop_count = sum(1 for deal in deals if deal.position_id in position_ids and deal.profit > 0)

    return stop_count

def total_profit_in_hours_with_comment(comments , hours=0):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_time = mt5_now - datetime.timedelta(hours=hours)
    deals = mt5.history_deals_get(start_time, mt5_now)
    
    if deals is None:
        print("No deals found.")
        return 0

    # تبدیل لیست کامنت‌ها به مجموعه برای جستجوی سریع‌تر
    comment_set = set(comments)
    position_ids = set(deal.position_id for deal in deals if deal.comment in comment_set and deal.type in (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL))
    
    profit_in_hours = sum(deal.profit for deal in deals if deal.position_id in position_ids)

    return profit_in_hours

def count_sl_with_comment(comment):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference
    deals = mt5.history_deals_get(start_of_day, mt5_now)
    if deals is None:
        print("No deals found.")
        return 0
    position_ids = set(deal.position_id for deal in deals if deal.comment in comment and deal.type in (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL))
    stop_count = sum(1 for deal in deals if deal.position_id in position_ids and deal.profit < 0)

    return stop_count

def total_profit_today_with_comment(comments):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference
    deals = mt5.history_deals_get(start_of_day, mt5_now)
    
    if deals is None:
        print("No deals found.")
        return 0

    # تبدیل لیست کامنت‌ها به مجموعه برای جستجوی سریع‌تر
    comment_set = set(comments)
    position_ids = set(deal.position_id for deal in deals if deal.comment in comment_set and deal.type in (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL))
    
    profit_today = sum(deal.profit for deal in deals if deal.position_id in position_ids)

    return profit_today

def count_tp_with_comment(comment):
    time_difference = datetime.timedelta(hours=2)
    mt5_now = datetime.datetime.now(datetime.timezone.utc) + time_difference
    start_of_day = datetime.datetime(mt5_now.year, mt5_now.month, mt5_now.day, tzinfo=datetime.timezone.utc) + time_difference
    deals = mt5.history_deals_get(start_of_day, mt5_now)
    if deals is None:
        print("No deals found.")
        return 0
    position_ids = set(deal.position_id for deal in deals if deal.comment in comment and deal.type in (mt5.ORDER_TYPE_BUY, mt5.ORDER_TYPE_SELL))
    profit_count = sum(1 for deal in deals if deal.position_id in position_ids and deal.profit > 0)

    return profit_count

def tsi(symbol , timeframe ,line = 'tsi',long_length = 25, short_length=13 , signal_length=13):
    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '2m':
        time = mt5.TIMEFRAME_M2
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1

    rates = mt5.copy_rates_from_pos(symbol, time, 0, 1000) 
    prices = pd.DataFrame(rates)['close'] 
    def ema(data, length):
        return data.ewm(span=length, adjust=False).mean()

    def double_smooth(src, long_length, short_length):
        first_smooth = ema(src, long_length)
        return ema(first_smooth, short_length)
    
    pc = prices.diff()
    double_smoothed_pc = double_smooth(pc, long_length, short_length)
    double_smoothed_abs_pc = double_smooth(np.abs(pc), long_length, short_length)
    tsi_value = 100 * (double_smoothed_pc / double_smoothed_abs_pc)
    signal_line = ema(tsi_value, signal_length)

    if line == 'tsi':
        return tsi_value.to_numpy()
    else:
        return signal_line.to_numpy()

def ut_bot(symbol, timeframe, a=2, atr_period=1):
    ohlc = kandel(timeframe, 50, symbol)
    
    candles = pd.DataFrame(ohlc[:], columns=['timestamp', 'open', 'high', 'low', 'close', 'volume'])
    
    high_low = candles['high'] - candles['low']
    high_close = np.abs(candles['high'] - candles['close'].shift())
    low_close = np.abs(candles['low'] - candles['close'].shift())
    tr = pd.concat([high_low, high_close, low_close], axis=1).max(axis=1)
    candles['ATR'] = tr.rolling(window=atr_period).mean()
    
    candles['nLoss'] = a * candles['ATR']
    
    candles['xATRTrailingStop'] = np.nan
    
    for i in range(1, len(candles)):
        prev_stop = candles.loc[i-1, 'xATRTrailingStop'] if not np.isnan(candles.loc[i-1, 'xATRTrailingStop']) else candles.loc[i, 'close']
        
        if candles.loc[i, 'close'] > prev_stop:
            candles.loc[i, 'xATRTrailingStop'] = max(prev_stop, candles.loc[i, 'close'] - candles.loc[i, 'nLoss'])
        else:
            candles.loc[i, 'xATRTrailingStop'] = min(prev_stop, candles.loc[i, 'close'] + candles.loc[i, 'nLoss'])
    
    candles['position'] = candles.apply(lambda row: 'long' if row['close'] > row['xATRTrailingStop'] else 'short', axis=1)

    position = candles['position'].iloc

    return position

def half_trend(symbol, timeframe, amplitude=2, channel_deviation=2):
    ohlc = kandel(timeframe, amplitude * 20, symbol)
    df = ohlc[:]

    df['high_price'] = df['high'].rolling(window=amplitude).max()
    df['low_price'] = df['low'].rolling(window=amplitude).min()

    df['TR'] = np.maximum(df['high'] - df['low'], 
    np.maximum(abs(df['high'] - df['close'].shift(1)), 
    abs(df['low'] - df['close'].shift(1))))
    df['ATR'] = df['TR'].rolling(window=100).mean() / 2
    df['Dev'] = channel_deviation * df['ATR']
 
    df['high_ma'] = df['high'].rolling(window=amplitude).mean()
    df['low_ma'] = df['low'].rolling(window=amplitude).mean()

    df['trend'] = 0
    df['max_low_price'] = df['low'].shift(1).copy()
    df['min_high_price'] = df['high'].shift(1).copy()
    df['up'] = 0.0
    df['down'] = 0.0
    df['HalfTrend'] = np.nan
 
    for i in range(1, len(df)):
        if df.at[i - 1, 'trend'] == 1:
            df.at[i, 'max_low_price'] = max(df.at[i, 'low_price'], df.at[i - 1, 'max_low_price'])

            if df.at[i, 'high_ma'] < df.at[i, 'max_low_price'] and df.at[i, 'close'] < df.at[i - 1, 'low']:
                df.at[i, 'trend'] = 0
                df.at[i, 'min_high_price'] = df.at[i, 'high_price']
            else:
                df.at[i, 'trend'] = df.at[i - 1, 'trend']
        else:
            df.at[i, 'min_high_price'] = min(df.at[i, 'high_price'], df.at[i - 1, 'min_high_price'])

            if df.at[i, 'low_ma'] > df.at[i, 'min_high_price'] and df.at[i, 'close'] > df.at[i - 1, 'high']:
                df.at[i, 'trend'] = 1
                df.at[i, 'max_low_price'] = df.at[i, 'low_price']
            else:
                df.at[i, 'trend'] = df.at[i - 1, 'trend']

        if df.at[i, 'trend'] == 0:
            df.at[i, 'up'] = max(df.at[i, 'max_low_price'], df.at[i - 1, 'up'] if i > 0 else df.at[i, 'max_low_price'])
        else:
            df.at[i, 'down'] = min(df.at[i, 'min_high_price'], df.at[i - 1, 'down'] if i > 0 else df.at[i, 'min_high_price'])

        df.at[i, 'HalfTrend'] = df.at[i, 'up'] if df.at[i, 'trend'] == 0 else df.at[i, 'down']

    positions = ['long' if df.at[i, 'HalfTrend'] <= df.at[i, 'close'] else 'short' for i in range(len(df))]

    return positions

def half_trend_signal(symbol , timeframe , amplitude , channel_deviation):

    half_trend(symbol , timeframe , amplitude , channel_deviation)

    half_candle = half_trend(symbol , timeframe , amplitude , channel_deviation)

    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1
        
    if half_candle[-3] == 'short' and half_candle[-2] == 'long' and half_candle[-1] == 'long' :
        return 'long'
    if half_candle[-3] == 'long' and half_candle[-2] == 'short' and half_candle[-1] == 'short' :
        return 'short'

def half_signal(symbol, timeframe, amplitude=2, channel_deviation=2) :
    half = half_trend(symbol, timeframe, amplitude, channel_deviation)

    if half[-3] == 'short' and half[-2] == 'long' :
        return 'buy'
    elif half[-3] == 'long' and half[-2] == 'short' :
        return 'sell'
    else :
        return 'hold'

def position_time_check(symbol , comment , timeframe , timezone = 3) : 
    time_map = {
        '1m': 1, '3m': 3, '5m': 5, '10m': 10, 
        '15m': 15, '20m': 20, '30m': 30, 
        '1h': 60, '4h': 240
    }
    time_plus = time_map.get(timeframe, 1)

    def broker_now(h = 0 ) : 
        custom_timezone = datetime.timezone(datetime.timedelta(hours=h))
        return datetime.datetime.now(custom_timezone)
    
    

    # adding history profits
    now = broker_now(timezone)
    from_time = datetime.datetime(now.year, now.month, now.day, tzinfo=now.tzinfo)
    to_time = from_time + datetime.timedelta(days=1)

    history = mt5.history_deals_get(from_time, to_time)
    positions = [i for i in history if i.entry == 0 and i.comment == comment and i.symbol == symbol]


    if len(positions) > 0 : 
        t = positions[-1].time
        dt_time = datetime.datetime.fromtimestamp(t, datetime.UTC)
        aware_dt_time = dt_time.replace(tzinfo=datetime.timezone(datetime.timedelta(hours=timezone)))
        new_time = aware_dt_time + datetime.timedelta(minutes=time_plus)
        if new_time < now : 
            return True
        else : 
            return False
    else : 
        return True

def candle(timeframe='30m', limit=500, symbol='BTCUSD'):
        timeframes = {
            '1m': mt5.TIMEFRAME_M1,
            '3m': mt5.TIMEFRAME_M3,
            '5m': mt5.TIMEFRAME_M5,
            '15m': mt5.TIMEFRAME_M15,
            '30m': mt5.TIMEFRAME_M30,
            '1h': mt5.TIMEFRAME_H1,
            '4h': mt5.TIMEFRAME_H4,
            '1d': mt5.TIMEFRAME_D1,
            '1w': mt5.TIMEFRAME_W1,
            '1mn': mt5.TIMEFRAME_MN1
        }
        
        time = timeframes.get(timeframe, mt5.TIMEFRAME_M30)  # Default to '30m' if not found
        candles = mt5.copy_rates_from_pos(symbol, time, 0, limit)
        
        if candles is None or len(candles) == 0:
            print(f"No data received for {symbol} on {timeframe}")
            return pd.DataFrame()  # Return empty DataFrame
        
        df = pd.DataFrame(candles, columns=['time', 'open', 'high', 'low', 'close'])
        df['time'] = pd.to_datetime(df['time'], unit='s')
        return df

def stochrsi(symbol , timeframe , line , rsi_length=14, stoch_length=14, k_period=3, d_period=3):
    candles = candle(timeframe , stoch_length * 10 , symbol)
    df_close = candles['close']
    stoch_rsi = pandas_ta.stochrsi(df_close, length=stoch_length, rsi_length=rsi_length, k=k_period, d=d_period)

    d = stoch_rsi['STOCHRSId_' + str(rsi_length) + '_' + str(stoch_length) + '_' + str(k_period) + '_' + str(d_period)].tolist()
    k = stoch_rsi['STOCHRSIk_' + str(rsi_length) + '_' + str(stoch_length) + '_' + str(k_period) + '_' + str(d_period)].tolist()
    
    return k if line == 'k' else d

def stoch(symbol, timeframe, type='k', period_k=14, smooth_k=1, period_d=3):
    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '2m':
        time = mt5.TIMEFRAME_M2
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1
    rates = mt5.copy_rates_from_pos(symbol, time, 0, 1000)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    lowest_low = df['low'].rolling(window=period_k).min()
    highest_high = df['high'].rolling(window=period_k).max()
    k = 100 * ((df['close'] - lowest_low) / (highest_high - lowest_low))
    k_smooth = k.rolling(window=smooth_k).mean()
    if type.lower() == 'k':
        return k_smooth.tolist()
    elif type.lower() == 'd':
        d = k_smooth.rolling(window=period_d).mean()
        return d.tolist()
    else:
       return False

def cmo(symbol , timeframe, period):

    ohlc = kandel(timeframe , period*10 , symbol)
    df = pd.DataFrame(ohlc[:])
    prices = df['close']

    delta = np.diff(prices)
    gains = np.where(delta > 0, delta, 0)
    losses = np.where(delta < 0, -delta, 0)

    sum_gains = pd.Series(gains).rolling(window=period).sum()
    sum_losses = pd.Series(losses).rolling(window=period).sum()

    cmo = 100 * (sum_gains - sum_losses) / (sum_gains + sum_losses)
    return cmo

def stochrsi_cross(symbol , timeframe , rsi_length=14, stoch_length=14, k_period=3, d_period=3) :
    d = stochrsi(symbol , timeframe , 'd' , rsi_length, stoch_length, k_period, d_period)
    k = stochrsi(symbol , timeframe , 'k' , rsi_length, stoch_length, k_period, d_period)

    if k[-4] < d[-2] and k[-3] < d[-2] and k[-2] <= d[-2] and k[-1] > d[-1] and d[-2] < 50:
        return 'buy'
    elif  k[-4] > d[-2] and k[-3] > d[-2] and k[-2] >= d[-2] and k[-1] < d[-1] and d[-2] > 50:
        return 'sell'
    else :
        None

def save_to_xlsx_h(hours_back, timezone = 3):  
    current_datetime_str = datetime.datetime.now().strftime("%Y-%m-%d-Hour-%H")
    
    if not current_datetime_str.lower().endswith('.xlsx'):
        current_datetime_str += '.xlsx'

    utc_to = datetime.datetime.now() + datetime.timedelta(hours=timezone)
    utc_from = utc_to - datetime.timedelta(hours=hours_back)

    
    
    trades = mt5.history_deals_get(utc_from, utc_to)
    
    if not trades:
        print(f"هیچ تاریخچه معامله‌ای در {hours_back} ساعت گذشته یافت نشد")
        mt5.shutdown()
        return

    
    df = pd.DataFrame(list(trades), columns=trades[0]._asdict().keys())

    
    df = df.drop(columns=[col for col in df.columns if 'time_msc' in col])

    
    merged_data = []

    
    for pos_id, pos_trades in df.groupby('position_id'):
        if len(pos_trades) > 1:
            
            merged_row = {}
            for column in pos_trades.columns:
                unique_values = pos_trades[column].unique()
                if column in ['مبلغ', 'comment', 'time']:
                    for i, value in enumerate(unique_values, start=1):
                        merged_row[f"{column}_{i}"] = value
                elif column == 'سود':
                    valid_profits = [value for value in unique_values if value != 0]
                    if valid_profits:
                        merged_row[column] = "/".join(map(str, valid_profits))
                elif column in ['type', 'entry']:
                    merged_row[column] = "/".join(map(str, unique_values))
                else:
                    merged_row[column] = unique_values[0] if len(unique_values) == 1 else "/".join(map(str, unique_values))
            merged_data.append(merged_row)
        else:
            
            row_data = pos_trades.iloc[0].to_dict()
            row_data['price_1'] = row_data.pop('profit')
            row_data['comment_1'] = row_data.pop('comment')
            row_data['time_1'] = row_data.pop('time')
            if row_data.get('سوود', 0) == 0:
                row_data.pop('profسوووit', None)
            merged_data.append(row_data)

    
    merged_df = pd.DataFrame(merged_data)
    
    
    try:
        
        merged_df.to_excel(current_datetime_str, index=False)
        print(f"تاریخچه معاملات ادغام شده برای {hours_back} ساعت گذشته در {current_datetime_str} ذخیره شد")
    except Exception as e:
        print(f"خطا در ذخیره‌سازی فایل اکسل: {str(e)}")

def get_adx(symbol, timeframe, period=14, adx_smoothing=14):
    
    timeframes = {
        '5m': mt5.TIMEFRAME_M5,
        '3m': mt5.TIMEFRAME_M3,
        '1m': mt5.TIMEFRAME_M1,
        '10m': mt5.TIMEFRAME_M10,
        '15m': mt5.TIMEFRAME_M15,
        '30m': mt5.TIMEFRAME_M30,
        '1h': mt5.TIMEFRAME_H1,
        '4h': mt5.TIMEFRAME_H4,
        '1d': mt5.TIMEFRAME_D1,
        '1w': mt5.TIMEFRAME_W1,
        '1mn': mt5.TIMEFRAME_MN1
    }

    if timeframe not in timeframes:
        return None

    time = timeframes[timeframe]
    rates = mt5.copy_rates_from_pos(symbol, time, 0, 500)
    if rates is None:
        return None

    data = pd.DataFrame(rates)

    high_prices = data['high'].values
    low_prices = data['low'].values
    close_prices = data['close'].values

    adx_raw = talib.ADX(high_prices, low_prices, close_prices, timeperiod=period)

    if adx_smoothing > 0:
        adx_smoothed = talib.SMA(adx_raw, timeperiod=max(int(adx_smoothing), 1))
    else:
        adx_smoothed = adx_raw
    
    return adx_smoothed

def total_profit_with_comment_now(comments):
    # اتصال به متاتریدر
    if not mt5.initialize():
        print("متاتریدر راه‌اندازی نشد")
        return

    # دریافت اطلاعات پوزیشن‌ها
    positions = mt5.positions_get()

    if positions is None:
        print("هیچ پوزیشنی یافت نشد.")
        mt5.shutdown()
        return

    total_profit = 0.0
    found_positions = False

    # تبدیل لیست کامنت‌ها به مجموعه برای جستجوی سریع‌تر
    comment_set = set(comments)

    # بررسی پوزیشن‌ها و محاسبه پروفیت
    for position in positions:
        if position.comment in comment_set:
            total_profit += position.profit
            found_positions = True

    if found_positions:
        return total_profit
    else:
        return 0.0

def winRate(symbol , timeframe , type='sell' , risk=None):
    
    if symbol in  ['XAUUSD' , 'XAUUSD.' , 'XAUUSD_i'] : 
        m = 1
    if  symbol in ['EURUSD' , 'GBPUSD'  , 'USDCHF' , 'AUDUSD' , 'EURUSD.' , 'EURUSD_i' , 'GBPUSD.' , 'GBPUSD_i' , 'USDCHF.' , 'USDCHF_i' , 'AUDUSD.' , 'AUDUSD_i']  :
        m = 0.0002
        
    win = 50

    half = half_trend(symbol , timeframe)
    p_sar = sar(symbol , timeframe , 0.01 , 0.1)
    s_s = sar_signal(symbol , timeframe , 0.01 , 0.1)
    h_s = half_signal(symbol , timeframe)
    mcd = cross_macd(symbol , timeframe , 34 , 144)
    ts = cross_tsi(symbol , timeframe )
    rsii =  rsi('1m' , symbol )
    rsiii =  rsi(timeframe , symbol )
    kandel_now = whatKandel(timeframe , -1 , symbol)
    histogram = macd(symbol , timeframe , 'histogram' , 34 , 144)[-1]
    if type == 'sell':
        
        if kandel_now == 'long':
            win -= 5
          
        if histogram > 0:
            win -= 5
            
              
        priceWin = mt5.symbol_info_tick(symbol).bid
        if check_time(13 , 16) :
            
            if priceWin < london_hl(symbol , 'h') - london_vol(symbol) / 2 or tokyo_hl(symbol , 'l') > london_hl(symbol , 'l'):
                win += 5
        
        if check_time(7 , 10) :
            if priceWin < tokyo_hl(symbol , 'h') - tokyo_vol(symbol) / 2:
                win += 3

        if nadaraya_signals(symbol , timeframe , 2 , 8)[-1] == 'short' :
            win += 5
        if supertrend_signal(symbol , timeframe , 9 , 3.9) == 'short' :
            win += 5
        if ichi_cross(symbol, timeframe , 4 , 9 ) == 'up to down' :
            win += 5
        if ema_cross(timeframe , symbol  , 50 , 100 )== 'up to down' :
            win += 5
        if half[-1] == 'short':
            win += 5
        if half[-2] == 'short':
            win += 2 
        if p_sar[-1] > priceWin:
            win += 5 
        if s_s == 'short' or h_s == 'sell' :
            win += 7 
        if mcd == 'sell' or ts == 'sell':
            win += 7 
            
        if rsii <= 25 or rsiii <= 35 :
            win -= 20 

        if check_time(7 , 15) and tokyo_vol(symbol) < m*15 :
            win += 2

        if check_time(7 , 15) and tokyo_vol(symbol) < m*12 :
            win += 2
        
        if check_time(7 , 15) and tokyo_vol(symbol) < m*10 :
            win += 2

        if check_time(7 , 15) and tokyo_vol(symbol) > m*20 :
            win -= 3
        
        if check_time(7 , 15) and tokyo_vol(symbol) > m*30 :
            win -= 3

        if check_time(3,8):
            win -= 20

        if check_time(19 , 2):
            win -= 30
        
        if check_time(11 , 13):
            win += 10

        if check_time(14 , 17):
            win += 5
        
        if check_time(13 , 19) and tokyo_vol(symbol) > m*15 and london_vol(symbol) > m*20 :
            win -= 5

    else:
        if kandel_now == 'short':
            win -= 5
            
        if histogram < 0:
            win -= 5
            
        priceWin = mt5.symbol_info_tick(symbol).ask
        if check_time(13 , 16) :
            
            if priceWin > london_hl(symbol , 'l') + london_vol(symbol) / 2 or tokyo_hl(symbol , 'h') < london_hl(symbol , 'h'):
                win += 5
        
        if check_time(7 , 10) :
            if priceWin > tokyo_hl(symbol , 'l') + tokyo_vol(symbol) / 2:
                win += 3
        
        if nadaraya_signals(symbol , timeframe , 2 , 8)[-1] == 'long' :
            win += 5
        if supertrend_signal(symbol , timeframe , 9 , 3.9) == 'long' :
            win += 5
        if ichi_cross(symbol, timeframe , 4 , 9 ) == 'down to up' :
            win += 5
        if ema_cross(timeframe , symbol  , 50 , 100 )== 'down to up' :
            win += 5
        if half[-1] == 'long':
            win += 5
        if half[-2] == 'long':
            win += 2 
        if p_sar[-1] < priceWin:
            win += 5 
        if s_s == 'long' or h_s == 'buy' :
            win += 7 
        if mcd == 'buy' or ts == 'buy':
            win += 7 
            
        if rsii >= 75 or rsiii >= 65 :
            win -= 20 

        if check_time(7 , 15) and  tokyo_vol(symbol) < m*15:
            win += 2

        if  check_time(7 , 15) and tokyo_vol(symbol) < m*12:
            win += 2
        
        if  check_time(7 , 15) and tokyo_vol(symbol) < m*10:
            win += 2

        if  check_time(7 , 15) and tokyo_vol(symbol) > m*20:
            win -= 3
        
        if  check_time(7 , 15) and tokyo_vol(symbol) > m*30:
            win -= 3

        if check_time(3,8):
            win -= 20

        if check_time(19 , 2):
            win -= 30
        
        if check_time(11 , 13):
            win += 10

        if check_time(14 , 17):
            win += 5
        
        if check_time(13 , 19) and tokyo_vol(symbol) > m*20 and london_vol(symbol) > m*30 :
            win -= 5

    if today == 'Friday' or today == 'Monday':
        win += 10

    if win >= 95:
        win = 95
    if win <= 5 :
        win = 5

    
    if risk == None:
        return win
    else:
        if win >= 80 :
            risk = risk* 2
            return risk
        elif win < 80 and win >= 65:
            risk = risk* 1.2
            return risk
        elif win < 65 and win >= 50:
            risk = risk
            return risk
        elif win < 50 and win >= 40:
            risk = risk / 1.3
            return risk
        elif win < 40 :
            risk = risk / 2
            return risk
        else:
            return risk

def tokyo_vol(symbol):

    high = time_high_low(symbol, 0, 8, '1h',8, 120)['high']
    low = time_high_low(symbol, 0, 8, '1h',8, 120)['low']

    return high - low

def london_vol(symbol):

    high = time_high_low(symbol, 7, 15, '1h',8, 120)['high']
    low = time_high_low(symbol, 7, 15, '1h',8, 120)['low']

    return high - low

def new_york_vol(symbol):

    high = time_high_low(symbol, 13, 21, '1h',8, 120)['high']
    low = time_high_low(symbol, 13, 21, '1h',8, 120)['low']

    return high - low

def tokyo_hl(symbol , hl):

    high = time_high_low(symbol, 0, 8, '1h',8, 120)['high']
    low = time_high_low(symbol, 0, 8, '1h',8, 120)['low']
    if hl == 'h':
        return high
    else:
        return low

def london_hl(symbol,hl):

    high = time_high_low(symbol, 7, 15, '1h',8, 120)['high']
    low = time_high_low(symbol, 7, 15, '1h',8, 120)['low']

    if hl == 'h':
        return high
    else:
        return low

def new_york_hl(symbol,hl):

    high = time_high_low(symbol, 13, 21, '1h',8, 120)['high']
    low = time_high_low(symbol, 13, 21, '1h',8, 120)['low']

    if hl == 'h':
        return high
    else:
        return low

def count_position_now(type , symbol):
    positions = mt5.positions_get()
    buy = 0
    sell = 0
    for position in positions :
        if position._asdict()['type'] == 0 and position._asdict()['symbol'] == symbol :
            buy += 1

        if position._asdict()['type'] == 1 and position._asdict()['symbol'] == symbol :
            sell += 1
    if type == 'buy':
        return buy
    else:
        return sell
    
def cross_tsi(symbol , timeframe ,long_length = 25, short_length=13 , signal_length=13 ):
    tsis = tsi(symbol , timeframe , 'tsi' , long_length , short_length , signal_length)
    signals = tsi(symbol , timeframe , 'signals' , long_length , short_length , signal_length)
    if  tsis[-1]> signals[-1] and tsis[-2]>=signals[-2] and tsis[-3]<signals[-3] and tsis[-4]<signals[-2] and tsis[-5]<signals[-2]:
        return 'buy'
    elif tsis[-1]< signals[-1] and tsis[-2]<=signals[-2] and tsis[-3]>signals[-3] and tsis[-4]>signals[-2] and tsis[-5]>signals[-2]:
        return 'sell'
    else:
        return False

def lot_calculator(symbol : str , risk : int , open_price : float , stop_loss : float) : 
    price = open_price
    sl = stop_loss
    amount_of_risk = mt5.account_info()._asdict()['balance'] * (risk / 100)
    raw_lot_size = 0
    lot_size = None
    if symbol in  ['XAUUSD' , 'XAUUSD.' , 'XAUUSD_i'] : 

        raw_lot_size = amount_of_risk / (abs(sl - price) * 100)
        raw_lot_size = math.floor(raw_lot_size *100) / 100.0

    elif symbol in ['USDJPY' , 'USDJPY.' , 'USDJPY_i']  : 
        pip_size = 0.01

        stop_pip_integer = abs((sl - price) / pip_size)

        pip_value = amount_of_risk / stop_pip_integer
        pip_value = pip_value * sl

        raw_lot_size = pip_value / 1000
    
    elif symbol in ['BTCUSD' , 'BTCUSD.' , 'BTCUSD_i']  : 
        pip_size = 1.00  

        stop_pip_integer = abs((sl - price) / pip_size)

        raw_lot_size = amount_of_risk / stop_pip_integer

    elif symbol in ['USDCAD' , 'USDCAD.' , 'USDCAD_i'] : 
        pip_size = 0.0001 

        stop_pip_integer = abs((sl - price) / pip_size)

        pip_value = amount_of_risk / stop_pip_integer 
        pip_value = pip_value * sl 

        raw_lot_size = pip_value / 10

    elif symbol in ['EURUSD' , 'GBPUSD'  , 'USDCHF' , 'AUDUSD' , 'EURUSD.' , 'EURUSD_i' , 'GBPUSD.' , 'GBPUSD_i' , 'USDCHF.' , 'USDCHF_i' , 'AUDUSD.' , 'AUDUSD_i']  :
        pip_size = 0.0001

        stop_pip_integer = abs((sl - price) / pip_size)

        pip_value = amount_of_risk / stop_pip_integer

        raw_lot_size = pip_value / 10
    else : 
        lot_size = 0.01


    if raw_lot_size != 0 : 
        lot_size = float(raw_lot_size)
        lot_size = math.floor(lot_size * 100) / 100.0

        if lot_size < 0.01  : 
            lot_size = 0.01 


    if lot_size is None :

        if symbol in ['EURUSD' , 'GBPUSD'  , 'USDCHF' , 'AUDUSD' , 'EURUSD.' , 'EURUSD_i' , 'GBPUSD.' , 'GBPUSD_i' , 'USDCHF.' , 'USDCHF_i' , 'AUDUSD.' , 'AUDUSD_i']  :
           lot_size = 0.05
     
        else:
            lot_size = 0.01
    
    return lot_size

def pnl_today( timezone = 3 ) : 
    def broker_now(h = 0 ) : 
        custom_utc_offset = datetime.timedelta(hours=h)
        custom_timezone = pytz.FixedOffset(custom_utc_offset.total_seconds() // 60)

        return datetime.datetime.now(custom_timezone)


    profit = 0

    # adding history profits
    now = broker_now(timezone)
    from_time = datetime.datetime(now.year , now.month , now.day , 0 , 0 , 0 , 0)
    to_time =  from_time + datetime.timedelta(days=1)

    history = mt5.history_deals_get(from_time ,to_time )

    for position in history : 
        profit += position.commission
        profit += position.swap
        profit += position.profit
        profit += position.fee

    
    positions = mt5.positions_get()

    for i in positions : 
        position = i._asdict()
        profit += position['swap']
        profit += position['profit']


    return round(profit , 2)

def draw_down_checker (balance , pnl , percent) : 
    if pnl < 0 : 
        if (abs(pnl)) >= (balance * percent) : 
            return True
        else : 
            return False
    else : 
        return False

def total_draw_down(balance , percent):
    equity = mt5.account_info()._asdict()['equity']
    if equity - balance < 0 : 
        if abs(equity - balance) >= (balance * percent) : 
            return True
        else : 
            return False
    else : 
        return False

def heiken_ashi_signals(timeframe='5m' , limit=500 , symbol='XAUUSD.'):

    ca = candle(timeframe , limit , symbol)
    hi = heiken_ashi(ca).iloc[-1]

    if hi['close'] > hi['open'] :
        hi_signal = 'long'
    elif hi['close'] < hi['open'] :
        hi_signal = 'short'
    else :
        hi_signal = None

    return hi_signal

def heiken_ashi(df):
        if len(df) < 2:
            return False
        
        ha_close = (df['open'] + df['high'] + df['low'] + df['close']) / 4.0
        ha_open = pd.Series(index=df.index, data=np.nan)
        ha_open.iloc[0] = (df['open'].iloc[0] + df['close'].iloc[0]) / 2.0
        
        for i in range(1, len(df)):
            ha_open.iloc[i] = (ha_open.iloc[i-1] + ha_close.iloc[i-1]) / 2.0
        
        ha_high = np.maximum(df['high'], np.maximum(ha_open, ha_close))
        ha_low = np.minimum(df['low'], np.minimum(ha_open, ha_close))
        
        ha_df = pd.DataFrame({
            'open': ha_open,
            'high': ha_high,
            'low': ha_low,
            'close': ha_close
        })
        return ha_df

def TS_signal(symbol, timeframe, ma_type="EMA", ma_period=9, alma_offset=0.85, alma_sigma=6, num_candles=100):
    last1 = trend_signal(symbol, timeframe, ma_type="EMA", ma_period=9, alma_offset=0.85, alma_sigma=6, num_candles=100)[-1]
    last2 = trend_signal(symbol, timeframe, ma_type="EMA", ma_period=9, alma_offset=0.85, alma_sigma=6, num_candles=100)[-2]
    last3 = trend_signal(symbol, timeframe, ma_type="EMA", ma_period=9, alma_offset=0.85, alma_sigma=6, num_candles=100)[-3]
    if last1 == 'long' and last2 == 'long' and last3 == 'short' :
        return 'buy'
    elif last1 == 'short' and last2 == 'short'and last3 == 'long':
        return 'sell'
    else:
        return False

def trend_signal(symbol, timeframe, ma_type="EMA", ma_period=9, alma_offset=0.85, alma_sigma=6, num_candles=100):

    if timeframe == '5m':
        time = mt5.TIMEFRAME_M5
    if timeframe == '3m':
        time = mt5.TIMEFRAME_M3
    if timeframe == '1m':
        time = mt5.TIMEFRAME_M1
    if timeframe == '2m':
        time = mt5.TIMEFRAME_M2
    if timeframe == '15m':
        time = mt5.TIMEFRAME_M15
    if timeframe == '30m':
        time = mt5.TIMEFRAME_M30
    if timeframe == '1h':
        time = mt5.TIMEFRAME_H1
    if timeframe == '4h':
        time = mt5.TIMEFRAME_H4
    if timeframe == '1d':
        time = mt5.TIMEFRAME_D1
    if timeframe == '1w':
        time = mt5.TIMEFRAME_W1
    if timeframe == '1mn':
        time = mt5.TIMEFRAME_MN1
    rates = mt5.copy_rates_from_pos(symbol, time, 0, num_candles)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')

    df['HA_Close'] = (df['close'] + df['open'] + df['high'] + df['low']) / 4
    df['HA_Open'] = (df['open'].shift(1) + df['close'].shift(1)) / 2
    df['HA_High'] = df[['high', 'HA_Open', 'HA_Close']].max(axis=1)
    df['HA_Low'] = df[['low', 'HA_Open', 'HA_Close']].min(axis=1)

    if ma_type == "ALMA":
        m = np.floor(0.5 * (ma_period - 1))
        s = ma_period / alma_sigma
        w = np.exp(-((np.arange(ma_period) - m)**2) / (2 * s**2))
        alma = np.convolve(df['HA_Close'], w / w.sum(), mode='valid')
        df['MA'] = pd.Series(alma, index=df.index[ma_period-1:])
    elif ma_type == "HMA":
        wma_half = df['HA_Close'].rolling(window=ma_period//2).mean()
        wma_full = df['HA_Close'].rolling(window=ma_period).mean()
        df['MA'] = 2 * wma_half - wma_full
        df['MA'] = df['MA'].rolling(window=int(np.sqrt(ma_period))).mean()
    elif ma_type == "SMA":
        df['MA'] = df['HA_Close'].rolling(window=ma_period).mean()
    elif ma_type == "EMA":
        df['MA'] = df['HA_Close'].ewm(span=ma_period, adjust=False).mean()
    elif ma_type == "ZLEMA":
        lag = (ma_period - 1) // 2
        df['MA'] = df['HA_Close'] + (df['HA_Close'] - df['HA_Close'].shift(lag)).ewm(span=ma_period, adjust=False).mean()
    else:
        raise ValueError("نوع MA معتبر نیست!")

    df['Trend'] = 100 * (df['HA_Close'] - df['HA_Open']) / (df['HA_High'] - df['HA_Low'])
    signals = np.where(df['Trend'] > 0, 'long', 'short')

    return signals

def williams(symbol , timeframe , period = 14) : 

    ohlc = kandel(timeframe , period*5 , symbol)
    candles = pd.DataFrame(ohlc[:])

    high = candles['high'].rolling(window=period).max()
    low = candles['low'].rolling(window=period).min()
    close = candles['close']
    williams_r = ((high - close) / (high - low)) * -100
    return williams_r.tolist()

def verify_hajm(symbol , type):
    kandels = kandel('1h' , 3 , symbol)
    if (((kandels[-1]['close'] - kandels[-2]['low'] ) >= 13 and whatKandel('1h' , -1 , symbol) == 'long' and whatKandel('1h' , -2 , symbol) == 'long' ) or (whatKandel('1h' , -2 , symbol) == 'long' and kandels[-2]['high'] - kandels[-2]['low'] > 13)) and type == 'long':
        return False
    elif (((kandels[-1]['high'] - kandels[-2]['close'] )  >= 13 and whatKandel('1h' , -1 , symbol) == 'short' and whatKandel('1h' , -2 , symbol) == 'short') or (whatKandel('1h' , -2 , symbol) == 'short' and kandels[-2]['high'] - kandels[-2]['low'] > 13 )) and type == 'short':
        return False
    else:
        return True

def confirmation(symbol , timeframe , type):
    if ravand_verify == 'long' and type == 'long' :
        return True
    if ravand_verify == 'short' and type == 'short' :
        return True
    else :
        return False
    
def pinbar(symbol , timeframe , type):
    if whatKandel(timeframe , -2 , symbol) == 'short' and ((body(timeframe , -2 , symbol)*3)+(kandel(timeframe , 5 , symbol)[-2]['open'])) < (kandel(timeframe , 5 , symbol)[-2]['high']) and type == 'short':
        return True
    if whatKandel(timeframe , -2 , symbol) == 'long' and ((body(timeframe , -2 , symbol)*3)-(kandel(timeframe , 5 , symbol)[-2]['open'])) > (kandel(timeframe , 5 , symbol)[-2]['low']) and type == 'long':
        return True
    else :
        return False
    
def engulfing(symbol, timeframe, num_candles=100):
    # تبدیل تایم‌فریم به شناسه متاتریدر
    timeframes = {
        '1m': mt5.TIMEFRAME_M1,
        '3m': mt5.TIMEFRAME_M3,
        '5m': mt5.TIMEFRAME_M5,
        '15m': mt5.TIMEFRAME_M15
    }
    
    if timeframe not in timeframes:
        raise ValueError("Invalid timeframe. Choose from '1m', '3m', '5m', '15m'.")
    
    # دریافت داده‌ها از متاتریدر 5
    rates = mt5.copy_rates_from_pos(symbol, timeframes[timeframe], 0, num_candles)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')

    # تعریف ویژگی‌های شمع‌ها
    df['bullish_engulfing'] = (df['close'] > df['open']) & (df['close'].shift(1) < df['open'].shift(1)) & (df['open'] < df['close'].shift(1))
    df['bearish_engulfing'] = (df['close'] < df['open']) & (df['close'].shift(1) > df['open'].shift(1)) & (df['open'] > df['close'].shift(1))
    
    # بررسی وجود Engulfing
    if df['bullish_engulfing'].iloc[-1]:
        return True  # Bullish Engulfing
    elif df['bearish_engulfing'].iloc[-1]:
        return True  # Bearish Engulfing
    else:
        return False  # No Engulfing pattern

def engulfing2(symbol , timeframe , type) :
    if body(timeframe , -2 , symbol) > body(timeframe , -3 , symbol) and whatKandel(timeframe , -2 , symbol) == 'long' and whatKandel(timeframe , -3 , symbol) == 'short' and type == 'long':
        return True
    if body(timeframe , -2 , symbol) > body(timeframe , -3 , symbol) and whatKandel(timeframe , -2 , symbol) == 'short' and whatKandel(timeframe , -3 , symbol) == 'long' and type == 'short':
        return True
    else :
        return False
    
def ichimoku_signals(symbol, timeframe, ts_bars=9, ks_bars=26, ssb_bars=52, cs_offset=26, ss_offset=26):
    ohlc = kandel(timeframe, limit=max(ts_bars, ks_bars, ssb_bars) + ss_offset + cs_offset + 50, symbol=symbol)
    candles = pd.DataFrame(list(ohlc))

    if candles.empty or 'high' not in candles or 'low' not in candles or 'close' not in candles:
        raise ValueError("Invalid data received from `kandel`. Check the function output.")
    
    tenkan_sen = (candles['high'].rolling(window=ts_bars).max() + candles['low'].rolling(window=ts_bars).min()) / 2
    kijun_sen = (candles['high'].rolling(window=ks_bars).max() + candles['low'].rolling(window=ks_bars).min()) / 2
    senkou_span_a = ((tenkan_sen + kijun_sen) / 2).shift(ss_offset)
    senkou_span_b = ((candles['high'].rolling(window=ssb_bars).max() + candles['low'].rolling(window=ssb_bars).min()) / 2).shift(ss_offset)
    chikou_span = candles['close'].shift(-cs_offset).fillna(candles['close'].iloc[-1])  # رفع NaN

    signals = []
    last_signal = "neutral"

    for i in range(max(ts_bars, ks_bars, ssb_bars) + ss_offset, len(candles)):
        tk_cross_bull = tenkan_sen.iloc[i] > kijun_sen.iloc[i]
        tk_cross_bear = tenkan_sen.iloc[i] < kijun_sen.iloc[i]
        cs_cross_bull = candles['close'].iloc[i] > chikou_span.iloc[i]
        cs_cross_bear = candles['close'].iloc[i] < chikou_span.iloc[i]
        price_above_kumo = candles['close'].iloc[i] > max(senkou_span_a.iloc[i], senkou_span_b.iloc[i])
        price_below_kumo = candles['close'].iloc[i] < min(senkou_span_a.iloc[i], senkou_span_b.iloc[i])

        bullish_signal = tk_cross_bull and cs_cross_bull and price_above_kumo
        bearish_signal = tk_cross_bear and cs_cross_bear and price_below_kumo
        
        if bullish_signal and last_signal != "buy":
            signals.append("buy")
            last_signal = "buy"
        elif bearish_signal and last_signal != "sell":
            signals.append("sell")
            last_signal = "sell"
        else:
            signals.append("neutral")

    return signals



def ichimoku_signals_pro(symbol, timeframe, ts_bars=9, ks_bars=26, ssb_bars=52, cs_offset=26, ss_offset=26):
    ohlc = kandel(timeframe, limit=max(ts_bars, ks_bars, ssb_bars) + ss_offset + cs_offset + 50, symbol=symbol)
    candles = pd.DataFrame(list(ohlc))

    if candles.empty or 'high' not in candles or 'low' not in candles or 'close' not in candles:
        raise ValueError("Invalid data received from `kandel`. Check the function output.")
    
    def heikin_ashi(data):
        ha_data = data.copy()
        ha_data['close'] = (data['open'] + data['high'] + data['low'] + data['close']) / 4
        ha_data['open'] = (data['open'].shift(1) + data['close'].shift(1)) / 2
        ha_data['high'] = ha_data[['open', 'close', 'high']].max(axis=1)
        ha_data['low'] = ha_data[['open', 'close', 'low']].min(axis=1)
        ha_data.iloc[0, ha_data.columns.get_loc('open')] = (data['open'].iloc[0] + data['close'].iloc[0]) / 2  # اولین مقدار open
        return ha_data

    tenkan_sen = (candles['high'].rolling(window=ts_bars).max() + candles['low'].rolling(window=ts_bars).min()) / 2
    kijun_sen = (candles['high'].rolling(window=ks_bars).max() + candles['low'].rolling(window=ks_bars).min()) / 2
    senkou_span_a = ((tenkan_sen + kijun_sen) / 2).shift(ss_offset)
    senkou_span_b = ((candles['high'].rolling(window=ssb_bars).max() + candles['low'].rolling(window=ssb_bars).min()) / 2).shift(ss_offset)
    chikou_span = candles['close'].shift(-cs_offset).fillna(candles['close'].iloc[-1])  # رفع NaN

    ha_candles = heikin_ashi(candles)

    signals = []
    last_signal = "neutral"

    for i in range(max(ts_bars, ks_bars, ssb_bars) + ss_offset, len(candles)):
        tk_cross_bull = tenkan_sen.iloc[i] > kijun_sen.iloc[i]
        tk_cross_bear = tenkan_sen.iloc[i] < kijun_sen.iloc[i]
        cs_cross_bull = candles['close'].iloc[i] > chikou_span.iloc[i]
        cs_cross_bear = candles['close'].iloc[i] < chikou_span.iloc[i]
        price_above_kumo = candles['close'].iloc[i] > max(senkou_span_a.iloc[i], senkou_span_b.iloc[i])
        price_below_kumo = candles['close'].iloc[i] < min(senkou_span_a.iloc[i], senkou_span_b.iloc[i])

        bullish_signal = tk_cross_bull and cs_cross_bull and price_above_kumo
        bearish_signal = tk_cross_bear and cs_cross_bear and price_below_kumo

        ha_bullish = ha_candles['close'].iloc[i] > ha_candles['open'].iloc[i]
        ha_bearish = ha_candles['close'].iloc[i] < ha_candles['open'].iloc[i]
        
        if bullish_signal and ha_bullish and last_signal != "buy":
            signals.append("buy")
            last_signal = "buy"
        elif bearish_signal and ha_bearish and last_signal != "sell":
            signals.append("sell")
            last_signal = "sell"
        else:
            signals.append("neutral")

    return signals

def ssl_hybrid(symbol, timeframe):
    df = candle(timeframe , 500 , symbol)

    atr_period = 14
    atr_multiplier = 1
    baseline_length = 60
    continuation_length = 5
    keltner_multiplier = 0.2

    def wma(data, period):
        weights = np.arange(1, period + 1)
        return data.rolling(period).apply(lambda prices: np.dot(prices, weights) / weights.sum(), raw=True)

    def hma(data, period):
        half_length = int(period / 2)
        sqrt_length = int(np.sqrt(period))
        return wma(2 * wma(data, half_length) - wma(data, period), sqrt_length)

    def jma(data, period, phase=3, power=1):
        
        if phase < -100:
            phase_ratio = 0.5
        elif phase > 100:
            phase_ratio = 2.5
        else:
            phase_ratio = phase / 100 + 1.5

        beta = 0.45 * (period - 1) / (0.45 * (period - 1) + 2)
        alpha = beta ** power

        e0 = data.copy()
        e1 = data.copy()
        e2 = data.copy()
        jma_val = data.copy()

        for i in range(1, len(data)):
            e0[i] = (1 - alpha) * data.iloc[i] + alpha * e0[i - 1]
            e1[i] = (data.iloc[i] - e0[i]) * (1 - beta) + beta * e1[i - 1]
            e2[i] = (e0[i] + phase_ratio * e1[i] - jma_val[i - 1]) * (1 - alpha) ** 2 + (alpha ** 2) * e2[i - 1]
            jma_val[i] = e2[i] + jma_val[i - 1]

        return jma_val


    # ATR Calculation
    df['tr'] = np.maximum(df['high'] - df['low'], np.maximum(abs(df['high'] - df['close'].shift(1)), abs(df['low'] - df['close'].shift(1))))
    df['atr'] = wma(df['tr'], atr_period)
    df['atr+'] = df['close'] + atr_multiplier * df['atr']
    df['atr-'] = df['close'] - atr_multiplier * df['atr']

    # MA Baseline (HMA)
    df['baseline'] = hma(df['close'], baseline_length)

    # Keltner Channel
    df['true_range'] = df['tr']
    df['range_ma'] = wma(df['true_range'], baseline_length)
    df['keltner_upper'] = df['baseline'] + df['range_ma'] * keltner_multiplier
    df['keltner_lower'] = df['baseline'] - df['range_ma'] * keltner_multiplier

    # MA Baseline Color
    df['baseline_color'] = np.where(
        (df['close'] > df['keltner_upper']), 'buy',
        np.where((df['close'] < df['keltner_lower']), 'sell', 'gray')
    )

    # SSL2 Calculation (JMA) using logic from Pine Script
    df['ssl2_high'] = jma(df['high'], continuation_length)
    df['ssl2_low'] = jma(df['low'], continuation_length)

    # Initialize Hlv2 with NaN
    Hlv2 = np.nan * np.ones(len(df))

    # Calculate Hlv2 according to the Pine Script logic
    for i in range(1, len(df)):
        if df['close'].iloc[i] > df['ssl2_high'].iloc[i]:
            Hlv2[i] = 1
        elif df['close'].iloc[i] < df['ssl2_low'].iloc[i]:
            Hlv2[i] = -1
        else:
            Hlv2[i] = Hlv2[i-1]

    # Calculate ssl2 using Hlv2 logic
    sslDown2 = np.where(Hlv2 < 0, df['ssl2_high'], df['ssl2_low'])
    df['ssl2'] = sslDown2

    # Output
    return {
        'atr+': df['atr+'].tolist(),
        'atr-': df['atr-'].tolist(),
        'ssl2': df['ssl2'].tolist(),
        'color': df['baseline_color'].tolist()
    }

def ssl_signal(symbol , timeframe) :

    ssl = ssl_hybrid(symbol , timeframe)['color'][-2]
    ssl1 = ssl_hybrid(symbol , timeframe)['color'][-3]

    if ssl1 != 'buy' and ssl == 'buy' :
        return 'long'
    elif ssl1 != 'sell' and ssl == 'sell' :
        return 'short'
    else :
        return 'False'

def atr_stop_loss(symbol, timeframe, length, multiplier, type):
    """
    Calculate ATR Stop Loss levels for a given symbol and timeframe.

    Args:
        symbol (str): The symbol to fetch data for (e.g., "EURUSD").
        timeframe (str): The timeframe as a string (e.g., "5m", "1h").
        length (int): The period for ATR calculation.
        multiplier (float): The multiplier for stop loss levels.
        trade_type (str): Either 'long' or 'short', to specify the type of stop loss.

    Returns:
        list: A list of stop loss levels based on the specified trade type.
    """
    # Map string timeframes to MetaTrader 5 timeframes
    timeframe_map = {
        '1m': mt5.TIMEFRAME_M1,
        '3m': mt5.TIMEFRAME_M3,
        '5m': mt5.TIMEFRAME_M5,
        '15m': mt5.TIMEFRAME_M15,
        '30m': mt5.TIMEFRAME_M30,
        '1h': mt5.TIMEFRAME_H1
    }

    # Check if the given timeframe is valid
    if timeframe not in timeframe_map:
        raise ValueError("Invalid timeframe. Valid options are: '1m', '5m', '15m', '30m', '1h'.")

    # Convert string timeframe to MetaTrader 5 timeframe
    mt5_timeframe = timeframe_map[timeframe]

    # Initialize MetaTrader 5 connection
    if not mt5.initialize():
        raise RuntimeError("MetaTrader 5 initialization failed")

    # Get historical data
    rates = mt5.copy_rates_from_pos(symbol, mt5_timeframe, 0, length + 100)
    if rates is None or len(rates) == 0:
        mt5.shutdown()
        raise RuntimeError("Failed to retrieve rates")

    # Convert to DataFrame for easier processing
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')

    # Calculate True Range (TR)
    df['high_low'] = df['high'] - df['low']
    df['high_close'] = abs(df['high'] - df['close'].shift(1))
    df['low_close'] = abs(df['low'] - df['close'].shift(1))
    df['tr'] = df[['high_low', 'high_close', 'low_close']].max(axis=1)

    # Calculate ATR using RMA (Smoothed Moving Average)
    df['atr'] = df['tr'].rolling(window=length, min_periods=1).mean()

    # Calculate upper and lower stop loss levels
    df['upper_stop_loss'] = df['high'] + (df['atr'] * multiplier)
    df['lower_stop_loss'] = df['low'] - (df['atr'] * multiplier)

    # Shutdown MetaTrader 5 connection
    mt5.shutdown()

    # Return the stop loss levels based on trade type
    if type == 'long':
        return df['lower_stop_loss'].tolist()[-1]
    elif type == 'short':
        return df['upper_stop_loss'].tolist()[-1]
    else:
        return False
    
    # EMA indicator calculation
def ema_indicator(data, window):
    return data.ewm(span=window, adjust=False).mean()

# ADX indicator calculation
def adx(high, low, close, window=14):
    plus_dm = high.diff()
    minus_dm = low.diff()
    tr = pd.concat([high - low, abs(high - close.shift(1)), abs(low - close.shift(1))], axis=1).max(axis=1)
    
    plus_di = 100 * (plus_dm.rolling(window).sum() / tr.rolling(window).sum())
    minus_di = 100 * (minus_dm.rolling(window).sum() / tr.rolling(window).sum())
    
    adx = 100 * abs(plus_di - minus_di) / (plus_di + minus_di)
    return adx.rolling(window).mean()

# ATR (Average True Range) calculation
def average_true_range(high, low, close, window=14):
    tr = pd.concat([high - low, abs(high - close.shift(1)), abs(low - close.shift(1))], axis=1).max(axis=1)
    return tr.rolling(window).mean()

# MACD indicator calculation
def macd_diff_indicator(data, fast, slow, signal):
    macd = ema_indicator(data, fast) - ema_indicator(data, slow)
    macd_signal = ema_indicator(macd, signal)
    return macd - macd_signal

def ravand_verify(symbol, timeframe, num_candles=100):
    # Define the parameter settings for each timeframe
    params = {
        '1m': {'atr_threshold': 0.1, 'adx_threshold': 15, 'ema_periods': (7, 14), 'macd_fast': 6, 'macd_slow': 13, 'macd_signal': 4},
        '3m': {'atr_threshold': 0.2, 'adx_threshold': 18, 'ema_periods': (9, 21), 'macd_fast': 9, 'macd_slow': 21, 'macd_signal': 6},
        '5m': {'atr_threshold': 0.3, 'adx_threshold': 20, 'ema_periods': (9, 21), 'macd_fast': 12, 'macd_slow': 26, 'macd_signal': 9},
        '15m': {'atr_threshold': 0.4, 'adx_threshold': 25, 'ema_periods': (9, 21), 'macd_fast': 12, 'macd_slow': 26, 'macd_signal': 9}
    }

    if timeframe not in params:
        raise ValueError("Invalid timeframe! Choose from '1m', '3m', '5m', or '15m'.")
    
    settings = params[timeframe]
    
    # Fetch market data for the given timeframe
    rates = mt5.copy_rates_from_pos(symbol, mt5.TIMEFRAME_M1, 0, num_candles)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    
    # Calculating Heiken Ashi
    df['HA_Close'] = (df['close'] + df['open'] + df['high'] + df['low']) / 4
    df['HA_Open'] = (df['open'].shift(1) + df['close'].shift(1)) / 2
    df['HA_High'] = df[['high', 'HA_Open', 'HA_Close']].max(axis=1)
    df['HA_Low'] = df[['low', 'HA_Open', 'HA_Close']].min(axis=1)

    # Calculating other indicators
    df['EMA_Fast'] = ema_indicator(df['close'], window=settings['ema_periods'][0])
    df['EMA_Slow'] = ema_indicator(df['close'], window=settings['ema_periods'][1])
    df['ADX'] = adx(df['high'], df['low'], df['close'], window=14)
    df['ATR'] = average_true_range(df['high'], df['low'], df['close'], window=14)
    macd_diff = macd_diff_indicator(df['close'], settings['macd_fast'], settings['macd_slow'], settings['macd_signal'])

    # Get the latest values
    latest_adx = df['ADX'].iloc[-1]
    latest_atr = df['ATR'].iloc[-1]
    ema_trend_up = df['EMA_Fast'].iloc[-1] > df['EMA_Slow'].iloc[-1]
    ema_trend_down = df['EMA_Fast'].iloc[-1] < df['EMA_Slow'].iloc[-1]
    
    # Heiken Ashi Trend
    ha_trend_up = df['HA_Close'].iloc[-1] > df['HA_Open'].iloc[-1]
    ha_trend_down = df['HA_Close'].iloc[-1] < df['HA_Open'].iloc[-1]
    
    # Check if market is not ranging (based on ATR and ADX thresholds)
    if latest_atr < settings['atr_threshold'] or latest_adx < settings['adx_threshold']:
        return "hold"
    
    # Long condition
    if ha_trend_up and ema_trend_up and macd_diff.iloc[-1] > 0:
        return "long"
    
    # Short condition
    if ha_trend_down and ema_trend_down and macd_diff.iloc[-1] < 0:
        return "short"
    
    # If no clear signal, return neutral
    return "hold"

def get_candles(symbol, timeframe, limit=100):
    # تبدیل رشته تایم فریم به تایم فریم متاتریدر
    timeframe_map = {
        '1m': mt5.TIMEFRAME_M1,
        '3m': mt5.TIMEFRAME_M3,
        '5m': mt5.TIMEFRAME_M5,
        '15m': mt5.TIMEFRAME_M15
    }
    
    if timeframe not in timeframe_map:
        raise ValueError("Invalid timeframe! Choose from '1m', '3m', '5m', or '15m'.")
    
    mt5_timeframe = timeframe_map[timeframe]
    
    # دریافت کندل‌ها از متاتریدر
    rates = mt5.copy_rates_from_pos(symbol, mt5_timeframe, 0, limit)
    df = pd.DataFrame(rates)
    df['time'] = pd.to_datetime(df['time'], unit='s')
    return df

# تابع برای تشخیص نوع کندل
def get_candle_type(symbol, timeframe, index):
    df = get_candles(symbol, timeframe)
    if df['close'].iloc[index] > df['open'].iloc[index]:
        return 'long'
    elif df['close'].iloc[index] < df['open'].iloc[index]:
        return 'short'
    else:
        return 'doji'

# تابع برای بررسی الگوهای کندل استیک
def candlestick(symbol, timeframe):
    df = get_candles(symbol, timeframe, limit=10)
    
    # Bullish Engulfing (کندل صعودی)
    if df['close'].iloc[-1] > df['open'].iloc[-1] and df['open'].iloc[-2] > df['close'].iloc[-2] and df['close'].iloc[-1] > df['open'].iloc[-2] and df['open'].iloc[-1] < df['close'].iloc[-2]:
        return 'long'
    
    # Bearish Engulfing (کندل نزولی)
    elif df['close'].iloc[-1] < df['open'].iloc[-1] and df['open'].iloc[-2] < df['close'].iloc[-2] and df['close'].iloc[-1] < df['open'].iloc[-2] and df['open'].iloc[-1] > df['close'].iloc[-2]:
        return 'short'
    
    # Doji (کندل دوجی)
    elif abs(df['close'].iloc[-1] - df['open'].iloc[-1]) <= 0.1 * (df['high'].iloc[-1] - df['low'].iloc[-1]):
        return False
    
    # Hammer (کندل چکش)
    elif (df['high'].iloc[-1] - df['low'].iloc[-1] > 3 * (df['open'].iloc[-1] - df['close'].iloc[-1])) and (df['close'].iloc[-1] > df['open'].iloc[-1] or df['close'].iloc[-1] < df['open'].iloc[-1]):
        if df['close'].iloc[-1] > df['open'].iloc[-1]:
            return 'long'
        else:
            return 'short'
    
    # Shooting Star (کندل ستاره دنباله‌دار)
    elif (df['high'].iloc[-1] - df['low'].iloc[-1] > 3 * (df['open'].iloc[-1] - df['close'].iloc[-1])) and (df['close'].iloc[-1] < df['open'].iloc[-1]):
        return 'short'
    
    return False

tokyo = check_time(0 , 8)
londen = check_time(7 , 15)
new_york = check_time(13 , 19)
sydney = check_time(22 , 5)
best_time = check_time(10 , 17)
off_time = check_time(22 , 23)
today = datetime.datetime.today().strftime('%A')
buy = mt5.ORDER_TYPE_BUY
buy_pending = mt5.ORDER_TYPE_BUY_LIMIT
sell = mt5.ORDER_TYPE_SELL
sell_pending = mt5.ORDER_TYPE_SELL_LIMIT
def manage(symbol) :
    x = 0.20
    positions = mt5.positions_get()

    for position in positions :

        p_symbol = position._asdict()['symbol']
        price = mt5.symbol_info_tick(p_symbol).ask if position._asdict()['type'] == 0 else mt5.symbol_info_tick(p_symbol).bid

        if position._asdict()['type'] == 0 :
            
            tp0 = position._asdict()['price_open']
            tp1 = position._asdict()['price_open'] + x
            tp2 = position._asdict()['price_open'] + (x*2)
            tp3 = position._asdict()['price_open'] + (x*3)
            tp4 = position._asdict()['price_open'] + (x*4)
            tp5 = position._asdict()['price_open'] + (x*5)
            tp6 = position._asdict()['price_open'] + (x*6)
            tp7 = position._asdict()['price_open'] + (x*7)
            tp8 = position._asdict()['price_open'] + (x*8)
            tp9 = position._asdict()['price_open'] + (x*9)
            tp10 = position._asdict()['price_open'] + (x*10)
            tp11 = position._asdict()['price_open'] + (x*11)
            tp12 = position._asdict()['price_open'] + (x*12)
            tp13 = position._asdict()['price_open'] + (x*13)
            tp14 = position._asdict()['price_open'] + (x*14)
            tp15 = position._asdict()['price_open'] + (x*15)
            tp16 = position._asdict()['price_open'] + (x*16)
            tp17 = position._asdict()['price_open'] + (x*17)
            tp18 = position._asdict()['price_open'] + (x*18)
            tp19 = position._asdict()['price_open'] + (x*19)
            tp20 = position._asdict()['price_open'] + (x*20)

            tps = [tp0 , tp1 , tp2 , tp3 , tp4 , tp5 , tp6 , tp7 , tp8 , tp9 , tp10 , tp11 , tp12 , tp13 , tp14 , tp15 , tp16 , tp17 , tp18 , tp19 , tp20 ]

            last_smaller = None

            for i in range(len(tps) - 1, -1 , -1) :
                if tps[i] <= price :
                    last_smaller = tps[i]
                    break
            if last_smaller is not None and last_smaller != tp0:
                index = tps.index(last_smaller)
                if index > 0 :
                    previous_value = tps[index - 1]
                    if previous_value > position._asdict()['sl'] :
                        modify_position(position._asdict()['ticket'] , previous_value)

        if position._asdict()['type'] == 1 :

            tp0 = position._asdict()['price_open'] 
            tp1 = position._asdict()['price_open'] - x
            tp2 = position._asdict()['price_open'] - (x*2)
            tp3 = position._asdict()['price_open'] - (x*3)
            tp4 = position._asdict()['price_open'] - (x*4)
            tp5 = position._asdict()['price_open'] - (x*5)
            tp6 = position._asdict()['price_open'] - (x*6)
            tp7 = position._asdict()['price_open'] - (x*7)
            tp8 = position._asdict()['price_open'] - (x*8)
            tp9 = position._asdict()['price_open'] - (x*9)
            tp10 = position._asdict()['price_open'] - (x*10)
            tp11 = position._asdict()['price_open'] - (x*11)
            tp12 = position._asdict()['price_open'] - (x*12)
            tp13 = position._asdict()['price_open'] - (x*13)
            tp14 = position._asdict()['price_open'] - (x*14)
            tp15 = position._asdict()['price_open'] - (x*15)
            tp16 = position._asdict()['price_open'] - (x*16)
            tp17 = position._asdict()['price_open'] - (x*17)
            tp18 = position._asdict()['price_open'] - (x*18)
            tp19 = position._asdict()['price_open'] - (x*19)
            tp20 = position._asdict()['price_open'] - (x*20)

            tps = [tp0 , tp1 , tp2 , tp3 , tp4 , tp5 , tp6 , tp7 , tp8 , tp9 , tp10 , tp11 , tp12 , tp13 , tp14 , tp15 , tp16 , tp17 , tp18 , tp19 , tp20 ]
            
            last_bigger = None
            for i in range(len(tps) - 1, -1, -1):
                if tps[i] >= price:
                    last_bigger = tps[i]
                    break
            if last_bigger is not None and last_bigger != tp0:
                index = tps.index(last_bigger)
                if index > 0:
                    previous_value = tps[index - 1]
                    if previous_value < position._asdict()['sl']:
                        modify_position(position._asdict()['ticket'], previous_value)

#position FVG -----------------------------------------------------------------------------------------------------------------
def fvg_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):

    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and FVG_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def fvg_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):

    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and FVG_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )


#position Sar -----------------------------------------------------------------------------------------------------------------
def sar_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):

    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and sar_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def sar_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):

    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and sar_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

                
#position MACD -----------------------------------------------------------------------------------------------------------------
def macd_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask 
    if total_positons_comment(list) and macd_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def macd_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and (new_york)and macd_verify(symbol , timeframe , 'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )


#position SuperTrend ------------------------------------------------------------------------------------------------------------
def supertrend_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and supertrend_verify(symbol , timeframe , 'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def supertrend_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):  
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'   

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and supertrend_verify(symbol , timeframe , 'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )


#position Nadaraya ------------------------------------------------------------------------------------------------------------
def nadaraya_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot ) :
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and nadaraya_verify(symbol , timeframe , 'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def nadaraya_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot ) :
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and nadaraya_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

        
#position Halftrend ------------------------------------------------------------------------------------------------------------
def halftrend_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and halftrend_verify(symbol , timeframe , 'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def halftrend_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and halftrend_verify(symbol , timeframe , 'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment)


#position Ut Bot --------------------------------------------------------------------------------------------------------------
def utbot_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask 
    if total_positons_comment(list) and utbot_verify(symbol , timeframe , 'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def utbot_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and utbot_verify(symbol , timeframe , 'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position Ichi corss --------------------------------------------------------------------------------------------------------------
def ichi_cross_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask 
    if total_positons_comment(list) and ichi_cross_verify(symbol , timeframe , 'long' ) == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def ichi_cross_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and ichi_cross_verify(symbol , timeframe , 'short ') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )


#position Avrage and RSI ----------------------------------------------------------------------------------------------------------
def avrg_rsi_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask 
    if total_positons_comment(list) and avrg_rsi_verify(symbol , timeframe , 'long' ) == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def avrg_rsi_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'
        
    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and avrg_rsi_verify(symbol , timeframe , 'short ') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl > price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position EMA Cross 50/100 -----------------------------------------------------------------------------------------------------
def ema50_100_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and ema50_100_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def ema50_100_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot): 
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and  ema50_100_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position TSI Cross -----------------------------------------------------------------------------------------------------------------
def tsi_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and tsi_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def tsi_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'
        
    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and tsi_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position Multi Signals -----------------------------------------------------------------------------------------------------------------
def multi_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and multi_signals_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def multi_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'
        
    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and multi_signals_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position ichimoku 2 -----------------------------------------------------------------------------------------------------------------
def ichi2_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and ichi2_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def ichi2_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'
        
    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and ichi2_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position ichimoku pro -----------------------------------------------------------------------------------------------------------------
def ichipro_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and ichipro_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def ichipro_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'
        
    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and ichipro_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

#position ssl -----------------------------------------------------------------------------------------------------------------
def ssl_long_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'

    price = mt5.symbol_info_tick(symbol).ask
    if total_positons_comment(list) and ssl_verify(symbol , timeframe ,'long') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price - sl_vol
        tp = price + tp_vol
        if sl < price:
            create_order(symbol , Lot , buy , price , sl , tp , comment )

def ssl_short_stg(symbol , timeframe , comment , list , tp_vol , sl_vol , Lot):
    if timeframe == '1m':
        comment += '1m'
    if timeframe == '3m':
        comment += '3m'
    if timeframe == '5m':
        comment += '5m'
    if timeframe == '15m':
        comment += '15m'
    if timeframe == '30m':
        comment += '30m'
    if timeframe == '1h':
        comment += '1h'
        
    price = mt5.symbol_info_tick(symbol).bid
    if total_positons_comment(list) and ssl_verify(symbol , timeframe ,'short') == True and position_time_check(symbol , comment , timeframe , 2) == True :
        sl = price + sl_vol
        tp = price - tp_vol
        if sl < price:
            create_order(symbol , Lot , sell , price , sl , tp , comment )

def nadaraya_verify(symbol , timeframe , type):

    if nadaraya_signals(symbol , timeframe , 2 , 8)[-1] == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    if nadaraya_signals(symbol , timeframe , 2 , 8)[-1] == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    else:
        return False
    
def halftrend_verify(symbol , timeframe , type):

    if half_trend_signal(symbol , timeframe , 2 , 2) == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    if half_trend_signal(symbol , timeframe , 2 , 2) == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    else:
        return False

def utbot_verify(symbol , timeframe ,type):

    if ut_bot(symbol , timeframe , 2 , 1)[-1] == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    if ut_bot(symbol , timeframe , 2 , 1)[-1] == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    else:
        return False
    
def supertrend_verify(symbol , timeframe , type):

    if supertrend_signal(symbol , timeframe , 9 , 3.9) == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    if supertrend_signal(symbol , timeframe , 9 , 3.9) == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    else:
        return False
    
def macd_verify(symbol , timeframe , type):
    
    if cross_macd(symbol , timeframe , 12 , 26 , 9) == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    if cross_macd(symbol , timeframe , 12 , 26 , 9) == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    else :
        return False

def FVG_verify(symbol , timeframe , type) :
    if fvg(symbol , timeframe) == True and confirmation(symbol , timeframe , 'short') == True and type == 'short':
        return True
    if fvg(symbol , timeframe) == True and confirmation(symbol , timeframe , 'long') == True and type == 'long':
        return True
    else :
        return False

def ichi_cross_verify(symbol , timeframe , type) :
    price1 = mt5.symbol_info_tick(symbol).ask
    if ichi_cross(symbol, timeframe , 4 , 9 ) == 'down to up' and confirmation(symbol , timeframe , 'long') == True and price1 > average100(timeframe , symbol) and kijun_sen(symbol , timeframe , 52) < price1 and type == 'long':
        return True
    price2 = mt5.symbol_info_tick(symbol).bid
    if ichi_cross(symbol, timeframe , 4 , 9 ) == 'up to down' and confirmation(symbol , timeframe , 'short') == True and  price2 < average100(timeframe , symbol) and kijun_sen(symbol , timeframe , 52) > price2 and type == 'short':
        return True
    else :
        return False
    
def avrg_rsi_verify(symbol , timeframe , type) :
    price1 = mt5.symbol_info_tick(symbol).ask
    if average200(timeframe , symbol) > average50(timeframe , symbol) and  average50(timeframe , symbol) > price1 and whatKandel(timeframe , -2 , symbol) == 'short' and whatKandel(timeframe , -4 , symbol) == 'short' and whatKandel(timeframe , -3 , symbol) == 'short' and whatKandel(timeframe , -1 , symbol) == 'long' and body(timeframe , -1 , symbol) > body(timeframe , -2 , symbol) and type == 'long' :
        return True
    price2 = mt5.symbol_info_tick(symbol).bid
    if average200(timeframe , symbol) < average50(timeframe , symbol) and  average50(timeframe , symbol) < price2 and whatKandel(timeframe , -2 , symbol) == 'long' and whatKandel(timeframe , -4 , symbol) == 'long' and whatKandel(timeframe , -3 , symbol) == 'long' and whatKandel(timeframe , -1 , symbol) == 'short' and body(timeframe , -1 , symbol) > body(timeframe , -2 , symbol) and type == 'short' :
        return True
    else :
        return False
        
def ema50_100_verify(symbol , timeframe , type):
    if ema_cross(timeframe , symbol  , 50 , 100 )== 'down to up' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    if ema_cross(timeframe , symbol , 50 , 100 )== 'up to down' and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    else : 
        False

def tsi_verify(symbol , timeframe , type):
    Tsi_val = tsi(symbol , timeframe , 'tsi' , 25 ,13 , 13)
    Signal_val = tsi(symbol , timeframe , 'signal' , 25 ,13 , 13)
    if Tsi_val[-1] > Signal_val[-1] and Tsi_val[-2] >= Signal_val[-2] and Tsi_val[-3] < Signal_val[-3] and Tsi_val[-4] < Signal_val[-2] and Tsi_val[-5] < Signal_val[-2] and confirmation(symbol , timeframe , 'long') == True and type =='long' :
        return True
    if Tsi_val[-1] < Signal_val[-1] and Tsi_val[-2] <= Signal_val[-2] and Tsi_val[-3] > Signal_val[-3] and Tsi_val[-4] > Signal_val[-2] and Tsi_val[-5] > Signal_val[-2] and confirmation(symbol , timeframe , 'short') == True and type == 'short' :
        return True
    else:
        return False
    
def sar_verify(symbol , timeframe , type):
    if sar_signal(symbol , timeframe , 0.02 , 0.2) == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long' :
        return True
    if sar_signal(symbol , timeframe , 0.02 , 0.2) == 'short' and confirmation(symbol , timeframe , 'short') == True and  type == 'short' :
        return True
    else:
        return False
    
def multi_signals(symbol, timeframe ):# wma, macd, atr, cci, dema, tema
    ohlc = kandel(timeframe, 100, symbol) 
    prices = pd.DataFrame(ohlc[:])

    wma_short = prices['close'].rolling(window=10).apply(lambda x: np.average(x, weights=np.arange(1, len(x) + 1)), raw=True)
    wma_long = prices['close'].rolling(window=50).apply(lambda x: np.average(x, weights=np.arange(1, len(x) + 1)), raw=True)
    macd_values = macd(symbol, timeframe, 'macd')
    signal_values = macd(symbol, timeframe, 'signal')
    if not isinstance(macd_values, list) or not isinstance(signal_values, list):
        raise ValueError("MACD یا خط سیگنال باید یک لیست باشد")
    typical_price = (prices['high'] + prices['low'] + prices['close']) / 3
    cci = (typical_price - typical_price.rolling(window=20).mean()) / (0.015 * typical_price.rolling(window=20).std())
    ewm_10 = prices['close'].ewm(span=10, adjust=False).mean()
    dema = 2 * ewm_10 - ewm_10.ewm(span=10, adjust=False).mean()
    ewm_10_second = ewm_10.ewm(span=10, adjust=False).mean()
    tema = 3 * ewm_10 - 3 * ewm_10_second + ewm_10_second.ewm(span=10, adjust=False).mean()

    signals = []
    for i in range(1, len(prices)):
        signal = None
        if wma_short[i] > wma_long[i] and wma_short[i-1] <= wma_long[i-1]:
            signal = 'long'
        elif wma_short[i] < wma_long[i] and wma_short[i-1] >= wma_long[i-1]:
            signal = 'short'
        if macd_values[i] > signal_values[i] and macd_values[i-1] <= signal_values[i-1]:
            signal = 'long'
        elif macd_values[i] < signal_values[i] and macd_values[i-1] >= signal_values[i-1]:
            signal = 'short'
        if cci[i] > 100 and cci[i-1] <= 100:
            signal = 'long'
        elif cci[i] < -100 and cci[i-1] >= -100:
            signal = 'short'
        if dema[i] > dema[i-1]:
            signal = 'long'
        elif dema[i] < dema[i-1]:
            signal = 'short'
        if tema[i] > tema[i-1]:
            signal = 'long'
        elif tema[i] < tema[i-1]:
            signal = 'short'
            signals.append(signal)
    return signals[-1]

def multi_signals_verify(symbol, timeframe , type):

    if (multi_signals(symbol, timeframe) == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long') == True :
        return True
    if (multi_signals(symbol, timeframe) == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short') == True :
        return True
    else :
        return False
    
def ichi2_verify(symbol, timeframe , type):
    if (ichimoku_signals(symbol , timeframe , 9 , 26 , 52 , 26 , 26)[-1] == 'buy' and confirmation(symbol , timeframe , 'long') == True and type == 'long') == True :
        return True
    if (ichimoku_signals(symbol , timeframe , 9 , 26 , 52 , 26 , 26)[-1] == 'sell' and confirmation(symbol , timeframe , 'short') == True and type == 'short') == True :
        return True
    else:
        return False

def ichipro_verify(symbol, timeframe , type):
    if (ichimoku_signals_pro(symbol , timeframe , 9 , 26 , 52 , 26 , 26)[-1] == 'buy' and confirmation(symbol , timeframe , 'long') == True and type == 'long') == True :
        return True
    if (ichimoku_signals_pro(symbol , timeframe , 9 , 26 , 52 , 26 , 26)[-1] == 'sell' and confirmation(symbol , timeframe , 'short') == True and type == 'short') == True :
        return True
    else:
        return False
    
def ssl_verify(symbol, timeframe , type):
    if (ssl_signal(symbol , timeframe) == 'long' and confirmation(symbol , timeframe , 'long') == True and type == 'long') == True :
        return True
    if (ssl_signal(symbol , timeframe) == 'short' and confirmation(symbol , timeframe , 'short') == True and type == 'short') == True :
        return True
    else:
        return False


nada_var = ['nadalong1m',
            'nadalong3m',
            'nadalong5m',
            'nadalong15m',
            'nadalong30m',
            'nadalong1h',
            'nadashort1m',
            'nadashort3m',
            'nadashort5m',
            'nadashort15m',
            'nadashort30m',
            'nadashort1h',]

ssl_var = ['ssllong1m',
            'ssllong3m',
            'ssllong5m',
            'ssllong15m',
            'ssllong30m',
            'ssllong1h',
            'sslshort1m',
            'sslshort3m',
            'sslshort5m',
            'sslshort15m',
            'sslshort30m',
            'sslshort1h',]

fvg_var = [ 'fvglong1m',
            'fvgshort1m',
            'fvglong3m',
            'fvgshort3m',
            'fvglong5m',
            'fvgshort5m',
            'fvglong15m',
            'fvgshort15m',
            'fvglong30m',
            'fvgshort30m',
            'fvglong1h',
            'fvgshort1h',]

sar_var = [ 'sarlong1m',
            'sarshort1m',
            'sarlong3m',
            'sarshort3m',
            'sarlong5m',
            'sarshort5m',
            'sarlong15m',
            'sarshort15m',
            'sarlong30m',
            'sarshort30m',
            'sarlong1h',
            'sarshort1h',]
            
supert_var = ['supertlong1m',
            'supertshort1m',
            'supertlong3m',
            'supertshort3m',
            'supertlong5m',
            'supertshort5m',
            'supertlong15m',
            'supertshort15m',
            'supertlong30m',
            'supertshort30m',
            'supertlong1h',
            'supertshort1h',]

avrg_var = ['avrg_long1m',
            'avrg_short1m',
            'avrg_long3m',
            'avrg_short3m',
            'avrg_long5m',
            'avrg_short5m',
            'avrg_long15m',
            'avrg_short15m',
            'avrg_long30m',
            'avrg_short30m',
            'avrg_long1h',
            'avrg_short1h',]

ichi_var = ['ichi_long1m',
            'ichi_short1m',
            'ichi_long3m',
            'ichi_short3m',
            'ichi_long5m',
            'ichi_short5m',
            'ichi_long15m',
            'ichi_short15m',
            'ichi_long30m',
            'ichi_short30m',
            'ichi_long1h',
            'ichi_short1h',]

half_var = ['halflong1m',
            'halfshort1m',
            'halflong3m',
            'halfshort3m',
            'halflong5m',
            'halfshort5m',
            'halflong15m',
            'halfshort15m',
            'halflong30m',
            'halfshort30m',
            'halflong1h',
            'halfshort1h',]

utbot_var = ['utbotlong1m',
            'utbotshort1m',
            'utbotlong3m',
            'utbotshort3m',
            'utbotlong5m',
            'utbotshort5m',
            'utbotlong15m',
            'utbotshort15m',
            'utbotlong30m',
            'utbotshort30m',
            'utbotshort1h',
            'utbotlong1h',]

ema_var = ['ema100_long1m',
            'ema100_short1m',
            'ema100_long3m',
            'ema100_short3m',
            'ema100_long5m',
            'ema100_short5m',
            'ema100_short15m',
            'ema100_long15m',
            'ema100_short30m',
            'ema100_long30m',
            'ema100_short1h',
            'ema100_long1h',]

macd_var = ['macdlong1m',
            'macdshort1m',
            'macdlong3m',
            'macdshort3m',
            'macdlong5m',
            'macdshort5m',
            'macdlong15m',
            'macdshort15m',
            'macdlong30m',
            'macdshort30m',
            'macdlong1h',
            'macdshort1h',]

tsi_var = [ 'tsishort1m',
            'tsilong1m',
            'tsishort3m',
            'tsilong3m',
            'tsilong5m',
            'tsishort5m',
            'tsilong15m',
            'tsishort15m',
            'tsilong30m',
            'tsishort30m',
            'tsilong1h',
            'tsishort1h',]

multi_var = ['multilong1m',
            'multilong3m',
            'multilong5m',
            'multilong15m',
            'multilong30m',
            'multilong1h',
            'multishort1m',
            'multishort3m',
            'multishort5m',
            'multishort15m',
            'multishort30m',
            'multishort1h',]

ichi2_var = ['ichi2long1m',
            'ichi2long3m',
            'ichi2long5m',
            'ichi2long15m',
            'ichi2long30m',
            'ichi2long1h',
            'ichi2short1m',
            'ichi2short3m',
            'ichi2short5m',
            'ichi2short15m',
            'ichi2short30m',
            'ichi2short1h',]

ichip_var = ['ichiplong1m',
            'ichiplong3m',
            'ichiplong5m',
            'ichiplong15m',
            'ichiplong30m',
            'ichiplong1h',
            'ichipshort1m',
            'ichipshort3m',
            'ichipshort5m',
            'ichipshort15m',
            'ichipshort30m',
            'ichipshort1h',]

mt5.initialize()
symbols = ['XAUUSD.']
timeframes = ['1m' , '3m' , '5m' , '15m']
while True :

    for i in symbols:
        verify_news = 1
        nada_ver = 1
        supert_ver = 1
        ema_ver = 1
        ichi_ver = 1
        macd_ver = 1
        avrg_ver = 1
        fvg_ver = 1
        half_ver = 1
        sar_ver = 1
        multi_ver = 1
        ichi2_ver = 1
        ichipro_ver = 1
        ssl_ver = 1

        symbol = f'{i}'
        
        #Start/Stop STG -------------------------------------------------------------------------------------------------------

        if (profit_today()+profit()) >= 25 :
            close_all_positions()
            raise Exception
        if is_news() :
            verify_news = 0

        #Smart TP/SL ------------------------------------------------------------------------------------------------------------ 
        manage(symbol) 
        #Positions ------------------------------------------------------------------------------------------------------------
        if verify_news == 1 :
            for timeframe in timeframes:
                sl_vol = 5.00
                tp_vol = ((atr(symbol,timeframe,14)[-1])*2)
                Lot = 0.05
        # XAUUSD --------------------------------------------------------------------------------------------------------------
                #long
                # if nada_ver == 1 :
                #     nadaraya_long_stg(symbol , timeframe , 'nadalong' , nada_var , tp_vol , sl_vol , Lot)
                if supert_ver == 1 :
                    supertrend_long_stg(symbol , timeframe , 'supertlong' , supert_var , tp_vol , sl_vol , Lot)
                if ema_ver == 1 :
                    ema50_100_long_stg(symbol , timeframe , 'ema100_long' , ema_var , tp_vol , sl_vol , Lot)
                if ichi_ver == 1 :
                    ichi_cross_long_stg(symbol , timeframe , 'ichi_long' , ichi_var , tp_vol , sl_vol , Lot)
                if ichi2_ver == 1 :
                    ichi2_long_stg(symbol , timeframe , 'ichi2long' , ichi2_var , tp_vol , sl_vol , Lot)
                if ichipro_ver == 1 :
                    ichipro_long_stg(symbol , timeframe , 'ichiplong' , ichip_var , tp_vol , sl_vol , Lot)
                if macd_ver == 1 :
                    macd_long_stg(symbol , timeframe , 'macdlong' , macd_var , tp_vol , sl_vol , Lot)
                if half_ver == 1 :
                    halftrend_long_stg(symbol , timeframe , 'halflong' , half_var , tp_vol , sl_vol , Lot)
                if avrg_ver == 1 :
                    avrg_rsi_long_stg(symbol , timeframe , 'avrg_long' , avrg_var , tp_vol , sl_vol , Lot)
                if sar_ver == 1 :
                    sar_long_stg(symbol , timeframe , 'sarlong' , sar_var , tp_vol , sl_vol , Lot)
                # if fvg_ver == 1 :
                #     fvg_long_stg(symbol , timeframe , 'fvglong' , fvg_var , tp_vol , sl_vol , Lot)
                if multi_ver == 1 :
                    multi_long_stg(symbol , timeframe , 'multilong' , tsi_var , tp_vol , sl_vol , Lot)
                if ssl_ver == 1 :
                    ssl_long_stg(symbol , timeframe , 'ssllong' , ssl_var , tp_vol , sl_vol , Lot)

                #short
                # if nada_ver == 1 :
                #     nadaraya_short_stg(symbol , timeframe , 'nadashort' , nada_var , tp_vol , sl_vol , Lot)
                if supert_ver == 1 :
                    supertrend_short_stg(symbol , timeframe , 'supertshort' , supert_var , tp_vol , sl_vol , Lot)
                if ema_ver == 1 :
                    ema50_100_short_stg(symbol , timeframe , 'ema100_short' , ema_var , tp_vol , sl_vol , Lot)
                if ichi_ver == 1 :
                    ichi_cross_short_stg(symbol , timeframe , 'ichi_short' , ichi_var , tp_vol , sl_vol , Lot)
                if ichi2_ver == 1 :
                    ichi2_short_stg(symbol , timeframe , 'ichi2short' , ichi2_var , tp_vol , sl_vol , Lot)
                if ichipro_ver == 1 :
                    ichipro_short_stg(symbol , timeframe , 'ichipshort' , ichip_var , tp_vol , sl_vol , Lot)
                if sar_ver == 1 :
                    sar_short_stg(symbol , timeframe , 'sarshort' , sar_var , tp_vol , sl_vol , Lot)
                if macd_ver == 1 :
                    macd_short_stg(symbol , timeframe , 'macdshort' , macd_var , tp_vol , sl_vol , Lot)
                if half_ver == 1 :
                    halftrend_short_stg(symbol , timeframe , 'halfshort' , half_var , tp_vol , sl_vol , Lot)
                if avrg_ver == 1 :
                    avrg_rsi_short_stg(symbol , timeframe , 'avrg_short' , avrg_var , tp_vol , sl_vol , Lot)
                # if fvg_ver == 1 :
                #     fvg_short_stg(symbol , timeframe , 'fvgshort' , fvg_var , tp_vol , sl_vol , Lot)
                if multi_ver == 1 :
                    multi_short_stg(symbol , timeframe , 'multishort' , tsi_var , tp_vol , sl_vol , Lot)
                if ssl_ver == 1 :
                    ssl_short_stg(symbol , timeframe , 'sslshort' , ssl_var , tp_vol , sl_vol , Lot)

