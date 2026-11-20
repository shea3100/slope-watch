# Building the testbed

Week 4. Please write this in `notebooks/03_testbed.ipynb`.

## The problem

Run your tracker on two real Sentinel-2 images, and it reports that the slope moved 1.4 metres. But how do we know it is actually accurate?

There’s no way to know. No one measured Ronti Peak with a tape measure in 2018. Without knowing the true displacement, you can't tell whether your method is accurate or just producing plausible-looking results.

This matters because the project relies on claims like "we can detect movements down to X metres." You can't make that claim without knowing the actual ground truth.

The solution is to create our own test data, where we control the movement and know exactly how much displacement was introduced.

Take a real image, copy it, shift a patch by exactly 0.37 pixels, add noise, and introduce some cloud cover. You now have a realistic before-and-after pair with a known answer. Run your method on it, and you can measure its error precisely.

Everything else, from detection limits and method comparisons to alert thresholds in the app, depends on this testbed. If you build it carelessly, every result downstream could be wrong without anyone noticing.

---

## Step 7: get the base image ready

Read data/chamoli/settings.csv from Notebook 02 to get the base_date and scar box coordinates. Use those values instead of hardcoding them, so you only need to update them in one place if the box changes.

Load the crop and normalize pixel values to a 0–1 range. This makes noise easier to work with and matches the input expected by the neural denoisers in week 5.

You're done when base.min() is close to 0, base.max() is close to 1, and the plot still looks like the original mountain.

---

## Step 8: move a patch

```python
def move_patch(img, dx, dy, box):
    """Shift the region `box` by (dx, dy) pixels. Fractions allowed."""
```

Only the patch moves. Everything else stays exactly the same.

This makes the test more useful. If the whole image shifted, you couldn't tell whether the tracker detected real movement or just noticed the images were misaligned. By moving only one block, you can measure accuracy inside it and false alarms outside it, all from a single image pair.

Use the scar box so the patch has the same texture as the actual failing slope, rather than a random part of the mountain.

Since the shift is smaller than a pixel, you'll need interpolation rather than simply changing array indices. Use scipy.ndimage.shift with order=3.

### The thing that will catch you out

Fractional shifts introduce slight blur. If you shift only the "after" image, it becomes blurrier than the "before" image, giving the tracker another difference to pick up besides actual movement.

A zero shift won't solve this either. Cubic interpolation leaves the original image virtually unchanged, so no smoothing is applied.

Instead, split the displacement between both images: shift "before" by −d/2 and "after" by +d/2. The total movement stays the same, but both images receive equal interpolation.

Test it with a 3-pixel shift and a 0.2-pixel shift. The first should be obvious, while the second should be nearly impossible to see by eye.

---

## Step 9: noise

```python
def add_noise(img, level, rng):
    """Add Gaussian noise. `level` is a fraction of the image's own contrast."""
```

Set noise based on the image's standard deviation instead of fixed values, so the noise level stays consistent across different images and bands.

### The one that bites hardest

The two images need independent noise. If you add the same noise to both, cross-correlation will match the noise pattern instead of the actual movement, giving you misleadingly good results.

This can happen without any errors or warnings. Before moving on, check that the correlation between the two noise arrays is close to zero.

---

## Step 10: clouds

```python
def add_clouds(img, fraction, rng):
    """Blank out `fraction` of pixels as cloud. Returns (image, mask)."""
```

Real clouds create large, connected, irregular holes, not randomly scattered missing pixels. Using random pixels would make the restoration task much easier than it actually is.

To create realistic cloud masks, generate random noise, smooth it to form connected blobs, then apply a quantile threshold to get the desired missing-pixel percentage.

Mark missing pixels as NaN rather than zero, so later steps don't mistake clouds for dark ground. Also return the mask, since week 4's restoration methods will need to know which pixels are missing.

Test with 30% cloud cover and check both the percentage and whether the result looks like real clouds.

---

## Step 11: put it together

```python
def make_test_pair(dx, dy, noise=0.1, cloud=0.0, seed=0):
    """Returns (before, after, mask, true_shift)."""
```

Apply the changes in the order they happen: ground movement, sensor noise, then cloud cover.

Use a single random seed so all three methods in week 6 get exactly the same inputs. Otherwise, differences in results could simply come from random noise rather than the methods themselves.

Test that the same seed produces identical arrays and a different seed produces different ones.
---

## When you're finished

You'll now have one function that generates test images with known displacement, noise, and cloud cover. This will be the basis for evaluating everything else in the project.

Save a few example figures too, including clean, noisy, and cloudy pairs. You'll need them for later a few weeks from now, and it's easier to save them now than recreate them later.

---

## Then: the checks

Don't skip these checks. Each one helps catch a problem that might otherwise go unnoticed.

Zero shift: Create a pair with no displacement, only noise. The measured shift should be close to zero. If there's a consistent offset, check for bias, especially in step 8.

0.5 and 1.0 pixel shifts: Test both and compare the results. A 1-pixel shift needs no interpolation, while 0.5 pixels is the hardest case. If accuracy drops significantly at 0.5, interpolation may be affecting your results.

Also, make sure a 0.5-pixel shift comes back close to 0.5. If not, investigate before moving on.

Noise independence: Check the correlation between the two noise arrays. It should be close to zero. If not, revisit step 9.

These checks help make sure your results are reliable rather than just looking correct.


**Check 1: zero shift**

Build pairs with no displacement at all, only noise, and measure them.

**Check 2: round trip**

Impose a known shift, measure it back, compare.

**Check 3: Noise independence.** Generate two noise arrays and check their correlation. It should be close to zero. If it's close to 1, both images have the same noise, which can make the tracker report misleadingly good results.

**Check 4: Determinism.** Generate the same pair twice with the same seed, then once with a different seed. The first two should be identical, while the third should differ. Use `np.array_equal(a, b, equal_nan=True)` because cloud pixels are `NaN`.
