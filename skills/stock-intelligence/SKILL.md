# stock-intelligence skill

## 触发条件
用户请求股票情报分析、多维度数据拉取、个股深度调研时触发。
触发词：「分析股票」「查情报」「拉数据」「个股调研」「股票分析」「监控XX」「股票情报」

---

## 数据源总览

| 数据类型 | 推荐数据源 | 免费限制 | Python库/接口 |
|---|---|---|---|
| 实时行情 | AKShare / yfinance | 完全免费 | akshare, yfinance |
| 历史K线 | AKShare / BaoStock | 完全免费 | akshare, baostock |
| 技术指标 | AKShare | 完全免费 | akshare |
| 基本面/财报 | Tushare / Financial Modeling Prep | 部分免费 | tushare, requests |
| 公告/重大事项 | Tushare / 东方财富 | 部分免费 | tushare |
| 新闻舆情 | Finnhub / AKShare | Finnhub免费tier | requests |
| 资金流向 | AKShare / 麦蕊智数 | AKShare免费 | akshare |
| 龙虎榜 | Tushare | 部分免费 | tushare |
| 宏观数据 | AKShare / BaoStock | 完全免费 | akshare, baostock |

---

## 输入参数

```
stock_code: str       # 股票代码，如 "000001"（A股）、"00700"（港股）、"AAPL"（美股）
market: str           # 市场标识："cn"（A股）、"hk"（港股）、"us"（美股），默认自动推断
intelligence_types: list[str]  # 要拉取的情报类型
date_range: str       # 时间范围，如 "1mo", "3mo", "6mo", "1y", "3y", "5y"，默认 "3mo"
output_format: str    # 输出格式："summary"（摘要）、"full"（完整）、"structured"（结构化）
```

### intelligence_types 可选值

- `realtime_quote`      — 实时行情（价格/涨跌幅/成交量/换手率）
- `kline`               — K线数据（日/周/月）
- `technical`           — 技术指标（MA/EMA/RSI/MACD/KDJ/BOLL）
- `fundamental`         — 基本面（PE/PB/总市值/流通市值/股本）
- `financial_report`    — 财务报表（营收/净利润/ROE/负债率）
- `announcements`       — 公告（重大事项/业绩预告/分红送转）
- `news`                — 新闻舆情（最近N条新闻及情感判断）
- `money_flow`          — 资金流向（大单/北向/主力净流入）
- `holder_trades`       — 股东增减持
- `macro`               — 宏观关联（行业指数/大盘情绪）
- `all`                 — 拉取全部类型

---

## 情报模块实现

### 1. 依赖检测

```python
def check_dependencies():
    """检测并列出所需依赖"""
    deps = {
        "akshare": "pip install akshare -U",
        "yfinance": "pip install yfinance",
        "tushare": "pip install tushare",
        "requests": "pip install requests",
        "pandas": "pip install pandas",
    }
    missing = []
    for lib, install_cmd in deps.items():
        try:
            __import__(lib)
        except ImportError:
            missing.append((lib, install_cmd))
    return missing
```

---

### 2. 股票代码标准化

```python
import re
from datetime import datetime, timedelta

def normalize_stock_code(stock_code: str, market: str = None) -> dict:
    """标准化股票代码，返回 {symbol, market, exchange, name, ts_code}"""
    stock_code = stock_code.strip().upper()

    # A股
    if market == "cn" or re.match(r"^\d{6}$", stock_code):
        if stock_code.startswith(("0", "3")):
            exchange = "sz"
        elif stock_code.startswith(("6", "9")):
            exchange = "sh"
        else:
            exchange = "sz"
        ts_code = f"{stock_code}.{exchange.upper()}"
        symbol = f"{exchange}.{stock_code}"
        return {"symbol": symbol, "ts_code": ts_code, "market": "cn", "exchange": exchange, "raw": stock_code}

    # 港股
    elif market == "hk" or (len(stock_code) <= 5 and stock_code.isdigit()):
        symbol = f"hk{stock_code}"
        ts_code = f"{stock_code}.HK"
        return {"symbol": symbol, "ts_code": ts_code, "market": "hk", "exchange": "hk", "raw": stock_code}

    # 美股（默认）
    else:
        return {"symbol": stock_code, "ts_code": stock_code, "market": "us", "exchange": "us", "raw": stock_code}
```

