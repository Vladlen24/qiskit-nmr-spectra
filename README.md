# Predicting NMR Spectra on a Quantum Computer

This project explores how to simulate liquid-state 1H NMR spectra with Qiskit (`nmr_advanced.ipynb`).

First we map an N-spin NMR Hamiltonian to qubit operators, apply a
rotating-frame shift, build Trotterized time-evolution circuits, estimate
time-dependent transverse magnetization, and reconstruct the frequency-domain
NMR spectrum with a Fourier transform. The results are compared against spectra
computed with HQS Spectrum Tools where reference data is available.

## Main Idea

The distinctive part of this solution is the clusterization approach used for
the initial X-basis states.

The direct trace-based NMR formulation requires summing over all X-basis product
states:

```text
2^N initial states
```

for an N-spin system. This becomes expensive very quickly. For example:

```text
2 spins  -> 4 initial states
5 spins  -> 32 initial states
12 spins -> 4096 initial states
```

Here it is suggested approach where these states are grouped into sectors by their total
X-magnetization. Instead of evaluating every bitstring, the notebook samples one
representative bitstring from each sector:

```python
for k in range(number_spins + 1):
    bitstring = sample_bitstring_in_sector(number_spins, k)
    bits_list.append(bitstring)
```

Each sector is then weighted by the exact binomial coefficient:

```python
weights.append(comb(N_binom, k_binom))
```

This changes the number of required initializations from exponential to linear:

```text
2^N -> N + 1
```

For the included systems this means:

```text
2 spins  -> 3 representative sectors instead of 4 states
5 spins  -> 6 representative sectors instead of 32 states
12 spins -> 13 representative sectors instead of 4096 states
```

The practical consequence is a much smaller number of circuits/Estimator
evaluations, which directly reduces execution time. The per-circuit qubit count
still matches the number of spins in the Hamiltonian, but the reduction in
state preparations makes larger spin systems much more tractable than the
brute-force basis-state approach.

## Repository Contents

- `nmr_advanced.ipynb` - main implementation with the clusterized advanced
  workflow.
- `nmr_intermediate.ipynb` - intermediate version of the NMR workflow.
- `files/` - Hamiltonian JSON files, reference spectrum data, and molecule
  images.
- `img/` - generated spectrum and clustering comparison figures.

## Supported Example Molecules

The notebooks include data for:

- `bmse000368` - cis-3-chloroacrylic acid, 2 observed 1H spins.
- `C6H5NO2` - nitrobenzene, 5 observed 1H spins.
- `exo-dicyclopentadiene_DFT` - 12 observed 1H spins.

## How It Works

1. Load an N-spin Hamiltonian from a `struqture_py` JSON file.
2. Subtract the mean Z term to move into a rotating frame and reduce the
   Hamiltonian scale for Trotterization.
3. Convert the Hamiltonian to Qiskit's `SparsePauliOp`.
4. Build a Trotterized time-evolution circuit.
5. Prepare representative X-basis sector states instead of every X-basis state.
6. Estimate total X and Y magnetization with Qiskit's Estimator.
7. Reconstruct correlation functions using binomial sector weights.
8. Apply damping and FFT to obtain the NMR spectrum.
9. Compare the simulated spectrum with HQS Spectrum Tools reference results.

## Requirements

The notebooks use:

- Python
- Qiskit `>=2.2,<3`
- `struqture_py`
- NumPy
- SciPy
- Matplotlib
- Jupyter / ipykernel / ipympl

The first cell in the notebook installs the required packages:

```python
%pip install -q "qiskit>=2.2,<3" struqture_py numpy scipy matplotlib ipykernel ipympl
```

## Notes

The clusterization optimization reduces the number of initial states and
execution jobs. It does not change the Hamiltonian encoding itself: an N-spin
system is still represented on N qubits in the current implementation.

For the 12-spin example the included HQS Spectrum Tools
reference file may contain an issue, so that case cannot currently be fully
validated against the stored reference spectrum.
