# Problem Summary
A seller chooses a price in the range of $0 to $70. The revenue per customer is the price times the probability of buying. This presents a tradeoff: higher prices mean more per sale but fewer sales. We want to determine the ideal price, but the true demand curve is unknown. The only information we can gather comes from running experiments through the sales() function which are noisy (random customer counts and random purchases). Every experiment costs something: if we show a customer a bad price, we earn less. The goal is to find the best possible price while capturing as much of the optimal $11.925 per customer during the search. The revenue curve has multiple local peaks, so a method may center on a price that looks ideal but isn't.

# Design
## Measuring Uncertainty
I tracked each customer's revenue individually (either 0 or the price) and pooled those results by price across repeated testing. For each pricepoint, I computed the mean revenue per customer and the standard error. By the Central Limit Theorem, the mean is approximately normal, so the mean + or - a z-value * the SE gives a confidence interval. Choosing z is a tradeoff: a larger z produces wider intervals, making me less likely to throw out the true best price, but it means spending more customers on worse prices whos intervals still overlap the leader's. A smaller z would drop prices faster but risks eliminating the true best value. I chose z = 1.645 (90% confidence) because, unlike a scientific test/medical trial, a wrong decision between two close prices costs very little in revenue, while extra certainty is paid for in customers. 

## 3-Phase Grid Search
I decided to start my testing in intervals of $5 (excluding $0) as a baseline that I could then narrow down with subsequent trials while still being broad enough to hopefully catch each of the peaks and not get stuck on one hill. 14 is enough to see the overall shape without spending too heavily. With something like a $1 grid (70 prices) at 100 customers each, that's 7,000 customers before things even start getting narrowed down. 

100 customers is also enough to tell the hills apart, with ~100 customers, SEs were around $1-2 which is small enough to separate strong regions from weak ones. Additionally, 100 customers is not so many that bad prices get expensive. Since most grid prices turn out to be bad, I want this phase cheap.

From these 14 prices, I utilized a candidate elimination function applied to the top 5 prices as a safety precaution against noise from the initial grid. Using the top 5 also included prices from more than one region ($25/30 and $55), so the elimination could decide between hills with more data

The top 5 prices were tested on 100 more customers per price for up to 5 rounds. 100 customers per round increase the data at each price enough to shrink the SE enough for confidence intervals to separate a bit more. A max of 5 rounds provided a stopping rule for close ties so I wouldn't test them forever and waste money. Each round compared the best lower bound from all price confidence intervals against every other confidence interval's upper bound. The price with the best lower bound and all prices with upper bounds still overlapping that were maintained into the next round until 2 tied prices remained after the 5th round.


Finally, I performed a refinement search over 2 more rounds that tested prices within +-5 of the surviving prices, dividing the step size by a quarter after the first iteration. A $1 step to start finds the right price neighborhood cheaply, and then reducing it to $0.25 pinpoints within that interval, yielding a final best price of 27.0.


# Challenges

The biggest statistical challenge was distinguishing genuinely good prices from lucky ones. On the initial grid, $55 earned $11.58 per customer, nearly matching $25, but its standard error was almost twice as large because purchases at high prices are rare and each one shifts the average substantially. That wide interval made $55 hard to rule out: it survived the first three elimination rounds and cost roughly 400 additional customers before the data showed it was clearly worse than $25 and it was dropped in round 4. Rather than trusting  a single noisy grid result, the elimination rule kept testing $55 until its intervals no longer overlapped the leader's.

Another issue is that the revenue curve has multiple peaks. A purely local one like the gradient ascent demo'd in the notebook climbs whichever hill it starts on. Starting with a broad grid and carrying the top 5 prices, not just the single best, into elimination meant hills were compared directly with more data before committing to one.

Finally, $25 and $30 never fully separated within 5 rounds, since their intervals still overlapped. Instead of spending infinite customers to break the tie, the round cap ended after elimination and the refinement phase searched the region around both prices, which is where the best price ($27) turned out to be. 

# Results

### Figure 1
| phase       | customers | Revenue    | Revenue Per Customer | Percentage of Optimal Price |
|-------------|-----------|------------|----------------------|-----------------------------|
| grid        | 1438      | $9,485.00  | $6.60                | 55.31%                      |
| elimination | 1630      | $15,210.00 | $9.33                | 78.25%                      |
| refinement  | 3865      | $40,231.75 | $10.41               | 87.29%                      |
| Total       | 6933      | $64,926.75 | $9.36                | 78.53%                      |

### Figure 2
| Metric                      | Value      |
|-----------------------------|------------|
| Final Price                 | $27.00     |
| Customers Experimented On   | 6,933      |
| Revenue During Experiments  | $64,926.75 |
| Revenue Per Customer        | $9.36      |
| % of optimal price captured | 78.5%      |

My algorithm selected a final price of $27.00. Across all three phases, I experimented on 6,933 customers and earned $64,926.75 in revenue, or $9.36 per customer. At the optimal price, the same customers would have generated $82,676.03 ($11.925 each), so the search captured 78.5% of the optimal profit. Figure 1 breaks this down by phase. The grid search was the least efficient phase, earning $6.60 per customer (55.3% of optimal) because it deliberately tested prices across the full range, including clearly poor ones like $60 - $70. Elimination improved to $9.33 per customer (78.3%) by concentrating customers on the top price candidates, and refinement reached $10.41 per customer (87.3%) since it only tested prices near the peak. However, refinement also used 3,865 customers, more than half of the total, and its second round at $0.25 spacing returned the same price as the $1.00 spacing. This suggests that a single refinement round, or fewer customers per refinement price, would have raised the share of the overall optimal profitwithout changing the final answer. 