---

### 3. 实时行情

```python
def get_realtime_quote(symbol: str, market: str) -> dict:
    """拉取实时行情"""
    import akshare as ak
    import yfinance as yf

    result = {}

    if market == "cn":
        try:
            df = ak.stock_zh_a_spot_em()
            row = df[df["代码"] == symbol[-6:]]
            if not row.empty:
                r = row.iloc[0]
                result = {
                    "名称": r.get("名称", ""),
                    "代码": r.get("代码", ""),
                    "最新价": float(r.get("最新价", 0)),
                    "涨跌幅": float(r.get("涨跌幅", 0)),
                    "涨跌额": float(r.get("涨跌额", 0)),
                    "成交量": float(r.get("成交量", 0)),
                    "成交额": float(r.get("成交额", 0)),
                    "换手率": float(r.get("换手率", 0)),
                    "市盈率-动态": r.get("市盈率-动态", ""),
                    "市净率": r.get("市净率", ""),
                    "总市值": r.get("总市值", ""),
                    "流通市值": r.get("流通市值", ""),
                    "涨停价": float(r.get("涨停价", 0)),
                    "跌停价": float(r.get("跌停价", 0)),
                    "时间": str(r.get("时间", "")),
                }
        except Exception as e:
            result["error"] = str(e)

    elif market == "us":
        try:
            ticker = yf.Ticker(symbol)
            info = ticker.fast_info
            price_info = ticker.info
            result = {
                "symbol": symbol,
                "marketPrice": info.get("market_price", 0),
                "regularMarketPrice": price_info.get("regularMarketPrice", 0),
                "regularMarketChange": price_info.get("regularMarketChange", 0),
                "regularMarketChangePercent": price_info.get("regularMarketChangePercent", 0),
                "marketCap": price_info.get("marketCap", 0),
                "peRatio": price_info.get("trailingPE", 0),
                "volume": price_info.get("regularMarketVolume", 0),
                "fiftyTwoWeekHigh": price_info.get("fiftyTwoWeekHigh", 0),
                "fiftyTwoWeekLow": price_info.get("fiftyTwoWeekLow", 0),
            }
        except Exception as e:
            result["error"] = str(e)

    elif market == "hk":
        try:
            ticker = yf.Ticker(f"{symbol.upper()}.HK")
            price_info = ticker.info
            hist = ticker.history(period="5d")
            result = {
                "symbol": symbol,
                "marketCap": price_info.get("marketCap", 0),
                "peRatio": price_info.get("trailingPE", 0),
                "regularMarketPrice": price_info.get("regularMarketPrice", 0),
                "regularMarketChange": price_info.get("regularMarketChange", 0),
                "regularMarketChangePercent": price_info.get("regularMarketChangePercent", 0),
                "volume": price_info.get("regularMarketVolume", 0),
                "latest_close": float(hist["Close"].iloc[-1]) if not hist.empty else None,
            }
        except Exception as e:
            result["error"] = str(e)

    return result
```

---

### 4. K线数据

