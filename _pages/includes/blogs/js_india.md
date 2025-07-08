---
permalink: /blogs/js_india
title: "Jane Street Manipulated India Market and Got Barred"
excerpt: ""
author_profile: false
---

July 03, 2025 in the [comprehensive report](https://www.sebi.gov.in/enforcement/orders/jul-2025/interim-order-in-the-matter-of-index-manipulation-by-jane-street-group_95040.html) published by **Securities and Exchange Board of India** (SEBI), the famous hedge fund **Jane Street** (JS) is barred from participating in the Indian domestic stock market, citing it manipulated the stock index and profited illegally. 

In this short blog, let's take a look at how Jane Street did it and how SEBI found out it.

## Beginning of Story

SEBI mentioned in the report that it started to be aware of Jane Street's abnormal trading activites by the lawsuit between Jane Street and Millennium about if the ex-JS employee had stolen the super lucrative strategy in India option market and adopted it in Millennium. 

During the court, Jane Street claimed it earned about \\$1 billion from the strategy alone last year. For reference, the company reported about \\$20.5 billion in trading revenue last year. 

Douglas Schadewald, one of the accused ex-JS employee, defended himself by stating that the complaint appears to suggest that the existence of inefficiencies within the Indian options market and the size of this market are trade secrets or proprietary to Jane Street. Retrospectively, his defense statement is actually indicative of some suspicious trading that Jane Street were doing in India. 

<img src="/images/blogs/media1.jpg" alt="media 1" width="600">


## Deep Dive into Report

**SEBI** is kind enough to give a brief derivative trading introduction at the beginning of the report. In a quick summary it goes as:

+ Prices of derivatives and their underlying stocks are interrelated and at equilibrium
    + otherwise, arbitrage opportunity exists which itself will push the price to equilibrium
+ On expiry day, volumes and participation in index options is far higher than in the underlying constituent stocks and futures
    + market participants enjoy the enormous leverage from options
+ Influencing underlying stock and index can help to profit from index options
    + If stock and index price goes up, the long-leg options become more valuable and short-leg ones devalue

**SEBI** summarized the manipulative trading patterns Jane Street were doing into 2 strategies and provided comphensive data evidence to back up the support the cases. We will look at 2 representive days for the 2 strategies respectively. The stock index involved in this matter is the **NIFTY** bank index comprising of **12** largest state-owned and private sector banks in India market.

### Intraday Index Manipulation

Sample day: January 17, 2024.

+ Phase 1: 09:15 AM - 11:46 AM
    + JS first bought about INR 4,370.03 crores (~ 526 million USD) **NIFTY** index constituents in stock and futures markets
    + The option market was misled and prices rose up accordingly in reaction to JS's long positions.
    + Simultaneously, JS built about INR 32,114.96 crores (~3.82 billion USD) **NIFTY** bearish positions by buying cheaper put options and selling more expensive call options
+ Phase 2: 11:46 AM - 15:30 PM
    + JS sold out its newly-bought index constituents aggressively in stock and futures markets at a moderate loss.
    + the **NIFTY** index drops as its constituent stocks price fell with JS selling.
    + JS gained significant profits from the bearish option positions

### Extended Marking the Close

Sample day:  July 10, 2024

In a similar fashion to the Intraday Index Manipulation, in the last hour of trading session (14:30 PM - 15:30 PM), JS

+ accumulated a significant position in the **NIFTY** index options.
+ aggressively bought/sold the **NIFTY** constituent stock and its futures to move the index to the favorable direction to its large open option positions

## Consequence

According to the **SEBI** interim order, JS would

+ to forfeit and turn in the unlawful profit of INR ₹48,435,770,168 (~\\$564.2 million USD)
+ be banned from participating in the India domestic financial security market

## Thoughts

**Alexander Gerko**, the CEO at XTX Markets, made an insightful comment on LinkedIn about this matter by July 5, 2025:

> Interesting questions to be answered are
> * How much of JS revenue in India index options is derived from similar activity? My current guess is 90% so a lot more for SEBI to dig out.
> * How much of JS revenue globally is derived from similar activity? What stumps me is how you have a 20+ bil revenue a year legit, highly leveraged business and have no qualms with 10% of it being fraud? With 300bil gross book one would expect exceptionally good controls throughout.  So either this function is intentionally stuffed with muppets while trading is done by IMO winners or the whole thing is company policy. Regulators elsewhere should pay attention

My own thoughts on this matter:

1. Why it took so long for the **SEBI** to find out that JS is manipulating their security market? The manipulation is not sophisticated (textbook material in the SIE exam in the U.S. context) and everyone knows JS is making a huge profit from the India market. 

2. Where would the impounded profit go? **SEBI** didn't say the money would go back to investors (probably not possible as well). So essentially the money from domestic and foreign investors would go directly into **SEBI**'s pocket. If that's the case, the JS is really doing a big favor for **SEBI** to collect and harvest the money for free. 

If we combine (1) and (2) together, it seems to indicative of some chess player behind the scene. 

But I will stop here.