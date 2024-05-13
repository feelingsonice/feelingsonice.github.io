+++
title = 'asd'
date = '2018-04-28'
readTime = 'true'
math = 'true'
+++

# Fibonacci Sequence and Binet's Formula

The Fibonacci sequence is a series of numbers where each number is the sum of the two preceding ones, usually starting with 0 and 1. That is, the sequence starts 0, 1, 1, 2, 3, 5, 8, 13, 21, and so on. The explicit formula for the Fibonacci sequence, also known as Binet's Formula, allows us to find the nth Fibonacci number directly without needing to compute all the previous Fibonacci numbers.

## Derivation of Binet's Formula

1. **Definition**:
   Let \(F_n\) denote the nth Fibonacci number. The Fibonacci sequence can be defined by the recurrence relation:

   $$
   F_n = F_{n-1} + F_{n-2}
   $$

   with initial conditions:

   $$
   F_0 = 0, \quad F_1 = 1
   $$

2. **Characteristic Equation**:
   Assume a solution of the form \(F_n = r^n\). Substituting into the recurrence relation, we get:

   $$
   r^n = r^{n-1} + r^{n-2}
   $$

   Simplifying, we find the characteristic equation:

   $$
   r^2 = r + 1
   $$

   Solving this quadratic equation, \(r^2 - r - 1 = 0\), gives:

   $$
   r = \frac{1 \pm \sqrt{5}}{2}
   $$

   Let \(\phi = \frac{1 + \sqrt{5}}{2}\) (the golden ratio) and \(\psi = \frac{1 - \sqrt{5}}{2}\).

3. **General Solution**:
   The general solution to the recurrence relation is:

   $$
   F_n = A\phi^n + B\psi^n
   $$

   Using the initial conditions to solve for \(A\) and \(B\):

   $$
   F_0 = A + B = 0 \quad \Rightarrow \quad B = -A
   $$

   $$
   F_1 = A\phi + B\psi = 1
   $$

   Solving these equations, we find:

   $$
   A = \frac{1}{\sqrt{5}}, \quad B = -\frac{1}{\sqrt{5}}
   $$

4. **Binet's Formula**:
   Substituting back, we get:
   $$
   F_n = \frac{\phi^n - \psi^n}{\sqrt{5}}
   $$
   This is the explicit formula for the nth Fibonacci number.

## Using Binet's Formula

To find any Fibonacci number \(F_n\), substitute the value of \(n\) into Binet's formula. This formula allows direct computation of \(F_n\) without the iterative or recursive calculation typically associated with the Fibonacci sequence.

# Finding the Explicit Formula for Homogeneous Linear Recurrence Relations Using Eigenvalues

Homogeneous linear recurrence relations are a type of sequence defined by a recurrence relation where each term is a linear combination of its previous terms. To find an explicit formula for these sequences using eigenvalues, follow these steps:

## Steps to Derive the Formula

1. **Define the Recurrence Relation**:
   Assume a linear recurrence relation of order \( k \):

   $$
   a_n = c_1a_{n-1} + c_2a_{n-2} + \ldots + c_ka_{n-k}
   $$

   where \( c_1, c_2, \ldots, c_k \) are constants.

2. **Form the Companion Matrix**:
   Construct the companion matrix \( C \) associated with the recurrence relation:

   $$
   C = \begin{bmatrix}
   c_1 & c_2 & \cdots & c_{k-1} & c_k \\
   1 & 0 & \cdots & 0 & 0 \\
   0 & 1 & \cdots & 0 & 0 \\
   \vdots & \vdots & \ddots & \vdots & \vdots \\
   0 & 0 & \cdots & 1 & 0
   \end{bmatrix}
   $$

3. **Find Eigenvalues**:
   Calculate the eigenvalues of the matrix \( C \). These are the roots \( \lambda_1, \lambda_2, \ldots, \lambda_k \) of the characteristic polynomial:

   $$
   \lambda^k - c_1\lambda^{k-1} - c_2\lambda^{k-2} - \ldots - c_k = 0
   $$

4. **General Solution**:
   The general solution to the recurrence relation can be expressed as:

   $$
   a_n = A_1\lambda_1^n + A_2\lambda_2^n + \ldots + A_k\lambda_k^n
   $$

   where \( A*1, A_2, \ldots, A_k \) are constants determined by the initial conditions \( a_0, a_1, \ldots, a*{k-1} \).

5. **Use Initial Conditions**:
   Set up equations using the initial conditions to solve for the constants \( A_1, A_2, \ldots, A_k \). This typically results in solving a system of linear equations.

## Example

For the Fibonacci sequence, \( a*n = a*{n-1} + a\_{n-2} \) with initial conditions \( a_0 = 0, a_1 = 1 \):

- The companion matrix is:
  $$
  C = \begin{bmatrix}
  1 & 1 \\
  1 & 0
  \end{bmatrix}
  $$
- The characteristic polynomial is:
  $$
  \lambda^2 - \lambda - 1 = 0
  $$
- Solving for eigenvalues gives \( \lambda_1 = \frac{1 + \sqrt{5}}{2}, \lambda_2 = \frac{1 - \sqrt{5}}{2} \).
- The general solution is:
  $$
  a_n = A_1\left(\frac{1 + \sqrt{5}}{2}\right)^n + A_2\left(\frac{1 - \sqrt{5}}{2}\right)^n
  $$
- Use initial conditions to find \( A_1 \) and \( A_2 \) similar to the Binet's formula derivation.

This method can be applied to any homogeneous linear recurrence relation to find an explicit formula for the sequence.