```python
def get_kline_data(symbol: str, market: str, period: str = "daily", date_range: str = "3mo") -> list[dict]:
    """拉取K线数据"""
    import akshare as ak
    import yfinance as yf
    import pandas as pd

    range_map = {"1mo": 30, "3mo": 90, "6mo": 180, "1y": 365, "3y": 1095, "5y": 1825}
    days = range_map.get(date_range, 90)
    end_date = datetime.now().strftime("%Y%m%d")
    start_date = (datetime.now() - timedelta(days=days)).strftime("%Y%m%d")

    result = []

    if market == "cn":
        try:
            period_map = {"daily": "daily", "weekly": "weekly", "monthly": "monthly"}
            df = ak.stock_zh_a_hist(
                symbol=symbol[-6:], period=period_map.get(period, "daily"),
                start_date=start_date, end_date=end_date, adjust="qfq"
            )
            if df is not None and not df.empty:
                result = df.tail(120).to_dict(orient="records")
        except Exception as e:
            return [{"error": str(e)}]

    elif market == "us":
        try:
            ticker = yf.Ticker(symbol)
            df = ticker.history(period=date_range)
            if df is not None and not df.empty:
                df.index = df.index.strftime("%Y-%m-%d")
                result = df.reset_index().to_dict(orient="records")
        except Exception as e:
            return [{"error": str(e)}]

    elif market == "hk":
        try:
            ticker = yf.Ticker(f"{symbol.upper()}.HK")
            df = ticker.history(period=date_range)
            if df is not None and not df.empty:
                df.index = df.index.strftime("%Y-%m-%d")
                result = df.reset_index().to_dict(orient="records")
        except Exception as e:
            return [{"error": str(e)}]

    return result
```

---

### 5. 技术指标

