# Documentation Scratchpad

## 2024-10-08

This is place where any documentation can be written, and this documentation
may later be moved or reformatted.

I added this file because I noticed sometimes I don't write documentation
simply because I don't yet know the best place to put the documentation,
which in retrospect seems incredibly silly so instead this file should exist.


### Batch and Symlinks

All filesystem operations will eventually be available through batch requests.
Since batch requests can also handle the cases for single files, it seems silly
to support those endpoints too, so eventually most calls will be done through
`/batch`. Puter's legacy filesystem endpoints will always be supported, but a
future `api.___/fs/v2.0` urlspace for the filesystem API might not include them.

This is batch:

```javascript
await (async () => {
    const endpoint = 'http://api.puter.localhost:4100/batch';

    const ops = [ 
      {
        op: 'mkdir',
        path: '/default_user/Desktop/some-dir',
      },
      {
        op: 'write',
        path: '/default_user/Desktop/some-file.txt',
      }
    ];

    const blob = new Blob(["12345678"], { type: 'text/plain' });
    const formData = new FormData();
    for ( const op of ops ) {
      formData.append('operation', JSON.stringify(op));
    }
    formData.append('fileinfo', JSON.stringify({
        name: 'file.txt',
        size: 8,
        mime: 'text/plain',
    }));
    formData.append('file', blob, 'hello.txt');

    const response = await fetch(endpoint, {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${puter.authToken}` },
        body: formData
    });
    return await response.json();
})();
```
Symlinks are also created via `/batch`

```javascript
await (async () => {
    const endpoint = 'http://api.puter.localhost:4100/batch';

    const ops = [ 
      {
        op: 'symlink',
        path: '~/Desktop',
        name: 'link',
        target: '/bb/Desktop/some'
      },
    ];

    const formData = new FormData();
    for ( const op of ops ) {
      formData.append('operation', JSON.stringify(op));
    }

    const response = await fetch(endpoint, {
        method: 'POST',
        headers: { 'Authorization': `Bearer ${puter.authToken}` },
        body: formData
    });
    return await response.json();
})();
```

## 2025-01-XX: Fix Sidebar Header Text Contrast Bug

### Bug Description
When adjusting the lightness level of screen themes, sidebar header texts in the explorer become unreadable due to poor contrast between text and background colors.

**Current Behavior:**
- Sidebar header text color is hardcoded to `#8f96a3` in `.window-sidebar-title` CSS class
- Sidebar background adapts to theme lightness via CSS variables
- At certain lightness values, the contrast between hardcoded text color and background becomes insufficient
- Text becomes difficult or impossible to read

**Expected Behavior:**
- Sidebar header text should remain readable with adequate contrast across all theme lightness settings
- Should meet WCAG accessibility standards (minimum 4.5:1 contrast ratio for normal text)

### Root Cause Analysis

**Files Involved:**
1. `src/gui/src/css/style.css` (lines 1217-1234)
   - `.window-sidebar-title` has hardcoded `color: #8f96a3;`
   - Sidebar background uses: `hsla(var(--window-sidebar-hue), var(--window-sidebar-saturation), var(--window-sidebar-lightness), calc(0.5 + 0.5*var(--window-sidebar-alpha)))`

2. `src/gui/src/services/ThemeService.js`
   - Sets CSS variables: `--window-sidebar-hue`, `--window-sidebar-saturation`, `--window-sidebar-lightness`, `--window-sidebar-alpha`
   - Sets `--window-sidebar-color` to `var(--primary-color)` which is either white or '#373e44'
   - Does NOT set a variable for sidebar title text color

3. `src/gui/src/css/style.css` (lines 99-103)
   - CSS variables defined: `--window-sidebar-hue`, `--window-sidebar-saturation`, `--window-sidebar-lightness`, `--window-sidebar-alpha`, `--window-sidebar-color`

### Proposed Solution

