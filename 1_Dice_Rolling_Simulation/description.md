# 1. Dice Rolling Simulation (Beginner)
- Goal: Simulate rolling dice and visualize probabilities.
- NumPy skills: Random number generation, array manipulation, probability calculations.
- Matplotlib skills: Bar plots, histograms, pie charts.
- Extra challenge: Simulate multiple players and compare outcomes

---

## **Dice Rolling Simulation Project Plan**

### **Goal**

Simulate rolling dice multiple times and visualize the outcome probabilities.

---

### **Step 1: Decide the parameters**

* Number of dice (e.g., 1 or 2)
* Number of rolls (e.g., 1000 or 10000)

---

### **Step 2: Simulate dice rolls with NumPy**

* Use `np.random.randint(1, 7, size=(num_rolls, num_dice))` to generate random rolls.
* Each row represents a roll; each column represents a die.

---

### **Step 3: Analyze results**

* If multiple dice, compute the **sum of each roll**: `np.sum(rolls, axis=1)`.
* Count how many times each possible outcome occurred using `np.bincount` or `np.histogram`.
* Compute probabilities: `counts / total_rolls`.

---

### **Step 4: Visualize results with Matplotlib**

* Use `plt.bar` for a bar chart of outcomes vs. frequency or probability.
* Label axes:

  * X-axis: Dice outcome (sum if multiple dice)
  * Y-axis: Frequency or probability

---

### **Step 5: Optional enhancements**

* Compare **1 die vs. 2 dice** in the same plot.
* Simulate **biased dice** by giving different probabilities to each face.
* Use a **pie chart** to show probabilities of outcomes.
* Animate dice rolling with `FuncAnimation` for fun visualization.

---

### **Example workflow**

1. Generate rolls → `rolls = np.random.randint(1,7,(1000,2))`
2. Compute sum → `sums = np.sum(rolls, axis=1)`
3. Count outcomes → `counts = np.bincount(sums)[2:]`
4. Plot → `plt.bar(outcomes, counts / counts.sum())`

---