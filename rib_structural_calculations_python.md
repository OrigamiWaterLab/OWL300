# Ribs

## Introduction

The OWL300 uses horizontal ribs to withstand the outward hydrostatic forces. The rib is a board of plastic sheet that runs perpendicularly on the outside of the plant. It is held in place on either end with a pipe at right angle that acts as a pin. Here's one rib:

And here's the outer structure assembly:

The goal of this document is to determine optimal rib spacing and size.

## Rib Spacing

The first failure mode to consider is when the side walls are not supported often enough, and they rupture in between ribs. Or more realistically, they deflect enough to allow a significant amount of water to bypass some unit processes. For this scenario, we consider the ribs to be infinitely strong, and we place them further and further apart to determine at what point the wall will fail.

To determine the total force acting on the wall generated by hydrostatic pressure, we can use the following equation:

```python
from aide_design.play import *

def vertical_wall_hydrostatic_force(h, w, water_height_above_wall=0*u.m, rho=exp.DENSITY_WATER, g=u.gravity):
  """Calculate the total hydrostatic force acting on a vertical wall.

  Parameters
  ----------
  h : L
      height of the wall resisting the force
  w : L
      width of the wall resisting the force
  water_height_above_wall : L
      height of the water above where the wall starts. This is used to
      calculate the static pressure difference between the two sides of the wall
  rho : kg/m^3
      The density of the fluid causing the hydrostatic forces
  g : m/s^2
      Gravity... I imagine this won't be changing!

  Returns
  -------
  Force
      The total force generated across the wall

  Tests
  -----
  >>> vertical_wall_hydrostatic_force(0.4*u.m, 4*u.inch)
  > <Quantity(79.7084512, 'newton')>
  """
  return (rho*g*h**2/2*w + rho*g*water_height_above_wall*h*w).to(u.newton)

# Test for 10 cm

vertical_wall_hydrostatic_force(0.4*u.m, 4*u.inch)


```

Now, we model the wall as a simply supported beam, with the ribs being the support at either end. According to wikipedia, the governing equation is:

>the elastic deflection (at the midpoint C) of a beam, loaded at its center, supported by two simple supports is given by:
>\[
>\delta_{C}={\frac  {FL^{3}}{48EI}}
>\]
>where:
>* F  = Force acting on the center of the beam
>* L = Length of the beam between the supports
>* E = Modulus of Elasticity
>* I = Area moment of inertia of cross section

Showing that the deflection increases with the distance between the ribs cubed, which suggests that the point of worse deflection will be at the top of the tank, where the ribs are the furthest apart (even though the wall is withstanding the least hydrostatic forces.)

So to determine the max distance between ribs, given the max allowable deformation, we can solve for L:

