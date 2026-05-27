# AI-Facebook-Marketplace-Scraper
Scrape FB Marketplace quickly in the browser, no dependencies needed.

## Step 1
Run a search in Facebook Marketplace and run this in your browser console:
```javascript
function scrapeListingPanel() {
  // ── Title: use the span INSIDE h1, not h1 itself ──────────────────────
  // The h1 text is "Search results"; the real title is in its child span
  const titleSpan = document.querySelector('h1 span[dir="auto"]');
  if (!titleSpan) return null;

  // Find panel: smallest ancestor of titleSpan containing 2+ h2 sections
  let panel = titleSpan.parentElement;
  while (panel && panel !== document.body) {
    if (panel.querySelectorAll('h2').length >= 2) break;
    panel = panel.parentElement;
  }
  if (!panel || panel === document.body) panel = document.body;

  const fullText = panel.innerText;

  function getSection(startMarker, ...endMarkers) {
    const si = fullText.indexOf(startMarker);
    if (si === -1) return '';
    const from = si + startMarker.length;
    let to = fullText.length;
    for (const m of endMarkers) {
      const i = fullText.indexOf(m, from);
      if (i !== -1 && i < to) to = i;
    }
    return fullText.slice(from, to).trim();
  }

  const JUNK = new Set(['See less', 'See more', 'NHTSA ratings', 'Save', 'Share',
                        'Message', 'Send', 'About this vehicle', "Seller's description",
                        'Seller information', 'Send seller a message', '·', '']);
  function cleanLines(text) {
    return text.split('\n').map(s => s.trim()).filter(s => s.length > 1 && !JUNK.has(s));
  }

  const vehicleSection = getSection('About this vehicle',   "Seller's description", 'Seller information');
  const descSection    = getSection("Seller's description", 'Seller information',   'Location is approximate');

  // ── Description: strip trailing location line ("City, ST ·") ──────────
  const descLines = cleanLines(descSection).filter(
    line => !/^[A-Za-z\s]+,\s*[A-Z]{2}\s*·?\s*$/.test(line)
  );

  // ── Location: strip "See more" prefix if present ───────────────────────
  const rawLocation = (fullText.match(/([A-Za-z\s]+,\s*[A-Z]{2})\s*·\s*Location is approximate/) || [])[1] || '';

  return {
    title:           titleSpan.innerText.trim(),
    price:           (fullText.match(/^\$[\d,]+/m)               || [])[0]?.trim() || '',
    listed:          (fullText.match(/Listed .+? ago in [^\n]+/)  || [])[0]?.trim() || '',
    location:        rawLocation.trim(),
    vehicle_details: cleanLines(vehicleSection),
    description:     descLines.join('\n'),
    url:             '' // filled in by the caller
  };
}

// ── Main runner (unchanged except calling updated scrapeListingPanel) ───
async function scrapeAllListings(scrolls = 20, scrollDelay = 1500) {
  console.log('📜 Scrolling to load all listings...');
  for (let i = 0; i < scrolls; i++) {
    window.scrollTo(0, document.body.scrollHeight);
    await new Promise(r => setTimeout(r, scrollDelay));
    console.log(`Scroll ${i + 1}/${scrolls}`);
  }

  const seen = new Set();
  const anchors = [];
  document.querySelectorAll('a[href*="/marketplace/item/"]').forEach(a => {
    const clean = `https://www.facebook.com${new URL(a.href).pathname}`;
    if (!seen.has(clean)) { seen.add(clean); anchors.push({ url: clean, el: a }); }
  });
  console.log(`\n✅ Found ${anchors.length} listings. Starting detail scrape...\n`);

  const results = [];

  for (let i = 0; i < anchors.length; i++) {
    const { url } = anchors[i];
    anchors[i].el.scrollIntoView({ behavior: 'smooth', block: 'center' });
    await new Promise(r => setTimeout(r, 600));
    anchors[i].el.click();
    await new Promise(r => setTimeout(r, 2800));

    const data = scrapeListingPanel();
    if (data) {
      data.url = url;
      results.push(data);
      console.log(`  ✓ "${data.title}" | ${data.price} | ${data.location}`);
    } else {
      console.warn(`  ⚠ Could not parse panel for ${url}`);
      results.push({ title: '', url, error: 'panel not found' });
    }

    history.back();
    await new Promise(r => setTimeout(r, 2800));

    // Refresh stale element references
    document.querySelectorAll('a[href*="/marketplace/item/"]').forEach(a => {
      const c = `https://www.facebook.com${new URL(a.href).pathname}`;
      const match = anchors.find(x => x.url === c);
      if (match) match.el = a;
    });
  }

  console.log('\n🎉 Done! Final JSON:\n');
  const json = JSON.stringify(results, null, 2);
  console.log(json);
  copy(json);
  console.log('\n📋 Copied to clipboard!');
  return results;
}

scrapeAllListings(10, 600);
```

## Step 2:
Copy the output and paste it into your AI chatbot of choice and ask it to give the best result with its url.
