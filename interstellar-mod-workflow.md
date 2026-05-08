# Interstellar Star System Mod — Production Workflow

> **Base:** ballisticfox's Sol mod · **Physics:** Principia (N-body, ICRS)  
> **Roles:** Tech (Principia proto / Kopernicus cfg / MM patches) · Art (textures / atmosphere / terrain visuals)

---

## Stage 1 — Research

*Data gathering and references before any production work begins.*

**Tech**
- Understand Principia's existing `sol_gravity_model.proto.txt` and `sol_initial_state_jd_*.proto.txt` structure
- Understand Sol mod folder layout and cfg conventions (`Sol-Configs`, `Sol-Visuals`)
- Survey Kopernicus cfg syntax from reference packs (Beyond Home, Extrasolar)
- Understand ICRS coordinate conversion pipeline: parallax → metres, proper motion → m/s

**Art**
- Research stellar type lighting: M-dwarf red, K-type orange, G-type solar, etc.
- Collect reference imagery per candidate system (NASA, ESA, artist concepts)
- Study Scatterer and EVE cfg structure from existing packs — understand tunable parameter ranges
- Survey Parallax terrain shader and scatter object configs for reference

---

## Stage 2 — Planning

*System selection, scope decisions, and architecture agreed by both.*

**Both**
- Survey all candidate star systems — full list, not just the obvious ones
- Decide which systems to implement via community vote or discussion
- For each selected system: confirm planet/moon list and scope (what gets modelled vs left out)
- Decide release structure: one pack per system (modular) or single combined release
- Confirm Principia coordinate approach: Sol at origin, other stars added as perturbers in ICRS

---

## Stage 3 — Technical Implementation

*Principia proto data and Kopernicus cfg skeleton. Art runs biome/lighting prep in parallel.*

**Tech**
- Write conversion script: Gaia DR3 observational data → ICRS Cartesian state vectors
- Add `body` blocks to `sol_gravity_model.proto.txt` (gravitational parameter, mean radius)
- Add Cartesian state vectors to `sol_initial_state_jd_*.proto.txt` (x, y, z, vx, vy, vz)
- Write Kopernicus cfg per planet/moon — Keplerian elements relative to host star
- Write MM patches to register new bodies without touching Sol's own files

**Art (parallel)**
- Confirm star lighting colour values for each system — feeds into Kopernicus `Star` node `lightColor`
- Draft biome map layouts per planet before texture work begins

---

## Stage 4 — Planet Production (GeoGen)

*Procedural terrain generation and texture export pipeline.*

**Art**
- Design planet surface in GeoGen using node-based workflow (noise, erosion, masks)
- Export: heightmap, colormap, normalmap — up to 16K resolution
- Post-process in Photoshop/Krita:
  - Apply polar distortion and fix polar pinching
  - Flip heightmap vertically (KSP renders textures upside-down)
  - Clean up seams at the equator and poles
