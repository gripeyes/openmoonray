# DwaBase native dielectric specular / IOR audit

Audit date: 2026-07-16

Audited revisions:

- `moonray`: `e403e8021ca760580d09a9a648a1fd9731a27de2`
- `moonshine`: `5bec7a4be722894ba70f22760d194c862eb67280`

The worktrees contained pre-existing unrelated changes. This audit changed no production shader or renderer behavior. It added only focused renderer tests.

## Executive finding

No Fresnel/IOR defect, double Fresnel, double energy compensation, or additive diffuse/specular error was found in the native `DwaBaseMaterial` path for `metallic=0`, `specular=1`, and `transmission=0`.

The normal-incidence dielectric response is exact: IOR 1.5 produces `F0=0.04`. The attribute is an absolute material IOR; MoonRay constructs the relative interface from the ray's current medium and correctly swaps incident/transmitted IOR on exit. The default isotropic lobe is Walter-style GGX with separable Smith masking-shadowing. Artist roughness is squared once inside the Cook-Torrance lobe, so `alpha=max(roughness^2, 0.001)` unless normal-map AA or ray roughness clamping increases the input first.

MoonRay is using an intentional DWA model with two approximations that may differ from other renderers:

1. a Kelemen/Kulla-Conty directional-albedo compensation lobe, enabled only when input roughness is greater than 0.5; and
2. an approximate, view-dependent `OneMinusRoughFresnel` attenuation for diffuse beneath rough specular, rather than attenuation by the actual integrated directional reflectance of the rough BRDF.

Neither explains an IOR 1.5 to 1.3 appearance shift. Compensation is absent through roughness 0.5 and raises the normal-direction IOR 1.5 reflectance at roughness 0.8 only from 0.02284 to 0.02498 (9.3%). Changing IOR from 1.5 to 1.3 lowers `F0` and rough-lobe energy by about 57%, a much larger effect.

## A. Native source path

1. `moonshine/dso/material/DwaBase/DwaBaseMaterial.cc`
   - `collectAttributeKeys()` binds native `specular`, `refractive_index`, `metallic`, metallic colors, `roughness`, anisotropy, transmission, diffuse, coat, normals, and normal-AA controls.
   - `DwaBaseMaterial::shade()` calls `resolveParameters()` and then `createLobes()`.
2. `moonshine/lib/material/dwabase/DwaBase.cc`
   - `DwaBase::update()` computes which parameter groups are required.
   - `resolveSpecularParams()` saturates specular, roughness, and metallic to their documented ranges.
   - `resolveParameters()` reads `refractive_index` directly and clamps only to positive epsilon.
3. `moonshine/lib/material/dwabase/DwaBaseLayerable.h`
   - `computeMicrofacetRoughness()` normally returns the artist roughness unchanged. Toksvig normal AA may increase it before lobe creation.
4. `moonshine/lib/material/dwabase/DwaBaseLayerable.cc`
   - `createLobes()` creates conductor, dielectric reflection/transmission, and diffuse lobes in physical layer order.
   - Smooth dielectric uses `MirrorBSDF`; rough isotropic dielectric uses `MicrofacetIsotropicBSDF` with GGX by default and Smith geometry.
5. `moonray/lib/rendering/shading/BsdfBuilder.cc`
   - `addComponent(const MicrofacetIsotropicBSDF&)` resolves interface IOR, constructs the native Cook-Torrance lobe, installs exact dielectric Fresnel once, and stages the underlayer attenuator.
6. `moonray/lib/rendering/shading/Ior.h`
   - `ShaderIor` resolves incident and transmitted media for entering, exiting, and thin geometry.
7. `moonray/lib/rendering/shading/bsdf/Fresnel.h`
   - `DielectricFresnel` implements exact unpolarized dielectric Fresnel.
   - `OneMinusRoughFresnel` implements the approximate underlayer attenuation.
8. `moonray/lib/rendering/shading/bsdf/cook_torrance/BsdfCookTorrance.cc`
   - Implements Beckmann and GGX Cook-Torrance reflection, Smith masking-shadowing, roughness squaring, sampling, and the compensation-lobe hook.
