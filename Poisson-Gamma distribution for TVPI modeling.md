# TVPI Simulation for Seed VC Funds using the Poisson-Gamma distribution

This project simulates the total value to paid-in (TVPI) multiple for a seed-stage venture capital fund using Bayesian inference and Monte Carlo simulation. It accounts for uncertainty in both the unicorn rate (via a Gamma-Poisson model) and the exit multiple (via a Gamma distribution), with user-defined priors based on previous fund performance.

---

## What It Does

- Uses your **previous fund performance** to build priors.
- Updates your belief after observing **new unicorns and multiples**.
- Runs a **Monte Carlo simulation** to estimate the TVPI distribution.
- Computes **posterior distributions** for:
  - Unicorn rate (`λ`)
  - Exit multiple
- Outputs key stats:
  - Expected unicorns and TVPI
  - Probabilities: TVPI > 2x, > 3x, and < 1x
- Generates visualizations:
  - Distribution of TVPI
  - Distribution of unicorn counts

---

## Model Overview

- **Unicorn rate** modeled with a Gamma-Poisson distribution:
  - Prior: Gamma(k, θ)
  - Posterior: Updated after new unicorn observations
- **Multiple** modeled with a Gamma distribution:
  - Prior: Gamma(α, β), estimated from range (e.g. x25–x100)
  - Posterior: Updated with observed multiples

---

## Interest of the model

Venture Capital funds operate under great uncertainty because of a signicant time-lag regarding the feedback loop (an investment can take 10+ years before it becomes clear if it was a good one or not). This long feedback loop
makes it hard to use a Frequentist approach for estimating probabilities. Instead, this model can be used to estimate the likelihood of a fund generating a TVPI > 1x for limited partners based on track record, be it 
an angel investing or previous fund track record. By using bayesian inference, the uncertainty regarding the parameters decreases after every new fund or sequence of investments. This can be useful for Limited Partners
trying to infer the likelihood of investing in an emerging manager with limited track record to be a good investment. 

## Requirements

- R 4.0+
- ggplot2 package

```r
install.packages("ggplot2")
```

## Model

```
library(ggplot2)

# === USER INPUT ===

# Previous fund performance (for priors)
prev_investments <- 40
prev_unicorns <- 1
prior_unicorn_sd <- 1  # std dev on unicorn rate

# Current fund data
current_investments <- 40
current_unicorns <- 2
observed_multiples <- c(27, 89)  # Multiples of observed unicorns

# Prior on multiple (Gamma)
prior_multiple_mean <- 50
prior_multiple_range <- c(25, 100)  # Used to compute sd
prior_multiple_sd <- (prior_multiple_range[2] - prior_multiple_range[1]) / 4

# Monte Carlo settings
num_simulations <- 10000

# === INFER PRIORS ===

# Unicorn rate prior: Gamma(k, θ) with mean = kθ, sd = θ√k
lambda_prior <- prev_unicorns / prev_investments
theta_prior <- prior_unicorn_sd / sqrt(lambda_prior)
k_prior <- lambda_prior / theta_prior

# Multiple prior: Gamma(α, β) with mean = α/β, sd = √(α)/β
beta_mult <- prior_multiple_mean / prior_multiple_sd^2
alpha_mult <- prior_multiple_mean * beta_mult

# === BAYESIAN UPDATES ===

# Updated Gamma posterior for unicorn rate
k_post <- k_prior + current_unicorns
theta_post <- theta_prior / (1 + theta_prior * current_investments)

# Updated Gamma posterior for multiples
alpha_post_mult <- alpha_mult + sum(observed_multiples)
beta_post_mult <- beta_mult + length(observed_multiples)

# === MONTE CARLO SIMULATION ===

# Sample unicorn rate λ from posterior
lambda_sim <- rgamma(num_simulations, shape = k_post, scale = theta_post)

# Sample number of unicorns from Poisson(λ × n)
unicorns_sim <- rpois(num_simulations, lambda_sim * current_investments)

# Sample multiple values from posterior Gamma for each unicorn
multiple_samples <- rgamma(num_simulations, shape = alpha_post_mult, rate = beta_post_mult)

# Compute TVPI = (number of unicorns × sampled multiple) / total investments
TVPI_sim <- (unicorns_sim * multiple_samples) / current_investments

# === OUTPUT ===

cat("---- Posterior Results ----\n")
cat("Updated lambda (mean unicorn rate):", round(mean(lambda_sim), 4), "\n")
cat("Gamma posterior (k, theta):", round(k_post, 4), ",", round(theta_post, 4), "\n")
cat("Posterior SD of lambda:", round(sd(lambda_sim), 4), "\n\n")

cat("Updated multiple (mean):", round(mean(multiple_samples), 2), "\n")
cat("Gamma posterior (alpha, beta):", round(alpha_post_mult, 2), ",", round(beta_post_mult, 2), "\n")
cat("Posterior SD of multiple:", round(sd(multiple_samples), 2), "\n\n")

cat("Monte Carlo average unicorns:", round(mean(unicorns_sim), 2), "\n")
cat("Monte Carlo average TVPI:", round(mean(TVPI_sim), 2), "\n")
cat("Probability TVPI > 2x:", round(mean(TVPI_sim > 2), 4), "\n")
cat("Probability TVPI > 3x:", round(mean(TVPI_sim > 3), 4), "\n")
cat("Probability TVPI < 1x:", round(mean(TVPI_sim < 1), 4), "\n")


# === PLOTS ===

# TVPI Distribution
tvpi_data <- data.frame(TVPI = TVPI_sim)
p1 <- ggplot(tvpi_data, aes(x = TVPI)) +
  geom_histogram(binwidth = 0.2, fill = "#69b3a2", color = "white") +
  geom_vline(xintercept = 2, linetype = "dashed", color = "blue") +
  geom_vline(xintercept = 3, linetype = "dashed", color = "red") +
  labs(title = "Distribution of TVPI from Monte Carlo Simulation",
       x = "TVPI", y = "Frequency")

# Unicorn Count Distribution
unicorn_data <- data.frame(Unicorns = unicorns_sim)
p2 <- ggplot(unicorn_data, aes(x = Unicorns)) +
  geom_bar(fill = "#f9c74f", color = "black") +
  labs(title = "Distribution of Unicorn Counts",
       x = "Number of Unicorns", y = "Frequency")

# Show Plots
print(p1)
print(p2)
```

## Example outpout
```
Updated lambda (mean unicorn rate): 0.0499 
Gamma posterior (k, theta): 2.004 , 0.0249 
Posterior SD of lambda: 0.0354 
Updated multiple (mean): 57.49 
Gamma posterior (alpha, beta): 123.11 , 2.14 
Posterior SD of multiple: 5.18 
Monte Carlo average unicorns: 2.01 
Monte Carlo average TVPI: 2.88 
Probability TVPI > 2x: 0.4998
Probability TVPI > 3x: 0.3729
Probability TVPI < 1x: 0.2478
```

![Capture d'écran 2025-05-01 151429](https://github.com/user-attachments/assets/2f4ab0bf-35f9-49b2-8496-da1e0c7c219e)

![Capture d'écran 2025-05-01 151523](https://github.com/user-attachments/assets/bfa26997-7acd-4936-90eb-c0cc97a0b593)
