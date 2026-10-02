# pebbleplot

An interactive plotter for pebble-like shapes: a circle perturbed by a few low-order sine modes in polar coordinates. Drag the sliders and watch the shape, its equation and its shape descriptors update live. You can then export it as an image or copy it as LaTeX, Python, SVG or JSON.

The site is one self-contained `index.html`. It has no framework, build step or external requests.

## The math

### The shape

The boundary is a polar curve $r(\theta)$ for $\theta \in [0, 2\pi)$: a unit circle plus three Fourier modes.

$$
r(\theta) = 1 + w_2\,a\sin(2\theta + 2\pi b) + w_3\,c\sin(3\theta + 2\pi d) + w_4\,e\sin(4\theta + 2\pi f)
$$

| Symbol | Meaning | Range |
| --- | --- | --- |
| $a, c, e$ | amplitude multipliers for modes 2, 3, 4 | $[0, 1]$ |
| $b, d, f$ | phases, as a fraction of a full turn | $[0, 1]$ |
| $w_2, w_3, w_4$ | mode weights, which cap each mode's amplitude | $[0, 2]$, default $0.3, 0.2, 0.1$ |

Mode 2 stretches the circle into an oval, mode 3 makes it rounded-triangular and mode 4 rounded-square. The weights shrink with mode number, so the higher, bumpier modes add only fine detail.

Writing phases as fractions of a turn keeps all six sliders on the same $[0, 1]$ scale. Phase $0$ and phase $1$ give the same shape.

**Validity.** Each sine term is at least $-w_k \cdot \text{amplitude}$, so

$$
r_{\min} \ge 1 - (w_2 a + w_3 c + w_4 e).
$$

With the default weights this bound is at least $0.4$, so the curve is always a simple closed shape. If you raise the weights far enough for $r(\theta)$ to reach $0$, the curve passes through the origin and folds over itself. The page shows a warning when that happens.

### Shape descriptors

The curve is sampled at $N = 1440$ points $(x_i, y_i) = (r_i\cos\theta_i,\ r_i\sin\theta_i)$ and treated as a polygon. All lengths are in units of the base radius.

| Descriptor | How it is computed |
| --- | --- |
| Area $A$ | Shoelace formula: $A = \tfrac12 \left\lvert \sum_i (x_i y_{i+1} - x_{i+1} y_i) \right\rvert$ |
| Perimeter $P$ | Sum of segment lengths |
| Centroid | Polygon centroid, $c_x = \frac{1}{6A}\sum_i (x_i + x_{i+1})(x_i y_{i+1} - x_{i+1} y_i)$, likewise for $c_y$. The page shows its distance from the origin and marks it on the plot. |
| Circularity | $4\pi A / P^2$, which is $1$ for a circle and smaller for anything else |
| Equivalent diameter | $2\sqrt{A/\pi}$, the diameter of the circle with the same area |
| Solidity | $A / A_\text{hull}$. The convex hull comes from Andrew's monotone chain algorithm, so solidity is $1$ for a convex shape. |
| Aspect ratio | $F_\max / F_\min$, the ratio of the largest to smallest Feret (caliper) diameter. Each Feret diameter is the width of the hull projected onto a direction; 180 directions are checked, one degree apart. |
| $r_\min$ / $r_\max$ | Extremes of $r(\theta)$ over the samples |

### Random shapes

**Randomize** draws $a, \dots, f$ uniformly from $[0, 1]$ and each weight $w_k$ uniformly from $[0, 0.4]$. If the weights add up to more than $0.8$, they are scaled down to sum to $0.8$, which keeps $r_\min \ge 0.2$ by the bound above, so a random shape never folds over.

**Suggested shapes.** The gallery shows 16 shapes, freshly drawn on every load or when you click "New set". Each one comes from rejection sampling: make the same random draw as Randomize and keep the shape if

$$
F_\max / F_\min \le 1.3 \quad\text{and}\quad r_\max / r_\min \le 2,
$$

which keeps the suggestions compact and pebble-like. After 400 rejected draws it falls back to the closest candidate. These limits apply only to suggestions; the sliders can make any shape.

### Animation

- **Transitions.** Randomize, Reset and gallery clicks tween every parameter over 320 ms with a cubic ease-out, $1 - (1 - u)^3$. Phases take the shorter way around the circle, so a phase moving from $0.95$ to $0.05$ steps forward $0.1$ instead of sweeping back $0.9$.
- **Animate phases.** This rotates each mode's phase at its own constant rate: $+0.06$, $-0.04$ and $+0.09$ turns per second for modes 2, 3 and 4. Because the rates differ, the shape keeps changing instead of simply spinning.

## The code

Everything lives in [`index.html`](index.html): CSS in a `<style>` block and two inline `<script>` blocks.

**Head script.** This runs before first paint:

- It applies the saved theme and colour, so the page doesn't flash.
- It draws a random pebble favicon from the same $r(\theta)$ formula, as both an SVG and a 64×64 PNG on a canvas.

**Main script.** This is a single IIFE split into commented sections:

| Section | What it does |
| --- | --- |
| state | Parameters and preferences, saved to `localStorage` |
| theme, colour picker | Light/dark toggle that follows the system setting until you choose; ten pastel pebble colours |
| geometry | `r(θ)`, point sampling, convex hull, polygon area and all the shape descriptors |
| shared SVG markup | Polar grid and per-mode overlays, shared by the live view and the exports |
| equation | The colour-coded equation and the live numeric form below it |
| controls | Slider and number inputs for each mode, built from a `MODES` table |
| live drawing | `update()`, which recomputes everything and redraws the SVG on each input |
| export | SVG download (the pebble alone); PNG/JPG download (the full figure with the equation, rendered at 2400 px wide through a canvas) |
| randomness, tweening, animation | Randomize, Reset, tweens and phase animation, using `requestAnimationFrame` |
| suggested shapes | The rejection-sampled gallery |
| copy | Clipboard export as LaTeX, Python/NumPy, SVG or JSON |

**The "glass" pebble.** The pebble is drawn as layered SVG:

- The polar grid is masked out where the pebble sits.
- A blurred copy of the grid is clipped to the pebble's outline, so the grid shows through it slightly distorted.
- A semi-transparent fill, a white diagonal gradient for the sheen and a rim stroke go on top.

**Accessibility.**

- Controls are native inputs with labels.
- Menus close on <kbd>Esc</kbd>.
- Status messages use `aria-live`.
- Animations are skipped when `prefers-reduced-motion` is set.

## Tools

- **HTML, CSS and vanilla JavaScript.** The page has no libraries, frameworks, bundler or package manager.
- **SVG** for the plot, the grid, the glass effect and the vector export.
- **Canvas 2D** for the PNG favicon and the PNG/JPG exports.
- **Web APIs:** `localStorage`, the Clipboard API, `matchMedia` for theme and reduced motion, and `requestAnimationFrame`.
- **Fonts:** system fonts only. Maths text uses STIX Two Text or Cambria Math where installed, with a serif fallback.
- **Static hosting.** [`_headers`](_headers) sets security headers in the Netlify/Cloudflare Pages format, including a strict Content-Security-Policy that allows only inline scripts and styles and `data:`/`blob:` images. [`robots.txt`](robots.txt) and a `noindex` meta tag keep the site out of search engines.

## Running locally

Open `index.html` in a browser. To serve it the way a host would:

```sh
python3 -m http.server
```

Then visit <http://localhost:8000>.

## Credits

Conceptualized by [shyam97](https://shyamhemamalini.com).
