import signal
import requests
from time import sleep

# Define a class for custom exceptions
class ApiException(Exception):
    pass

# Define the signal handler function
def signal_handler(signum, frame):
    global shutdown
    signal.signal(signal.SIGINT, signal.SIG_DFL)
    shutdown = False

# set your API key to authenticate to the RIT client
API_KEY = {'X-API-Key': '1J7HXV8J'}
SPREAD = 0.02
BUY_VOLUME = 5000  
SELL_VOLUME = 5000
POSITION_LIMIT = 25000
Target_Security = 'ALGO'
sleep_value = 0.1
orders = []  # Initialize the 'orders' list

# this helper method returns the current 'tick' of the running case
def get_tick(session):
    resp = session.get('http://localhost:9999/v1/case')
    if resp.status_code == 401:
        raise ApiException('Input correct API Key')
    case = resp.json()
    return case['tick']

# this helper method returns the last close price for the given security, one tick ago
def ticker_close(session, ticker):
    payload = {'ticker': ticker, 'limit': 1}
    resp = session.get('http://localhost:9999/v1/securities/history', params=payload)
    if resp.status_code == 401:
        raise ApiException('Input correct API Key')
    ticker_history = resp.json()
    if ticker_history:
        return ticker_history[0]['close']
    else:
        raise ApiException('Response error. Unexpected JSON response.')

# this helper method gets all the orders of a given type (OPEN/TRANSACTED/CANCELLED)
def get_orders(session, status):
    payload = {'status': status}
    resp = session.get('http://localhost:9999/v1/orders',params=payload)
    if resp.status_code == 401:
        raise ApiException('Input correct API Key')
    orders = resp.json()
    return orders

#Function to get the position held of the security
def get_position(s, security):
    response = s.get('http://localhost:9999/v1/securities')
    securities = response.json()
    for sec in securities:
        if sec['ticker'] == security:
            return sec['position']
    return 0


# this helper method submits a pair of limit orders to buy and sell VOLUME of each security, at the last price +/- SPREAD
def buy_sell(session, to_buy, to_sell, last):
    # Get the current positions
    position = get_position(session, to_buy)
    
    # Calculate the quantity for the orders
    if position <= 20000:
        quantity_to_buy = BUY_VOLUME
    else:
        quantity_to_buy = POSITION_LIMIT - abs(position)

    if position >= -20000:
        quantity_to_sell = SELL_VOLUME
    else:
        quantity_to_sell = POSITION_LIMIT - abs(position)

    # If the position to buy is equal to or less than -25000, only place a buy order
    if position <= -24999:
        buy_payload = {'ticker': to_buy, 'type': 'LIMIT', 'quantity': quantity_to_buy, 'action': 'BUY', 'price': last - SPREAD}
        session.post('http://localhost:9999/v1/orders', params=buy_payload)
    # If the position to sell is equal to or greater than 25000, only place a sell order
    elif position >= 24999:
        sell_payload = {'ticker': to_sell, 'type': 'LIMIT', 'quantity': quantity_to_sell, 'action': 'SELL', 'price': last + SPREAD}
        session.post('http://localhost:9999/v1/orders', params=sell_payload)
    # Otherwise, place both orders
    else:
        buy_payload = {'ticker': to_buy, 'type': 'LIMIT', 'quantity': quantity_to_buy, 'action': 'BUY', 'price': last - SPREAD}
        sell_payload = {'ticker': to_sell, 'type': 'LIMIT', 'quantity': quantity_to_sell, 'action': 'SELL', 'price': last + SPREAD}
        session.post('http://localhost:9999/v1/orders', params=buy_payload)
        session.post('http://localhost:9999/v1/orders', params=sell_payload)

# Define the main function
def main():
    # creates a session to manage connections and requests to the RIT Client
    with requests.Session() as s:
        # add the API key to the session to authenticate during requests
        s.headers.update(API_KEY)
        # get the current time of the case
        tick = get_tick(s)

        # while the time is between 5 and 295, do the following
        while tick > 5 and tick < 295:

            # get the open order book and security last tick's close price
            orders = get_orders(s, 'OPEN')
            security_close = ticker_close(s, Target_Security) 
            
            # Update the position
            position = get_position(s, Target_Security) 

            # Calculate the potential position after executing all buy and sell orders
            potential_buy_position = position + sum(order['quantity'] for order in orders if order['action'] == 'BUY')
            potential_sell_position = position - sum(order['quantity'] for order in orders if order['action'] == 'SELL')

            # Delete buy orders if potential position is greater than 19999
            if potential_buy_position > 19999:
                for order in orders:
                    if order['action'] == 'BUY':
                        s.delete('http://localhost:9999/v1/orders/{}'.format(order['order_id']))
                        sleep(sleep_value)

            # Delete sell orders if potential position is less than -19999
            if potential_sell_position < -19999:
                for order in orders:
                    if order['action'] == 'SELL':
                        s.delete('http://localhost:9999/v1/orders/{}'.format(order['order_id']))
                        sleep(sleep_value)

             # delete buy orders if position is greater than 19999
            if position > 19999:
                for order in orders:
                    if order['action'] == 'BUY':
                        s.delete('http://localhost:9999/v1/orders/{}'.format(order['order_id']))
                        sleep(sleep_value)

            # delete sell orders if position is less than -19999
            if position < -19999:
                for order in orders:
                    if order['action'] == 'SELL':
                        s.delete('http://localhost:9999/v1/orders/{}'.format(order['order_id']))
                        sleep(sleep_value)
                        
            # delete orders older than 20 ticks
            orders = get_orders(s, 'OPEN')
            for order in orders:
                if order['tick'] < tick - 20:
                    s.delete('http://localhost:9999/v1/orders/{}'.format(order['order_id']))
                    sleep(sleep_value)

            # update the orders and num_orders
            orders = get_orders(s, 'OPEN')

            # Separate buy and sell orders
            buy_orders = [order for order in orders if order['action'] == 'BUY']
            sell_orders = [order for order in orders if order['action'] == 'SELL']

            # Check if buy orders is not empty
            if buy_orders:
                while len(buy_orders) > 3:
                    # delete the oldest buy order
                    oldest_order = min(buy_orders, key=lambda order: order['tick'])
                    s.delete('http://localhost:9999/v1/orders/{}'.format(oldest_order['order_id']))
                    sleep(sleep_value)

                    # update the buy orders
                    buy_orders = [order for order in get_orders(s, 'OPEN') if order['action'] == 'BUY']

            # Check if sell orders is not empty
            if sell_orders:
                while len(sell_orders) > 3:
                    # delete the oldest sell order
                    oldest_order = min(sell_orders, key=lambda order: order['tick'])
                    s.delete('http://localhost:9999/v1/orders/{}'.format(oldest_order['order_id']))
                    sleep(sleep_value)

                    # update the sell orders
                    sell_orders = [order for order in get_orders(s, 'OPEN') if order['action'] == 'SELL']

            # check if you have less than 9 open orders
            if len(orders) < 9:

                # Submit a pair of orders and update your order book
                buy_sell(s, Target_Security, Target_Security, security_close)
                orders = get_orders(s, 'OPEN')
                sleep(sleep_value)
                num_orders = len(orders)

            # refresh the case time. THIS IS IMPORTANT FOR THE WHILE LOOP
            tick = get_tick(s)
         
if __name__ == '__main__':
    signal.signal(signal.SIGINT, signal_handler)
    main()