- Convert to DDS: BC4 for heightmaps, BC7 for colormaps/normalmaps (matching Sol's format)
- Place final textures in `PluginData/` folder for Kopernicus on-demand loading

**Tech (parallel)**
- Verify texture path references in Kopernicus cfg match the exported filenames and folder structure

---

## Stage 5 — Visual Production

*Atmosphere, clouds, terrain shader, and atmospheric effects.*

**Art**
- Write Scatterer cfg per body: scattering coefficients, fog density, sunset colour
- Write EVE cfg per body: cloud layers, altitude, opacity, rotation speed, auroras where applicable
- Write Volumetric Clouds cfg (V5 primary + V3 free fallback in parallel)
- Write Parallax cfg: terrain shader parameters, scatter object placement
- Write Firefly atmospheric effects cfg per body

**Tech (support)**
- Confirm all visual cfgs live in the mod's own folder — no writes into Sol's directory
- Verify DDS formats are correct for KSPTextureLoader

---

## Stage 6 — Integration & Testing

*Full verification against Sol + Principia on a clean install.*

**Tech**
- Verify Principia loads the extended ephemeris — check orbital stability of added bodies
- Verify Kopernicus body registration: SOI, gravity, rotation axis
- Check MM patch application order — no conflicts with Sol's own patches (`:AFTER[Sol]`, etc.)
- Test compatibility: Firefly, Distant Object Enhancement, PlanetShine

**Art**
- In-game atmosphere check: sunrise/sunset colour, limb colour visible from orbit
- Parallax terrain shader visual verification on each body's surface
- Confirm star lighting colour reads correctly on planet surfaces
- Distant Object Enhancement: verify planets appear as correct-colour point sources

---

## Stage 7 — Release

*GitHub, CKAN, and community announcement.*

**Both**
- GitHub release — per-system modules or single combined package (per scope decision in Stage 2)
- CKAN metadata — dependencies: Kopernicus, Principia, Sol
- KSP forum thread: install guide, screenshots, known issues
- Discord announcement (including Sol/ballisticfox community)
- Document V3/V5 Volumetrics split — clear install instructions for users without Patreon access

---

## Key principles

1. Before production starts, collect as much data as possible and establish solid sources for every system being implemented.
2. Use that data to drive the visuals as closely as the science allows, from atmospheric colour and temperature down to generating EVE texture drafts via atmospheric circulation simulation (MPAS for N2/O2-dominant atmospheres, ExoPlaSim for others).
3. GeoGen is the standard tool for planet surface production.
4. Sol mod is the reference for all file structure and cfg conventions.

---

## Appendix — Art accuracy philosophy & planned tooling

### Philosophy

Where observational data exists, the art follows it. Where it doesn't — surface detail, speculative geology, undiscovered moons — creative decisions are made, but they should be consistent with the physical constraints of the system (stellar flux, atmospheric composition, equilibrium temperature, etc.).

The goal is that every visual choice can be traced back to either a data source or a clearly stated creative assumption.

---

### Planned art pipeline tools

These tools are in design and will be built before or alongside the first full system production run.

**Tool 1 — Stellar colour picker**  
Input: stellar effective temperature (K)  
Output: Planckian locus RGB, Scatterer `sunColor` value, surface lighting colour (pre- and post-atmospheric scattering)

**Tool 2 — Planetary sky colour calculator**  
Input: atmospheric composition (component ratios %), surface pressure (atm), scale height (km) or temperature for auto-calculation, stellar temperature (linked from Tool 1)  
Output: altitude-layered Rayleigh/Mie coefficients, sky colour preview at dawn/noon/dusk, **Scatterer cfg block ready to paste**  
Note: exponential atmosphere profile as the baseline, with manual adjustment sliders for aerosols and other non-calculable factors.

**Tool 3 — Research-to-art briefing generator**  
Input: multiple papers (PDF upload), supplementary parameter overrides  
Output: physical calculation results (equilibrium temperature, atmosphere retention probability, etc.) + art brief describing surface conditions, sky appearance, and recommended visual direction  
Note: source tracing per output value — every number can be attributed to a specific paper. Design deferred; to be discussed with collaborator.

---

### Cloud texture pipeline

The intended workflow for generating EVE Volumetric Clouds textures from simulation data:

```
GeoGen heightmap
  + orbital parameters (semi-major axis, eccentricity, rotation period)
  + atmospheric composition & surface pressure
          ↓
  Atmospheric circulation simulation
          ↓
  Cloud distribution / precipitation output
          ↓
  EVE volumetric texture conversion
```

**Simulation tool choice:**
- **N₂/O₂-dominant atmospheres** (Earth-analog): MPAS is viable. KWP (Kerbal Weather Project) confirmed this pipeline — the developer explicitly noted that MPAS IR satellite output can serve as a basis for realistic cloud textures in visual mods. Resolution of 2×2° is achievable on a modern desktop PC in days, not weeks.
- **Other atmospheric compositions** (CO₂-dominant, CH₄-rich, etc.): ExoPlaSim — an open-source lightweight GCM that accepts arbitrary atmospheric parameters. Less precise than MPAS but appropriate for creative planets where the composition diverges significantly from Earth.

Reference: [Kerbal Weather Project v1.0](https://forum.kerbalspaceprogram.com/topic/199347-18x-111x-kerbal-weather-project-kwp-v100/) — confirms the MPAS → cloud texture concept; used 5 years of simulation data and explicitly mentions EVE as a downstream application.

---

## Tech stack reference

| Layer | Mod |
|---|---|
| Planet framework | Kopernicus Bleeding Edge + KSPTextureLoader |
| Physics | Principia |
| Base solar system | Sol (ballisticfox) |
| Atmosphere scattering | Scatterer |
| Clouds / aurora | EVE-Redux |
| Volumetric clouds | Blackrack Volumetric Clouds (V5 / V3 fallback) |
| Deferred rendering | Deferred (blackrack) |
| Post-processing | TUFX |
| Terrain shader | Parallax Continued |
| Re-entry effects | Firefly + FireflyAPI |
| Planet visibility | Distant Object Enhancement |
| Planet lighting | PlanetShine |
| Planet generation | GeoGen (JangaFX) |
| In-game editor | KittopiaTech (dev only) |
