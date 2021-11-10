---
layout: post
title: "경기 선행 지수와 코스피 지수"
author: "Jonghyun Ho"
categories: Data Analysis
tags: [Crawling, Python, CLI, 경기선행지수, Kospi, 코스피, Nasdaq, 나스닥]
image: posts/20200818/cli-and-kospi.png
---


`경기 선행 지수`의 추세 방향을 알면 경제의 순환 구조를 이해할 수 있을까?

이는 주가에 어떤 영향을 미치는지 확인해보려고 한다.


## 경기 선행 지수

`경기 선행 지수`는 각 국가별, 지역별로 6~9개월 뒤 경기흐름을 예측하는 지수로, 개별 국가 및 지역의 경기 전환점 예측을 위해 이용된다.

`OECD`와 `통계청`에서 `경기 선행 지수`를 발표하고 있는데, 각 기관에서 산출하는 계산 방식에는 약간의 차이가 있다.

`통계청 경기 선행 지수`의 경우 총 9개의 변수(구인구직비율, 재고순환지표, 소비자기대지수, 기계류 내수출하지수, 건설수주액, 코스피지수, 장단기금리차, 원자재지수, 수출입물가비율)를 이용하는 반면, `OECD 경기 선행 지수`에서 우리나라 지수는 6개의 변수(업황, 코스피 지수, 재고순환지표, 재고량, 장단기 금리차(3년물-1일물 금리), 순교역조건)만을 이용하고 있다.

