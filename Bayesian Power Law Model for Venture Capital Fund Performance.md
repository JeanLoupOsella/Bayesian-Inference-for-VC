# Bayesian Power Law Model for Venture Capital Fund Performance

This project implements a **Bayesian Mixture Model for Power Law Distributions with Zero-Inflation** in R to estimate the alpha parameter and predict the performance of venture capital funds. The model handles write-offs explicitly and provides probabilistic estimates of the Total Value to Paid-In capital (TVPI) for the next fund using previous track record (serving as a prior).

## Introduction

Venture capital (VC) funds often exhibit power-law distributions in their investment returns, where a few investments yield very high returns while most yield modest or zero returns. 
Estimating the alpha parameter of this power-law distribution is crucial for understanding the fund's performance and making informed investment decisions.

### Why Alpha Matters

The alpha parameter determines the shape of the power-law distribution and has significant implications for the fund's performance:

- **Alpha < 2**: Both the mean and variance are infinite. More investments increase both the expected median and mean returns.
- **2 ≤ Alpha ≤ 3**: The distribution has a finite mean but infinite variance. More investments increase the expected median return but not the expected mean.
- **Alpha > 3**: The distribution has finite mean and variance, allowing the Central Limit Theorem (CLT) to apply. Metrics like the Sharpe Ratio become meaningful. Increasing the number of investments does not affect the expected mean or median return.

