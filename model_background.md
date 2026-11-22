## Model Background

This framework links three quantitative models for automated market making into a single pipeline:

| Stage | Model | What It Solves | Output Used Downstream |
| :--- | :--- | :--- | :--- |
| **1. Volatility** | **Yang–Zhang (2000)** | Estimates price volatility from OHLC data across overnight gaps and intraday ranges without drift bias. | Volatility estimate $\sigma$ |
| **2. Order Flow** | **Hawkes Process (1971)** | Captures order clustering and cross-excitation instead of assuming flat Poisson arrivals. | Dynamic intensities $\lambda_b(t), \lambda_a(t)$ & liquidity parameters |
| **3. Quoting** | **Avellaneda–Stoikov (2008)** | Sets inventory-adjusted reservation prices and spreads under exponential utility. | Bid/ask quotes ($p^b, p^a$) & spread ($\delta^b + \delta^a$) |

---

### 1. Volatility Estimation: Yang–Zhang (2000)

A market maker needs an estimate of mid-price volatility ($\sigma$) that responds to market movement without getting distorted by bid–ask bounce, overnight gaps, or drift. Close-to-close volatility ignores intraday paths, while Parkinson and Garman–Klass estimators struggle with non-zero drift or overnight jumps.

Over a rolling window of $N$ bars with Open ($O_i$), High ($H_i$), Low ($L_i$), and Close ($C_i$) prices, total variance is:

$$\sigma_{\text{YZ}}^2 = \sigma_o^2 + k \sigma_c^2 + (1 - k)\sigma_{\text{RS}}^2$$

* **Overnight variance ($\sigma_o^2$):** Sample variance of overnight log returns $o_i = \ln(O_i / C_{i-1})$.
* **Open-to-close variance ($\sigma_c^2$):** Sample variance of intraday log returns $c_i = \ln(C_i / O_i)$.
* **Rogers–Satchell variance ($\sigma_{\text{RS}}^2$):** Drift-independent intraday range variance:

$$\sigma_{\text{RS}}^2 = \frac{1}{N} \sum_{i=1}^{N} \left[ \ln\left(\frac{H_i}{C_i}\right)\ln\left(\frac{H_i}{O_i}\right) + \ln\left(\frac{L_i}{C_i}\right)\ln\left(\frac{L_i}{O_i}\right) \right]$$

* **Variance-minimizing weight ($k$):**

$$k = \frac{0.34}{1.34 + \frac{N + 1}{N - 1}}$$

> **System Role:** Supplies a low-variance, drift-resistant volatility estimate $\sigma$ that accounts for session gaps and feeds into the quoting engine's risk penalty.

---

### 2. Order-Flow Dynamics: Hawkes Process (1971)

Classical Avellaneda–Stoikov assumes order arrivals follow independent, memoryless homogeneous Poisson processes. In practice, limit order book events cluster in time and excite other events—for example, aggressive buy activity can trigger follow-on buys and ask cancellations.

A **multivariate Hawkes process** models the conditional arrival intensity $\lambda_m(t)$ for event type $m \in \{b, a\}$ (bid/buy, ask/sell) as a self-exciting point process:

$$\lambda_m(t) = \mu_m + \sum_{n \in \{b, a\}} \int_{-\infty}^{t} \alpha_{mn} e^{-\beta_{mn}(t - s)} \, \mathrm{d}N_n(s)$$

* **$\mu_m > 0$:** Baseline (exogenous) arrival intensity for event type $m$.
* **$\alpha_{mn} \ge 0$:** Excitation jump size showing how much a type-$n$ event boosts the arrival rate of type $m$.
* **$\beta_{mn} > 0$:** Exponential decay rate controlling how fast excitation fades.
* **$N_n(s)$:** Counting process of past events of type $n$.

> **System Role:** Replaces static Poisson arrival assumptions with time-varying intensities $\lambda_b(t)$ and $\lambda_a(t)$, helping the strategy widen spreads or skew quotes during bursts of one-sided or toxic flow.

---

### 3. Optimal Market Making: Avellaneda–Stoikov (2008)

The **Avellaneda–Stoikov model** frames quoting as a stochastic optimal control problem. Around a mid-price $s$ evolving as arithmetic Brownian motion $\mathrm{d}S_t = \sigma\,\mathrm{d}W_t$, the market maker posts bid ($p^b = s - \delta^b$) and ask ($p^a = s + \delta^a$) quotes to maximize expected exponential utility of terminal wealth over horizon $T$:

$$\max_{\delta^b, \delta^a} \mathbb{E}_t \left[ -\exp\left(-\gamma (X_T + q_T S_T)\right) \right]$$

Solving the Hamilton–Jacobi–Bellman (HJB) equation with exponential fill intensity $\lambda(\delta) = A e^{-\kappa \delta}$ yields two closed-form expressions:

* **Reservation (Indifference) Price ($r$):** Shifts reference value away from mid-price based on inventory $q$, risk aversion $\gamma$, volatility $\sigma$, and remaining horizon $(T - t)$:

$$r(s, q, t) = s - q \gamma \sigma^2 (T - t)$$

  When long ($q > 0$), $r < s$, pulling quotes lower to encourage buying/unwinding inventory.

* **Optimal Bid–Ask Spread ($\delta^b + \delta^a$):** Balances capture per fill against adverse selection and fill probability:

$$\delta^b + \delta^a = \gamma \sigma^2 (T - t) + \frac{2}{\gamma} \ln\left(1 + \frac{\gamma}{\kappa}\right)$$

> **System Role:** Serves as the core quoting engine, combining volatility ($\sigma$) and order-flow dynamics ($\lambda, \kappa$) into concrete bid and ask quotes.