[통계청에서 발표하는 경기선행지수](http://www.index.go.kr/potal/main/EachDtlPageDetail.do?idx_cd=1057)는 현재 8월 기준으로 최신 데이터는 6월이다. 이는 2개월 전의 데이터로 활용성이 다소 떨어진다.

`OECD` 에서 발표하는 `경기 선행 지수`는 현재 7월까지 정보가 업데이트 되어 있어 활용성이 상대적으로 높다.

이러한 이유로 `OECD`의 산출 데이터를 활용하려고 한다.

#### 출처

- [OECD 경기선행지수](https://terms.naver.com/entry.nhn?docId=3534606&cid=40942&categoryId=31906)

- [선행지수 순환변동치](https://kostat.go.kr/understand/info/info_lge/1/detail_lang.action?bmode=detail_lang&pageNo=1&keyWord=7&cd=SL4428&sTt=)


## 경기 선행 지수 데이터 얻기

필요한 라이브러리를 선언한다.

``` python
from datetime import datetime
import requests
import json
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import yfinance as yf

plt.rcParams["figure.figsize"] = (10, 5)
plt.rcParams['lines.linewidth'] = 1
plt.rcParams['lines.color'] = 'b'
```

`getCLI` 함수는 `OECD` 에서 json 포맷의 `경기 선행 지수` 데이터를 얻을 수 있다.

`country` 코드는 한국의 경우 `KOR`, 미국의 경우 `USA` 를 입력으로 받는다.

``` python
# Composite Leading Indicator
# https://data.oecd.org/leadind/composite-leading-indicator-cli.htm
def getCLI(country):
    uri = 'https://stats.oecd.org/sdmx-json/data/DP_LIVE/' + country + '.CLI.AMPLITUD.LTRENDIDX.M/OECD?json-lang=en&dimensionAtObservation=allDimensions&startPeriod=2005-01&endPeriod=2020-12'
    resp = requests.get(uri)
    result = json.loads(resp.text)

    dates = []
    cli = []
    cli_code = 'CLI.' + country

    observations = result['dataSets'][0]['observations']
    for key in observations:
        obs = observations[key][0]
        cli.append(obs)

    time_period = result['structure']['dimensions']['observation'][5]['values']
    for date in time_period:
        date = date['id']
        year = int(date[:4])
        month = int(date[5:7])
        dates.append(datetime(year=year, month=month, day=1))

    df = pd.DataFrame(data=cli, index=dates, columns=[cli_code])
    return df
```


## 주가 데이터 얻을 수 있는 함수

`경기 선행 지수`와의 비교를 얻기 위해 주가 데이터를 얻는다.

``` python
def GetYahooFinance(name, code):
    ticker = yf.Ticker(code)
    ticker = ticker.history(period='16y')
    ticker = ticker[['Close']]
    ticker.rename(columns={'Close': name}, inplace=True)
    return ticker
```


## 경기 선행 지수와 주가 지수의 비교, 시각화

한국과 미국 두 데이터를 함께 살펴본다.

``` python
code_names = [('KOR', 'Kospi', '^KS11'),
              ('USA', 'Nasdaq', '^DJI')]
```

`경기 선행 지수`와 주가 데이터를 얻어 하나의 데이터 프레임으로 합친다.

``` python
for country_code, yf_name, yf_code in code_names:
    cli_code = 'CLI.' + country_code
    cli = getCLI(country_code)

    ticker = GetYahooFinance(yf_name, yf_code)

    df = pd.concat([cli[cli_code], ticker[yf_name]], axis=1)
    df = df.interpolate(limit_direction='backward')
```

그래프로 시각화를 하는데 좌 축은 `경기 선행 지수`를, 우 축은 주가 정보를 표시하였다.

``` python
    fig = plt.figure()
    ax = fig.add_subplot(111)
    df[cli_code].plot(ax=ax)
    ax2 = ax.twinx()
    df[yf_name].plot(ax=ax2, color='orangered')
    fig.legend(loc="upper left", bbox_to_anchor=(0, 1), bbox_transform=ax.transAxes)
```

`gradient` 변수에 `경기 선행 지수`의 기울기를 저장한다.

투자의 관점에서 주의해야 할 구간을 표시하기 위해 `warnings` 변수에는 지수의 기울기가 감소하는 구간만 저장하였다.

``` python
    gradient = np.gradient(df[cli_code])
    warnings = pd.Series(gradient < 0, index=df.index)
```

`warnings`로 계산된 영역을 계산하여 그래프에 회색 음영처리를 하였다.

``` python
    range_list = []
    prev_val = False
    for index, value in warnings.iteritems():
        if prev_val != value:
            if value:
                begin = index
            else:
                range_list.append((begin, index))

        prev_inx = index
        prev_val = value

    for (begin, end) in range_list:
        plt.axvspan(begin, end, color='gray', alpha=0.3)

    plt.grid()
    plt.show()
```

아래는 `한국의 경기 선행 지수` 와 `코스피` 지수를 함께 시각화 한 결과이다.

![CLI and Kospi](/assets/img/posts/20200818/cli-and-kospi.png)

또한, `미국의 경기 선행 지수` 와 `나스닥` 지수를 함께 표시한 결과이다.

![CLI and Nasdaq](/assets/img/posts/20200818/cli-and-nasdaq.png)

위의 그래프에서 살펴보면 `경기 선행 지수`의 기울기가 상승하면서 하락으로 기울기가 전환되는 지점(음영 처리된 구간) 이후에는 주가도 함께 하락하는 경향을 확인할 수 있었다.

하지만 코스피의 2010년과 나스닥의 2019년의 경우를 보면 선행 지수가 감소하기 시작한 이후에도 1년 이상 주가는 상승하는 모습을 보이는 경우도 있었기 때문에, 100% 일치한다고 볼 수는 없지만 주의의 관점에서 지켜볼 필요는 있을 것 같다.


## 결론

이 지수에는 두 가지 함정이 있는 것으로 보인다.

첫번째는 **선행** 지수인데 이 수치에 대한 발표가 한두달 늦어진다는 점이어서 선행이라는 점의 메리트가 떨어지는 것 같다.

그럼에도 불구하고, 이 축을 한달 정도 미루어 이동해서 보아도 어느 정도 위의 규칙은 성립하는 것으로 보인다.

두번째는 `경기 선행 지수` 자체에 `코스피 지수` 정보를 포함하고 있다는 점이다. 그래서 함께 움직일 수 밖에 없는 것이다.

<pre>
OECD 경기 선행 지수에서 우리나라 지수는 6개의 변수(업황, 코스피 지수, 재고순환지표, 재고량, 장단기 금리차(3년물-1일물 금리), 순교역조건)만을 이용하고 있다.
</pre>

그렇기에 `장단기 금리차`와 같은 지표를 별도로 분리해서 확인해 볼 필요도 있을 것 같다.

최근 지표에서는 `경기 선행 지수`의 하락은 코로나 바이러스 확산 때문이 아니라 그 이전부터 진행이 되어오고 있었고, 2019년 후반부터 현재까지 다시 상승하고 있는 추세이다.

아직 지표가 상승 전환한 초반인데 반해 주가 상승폭이 높아서 방향성은 일치하지만 두 수치가 어떠한 형태로 수렴하게 될 지 궁금해지는 시점이다.