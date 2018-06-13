# MachineLearningStocks in python: a starter project and guide

[![forthebadge made-with-python](https://ForTheBadge.com/images/badges/made-with-python.svg)](https://www.python.org/)

[![GitHub license](https://img.shields.io/badge/License-MIT-brightgreen.svg?style=flat-square)](https://github.com/AkashPatel1/MachineLearningStocks/blob/master/LICENSE.txt) [![Maintenance](https://img.shields.io/badge/Maintained%3F-yes-brightgreen.svg?style=flat-square)](https://github.com/AkashPatel1/MachineLearningStocks/graphs/commit-activity) [![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=flat-square)](http://makeapullrequest.com)

MachineLearningStocks is designed to be an **intuitive** and **highly extensible** template project applying machine learning to making stock predictions. My hope is that this project will help you understand the overall workflow of using machine learning to predict stock movements and also appreciate some of its subtleties. And of course, after following this guide and playing around with the project, you should definitely **make your own improvements** â€“ if you're struggling to think of what to do, at the end of this readme I've included a long list of possiblilities: take your pick.

Concretely, we will be cleaning and preparing a dataset of historical stock prices and fundamentals using `pandas`, after which we will apply a `scikit-learn` classifier to discover the relationship between stock fundamentals (e.g PE ratio, debt/equity, float, etc) and the subsequent annual price change (compared with the an index). We then conduct a simple backtest, before generating predictions on current data.

While I would not live trade based off of the predictions from this exact code, I do believe that you can use this project as starting point for a profitable trading system â€“ I have actually used code based on this project to live trade, with pretty decent results (around 20% returns on backtest and 10-15% on live trading).

This project has quite a lot of personal significance for me. It was my first proper python project, one of my first real encounters with ML, and the first time I used git. At the start, my code was rife with bad practice and inefficiency: I have since tried to amend most of this, but please be warned that some minor issues may remain (feel free to raise an issue, or fork and submit a PR). Both the project and myself as a programmer have evolved a lot since the first iteration, but there is always room to improve.

*As a disclaimer, this is a purely educational project. Be aware that backtested performance may often be deceptive â€“ trade at your own risk!*

*This guide has been cross-posted at my academic blog, [reasonabledeviations.science](https://reasonabledeviations.science/)*

## Contents

- [Contents](#contents)
- [Overview](#overview)
  - [EDIT as of 24/5/18](#edit-as-of-24-5-18)
- [Quickstart](#quickstart)
- [Preliminaries](#preliminaries)
- [Historical data](#historical-data)
  - [Historical stock fundamentals](#historical-stock-fundamentals)
  - [Historical price data](#historical-price-data)
- [Creating the training dataset](#creating-the-training-dataset)
  - [Preprocessing historical price data](#preprocessing-historical-price-data)
  - [Features](#features)
    - [Valuation measures](#valuation-measures)
    - [Financials](#financials)
    - [Trading information](#trading-information)
  - [Parsing](#parsing)
- [Backtesting](#backtesting)
- [Current fundamental data](#current-fundamental-data)
- [Stock prediction](#stock-prediction)
- [Unit testing](#unit-testing)
- [Where to go from here](#where-to-go-from-here)
  - [Data acquisition](#data-acquisition)
  - [Data preprocessing](#data-preprocessing)
  - [Machine learning](#machine-learning)
- [Contributing](#contributing)

## Overview

The overall workflow to use machine learning to make stocks prediction is as follows:

1. Acquire historical fundamental data â€“ these are the *features* or *predictors*
2. Acquire historical stock price data â€“ this is will make up the dependent variable, or label (what we are trying to predict).
3. Preprocess data
4. Use a machine learning model to learn from the data
5. Backtest the performance of the machine learning model
6. Acquire current fundamental data
7. Generate predictions from current fundamental data

This is a very generalised overview, but in principle this is all you need to build a fundamentals-based ML stock predictor.

### EDIT as of 24/5/18

This project uses pandas-datareader to download historical price data from Yahoo Finance. However, in the past few weeks this has become extremely inconsistent â€“ it seems like Yahoo have added some measures to prevent the bulk download of their data. I will try to add a fix, but for now, take note that `download_historical_prices.py` may be deprecated.

As a temporary solution, I've uploaded `stock_prices.csv` and `sp500_index.csv`, so the rest of the project can still function.

## Quickstart

If you want to throw away the instruction manual and play immediately, clone this project, then download and unzip the [data file](https://pythonprogramming.net/data-acquisition-machine-learning/) into the same directory. Then, open an instance of terminal and cd to the project's file path, e.g

```bash
cd Users/User/Desktop/MachineLearningStocks
```

Then, run the following in terminal:

```bash
pip install -r requirements.txt
python download_historical_prices.py
python parsing_keystats.py
python backtesting.py
python current_data.py
pytest -v
python stock_prediction.py
```

Otherwise, follow the step-by-step guide below.

## Preliminaries

This project uses python 3.6, and the common data science libraries `pandas` and `scikit-learn`. If you are on python 3.x less than 3.6, you will find some syntax errors wherever f-strings have been used for string formatting. These are fortunately very easy to fix (just rebuild the string using your preferred method), but I do encourage you to upgrade to 3.6 to enjoy the elegance of f-strings. A full list of requirements is included in the `requirements.txt` file. To install all of the requirements at once, run the following code in terminal:

```bash
pip install -r requirements.txt
```

To get started, clone this project and unzip it. This folder will become our working directory, so make sure you `cd` your terminal instance into this directory.

## Historical data

Data acquisition and preprocessing is probably the hardest part of most machine learning projects. But it is a necessary evil, so it's best to not fret and just carry on.

For this project, we need three datasets:

1. Historical stock fundamentals
2. Historical stock prices
3. Historical S&P500 prices

We need the S&P500 index prices as a benchmark: a 5% stock growth does not mean much if the S&P500 grew 10% in that time period, so all stock returns must be compared to those of the index.

### Historical stock fundamentals

Historical fundamental data is actually very difficult to find (for free, at least). Although sites like [Quandl](https://www.quandl.com/) do have datasets available, you often have to pay a pretty steep fee.

It turns out that there is a way to parse this data, for free, from [Yahoo Finance](https://finance.yahoo.com/). I will not go into details, because [Sentdex has done it for us](https://pythonprogramming.net/data-acquisition-machine-learning/). On his page you will be able to find a file called `intraQuarter.zip`, which you should download, unzip, and place in your working directory. Relevant to this project is the subfolder called `_KeyStats`, which contains html files that hold stock fundamentals for all stocks in the S&P500 between 2003 and 2013, sorted by stock. However, at this stage, the data is unusable â€“ we will have to parse it into a nice csv file before we can do any ML.

### Historical price data

In the first iteration of the project, I used `pandas-datareader`, an extremely convenient library which can load stock data straight into `pandas`. However, after Yahoo Finance changed their UI, `datareader` no longer worked, so I switched to [Quandl](https://www.quandl.com/), which has free stock price data for a few tickers, and a python API. However, as `pandas-datareader` has been [fixed](https://github.com/ranaroussi/fix-yahoo-finance), we will use that instead.

Likewise, we can easily use `pandas-datareader` to access data for the SPY ticker. Failing that, one could manually download it from [yahoo finance](https://finance.yahoo.com/quote/%5EGSPC/history?p=%5EGSPC), place it into the project directory and rename it `sp500_index.csv`.

The code for downloading historical price data can be run by entering the following into terminal:

```bash
python download_historical_prices.py
```

## Creating the training dataset

Our ultimate goal for the training data is to have a 'snapshot' of a particular stock's fundamentals at a particular time, and the corresponding subsequent annual performance of the stock.

For example, if our 'snapshot' consists of all of the fundamental data for AAPL on the date 28/1/2005, then we also need to know the percentage price change of AAPL between 28/1/05 and 28/1/06. Thus our algorithm can learn how the fundamentals impact the annual change in the stock price.

In fact, this is a slight oversimplification. In fact, what the algorithm will eventually learn is how fundamentals impact the *outperformance of a stock relative to the S&P500 index*. This is why we also need index data.

### Preprocessing historical price data

When `pandas-datareader` downloads stock price data, it does not include rows for weekends and public holidays (when the market is closed).

However, referring to the example of AAPL above, if our snapshot includes fundamental data for 28/1/05 and we want to see the change in price a year later, we will get the nasty surprise that 28/1/2006 is a Saturday. Does this mean that we have to discard this snapshot?

By no means â€“ data is too valuable to callously toss away. As a workaround, I instead decided to 'fill forward' the missing data, i.e we will assume that the stock price on Saturday 28/1/2006 is equal to the stock price on Friday 27/1/2006.

### Features

Below is a list of some of the interesting variables that are available on Yahoo Finance.

#### Valuation measures

- 'Market Cap'
- Enterprise Value
- Trailing P/E
- Forward P/E
- PEG Ratio
- Price/Sales
- Price/Book
- Enterprise Value/Revenue
- Enterprise Value/EBITDA

#### Financials

- Profit Margin
- Operating Margin
- Return on Assets
- Return on Equity
- Revenue
- Revenue Per Share
- Quarterly Revenue Growth
- Gross Profit
- EBITDA
- Net Income Avi to Common
- Diluted EPS
- Quarterly Earnings Growth
- Total Cash
- Total Cash Per Share
- Total Debt
- Total Debt/Equity
- Current Ratio
- Book Value Per Share
- Operating Cash Flow
- Levered Free Cash Flow

#### Trading information

- Beta
- 50-Day Moving Average
- 200-Day Moving Average
- Avg Vol (3 month)
- Shares Outstanding
- Float
- % Held by Insiders
- % Held by Institutions
- Shares Short
- Short Ratio
- Short % of Float
- Shares Short (prior month)

### Parsing

However, all of this data is locked up in HTML files. Thus, we need to build a parser. In this project, I did the parsing with regex, but please note that generally it is [really not recommended](https://stackoverflow.com/questions/1732348/regex-match-open-tags-except-xhtml-self-contained-tags) to use regex to parse HTML. However, I think regex probably wins out for ease of understanding (this project being educational in nature), and from experience regex works fine in this case.

This is the exact regex used:

```python
r'>' + re.escape(variable) + r'.*?(\-?\d+\.*\d*K?M?B?|N/A[\\n|\s]*|>0|NaN)%?(</td>|</span>)'
```

While it looks pretty arcane, all it is doing is searching for the first occurence of the feature (e.g "Market Cap"), then it looks forward until it finds a number immediately followed by a `</td>` or `</span>` (signifying the end of a table entry). The complexity of the expression above accounts for some subtleties in the parsing:

- the numbers could be preceeded by a minus sign
- Yahoo Finance sometimes uses K, M, and B as abbreviations for thousand, million and billion respectively.
- some data are given as percentages
- some datapoints are missing, so instead of a number we have to look for "N/A" or "NaN.

Both the preprocessing of price data and the parsing of keystats are included in `parsing_keystats.py`. Run the following in your terminal:

```bash
python parsing_keystats.py
```

You should see the file `keystats.csv` appear in your working directory. Now that we have the training data ready, we are ready to actually do some machine learning.

## Backtesting

Backtesting is arguably the most important part of any quantitative strategy: you must have some way of testing the performance of your algorithm before you live trade it.

Due to the importance of backtesting, I decided that I can't really call this a 'template machine learning stocks project' without backtesting. Thus, I have included a simplistic backtesting script.

Run the following in terminal:

```bash
python backtesting.py
```

You should get something like this:

```txt
Classifier performance
======================
Accuracy score:  0.81
Precision score:  0.75

Stock prediction performance report
===================================
Total Trades: 177
Average return for stock predictions:  37.8 %
Average market return in the same period:  9.2%
Compared to the index, our strategy earns  28.6 percentage points more
```

Again, the performance looks too good to be true and almost certainly is.

## Current fundamental data

Now that we have trained and backtested a model on our data, we would like to generate actual predictions on current data.

As always, we can scrape the data from good old Yahoo Finance. My method is to literally just download the statistics page for each stock (here is the [page](https://finance.yahoo.com/quote/AAPL/key-statistics?p=AAPL) for Apple), then to parse it using regex as before.

In fact, the regex should be almost identical, but because Yahoo has changed their UI a couple of times, there are some minor differences. This part of the projet has to be fixed whenever yahoo finance changes their UI, so if you can't get the project to work, the problem is most likely here.

Run the following in terminal:

```bash
python current_data.py
```

The script will then begin downloading the HTML into the `forward/` folder within your working directory, before parsing this data and outputting the file `forward_sample.csv`. You might see a few miscellaneous errors for certain tickers (e.g 'Exceeded 30 redirects.'), but this is to be expected.

## Stock prediction

Now that we have the training data and the current data, we can finally generate actual predictions. This part of the project is very simple: the only thing you have to decide is the value of the `OUTPERFORMANCE` parameter (the percentage by which a stock has to beat the S&P500 to be considered a 'buy'). I have set it to 10 by default, but it can easily be modified by changing the variable at the top of the file. Go ahead and run the script:

```bash
python stock_prediction.py
```

You should get something like this:

```txt
15 stocks predicted to outperform the S&P500 by more than 10%:
NOC NFX LH SCHL KSU DDS GWW AIZ ORLY R SFLY GME DLX DIS AMP 
```

## Testing
To run the tests, simply enter the following into a terminal instance in the project directory:

```bash
pytest -v
```


## Contributing

Feel free to fork, play around, and submit PRs. I would be very grateful for any bug fixes or more unit tests.

This project was originally based on Sentdex's excellent [machine learning tutorial](https://www.youtube.com/playlist?list=PLQVvvaa0QuDd0flgGphKCej-9jp-QdzZ3), but it has since evolved far beyond that and the code is almost completely different. The complete series is also on [his website](https://pythonprogramming.net/machine-learning-python-sklearn-intro/).
Also I have used parsing and downloaading codes from various repos.
---

