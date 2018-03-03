# beam_length_central_deflection


```python
import sympy.physics.units as u
from sympy import symbols, init_printing, Eq, simplify
from sympy.solvers import solve

# enable latex printing
init_printing()

# beam symbols
h, w, E, L, delta = symbols('h, w, E, L, delta')

# hydrostatic pressure
rho, g, h_water, p = symbols('rho g h_water p')
pressure = (rho * g * h_water)
# force/length due to hydrostatic pressure from above (continuous load)
f_uniform = pressure*w
# force/length max that decreases to 0 from the varying static load
f_declining = pressure.subs(h_water, L)

# moment of inertia for a rectangular cross-section beam
I = w*h**3/3

# deflection of beam calculation for fixed_beam
# https://www.engineeringtoolbox.com/beams-fixed-both-ends-support-loads-deflection-d_809.html
hydrostatic_beam = Eq(delta, f_declining*L**4/(764*E*I) + f_uniform*L**4/(764*E*I))

Length = hydrostatic_beam.subs({h_water: 0*u.meter, delta: 5*u.millimeter, rho: 998*u.kilogram/u.meter, g: 9.8*u.meter/u.second**2, w: 4*u.inch, E : 2240*10**6*u.pascal, h: 0.25*u.inch})

Length

hi = Eq(10*u.meter, (2.25425223*h*u.second)**4*w**3)

solve(hi, h)
solve(Length, L)

solve(u.convert_to(Length, u.meter), L)


```