9. `moonray/lib/rendering/shading/bsdf/cook_torrance/energy_compensation/CookTorranceEnergyCompensation.cc`
   - Implements the Kelemen/Kulla-Conty compensation term using tabulated directional albedo.

The scalar C++ and vector ISPC implementations follow the same structure. The numerical audit exercised the C++ lobe directly.

## B. Exact formulas and conventions

### Parameters

- `refractive_index`: absolute material IOR, default 1.5. It is not an F0 or relative-IOR input.
- `specular`: scalar weight in `[0,1]`, default 1. The documentation intends binary 0/1 use for physical plausibility. The value becomes the dielectric Fresnel weight.
- dielectric specular color: none. `metallic_color` and `metallic_edge_color` affect only the conductor branch. `transmission_color` tints transmission, not ordinary dielectric reflection. Iridescence is a separate optional modifier.
- `roughness`: perceptual roughness in `[0,1]`, default 0.5.
- `metallic`: default 0. At zero, no conductor lobe is created. At nonzero values DwaBase uses its own layered conductor/dielectric semantics; this is not Autodesk Standard Surface's documented linear statistical metalness mixture. That does not affect the audited `metallic=0` case.

### Interface IOR and Fresnel

For ordinary front-facing geometry in air:

`eta_i = 1`, `eta_t = refractive_index`.

On exit, MoonRay swaps these. Thin geometry is always treated as viewed from outside. The exact dielectric implementation applies Snell's law, total internal reflection, and the average of squared parallel/perpendicular polarization amplitudes. At normal incidence it reduces to:

`F0 = ((eta_t - eta_i) / (eta_t + eta_i))^2`.

Thus from air:

`F0 = ((ior - 1) / (ior + 1))^2`.

The specular weight is applied once inside the Fresnel object. There is no second IOR-level scale.

### GGX microfacet BRDF

For input roughness `r`:

`alpha = max(r^2, 0.001)`.

Normal-map Toksvig AA and the ray's minimum-roughness clamp can increase `r` before this conversion. There is no other ordinary parameter remap. The sampling distribution is widened slightly at grazing view angles to reduce variance; evaluation remains at the original alpha.

The reflection lobe is:

`f_r(wo, wi) = F(h.wi) D_GGX(h) G1(wo) G1(wi) / (4 |n.wo| |n.wi|)`

with:

`D_GGX = alpha^2 / [pi cos(theta_h)^4 (alpha^2 + tan(theta_h)^2)^2]`

and separable Smith:

`G1(mu) = 2 / [1 + sqrt(1 + alpha^2 (1-mu^2)/mu^2)]`.

### Multiple-scattering compensation

This is not a Heitz random-walk multiple-scattering BSDF. It is the Kelemen 2001 compensation lobe as presented by Kulla and Conty in 2017, using precomputed single-scatter directional albedo `E(mu,r)`:

`f_ms = [(1-E_o)(1-E_i)/(1-E_avg)] * [F_avg^2 E_avg / (1-F_avg(1-E_avg))] / pi`.

For entering dielectrics, MoonRay approximates:

`F_avg = (eta-1) / (4.08567 + 1.00071 eta)`.

The implementation computes a sampling mixture weight:

`w_ms = max(0, r-0.5)`.

It also uses `w_ms > 0` as the gate for evaluating the compensation term. Consequently compensation is exactly absent at roughness 0.1, 0.25, and 0.5, and active at 0.8 in the requested matrix. It is not applied twice.

### Diffuse beneath specular

All relevant lobes are added with `BSDFBUILDER_PHYSICAL`, meaning they receive attenuation from earlier layers and attenuate later layers. Diffuse is not simply added at full strength.

For a rough dielectric over diffuse, the builder wraps diffuse with `OneMinusRoughFresnel`. With weighted Fresnel `F_w` and `mu_o=n.wo`:

`t = 1 - (1-r^2)^3`

`A(mu_o,r) = lerp(1-F_w(mu_o), 1-F_w(1), t)`.

