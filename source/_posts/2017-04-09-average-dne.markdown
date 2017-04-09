---
layout: post
title: "Average Doesn't Always Exist"
date: 2017-04-09 11:35
comments: true
categories: en
---

I believed average always exists. We are dealing with average very often in real life: average height of Japanese, average rate of divorcing in US, and average salary in the world,
the idea of average is very useful and we use it all times.
I also believed that more sample data you get, more accurately the reflects the reality. So, if you could get large enough data, it sounds reasonable to think that you can get the average of anything.

However, today I learned that this is not always the case. Average doesn't always exist and let me illustlate how this can happen.

## Average in Coin Toss Game

Let's being with the case when average exists. I'll use a very simple coin toss game. You flip a coin, and you get $1 if the result is head, $0 if the result is tail. Very simple.
When you toss the coin multiple times, what's the average of dollar you get? You can get the average very easily by the following:

`avg. of $ = (total number of $ you won) / number of times you toss the coin`

And you will see that the average is 0.5 because the chance of getting head is 50%. Maybe the average is not exactly 0.5, but more you toss coin, closer the average becomes to 0.5.

Let's do a few experiments with small programs. I'll use Clojure to simulate coin toss game and R to draw graph.

Here is a small Clojure code `coin-toss.clj` to simulate the game.

```clojure
(defn toss-coin []
  (rand-int 2))

(defn do-coin-toss [ite]
  (loop [n 1 total 0 avg []]
    (if (> n ite)
      (map float avg)
      (do
        (let [outcome (toss-coin)]
          (recur (+ n 1) (+ total outcome) (conj avg (/ total n))))))))

(defn write-data [col]
  (let [vec (map-indexed (fn [idx item]
                           (str idx " " item)) col)
        data (clojure.string/join "\n" vec)
        header "n avg\n"
        file "/tmp/coin-toss-result.txt"]
    (spit file header)
    (spit file data :append true)
    (spit file "\n" :append true)))
```

Here is R program `draw.R` to draw a graph.

```r
ind = commandArgs(trailingOnly=TRUE)[1]
file = "coin-toss.png"

(x <- read.table("/tmp/coin-toss-result.txt", header=T))
png(file)
plot(x$avg, type="l")
dev.off()
browseURL(file)
```

If you want to toss the coin 10 times and output the result to a text file, you can do `(write-data (do-coin-toss 10))` from REPL and run `R draw.R` from your terminal.

I'll denote `n` as the number of times you toss the coin for brevity.

Here is the graph of the average when n is 10.

![coin-toss-10](/images/coin-toss-10.png)

As you can see the average is very quickly approaching to 0.5. Let's do the same thing with larger n.

When n is 1000

![coin-toss-1000](/images/coin-toss-1000.png)

When n is 10000

![coin-toss-10000](/images/coin-toss-10000.png)

By looking at this graph, we can safely say that the average of dollar you win in this coin toss game is $0.5.

## When Average Doesn't Exist.

Now we are entering an interesting area where the average disappears. To illustlate this, we will make the coin toss game a bit more complicated.
This time, you keep flipping the coin until you get head and you will get 2 ^ number of times you flip the coin.

For example, if your result is tail, tail, and then head, then you get 2 ^ 3 (because you flipped three times) which is $8.

Here is a Clojure code to simulate this game.

```clojure
(defn exp [x n]
  (if (zero? n) 1
      (* x (exp x (dec n)))))

(defn toss-coin-until-head []
  (loop [n 1]
    (let [outcome (toss-coin)]
      (if (= 0 (toss-coin))
        (exp 2 n)
        (recur (+ n 1))))))

(write-data (do-coin-toss 10 toss-coin-until-head))
```

![coin-toss-until-head-10](/images/coin-toss-until-head-10.png)

![coin-toss-until-head-100](/images/coin-toss-until-head-100.png)

![coin-toss-until-head-10000](/images/coin-toss-until-head-10000.png)

![coin-toss-until-head-50000](/images/coin-toss-until-head-50000.png)