I highly recommend [this report from AngelList](https://angel.co/pdf/growth.pdf) explaining in depth what are the implications of (not) having an infinite expected value and variance but essentially, an infinite expected value
means the more you invest, the bigger your expected value becomes. This is because it does not matter how many trash investments you do, having a track record with an alpha parameter < 2 means you do take enough risk 
to have some crazy multiples that completely overrides your failures. A Pre-Seed/Seed Fund should definitely have an alpha parameter < 2, otherwise that means they are not taking enough risk and more likely to 
underperform. 

### Model Overview

The model is a Bayesian Mixture Model that combines:
- A power-law distribution for non-zero investment multiples.
- A point mass at zero to account for write-offs (treating write-offs differently is necessary because just inputting 0s in the power law would break the model).

The Bayesian framework allows for the incorporation of prior knowledge (track record), provides a full posterior distribution for uncertainty quantification, and offers flexibility and robustness in handling complex data structures.

### Usage

1. **Data Preparation**: Prepare your investment multiples data, including write-offs as 0.
2. **Model Fitting**: Compile and fit the Stan model to the data.
3. **Posterior Analysis**: Extract and analyze the posterior samples of alpha and p_zero.
4. **Simulation**: Simulate investment multiples based on the posterior distribution.
5. **TVPI Calculation**: Calculate the TVPI for each simulation.
6. **Probability Estimation**: Estimate the probabilities of different TVPI outcomes.
7. **Visualization**: Plot the posterior distribution of alpha and the PDF of the TVPI.

### Model 

```
# Install and load necessary packages
if (!requireNamespace("rstan", quietly = TRUE)) {
  install.packages("rstan")
}
if (!requireNamespace("ggplot2", quietly = TRUE)) {
  install.packages("ggplot2")
}
library(rstan)
library(ggplot2)

# Define the Stan model as a string
stan_model_code <- '
data {
  int<lower=0> N; // number of observations
  vector<lower=0>[N] x; // observations
  real<lower=0> x_min; // minimum value of x
}
parameters {
  real<lower=1> alpha; // power law exponent
  real<lower=0, upper=1> p_zero; // probability of zero
}
model {
  // Prior for alpha
  alpha ~ normal(2, 1);

  // Prior for p_zero
  p_zero ~ beta(2, 2);

  // Likelihood
  for (n in 1:N) {
    if (x[n] == 0) {
      target += log(p_zero);
    } else {
      target += log(1 - p_zero) + log(alpha - 1) - alpha * log(x[n] / x_min);
    }
  }
}
'

# Write the model code to a file
writeLines(stan_model_code, "power_law_with_zero.stan")

# Your data with write-offs as 0
multiples <- c(54, 27, 5, 3, 3, 2, 2, 1, 1, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0)
x_min <- min(multiples[multiples > 0])

# Prepare data for Stan
data_list <- list(
  N = length(multiples),
  x = multiples,
  x_min = x_min
)

# Compile the Stan model
model <- stan_model("power_law_with_zero.stan")

# Fit the model
fit <- sampling(model, data = data_list, chains = 4, iter = 2000)

# Extract the posterior samples of alpha and p_zero
alpha_samples <- extract(fit)$alpha
p_zero_samples <- extract(fit)$p_zero

# Plot the posterior distribution of alpha
alpha_df <- data.frame(alpha = as.vector(alpha_samples))
ggplot(alpha_df, aes(x = alpha)) +
  geom_density(fill = "blue", alpha = 0.5) +
  labs(title = "Posterior Distribution of Alpha",
       x = "Alpha",
       y = "Density") +
  theme_minimal()

# Print summary statistics for the posterior distribution of alpha
cat("Summary statistics for the posterior distribution of alpha:\n")
cat("Mean:", mean(alpha_samples), "\n")
cat("Standard Deviation:", sd(alpha_samples), "\n")
cat("Median:", median(alpha_samples), "\n")
cat("95% Credible Interval:", quantile(alpha_samples, probs = c(0.025, 0.975)), "\n")

# Function to simulate power-law distributed random variables
rplcon <- function(n, alpha, xmin) {
  u <- runif(n)
  x <- xmin * (1 - u)^(-1 / (alpha - 1))
  return(x)
}

# Simulate investment multiples
n_simulations <- 10000
n_investments <- 20

simulate_multiples <- function(alpha_samples, p_zero_samples, n_investments, x_min) {
  multiples <- matrix(nrow = length(alpha_samples), ncol = n_investments)
  for (i in 1:length(alpha_samples)) {
    alpha <- alpha_samples[i]
    p_zero <- p_zero_samples[i]
    zero_indices <- sample(1:n_investments, size = round(p_zero * n_investments), replace = FALSE)
    multiples[i, zero_indices] <- 0
    multiples[i, -zero_indices] <- rplcon(n_investments - length(zero_indices), alpha, x_min)
  }
  return(multiples)
}

# Simulate multiples
simulated_multiples <- simulate_multiples(alpha_samples, p_zero_samples, n_investments, x_min)

# Calculate TVPI for each simulation
calculate_tvpi <- function(multiples) {
  tvpi <- apply(multiples, 1, mean)
  return(tvpi)
}

# Calculate TVPI for all simulations
tvpi_values <- calculate_tvpi(simulated_multiples)

# Filter out non-finite values
tvpi_values <- tvpi_values[is.finite(tvpi_values)]

# Calculate probabilities for TVPI > 2x, > 3x, and < 1x
prob_2x <- mean(tvpi_values > 2)
prob_3x <- mean(tvpi_values > 3)
prob_less_1x <- mean(tvpi_values < 1)

# Print the probabilities
cat("Probability of TVPI > 2x:", prob_2x, "\n")
cat("Probability of TVPI > 3x:", prob_3x, "\n")
cat("Probability of TVPI < 1x:", prob_less_1x, "\n")

# Plot the PDF of TVPI (limited to x-axis range of 0 to 10)
tvpi_df <- data.frame(tvpi = tvpi_values)
ggplot(tvpi_df, aes(x = tvpi)) +
  geom_density(fill = "red", alpha = 0.5) +
  labs(title = "PDF of TVPI",
       x = "TVPI",
       y = "Density") +
  theme_minimal() +
  xlim(0, 10)
```

### Example of Results

```
Summary statistics for the posterior distribution of alpha:
Mean: 1.804318
Standard Deviation: 0.2351895
Median: 1.781037
95% Credible Interval: 1.418044 2.326228

# Probability of the TVPI of the next fund using the same strategy considering the track record, assuming equal amount invested in each investment and no follow-on
Probability of TVPI > 2x: 0.6595
Probability of TVPI > 3x: 0.514
Probability of TVPI < 1x: 0.09725
```
![Capture d'écran 2025-05-03 211607](https://github.com/user-attachments/assets/e0c6a219-73e4-49b7-845c-aeae267de304)
![Capture d'écran 2025-05-03 211723](https://github.com/user-attachments/assets/611e3f02-25b0-484a-b03a-afed22e6e574)

### Interest of the model

- This can be very useful for **Limited Partners** trying to put a probability of getting a certain multiple based on the track record of the GP. The Bayesian nature of the model means this can be updated after every
subsequent fund to reduce uncertainty around the alpha parameters and thus, the probabilities of a success of the next fund.
- This can also be useful for **Investors** trying to assess their investment strategy. As a proxy, the alpha parameter of a seed stage fund should be < 2, Series A and B should be between 2 and 3, and Series C and should be between 3 and 4.
Beyond 4, results does not follow a Power Law distribution anymore, meaning more risks must be taken.

 ### Some data about alpha parameters of early-stage investing

 ![Capture d'écran 2025-05-03 212701](https://github.com/user-attachments/assets/6f3a8bd5-2597-400d-97af-91e0e4bc811e)

[Source](https://reactionwheel.net/2015/06/power-laws-in-venture.html)
