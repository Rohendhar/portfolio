---
name: gsap-core
description: Comprehensive GreenSock Animation Platform (GSAP 3) rules, patterns, timelines, ScrollTrigger architecture, and performance best practices.
---

# GSAP Core & ScrollTrigger Best Practices

The GreenSock Animation Platform (GSAP 3) is the industry standard for high-performance, robust, and mathematically precise web animations. This skill provides architectural guidelines, code patterns, and anti-patterns for crafting modern GSAP animations.

---

## 1. Core Principles & Modern Syntax (GSAP 3)

### Basic Methods
- `gsap.to(target, vars)`: Animates properties from their current state to specified end values.
- `gsap.from(target, vars)`: Animates properties from specified starting values to their existing CSS state.
- `gsap.fromTo(target, fromVars, toVars)`: Explicitly defines both starting and ending values. Essential when dealing with dynamic scroll states or preventing FOUC.
- `gsap.set(target, vars)`: Instantly sets properties without animation (zero-duration tween).

### Essential Performance Rules
1. **Always use transforms, NEVER layout properties**:
   - Use `x`, `y`, `xPercent`, `yPercent`, `scale`, `rotation`, `skewX`, `skewY`.
   - Never animate `top`, `left`, `margin`, `padding`, `width`, or `height` unless unavoidable (transforms run on the GPU compositor thread; layout triggers CPU reflow).
2. **Use `autoAlpha` over `opacity`**:
   - `autoAlpha: 0` sets `opacity: 0` and `visibility: hidden`. When animating in (`autoAlpha: 1`), it immediately flips `visibility: visible` and fades in `opacity`. This prevents hidden elements from intercepting clicks or screen readers.
3. **Use percentage helpers**:
   - `xPercent: -50`, `yPercent: -50` for centered elements and responsive offsets instead of hardcoded pixel transforms.
4. **Avoid CSS transitions on GSAP properties**:
   - Never apply `transition: all 0.3s` in CSS to elements controlled by GSAP. CSS transitions fight GSAP's `requestAnimationFrame` ticker, causing jitter and dropped frames.

---

## 2. Timelines & Sequencing

Timelines (`gsap.timeline()`) allow orchestrating sequences of animations with precise timing and easy control (`play()`, `pause()`, `reverse()`, `seek()`).

```javascript
const tl = gsap.timeline({
  defaults: {
    duration: 0.8,
    ease: "power3.out"
  },
  onComplete: () => console.log("Sequence complete")
});

tl.from(".hero-title", { y: 40, autoAlpha: 0 })
  .from(".hero-subtitle", { y: 20, autoAlpha: 0 }, "-=0.4")
  .from(".hero-cta", { scale: 0.9, autoAlpha: 0 }, "<0.2")
  .from(".nav-item", { y: -20, autoAlpha: 0, stagger: 0.05 }, 0);
```

### Position Parameter Cheat Sheet
- `undefined` / omitted: Inserts at the end of the timeline (sequential).
- `"+=0.5"`: 0.5 seconds after the end of the timeline.
- `"-=0.5"`: 0.5 seconds before the end of the timeline (overlap).
- `"<"`: Exactly aligned with the start of the previous tween.
- `"<0.2"`: 0.2 seconds after the start of the previous tween.
- `">"`: Exactly aligned with the end of the previous tween.
- `0` or numeric: Absolute timeline timestamp in seconds.
- `"label"`: At a defined timeline label.

---

## 3. ScrollTrigger Architecture

Register the plugin once before creating any triggers:
```javascript
gsap.registerPlugin(ScrollTrigger);
```

### Triggered vs. Scrubbed Animations

#### A. Triggered (Play once or toggle when scrolled into view)
```javascript
gsap.from(".card", {
  scrollTrigger: {
    trigger: ".card",
    start: "top 85%",
    end: "bottom 20%",
    toggleActions: "play none none reverse",
    once: false
  },
  y: 60,
  autoAlpha: 0,
  duration: 1,
  ease: "power2.out"
});
```

#### B. Scrubbed (Directly tied to the scrollbar position)
```javascript
gsap.to(".progress-bar", {
  scrollTrigger: {
    trigger: "#main-container",
    start: "top top",
    end: "bottom bottom",
    scrub: 1
  },
  scaleX: 1,
  transformOrigin: "left center",
  ease: "none"
});
```

#### C. Pinning & Sticky Sections
```javascript
ScrollTrigger.create({
  trigger: ".pinned-section",
  start: "top top",
  end: "+=1000",
  pin: true,
  pinSpacing: true,
  anticipatePin: 1
});
```

### CRITICAL ScrollTrigger Rule: No Nested ScrollTriggers in Timelines
NEVER put a `scrollTrigger` property inside individual tweens of a timeline:
```javascript
// WRONG:
const tl = gsap.timeline();
tl.to(".a", { scrollTrigger: { ... }, x: 100 }); // BAD!

// CORRECT: Attach ScrollTrigger to the parent timeline
const tl = gsap.timeline({
  scrollTrigger: {
    trigger: ".section",
    start: "top center",
    end: "bottom center",
    scrub: true
  }
});
tl.to(".a", { x: 100 })
  .to(".b", { y: 50 });
```

---

## 4. Batch Processing for Performance (Grids / Card Lists)

When animating long lists or repeating elements (cards, list items), use `ScrollTrigger.batch()` instead of attaching individual ScrollTriggers to dozens of DOM nodes:

```javascript
ScrollTrigger.batch(".project-card", {
  start: "top 88%",
  onEnter: batch => gsap.from(batch, {
    autoAlpha: 0,
    y: 50,
    stagger: 0.15,
    duration: 0.8,
    ease: "power2.out",
    overwrite: "auto"
  }),
  once: true
});
```

---

## 5. Responsive Animations with `gsap.matchMedia()`

Cleanly handle different viewports without manual resize listener bloat:

```javascript
const mm = gsap.matchMedia();

mm.add("(min-width: 1024px)", () => {
  gsap.to(".sidebar", {
    scrollTrigger: {
      trigger: ".content",
      pin: ".sidebar",
      start: "top top",
      end: "bottom bottom"
    }
  });
});

mm.add("(max-width: 1023px)", () => {
  gsap.from(".sidebar", { autoAlpha: 0, y: 20 });
});
```

---

## 6. Cleanup & Lifecycle Management (`gsap.context()`)

To avoid memory leaks and ghost triggers:
```javascript
const ctx = gsap.context(() => {
  gsap.from(".heading", { y: 30, autoAlpha: 0 });
  ScrollTrigger.create({ trigger: ".hero", ... });
}, containerElement);

// On unmount:
ctx.revert();
```

---

## 7. Performance & 120 FPS Optimization Checklist

1. **Hardware Acceleration**: Use `force3D: true` or `transform: translateZ(0)` on animated layers.
2. **Text Animation Performance**: Wrap letters/words in `<span>` with `display: inline-block` and `overflow: hidden`.
3. **Avoid Layout Thrashing**: Call `ScrollTrigger.refresh()` after dynamic content/images load.
4. **Eases Reference**:
   - Entrance / Reveal: `"power2.out"`, `"power3.out"`, `"expo.out"`.
   - UI Snappiness: `"back.out(1.7)"` for subtle spring.
   - Continuous / Scroll scrub: `"none"` (linear).