```python
def get_technical_indicators(symbol: str, market: str, date_range: str = "3mo") -> dict:
    """拉取技术指标（MA/EMA/RSI/MACD/KDJ/BOLL）"""
    import akshare as ak
    import pandas as pd

    range_map = {"1mo": 30, "3mo": 90, "6mo": 180, "1y": 365, "3y": 1095, "5y": 1825}
    days = range_map.get(date_range, 90)
    end_date = datetime.now().strftime("%Y%m%d")
    # 多取一些数据用于计算指标
    start_date = (datetime.now() - timedelta(days=days + 100)).strftime("%Y%m%d")

    result = {}

    if market == "cn":
        try:
            df = ak.stock_zh_a_hist(
                symbol=symbol[-6:], period="daily",
                start_date=start_date, end_date=end_date, adjust="qfq"
            )

            if df is not None and not df.empty and len(df) > 20:
                close = df["收盘"].astype(float)
                high = df["最高"].astype(float)
                low = df["最低"].astype(float)

                # === MA 均线 ===
                ma5 = close.rolling(5).mean()
                ma10 = close.rolling(10).mean()
                ma20 = close.rolling(20).mean()
                ma60 = close.rolling(60).mean()
                ma120 = close.rolling(120).mean()
                latest_close = close.iloc[-1]
                ma5_v, ma10_v, ma20_v, ma60_v, ma120_v = ma5.iloc[-1], ma10.iloc[-1], ma20.iloc[-1], ma60.iloc[-1], ma120.iloc[-1]
                # 均线多头：价格 > MA5 > MA10 > MA20 > MA60
                ma_bullish = latest_close > ma5_v > ma20_v > ma60_v if (pd.notna(ma60_v) and pd.notna(ma20_v)) else False
                result["MA"] = {
                    "MA5":  round(ma5_v, 2) if pd.notna(ma5_v) else None,
                    "MA10": round(ma10_v, 2) if pd.notna(ma10_v) else None,
                    "MA20": round(ma20_v, 2) if pd.notna(ma20_v) else None,
                    "MA60": round(ma60_v, 2) if pd.notna(ma60_v) else None,
                    "MA120": round(ma120_v, 2) if pd.notna(ma120_v) else None,
                    "当前价": round(latest_close, 2),
                    "均线形态": "多头排列" if ma_bullish else ("空头排列" if latest_close < ma5_v < ma20_v < ma60_v else "震荡"),
                }

                # === MACD ===
                ema12 = close.ewm(span=12, adjust=False).mean()
                ema26 = close.ewm(span=26, adjust=False).mean()
                dif = ema12 - ema26
                dea = dif.ewm(span=9, adjust=False).mean()
                macd = (dif - dea) * 2
                dif_v, dea_v, macd_v = dif.iloc[-1], dea.iloc[-1], macd.iloc[-1]
                prev_dif, prev_dea = dif.iloc[-2], dea.iloc[-2]
                result["MACD"] = {
                    "DIF": round(dif_v, 4),
                    "DEA": round(dea_v, 4),
                    "MACD": round(macd_v, 4),
                    "信号": "金叉" if (dif_v > dea_v and prev_dif <= prev_dea) else ("死叉" if (dif_v < dea_v and prev_dif >= prev_dea) else ("金叉" if dif_v > dea_v else "死叉")),
                }

                # === RSI ===
                delta = close.diff()
                gain = delta.where(delta > 0, 0.0)
                loss = -delta.where(delta < 0, 0.0)
                avg_gain = gain.rolling(14).mean()
                avg_loss = loss.rolling(14).mean()
                rs = avg_gain / avg_loss.replace(0, float("nan"))
                rsi = 100 - (100 / (1 + rs))
                result["RSI"] = {
                    "RSI6":  round(float(rsi.iloc[-6]) if len(rsi) > 6 else 0, 2),
                    "RSI12": round(float(rsi.iloc[-12]) if len(rsi) > 12 else 0, 2),
                    "RSI24": round(float(rsi.iloc[-24]) if len(rsi) > 24 else 0, 2),
                    "RSI14": round(float(rsi.iloc[-1]), 2),
                    "超买": rsi.iloc[-1] > 70,
                    "超卖": rsi.iloc[-1] < 30,
                }

                # === KDJ ===
                if len(df) > 9:
                    low9 = low.rolling(9).min()
                    high9 = high.rolling(9).max()
                    rsv = (close - low9) / (high9 - low9).replace(0, float("nan")) * 100
                    K = rsv.ewm(com=2, adjust=False).mean()
                    D = K.ewm(com=2, adjust=False).mean()
                    J = 3 * K - 2 * D
                    k_v, d_v, j_v = K.iloc[-1], D.iloc[-1], J.iloc[-1]
                    prev_k, prev_d = K.iloc[-2], D.iloc[-2]
                    result["KDJ"] = {
                        "K": round(k_v, 2),
                        "D": round(d_v, 2),
                        "J": round(j_v, 2),
                        "信号": "金叉" if (k_v > d_v and prev_k <= prev_d) else ("死叉" if (k_v < d_v and prev_k >= prev_d) else ("金叉" if k_v > d_v else "死叉")),
                    }

                # === BOLL ===
                if len(df) > 20:
                    mid = close.rolling(20).mean()
                    std = close.rolling(20).std()
                    upper = mid + 2 * std
                    lower = mid - 2 * std
                    u_v, m_v, l_v, c_v = upper.iloc[-1], mid.iloc[-1], lower.iloc[-1], latest_close
                    result["BOLL"] = {
                        "上轨": round(u_v, 2),
                        "中轨": round(m_v, 2),
                        "下轨": round(l_v, 2),
                        "当前价": round(c_v, 2),
                        "位置": "突破上轨" if c_v > u_v else ("跌破下轨" if c_v < l_v else "轨道内"),
                    }

        except Exception as e:
            result["error"] = str(e)

    elif market == "us" or market == "hk":
        try:
            ticker = yf.Ticker(symbol if market == "us" else f"{symbol.upper()}.HK")
            df = ticker.history(period=date_range)
            if df is not None and not df.empty and len(df) > 20:
                close = df["Close"]
                high = df["High"]
                low = df["Low"]

                # MA
                result["MA"] = {
                    "MA20": round(float(close.rolling(20).mean().iloc[-1]), 2),
                    "MA60": round(float(close.rolling(60).mean().iloc[-1]), 2),
                    "MA120": round(float(close.rolling(120).mean().iloc[-1]), 2),
                    "当前价": round(float(close.iloc[-1]), 2),
                }

                # RSI
                delta = close.diff()
                gain = delta.where(delta > 0, 0.0)
                loss = -delta.where(delta < 0, 0.0)
                avg_gain = gain.rolling(14).mean()
                avg_loss = loss.rolling(14).mean()
                rs = avg_gain / avg_loss.replace(0, float("nan"))
                rsi = 100 - (100 / (1 + rs))
                result["RSI"] = {
                    "RSI14": round(float(rsi.iloc[-1]), 2),
                    "超买": float(rsi.iloc[-1]) > 70,
                    "超卖": float(rsi.iloc[-1]) < 30,
                }
        except Exception as e:
            result["error"] = str(e)

    return result
```

