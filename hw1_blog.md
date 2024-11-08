# CENG795 Advanced Ray Tracing - Homework 1
This homework is the first step in my ray tracing journey. On top of the 477 homework, I implemented support for PLY parsing, fresnel and refraction.
<p align=center> <img src="https://github.com/user-attachments/assets/a5eb37be-2b6d-4108-8b68-edf1cace5595" width=50% height=50%> </p>

## Initial miscellaneous steps
1. The output image format was changed from PPM to PNG. I used stb_image_write.h library functions to save PNG's to disk.
2. The 477 homework code was hacky and badly written in some places. I moved around some code and rewrote a few functions.

## Parser problems
The given parser failed to load some of the input scenes. This is caused by a hard-to-diagnose bug in the camera parsing stage. Some scenes have 5 floats denoting the near plane whereas others have 4. I fixed this by modifying the flawed inputs.
```xml
<NearPlane>-1 1 -1 1 20</NearPlane> // broken in spheres_mirror.xml
```

## Part 1: Dielectrics
The hardest part of adding support for dielectric materials was correctly calculating the refracted light. I had to switch from using an iterative approach to a recursive one because I realized that it is impossible to implement refraction without using a stack/recursion.
```cpp
vec4f trace(const Scene &scene, Ray ray, int depth, vec4f energy, bool is_inside, vec4fc absorption_coefficient) // boolean to track refraction state
```
The signature of the main ray tracing function now has 3 additional parameters:
1. energy is multiplied by the appropriate coefficients when the ray bounces and keeps track of the color multiplier.
2. is_inside is set to when a refracted ray enters/exit a dielectric material. It is used to obtain the correct refraction indexes and additionally to disable diffuse/specular shading.
3. absorption_coefficient is set together with is_inside and keeps track of the current medium's well... absorption coefficient.

Now for the math. When a ray collision occurs, the code enters a different case depending on the hit object's material type. In this section I will explain the dielectric code path. The slide formulas are used pretty much as-is. Here is the code code:
```cpp
// here the code is omitted for brevity
// basically:
// use is_inside to do refraction index calculation
// flip normal if it faces the same direction as the incident ray
// use formulas to calculate wt, fr, ft
// run fresnel calculation

Ray reflected_ray(hit_point + norm * scene.ray_epsilon, 2.0f * costheta * norm + ray.direction);
Ray refracted_ray(hit_point - norm * scene.ray_epsilon, wt);

vec4f reflection_color = trace(scene, reflected_ray, depth - 1, energy * fr, !is_inside, absorption_coefficient); // add color from reflection
color += reflection_color;

vec4f refraction_color = trace(scene, refracted_ray, depth - 1, energy * ft, is_inside, is_inside ? material.absorption_coefficient : zero()); // same for refraction
color += refraction_color;
```

(Also we check the term inside the square root, if it's negative only the reflected ray is cast as it's a total internal reflection)

## Part 2: Conductors
The conductor implementation was easier than I expected. It is literally the same as the mirror, the only difference being the energy calculation involves the same Fresnel formula as above:

```cpp
float t1 = (n2 * n2) + (material.absorption_index * material.absorption_index);
float t2 = n2 * costheta;
float t3 = costheta * costheta;

float rs = (t1 - 2*t2 + t3) / (t1 + 2*t2 + t3);
float rp = (t1*t3 - 2*t2 + 1) / (t1*t3 + 2*t2 + 1);

float fr = (rs + rp) / 2; // this is the energy of the reflected ray
```

## Part 3: Attenuation (Beer's Law)
Beer's law simulates the energy loss of the photon as it traverses a material. This is implemented by scaling the energy parameter using a formula with t_min (the distance traveled by the ray before it hits an object) and the material's absorption coefficient:

```cpp
energy = attenuate(energy, absorption_coefficient, t_min); // from the trace function

// and this is the function definition (beer's law)
vec4f attenuate(vec4fc energy, vec4fc coefficient, float distance)
{
    return vec4f{
		energy[0] * exp(-coefficient[0] * distance),
		energy[1] * exp(-coefficient[1] * distance),
		energy[2] * exp(-coefficient[2] * distance),
		0
	};
}
```

## Gallery
<p align=center> <img src="https://github.com/user-attachments/assets/05d1a9c9-bfed-4c06-a1b8-6a43d7c099c7" width=50% height=50%> </p>
<p align=center> <img src="https://github.com/user-attachments/assets/490fc225-142e-4e94-a9dc-15dce9868187" width=50% height=50%> </p>
<p align=center> <img src="https://github.com/user-attachments/assets/0ea3411d-00c6-4afb-b145-319d8dafa2a9" width=50% height=50%> </p>
<p align=center> <img src="https://github.com/user-attachments/assets/d0732aba-3a8c-4b6d-945e-55e5806cf834" width=50% height=50%> </p>
The last image is a bit pixelated because I had to lower the resolution to render it in time.
