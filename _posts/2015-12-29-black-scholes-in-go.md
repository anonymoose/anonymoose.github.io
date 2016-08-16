---
layout: post
title: "Black Scholes in Go"
description: "Equity derivative pricing in Go"
date: 2015-12-29 12:00:00 -0400
category: finance
tags: [golang, finance, derivatives]
---

I am working on a medium frequency trading system. I have collected a bunch of historical data I use for back testing. Daily, I connect to my broker, pull current position and pricing info, and make buy/sell/adjust decisions on liquid options chains based on current market conditions and probabilities ITM/OTM. I’m not making sub-second trades, but it needs to be snappy as a dollar here and there matter over time. Given the performance requirements I decided to try this in Go. If it’s fast enough for high ingestion databases, it’s certainly fast enough for me.
<br/><br/>
Black-Scholes is a derivative pricing algorithm that attempts to find the fair value for an option given a set of constraints. Arguably, these constraints place us in an unrealistic universe (frictionless market, zero arbitrage, etc). In practice however, Black-Scholes provides a sufficiently accurate set of numbers to guide decision making provided you don’t expect too much. And it’s easy to implement.
<br/><br/>
The rules in my algorithm require some of the numbers that come out of BS for determining risk/reward.
<br/><br/>
{% highlight go %}
import (
 “github.com/chobie/go-gaussian”
 “math”
 “time”
)
type Option struct {
 K         float64 // strike price
 S0        float64 // strike at time 0
 r         float64 // risk free rate
 sigma     float64 // volatility
 eval_date string // current time
 exp_date  string // expiration date
 T         float64 // distance between exp and current
 right     string // ‘C’ = call, ‘P’ = put
 price     float64 // derived from info above
 delta     float64 // derived from info above
 theta     float64 // derived from info above
 gamma     float64 // derived from info above
}
// if you don’t know the price, 
//    pass -1.0, but you have to know volatility.
// if you don’t know volatility, 
//    pass -1.0, but you have to know price.
func NewOption(right string, S0 float64, K float64, 
               eval_date string, exp_date string, r float64, 
               sigma float64, price float64) *Option {
 o := &Option{
   K: K,
   S0: S0,
   r: r,
   eval_date: eval_date,
   exp_date: exp_date,
   T: calculateT(eval_date, exp_date),
   right: right,
   sigma: sigma,
   price: price,
 }
 o.Initialize()
 return o
}
func calculateT(eval_date string, exp_date string) float64 {
 dtfmt := “20060102”
 evalDt, _ := time.Parse(dtfmt, eval_date)
 expDt, _ := time.Parse(dtfmt, exp_date)
 return (expDt.Sub(evalDt).Hours() / 24) / 365.0
}
func (self *Option) d1() float64 {
 return (math.Log(self.S0/self.K) 
      + (self.r+math.Pow(self.sigma, 2)/2)*self.T) 
      / (self.sigma * math.Sqrt(self.T))
}
func (self *Option) d2() float64 {
 return (math.Log(self.S0/self.K) 
      + (self.r-math.Pow(self.sigma, 2)/2)*self.T) 
      / (self.sigma * math.Sqrt(self.T))
}
const PI float64 = 3.14159265359
// calculate Black Scholes price and greeks
func (self *Option) Initialize() {
 norm := gaussian.NewGaussian(0, 1)
 td1 := self.d1()
 td2 := self.d2()
 nPrime := math.Pow((2*PI), -(1/2)) 
        * math.Exp(math.Pow(-0.5*(td1), 2))
 // we know volatility and want a price, or we’re guessing at 
 // volatility and we want a price.
 if self.price < 0 {
   if self.right == “C” {
     self.price = self.S0*norm.Cdf(td1) 
                — self.K*math.Exp(-self.r*self.T)*norm.Cdf(td2)
   } else if self.right == “P” {
     self.price = self.K*math.Exp(-self.r*self.T)*norm.Cdf(-td2) 
                — self.S0*norm.Cdf(-td1)
   }
 } else if self.sigma < 0 {
   self.sigma = self.impliedVol()
 }
 // handle the rest of the greeks now that we know everything else.
 if self.right == “C” {
   self.price = self.S0*norm.Cdf(td1) 
              — self.K*math.Exp(-self.r*self.T)*norm.Cdf(td2)
   self.delta = norm.Cdf(td1)
   self.gamma = (nPrime / (self.S0 * self.sigma 
              * math.Pow(self.T, (1/2))))
   self.theta = (nPrime)*(-self.S0*self.sigma*0.5/math.Sqrt(self.T)) 
              — self.r*self.K*math.Exp(-self.r*self.T)*norm.Cdf(td2)
 } else if self.right == “P” {
   self.price = self.K*math.Exp(-self.r*self.T)*norm.Cdf(-td2) 
              — self.S0*norm.Cdf(-td1)
   self.delta = norm.Cdf(td1) — 1
   self.gamma = (nPrime / (self.S0 * self.sigma 
              * math.Pow(self.T, (1/2))))
   self.theta = (nPrime)*(-self.S0*self.sigma*0.5/math.Sqrt(self.T)) 
              + self.r*self.K*math.Exp(-self.r*self.T)
              *norm.Cdf(-td2)
 }
}
// use newton raphson method to find volatility
func (self *Option) impliedVol() float64 {
  norm := gaussian.NewGaussian(0, 1)
  v := math.Sqrt(2*PI/self.T) * self.price / self.S0
  //fmt.Printf(“ — initial vol: %v\n”, v)
  for i := 0; i < 100; i++ {
    d1 := (math.Log(self.S0/self.K) 
       + (self.r+0.5*math.Pow(v, 2))*self.T) 
       / (v * math.Sqrt(self.T))
    d2 := d1 — v*math.Sqrt(self.T)
    vega := self.S0 * norm.Pdf(d1) * math.Sqrt(self.T)
    cp := 1.0
    if self.right == “P” {
      cp = -1.0
    }
    price0 := cp*self.S0*norm.Cdf(cp*d1) 
           — cp*self.K*math.Exp(-self.r*self.T)*norm.Cdf(cp*d2)
    v = v — (price0-self.price)/vega
    //fmt.Printf(“ — next vol %v : %v / %v \n”, i, v, 
    //             math.Pow(10, -25))
    if math.Abs(price0-self.price) < math.Pow(10, -25) {
       break
    }
 }
 return v
}
{% endhighlight %}

<br/><br/>
To exercise this, we feed it some numbers. Feed BS some info about the option and get back price, volatility, and [the greeks](https://en.wikipedia.org/wiki/Greeks_%28finance%29).

<br/><br/>

{% highlight go %}
func testBS() {
  S0        := 15.45 // Underlying price
  K         := 17.0 // contract strike
  exp_date  := “20160115” // expiration date
  eval_date := “20151228” // “today”
  r         := 0.001 // risk free rate
  vol       := 0.525 // implied volatility (guess)

  opt := NewOption(“C”, S0, K, eval_date, exp_date, r, vol, -1)

  fmt.Println(“CALL”)
  fmt.Printf(“Price: %v\n”, opt.price)
  fmt.Printf(“Delta: %v\n”, opt.delta)
  fmt.Printf(“Theta: %v\n”, opt.theta)
  fmt.Printf(“Gamma: %v\n”, opt.gamma)

  opt = NewOption(“C”, S0, K, eval_date, exp_date, r, -1, 0.20)
  fmt.Printf(“Volatility: %v\n”, opt.sigma)
}
{% endhighlight%}



## Disclaimer
<b>
I make no warranty as to the correctness of the numbers generated by this code or its suitability for any purpose. If you use this code and blow up your account, you were warned.
</b>