The diffuse lobe is multiplied by `A`. This accounts for Fresnel and specular weight, but it is only a view-direction approximation to rough directional albedo. It is not full layered transport and is not the exact `1-reflectance(rough_BRDF)` formula specified by Autodesk Standard Surface. The approximation can also break strict BSDF reciprocity for the combined underlayer, although the GGX specular lobe itself is reciprocal.

## C. Smooth dielectric Fresnel results

The new test calls the native exact `DielectricFresnel` directly at normal incidence, from air, with unit weight and no other lobe:

| IOR | expected F0 | MoonRay result |
|---:|---:|---:|
| 1.0 | 0 | 0 |
| 1.1 | 0.0022676 | 0.0022676 |
| 1.3 | 0.0170132 | 0.0170132 |
| 1.5 | 0.0400000 | 0.0400000 |
| 2.0 | 0.1111111 | 0.1111111 |

All RGB channels pass at absolute tolerance `1e-7`.

## D. Rough dielectric numerical results

Definitions:

- `Rdir`: directional-hemispherical specular reflectance for `wo=n`.
- `Rhh`: cosine-weighted hemispherical average of directional specular reflectance.
- `peak`: peak normal-view BRDF value in `sr^-1`.
- `HWHM`: polar half-maximum angular radius of the normal-view lobe.
- `Tdir` and `Thh`: total specular plus a unit-albedo Lambert diffuse substrate using MoonRay's actual rough-Fresnel underlayer attenuation. This is a white-furnace stress case.

The integrator uses the native GGX lobe and its native sampler with a deterministic low-discrepancy sequence. Values are raw linear. The last few digits are numerical estimates.

| r | IOR | Rdir | Rhh | peak | HWHM deg | Tdir | Thh |
|---:|---:|---:|---:|---:|---:|---:|---:|
| 0.10 | 1.0 | 0 | 0 | 0 | 0 | 1.00000 | 1.00000 |
| 0.10 | 1.3 | 0.01701 | 0.06062 | 13.6035 | 0.735 | 1.00000 | 1.00079 |
| 0.10 | 1.5 | 0.04000 | 0.09125 | 31.9833 | 0.735 | 1.00000 | 1.00101 |
| 0.10 | 2.0 | 0.11110 | 0.16008 | 88.8425 | 0.735 | 0.99999 | 1.00095 |
| 0.25 | 1.0 | 0 | 0 | 0 | 0 | 1.00000 | 1.00000 |
| 0.25 | 1.3 | 0.01697 | 0.05375 | 0.34663 | 4.632 | 0.99995 | 1.00038 |
| 0.25 | 1.5 | 0.03987 | 0.08377 | 0.81497 | 4.632 | 0.99987 | 1.00110 |
| 0.25 | 2.0 | 0.11069 | 0.15232 | 2.26381 | 4.632 | 0.99957 | 1.00043 |
| 0.50 | 1.0 | 0 | 0 | 0 | 0 | 1.00000 | 1.00000 |
| 0.50 | 1.3 | 0.01584 | 0.03338 | 0.02166 | 20.136 | 0.99883 | 0.99775 |
| 0.50 | 1.5 | 0.03705 | 0.05846 | 0.05093 | 20.133 | 0.99705 | 0.99661 |
| 0.50 | 2.0 | 0.10225 | 0.12079 | 0.14147 | 20.129 | 0.99114 | 0.98881 |
| 0.80 | 1.0 | 0 | 0 | 0 | 0 | 1.00000 | 1.00000 |
| 0.80 | 1.3 | 0.01066 | 0.01658 | 0.00506 | >=90 | 0.99365 | 0.99751 |
| 0.80 | 1.5 | 0.02498 | 0.03324 | 0.01077 | >=90 | 0.98498 | 0.99083 |
| 0.80 | 2.0 | 0.06988 | 0.08023 | 0.02679 | >=90 | 0.95877 | 0.96681 |

The tiny maximum `Thh` overshoot is 0.11%. It is within the scale of the numerical integration and MoonRay's documented approximate rough-Fresnel underlayer attenuation. There is no material energy excess remotely large enough to explain a 1.5-to-1.3 appearance shift. At medium/high roughness the approximation is mildly energy-losing rather than energy-adding.