---

### 6. 基本面数据

```python
def get_fundamental(symbol: str, market: str) -> dict:
    """拉取基本面数据"""
    import akshare as ak
    import yfinance as yf

    result = {}

    if market == "cn":
        try:
            info = ak.stock_individual_info_em(symbol=symbol[-6:])
            if info is not None and not info.empty:
                info_dict = dict(zip(info["item"], info["value"]))
                result = {
                    "总市值": info_dict.get("总市值", ""),
                    "流通市值": info_dict.get("流通市值", ""),
                    "市盈率-动态": info_dict.get("市盈率(动态)", ""),
                    "市净率": info_dict.get("市净率", ""),
                    "市销率": info_dict.get("市销率TTM", ""),
                    "总股本": info_dict.get("总股本", ""),
                    "流通股本": info_dict.get("流通股本", ""),
                    "所属行业": info_dict.get("行业", ""),
                    "上市时间": info_dict.get("上市时间", ""),
                    "概念题材": info_dict.get("概念题材", ""),
                }
        except Exception as e:
            result["error"] = str(e)

    elif market == "us":
        try:
            ticker = yf.Ticker(symbol)
            info = ticker.info
            result = {
                "marketCap": info.get("marketCap", ""),
                "peRatio": info.get("trailingPE", ""),
                "forwardPE": info.get("forwardPE", ""),
                "dividendYield": info.get("dividendYield", ""),
                "beta": info.get("beta", ""),
                "revenue": info.get("totalRevenue", ""),
                "grossMargins": info.get("grossMargins", ""),
                "operatingMargins": info.get("operatingMargins", ""),
                "profitMargins": info.get("profitMargins", ""),
                "recommendationKey": info.get("recommendationKey", ""),
                "shortName": info.get("shortName", ""),
                "sector": info.get("sector", ""),
                "industry": info.get("industry", ""),
            }
        except Exception as e:
            result["error"] = str(e)

    elif market == "hk":
        try:
            ticker = yf.Ticker(f"{symbol.upper()}.HK")
            info = ticker.info
            result = {
                "marketCap": info.get("marketCap", ""),
                "peRatio": info.get("trailingPE", ""),
                "dividendYield": info.get("dividendYield", ""),
                "revenue": info.get("totalRevenue", ""),
                "shortName": info.get("shortName", ""),
                "sector": info.get("sector", ""),
            }
        except Exception as e:
            result["error"] = str(e)

    return result
```

---

### 7. 财务报表

```python
def get_financial_report(symbol: str, market: str) -> dict:
    """拉取财务报表"""
    import akshare as ak
    import yfinance as yf

    result = {}

    if market == "cn":
        try:
            # 主要财务指标摘要
            indicator = ak.stock_financial_abstract(symbol=symbol[-6:])
            if indicator is not None and not indicator.empty:
                cols_to_keep = [c for c in ["股票代码", "股票简称", "营业总收入", "净利润", "资产总计",
                                             "负债合计", "加权平均净资产收益率", "基本每股收益", "稀释每股收益",
                                             "每股净资产", "净资产收益率(%)"]
                               if c in indicator.columns]
                sub = indicator[cols_to_keep].head(4)
                result["financial_summary"] = sub.to_dict(orient="records")
        except Exception as e:
            result["indicator_error"] = str(e)

        try:
            # 杜邦分析
            dupont = ak.stock_financial_duanalysis(symbol=symbol[-6:])
            if dupont is not None and not dupont.empty:
                result["dupont"] = dupont.head(4).to_dict(orient="records")
        except Exception as e:
            result["dupont_error"] = str(e)

    elif market == "us" or market == "hk":
        try:
            ticker = yf.Ticker(symbol if market == "us" else f"{symbol.upper()}.HK")
            income = ticker.income_history
            balance = ticker.balance_sheet
            cashflow = ticker.cashflow
            if income is not None and not income.empty:
                result["income"] = income.head(4).to_dict(orient="records")
            if balance is not None and not balance.empty:
                result["balance"] = balance.head(4).to_dict(orient="records")
            if cashflow is not None and not cashflow.empty:
                result["cashflow"] = cashflow.head(4).to_dict(orient="records")
        except Exception as e:
            result["error"] = str(e)

    return result
```

