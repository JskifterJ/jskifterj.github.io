# Portfolio Website Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and deploy a single-page personal portfolio at `jskifterj.github.io` showcasing Johannes Skifter's background at the intersection of finance, technology, and energy.

**Architecture:** Pure HTML/CSS/JS — no build step, no framework. Single `index.html`, one `style.css`, one `script.js`. Deployed to GitHub Pages from the `main` branch of a repo named `jskifterj.github.io`.

**Tech Stack:** HTML5, CSS3, Vanilla JS, GitHub Pages

---

## File Map

| File | Responsibility |
|------|---------------|
| `index.html` | All page structure and content |
| `style.css` | All visual styles, CSS custom properties, responsive breakpoints |
| `script.js` | Smooth scroll, active nav highlight on scroll |
| `assets/favicon.svg` | SVG lightning-bolt favicon |
| `README.md` | One-line description for GitHub |

---

### Task 1: Repo structure, CSS variables, base HTML shell

**Files:**
- Create: `index.html`
- Create: `style.css`
- Create: `script.js` (empty for now)
- Create: `assets/favicon.svg`
- Create: `README.md`

- [ ] Create `style.css` with CSS variables and reset:

```css
:root {
  --bg: #0d1117;
  --bg-deep: #090c10;
  --bg-card: #161b22;
  --border: #21262d;
  --border-hover: #30363d;
  --text: #e6edf3;
  --text-muted: #8b949e;
  --text-dim: #6e7681;
  --green: #7ee787;
  --blue: #58a6ff;
  --blue-light: #a5d6ff;
  --orange: #f78166;
  --yellow: #d29922;
  --purple: #d2a8ff;
  --mono: 'SF Mono', 'Fira Code', 'Fira Mono', monospace;
  --sans: system-ui, -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --radius: 8px;
  --radius-lg: 12px;
}
*, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
body {
  background: var(--bg-deep);
  color: var(--text);
  font-family: var(--sans);
  line-height: 1.6;
  font-size: 15px;
}
a { color: var(--blue); text-decoration: none; }
a:hover { text-decoration: underline; }
.container { max-width: 900px; margin: 0 auto; padding: 0 24px; }
.mono { font-family: var(--mono); }
.green { color: var(--green); }
.blue { color: var(--blue); }
.blue-light { color: var(--blue-light); }
.orange { color: var(--orange); }
.purple { color: var(--purple); }
.dim { color: var(--text-dim); }
.small { font-size: 11px; }
```

- [ ] Create `index.html` base shell:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta name="description" content="Johannes Skifter — MSc candidate at EPFL/IMD/HEC. Strategy, technology, and energy.">
  <meta property="og:title" content="Johannes Skifter">
  <meta property="og:description" content="At the intersection of business, technology, and energy.">
  <meta property="og:type" content="website">
  <title>Johannes Skifter</title>
  <link rel="icon" type="image/svg+xml" href="assets/favicon.svg">
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <nav id="nav"></nav>
  <main>
    <section id="hero"></section>
    <section id="experience"></section>
    <section id="projects"></section>
    <section id="education"></section>
    <section id="contact"></section>
  </main>
  <script src="script.js"></script>
</body>
</html>
```

- [ ] Create `assets/favicon.svg`:

```svg
<svg xmlns="http://www.w3.org/2000/svg" viewBox="0 0 32 32">
  <rect width="32" height="32" rx="6" fill="#0d1117"/>
  <path d="M19 4l-9 14h7l-4 10 11-15h-7z" fill="#7ee787"/>
</svg>
```

- [ ] Create empty `script.js`:

```javascript
// populated in Task 8
```

- [ ] Create `README.md`:

```markdown
# jskifterj.github.io

Personal portfolio — built with vanilla HTML/CSS/JS, deployed via GitHub Pages.
```

- [ ] Open `index.html` in a browser (double-click or `open index.html`). Verify: dark background renders, no console errors, favicon shows in tab.

- [ ] Commit:

```bash
git add index.html style.css script.js assets/favicon.svg README.md
git commit -m "feat: base HTML shell and CSS variables"
```

---

### Task 2: Nav bar

**Files:**
- Modify: `index.html` — replace `<nav id="nav"></nav>`
- Modify: `style.css` — append nav styles

- [ ] Replace `<nav id="nav"></nav>` in `index.html` with:

```html
<nav id="nav">
  <div class="nav-inner">
    <a href="#hero" class="nav-logo"><span class="mono green">jjs</span> <span class="dim">~</span></a>
    <ul class="nav-links">
      <li><a href="#experience">work</a></li>
      <li><a href="#projects">projects</a></li>
      <li><a href="#education">about</a></li>
      <li><a href="#contact">contact</a></li>
    </ul>
  </div>