Compensation impact at roughness 0.8:

| IOR | Rdir single | Rdir compensated | increase | Rhh single | Rhh compensated | increase |
|---:|---:|---:|---:|---:|---:|---:|
| 1.3 | 0.00985 | 0.01066 | 8.3% | 0.01583 | 0.01658 | 4.8% |
| 1.5 | 0.02284 | 0.02498 | 9.3% | 0.03124 | 0.03324 | 6.4% |
| 2.0 | 0.06244 | 0.06988 | 11.9% | 0.07321 | 0.08023 | 9.6% |

## E. Why IOR 1.5 becomes silver-like faster than IOR 1.3

Within MoonRay itself, the effect is the expected Fresnel ratio, not an anomalous remap:

- `F0(1.5) / F0(1.3) = 0.04 / 0.0170132 = 2.351`.
- At roughness 0.5, `Rdir(1.5) / Rdir(1.3) = 2.339`.
- At roughness 0.8, the same ratio is 2.342.

Lowering IOR from 1.5 to 1.3 therefore removes about 57% of the specular energy at both smooth and rough settings. The visual match around 1.3 is consistent with another setup having a lower effective specular weight/F0, but MoonRay is not internally converting 1.5 to an excessive Fresnel value.

Broad rough reflection can look pale because it carries achromatic environment illumination over a wider solid angle while the substrate is colored. The numerical data show that MoonRay broadens and lowers the peak as expected; it does not increase normal-direction integrated reflectance with roughness. At IOR 1.5, `Rdir` falls from 0.04000 at roughness 0.1 to 0.02498 at 0.8.

The remaining cross-render difference must be isolated at closure level. Relevant non-equivalences include exact rough-BRDF implementation/compensation, diffuse attenuation strategy, base/specular weights and colors, normal AA, ray roughness clamping, and which incident radiance is visible to the specular lobe. “Same-looking” controls are not sufficient evidence of equal closures.

## F. Mathematical validity

The dielectric Fresnel, GGX NDF, and separable Smith masking-shadowing implementation are mathematically valid standard microfacet components. The compensation formula is a recognized physically motivated approximation to missing multiple scattering.

The full Dwa layer stack is not an exact layered BSDF. Its view-only rough-Fresnel underlayer attenuation is an approximation and can deviate slightly from exact directional-albedo conservation and reciprocity.

## G. Physical plausibility

The audited ordinary dielectric is physically plausible. It has correct F0, correct medium-side handling, no meaningful white-furnace energy excess, and sensible roughness trends. The small total-energy deviations are characteristic of the approximate layer combination, not evidence of a grossly nonphysical silver boost.

## H. Compatibility with Karma / Arnold / Standard Surface

The current Dwa model is not specified to be numerically appearance-compatible with Karma or Arnold.

DreamWorks' 2017 primary source describes the DWA Physical Materials as a DreamWorks parameter-blended uber-material. It says Refractive used anisotropic Cook-Torrance/Beckmann with a non-mappable IOR default of 1.5, and Solid Dielectric shared that specular model. Current public source defaults to GGX, showing that the internal DWA model evolved, not that it became Autodesk Standard Surface.

Autodesk Standard Surface also specifies GGX and squared perceptual roughness, but its layer equation attenuates the substrate by `1 - specular * specular_color * reflectance(specular_brdf)`, where `reflectance` is the rough BRDF's directional albedo. DwaBase instead uses `OneMinusRoughFresnel`. Standard Surface also exposes dielectric `specular_color`, a separate base weight, and a documented statistical metalness mix that DwaBase does not share.

MaterialX's PBS specification permits renderer-dependent microfacet implementations and says implementations are expected to preserve rough-surface energy, commonly using multiple-scattering compensation. That is not a numerical appearance guarantee. Arnold documentation identifies its Standard Surface implementation and Fresnel behavior, but Arnold's closure source is not available here for a direct numerical source comparison.

Primary references:

