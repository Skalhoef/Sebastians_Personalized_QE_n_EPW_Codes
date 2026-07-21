# Sebastian's Personalized Quantum ESPRESSO and EPW Codes

This repository contains a modified version of Quantum ESPRESSO 7.2 and EPW. The modifications provide higher-resolution density-of-states output and additional electron, phonon, and electron–phonon data for post-processing with the GRIT code.

## Changes relative to Quantum ESPRESSO 7.2

### Density of states

The output formatting in `q-e-qe-7.2/PP/src/dos.f90` has been changed from three to six digits after the decimal point for the energy column:

```text
f8.3  ->  f12.6
```

This prevents fine energy grids selected with `DeltaE` from being rounded to a resolution of 0.001 eV in the DOS output file. It is useful, for example, when the DOS is used in low-temperature specific-heat calculations.

### EPW data output

EPW has been extended with two optional printing modes:

- `prtgkk_sebbe` prints selected electron energies, phonon frequencies, wave vectors, and, when requested, electron–phonon matrix elements on the full fine grids.
- `print_fine_Fermi` prints electron energies and electron–phonon matrix elements only for fine-grid electronic states lying inside the Fermi window defined by `fsthick` and `fermi_energy`.

The matrix elements are averaged over degenerate phonon and electronic states in the same way as EPW's original `print_gkk` routine. Electron energies are written in eV, phonon frequencies and electron–phonon matrix elements in meV, and wave vectors in crystal coordinates.

## New EPW input variables

The following variables can be set in the `&inputepw` namelist:

| Variable | Type | Default | Purpose |
| --- | --- | --- | --- |
| `prtgkk_sebbe` | logical | `.false.` | Enable the full-grid personalized output. |
| `print_fine_Fermi` | logical | `.false.` | Enable output restricted to electronic states within `fsthick` of `fermi_energy`. |
| `sebbe_interacting` | logical | `.true.` | Calculate and print electron–phonon couplings. If `.false.`, run the personalized full-grid output in a non-interacting, band-energy-only mode. |
| `print_phonons` | logical | `.true.` | Print fine-grid phonon frequencies when `prtgkk_sebbe = .true.`. |
| `print_electrons` | logical | `.true.` | Print fine-grid electron energies when `prtgkk_sebbe = .true.`. |
| `n_wan_min` | integer | no added default | First band index included in the personalized output. This must be set when either personalized printing mode is used. |
| `n_wan_max` | integer | no added default | Last band index included in the personalized output. This must be set when either personalized printing mode is used. |

An example input fragment is:

```fortran
&inputepw
  ...
  n_wan_min        = 1
  n_wan_max        = 4
  prtgkk_sebbe     = .true.
  sebbe_interacting = .true.
  print_electrons  = .true.
  print_phonons    = .true.
/
```

Choose `n_wan_min` and `n_wan_max` for the band range in the particular EPW calculation; the values above are only illustrative.

## Generated files

### Full-grid interacting output

With `prtgkk_sebbe = .true.` and `sebbe_interacting = .true.`, EPW can create:

| File | Contents |
| --- | --- |
| `uniform_g_data.txt` | Electron–phonon matrix elements for the selected initial and final bands and all phonon modes. |
| `uniform_epsilon_data.txt` | Fine-grid electron energies for the selected bands. |
| `uniform_omega_data.txt` | Fine-grid phonon frequencies. |
| `uniform_k_data.txt` | Fine q-point coordinates associated with the phonon data. |

`uniform_epsilon_data.txt` is controlled by `print_electrons`; `uniform_omega_data.txt` is controlled by `print_phonons`. The q-point file is written when both switches are enabled.

### Full-grid non-interacting output

With `prtgkk_sebbe = .true.` and `sebbe_interacting = .false.`, EPW can create:

| File | Contents |
| --- | --- |
| `epsilon_data.txt` | Fine-grid electron energies for the selected bands. |
| `k_epsilon_data.txt` | Fine k-point coordinates associated with the electron energies. |
| `omega_data.txt` | Fine-grid phonon frequencies. |
| `k_data.txt` | Fine q-point coordinates associated with the phonon data. |

No `g_data.txt` values are written in non-interacting mode.

### Fermi-window output

With `print_fine_Fermi = .true.`, EPW creates:

| File | Contents |
| --- | --- |
| `SE_g_data.txt` | Electron–phonon matrix elements for fine-grid k points having at least one electronic state inside the Fermi window. |
| `SE_epsilon_data.txt` | Corresponding electron energies at k. |
| `SE_epsilon_kplusq_data.txt` | Corresponding electron energies at k+q. |
| `SE_omega_data.txt` | Fine-grid phonon frequencies. |

The Fermi-window condition used by the code is

```fortran
MINVAL(ABS(etf(:, k) - fermi_energy)) < fsthick
```

The personalized output files are opened in append mode. Remove or rename old files before starting a new calculation if each run should produce a fresh data set.

## Modified EPW source files

- `q-e-qe-7.2/EPW/src/epwcom.f90`: declares the new input and band-selection variables.
- `q-e-qe-7.2/EPW/src/epw_readin.f90`: adds the variables to `&inputepw` and defines the logical defaults.
- `q-e-qe-7.2/EPW/src/bcast_epw_input.f90`: broadcasts the new logical options in MPI calculations.
- `q-e-qe-7.2/EPW/src/ephwann_shuffle.f90`: invokes the new output routines in the standard-memory interpolation path.
- `q-e-qe-7.2/EPW/src/ephwann_shuffle_mem.f90`: invokes the same routines in the memory-optimized interpolation path.
- `q-e-qe-7.2/EPW/src/printing.f90`: implements `print_gkk_sebbe` and `print_fine_Fermi_constants`.

All other Quantum ESPRESSO and EPW functionality remains governed by the original Quantum ESPRESSO 7.2 documentation and license.
