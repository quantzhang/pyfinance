# pyfinance

pyfinance is a Python package built for investment management and analysis of security returns.

It is meant to be a complement to existing packages geared towards quantitative finance, such as [pyfolio](https://github.com/quantopian/pyfolio), 
[pandas-datareader](https://github.com/pydata/pandas-datareader), and [fecon235](https://github.com/rsvp/fecon235).

<table>
<tr>
  <td>Supports</td>
  <td>Python 3.5 | 3.6 | 3.7</td>
</tr>
<tr>
  <td>Latest Release</td>
  <td>
    <a href="https://pypi.org/project/pyfinance/">
    <img src="https://img.shields.io/pypi/v/pyfinance.svg" alt="latest release" />
    </a>
  </td>
</tr>
<tr>
  <td>Package Status</td>
  <td>
    <a href="https://pypi.org/project/pyfinance/">
    <img src="https://img.shields.io/pypi/status/pyfinance.svg" alt="status" /></td>
    </a>
</tr>
<tr>
  <td>License</td>
  <td>
    <a href="https://github.com/bsolomon1124/pyfinance/blob/master/LICENSE">
    <img src="https://img.shields.io/pypi/l/pyfinance.svg" alt="license" />
    </a>
</td>
</tr>
</table>

## Contents


pyfinance is best explored on a module-by-module basis:

| Module | Description |
| ------ | ----------- |
| `datasets.py`  | Financial dataset download & assembly via `requests`. |
| `general.py`   | General-purpose financial computations, such as active share calculation, returns distribution approximation, and tracking error optimization. |
| `ols.py`       | Ordinary least-squares (OLS) regression, supporting static and rolling cases, built with a matrix formulation and implemented with NumPy. |
| `options.py`   | Vectorized option calculations, including Black-Scholes Merton European option valuation, Greeks, and implied volatility, as well as payoff determination for common money-spread option strategies. |
| `returns.py`   | Statistical analysis of financial time series through the CAPM framework, designed to mimic functionality of software such as FactSet Research Systems and Zephyr, with improved speed and flexibility. |
| `utils.py`     | Utilities not fitting into any of the above. |

Please note that `returns` and `general` are still in development; they are not thoroughly tested and have some NotImplemented features.

Installation

pyfinance is available via [PyPI](https://pypi.python.org/pypi/pyfinance/).  The latest version is 1.0.1 as of March 2018.  Install with pip:

```
$ pip3 install pyfinance
```

**Note**: pyfinance aims for compatibility with all minor releases of Python 3.x, but does not guarantee workability with Python 2.x.

## Dependencies

pyfinance relies primarily on Python's scientific stack, including NumPy, Pandas, Matplotlib, Seaborn, Scikit-Learn, and StatsModels.  Other dependencies include Beautiful Soup, Requests, xrld, and xmltodict.

See `setup.py` for specific version threshold requirements.

## Tutorial

This is a walkthrough of some of pyfinance's features.

The `returns.py` module is designed for statistical analysis of financial time series through the CAPM framework, designed to mimic functionality of software such as FactSet Research Systems and Zephyr, with improved speed and flexibility.

Its main class is `TSeries`, a subclassed Pandas Series.  The DataFrame equivalent, `TFrame`, is not yet implemented as of March 2018.

`TSeries` implements a collection of new methods that pertain specifically to investment management and the study of security returns and asset performance, such cumulative return indices and drawdown.

Here's an example of construction:

```python
>>> import numpy as np
>>> import pandas as pd
>>> from pyfinance import TSeries

>>> np.random.seed(444)

# Normally distributed with 0.08% daily drift term.
>>> s = np.random.randn(400) / 100 + 0.0008
>>> idx = pd.date_range(start='2016', periods=len(s))  # default daily freq.
>>> ts = TSeries(s, index=idx)

>>> ts.head()
2016-01-01    0.0044
2016-01-02    0.0046
2016-01-03    0.0146
2016-01-04    0.0126
2016-01-05   -0.0086
Freq: D, dtype: float64
```

And a few "new" methods:

```python
>>> ts.max_drawdown()
-0.12374551561531844

# Downsample to quarterly compounded returns.
>>> ts.rollup('Q')
2016-03-31    0.0450
2016-06-30    0.1240
2016-09-30    0.0631
2016-12-31   -0.0081
2017-03-31    0.1925
Freq: Q-DEC, dtype: float64

>>> ts.anlzd_stdev()
0.16318780660107757

>>> ts.sharpe_ratio(ddof=1)
2.501797257311737
```

Some statistics are benchmark-relative.  For methods that take a `benchmark` parameter, `benchmark` can be either another `TSeries`, a Pandas Series, a 1d NumPy array.

```python
>>> bmk = TSeries(np.random.randn(400) / 100 + .0005,
...               index=ts.index)
>>> ts.beta_adj(bmk)
0.3176455956603447

>>> ts.tracking_error(benchmark=bmk)
0.23506660057562254
```

With CAPM-related statistics such as alpha, beta, and R-squared, it can also be a Pandas DataFrame or 2d NumPy array.

```python
>>> multi_bmk = pd.DataFrame(np.random.randn(400, 2) / 100 + .0005,
...                          index=ts.index)

>>> # Multifactor model support.
>>> ts.alpha(multi_bmk)
0.0010849614688207107
```

`TSeries` comes with just one additional and optional argument that must be as a keyword argument: `freq` (default `None`) allows for manual specification of the time-series frequency.  It may be any frequency string or anchored offset string recognized by Pandas, such as 'D', '5D', 'Q', 'Q-DEC', or 'BQS-APR'.

```python
>>> # This is okay as long as a frequency can be inferred.
>>> ts.freq is None
True
```

The purpose of this extra parameter is to create an annualization factor for statistics that are given on an annualized basis, such as standard deviation.

If no frequency is passed explicitly, pyfinance will attempt to infer an annualization factor from the Index, with an exception being raised if neither of these yield a frequency.

```python
>>> no_idx = TSeries(np.random.laplace(size=24) * .01 + .005,
                     freq='M')

>>> no_idx.freq
'M'

>>> no_idx.anlzd_ret()
0.04975219957136123
```

`freq` can also be passed within some methods, which will override the class instance's `.freq` if it exists:

```python
>>> no_idx.anlzd_ret(freq='W')  # Treat `no_idx` as weekly returns.
0.2341731795205313
```

`datasets.py` provides for financial dataset download & assembly via `requests`.  It leverages sources including:

- Ken French's data library (via `pandas-datareader`);
- SEC.gov;
- cboe.com;
- AQR's dataset page;
- fred.stlouisfed.org;
- Robert Shiller's page at econ.yale.edu.

Below is a batch of examples.

Load SEC 13F filings:

```python
# Third Point LLC June 2017 13F
>>> from pyfinance import datasets
>>> url = 'https://www.sec.gov/Archives/edgar/data/1040273/000108514617001787/form13fInfoTable.xml'  # noqa
>>> df = datasets.load_13f(url=url)
>>> df.head()
          nameOfIssuer   titleOfClass      cusip   value  votingAuthority
0  ALEXION PHARMACE...            COM  015351109  152088          1250000
1  ALIBABA GROUP HL...  SPONSORED ADS  01609W102  634050          4500000
2         ALPHABET INC   CAP STK CL A  02079K305  534566           575000
3           ANTHEM INC            COM  036752103  235162          1250000
4       BANCO MACRO SA     SPON ADR B  05961W105   82971           900000
```

Industry-portfolio monthly returns:

```python
>>> from pyfinance import datasets
>>> ind = datasets.load_industries()
>>> ind.keys()
dict_keys([5, 10, 12, 17, 30, 38, 48])

# Monthly returns to 5 industry portfolios
>>> ind[5].head()
            Cnsmr  Manuf  HiTec  Hlth   Other
Date
1950-01-31   1.26   1.47   3.21   1.06   3.19
1950-02-28   1.91   1.29   2.06   1.92   1.02
1950-03-31   0.28   1.93   3.46  -2.90  -0.68
1950-04-30   3.22   5.21   3.58   5.52   1.50
1950-05-31   3.81   6.18   1.07   3.96   1.36
```

S&P 500 and interest rate data from Robert Shiller's website, 1871-present:

```python
>>> from pyfinance import datasets
>>> shiller = datasets.load_shiller()
>>> shiller.iloc[:7, :5]
            sp50p  sp50d  sp50e      cpi  real_rate
date
1871-01-31   4.44   0.26    0.4  12.4641     5.3200
1871-02-28   4.50   0.26    0.4  12.8446     5.3233
1871-03-31   4.61   0.26    0.4  13.0350     5.3267
1871-04-30   4.74   0.26    0.4  12.5592     5.3300
1871-05-31   4.86   0.26    0.4  12.2738     5.3333
1871-06-30   4.82   0.26    0.4  12.0835     5.3367
1871-07-31   4.73   0.26    0.4  12.0835     5.3400
```

The `ols.py` module provides ordinary least-squares (OLS) regression, supporting static and rolling cases, and is built with a matrix formulation and implemented with NumPy.

First, let's load some data on currencies, interest rates, and commodities to generate a regression of changes in the trade-weighted USD against interest rate term spreads and copper.

```python
>>> from pandas_datareader import DataReader

>>> syms = {
...     'TWEXBMTH': 'usd',
...     'T10Y2YM': 'term_spread',
...     'PCOPPUSDM': 'copper'
...     }

>>> data = DataReader(syms.keys(), data_source='fred',
...                   start='2000-01-01', end='2016-12-31')\
...     .pct_change()\
...     .dropna()\
...     .rename(columns=syms)

>>> y = data.pop('usd')

>>> data.head()
            term_spread  copper
DATE
2000-02-01      -1.4091 -0.0200
2000-03-01       2.0000 -0.0372
2000-04-01       0.5185 -0.0333
2000-05-01      -0.0976  0.0614
2000-06-01       0.0270 -0.0185

>>> y.head()
DATE
2000-02-01    0.0126
2000-03-01   -0.0001
2000-04-01    0.0056
2000-05-01    0.0220
2000-06-01   -0.0101
```

The `OLS` class implements "static" (single) linear regression, with the model being fit when the object is instantiated.

It is designed primarily for statistical inference, not out-of-sample prediction, and its attributes largely mimic the structure of StatsModels' [RegressionResultsWrapper](http://www.statsmodels.org/dev/generated/statsmodels.regression.linear_model.RegressionResults.html).

```python
>>> from pyfinance import ols

>>> model = ols.OLS(y=y, x=data)

>>> model.alpha  # the intercept - a scalar
0.0012303204434167458

>>> model.beta  # the coefficients
array([-0.0006, -0.0949])

>>> model.fstat
33.42923069295481

# Residuals and predicted y values are NumPy arrays
# with the same shape as `y`.
>>> model.resids.shape
(203,)

>>> model.predicted.shape
(203,)
```

The module also supports rolling regression.  (Iterative regressions done on sliding windows over the data.)

- `RollingOLS` has methods that generate NumPy arrays as outputs.
- `PandasRollingOLS` is a wrapper around `RollingOLS` and is meant to mimic the look of Pandas's deprecated `MovingOLS` class.  It generates Pandas DataFrame and Series outputs.

**Note**: all solutions are generated through a matrix formulation, which takes advantage of NumPy's broadcasting capabilities to expand the classical [matrix formulation](https://onlinecourses.science.psu.edu/stat501/node/382) to an additional dimension.  This approach may be slow for significantly large datasets.

Also, note that windows are not "time-aware" in the way that Pandas time functionality is.  Because of the NumPy implementation, specifying a window of 12 where the index contains one missing months would generate a regression over 13 months.  To avoid this, simply reindex the input data to a set frequency.

```python
# 12-month rolling regressions
# First entry would be the "12 months ending" 2001-01-30
>>> rolling = ols.PandasRollingOLS(y=y, x=data, window=12)

>>> rolling.beta.head()
            term_spread  copper
DATE
2001-01-01   9.9127e-05  0.0556
2001-02-01   4.7607e-04  0.0627
2001-03-01   1.4671e-03  0.0357
2001-04-01   1.6101e-03  0.0296
2001-05-01   1.5839e-03 -0.0449

>>> rolling.alpha.head()
DATE
2001-01-01    0.0055
2001-02-01    0.0050
2001-03-01    0.0067
2001-04-01    0.0070
2001-05-01    0.0048

>>> rolling.pvalue_alpha.head()
DATE
2001-01-01    0.0996
2001-02-01    0.1101
2001-03-01    0.0555
2001-04-01    0.0479
2001-05-01    0.1020
```

`options.py` is built for vectorized options calculations.

`BSM` encapsulates a European option and its associated value, Greeks, and implied volatility, using the Black-Scholes Merton model.

```python
>>> from pyfinance.options import BSM
>>> op = BSM(S0=100, K=100, T=1, r=.04, sigma=.2)

>>> op.summary()
OrderedDict([('Value', 9.925053717274437),
             ('d1', 0.3),
             ('d2', 0.09999999999999998),
             ('Delta', 0.6179114221889526),
             ('Gamma', 0.019069390773026208),
             ('Vega', 38.138781546052414),
             ('Theta', -5.888521694670074),
             ('Rho', 51.86608850162082),
             ('Omega', 6.225774084360724)])

# What is the implied annualized volatility at P=10?
>>> op.implied_vol(value=10)
0.20196480875586834

# Vectorized - pass an array of strikes.
>>> import numpy as np
>>> ops = BSM(S0=100, K=np.arange(100, 110), T=1, r=.04, sigma=.2)

>>> ops.value()
array([9.9251, 9.4159, 8.9257, 8.4543, 8.0015, 7.567 , 7.1506, 6.7519,
       6.3706, 6.0064])

# Multiple array inputs are evaluated elementwise/zipped.
>>> ops2 = BSM(S0=np.arange(100, 110), K=np.arange(100, 110),
...            T=1, r=.04, sigma=.2)

>>> ops2
BSM(kind=call,
    S0=[100 101 102 103 104 105 106 107 108 109],
    K=[100 101 102 103 104 105 106 107 108 109],
    T=1,
    r=0.04,
    sigma=0.2)

>>> ops2.value()
array([ 9.9251, 10.0243, 10.1236, 10.2228, 10.3221, 10.4213, 10.5206,
       10.6198, 10.7191, 10.8183])
```

`options.py` also exports a handful of options *strategies*, such as `Straddle`, `Straddle`, `Strangle`, `BullSpread`, and `ShortButterfly`, to name a few.

All of these inherit from a generic and customizable `OpStrat` class, which can be built from an arbitrary number of puts and/or calls.

Here is an example of constructing a bear spread, which is a combination of 2 puts or 2 calls (*put* is the default).  Here, we are short a put at 1950 and long a put at 2050.  Like the case of a single option, the instance methods are vectorized, so we can compute payoff and profit across a vector or grid:

```python
>>> from pyfinance import options as op

>>> spread = op.BearSpread(St=np.array([2100, 2000, 1900]),
...                        K1=1950., K2=2050.,
...                        price1=56.01, price2=107.39)

>>> spread.payoff()
array([  0.,  50., 100.])

>>> spread.profit()
array([-51.38,  -1.38,  48.62])
```

The `utils.py` module contains odds-and-ends utilities.

```python
>>> from pyfinance import utils

# Generate 7 unique 5-letter mutual fund tickers
>>> utils.random_tickers(length=5, n_tickers=7, endswith='X')
['JXNQX', 'DPTJX', 'WAKOX', 'DZIHX', 'MDYXX', 'HSKWX', 'IDMZX']

# Same for ETFs
>>> utils.random_tickers(3, 8)
['FIS', 'FNN', 'FZC', 'PWV', 'PBA', 'RDG', 'BKY', 'CDW']

# Five-asset portfolio leveraged 1.5x.
>>> utils.random_weights(size=5, sumto=1.5)
array([0.3263, 0.1763, 0.4703, 0.4722, 0.0549])

# Two 7-asset portfolios leverage 1.0x and 1.5x, respectively.
>>> utils.random_weights(size=(2, 7), sumto=[1., 1.5])
array([[0.1418, 0.2007, 0.0255, 0.2575, 0.0929, 0.2272, 0.0544],
       [0.3041, 0.109 , 0.2561, 0.2458, 0.3001, 0.0333, 0.2516]])

>>> utils.random_weights(size=(2, 7), sumto=[1., 1.5]).sum(axis=1)
array([1. , 1.5])

# Convert Pandas offset alises to periods per year.
>>> from pyfinance import utils

>>> utils.get_anlz_factor('M')
12.0
>>> utils.get_anlz_factor('BQS-DEC')
4.0
```

## API

For in-depth call syntaxes, see the source docstrings.