- [OpenMoonRay DWA family documentation](https://docs.openmoonray.org/user-reference/scene-objects/materials/dwa/)
- [Physically Based Shading at DreamWorks Animation, Xie and Lanz, 2017](https://blog.selfshadow.com/publications/s2017-shading-course/dreamworks/s2017_pbs_dreamworks_notes.pdf)
- [Autodesk Standard Surface specification](https://autodesk.github.io/standard-surface/)
- [MaterialX PBS specification](https://github.com/AcademySoftwareFoundation/MaterialX/blob/main/documents/Specification/MaterialX.PBRSpec.md)
- [Arnold Standard Surface documentation](https://help.autodesk.com/cloudhelp/ENU/AR-Core/files/ac-shading/ac-surface-shaders/arnold_user_guide_ac_surface_shaders_ac_standard_surface_html.html)
- [Walter et al., Microfacet Models for Refraction through Rough Surfaces, 2007](https://diglib.eg.org/bitstream/handle/10.2312/EGWR.EGSR07.195-206/195-206.pdf?sequence=1)
- [Kulla and Conty, Revisiting Physically Based Shading at Imageworks, 2017](https://blog.selfshadow.com/publications/s2017-shading-course/imageworks/s2017_pbs_imageworks_slides.pdf)

## I. Bug classification

### Confirmed bug

None in the requested native ordinary-dielectric path.

### Likely bug

None established by source trace or numerical tests.

### Intentional model difference

Confirmed: DWA's Kelemen/Kulla-Conty compensation gate and `OneMinusRoughFresnel` underlayer approximation differ from other possible GGX/layering implementations.

### Parameter convention mismatch

Rejected for IOR and isotropic roughness: DwaBase uses absolute IOR and `alpha=roughness^2`, matching the stated conventions of Autodesk Standard Surface for these two controls. A broader material-parameter mismatch remains possible because DwaBase does not have Standard Surface's exact base/specular-color/metalness semantics.

### Inconclusive

The exact source of the reported MoonRay-vs-Karma/Arnold image difference remains unproven until those renderers' isolated closures are sampled numerically under identical incident radiance. Their screenshots alone cannot distinguish closure response from material weights, lighting visibility, or integrator behavior.

One unrelated edge-case code smell should receive a separate test: `BsdfBuilder.cc` constructs the diffuse underlayer attenuator using `component.getTransmissionRoughness()`. This equals reflection roughness in the audited default path, but may differ when independent transmission roughness is enabled even if transmission weight is zero. It cannot explain the reported default case and is not classified as a bug here.

## J. Recommended next action

Do not change production behavior, default IOR, or roughness mapping.

1. Land or review the focused native numerical tests added by this audit.
2. Build a renderer-neutral closure probe that records `f(wo,wi)`, directional albedo, and white-furnace totals from MoonRay, Arnold, and Karma for the same GGX alpha and exact F0, bypassing their artist materials.
3. In the existing comparison scene, verify numerically rather than visually:
   - raw base/diffuse weight and color;
   - specular weight and color;
   - actual alpha after every remap/normal-AA/min-roughness operation;
   - direct and indirect incident radiance visible to the specular lobe;
   - light-linking/specular visibility;
   - front-face state and current medium IOR.
4. If the closure probe shows a MoonRay-only excess, isolate it by toggling only `mFavg` compensation and then substituting exact directional-albedo diffuse attenuation in a test harness. Do not alter production until one term reproduces the measured delta.
5. If a defect is demonstrated, patch only that term and add a legacy-render impact note. Any compensation or underlayer change will affect existing rough dielectric and conductor renders, especially roughness above 0.5.

## Tests added

- `moonray/tests/lib/rendering/shading/TestDielectricFresnel.h`
- `moonray/tests/lib/rendering/shading/TestDielectricFresnel.cc`
- registered in `moonray/tests/lib/rendering/shading/CMakeLists.txt`

The test executable builds successfully and reports `OK (8)`. The new suite directly tests exact normal-incidence Fresnel and the requested roughness/IOR matrix, including compensated versus single-scatter energy, peak, width, and unit-diffuse totals.
