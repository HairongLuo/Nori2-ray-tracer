# Nori2-ray-tracer
Project of Computer Graphics at ETH Zürich, 2023 by M. Gross and M. Papas.

Nori2 is an minimalistic ray tracer written in C++.
In the course assignments we are required to implement and validate key rendering and graphics techniques based on Nori2.
The features include a variety of integrators, shapes, textures, warpings, samplings, light sources and BRDFs to go from simple visibility computations to rendering complex scenes with global illumination with indirect lighting.

The final project features the theme: the more you look.
We further implemented from a selection of advanced ray tracer features and eventually render a scene with the theme using our ray tracer. 

The features I implemented are listed below:

| Short Name                      | Features (if required) & Comments                    |
|----------------------------------|------------------------------------------------------|
| Advanced Camera Effects          | depth of field                                        |
| Simple Extra Emitters            | directional light                                     |
| Simple Extra Emitters            | spotlight                                             |                                                 |
| Homogeneous Participating Media  | with path tracing integrator                          |
| Emissive Participating Media     | -                                                    |
| Disney BSDF                      | metallic, specular, roughness, specularTint, clearcoat|

Validation is performed by comparing rendering results with [PBRT](https://pbrt.org) or [Mitsuba](https://www.mitsuba-renderer.org).


## Implementation
### Advanced Camera Effects 
#### Depth of Field

This feature implements a thin lens camera with depth of field. Apart from the normal parameters of a pinhole camera, it also takes a `lensRadius` and a `focalDistance`, which characterize the parameters of the thin len.

In this depth of field implementation, the thin lens camera model is utilized based on two key properties: any ray originating from the same point on the film plane and passing through the thin lens will converge at the same point on the plane of focus, and a ray passing through the center of the lens remains directionally unchanged. First, the corresponding point on the near plane is computed relative to the sampled position and transformed into camera coordinates. A ray is then traced from the center of the lens through this near plane point, and its intersection with the plane of focus determines the final point for the sampled ray. A point is uniformly sampled on the lens, and the ray originates from the sampled point on the film, passes through the lens sample, and intersects the focus plane. Finally, the ray is transformed into world coordinates and returned.

---

### Simple Extra Emitters
#### Directional Light

A directional light simulates distant light sources, such as sunlight, that uniformly illuminate an entire scene. It has a fixed `direction` and remains constant as it travels through space. The light is characterized by two parameters: `radiance`, which defines the intensity, and `direction`, which specifies the light's orientation. Since the light covers the entire scene uniformly and does not attenuate with distance, the probability density function (PDF) is 1.0 everywhere, provided the light is not occluded. The evaluation method consistently returns the radiance. When sampling a directional light, the sampled point is considered to be infinitely far in the direction of the light's source.

#### Spotlight

A spotlight is a delta emitter that emits light in a conical shape from a specific position. The light intensity remains constant within a defined angle relative to its direction but gradually attenuates as the angle increases, following a specified falloff curve. Additionally, the radiance decreases with the square of the distance from the source. The spotlight is characterized by several properties: `position`, `direction`, `intensity`, and two parameters that define the angular falloff, `cosFalloffStart` and `cosFalloffEnd`. When sampling the spotlight, the sampled point is always at its position, and the probability density function (PDF) is consistently 1.0. The evaluation method returns the intensity, scaled by the falloff curve and further attenuated by the inverse square of the distance.

---

### Homogeneous Participating Media (with Path Tracing Integrator)

To begin, a `Medium` class is defined with properties including `sigmaA` (absorption coefficient), `sigmaS` (scattering coefficient), `sigmaT` (extinction coefficient), and a `phase function`. Each medium is associated with a `shape`, and a shape can have both an `interior and exterior medium`. 

The `HomogeneousMedium` class inherits from `Medium`, implementing key methods such as `evalTransmittance`, `sampleDistance`, and `eval`. The transmittance of a ray segment within the medium is evaluated by calculating the path length and applying the transmittance equation to this length. For free-path sampling, the inversion method is used to determine the sampled path length. If the ray intersects a surface before the end of the sampled path, the free-path sampling is considered to have failed. `pdfSuccess` and `pdfFailure` are computed using their respective formulas.

Additionally, `PhaseFunction` class is implemented as an abstract base class, providing pure virtual functions `sample`, `eval`, and `pdf`. The `Isotropic` class inherits from `PhaseFunction`, where both the `eval` and `pdf` methods return \( \frac{1}{4\pi} \). The `sample` method uniformly samples a direction on a sphere in the local frame of outgoing direction `wo`, transforms it to the world coordinate frame, and returns a `PhaseFunctionSample` object containing the `phase value`, the sampled ingoing direction `wi`, and the corresponding `pdf`.

For the boundary of a medium, a `trivial BSDF` (Bidirectional Scattering Distribution Function) class is implemented. This discrete BSDF simply allows the ray to continue propagating through the medium boundary without altering its direction or properties.

Finally, a volumetric path tracing integrator class `VolpathMATS` is developed. The process begins by retrieving the initial medium from the scene and casting the first ray. Inside the main loop, two cases are handled:

1. **Inside a Medium and Free-Path Sampling Success:** If the ray segment lies within a medium and free-path sampling succeeds, the throughput for the segment is accumulated. The next ray direction is sampled using the phase function, and the phase value is accumulated into the throughput.
   
2. **Outside Medium or Free-Path Sampling Failure:** If the ray segment is outside any medium or the free-path sampling fails, throughput for the segment is accumulated if the segment is inside the medium. If the ray does not intersect with any geometry, the path tracing ends. Otherwise, if the ray hits an emitter, the radiance is accumulated. The next ray segment direction is determined by sampling the BSDF, and both the throughput and relative index of refraction (`eta`) are updated accordingly. When the ray intersects a medium boundary, `current medium` is updated for the new ray segment.

To prevent infinite recursion, Russian roulette is applied for probabilistic path termination based on the accumulated throughput.

---

### Emissive Participating Media

The `EmissiveHomogeneousMedium` class is built upon the `HomogeneousMedium`, with an additional radiance field, `Le`. 

In the `EmisvolpathMATS` integrator (material sampling), the primary distinction from the non-emissive version is that the radiance contribution from the medium is accumulated during scattering events within the medium.

In the `EmisvolpathEMS` integrator (emitter sampling), a clear separation is made between direct and indirect illumination. If the first camera ray intersects an emitter, its contribution is considered; otherwise, no radiance is accumulated upon hitting an emitter. Similarly, if the first camera ray terminates inside an emissive medium, the medium's contribution is included, but in other cases, the radiance is not directly accumulated.

This integrator also traces a ray path through phase function and BSDF sampling, with emitter sampling performed at each vertex. In the implementation, both an emitter and an emissive medium are sampled at each ray vertex, and their contributions are combined using importance sampling. For emissive medium sampling, a random point is sampled inside the medium by uniformly sampling within the bounding box and determining whether the point is inside the medium by casting rays from the current path vertex. The total number of surface intersections is counted, and based on whether the current vertex is within the medium, the point's inclusion is determined by checking if the number of intersections is even or odd (assuming no other media are inside the bounding box). This method applies to both convex and concave media.

To compute the PDF of emissive medium sampling, the volume of the medium is estimated via Monte Carlo sampling within the medium's bounding box.

After emitter sampling, the integrator estimates the transmittance from the current vertex to the sampled point by casting successive rays between the two points and multiplying the transmittances of each segment. If one segment is blocked by a non-medium shape, the transmittance is set to zero, equivalent to a shadow ray failure.

Since no emissive participating medium implementation exists in PBRT or Mitsuba, comparisons are made with the regular homogeneous medium for validation.

---

### Disney BSDF

Disney BSDF was implemented following the guidelines provided in a [UCSD assignment manual](https://cseweb.ucsd.edu/~tzli/cse272/wi2023/homework1.pdf). This implementation comprises five components to together form each feature in Disney BSDF, though the sheen component was omitted as it was not required. The remaining four components—`diffuse`, `metallic`, `clearcoat`, and `glass`—were implemented individually as separate BSDF classes, with the Disney class maintaining pointers to each component.

#### Diffuse

The diffuse lobe captures the base diffusive color of the surface and utilizes a modified version of the Schlick Fresnel approximation, which accounts for both `baseColor` and `roughness`. The subsurface scattering lobe was excluded from this implementation, as it was not part of the project requirements. Therefore, the final diffuse reflection, \( f_{\text{diffuse}} \), is equivalent to \( f_{\text{baseDiffuse}} \). Cosine hemisphere sampling is employed to sample the diffuse lobe.

#### Metallic

The metallic lobe models significant specular highlights and incorporates `roughness` and `anisotropic` parameters. It is based on the standard Cook-Torrance microfacet BRDF. The Fresnel term, \( F_m \), is computed using the Schlick approximation, while the normal distribution function, \( D_m \), employs the anisotropic Trowbridge-Reitz (GGX) distribution. Anisotropy is also implemented in this component. The average occlusion factor, \( G_m \), is modeled using the Smith model.

#### Clearcoat

The clearcoat lobe models the heavy tails of specular reflections and uses `clearcoatGloss` as a parameter. The Fresnel term, \( F_c \), is based on a hard-coded index of refraction \( \eta = 1.5 \). The normal distribution function, \( D_c \), assumes isotropic roughness, while the geometric attenuation term, \( G_c \), utilizes a fixed roughness value of 0.25.

#### Glass

The glass lobe models light transmission through surfaces and applies the full Fresnel equation for dielectric materials. The normal distribution function, \( D_g \), and the geometric attenuation term, \( G_g \), are identical to those used in the metallic lobe.

#### Additional

To incorporate dielectric specular reflection into the BSDF, the Fresnel term \( F_m \) is modified to include an achromatic specular component. This adjustment introduces additional parameters—`specular`, `specularTint`, and `metallic`—to the metallic lobe, enhancing its ability to model specular reflection in a more physically accurate manner.

#### Overall

The components of the Disney BSDF are combined using the following formula:

\[
f_{\text{disney}} = (1 - \text{specularTransmission}) \cdot (1 - \text{metallic}) \cdot f_{\text{diffuse}} + (1 - \text{specularTransmission} \cdot (1 - \text{metallic})) \cdot f_{\text{metallic}} + 0.25 \cdot \text{clearcoat} \cdot f_{\text{clearcoat}} + (1 - \text{metallic}) \cdot \text{specularTransmission} \cdot f_{\text{glass}}
\]

This formula blends the diffuse, metallic, clearcoat, and glass components by modulating them with the `specularTransmission`, `metallic`, and `clearcoat` parameters, ensuring a physically based representation of surface reflectance and transmission.