---

### 8. 公告与重大事项

```python
def get_announcements(symbol: str, market: str, date_range: str = "3mo") -> list[dict]:
    """拉取最近公告"""
    import akshare as ak

    range_map = {"1mo": 30, "3mo": 90, "6mo": 180, "1y": 365}
    days = range_map.get(date_range, 90)
    end_date = datetime.now().strftime("%Y%m%d")
    start_date = (datetime.now() - timedelta(days=days)).strftime("%Y%m%d")

    result = []

    if market == "cn":
        try:
            # 东方财富个股公告
            df = ak.stock_zh_a_disclosure(symbol=symbol[-6:], start_date=start_date, end_date=end_date)
            if df is not None and not df.empty:
                # 只保留关键列
                cols = [c for c in ["公告标题", "公告时间", "公告类型"] if c in df.columns]
                result = df[cols].head(20).to_dict(orient="records")
        except Exception as e:
            result = [{"error": str(e)}]

    return result
```

---

### 9. 新闻舆情

```python
import os
import requests

def get_news_sentiment(symbol: str, market: str) -> dict:
    """拉取新闻及情感分析"""
    result = {"news": [], "summary": {}}

    # Finnhub 免费tier（需要设置环境变量 FINNHUB_API_KEY）
    api_key = os.environ.get("FINNHUB_API_KEY", "")

    if api_key:
        try:
            to_date = datetime.now().strftime("%Y-%m-%d")
            from_date = (datetime.now() - timedelta(days=30)).strftime("%Y-%m-%d")

            # symbol 映射到 Finnhub 格式
            fsym_map = {"AAPL": "AAPL", "MSFT": "MSFT", "GOOGL": "GOOGL", "AMZN": "AMZN",
                        "TSLA": "TSLA", "NVDA": "NVDA", "META": "META",
                        "00700": "0700.HK", "09988": "9988.HK"}
            fsym = fsym_map.get(symbol, symbol if market == "us" else symbol)

            url = "https://finnhub.io/api/v1/company-news"
            params = {"symbol": fsym, "from": from_date, "to": to_date, "token": api_key}
            resp = requests.get(url, params=params, timeout=10)
            news = resp.json()

            positive_kw = ["beat", "surge", "growth", "profit", "upgrade", "buy", "expand",
                            "超预期", "增长", "盈利", "买入", "上调", "突破", "创新高"]
            negative_kw = ["miss", "drop", "loss", "downgrade", "sell", "lawsuit", "recall",
                             "不及预期", "下跌", "亏损", "下调", "卖出", "诉讼", "召回", "警告"]

            positive_count, negative_count = 0, 0

            for item in news[:30]:
                headline = (item.get("headline", "") + item.get("summary", "")).lower()
                for kw in positive_kw:
                    if kw.lower() in headline:
                        positive_count += 1
                        break
                for kw in negative_kw:
                    if kw.lower() in headline:
                        negative_count += 1
                        break

                result["news"].append({
                    "headline": item.get("headline", ""),
                    "source": item.get("source", ""),
                    "datetime": item.get("datetime", ""),
                    "url": item.get("url", ""),
                })

            total = positive_count + negative_count
            score = (positive_count - negative_count) / total if total > 0 else 0
            result["summary"] = {
                "positive": positive_count,
                "negative": negative_count,
                "neutral": max(0, len(news) - positive_count - negative_count),
                "sentiment_score": round(score, 3),
                "sentiment_label": "正面" if score > 0.2 else ("负面" if score < -0.2 else "中性"),
            }
        except Exception as e:
            result["error"] = str(e)
    else:
        # AKShare 财经新闻备选（国内A股）
        if market == "cn":
            try:
                import akshare as ak
                news_df = ak.stock_news_em(symbol=symbol[-6:])
                if news_df is not None and not news_df.empty:
                    result["news"] = news_df.head(20).to_dict(orient="records")
            except Exception as e:
                result["news_error"] = str(e)
        else:
            result["warning"] = "请设置 FINNHUB_API_KEY 环境变量以获取美股/港股新闻"

    return result
```