#### Step 1: Create Contrast Calculation Utility
- **File**: `src/gui/src/services/ThemeService.js` (or create separate utility)
- **Action**: Add helper functions to:
  - Convert HSL to RGB
  - Calculate relative luminance (WCAG formula)
  - Calculate contrast ratio between two colors
  - Determine optimal text color (black or white) based on background color
  - Consider sidebar background's effective color: `calc(0.5 + 0.5*alpha)` means final alpha is `0.5 + 0.5*alpha`

#### Step 2: Calculate Sidebar Title Color Dynamically
- **File**: `src/gui/src/services/ThemeService.js`
- **Location**: In `reload_()` method
- **Action**:
  - Calculate effective sidebar background color (considering alpha blend: `0.5 + 0.5*alpha`)
  - Determine if background is light or dark
  - Calculate appropriate text color that meets WCAG standards
  - Set CSS variable `--window-sidebar-title-color` with calculated color
  - Consider fallback to lighter/darker shades of the theme color if pure black/white doesn't work

#### Step 3: Update CSS to Use Dynamic Color
- **File**: `src/gui/src/css/style.css`
- **Location**: `.window-sidebar-title` rule (line 1221)
- **Action**:
  - Replace hardcoded `color: #8f96a3;` with `color: var(--window-sidebar-title-color, #8f96a3);`
  - Use fallback color in case CSS variable is not set (for backwards compatibility)

#### Step 4: Testing Plan
1. Test with various lightness values (0-100%)
2. Test with different hue and saturation values
3. Verify contrast ratio meets WCAG AA standards (4.5:1) for normal text
4. Test edge cases:
   - Very light backgrounds (lig > 90%)
   - Very dark backgrounds (lig < 10%)
   - Medium backgrounds (lig ~ 50-60%)
5. Test with different alpha values
6. Visual regression testing - ensure text is readable at all settings

#### Step 5: Implementation Details

**Contrast Calculation Algorithm:**
```javascript
// Calculate relative luminance (WCAG)
function getLuminance(rgb) {
  const [r, g, b] = rgb.map(val => {
    val = val / 255;
    return val <= 0.03928 ? val / 12.92 : Math.pow((val + 0.055) / 1.055, 2.4);
  });
  return 0.2126 * r + 0.7152 * g + 0.0722 * b;
}

// Calculate contrast ratio
function getContrastRatio(color1, color2) {
  const l1 = getLuminance(color1);
  const l2 = getLuminance(color2);
  const lighter = Math.max(l1, l2);
  const darker = Math.min(l1, l2);
  return (lighter + 0.05) / (darker + 0.05);
}

// Get optimal text color
function getOptimalTextColor(backgroundColor) {
  // Calculate effective background color (considering alpha blend)
  // Try black and white, choose one with better contrast
  const blackContrast = getContrastRatio([0, 0, 0], backgroundColor);
  const whiteContrast = getContrastRatio([255, 255, 255], backgroundColor);
  
  if (blackContrast >= 4.5 || whiteContrast < 4.5) {
    return '#000000'; // or darker shade if needed
  } else {
    return '#ffffff'; // or lighter shade if needed
  }
}
```

**ThemeService Integration:**
- In `reload_()` method, after setting other CSS variables:
  1. Calculate effective sidebar background RGB
  2. Determine optimal text color
  3. Set `--window-sidebar-title-color` CSS variable

### Files to Modify

1. ✅ `src/gui/src/services/ThemeService.js`
   - Add contrast calculation utilities
   - Calculate and set `--window-sidebar-title-color` in `reload_()`

2. ✅ `src/gui/src/css/style.css`
   - Update `.window-sidebar-title` to use CSS variable with fallback

### Alternative Approaches Considered

1. **CSS-only solution using `mix-blend-mode`**: 
   - Pros: No JS needed
   - Cons: Browser compatibility issues, may affect other elements

2. **CSS `color-contrast()` function**:
   - Pros: Native CSS solution
   - Cons: Limited browser support as of 2024

3. **Predefined color palettes**:
   - Pros: Simple, predictable
   - Cons: May not work for all lightness values, less flexible

### Implementation Priority
- **High**: This is an accessibility issue affecting user experience
- **Impact**: Affects all users who customize theme lightness
- **Risk**: Low - isolated change to theme service and CSS