</nav>
```

- [ ] Append to `style.css`:

```css
#nav {
  position: sticky;
  top: 0;
  z-index: 100;
  background: rgba(9, 12, 16, 0.92);
  backdrop-filter: blur(8px);
  border-bottom: 1px solid var(--border);
}
.nav-inner {
  max-width: 900px;
  margin: 0 auto;
  padding: 14px 24px;
  display: flex;
  justify-content: space-between;
  align-items: center;
}
.nav-logo { text-decoration: none; font-size: 14px; }
.nav-links { list-style: none; display: flex; gap: 28px; }
.nav-links a { color: var(--text-muted); font-size: 13px; text-decoration: none; transition: color 0.15s; }
.nav-links a:hover, .nav-links a.active { color: var(--text); }
```

- [ ] Verify in browser: dark sticky nav with green `jjs ~` on left, four grey links on right.

- [ ] Commit:

```bash
git add index.html style.css
git commit -m "feat: sticky nav bar"
```

---

### Task 3: Hero section

**Files:**
- Modify: `index.html` — replace `<section id="hero"></section>`
- Modify: `style.css` — append hero styles

- [ ] Replace `<section id="hero"></section>` with:

```html
<section id="hero">
  <div class="container">
    <p class="mono dim small top-label">// nice to meet you</p>
    <h1 class="hero-name">Johannes Skifter</h1>
    <p class="hero-sub">
      MSc candidate · <span class="blue">EPFL / IMD / HEC</span> &nbsp;·&nbsp; Chief of Staff, <span class="blue">Impossible Cloud</span><br>
      <em class="dim">Technically not an engineer — but I've been taking things apart since before that was a career path.</em>
    </p>
    <div class="tag-row">
      <span class="tag">⚡ Energy systems</span>
      <span class="tag">📊 Finance &amp; analytics</span>
      <span class="tag">🐍 Python / ML / SQL</span>
      <span class="tag">🔧 Hardware + software</span>
    </div>
    <div class="metrics-strip">
      <div class="metric"><span class="metric-val green">5.7<small>/6</small></span><span class="metric-label">GPA</span></div>
      <div class="metric"><span class="metric-val blue">40+</span><span class="metric-label">Co's screened</span></div>
      <div class="metric"><span class="metric-val blue-light">3.5M</span><span class="metric-label">DKK revenue</span></div>
      <div class="metric"><span class="metric-val orange">€30k+</span><span class="metric-label">Flipped</span></div>
      <div class="metric"><span class="metric-val purple">age 16</span><span class="metric-label">First biz</span></div>
    </div>
    <div class="venture-log">
      <p class="mono dim small venture-header">git log --oneline --ventures</p>
      <div class="venture-lines">
        <p><span class="mono dim">16 —</span> <strong>Imported stickers from China, sold locally.</strong> <em class="dim">Margin was great. Scale was not.</em></p>
        <p><span class="mono dim">17 —</span> <strong>Mini ad agency: websites + Google Ads.</strong> <em class="dim">Closed it. The clients weren't ready. (Neither was I.)</em></p>
        <p><span class="mono dim">15→</span> <strong>Bought, fixed &amp; resold €30k+ of electronics.</strong> <em class="dim">Still my most reliable income source.</em></p>
        <p><span class="mono dim">20→</span> <strong>Co-founded Orbitvu Nordic.</strong> <em class="dim">DKK 500k → 3.5M. Learned more than any class.</em></p>
      </div>
    </div>
    <div class="cta-row">
      <a href="#projects" class="btn btn-primary">View work →</a>
      <a href="assets/cv.pdf" class="btn btn-ghost" download>Download CV</a>
    </div>
  </div>
