# Roll'd Prototype — Session Summary

## Project
Single self-contained HTML/CSS/vanilla-JS mobile ordering prototype for "Roll'd" (Vietnamese restaurant).

- **Canonical source**: `/Users/abacusindonesia/Documents/OU Reimagining/Claude Code/Repo OO Reimagining/prototype.html`
- **Served copy**: `/tmp/rolld-preview/prototype.html` — must `cp` the canonical file here after every edit for the browser preview to pick up changes.
- **Dev server**: custom Python server at `/tmp/rolld-preview/serve.py`, runs on port 8934 (`python3 serve.py` or via `preview_start`). Recreate if lost (e.g. after a `/tmp` reset):

```python
import functools
import http.server
import os
import socketserver

PORT = int(os.environ.get("PORT", 8934))
DIRECTORY = "/tmp/rolld-preview"

class Handler(http.server.SimpleHTTPRequestHandler):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, directory=DIRECTORY, **kwargs)

    def do_GET(self):
        path = self.path.split('?')[0].split('#')[0].rstrip('/')
        last_segment = path.rsplit('/', 1)[-1]
        if last_segment in ('hq', 'branch'):
            self.path = '/prototype.html'
        super().do_GET()

socketserver.TCPServer.allow_reuse_address = True

with socketserver.TCPServer(("127.0.0.1", PORT), Handler) as httpd:
    print(f"serving {DIRECTORY} at http://127.0.0.1:{PORT}")
    httpd.serve_forever()
```

## Standard workflow after any edit
1. Edit `prototype.html` (canonical path above).
2. Python check: `content.count('<div')` vs `content.count('</div>')` must match (div-balance sanity check).
3. `cp` the canonical file to `/tmp/rolld-preview/prototype.html`.
4. Browser `navigate` to the preview URL with an incremented `?t=N` cache-bust query param, `force:true`.
5. Use `javascript_exec` to drive app state (e.g. jump to a specific screen) and `computer{screenshot}` to visually confirm.

## Key architecture notes
- `.screen` class pattern (`position:absolute;inset:0;display:flex;flex-direction:column;animation:screen-fade-in`) shared by all top-level views; `hideAllScreens()` plus per-screen show/hide functions.
- Meal-detail body dynamically swaps between `#mealdetail-body-combo` and `#mealdetail-body-bao` in `openMealDetail(card)`, based on whether `card.querySelector('.meal-sub').textContent.trim() === 'Bao'`.
- **CSS specificity gotcha**: `.mealdetail-hero img{...}` (specificity 0,1,1) can silently beat class rules like `.mealdetail-hero-fg` (0,1,0) that look more specific. Fix pattern used: escalate to an ID selector, e.g. `.mealdetail-hero #mealdetail-hero-fg`.
- **`.btn-add` is a shared/global class** used by `#cart-pay-btn`, the meal-detail Add button, and the Adyen sheet button. Do NOT change it globally for one context — add a scoped override instead (e.g. `.mealdetail-footer .btn-add{flex:1;width:auto;}`) or you'll regress the Pay button's width.
- **Known unresolved rendering bug** (Claude Browser preview tool only): `border-radius` + `overflow:hidden`/`clip`/`clip-path`/`contain:paint` fails to visually clip a flex-item's background-color to a rounded corner, even though hit-testing and border rendering are correct. Multiple workarounds attempted (radial-gradient pseudo-element mask) and abandoned per explicit user instruction — `.mealdetail-popup-card` currently has **plain square corners**, no border-radius. Don't re-attempt border-radius fixes here without discussing with the user first.
- Real-asset sourcing pattern: browse live site via Browser tools → `curl` download → `sips` compress → base64-embed via a small Python script matching a unique anchor string (never print/match raw base64).
- User-provided pasted images: ask the user to save/drop the file into the project folder, then process from disk with `sips`/`base64` (raw bytes of pasted chat images aren't directly extractable).

## Features completed & verified this session
- "Bao" menu category with 7 real products sourced from rolld.com.au (thumbnails + names).
- Dynamic meal-detail hero: background (blurred/darkened) + foreground layered image matching whichever item was clicked, including user-supplied exact images (`Bao.png` / `Bao BG Image.png`) for BBQ Chicken Bao.
- Bao-specific single-item detail body (description, promo banner, Size, "Customise bao" addons, "Select chili", Note field) distinct from the generic combo body.
- Meal-detail popup redesign: 96px-from-top popup, dark 80% overlay behind it, 60px header, footer with qty stepper next to "Add" button showing dynamic total price (base + addons) × qty.
- Font-weight refinements on the Pick-a-time screen (`.time-slot`, `.date-pill`, etc. — see git history / file for exact current values).
- Floating "N items · $total →" cart bar (`.menu-cart-bar-wrap`) shown after dismissing the upsell/suggestion sheet; wired via `closeSuggestionSheet()` to navigate like "Continue to checkout".
- Cart screen guest-checkout consolidation (see "Current/last task" below).

## Current / last task (in progress at time of compaction)
User request: *"Refine the CTA button 'Pay $12.00' label to be 'Pay as a guest $12.00' and move the small label 'No rewards on this order' below the button. This means, the existing CTA 'Continue as a guest' should be taken out."*

Status:
- ✅ `.earn-card` block edited: removed the "or" divider, the `<button class="btn-outline" onclick="openGuestSheet()">Continue as a guest</button>`, and the old `guest-link-note` div. `.earn-card` now only contains the title + "Log in with mobile" button.
- ✅ `.cart-footer` block edited to:
  ```html
  <div class="cart-footer">
    <button class="btn-add" id="cart-pay-btn" onclick="goToCheckout()">
      <span>Pay as a guest</span>
      <span id="cart-pay-price">$12.00</span>
    </button>
    <div class="guest-link-note">No rewards on this order</div>
  </div>
  ```
- ⏳ **Not yet done**: div-balance verification, `cp` sync to `/tmp/rolld-preview/prototype.html`, browser reload + screenshot confirmation that the cart screen shows "Pay as a guest $12.00" with "No rewards on this order" beneath it, and that the old "Continue as a guest" button/divider are gone.

## Known orphaned code (flagged, NOT removed — do not delete without confirming with user)
- `function openGuestSheet(){...}` and `function continueGuestCheckout(){...}` (~line 4045-4056) are now unreferenced since the "Continue as a guest" button was removed.
- `.earn-card-divider` CSS rule (~line 1345) is now unused.

## Notable past corrections from the user (apply going forward)
- "Full bleed edge-to-edge" for a card means **match the content column width** (standard side margins preserved), NOT touching the physical phone screen edges. Don't use negative margins for this.
- If a rendering bug (like the border-radius clipping issue) can't be fixed after reasonable effort, the user prefers dropping the problematic style entirely over persisting with workarounds — ask/confirm fallback only if not already given explicit instruction.
