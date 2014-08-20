.. CRITICAL TECHNICAL PRACTICE

:author: Esteban Fuentes
:email: esteban.fuentes@tum.de
:institution: Electric Drives and Power Electronics group at the Technical University of Munich

:author: Hector E. Martinez
:email: hector.martinez@tum.de
:institution: Physics Department T30F for Theory on Particles and Nuclear Physics at the Technical University of Munich


-------------------------------------------------------------------
SClib, a hack for straightforward embedded C functions in (I)Python
-------------------------------------------------------------------

.. class:: abstract

   We present SClib, a simple hack that allows easy and straightforward
   evaluation of C functions within python code, boosting flexibility for
   better trade-off between computation power and feature availability, such as
   visualization and existing computation routines in SciPy.

   We also present two use cases for SClib.

   In the first set of applications we use SClib to implement a Python script
   that solves efficiently the Schrödinger equation for bound-states in the
   context of particle physics. We present an introduction to the situations
   where this script have been used. We also describe the solution to the
   related problem of solving a set of coupled Schrödinger-like equations
   where SClib is used to implement the speed-critical parts of the code. We
   argue that when using SClib within IPython we can use NumPy and Matplotlib
   for the manipulation and visualization of the solutions in an interactive
   environment with no performance conceading.

   The second case is an engineering application. We use SClib to evaluate the
   control and system derivatives in a feedback control loop for electrical
   motors.  With this and the integration routines available in SciPy, we can
   run simulations of the control loop a la Simulink. The use of C code not
   only boosts the speed of the simulations, but also enables to test the
   exact same code that we use in the test rig to get experimental results.
   Again, integration with (I)Python gives us the flexibility to analyze and
   visualize the data.

.. class:: keywords

   embedded C code, particle physics, control engineering


SClib
=====

