# Concurrent Scalping Algo
This algorithm utilizes a scalping strategy, which focuses on taking advantage of small price movements within short time frames. 
The algorithm operates based on the 20-minute simple moving average crossover, a popular technical indicator for trend identification.
 This strategy assumes that the market is experiencing an upward momentum when the crossover occurs, prompting a buy.

Below are a group of related links for term defenitions

[scalping trading](https://www.investopedia.com/articles/trading/05/scalping.asp) 
[Alpaca API](https://alpaca.markets). 
This algorithm uses real time order updates as well as minute level bar streaming from Polygon via Websockets (see the
[document](https://docs.alpaca.markets/market-data/#consolidated-market-data) for
Polygon data access).
One of the contributions of this example is to demonstrate how to handle
multiple stocks concurrently as independent routine using Python's
[asyncio](https://docs.python.org/3/library/asyncio.html).

The strategy holds positions for very short period and exits positions quickly, so
you have to have more than $25k equity in your account due to the Pattern Day Trader rule,
to run this example. For more information about PDT rule, please read the
[document](https://support.alpaca.markets/hc/en-us/articles/360012203032-Pattern-Day-Trader).


## Dependency
This script needs latest [Alpaca Python SDK](https://github.com/alpacahq/alpaca-trade-api-python).
Please install it using pip

```sh
$ pip3 install alpaca-trade-api
```

or use [pipenv](https://github.com/pypa/pipenv) using `Pipfile` in this directory.

```sh
$ pipenv install
```

## Usage

```sh
$ python main.py --lot=2000 TSLA FB AAPL
```

You can specify as many symbols as you want.  The script is designed to kick off while market
is open. Nothing would happen until 21 minutes from the market open as it relies on the
simple moving average as the buy signal.


# Strategy
This algorithm aims to buy stocks when a 20-minute moving average crossover occurs, signaling potential upward momentum. 
It buys as much stock as the lot value allows and immediately places a limit sell order at or above the entry price to avoid slippage. 
If the buy order isn't filled within 2 minutes, it is canceled, assuming the signal has lost relevance. The sell order remains open indefinitely,
 though this may result in losses if the market turns. All positions are liquidated with a market order before the market closes (3:55 PM ET).

The buy signal is recalculated with every new minute bar, which arrives around 4 seconds after each minute's start (based on Polygon's data feed behavior).

# Why the 20-Minute Moving Average?
The 20-minute moving average is used to capture trends over a short period without being overly reactive to temporary price fluctuations.
This makes it ideal for short-term trading strategies like scalping, where timing is crucial to maximize profit potential.

# Risk Management
Effective risk management is vital in high-frequency strategies like scalping. Users can extend the base code to add stop-loss and take-profit mechanisms.

## Implementation
This example heavily relies on Python's asyncio. Although the thread is single, we handle
multiple symbols concurrently using this async loop.

We keep track of each symbol state in a separate `ScalpAlgo` class instance. That way,
everything stays simple without complex data structure and easy to read. The `main()`
function creates the algo instance for each symbol and creates streaming object
to listen the bar events. As soon as we receive a minute bar, we invoke event handler
for each symbol.

The main routine also starts a period check routine to do some work in background every 30 seconds.
In this background task, we check market state with the clock API and liquidate positions
before the market closes.

### Algo Instance and State Management
Each algo instance initializes its state by fetching day's bar data so far and position/order
from Alpaca API to synchronize, in case the script restarts after some trades. There are
four internal states and transitions as events happen.

- `TO_BUY`: no position, no order. Can transition to `BUY_SUBMITTED`
- `BUY_SUBMITTED`: buy order has been submitted. Can transition to `TO_BUY` or `TO_SELL`
- `TO_SELL`: buy is filled and holding position. Can transition to `SELL_SUBMITTED`
- `SELL_SUBMITTED`: sell order has been submitted. Can transition to `TO_SELL` or `TO_BUY`
- Stop-Loss and Take-Profit
- Stop-Loss: Automatically exits a position if the stock price drops a certain percentage below the entry price, minimizing potential losses.
- Take-Profit: Locks in gains by selling once the price rises above a predefined level.

stop_loss_percentage = 0.1  # 10% below the entry price
take_profit_percentage = 0.2  # 20% above the entry price


### Event Handlers
`on_bar()` is an event handler for the bar data. Here we calculate signal that triggers
a buy order in the `TO_BUY` state. Once order is submitted, it goes to the `BUY_SUBMITTED`
state.

If order is filled, `on_order_update()` handler is called with `event=fill`. The state
transitions to `TO_SELL` and immediately submits a sell order, to transition to the
`SELL_SUBMITTED` state.

Orders may be canceled or rejected (caused by this script or you manually cancel them
from the dashboard). In these cases, the state transitions to `TO_BUY` (if not holding
a position) or `TO_SELL` (if holding a position) and wait for the next events.

`checkup()` method is the background periodic job to check several conditions, where
we cancel open orders and sends market sell order if there is an open position.

It exits once the market closes.

### Note
Each algo instance owns its child logger, prefixed by the symbol name. The console
log is also emitted to a file `console.log` under the same directory for your later review.

Again, the beautify of this code is that there is no multithread code but each
algo instance can focus on the bar/order/position data only for its own. It still
handles multiple symbols concurrently plus runs background periodic job in the
same async loop.

The trick to run additional async routine is as follows.

```py
    loop = stream.loop
    loop.run_until_complete(asyncio.gather(
        stream.subscribe(channels),
        periodic(),
    ))
    loop.close()
```

We use `asyncio.gather()` to run all bar handler, order update handler and periodic job
in one async loop indifinitely. You can kill it by `Ctrl+C`.

# Advanced Strategy Customization
This algorithm is highly customizable, allowing for advanced users to adapt it for different market conditions. Below are some examples of how you can modify the code:

Example Configuration
You can modify the scalping algorithm by adjusting the following parameters:Instead of using this buy signal of 20 minute simple moving average cross over, you can
use your own buy signal. To do so, extend the `ScalpAlgo` class and write your own
`_calc_buy_signal()` method.

```py
    class MyScalpAlgo(ScalpAlgo):
        def _calculate_buy_signal(self):
            '''self._bars has all minute bars in the session so far. Return True to
            trigger buy order'''
            pass
```

moving_average_window = 20  # Adjust for a different time period
stop_loss_percentage = 0.1  # Stop loss threshold
take_profit_percentage = 0.2  # Take profit threshold

And use it instead of the original class.


# Using Alternative Indicators
Instead of relying solely on the 20-minute moving average, users can enhance the strategy by incorporating other indicators such as:

Relative Strength Index (RSI): Detect overbought or oversold conditions.
MACD (Moving Average Convergence Divergence): Identify bullish or bearish momentum.
For example, to add an RSI-based buy signal:

def _calculate_buy_signal(self):
    rsi_value = calculate_rsi(self._bars)
    if rsi_value < 30:  # Buy when RSI is below 30 (oversold)
        return True
    return False


# Performance & Troubleshooting
Common Installation Issues
Asyncio Errors: Ensure you are using Python 3.7 or newer and all dependencies are properly installed using:

pip install -r requirements.txt

Alpaca API Issues: Verify your Alpaca API keys and ensure they have the correct permissions.


# API Rate Limits and Concurrency
The algorithm uses Python’s asyncio to handle multiple stock streams concurrently. However, it's important to monitor Alpaca’s API rate limits to avoid throttling. Running the script with too many symbols or over a poor network connection could degrade performance.