---

### 10. 资金流向

```python
def get_money_flow(symbol: str, market: str) -> dict:
    """拉取资金流向"""
    import akshare as ak

    result = {}

    if market == "cn":
        try:
            # 个股资金流向（按日）
            exchange_code = "sh" if symbol.startswith("6") else "sz"
            df = ak.stock_individual_fund_flow(stock=symbol[-6:], market=exchange_code)
            if df is not None and not df.empty:
                result["daily_flow"] = df.tail(5).to_dict(orient="records")
        except Exception as e:
            result["flow_error"] = str(e)

        try:
            # 北向资金持股明细
            north_df = ak.stock_hsgt_north_hold_stock_em(symbol=symbol[-6:])
            if north_df is not None and not north_df.empty:
                result["north_holdings"] = north_df.head(5).to_dict(orient="records")
        except Exception as e:
            result["north_error"] = str(e)

        try:
            # 行业资金流向排名
            ind_df = ak.stock_sector_fund_flow_rank(indicator="今日", sector_type="行业资金流向")
            if ind_df is not None and not ind_df.empty:
                # 找到该股所属行业的资金流向
                result["sector_flow"] = ind_df.head(20).to_dict(orient="records")
        except Exception as e:
            result["sector_error"] = str(e)

    return result
```

---

### 11. 主入口：情报聚合

```python
def gather_stock_intelligence(
    stock_code: str,
    market: str = None,
    intelligence_types: list = None,
    date_range: str = "3mo",
    output_format: str = "summary"
) -> dict:
    """
    股票情报主入口
    自动推断市场，聚合多维度数据

    参数:
        stock_code: 股票代码
        market: 市场标识，None则自动推断
        intelligence_types: 情报类型列表，None则使用默认 ['realtime_quote', 'technical', 'fundamental']
        date_range: 时间范围，默认 "3mo"
        output_format: 输出格式，"summary" | "full" | "structured"
    返回:
        dict: 聚合情报结果
    """
    import os
    os.environ.setdefault("PYTHONDONTWRITEBYTECODE", "1")

    # 参数默认值
    if intelligence_types is None:
        intelligence_types = ["realtime_quote", "technical", "fundamental"]
    if "all" in intelligence_types:
        intelligence_types = [
            "realtime_quote", "kline", "technical", "fundamental",
            "financial_report", "announcements", "news", "money_flow"
        ]

    # 代码标准化
    normalized = normalize_stock_code(stock_code, market)
    symbol = normalized["symbol"]
    ts_code = normalized["ts_code"]
    market = normalized["market"]
    raw_code = normalized["raw"]

    result = {
        "stock_code": raw_code,
        "symbol": symbol,
        "ts_code": ts_code,
        "market": market,
        "intelligence_types": intelligence_types,
        "date_range": date_range,
        "timestamp": datetime.now().strftime("%Y-%m-%d %H:%M:%S"),
        "data": {},
        "warnings": [],
    }

    # 按类型依次拉取（每个模块独立 try/catch）
    dispatch = {
        "realtime_quote":   lambda: get_realtime_quote(symbol, market),
        "kline":            lambda: get_kline_data(symbol, market, "daily", date_range),
        "technical":        lambda: get_technical_indicators(symbol, market, date_range),
        "fundamental":      lambda: get_fundamental(symbol, market),
        "financial_report": lambda: get_financial_report(symbol, market),
        "announcements":    lambda: get_announcements(symbol, market, date_range),
        "news":             lambda: get_news_sentiment(symbol, market),
        "money_flow":       lambda: get_money_flow(symbol, market),
    }

    for itype in intelligence_types:
        if itype in dispatch:
            try:
                result["data"][itype] = dispatch[itype]()
            except Exception as e:
                result["data"][itype] = {"error": str(e)}
                result["warnings"].append(f"{itype}: {str(e)}")
        else:
            result["warnings"].append(f"unknown type: {itype}")

    return result
```