At the core of SClib [#]_ is ctypes [Hell]_, which actually does the whole
work: it maps Python data to C compatible data and provides a way to call
functions in DLLs or shared libraries.  SClib acts as glue: it puts things
together for the user, to provide him with an easy to use interface.

.. [#] The code for SClib and example use are availible at <https://github.com/drestebon/SClib>

The requirements for SClib are very simple: call a function on an array of
numbers of arbitrary type and size and return the output of the function, again
of arbitrary type and size.

The resulting interface is also very simple: A library is initialized in the
python side with the path to the DLL (or shared library) and a list with the
names of the functions to be called:

.. code-block:: python

   In [1]: import SClib as sc
   In [2]: lib = sc.Clib('test.so', ['fun'])

The functions are then available as a members of the library and can be called
with the appropriate number of arguments, which are one dimensional arrays of
numbers.  The function returns a list containing the output arrays of the
function:

.. code-block:: python

   In [3]: out, = lib.fun([0])

In the C counterpart, the function declaration must be accompanied with
specifications of the inputs and outputs lengths and types. This is
accomplished with the helper macros defined in sclib.h:

.. code-block:: c

   #include <sclib.h>
         PYO(fun, 1,   1);
   PYO_TYPES(fun, 1, INT);
         PYI(fun, 1,   1);
   PYI_TYPES(fun, 1, INT);
   void fun(int * out, int * in) {
       *out = 42;
   }

An arbitrary number of inputs or outputs can be specified, for example:

.. code-block:: c

   #include <math.h>
   #include <sclib.h>
         PYO(fun, 2,   1,     2);
   PYO_TYPES(fun, 2, INT, FLOAT);
         PYI(fun, 2,   1,     2);
   PYI_TYPES(fun, 2, INT, FLOAT);
   void fun(int * out0, float * out1,
            int * in0, float * in1) {
       *out0 = 42*in0[0];
       out1[0] = in1[0]*in1[1];
       out1[1] = powf(in1[0], in1[1]);
   }

In the function declaration, all the outputs must precede the inputs and must
be placed in the same order as in the PY macros.

These specifications are processed during compilation time, but only the number
of inputs and outputs is static, the lengths of each component can be
overridden at run time:

.. code-block:: python

   In [4]: lib.INPUT_LEN['fun'] = [10, 1]
   In [5]: lib.retype()

In these use cases the length of the arguments should be given to the function
through an extra integer argument.

In the function body, both inputs and outputs should be treated as one
dimensional arrays.


Application in Quarkonium Physics
=================================

Motivation
----------
The Schrödinger equation is the one the fundamental equations in physics for
describing non-relativistic quantum mechanical dynamics. For the applications
we will present in this section we will focus on the time-independent version
which in natural units is given by 

.. math::
   :label: schroe

   \left(-\frac{\nabla_{\mathbf r}^2}{2m}+V(\mathbf{r})\right)\psi(\mathbf{r}) = E\psi(\mathbf{r}).

It corresponds to an eigenvalue equation where the term inside the parenthesis
in l.h.s. is called the Hamiltonian operator, the value :math:`E`, its
eigenvalue, is the measurable quantity (the energy) associated with it,
:math:`m` is the mass (or the reduced mass if the system formed by more than
one particle) and the wavefunction, :math:`\psi(\mathbf{r})`, is the entity
containing all the information about the system, since its modulus squared
correspond to the probability density of a given measurement, it has to be
normalized to unity. The term :math:`V({\mathbf r})` in the Hamiltonian is
called the potential.

Since its discovery, the Schrödinger equation has played an important role in
our understanding of nature and it is present in almost every aspect of modern
physics. In this section we will review some cases where SClib has been used to
implement solutions of the computing problems associated with eq.
(:ref:`schroe`) that arise in the study of heavy quarkonia [#]_.

.. [#] For a comprehensive review of the status and perspectives of the
   research in heavy quarkonia we refer the reader to chapter four of
   [Bra14]_.

Quarkonium is a bound-state composed by a quark and its corresponding
antiquark. By heavy we mean states composed by the charm and bottom quarks,
called charmonium and bottomonium respectively. Due to its large mass, the top
quark decays before forming a bound state. For heavy quarkonium the relative
velocity between the quark and antiquark inside of the bound-system is believed
to be small enough for the system to be considered, at least in a first
approximation, non-relativistic, making it suitable for being described by eq.
(:ref:`schroe`). Considering the equal mass case with a spherically symmetric
potential, the angular part can be neglected (it correspond to the spherical
harmonics) and the relevant part of eq. (:ref:`schroe`) reduces to the
one-dimensional equation given by

.. math::
   :label: reduced

   \left[-\frac{1}{m}\frac{d^2}{dr^2}+\frac{l(l+1)}{r^2}+V(r)\right]y_{n,l}(r)=E_{n,l}y_{n,l}(r),

where :math:`r` is the relative distance between the quark and the antiquark,
:math:`l` is the angular momentum quantum number, :math:`m` is the (anti)quark
mass, :math:`y_{n,l}` is called the reduced wavefunction and the eigenvalue
:math:`E_{n,l}` is interpreted as the binding energy of the bound-system, where
:math:`n=0,1,2,\dots` accounts for the number of nodes (radial excitations) of
the wavefunction. The total mass of the quarkonium is then given by 

.. math::
   :label: lomass

    M=2m+E_{n,l}.

The potential :math:`V(r)` describes the quark-antiquark interaction, it is a
function of :math:`r` and :math:`\Lambda_{\rm QCD}`, the typical hadronic scale
(:math:`\sim 200\,{\rm MeV}`). For :math:`r\Lambda_{\rm QCD} \ll 1`
(short-distance regime) the potential may be evaluated perturbatively, but for
:math:`r\Lambda_{\rm QCD} \sim 1` (long-distance regime) it cannot. To
overcome this issue, models based on non-relativistic reductions of
phenomenological observations have been used to describe heavy quarkonia, one
these being the so-called Cornell potential
[Eich74]_, [Eich78]_, [Eich79]_)

.. math::
   :label: cornell

   V(r) = \frac{a}{r}+kr,

where :math:`a` and :math:`k` are unknown parameters which need to be fixed by
experimental (or lattice) data of some observable. This potential incorporates
two of the main observed characteristics of the quark-antiquark interaction: at
short distances it exhibits a Coulombic behavior and in the long-distance
regime the interaction is dominated by a confinement phase.

Since the beginning of the last decade, non-relativistic effective field
theories (EFT), in particular non-relativistic QCD (NRQCD) [Cas85]_, [Bod94]_
and potential NRQCD (pNRQCD) [Bra99]_, have become the state-of-the-art tools
for the study of heavy quarkonia (for review see [Bra04]_).  NRQCD is obtained
from QCD integrating out modes that scale like :math:`m`, while pNRQCD is
obtained from NRQCD integrating out modes that scale like the quark momentum
[#]_.

.. [#] These EFT exploit the hierarchy of energy scales present in the
    bound-system. If the relative velocity of between the quark and the antiquark,
    :math:`v`, is small, we have that :math:`mv^2(\sim E)\ll mv(\sim p) \ll m`,
    where :math:`p` is the momentum of the particles and :math:`E` its kinetic
    energy. If one is interested in studying a process that happen at the scale
    :math:`E` (like the binding) it is more suitable to integrate out degrees of
    freedoms with energies that scale like the other two higher scales, this is the
    motivation behind pNRQCD. For a detailed analysis of the scales in heavy
    quarkonia we refer the reader to [Bra04]_.

The physics of the modes that have been integrated out is encoded in Wilson
coefficients that must be calculated comparing at the same  scale the results
(observables, Green functions) of the EFT, with the ones of QCD (for NRQCD) or
NRQCD (for pNRQCD). A key feature of pNRQCD is that it allows the relativistic
corrections to the quark-antiquark potential to be organized as an expansion in
powers of :math:`1/m`, up to second order :math:`V(r)` can be written as

.. math::
    :label: pnrqcdpot

    V(r)=V^{(0)}(r)+\frac{V^{(1/m)}(r)}{m}+\frac{V^{(1/m^2)}(r)}{m^2},

where :math:`V^{(1/m)}` and :math:`V^{(1/m^2)}` are derived from QCD (through
the matching procedure with NRQCD). The details about :math:`V^{(1/m)}` and
:math:`V^{(1/m^2)}`  and how they are obtained are beyond the scope of this
document, however, we can list some of their features:

- They correspond to Green functions that in the short-distance regime can
  be computed in perturbation theory.
- In the long-distance regime they can be computed in in lattice QCD,
  however only some of these Green functions have been already calculated.
- Eq. (:ref:`cornell`) correspond, at least qualitatively, to the leading
  order :math:`V^{(0)}` in eq. (:ref:`pnrqcdpot`)

For the details about the derivation of the terms present in eq.
(:ref:`pnrqcdpot`) we refer the reader to refs. [Bra00]_ and
[Pin00]_. It is important to recall that, although it can not be
evaluated analytically in the whole range of :math:`r`, eq. (:ref:`pnrqcdpot`)
represents a definite model-independent expression for the quark-antiquark
potential, contrary to models like the one presented in eq. ({\ref{cornell}).

Including the relativistic corrections to the potential the expression for the
bound-state mass reads

.. math::
   :label: mass
   :type: eqnarray

   M&=&2m+E_{n,l}^{(0)}+\frac{\langle nl| V^{(1/m)}(r)|nl \rangle}{m}\\ \nonumber
    &+&\frac{\langle nl| V^{(1/m^2)}(r)|nl \rangle}{m^2}+\frac{1}{m^2}\sum_{m\neq n}^{\infty}\frac{| \langle nl|V^{(1/m)} | ml \rangle|^2}{E_{n,l}^{(0)}-E_{ml}^{(0)}},

where :math:`E_{il}^{(0)}` correspond to solve eq. (\ref{reduced}) with
:math:`V(r)=V^{(0)}(r)` and

.. math::

    \langle nl | f(r) | n'l' \rangle  \propto  \int_0^\infty dr\, y_{n,l}(r)f(r)y_{n'l'}(r),

where the proportionality factor will depended on the corresponding quantum
numbers of the operators appearing in :math:`V^{(1/m)}` and :math:`V^{(1/m^2)}`.

.. figure:: fig-1.png
   :align: center
   :figclass: htb

   Reduced wavefunctions :math:`y_{n,l}(r)` for two steps in the search of the
   eigenvalue :math:`E_{n=1\,l=1}`. For each step in the process to find the
   eigenvalue the nodes of the :math:`y_{n,l}` are counted, only when the
   value of :math:`E_{n,l}` correspond to an eigenvalue :math:`y_{n,l}` is not
   divergent. In the plot (a) :math:`E_{n,l} = 3.1\,m` and :math:`y_{n,l}`
   fulfills the condition of have one node, however, the accuracy in the value
   of :math:`E_{n,l}` is too low and the function diverges. In the plot (b)
   :math:`E_{n,l} = 3.10952\,m` so :math:`y_{n,l}\rightarrow 0` for larger
   values of :math:`r`. We have used the Cornell potential eq. (:ref:`cornell`)
   with parameters :math:`m = 1` :math:`a = 0.1`, :math:`k=0.5m^2`, all
   dimensions defined in terms of the mass.

Applications of SClib
---------------------

The simplest computational problem related to eq. (:ref:`reduced`) is to find
:math:`E_{n,l}` for a given :math:`n` and :math:`l`. Methods to solve this
problem have been implemented since long ago (see for instance [Fal85]_), in a
nutshell, the standard method consist in to apply two known constraints to the
reduced wavefunction :math:`y_{n,l}`:

- The number of nodes of :math:`y_{n,l}(r)` must be equal to :math:`n`.
- :math:`y_{n,l}(r)`  has to be normalizable

.. math::
   :label: norm

   \int_0^\infty dr[y_{n,l}(r)]^2 = 1.

In general :math:`y_{n,l}(r)` will diverge except when :math:`E_{n,l}`
correspond to an eigenvalue. The procedure to find the eigenvalue consists in
to perform a scan of values of :math:`E_{n,l}` until :math:`y_{n,l}(r)`  has
:math:`n` nodes and converges for a large enough value of :math:`r` (see Fig.
1). This implies that for each test value of :math:`E_{n,l}` eq.
(:ref:`reduced`) must to be (numerically) solved.  A popular [#]_ Mathematica
[Mat9]_ implementation of this method to solve eq. (:ref:`reduced`) has been
available in [Luc98]_.  This script has the advantage that the user can profit
from the Mathematica built-in functions to plot, integrate or store the
resulting wavefunctions, however, it has a very poor performance.  With the
goal of mimic the advantages of this script but without compromising in speed
we have developed SChroe.py [#]_, a Python script that uses SClib to implement
the speed-critical parts of the algorithm. In this script the wavefunctions are
stored as NumPy arrays [NumPy]_ so when the script is run within IPython [IPy]_
together with SciPy [SciPy]_, NumPy and Matplotlib [Mplot]_ the user can profit
of the same or more flexibility as with the Mathematica script plus a boosted
speed. In table 1 we compare the performance of SChroe.py against other
implementations of the same algorithm.

.. [#] The paper describing the script ranks fifth among the most cited papers
   (91 citations) of the International Journal of Modern Physics C with the last
   citation from  July 2014.}

.. [#] Code available in <https://github.com/heedmane/schroepy/>


.. table:: Time in seconds taken to compute the eigenvalues and reduced wavefunctions for the Cornell potential eq. (\ref{cornell}). The column Python correspond to the implementation of the algorithm in Python without using SClib. The parameters of the potential are the same as in fig. (\ref{wavefunctions}). All the scripts were tested in the same machine, a notebook with a 2.4 Ghz core i5 processor (dual core) and 8 GB of RAM.

   +----------+-------------------------+--------------------+-------+----------+
   | :math:`n`| :math:`E_{n,l=1}\,\,[m]`| schroe.nb [Luc98]_ | Python| SChroe.py|
   +----------+-------------------------+--------------------+-------+----------+
   | 0        | 2.15789                 | 98.88              | 25.46 | 11.11    |
   +----------+-------------------------+--------------------+-------+----------+
   | 1        | 3.10952                 | 124.14             | 30.95 | 12.65    |
   +----------+-------------------------+--------------------+-------+----------+
   | 2        | 3.93850                 | 135.68             | 35.32 | 14.93    |
   +----------+-------------------------+--------------------+-------+----------+
   | 20       | 13.5995                 | 370.0              | 88.04 | 32.13    |
   +----------+-------------------------+--------------------+-------+----------+

In [Bra14]_ SChroe.py has been used to evaluate the relativistic corrections to
the mass spectrum of quarkonium in the long-distance regime. In that paper the
relativistic corrections :math:`V^{(1/m)}` and :math:`V^{(1/m^2)}` appearing in
(:ref:`mass`) were evaluated assuming the hypothesis that in the long-distance
regime the interaction between the quark and the antiquark can be described by
a string. In Fig. 2 we show some of the energy levels (masses) corresponding to
the string spectrum. It is noteworthy to mention that all the numerical
calculations and plots of that paper were done with (I)Python using the SciPy
library.

.. figure:: fig-2.png
   :align: center
   :figclass: htb

   Long-range energy levels of the first triplet quarkonium state. The lines
   are calculated from eq. (:ref:`mass`) using the relativistic corrections
   derived from the string hypothesis [Bra14a]_. The leading order (LO)
   correspond to eq. (\ref{cornell}) setting :math:`a=0` and :math:`k=1` (in
   the plot labeled :math:`\sigma`) and :math:`m=3\sqrt{k}`.  This plot shows
   the relative size of the next-to-leading-order (NLO) correction (the term
   proportional to :math:`1/m` in the r.h.s. of eq.  (:ref:`mass`) and the
   newly computed next-to-next-to-leading-order (NNLO) corrections (the terms
   proportional :math:`1/m^2`). For more details see [Bra14a]_.

An application in which the speed of SChroe.py plays an important role is
fixing the parameters of the potential given some experimental input. For
instance, consider the problem of finding the parameters :math:`a` and
:math:`k` of eq.  (:ref:`cornell`) together with :math:`m`, given the
experimental values of the masses of three different quarkonium states. If
relativistic corrections are included, in order to find the parameters  we must
solve a system of three equations like eq. (:ref:`mass`). For each probe value
of :math:`(a,k,m)` we have to find the eigenvalues and reduced wavefunctions of
eq. (:ref:`reduced`) and then with these values evaluate the sums and integrals
in (:ref:`mass`). A parameter fixing of this type was necessary to implement in
[Bra14b]_. The implementation has been carried out using SChroe.py together
with a mixture of C and SciPy functions using SClib to link both environments
[#]_.

.. [#] Some of the code will be available once the paper appear online.

Another related computational problem that arises from the study of heavy
quarkonium hybrids, bound-states composed by a quark-antiquark pair plus an
exited gluon, is to solve a system of :math:`N` Schr\"odinger-like coupled
equations.  Explicitly the system to solve reads

.. math::
    :label: coupled

    \left(-\frac{\delta_{ij}}{m}\frac{d^2}{dr^2}+V_{ij}(r,l)\right)u_{j,(n,l)}(r)=E_{n,l}\,u_{i,(n,l)}(r),

where :math:`i = 1,2,..N` and the angular momentum dependence has been included
in the potential matrix. A method to solve this equation for the case
:math:`N=2` has been implemented in [Ber14]_. The method relies on an extension
of the nodal theorem [Ama95]_ and convergence conditions for the components of
the vector wavefunction :math:`u_{j,(n,l)}(r)`. The extension of the nodal
theorem states that the number of nodes of the determinant of the matrix
:math:`U_{n,l}(r)`, whose columns are :math:`N` lineal-independent solutions of
eq. (\ref{coupled}), is equal to :math:`n`. The procedure then consist in a
scan of values :math:`E_{n,l}`; in each step the set of equations
(\ref{coupled}) is solved and the nodes of :math:`|U_{n,l}(r)|` are counted for
a large enough interval of :math:`r`. As in the one-dimensional case, if
:math:`E_{n,l}` approached to an eigenvalue the components of
:math:`u_{j,(n,l)}` converge for large :math:`r`. In the solution presented in
[Ber14]_ the performance-intensive parts of the implementation rely on C
functions linked to the (I)Python interface trough SClib.

As an example of the application of the method implemented in [Ber14]_, in Fig.
3 we show the results for the search of the first two eigenvalues and
wavefunctions with the matrix potential given by

.. math::
    :label: matrixpotential

    V_{ij}(r,l) = \begin{pmatrix} \frac{l(l+1)+2}{mr^2}+F_0(r) & -\frac{2\sqrt{l(l+1)}}{mr^2} \\ -\frac{2\sqrt{l(l+1)}}{mr^2} &  \frac{l(l+1)}{mr^2}+F_1(r) \end{pmatrix}

where

.. math::

    F_i(r)=\ln(a_i+b_ir).

.. figure:: fig-3.png
   :align: center
   :figclass: htb

   Solutions for the components of the vector wavefunction :math:`u_{n,l}(r)`
   for the first two eigenvalues (:math:`l=1`) of eq. (:ref:`coupled`) with the
   matrix potential given in (:ref:`matrixpotential`). We have used
   :math:`m=1`, :math:`a_0=1`, :math:`b_0=0.5`, :math:`a_1=2` and
   :math:`b_1=0.1`. The eigenvalues are :math:`E_{n=0,l=1} = 1.01727\,m` for
   fig. (a) and  :math:`E_{n=1,l=1} = 1.18789\,m` for fig. (b).

In all the applications described in this section the combination of SClib and
the SciPy library within IPython provided a powerful interactive environment
based entirely on open source software for solving problems that require a high
performance and visualization tools.

Application in Control Engineering
==================================

.. figure:: fig-4.png

    General scheme of a control system.

Most control systems have the structure depicted in Fig 4.  :math:`G` is the
plant, it represent the natural phenomena we wish to control.  We usually
describe it using ordinary differential equations:

.. math::
    :label: eq:dxdt

    G:\;\left\{
        \begin{array}{rl}
            \frac{dx}{dt} &= f(x,u,d)\\
            y & = c(x,u,d).
        \end{array}
        \right.

:math:`x` represents the internal state of the plant and :math:`y` its output
(the measurements). :math:`d` is an independent variable, usually not
measurable, named the perturbation and :math:`u` is the actuation: the degree
of freedom used by the controller :math:`C` to achieve the control goal
:math:`r`. In general the controller is a function of the measurements and the
reference :math:`r`:

.. math::

    C:\;u = \pi(y,r),

but it also may comprise internal states. They are commonly used to reconstruct
the state :math:`x` out of the history of :math:`y` and :math:`u`. The latter
systems are called state observers and the whole is called feedback control.

We use SClib to put together a simulator for these kind of systems.  Both the
system derivatives :math:`f(\cdot)` and the control :math:`\pi(\cdot)` are
written in C and are evaluated using SClib. As stated before, the system state
represents a natural phenomena, therefore it is natural to describe it as a
continuous time variable, as eq. \eqref{eq:dxdt} suggests. To calculate the
system state we have to solve this equation. In our simulator this is achieved
using numerical methods, namely the integration routines available in
\verb+scipy.integrate+. On the other hand, the controller is usually
implemented in a real-time computer, which can only sample :math:`y` at a fixed
interval (called :math:`h`): it is a discrete-time system.  This means, that
the simulator only needs to evaluate :math:`\pi(\cdot)` at given times.

Traditional controllers took the form of linear filters, which could even be
implemented using analog circuitry. As control techniques and requirements
advance, more complex controllers are devised. Many modern control techniques
are based on optimization methods. Time-optimal controllers, for example,
require the solution of an usually very complex optimization problem, to find a
control :math:`u` that leads the system state :math:`x` towards its target :math:`r` in minimum
time [Gru11]_:


latex-preamble: \DeclareMathOperator*{\argmin}{arg\,min}

.. math::
    :label: feedback

    u ^*= \pi^*(x)=\underset{\pi\in U,\, x\in X}{\operatorname{argmin}}\left\lbrace T_{x}(u)\right\rbrace.


Here :math:`T_x(u)` is the time required to lead :math:`x` towards its target
and :math:`X` and :math:`U` are the regions where we want :math:`x` and
:math:`u` to be confined, they constitute the constraints for the control
problem.  These kind of controllers require exhaustive computation and it is
natural to implement them in C.

For motivation, we present the results for a minimum-time control strategy for
a relatively simple and well known problem, the double integrator
[Fu13]_:

.. latex::
    :usepackage: nicefrac

.. math::
    :label: eq:di

    \frac{d}{dt}\left[
        \begin{array}{c}
            x_0\\
            x_1
        \end{array}
    \right]
    =
    \left(
    \begin{array}{c}
        \nicefrac{u}{\tau_0}\\
        \nicefrac{x_0}{\tau_1}
    \end{array}
    \right).

The relevance of this system lays in that it models many mechanical systems:
:math:`u`, :math:`x_0` and :math:`x_1` may represent acceleration, speed and
position, for example.

Fig. 5 presents a minimum time control strategy for this system.

.. figure:: fig-5.png
   :figclass: htb

   Time optimal control for the double integrator considering
   :math:`\tau_0=\tau_1=5`, :math:`u\in[-1,1]`, :math:`h=1` and :math:`x\in[-1,
   1]\times\mathbb{R}`.

The form of :math:`\pi(x)` for this case reveals its non-linear nature.

Fig. 6 presents the trajectory developed by the state using this control
strategy and random initial conditions.

.. figure:: fig-6.png
   :figclass: htb

   Time optimal trajectories for the double integrator, with random initial
   conditions.

These results were obtained using SClib and the devised simulator. The example
code is ready to reproduce them.

The main advantage we obtained from this work was that, since we were using a
Linux based real time system in our test rig, we could use exactly the same
code for the simulations and the experimental tests.  Another feature of this
work is that it effectively replaces Simulink in all of our use cases using
only free software.


Final Remark
============

We hope the applications of SClib scope beyond the ones listed in this paper
since we believe it provides a simple but powerful way to boost (I)Python
performance.


Acknowledgments
===============

H.M. acknowledges financial support from DAAD and the TUM Graduate School
during the realization of this work.


References
==========

.. [Hell]   Heller. *The ctypes module.*,
            https://docs.python.org/3.4/library/ctypes.html#module-ctypes
.. [Bra14]  N. Brambilla, S. Eidelman, P. Foka, S.Gardner, A. S. Kronfeld, M. G. Alford, R. Alkofer and M. Butenschoen et al.,
            *QCD and Strongly Coupled Gauge Theories: Challenges and Perspectives*
            arXiv:1404.3723
.. [Eich74] E. Eichten, K. Gottfried, T. Kinoshita, J. B. Kogut, K. D. Lane and T. M. Yan,
            *The Spectrum of Charmonium,*
            Phys.  Rev.  Lett.     34 , 369 (1975)
            Erratum-ibid.     36 , 1276 (1976)
.. [Eich78]  E. Eichten, K. Gottfried, T. Kinoshita, K. D. Lane and T. M. Yan,
            *Charmonium: The Model,*
            Phys.  Rev.  D   17 , 3090 (1978)
            Erratum-ibid.  D   21 , 313 (1980)
.. [Eich79] E. Eichten, K. Gottfried, T. Kinoshita, K. D. Lane and T. M. Yan,
            *Charmonium: Comparison with Experiment,*
            Phys.  Rev.  D   21 , 203 (1980).
.. [Cas85]  W. E. Caswell and G. P. Lepage,
            *Effective Lagrangians for Bound State Problems in QED, QCD, and Other Field Theories,*
            Phys.  Lett.  B   167 , 437 (1986).
.. [Bod94]  G. T. Bodwin, E. Braaten and G. P. Lepage,
            *Rigorous QCD analysis of inclusive annihilation and production of heavy quarkonium,*
            Phys.  Rev.  D   51 , 1125 (1995)
            Erratum-ibid.  D    55 , 5853 (1997)
.. [Bra04]  N. Brambilla, A. Pineda, J. Soto and A. Vairo,
            *Effective field theories for heavy quarkonium,*
            Rev.  Mod.  Phys.      77 , 1423 (2005)
.. [Bra99] N. Brambilla, A. Pineda, J. Soto and A. Vairo,
            *Potential NRQCD: An Effective theory for heavy quarkonium,*
            Nucl.  Phys.  B    566 , 275 (2000)
.. [Bra00]   N. Brambilla, A. Pineda, J. Soto and A. Vairo,
            *The QCD potential at O(1/m),*
            Phys.  Rev.  D    63 , 014023 (2001)
.. [Pin00]  A. Pineda and A. Vairo,
            *The QCD potential at O(1/m^2): Complete spin dependent and spin independent result,*
            Phys.  Rev.  D    63 , 054007 (2001)
            Erratum-ibid.  D    64 , 039902 (2001)
.. [Fal85]  P. Falkensteiner and H. Grosse and F. Schoeberl and P. Hertel
            Comput. Phys. Comm.    34 , 287 (1985)
.. [Luc98]  W. Lucha and F. F. Schoberl,
            *Solving the Schrodinger equation for bound states with Mathematica 3.0,*
            Int.  J.  Mod.  Phys.  C    10 , 607 (1999)
.. [Mat9]   Wolfram Research, Inc.
            Mathematica Version 9.0 (2012)
.. [Bra14a] N. Brambilla, M. Groher, H. E. Martinez and A. Vairo,
            *Effective string theory and the long-range relativistic corrections to the quark-antiquark potential,*
            arXiv:1407.7761
.. [Bra14b] N. Brambilla, H. E. Martinez and A. Vairo,
            TUM-EFT 40/13, In preparation.
.. [SciPy]  Eric Jones and Travis Oliphant and Pearu Peterson and others
             http://www.scipy.org/  (2001--)
.. [NumPy]  Stefan van der Walt, S. Chris Colbert and Gaël Varoquaux. The NumPy Array: A Structure for Efficient Numerical Computation, Computing in Science  & Engineering,    13 , 22-30 (2011)
.. [Ber14]  M. Berwein and H. E. Martinez,
            TUM-EFT 48/14, In preparation.
.. [Ama95]  H. Amann and P. Quittner,
            *A nodal theorem for coupled systems of Schroedinger equations and the number of bound states,*
            Journal of Mathematical Physics    36 , 4553 (1995),
            doi:10.1063/1.530907.
.. [Mplot] John D. Hunter. Matplotlib: A 2D Graphics Environment, Computing in Science  & Engineering,    9 , 90-95 (2007)	
.. [IPy]    Fernando Perez and Brian E. Granger. IPython: A System for Interactive Scientific Computing, Computing in Science  & Engineering,    9 , 21-29 (2007)
.. [Gru11]  L. Gruene and J. Pannek,  *Nonlinear Model Predictive Control: Theory and Algorithms,*
            Springer-Verlag, 2011.
.. [Fu13]   E. Fuentes, D. Kalise, J. Rodriguez, and R. Kennel
            *Cascade-free predictive speed control for electrical drives,*
            Industrial Electronics, IEEE Transactions on , vol. PP, no. 99, pp. 1--1, 2013.