# Pandas/NumPy/SciPy Integration and xp.Dot Performance

## xp.Dot Performance: Avoiding Symbolic Expression Overhead

**Critical rule:** Pre-compute coefficient matrices with NumPy before involving variables. Avoid creating intermediate symbolic expressions.

**What are symbolic expressions?** When you write `2*x + 3*y`, Xpress stores a formula object, not a number. When `xp.Dot()` operates on arrays of already-complex symbolic expressions, it must expand nested formulas algebraically -- extremely slow.

```python
# GOOD: Numeric computation first, then involve variables
Q = A @ B @ A.T              # NumPy computation
result = xp.Dot(x, Q, x)     # Single symbolic operation

# AVOID: Intermediate symbolic expressions
temp = xp.Dot(x, A)          # Creates complex expressions (SLOW)
result = xp.Dot(temp, B, temp)
```

## Integration with NumPy/Pandas

### Enhanced Pandas Integration (Xpress 9.8+)

Starting with Xpress 9.8, you can store decision variables directly in Pandas DataFrame columns using `dtype='xpressobj'`. This enables using native Pandas operations for model building.

**Key concept:** Use `dtype='xpressobj'` when adding Xpress variables to DataFrame columns.

```python
import xpress as xp
import pandas as pd

p = xp.problem("Portfolio")

# Load data into DataFrame
stocks_df = pd.DataFrame({
    'stock': ['A', 'B', 'C', 'D'],
    'return': [0.12, 0.10, 0.07, 0.08],
    'sector': ['Tech', 'Finance', 'Tech', 'Healthcare']
})

# Add decision variables directly to DataFrame - MUST use dtype='xpressobj'
stocks_df['allocation'] = pd.Series(
    p.addVariables(len(stocks_df), vartype=xp.continuous),
    dtype='xpressobj'  # Essential for Pandas compatibility!
)

# Binary selection variables
stocks_df['selected'] = pd.Series(
    p.addVariables(len(stocks_df), vartype=xp.binary),
    dtype='xpressobj'
)
```

### Vectorized Modeling Patterns with Pandas

Once variables are in the DataFrame, standard Pandas operations work directly:

**1. Weighted sums (objectives and constraints):**
```python
# Objective: maximize total return
p.setObjective(
    (stocks_df['return'] * stocks_df['allocation']).sum(),
    sense=xp.ObjSense.MAXIMIZE
)

# Weighted average constraint
p.addConstraint(
    (stocks_df['esg_score'] * stocks_df['allocation']).sum() >= 70
)
```

**2. Boolean indexing for conditional constraints:**
```python
# Constraint only on high-risk stocks
high_risk = stocks_df[stocks_df['risk'] > 0.7]
p.addConstraint(high_risk['allocation'].sum() <= 0.20)

# Constraint on specific sector
tech_stocks = stocks_df[stocks_df['sector'] == 'Tech']
p.addConstraint(tech_stocks['allocation'].sum() <= 0.30)
```

**3. Groupby for category constraints:**
```python
# Limit allocation per sector - generates one constraint per sector
p.addConstraint(
    stocks_df.groupby('sector')['allocation'].sum() <= 0.25
)

# Multi-level groupby for nested categories
p.addConstraint(
    stocks_df.groupby(['sector', 'region'])['allocation'].sum() <= 0.15
)
```

**4. Linking binary and continuous variables:**
```python
# If selected, allocate at least 1%
p.addConstraint(stocks_df['allocation'] >= 0.01 * stocks_df['selected'])

# If selected, allocate at most 20%
p.addConstraint(stocks_df['allocation'] <= 0.20 * stocks_df['selected'])
```

**5. Extract solution back to DataFrame:**
```python
p.optimize()
stocks_df['solution'] = p.getSolution(stocks_df['allocation'])

# Post-processing with Pandas
approved = stocks_df[stocks_df['solution'] > 0.001]
print(approved.groupby('sector')['solution'].sum())
```

### Pandas 3.0 Compatibility

**Xpress 9.9 fully supports Pandas 3.0** -- do NOT pin `pandas<3.0` for 9.9+ code.

Earlier versions (pre-9.9) had a breaking issue where `groupby()` on columns containing Xpress variables failed with Pandas 3.0+. This is fixed in 9.9.

```python
# Fine in Xpress 9.9 + Pandas 3.0+
p.addConstraint(df.groupby('sector')['allocation'].sum() <= 0.25)
```

**Only pin `pandas<3.0` if the code must run on Xpress < 9.9.**

### NumPy Array Operations with xpress.ndarray

`xpress.ndarray` is a subclass of `numpy.ndarray` that customizes operators for constraint creation:

```python
# p.addVariables with integer args returns xpress.ndarray
flow = p.addVariables(n_origins, n_destinations)  # 2D array

# Matrix operations work naturally
p.addConstraint(flow.sum(axis=1) <= supply)  # Row sums (outflow per origin)
p.addConstraint(flow.sum(axis=0) >= demand)  # Column sums (inflow per dest)

# Element access
p.addConstraint(flow[0, 0] + flow[1, 1] <= 10)
```

### SciPy Sparse Matrix Support

`xp.Dot()` supports SciPy sparse matrices (CSR and CSC formats) for efficient large-scale model building:

```python
import scipy.sparse as sp

# Create sparse coefficient matrix
A = sp.random(1000, 5000, density=0.01, format='csr')
x = p.addVariables(5000)
rhs = np.random.random(1000)

# Efficient constraint creation
p.addConstraint(xp.Dot(A, x) <= rhs)  # Creates 1000 constraints efficiently
```

This is orders of magnitude faster than loop-based formulations for large sparse problems.

### Performance Tips for Pandas/NumPy Integration

1. **Always use `dtype='xpressobj'`** when adding variables to DataFrame columns
2. **Use vectorized operations** (.sum(), element-wise *) instead of explicit loops
3. **Use xp.Dot()** for matrix-vector products with large coefficient matrices
4. **Use SciPy sparse** for large sparse problems (>1% density threshold varies)
5. **Groupby works in Xpress 9.9 + Pandas 3.0** -- no workaround needed

### Example: Portfolio Risk Calculation

For `risk = w' * E * C * E' * w`:

```python
n_assets = 2500
n_factors = 250
exposure_mat = np.random.rand(n_assets, n_factors)   # E
factor_cov   = np.random.rand(n_factors, n_factors)  # C
weights = p.addVariables(n_assets, name="w")

# FAST: Pre-compute coefficient matrix with NumPy first
factor_exposure = exposure_mat @ factor_cov @ exposure_mat.T  # NumPy: 2500x2500
risk = xp.Dot(weights, factor_exposure, weights)  # Single symbolic operation
# Time: ~0.8 seconds

# SLOW: Intermediate symbolic expressions (200x slower!)
exposure = xp.Dot(weights, exposure_mat)          # Creates 250 complex expressions
risk = xp.Dot(exposure, factor_cov, exposure)     # Must expand nested expressions
# Time: ~165 seconds
```

**Why the difference?** Approach 2 creates 250 symbolic expressions each with 2500 linear terms, then must algebraically expand all quadratic cross-terms. Approach 1 is pure NumPy until the final symbolic step.