```python
# This could and probably should be implemented with sympy. They have a Beam class under sympy.physics.continuum_mechanics.beam
def beam_length_deflection(max_deflection, forces, E, geometry, boundary_conditions, **kwargs):
  """
  Beam lengths given a max deflection not to exceed.

  This function can handle the following geometries types:

  * rectangle, with 'w' signifying the width of the cross-section, and 'h' is the height.

  This function can handle the following boundary condition types:

  * simply_supported, where one end is pinned, and the other is on a roller
  * fixed, where both ends are fixed so they can resist the bending moment

  This can handle the following force types:

  * point there is a point load at 'a' distance from one end
  * uniformly_distributed is a load with 'omega' force/length across the full length
  * uniformly_varying is a load that varies from 'omega_0' force/length to 'omega_L' force/length

  Return the length of the beam that ensures no more than the specified max deflection.

  >>> beam_length_deflection(5.5*u.mm, 80*u.newton, 2240*10**6*u.pascal, {"type": "rectangle", "w":4*u.inch, "h":0.25*u.inch}, {"type": "fixed"})
  <Quantity(0.6352897664188726, 'meter')>

  """

  #Calculate the Area moment of inertia for the beam cross section:
  g_type = geometry["type"]
  if g_type == "rectangle":
    I = geometry["w"]*geometry["h"]**3/3
  #https://www.engineersedge.com/material_science/moment-inertia-gyration-7.htm
  elif g_type == "angle":
    a,t = geometry["a"], geometry["t"]
    y = a-(a**2+a*t-t**2)/(a*(2*a-t))
    I = 1/3*(t*y**3 + a*(a-y)**3 - (a-t)*(a-y-t)**3)
  else:
    raise UserWarning("no valid geometry type specified")

  # Calculate deflection based on the boundary conditions
  bc_type = boundary_conditions["type"]
  if bc_type == "simply_supported":
    L = ((max_deflection*48*E*I/forces)**(1/3)).to(u.m)
  if bc_type == "fixed":
    L = (max_deflection*192*E*I/forces)**(1/3)



  return L.to(u.m)


print(beam_length_deflection(5.5*u.mm, 80*u.newton, 2240*10**6*u.pascal, {"type": "rectangle", "w":4*u.inch, "h":0.25*u.inch}, {"type": "fixed"}))

def deflection_simply_supported(F, E, h, w, L):
  """
  >>> deflection_simply_supported(80*u.newton, 2240*10**6*u.pascal, 0.25*u.inch, 4*u.inch, 0.4*u.m)
  <Quantity(5.491450537208758, 'millimeter')>
  """
  I = (w*h**3)/3
  deflection = ((F*(L**3))/(48*E*I)).to(u.mm)
  return deflection

deflection_simply_supported(80*u.newton, 2240*10**6*u.pascal, 0.25*u.inch, 4*u.inch, 0.4*u.m)



```

The following are the properties for acrylic, PVC and ABS:
Materials  | Modulus of Elasticity  |   |  
--|---|---|--
ABS  | 2.6 GPa  |   |  
Acrylic  |   |   |  
PVC  |   |   |  

Now we should be able to use these two equations to determine the max allowable rib spacing, and therefore the force/meter that each rib is expected to counteract.

```python

def support_spacing_and_contact_force(t, max_deflection, **kwargs):
  """
  The max spacing of supports of a simply supported beam that has no more deflection than max_deflection, and is acted upon by hydrostatic forces. Consider a vertical plate holding back some fluid. Now the goal is to know the vertical spacing of the next rib that holds the plate vertical and keeps it from deforming. t is the thickness of the plate

  We first take a stab at the force, and then refine the guess until we arrive within 99% of the correct answer.
  """
  # Guess a force
  F = 1000*u.newton

  # The error so far
  err = 1.0

  # The modulus of elasticity for ABS
  E = 2240 * 10**6 *u.pascal

  w = 4*u.inch

  force = {}

  while abs(err) > 0.01 :
    L = beam_length_deflection(max_deflection, F, 2240*10**6*u.pascal, {"type": "rectangle", "w":w, "h":t}, {"type": "simply_supported"})
    F_new = vertical_wall_hydrostatic_force(L, w)
    err = (F_new-F)/(F_new+F)
    F = (err + 1)*F
    print(F)

  return L,F


support_spacing_and_contact_force(0.25*u.inch, 1*u.mm)
# >>> (<Quantity(0.28, 'meter')>,
# >>>  <Quantity(1588, 'newton')>)

```

This suggests that the first uppermost two ribs need to be spaced within 22 cm of one another to guarantee the maximum deflection is within 1 mm.

Now we'll get the spacing of the additional ribs by chaining these answers together and producing a list of spacings and relative rib forces.

```python


```

Let's verify this result with Fusion's built in hydrostatic simulator.



Therefore, to optimize materials by equalizing rib loading, fewer ribs are needed in the higher portion of the plant.

The goal of the ribs is to reduce deformation at the center of each wall span.

The hydrostatic pressure increases linearly with depth, according to:


Luckily, Fusion360 has a built in Hydrostatic force static analysis simulator that we can use to determine the maximum acceptable spacing for a given maximum deflection:


Let's test all the functions:
```python
import doctest
doctest.testmod(verbose=True)
```