</section>
```

- [ ] Append to `style.css`:

```css
#hero { padding: 72px 0 56px; }
.top-label { margin-bottom: 10px; opacity: 0.6; }
.hero-name { font-size: clamp(32px, 5vw, 48px); font-weight: 700; margin-bottom: 12px; }
.hero-sub { color: var(--text-muted); margin-bottom: 24px; line-height: 1.7; }
.hero-sub em { font-size: 13px; }

.tag-row { display: flex; flex-wrap: wrap; gap: 8px; margin-bottom: 28px; }
.tag {
  background: var(--bg-card);
  border: 1px solid var(--border);
  padding: 4px 12px;
  border-radius: 20px;
  font-size: 12px;
  color: var(--text-muted);
}

.metrics-strip {
  display: grid;
  grid-template-columns: repeat(5, 1fr);
  gap: 1px;
  background: var(--border);
  border-radius: var(--radius-lg);
  overflow: hidden;
  margin-bottom: 24px;
}
.metric {
  background: var(--bg-card);
  padding: 14px 10px;
  text-align: center;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.metric-val { font-size: 20px; font-weight: 700; font-family: var(--mono); }
.metric-val small { font-size: 11px; color: var(--text-dim); }
.metric-label { font-size: 9px; text-transform: uppercase; letter-spacing: 1px; color: var(--text-dim); }

.venture-log {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 16px;
  margin-bottom: 24px;
}
.venture-header { margin-bottom: 10px; }
.venture-lines { display: flex; flex-direction: column; gap: 7px; font-size: 13px; }
.venture-lines p { line-height: 1.5; }

.cta-row { display: flex; gap: 12px; flex-wrap: wrap; }
.btn {
  padding: 9px 22px;
  border-radius: 6px;
  font-size: 13px;
  font-weight: 600;
  cursor: pointer;
  text-decoration: none;
  display: inline-block;
  transition: opacity 0.15s;
}
.btn:hover { opacity: 0.85; text-decoration: none; }
.btn-primary { background: #238636; color: #fff; }
.btn-ghost { background: transparent; border: 1px solid var(--border); color: var(--text-muted); }
.btn-sm { padding: 6px 14px; font-size: 12px; }
```

- [ ] Verify in browser: hero shows name, italic tagline, 4 tag pills, 5-cell metrics strip, venture log card, two green/ghost CTAs.

- [ ] Commit:

```bash
git add index.html style.css
git commit -m "feat: hero section — metrics strip and venture log"
```

---

### Task 4: Experience section

**Files:**
- Modify: `index.html` — replace `<section id="experience"></section>`
- Modify: `style.css` — append section + timeline styles

- [ ] Append shared section styles to `style.css`:

```css
section { padding: 56px 0; border-top: 1px solid var(--border); }
.section-title {
  font-size: 11px;
  text-transform: uppercase;
  letter-spacing: 2px;
  color: var(--text-dim);
  margin-bottom: 32px;
  font-family: var(--mono);
}
```

- [ ] Replace `<section id="experience"></section>` with:

```html
<section id="experience">
  <div class="container">
    <h2 class="section-title">Work</h2>
    <div class="timeline">

      <div class="timeline-item">
        <div class="timeline-meta">
          <span class="timeline-date">2026 – present</span>
          <span class="timeline-location">Zug, Switzerland</span>
        </div>
        <div class="timeline-content">
          <div class="timeline-header">
            <h3>Chief of Staff</h3>
            <span class="timeline-company">Impossible Cloud Network</span>
          </div>
          <ul class="timeline-bullets">
            <li>Spearheading gen-AI adoption across the company, including developing my own personal agent (Claude CLI)</li>
            <li>Month-long deep dive into GPU compute infrastructure trajectories — hardware, software, and energy interdependencies</li>
            <li>Developing working knowledge of how power infrastructure constraints shape data centre economics and AI scaling</li>
          </ul>
        </div>
      </div>

      <div class="timeline-item">
        <div class="timeline-meta">
          <span class="timeline-date">2023 – 2024</span>
          <span class="timeline-location">Copenhagen, Denmark</span>
        </div>
        <div class="timeline-content">
          <div class="timeline-header">
            <h3>Investment Banking Analyst</h3>
            <span class="timeline-company">FIH Partners A/S</span>
          </div>
          <ul class="timeline-bullets">
            <li>Supported Associates-to-Partners on market screening, financial modelling, and presentation material</li>
            <li>Screened 40+ technology companies in electrification and industrial automation sectors</li>
            <li>Professional development at Denmark's #1 ranked IB (Prospera, 8th consecutive year)</li>
          </ul>
        </div>
      </div>

      <div class="timeline-item">
        <div class="timeline-meta">
          <span class="timeline-date">2020 – 2022</span>
          <span class="timeline-location">Nivå, Denmark</span>
        </div>
        <div class="timeline-content">
          <div class="timeline-header">
            <h3>Co-Founder &amp; Technical Lead</h3>
            <span class="timeline-company">Orbitvu Nordic</span>
          </div>
          <p class="timeline-desc">Scandinavian distributor of automated product photography studio solutions</p>
          <ul class="timeline-bullets">
            <li>Expanded market presence to SE, NO, FI &amp; IS — web development, Google/LinkedIn Ads</li>
            <li>Grew revenue from DKK 500k to 3.5M profitably, run by three part-time students</li>
          </ul>
        </div>
      </div>

    </div>
  </div>
</section>
```

- [ ] Append timeline styles to `style.css`:

```css
.timeline { display: flex; flex-direction: column; gap: 32px; }
.timeline-item { display: grid; grid-template-columns: 160px 1fr; gap: 24px; }
.timeline-meta { display: flex; flex-direction: column; gap: 4px; padding-top: 3px; }
.timeline-date { font-size: 12px; color: var(--text-dim); font-family: var(--mono); }
.timeline-location { font-size: 11px; color: var(--text-dim); opacity: 0.7; }
.timeline-header { display: flex; align-items: baseline; gap: 10px; margin-bottom: 10px; flex-wrap: wrap; }
.timeline-header h3 { font-size: 16px; font-weight: 600; }
.timeline-company { font-size: 13px; color: var(--blue); }
.timeline-desc { font-size: 13px; color: var(--text-dim); margin-bottom: 8px; font-style: italic; }
.timeline-bullets { list-style: none; display: flex; flex-direction: column; gap: 5px; padding-left: 14px; }
.timeline-bullets li { font-size: 13px; color: var(--text-muted); position: relative; line-height: 1.5; }
.timeline-bullets li::before { content: '—'; position: absolute; left: -18px; color: var(--text-dim); }
```

- [ ] Verify: three timeline rows, date/location flush left, content right.

- [ ] Commit:

```bash
git add index.html style.css
git commit -m "feat: experience timeline"
```

---

### Task 5: Projects section

**Files:**
- Modify: `index.html` — replace `<section id="projects"></section>`
- Modify: `style.css` — append project styles

**Note:** The live dashboard URL (`https://eu-grid.onrender.com`) and the GitHub repo URL (`https://github.com/jskifterj/eu-grid-dashboard`) are placeholders — update them once the dashboard is deployed. The repo name for the portfolio on GitHub Pages must be `jskifterj.github.io`.

- [ ] Replace `<section id="projects"></section>` with:

```html
<section id="projects">
  <div class="container">
    <h2 class="section-title">Projects</h2>

    <div class="project-featured">
      <p class="mono dim small project-featured-label">featured project</p>
      <h3 class="project-title">EU Grid Dashboard <span class="tag tag-live">live ●</span></h3>
      <p class="project-desc">Real-time European electricity grid — generation mix, day-ahead prices, cross-border flows, and CO₂ intensity per country. Includes an AI-generated plain-English grid briefing. Built with Python (FastAPI), ENTSO-E Transparency Platform, D3.js, and Chart.js.</p>
      <div class="project-tags">
        <span class="tag">Python</span><span class="tag">FastAPI</span><span class="tag">D3.js</span><span class="tag">ENTSO-E API</span><span class="tag">Claude API</span>
      </div>
      <div class="project-links">
        <a href="https://eu-grid.onrender.com" class="btn btn-primary btn-sm" target="_blank" rel="noopener">View live →</a>
        <a href="https://github.com/jskifterj/eu-grid-dashboard" class="btn btn-ghost btn-sm" target="_blank" rel="noopener">GitHub</a>
      </div>
    </div>

    <div class="projects-grid">
      <div class="project-card">
        <h4>Recommendation Platform</h4>
        <p>End-to-end data science project: schema design, Python backend, recommendation engine. Built on the Boligportal.dk dataset at TechLabs Copenhagen.</p>
        <div class="project-tags"><span class="tag">Python</span><span class="tag">SQL</span><span class="tag">ML</span></div>
        <a href="https://github.com/jskifterj/Re-valuation-modelling" class="project-link" target="_blank" rel="noopener">View on GitHub →</a>
      </div>
      <div class="project-card">
        <h4>Reddit Post Prediction</h4>
        <p>Predictive modelling on Reddit content — NLP features, classification, and model performance analysis.</p>
        <div class="project-tags"><span class="tag">Python</span><span class="tag">NLP</span><span class="tag">scikit-learn</span></div>
        <a href="https://github.com/jskifterj/BDA-Reddit-post-prediction" class="project-link" target="_blank" rel="noopener">View on GitHub →</a>
      </div>
      <div class="project-card">
        <h4>DK Investment Dashboard</h4>
        <p>Personal dashboard for tracking Danish investment portfolio data and performance metrics.</p>
        <div class="project-tags"><span class="tag">Data</span><span class="tag">Finance</span></div>
        <a href="https://github.com/jskifterj/DK_inv_dashboard" class="project-link" target="_blank" rel="noopener">View on GitHub →</a>
      </div>
    </div>
  </div>
</section>
```

- [ ] Append project styles to `style.css`:

```css
.project-featured {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius-lg);
  padding: 28px;
  margin-bottom: 20px;
}
.project-featured-label { margin-bottom: 10px; }
.project-title { font-size: 20px; font-weight: 700; margin-bottom: 12px; display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.tag-live { background: #0d2016; border-color: var(--green); color: var(--green); font-size: 10px; padding: 3px 10px; }
.project-desc { font-size: 13px; color: var(--text-muted); margin-bottom: 16px; line-height: 1.7; max-width: 640px; }
.project-links { display: flex; gap: 10px; flex-wrap: wrap; }

.projects-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 14px; }
.project-card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 20px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.project-card h4 { font-size: 14px; font-weight: 600; }
.project-card p { font-size: 12px; color: var(--text-muted); line-height: 1.6; flex: 1; }
.project-link { font-size: 12px; color: var(--blue); }
.project-link:hover { text-decoration: underline; }
```

- [ ] Verify: featured card at top with live badge, 3-column grid below.

- [ ] Commit:

```bash
git add index.html style.css
git commit -m "feat: projects section with featured dashboard card"
```

---

### Task 6: Education and skills section

**Files:**
- Modify: `index.html` — replace `<section id="education"></section>`
- Modify: `style.css` — append education/skills styles

- [ ] Replace `<section id="education"></section>` with:

```html
<section id="education">
  <div class="container">
    <h2 class="section-title">Education &amp; Skills</h2>

    <h3 class="section-subtitle">Education</h3>
    <div class="edu-items">
      <div class="edu-item">
        <div class="edu-header">
          <div><h4>MSc Sustainable Management and Technology</h4><p class="edu-school blue">IMD · EPFL · HEC Lausanne</p></div>
          <div class="edu-meta"><span class="mono dim small">2024 – 2026</span><span class="tag">5.7/6 · full scholarship</span></div>
        </div>
        <p class="edu-detail">Energy systems &amp; grid transitions, data science &amp; ML (Python), risk analytics, advanced statistics, optimization modelling.</p>
      </div>
      <div class="edu-item">
        <div class="edu-header">
          <div><h4>Exchange Semester</h4><p class="edu-school blue">The Wharton School, University of Pennsylvania</p></div>
          <div class="edu-meta"><span class="mono dim small">2022</span><span class="tag">3.76/4.0 · full scholarship</span></div>
        </div>
        <p class="edu-detail">M&amp;A, private equity finance, business strategy. 1 of 4 places for 1,500+ students.</p>
      </div>
      <div class="edu-item">
        <div class="edu-header">
          <div><h4>BSc Business Administration &amp; Digital Management</h4><p class="edu-school blue">Copenhagen Business School</p></div>
          <div class="edu-meta"><span class="mono dim small">2020 – 2023</span><span class="tag">10.6/12 · full scholarship</span></div>
        </div>
        <p class="edu-detail">Data science, ML, visualisations, AI ethics, international management. Represented CBS in international case competitions — 1st places at ICC@M and MT Højgaard x Valcon.</p>
      </div>
    </div>

    <h3 class="section-subtitle" style="margin-top:40px">Skills</h3>
    <div class="skills-grid">
      <div class="skill-group">
        <p class="skill-group-label">Languages</p>
        <div class="skill-items">
          <span class="skill-item">English <em>native</em></span>
          <span class="skill-item">Danish <em>native</em></span>
          <span class="skill-item">German <em>intermediate</em></span>
          <span class="skill-item">French <em>intermediate</em></span>
        </div>
      </div>
      <div class="skill-group">
        <p class="skill-group-label">Programming</p>
        <div class="skill-items">
          <span class="skill-item green">Python / R <em>advanced</em></span>
          <span class="skill-item green">HTML / CSS <em>advanced</em></span>
          <span class="skill-item">SQL</span>
          <span class="skill-item">PowerBI</span>
        </div>
      </div>
      <div class="skill-group">
        <p class="skill-group-label">Tools</p>
        <div class="skill-items">
          <span class="skill-item">Excel · PowerPoint · Word</span>
          <span class="skill-item">Git / GitHub</span>
          <span class="skill-item">FastAPI · D3.js · Chart.js</span>
          <span class="skill-item">Claude CLI / Anthropic API</span>
        </div>
      </div>
    </div>

  </div>
</section>
```

- [ ] Append education/skills styles to `style.css`:

```css
.section-subtitle { font-size: 13px; font-weight: 600; color: var(--text-muted); margin-bottom: 20px; }
.edu-items { display: flex; flex-direction: column; gap: 0; }
.edu-item { padding: 20px 0; border-bottom: 1px solid var(--border); }
.edu-item:last-child { border-bottom: none; }
.edu-header { display: flex; justify-content: space-between; align-items: flex-start; gap: 12px; margin-bottom: 6px; flex-wrap: wrap; }
.edu-header h4 { font-size: 14px; font-weight: 600; }
.edu-school { font-size: 13px; margin-top: 3px; }
.edu-meta { display: flex; flex-direction: column; align-items: flex-end; gap: 5px; flex-shrink: 0; }
.edu-detail { font-size: 12px; color: var(--text-dim); line-height: 1.6; }

.skills-grid { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
.skill-group-label { font-size: 10px; text-transform: uppercase; letter-spacing: 1px; color: var(--text-dim); margin-bottom: 10px; }
.skill-items { display: flex; flex-direction: column; gap: 6px; }
.skill-item { font-size: 12px; color: var(--text-muted); }
.skill-item em { font-style: normal; color: var(--text-dim); margin-left: 4px; }
.skill-item.green { color: var(--green); }
```

- [ ] Verify: three education rows with GPA tags right-aligned, three-column skills grid below.

- [ ] Commit:

```bash
git add index.html style.css
git commit -m "feat: education and skills section"
```

---

### Task 7: Contact footer

**Files:**
- Modify: `index.html` — replace `<section id="contact"></section>`
- Modify: `style.css` — append contact styles

- [ ] Replace `<section id="contact"></section>` with:

```html
<section id="contact">
  <div class="container">
    <h2 class="section-title">Contact</h2>
    <p class="contact-intro">Currently based in Lausanne. Always open to interesting conversations about energy, technology, or both.</p>
    <div class="contact-links">
      <a href="https://github.com/jskifterj" class="contact-link" target="_blank" rel="noopener">
        <span class="contact-icon mono">gh</span>github.com/jskifterj
      </a>
      <a href="https://linkedin.com/in/j-johannes-skifter" class="contact-link" target="_blank" rel="noopener">
        <span class="contact-icon">in</span>linkedin.com/in/j-johannes-skifter
      </a>
    </div>
    <p class="footer-note mono dim small" style="margin-top:56px">built by jskifterj · <a href="https://github.com/jskifterj/jskifterj.github.io" class="dim" target="_blank" rel="noopener">source</a></p>
  </div>
</section>
```

- [ ] Append contact styles to `style.css`:

```css
#contact { padding-bottom: 80px; }
.contact-intro { font-size: 15px; color: var(--text-muted); margin-bottom: 28px; max-width: 480px; }
.contact-links { display: flex; flex-direction: column; gap: 14px; }
.contact-link {
  display: flex;
  align-items: center;
  gap: 14px;
  font-size: 14px;
  color: var(--text-muted);
  text-decoration: none;
  width: fit-content;
  transition: color 0.15s;
}
.contact-link:hover { color: var(--text); text-decoration: none; }
.contact-icon {
  width: 32px; height: 32px;
  display: flex; align-items: center; justify-content: center;
  background: var(--bg-card); border: 1px solid var(--border);
  border-radius: 6px; font-size: 13px; color: var(--green); flex-shrink: 0;
}
```

- [ ] Verify: intro text, two contact rows with icon boxes, footer note at bottom.

- [ ] Commit:

```bash
git add index.html style.css
git commit -m "feat: contact section and footer"
```

---

### Task 8: JavaScript — smooth scroll + active nav

**Files:**
- Modify: `script.js` — replace placeholder comment

- [ ] Replace `script.js` content with:

```javascript
// Smooth scroll for all internal anchor links
document.querySelectorAll('a[href^="#"]').forEach(link => {
  link.addEventListener('click', e => {
    const target = document.querySelector(link.getAttribute('href'));
    if (!target) return;
    e.preventDefault();
    target.scrollIntoView({ behavior: 'smooth', block: 'start' });
  });
});

// Active nav link highlight on scroll
const sections = document.querySelectorAll('section[id]');
const navLinks = document.querySelectorAll('.nav-links a');

const observer = new IntersectionObserver(entries => {
  entries.forEach(entry => {
    if (!entry.isIntersecting) return;
    navLinks.forEach(l => l.classList.remove('active'));
    const active = document.querySelector(`.nav-links a[href="#${entry.target.id}"]`);
    if (active) active.classList.add('active');
  });
}, { rootMargin: '-30% 0px -60% 0px' });

sections.forEach(s => observer.observe(s));
```

- [ ] Open `index.html` in browser. Scroll slowly through the full page. Verify:
  - Nav links turn white as you scroll into each section
  - Clicking a nav link scrolls smoothly to the section
  - No console errors

- [ ] Commit:

```bash
git add script.js
git commit -m "feat: smooth scroll and active nav highlight"
```

---

### Task 9: Responsive styles and GitHub Pages deploy

**Files:**
- Modify: `style.css` — append responsive media queries

- [ ] Append responsive breakpoints to `style.css`:

```css
@media (max-width: 720px) {
  .metrics-strip { grid-template-columns: repeat(3, 1fr); }
  .metrics-strip .metric:nth-child(4),
  .metrics-strip .metric:nth-child(5) { display: none; }
  .timeline-item { grid-template-columns: 1fr; gap: 2px; }
  .timeline-meta { flex-direction: row; gap: 12px; }
  .projects-grid { grid-template-columns: 1fr; }
  .skills-grid { grid-template-columns: 1fr 1fr; }
  .edu-header { flex-direction: column; }
  .edu-meta { align-items: flex-start; flex-direction: row; }
  .nav-links { gap: 16px; }
}
@media (max-width: 480px) {
  .skills-grid { grid-template-columns: 1fr; }
  .nav-links a:not(:first-child):not(:last-child) { display: none; }
}
```

- [ ] Commit responsive styles:

```bash
git add style.css
git commit -m "feat: responsive breakpoints"
```

- [ ] On GitHub.com: create a new **public** repository named exactly `jskifterj.github.io`. Leave it empty (no README).

- [ ] Push to GitHub:

```bash
git remote add origin https://github.com/jskifterj/jskifterj.github.io.git
git branch -M main
git push -u origin main
```

- [ ] On GitHub.com → repo Settings → Pages → Source: **Deploy from a branch** → Branch: `main` / `/ (root)` → Save.

- [ ] Wait 2–3 minutes. Open `https://jskifterj.github.io` in browser. Verify the full site loads with correct styles, no 404s for CSS/JS, favicon shows.

- [ ] Final commit if any last-minute tweaks needed:

```bash
git add -p   # stage only what changed
git commit -m "fix: post-deploy tweaks"
git push
```