---

## 输出格式示例

### summary 格式（默认）

```
📊 个股情报报告：平安银行 (000001)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
💰 实时行情 | 最新价: 12.34元 | 涨跌幅: +1.23%
📈 技术指标 | MA多头排列 | MACD金叉 | RSI: 58.3
💎 基本面   | PE: 5.2 | 市净率: 0.65 | 总市值: 2300亿
📰 新闻舆情 | 近30天共12条，正面6条/负面2条/中性4条
💵 资金流向 | 主力净流入: +2.3亿 | 北向持股: 3.2%
⚠️  风险提示 | 市净率低于1，存在破净风险
```

### structured 格式
返回完整 JSON 嵌套结构，保留所有原始字段，适合程序化处理。

---

## 依赖安装

```bash
pip install akshare yfinance tushare pandas requests
```

首次使用前验证依赖：
```bash
python -c "import akshare, yfinance, pandas, requests; print('✅ 依赖就绪')"
```

---

## 使用示例

```python
# 基础用法：实时行情 + 技术指标
gather_stock_intelligence("000001",
                          intelligence_types=["realtime_quote", "technical"])

# 全维度情报（A股，默认3个月）
gather_stock_intelligence("000001",
                          intelligence_types=["all"], date_range="3mo",
                          output_format="summary")

# 美股全维度
gather_stock_intelligence("AAPL", market="us",
                          intelligence_types=["all"], date_range="1y",
                          output_format="summary")

# 港股
gather_stock_intelligence("00700", market="hk",
                          intelligence_types=["realtime_quote", "news"])

# 指定 Tushare Token（财务数据接口需要）
# import os
# os.environ["TUSHARE_TOKEN"] = "your_token_here"
# gather_stock_intelligence("000001", intelligence_types=["financial_report"])

# 设置 Finnhub API Key（新闻情感分析）
# os.environ["FINNHUB_API_KEY"] = "your_key_here"
# gather_stock_intelligence("AAPL", market="us", intelligence_types=["news"])
```

---

## 注意事项

1. **积分限制**：Tushare 部分接口需积分，注册 tushare.pro 送120积分，够用日线基础数据；分钟级数据需单独开权限
2. **频率限制**：Finnhub 免费tier每分钟60次/日有限额，设置 API Key 环境变量 `FINNHUB_API_KEY` 即可使用
3. **数据延迟**：免费接口盘中数据有约 **15分钟延迟**，实盘需对接券商 L2 行情接口
4. **复权处理**：K线默认使用**前复权（qfq）**，所有技术指标计算基于复权数据，确保准确性
5. **市场推断**：不指定 market 时，6位数字代码默认推断为A股；港股需明确指定 market="hk"
6. **模块隔离**：每个情报模块独立 try/catch，单个模块失败不影响其他模块，返回结果中含 warnings 列表
7. **北向数据**：沪深港通北向持股数据依赖 AKShare 东方财富接口，可能因接口变更失效
8. **港股限制**：港股基本面/财务数据相对薄弱，yfinance 覆盖部分数据，部分字段可能为空
