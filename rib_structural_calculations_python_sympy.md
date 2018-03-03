# beam_length_central_deflection


```python
import sympy.physics.units as u
from sympy.physics.continuum_mechanics.beam import Beam
from sympy import symbols, init_printing, Eq

init_printing()

h, w, E, L = symbols('h, w, E, L')
rho, g, h, p = symbols('rho g h p')

p.equals(rho * g * h)

I = w*h**3/3
b = Beam(L, E, I)
b.apply_load(L*u.acceleration_due_to_gravity*rho)
```
