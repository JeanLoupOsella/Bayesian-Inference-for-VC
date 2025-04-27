# Why Dirichlet Priors and Markov Chains?

### Overview
In this project, the objective is to estimate the probability that an investment made at the Seed stage eventually becomes a unicorn within 10 years.

The modeling approach combines two core probabilistic tools:

- **Markov Chains** to simulate the startup's yearly progression through funding stages

- **Dirichlet Distributions** to model uncertainty and update beliefs based on observed investment outcomes

### Why Use a Markov Chain?

A Markov chain is a mathematical model that describes transitions between a set of states over time, where the probability of moving to the next state depends only on the current state (the "Markov property").
I believe this model is particularly relevant for describing the evolution of a startup and its fundraising journey, as **each new Series is basically a new step in the chain** (Seed -> Series A -> Series B...) and
the properties of absorbing states can describe liquidity events (acquisition, IPO, bankruptcy) or a particular state (becoming a unicorn).

In the context of startup investments, we can simplify reality by stating that each startup can be in one of several discrete states:

- **Operating**
- **Next Stage** (raising a new round, usually priced)
- **Bankrupt** (absorbing state)
- **Unicorn** (can aslo be considered an absorbing state, even though in reality a unicorn can still bankrupt)

At every year (time step), a startup can:

- Stay in the same stage (Operating)

- Advance to the next funding stage (Next Stage)

- Exit the process through a bankruptcy (bad absorbing state)

- Exit through a unicorn outcome (good absorbing state)

Once a startup becomes bankrupt or a unicorn, it stays there (absorbing states).

The Markov chain captures this dynamic naturally, allowing simulation of startup trajectories over multiple years (up to 10 years in this model, chosen because it is usually the lifetime of a fund).

### Why Use a Dirichlet Prior?

A Dirichlet distribution is a probability distribution over probability vectors. It is commonly used as the **conjugate prior for multinomial processes** — exactly the case here, where each startup faces multiple possible discrete outcomes each year.

The Dirichlet prior allows us to:

- **Start with a somewhat reasonable belief** about transition probabilities (e.g., 27% chance of raising Series A at Seed stage)

- **Update these probabilities systematically** as we collect real-world data from investment outcomes

- **Quantify uncertainty** in the transition probabilities, rather than assuming they are fixed and perfectly known

When new observations arrive (e.g., a startup raises a Series A, stays operating, or goes bankrupt), **we can update the Dirichlet parameters by simply adding counts to the corresponding outcomes**.
This results in a **posterior Dirichlet distribution that reflects both prior knowledge and new data**, balancing exploration and exploitation.

### Benefits of This Approach

✅ **Bayesian Updating**: The investor can refine their beliefs as real investment results come in, rather than keeping static assumptions.

✅ **Handles Uncertainty**: Instead of a single probability value, the model maintain a distribution over possible probabilities.

✅ **Easily adaptable**: The model can be easily implemented in R and adapted to every investment thesis, giving an inference that suits each investor, with their investments.

✅ **Flexible**: As the investor gathers more data across stages (Seed, Series A, Series B, etc.), the model automatically adapts.

✅ **Suitable for Simulation**: By sampling from the Dirichlet posterior, it is possible to run Monte Carlo simulations to predict outcomes under realistic uncertainty.

✅ **Handles the very long feedback loop of Venture Capital**: A frequentist approach requires tons of data to give relevant results. On the contrary, a Bayesian approach allows to start inferencing probabilities 
under uncertainty, which is crucial in Venture Capital where the feedback loop will take a long time, making the Frequentist approach impossible to use for a vast majority of investor with no significant years of experience
in early-stage investing.
