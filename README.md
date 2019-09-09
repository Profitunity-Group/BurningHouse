# BurningHouse
This repository contains the precious code for Burning House sensor. Written in python 2.7, requires numpy and scipy.


```python
import math
from copy import deepcopy

import numpy as np
from scipy.stats import linregress


class BurningHouseSensor:

    # noinspection SpellCheckingInspection,PyPep8Naming
    def __init__(self, klines_list_1440_close):
        """"
        WORKING
        BurningHouse Sensor Type OneMinutes:
        NOTE: This is a sensor that is well suited for day trading,
        for example, in crypto market, the sensor senses the chaos moment
        in the market and returns some useful info for trading bot to trade
        the ticker
        CAPABILITY: From 0.02 BTC with 10x to 20x Leverage to up to 20 BTC
        :param klines_list_1440_close: 1440 tickers info, example
        [{
            "close": "15680",
            "high": "15690",
            "low": "15670",
            "volume": "3"
        }]
        :exception: RuntimeError: When klines_list is not a list, a RuntimeError will raise
        :exception: ValueError: When some element in the k_line_list does not contain a "close" key
        :exception: AttributeError: When klines_list_1440_close length is not equal to 1440
        WARNING: WE DONT CHECK WHETHER THE LIST IS SORTED :) It's your duty.
        """
        if not isinstance(klines_list_1440_close, list):
            raise RuntimeError("klines_list is not a list")
        if not 1430 <= len(klines_list_1440_close) <= 1450:
            raise AttributeError("kline_list length is not 1440")
        for i in klines_list_1440_close:
            if "close" not in i:
                raise ValueError("Some element in kline_list does not contain \" close \" key")
        for i in klines_list_1440_close:
            if "high" not in i:
                raise ValueError("Some element in kline_list does not contain \" high \" key")
        for i in klines_list_1440_close:
            if "low" not in i:
                raise ValueError("Some element in kline_list does not contain \" low \" key")
        for i in klines_list_1440_close:
            if "volume" not in i:
                raise ValueError("Some element in kline_list does not contain \" volume \" key")

        self.klines = deepcopy(klines_list_1440_close)
        # This is the most important parameter of the sensor
        self.band_height = 3

        x_list = range(len(self.klines))
        y_list = []
        for i in self.klines:
            y_list.append(float(i['close']))

        deviationSum = 0
        slope, intercept, r_value, p_value, std_err = linregress(x_list, y_list)
        for count, i in enumerate(self.klines):
            deviationSum += (float(i['close']) - (slope * count + intercept)) ** 2
        deviation = math.sqrt(deviationSum / len(self.klines))
        self._dn = deviation * self.band_height
        self._up = deviation * self.band_height
        self._reg_m = slope
        self._reg_c = intercept
        self._reg_r = r_value  # This is the Correlation Coefficient Value
        self._reg_p = p_value
        self._reg_sd = std_err  # This is the standard deviation value

    # noinspection SpellCheckingInspection
    def __str__(self):
        """
        WORKING
        This one describes the current situation at the battle field
        :return: str
        """
        upper_band = self.short_loot_minimum_price()
        lower_band = self.long_loot_maximum_price()
        average_volume = self.average_volume()
        volume = self.klines[-1].get("volume")
        close_price = self.klines[-1].get("close")
        high_price = self.klines[-1].get("high")
        low_price = self.klines[-1].get("low")
        is_chaoting = self._is_chaoting()
        is_outside = self._is_outside_channel()
        sit = self.battlefield_situtation()
        reg_r = self._reg_r
        expected_profit = self.expected_profit_in_percentage()
        _, expected_hodling_time = self.expected_hodling_time()
        is_strong_up_trend = self._is_strong_uptrend()
        is_strong_down_trend = self._is_strong_downtrend()
        is_big_volume = self._is_big_volume()
        describe = "UpperBand: %s \n" \
                   "LowerBand: %s \n" \
                   "AverageVolume: %s\n" \
                   "Volume: %s\n" \
                   "ClosePrice: %s\n" \
                   "HighPrice: %s\n" \
                   "LowPrice: %s\n" \
                   "IsChaoting: %s\n" \
                   "IsOutside: %s\n" \
                   "IsStrongUpTrend: %s\n" \
                   "IsStrongDownTrend: %s\n" \
                   "IsBigVolume: %s\n" \
                   "Situtation: %s\n" \
                   "Correlation Coefficient: %s\n" \
                   "Expected Profit Percent: %s Percent\n" \
                   "Expected Hodling Time: %s Seconds\n" % (
                       str(upper_band),
                       str(lower_band),
                       str(average_volume),
                       str(volume),
                       str(close_price),
                       str(high_price),
                       str(low_price),
                       str(is_chaoting),
                       str(is_outside),
                       str(is_strong_up_trend),
                       str(is_strong_down_trend),
                       str(is_big_volume),
                       str(sit),
                       str(reg_r),
                       str(expected_profit),
                       str(expected_hodling_time))
        return describe

    def expected_hodling_time(self):
        """
        The expected holding time
        :return: string: description
        :return: int: hodling time in seconds
        """
        return "The expected holding time is 120 Minutes after entry", int(120 * 60)

    def expected_profit_in_percentage(self):
        """
        WORKING
        Returns the expected profit in percentage, for example
        if we expect the BTCUSD to be up 1 percent, this function
        returns 1.0
        :return: float
        """
        reg_price = self._reg_m * 1439 + self._reg_c
        profit = float(self._up) / float(reg_price) * float(2.0 / 3.0)
        return float(profit) * 100.0

    # noinspection SpellCheckingInspection
    def battlefield_situtation(self):
        """
        WORKING
        Whether the sensor saw a fire in the house
        :return: "shortloot" for short | "longloot" for long | "peace" for peace
        """
        if self._is_chaoting() and \
                self._is_big_volume() and \
                self._is_strong_uptrend() and \
                self._is_outside_channel():
            return "shortloot"
        elif self._is_chaoting() and \
                self._is_big_volume() and \
                self._is_strong_downtrend() and \
                self._is_outside_channel():
            return "longloot"
        else:
            return "peace"

    def long_loot_maximum_price(self):
        """
        WORKING
        :return: The Maximum Entry Price of Long Loot, The Lower Band
        """
        return self._reg_m * 1439 + self._reg_c - self._dn

    def short_loot_minimum_price(self):
        """
        WORKING
        :return: The Minimum Entry Price of Short Loot, The Upper Band
        """
        return self._reg_m * 1439 + self._reg_c + self._up

    def _is_outside_channel(self):
        """
        WORKING
        :return: True or False
        """
        close_price = float(self.klines[-1].get("close"))
        if close_price < self.long_loot_maximum_price() or \
                close_price > self.short_loot_minimum_price():
            return True
        else:
            return False

    # noinspection SpellCheckingInspection
    def _is_chaoting(self):
        """
        WORKING
        :return: Whether we see a huge increase in price range when compared to the previous bar
        """
        k = self.klines
        ticker = k[-1]
        previous_ticker = k[-2]
        previous_body_length = abs(float(previous_ticker.get("high")) - float(previous_ticker.get("low")))
        ticker_body_length = abs(float(ticker.get("high")) - float(ticker.get("low")))

        # Here is another important parameter for the sensor
        if (3 * previous_body_length) < ticker_body_length or (3 * ticker_body_length) < previous_body_length:
            return True
        else:
            return False

    def average_volume(self):
        """
        WORKING
        This function returns the average volume of the security in 1440 bars,
        :return: float
        """
        volumes = []
        for i in self.klines:
            volumes.append(int(i.get("volume")))
        return np.mean(volumes)

    def _is_big_volume(self):
        """
        WORKING
        Whether the current volume is greater than the average volume of 1440 tickers
        :return: True or False
        """

        last_volume = int(self.klines[-1].get("volume"))
        avg_vol = self.average_volume()

        if last_volume > avg_vol:
            return True
        else:
            return False

    def _is_strong_uptrend(self):
        """
        WORKING
        Whether the battlefield is in a strong uptrend
        :return: True or False
        """
        if self._reg_r >= 0.5:
            return True
        else:
            return False

    def _is_strong_downtrend(self):
        """
        WORKING
        Whether the battlefield is in a strong downtrend
        :return: True or False
        """
        if self._reg_r <= -0.5:
            return True
        else:
            return False

```
