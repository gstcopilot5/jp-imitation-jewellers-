import React, { useState, useEffect, useRef, useCallback } from "react";

/* ============================================================
   J.P. IMITATION JEWELLERS — Luxury 3D Interactive Website
   Single-file React (Vite-ready). Drop into src/App.jsx.

   TO ADD YOUR HERO VIDEO:
   1. Put your video file in:  public/hero.mp4
   2. (Optional) a still frame in: public/hero-poster.jpg
   The hero already references "/hero.mp4". If the video is
   missing, an animated burgundy-silk fallback shows instead.

   TO ADD REAL PHOTOS LATER:
   Replace the gradient "media" blocks (search PLACEHOLDER) with
   <img> tags pointing at your WhatsApp catalog images.
   ============================================================ */

const GOLD = "#D4AF37";
const BG = "#090909";
const PHONE = "+919033385620";
const PHONE_DISPLAY = "+91 90333 85620";
const WA = (msg) =>
  `https://wa.me/919033385620?text=${encodeURIComponent(
    msg || "Hi J.P. Imitation Jewellers, I'd love to see your latest collection."
  )}`;
const ADDRESS = "Divanpara Main Road, Near Vishwakarma Temple, Rajkot, Gujarat 360001";
const MAP_EMBED =
  "https://www.google.com/maps?q=J.P.+Imitation+Jewellers+Divanpara+Main+Road+Rajkot+Gujarat+360001&output=embed";
const MAP_LINK =
  "https://www.google.com/maps/search/?api=1&query=J.P.+Imitation+Jewellers+Divanpara+Main+Road+Rajkot";

/* ============================================================
   OFFER + ADMIN — Supabase-ready (no SDK; uses REST + Auth via fetch)
   To go live: create the table (SQL is shown inside /#admin), then
   paste your Project URL + anon key below. Until then the site runs
   in safe "demo" mode and shows the default offer.
   ============================================================ */
const SUPABASE_URL = "";        // e.g. "https://xxxx.supabase.co"
const SUPABASE_ANON_KEY = "";   // your project's anon public key
const sbReady = () => !!(SUPABASE_URL && SUPABASE_ANON_KEY);

const DEFAULT_OFFER = {
  title: "Bridal Season Invitation",
  highlight: "Up to 20% Off",
  subtitle:
    "On selected bridal & American-diamond sets this season. Show this code in store or on WhatsApp to claim.",
  code: "JP-BRIDAL",
  cta: "Claim On WhatsApp",
  active: true,
};

async function sbGetOffer(activeOnly = true) {
  if (!sbReady()) return null;
  const q = activeOnly ? "&active=eq.true" : "";
  const r = await fetch(
    `${SUPABASE_URL}/rest/v1/offers?select=*${q}&order=updated_at.desc&limit=1`,
    { headers: { apikey: SUPABASE_ANON_KEY, Authorization: `Bearer ${SUPABASE_ANON_KEY}` } }
  );
  if (!r.ok) return null;
  const rows = await r.json();
  return rows[0] || null;
}
async function sbLogin(email, password) {
  const r = await fetch(`${SUPABASE_URL}/auth/v1/token?grant_type=password`, {
    method: "POST",
    headers: { apikey: SUPABASE_ANON_KEY, "Content-Type": "application/json" },
    body: JSON.stringify({ email, password }),
  });
  if (!r.ok) throw new Error("auth");
  return r.json();
}
async function sbSaveOffer(token, offer) {
  const headers = {
    apikey: SUPABASE_ANON_KEY,
    Authorization: `Bearer ${token}`,
    "Content-Type": "application/json",
    Prefer: "return=representation",
  };
  const body = JSON.stringify({ ...offer, updated_at: new Date().toISOString() });
  const url = offer.id
    ? `${SUPABASE_URL}/rest/v1/offers?id=eq.${offer.id}`
    : `${SUPABASE_URL}/rest/v1/offers`;
  const r = await fetch(url, { method: offer.id ? "PATCH" : "POST", headers, body });
  if (!r.ok) throw new Error("save");
  return (await r.json())[0];
}

const SETUP_SQL = `create table if not exists offers (
  id uuid primary key default gen_random_uuid(),
  title text, highlight text, subtitle text,
  code text, cta text,
  active boolean default true,
  updated_at timestamptz default now()
);
alter table offers enable row level security;
create policy "read active" on offers
  for select to anon using (active = true);
create policy "admin write" on offers
  for all to authenticated using (true) with check (true);`;

/* ---------- tiny inline sparkle / diamond mark ---------- */
const Diamond = ({ size = 26, c = GOLD }) => (
  <svg width={size} height={size} viewBox="0 0 24 24" fill="none" aria-hidden="true">
    <path d="M12 2L7 8l5 14 5-14-5-6z" stroke={c} strokeWidth="1" strokeLinejoin="round" />
    <path d="M3.5 8.5h17M12 2L9.5 8.5M12 2l2.5 6.5" stroke={c} strokeWidth=".7" opacity=".7" />
  </svg>
);

/* ---------- WhatsApp glyph for CTAs ---------- */
const WaGlyph = ({ s = 16, c = "currentColor" }) => (
  <svg width={s} height={s} viewBox="0 0 24 24" fill={c} aria-hidden="true">
    <path d="M12 2a10 10 0 00-8.6 15l-1.3 4.7 4.8-1.3A10 10 0 1012 2zm0 2a8 8 0 11-4.2 14.8l-.3-.2-2.8.8.8-2.7-.2-.3A8 8 0 0112 4zm4.3 9.6c-.2-.1-1.4-.7-1.6-.8-.2-.1-.4-.1-.5.1l-.7.9c-.1.1-.3.2-.5.1a6.5 6.5 0 01-1.9-1.2 7 7 0 01-1.3-1.6c-.1-.2 0-.4.1-.5l.4-.4.2-.4v-.4l-.7-1.7c-.2-.4-.4-.4-.5-.4h-.5a1 1 0 00-.7.3A2.8 2.8 0 006 8.7c0 1.6 1.2 3.2 1.4 3.4.1.2 2.3 3.6 5.6 4.9 2.7 1.1 3 .9 3.5.8.6-.1 1.4-.6 1.6-1.1.2-.6.2-1 .1-1.1l-.4-.2z" />
  </svg>
);

/* ---------- sparkles for hero title ---------- */
const SparkStar = () => (
  <svg viewBox="0 0 24 24" aria-hidden="true">
    <path d="M12 0C12.7 6.5 17.5 11.3 24 12 17.5 12.7 12.7 17.5 12 24 11.3 17.5 6.5 12.7 0 12 6.5 11.3 11.3 6.5 12 0Z" fill="currentColor" />
  </svg>
);
const SPARKS = [
  { l: "4%", t: "20%", s: 16, d: "0s", u: "2.6s" },
  { l: "20%", t: "64%", s: 10, d: ".7s", u: "3.1s" },
  { l: "33%", t: "6%", s: 18, d: "1.2s", u: "2.4s" },
  { l: "48%", t: "72%", s: 9, d: ".3s", u: "2.9s" },
  { l: "60%", t: "16%", s: 13, d: "1.7s", u: "3.3s" },
  { l: "74%", t: "60%", s: 15, d: ".9s", u: "2.5s" },
  { l: "88%", t: "22%", s: 11, d: "2.1s", u: "2.8s" },
  { l: "14%", t: "42%", s: 8, d: "1.4s", u: "3.2s" },
  { l: "80%", t: "82%", s: 12, d: ".2s", u: "2.7s" },
];

/* ---------- scroll reveal hook ---------- */
function useReveal() {
  const ref = useRef(null);
  useEffect(() => {
    const el = ref.current;
    if (!el) return;
    const io = new IntersectionObserver(
      (entries) => {
        entries.forEach((e) => {
          if (e.isIntersecting) {
            e.target.classList.add("in");
            io.unobserve(e.target);
          }
        });
      },
      { threshold: 0.18 }
    );
    el.querySelectorAll(".reveal").forEach((n) => io.observe(n));
    return () => io.disconnect();
  }, []);
  return ref;
}

/* ---------- 3D tilt wrapper ---------- */
function Tilt({ children, className = "", max = 9, style }) {
  const ref = useRef(null);
  const canTilt = useRef(false);
  useEffect(() => {
    canTilt.current = window.matchMedia("(hover: hover) and (pointer: fine)").matches;
  }, []);
  const move = useCallback(
    (e) => {
      if (!canTilt.current) return;
      const el = ref.current;
      const r = el.getBoundingClientRect();
      const px = (e.clientX - r.left) / r.width - 0.5;
      const py = (e.clientY - r.top) / r.height - 0.5;
      el.style.transform = `perspective(900px) rotateY(${px * max}deg) rotateX(${-py * max}deg) translateY(-6px)`;
      el.style.setProperty("--gx", `${(px + 0.5) * 100}%`);
      el.style.setProperty("--gy", `${(py + 0.5) * 100}%`);
    },
    [max]
  );
  const leave = useCallback(() => {
    const el = ref.current;
    if (el) el.style.transform = "";
  }, []);
  return (
    <div ref={ref} className={`tilt ${className}`} style={style} onMouseMove={move} onMouseLeave={leave}>
      {children}
    </div>
  );
}

/* ---------- animated heading: masked word rise + gold glint ---------- */
function Heading({ children, className = "" }) {
  if (typeof children === "string") {
    const words = children.split(" ");
    return (
      <h2 className={`h2 split ${className}`}>
        {words.map((w, i) => (
          <React.Fragment key={i}>
            <span className="wm">
              <span className="w" style={{ transitionDelay: `${i * 55}ms` }}>{w}</span>
            </span>{" "}
          </React.Fragment>
        ))}
        <span className="glint" aria-hidden="true" />
      </h2>
    );
  }
  return (
    <h2 className={`h2 glintwrap ${className}`}>
      {children}
      <span className="glint" aria-hidden="true" />
    </h2>
  );
}

/* ============================ NAV ============================ */
function Nav({ route }) {
  const [solid, setSolid] = useState(false);
  const [open, setOpen] = useState(false);
  useEffect(() => {
    const onScroll = () => setSolid(window.scrollY > 60);
    window.addEventListener("scroll", onScroll, { passive: true });
    return () => window.removeEventListener("scroll", onScroll);
  }, []);
  const links = [
    ["Home", "#/", "home"],
    ["Collections", "#/collections", "collections"],
    ["About", "#/about", "about"],
    ["Visit", "#/visit", "visit"],
  ];
  return (
    <header className={`nav ${solid ? "nav-solid" : ""}`}>
      <div className="nav-inner">
        <a href="#/" className="brand">
          <Diamond size={22} />
          <span>
            J.P. <em>Imitation Jewellers</em>
          </span>
        </a>
        <nav className={`nav-links ${open ? "open" : ""}`}>
          {links.map(([t, h, key]) => (
            <a key={t} href={h} className={route === key ? "active" : ""} onClick={() => setOpen(false)}>
              {t}
            </a>
          ))}
          <a className="nav-cta" href={WA()} target="_blank" rel="noreferrer" onClick={() => setOpen(false)}>
            <WaGlyph s={14} /> WhatsApp
          </a>
        </nav>
        <button className="burger" aria-label="Menu" onClick={() => setOpen((o) => !o)}>
          <span /><span /><span />
        </button>
      </div>
    </header>
  );
}

/* ============================ HERO ============================ */
function Hero() {
  const layer = useRef(null);
  const [videoOk, setVideoOk] = useState(true);

  useEffect(() => {
    const el = layer.current;
    if (!el) return;
    const isFine = window.matchMedia("(hover: hover) and (pointer: fine)").matches;
    if (!isFine) return;
    const onMove = (e) => {
      const x = e.clientX / window.innerWidth - 0.5;
      const y = e.clientY / window.innerHeight - 0.5;
      el.style.setProperty("--mx", x.toFixed(3));
      el.style.setProperty("--my", y.toFixed(3));
    };
    window.addEventListener("mousemove", onMove);
    return () => window.removeEventListener("mousemove", onMove);
  }, []);

  const motes = Array.from({ length: 16 });

  return (
    <section className="hero" id="top" ref={layer}>
      <div className="hero-silk" aria-hidden="true" />
      {videoOk && (
        <video
          className="hero-video"
          autoPlay
          muted
          loop
          playsInline
          poster="/hero-poster.jpg"
          onError={() => setVideoOk(false)}
        >
          <source src="/hero.mp4" type="video/mp4" />
        </video>
      )}
      <div className="hero-grade" aria-hidden="true" />
      <div className="hero-vignette" aria-hidden="true" />

      <div className="particles" aria-hidden="true">
        {motes.map((_, i) => (
          <span
            key={i}
            className="mote"
            style={{
              left: `${(i * 67) % 100}%`,
              top: `${(i * 41) % 100}%`,
              animationDelay: `${(i % 8) * 0.9}s`,
              animationDuration: `${7 + (i % 5) * 1.6}s`,
              transform: `scale(${0.5 + (i % 4) * 0.35})`,
            }}
          />
        ))}
      </div>

      <div className="hero-content">
        <p className="hero-kicker reveal in">
          <span className="rule" /> RAJKOT · CRAFTED TO BE REMEMBERED <span className="rule" />
        </p>
        <div className="title-wrap">
          <h1 className="hero-title">
            <span className="hline">
              {["J.P.", "Imitation"].map((w, i) => (
                <React.Fragment key={i}>
                  <span className="wm">
                    <span className="w hw" style={{ animationDelay: `${0.15 + i * 0.09}s` }}>{w}</span>
                  </span>{" "}
                </React.Fragment>
              ))}
            </span>
            <span className="hline l2">
              <span className="wm">
                <span className="w hw" style={{ animationDelay: "0.42s" }}>Jewellers</span>
              </span>
            </span>
          </h1>
          <span className="sparkles" aria-hidden="true">
            {SPARKS.map((s, i) => (
              <i
                key={i}
                className="spark"
                style={{ left: s.l, top: s.t, width: s.s, height: s.s, animationDelay: s.d, animationDuration: s.u }}
              >
                <SparkStar />
              </i>
            ))}
          </span>
        </div>
        <p className="hero-sub">
          Every piece chosen for its light and its finish — bridal sets to everyday
          shine, trusted across Rajkot for generations.
        </p>
        <div className="hero-ctas">
          <a className="btn btn-gold" href="#/collections">
            Explore Collection
          </a>
          <a className="btn btn-ghost" href={WA()} target="_blank" rel="noreferrer">
            <WaGlyph s={16} /> Chat on WhatsApp
          </a>
        </div>
        <div className="hero-badge">
          <strong>4.8</strong>
          <span className="stars-sm">★★★★★</span>
          <span>33+ Google reviews</span>
        </div>
      </div>

      <a href="#trust" className="scroll-cue" aria-label="Scroll down">
        <span />
      </a>
    </section>
  );
}

/* ============================ TRUST ============================ */
function Trust() {
  const ref = useReveal();
  const items = [
    ["★", "4.8 Rating", "Verified on Google"],
    ["✦", "33+ Reviews", "Loved by customers"],
    ["📍", "Rajkot, Gujarat", "Divanpara Main Road"],
    ["🏪", "Physical Showroom", "Visit & try in person"],
    ["💎", "Premium Collections", "Bridal to daily wear"],
    ["✓", "Customer-First", "Friendly service"],
  ];
  return (
    <section className="section trust" id="trust" ref={ref}>
      <div className="trust-grid">
        {items.map(([ic, t, s], i) => (
          <Tilt key={t} className="reveal glass trust-card" style={{ transitionDelay: `${i * 60}ms` }}>
            <span className="trust-ic">{ic}</span>
            <h3>{t}</h3>
            <p>{s}</p>
          </Tilt>
        ))}
      </div>
    </section>
  );
}

/* ========================= COLLECTIONS ========================= */
function Collections() {
  const ref = useReveal();
  const cats = [
    ["Bridal Collection", "Showstopping wedding sets", "#2a0010", "#8a0f28"],
    ["American Diamond", "Brilliant AD sparkle", "#0a1426", "#2c5aa0"],
    ["Necklace Sets", "Statement neckpieces", "#1c0009", "#7a0c1f"],
    ["Pendant Sets", "Everyday elegance", "#07150e", "#1f7a4d"],
    ["Earrings", "From studs to jhumkas", "#26000d", "#9a1030"],
    ["Bangles", "Kadas & bangle sets", "#140d05", "#5c3a14"],
    ["Fashion Jewellery", "Trend-led pieces", "#180009", "#5c0018"],
  ];
  return (
    <section className="section" id="collections" ref={ref}>
      <div className="section-head reveal">
        <span className="eyebrow">The Collections</span>
        <Heading>Curated To Be Worn &amp; Remembered</Heading>
        <p className="lead">
          Seven signature lines — each chosen for craft, finish and the way it catches the light.
        </p>
      </div>
      <div className="col-grid">
        {cats.map(([name, sub, c1, c2], i) => (
          <Tilt
            key={name}
            className={`reveal col-card ${i === 0 ? "feature" : ""}`}
            max={7}
            style={{ transitionDelay: `${(i % 4) * 70}ms` }}
          >
            {/* PLACEHOLDER media — swap with <img> later */}
            <div className="col-media" style={{ background: `radial-gradient(120% 120% at 30% 20%, ${c2}, ${c1} 70%)` }}>
              <div className="col-shine" />
              <Diamond size={i === 0 ? 40 : 30} c="rgba(212,175,55,.85)" />
            </div>
            <div className="col-overlay">
              <h3>{name}</h3>
              <p>{sub}</p>
              <a href={WA(`Hi, please share your latest ${name}.`)} target="_blank" rel="noreferrer" className="col-link">
                Enquire →
              </a>
            </div>
          </Tilt>
        ))}
      </div>
    </section>
  );
}

/* ========================= EDITORIAL ========================= */
function Editorial() {
  const ref = useReveal();
  const rows = [
    {
      tag: "Crafted to shine",
      title: "Every design chosen to elevate the moment",
      body:
        "From wedding celebrations to everyday fashion, each piece is selected to combine elegance, beauty and affordability — luxury that feels effortless to wear.",
      g: "radial-gradient(110% 110% at 70% 28%, #8a0f28, #1c0009 74%)",
    },
    {
      tag: "Bridal specialists",
      title: "Made for the day everyone remembers",
      body:
        "Our bridal sets are built around presence — layered necklaces, matching earrings and finishing details that hold their shine under every light.",
      g: "radial-gradient(110% 110% at 30% 28%, #6a4a14, #100b04 74%)",
      reverse: true,
    },
  ];
  return (
    <section className="section editorial" id="editorial" ref={ref}>
      {rows.map((r) => (
        <div key={r.title} className={`ed-row reveal ${r.reverse ? "rev" : ""}`}>
          {/* PLACEHOLDER media */}
          <Tilt className="ed-media" max={6} style={{ background: r.g }}>
            <div className="ed-shine" />
            <Diamond size={48} c="rgba(212,175,55,.8)" />
          </Tilt>
          <div className="ed-body">
            <span className="eyebrow">{r.tag}</span>
            <Heading>{r.title}</Heading>
            <p className="lead">{r.body}</p>
            <a href={WA()} target="_blank" rel="noreferrer" className="btn btn-ghost sm">
              See it on WhatsApp
            </a>
          </div>
        </div>
      ))}
    </section>
  );
}

/* ========================= WHY CHOOSE US ========================= */
function Why() {
  const ref = useReveal();
  const items = [
    ["Premium Craftsmanship", "Finishing you can feel in the hand."],
    ["Latest Trending Designs", "New arrivals, the season's looks."],
    ["Extensive Collection", "Hundreds of sets across every category."],
    ["Affordable Luxury", "The look of fine jewellery, fair prices."],
    ["Trusted Store", "Years of goodwill on Divanpara Road."],
    ["Bridal Specialists", "Complete wedding sets, start to finish."],
  ];
  return (
    <section className="section why" ref={ref}>
      <div className="section-head reveal">
        <span className="eyebrow">Why Choose Us</span>
        <Heading>Built On Trust, Worn With Pride</Heading>
      </div>
      <div className="why-grid">
        {items.map(([t, s], i) => (
          <Tilt key={t} className="reveal glass why-card" max={8} style={{ transitionDelay: `${(i % 3) * 70}ms` }}>
            <span className="why-num">{String(i + 1).padStart(2, "0")}</span>
            <h3>{t}</h3>
            <p>{s}</p>
          </Tilt>
        ))}
      </div>
    </section>
  );
}

/* ========================= GALLERY ========================= */
function Gallery() {
  const ref = useReveal();
  // varied heights → masonry feel. PLACEHOLDER tiles.
  const tiles = [
    ["Bridal Set", 360, "#5a0b16", "#2a0010"],
    ["Earrings", 240, "#3a2a10", "#161208"],
    ["Necklace", 300, "#6a0b1e", "#22000d"],
    ["AD Pendant", 280, "#2c5aa0", "#0a1426"],
    ["Bangles", 320, "#4a2f12", "#1c1208"],
    ["Fashion", 230, "#4A0012", "#120008"],
    ["Choker", 290, "#1f7a4d", "#07150e"],
    ["Jhumkas", 260, "#5a4012", "#161208"],
  ];
  return (
    <section className="section gallery" id="gallery" ref={ref}>
      <div className="section-head reveal">
        <span className="eyebrow">The Gallery</span>
        <Heading>A Closer Look</Heading>
      </div>
      <div className="masonry reveal">
        {tiles.map(([name, h, c1, c2], i) => (
          <Tilt
            key={name + i}
            className="m-item"
            max={10}
            style={{ height: h, background: `radial-gradient(120% 120% at 40% 25%, ${c1}, ${c2} 75%)` }}
          >
            <div className="m-shine" />
            <span className="m-label">{name}</span>
          </Tilt>
        ))}
      </div>
    </section>
  );
}

/* ========================= REVIEWS ========================= */
function Reviews() {
  const ref = useReveal();
  const revs = [
    ["Awesome collection", "Verified Google review"],
    ["Awesome imitation style & a huge collection", "Verified Google review"],
    ["Great collection — lovely staff", "Verified Google review"],
  ];
  return (
    <section className="section reviews" id="reviews" ref={ref}>
      <div className="section-head reveal">
        <span className="eyebrow">Reviews</span>
        <Heading>
          4.8<span className="of"> / 5</span> · 33+ Voices
        </Heading>
        <p className="lead">What customers say after they visit the showroom.</p>
      </div>
      <div className="rev-grid">
        {revs.map(([q, who], i) => (
          <Tilt key={i} className="reveal glass rev-card" max={7} style={{ transitionDelay: `${i * 80}ms` }}>
            <span className="stars">★★★★★</span>
            <p className="rev-q">“{q}”</p>
            <span className="rev-who">{who}</span>
          </Tilt>
        ))}
      </div>
      <div className="reveal rev-cta">
        <a className="btn btn-ghost sm" href={MAP_LINK} target="_blank" rel="noreferrer">
          Read all on Google →
        </a>
      </div>
    </section>
  );
}

/* ========================= ABOUT ========================= */
function About() {
  const ref = useReveal();
  return (
    <section className="section about" ref={ref}>
      <div className="about-wrap reveal">
        <span className="eyebrow">Our Story</span>
        <Heading>A Trusted Jewellery Destination In Rajkot</Heading>
        <p className="lead">
          On Divanpara Main Road, J.P. Imitation Jewellers offers a carefully curated collection of
          premium imitation jewellery — bridal sets, necklaces, earrings, bangles and American diamond
          lines.
        </p>
        <div className="about-pillars">
          {["Beautiful designs", "Excellent quality", "Affordable luxury"].map((p) => (
            <div key={p} className="pillar">
              <Diamond size={20} />
              <span>{p}</span>
            </div>
          ))}
        </div>
      </div>
    </section>
  );
}

/* ========================= STORE ========================= */
function Store() {
  const ref = useReveal();
  return (
    <section className="section store" id="store" ref={ref}>
      <div className="store-grid">
        <div className="map-wrap reveal">
          <iframe title="J.P. Imitation Jewellers location" src={MAP_EMBED} loading="lazy" />
        </div>
        <div className="store-info reveal">
          <span className="eyebrow">Visit The Showroom</span>
          <Heading>Come See Them Shine</Heading>
          <ul className="store-list">
            <li><span>Address</span>{ADDRESS}</li>
            <li><span>Phone</span>{PHONE_DISPLAY}</li>
            <li><span>Hours</span>Open daily · 10:00 AM – 9:00 PM</li>
          </ul>
          <div className="store-actions">
            <a className="btn btn-gold sm" href={MAP_LINK} target="_blank" rel="noreferrer">Get Directions</a>
            <a className="btn btn-ghost sm" href={`tel:${PHONE}`}>Call Now</a>
            <a className="btn btn-ghost sm" href={WA()} target="_blank" rel="noreferrer"><WaGlyph s={15} /> WhatsApp</a>
          </div>
        </div>
      </div>
    </section>
  );
}

/* ========================= WHATSAPP CTA ========================= */
function CTA() {
  const ref = useReveal();
  return (
    <section className="wacta" ref={ref}>
      <div className="wacta-silk" aria-hidden="true" />
      <div className="wacta-inner reveal">
        <Diamond size={34} />
        <Heading>Looking For The Perfect Jewellery Design?</Heading>
        <p className="lead">Chat directly with us and discover our latest collections — replies are quick.</p>
        <a className="btn btn-gold lg" href={WA()} target="_blank" rel="noreferrer">
          <WaGlyph s={18} c="#1a0a0a" /> Chat On WhatsApp
        </a>
      </div>
    </section>
  );
}

/* ========================= FOOTER ========================= */
function Footer() {
  return (
    <footer className="footer">
      <div className="foot-grid">
        <div className="foot-col">
          <a href="#/" className="brand">
            <Diamond size={20} />
            <span>J.P. <em>Imitation Jewellers</em></span>
          </a>
          <p className="foot-note">Premium imitation jewellery · Rajkot, Gujarat.</p>
        </div>
        <div className="foot-col">
          <h4>Visit</h4>
          <p>{ADDRESS}</p>
          <a href={MAP_LINK} target="_blank" rel="noreferrer">Open in Maps →</a>
        </div>
        <div className="foot-col">
          <h4>Contact</h4>
          <a href={`tel:${PHONE}`}>{PHONE_DISPLAY}</a>
          <a href={WA()} target="_blank" rel="noreferrer">WhatsApp us</a>
        </div>
        <div className="foot-col">
          <h4>Hours</h4>
          <p>Mon – Sun</p>
          <p>10:00 AM – 9:00 PM</p>
        </div>
      </div>
      <div className="foot-bar">
        <span>© {new Date().getFullYear()} J.P. Imitation Jewellers. All rights reserved.</span>
        <span className="foot-credit">Crafted by Shringaar Digital Studio</span>
      </div>
    </footer>
  );
}

/* ---------- interactive zigzag divider (gold dot follows cursor) ---------- */
function ZigZag() {
  const W = 1200, S = 100, TOP = 16, BOT = 52;
  const zy = (x) => {
    const idx = Math.floor(x / S), within = (x - idx * S) / S;
    return idx % 2 === 0 ? BOT + (TOP - BOT) * within : TOP + (BOT - TOP) * within;
  };
  const pts = [];
  for (let x = 0; x <= W; x += S) pts.push(`${x},${zy(x)}`);
  const d = "M" + pts.join(" L");
  const [dot, setDot] = useState({ x: W / 2, y: zy(W / 2) });
  const move = (e) => {
    const r = e.currentTarget.getBoundingClientRect();
    const x = Math.max(0, Math.min(W, ((e.clientX - r.left) / r.width) * W));
    setDot({ x, y: zy(x) });
  };
  return (
    <div className="zz-wrap reveal">
      <svg className="zz" viewBox="0 0 1200 68" preserveAspectRatio="none" onMouseMove={move}>
        <path className="zline" d={d} />
        <circle className="zdot" cx={dot.x} cy={dot.y} r="5" />
      </svg>
    </div>
  );
}

const ZTeeth = ({ flip }) => (
  <svg
    className={`zteeth ${flip ? "bot" : "top"}`}
    viewBox="0 0 120 10"
    preserveAspectRatio="none"
    style={{ transform: flip ? "scaleY(-1)" : "none" }}
    aria-hidden="true"
  >
    <path d="M0 10L10 0L20 10L30 0L40 10L50 0L60 10L70 0L80 10L90 0L100 10L110 0L120 10Z" fill="var(--gold)" />
  </svg>
);

/* ---------- public offer (reads Supabase, falls back to default) ---------- */
function Offer() {
  const ref = useReveal();
  const [offer, setOffer] = useState(DEFAULT_OFFER);
  useEffect(() => {
    let on = true;
    (async () => {
      try { const o = await sbGetOffer(true); if (on && o) setOffer(o); } catch (e) {}
    })();
    return () => { on = false; };
  }, []);
  if (offer && offer.active === false) return null;
  return (
    <section className="section offer" id="offer" ref={ref}>
      <ZigZag />
      <div className="section-head reveal" style={{ marginBottom: "1.2rem" }}>
        <span className="eyebrow">A Special Invitation</span>
      </div>
      <Tilt className="reveal" max={6}>
        <div className="offer-ticket">
          <ZTeeth />
          <div className="seal" aria-hidden="true">
            <svg viewBox="0 0 100 100" className="seal-ring">
              <circle cx="50" cy="50" r="42" fill="none" stroke="rgba(212,175,55,.7)" strokeWidth="2" strokeDasharray="4 6" />
            </svg>
            <span className="seal-txt">SAVE</span>
          </div>
          <p className="offer-kicker">Limited Time</p>
          <p className="offer-highlight">{offer.highlight}</p>
          <h3 className="offer-title">{offer.title}</h3>
          <p className="offer-sub">{offer.subtitle}</p>
          <span className="offer-code">CODE · {offer.code}</span>
          <div>
            <a
              className="btn btn-gold"
              href={WA(`Hi, I'd like to claim the offer ${offer.code} — ${offer.highlight}.`)}
              target="_blank"
              rel="noreferrer"
            >
              <WaGlyph s={16} c="#1a0a0a" /> {offer.cta || "Claim On WhatsApp"}
            </a>
          </div>
          <ZTeeth flip />
        </div>
      </Tilt>
    </section>
  );
}

/* ---------- hash route + admin panel ---------- */
function useHashRoute() {
  const [hash, setHash] = useState(typeof window !== "undefined" ? window.location.hash : "");
  useEffect(() => {
    const on = () => setHash(window.location.hash);
    window.addEventListener("hashchange", on);
    return () => window.removeEventListener("hashchange", on);
  }, []);
  return hash;
}

function Admin() {
  const configured = sbReady();
  const [token, setToken] = useState(null);
  const [email, setEmail] = useState("");
  const [pw, setPw] = useState("");
  const [offer, setOffer] = useState(DEFAULT_OFFER);
  const [busy, setBusy] = useState(false);
  const [msg, setMsg] = useState("");

  useEffect(() => {
    if (token && token !== "demo") {
      (async () => { try { const o = await sbGetOffer(false); if (o) setOffer(o); } catch (e) {} })();
    }
  }, [token]);

  const set = (k) => (e) =>
    setOffer((o) => ({ ...o, [k]: e.target.type === "checkbox" ? e.target.checked : e.target.value }));

  const login = async () => {
    setBusy(true); setMsg("");
    try {
      if (!configured) { setToken("demo"); setMsg("Demo mode — add your Supabase keys to go live."); }
      else { const dt = await sbLogin(email, pw); setToken(dt.access_token); setMsg("Signed in."); }
    } catch { setMsg("Login failed — check email & password."); }
    finally { setBusy(false); }
  };
  const save = async () => {
    setBusy(true); setMsg("");
    try {
      if (!configured || token === "demo") setMsg("Saved locally (demo). Connect Supabase to make it live.");
      else { const s = await sbSaveOffer(token, offer); setOffer(s); setMsg("Saved — it's live on the site."); }
    } catch { setMsg("Save failed — check the table and your permissions."); }
    finally { setBusy(false); }
  };

  return (
    <div className="admin">
      <div className="admin-card">
        <a href="#" className="brand" style={{ marginBottom: "1.4rem" }}>
          <Diamond size={20} /> <span>J.P. <em>Admin</em></span>
        </a>

        {!token ? (
          <>
            <h3 className="admin-h">Sign in to manage your offer</h3>
            {!configured && (
              <p className="note">Supabase isn't connected yet — you can still try the editor in demo mode.</p>
            )}
            <div className="field">
              <label>Email</label>
              <input className="inp" type="email" value={email} onChange={(e) => setEmail(e.target.value)} placeholder="you@store.com" />
            </div>
            <div className="field">
              <label>Password</label>
              <input className="inp" type="password" value={pw} onChange={(e) => setPw(e.target.value)} placeholder="••••••••" />
            </div>
            <button className="btn btn-gold" style={{ width: "100%" }} onClick={login} disabled={busy}>
              {busy ? "Signing in…" : "Sign In"}
            </button>
          </>
        ) : (
          <>
            <h3 className="admin-h">Edit the live offer</h3>
            <div className="field"><label>Highlight (big text)</label>
              <input className="inp" value={offer.highlight || ""} onChange={set("highlight")} /></div>
            <div className="field"><label>Title</label>
              <input className="inp" value={offer.title || ""} onChange={set("title")} /></div>
            <div className="field"><label>Subtitle</label>
              <textarea className="inp" rows={3} value={offer.subtitle || ""} onChange={set("subtitle")} /></div>
            <div className="row">
              <div className="field" style={{ flex: 1 }}><label>Code</label>
                <input className="inp" value={offer.code || ""} onChange={set("code")} /></div>
              <div className="field" style={{ flex: 1 }}><label>Button text</label>
                <input className="inp" value={offer.cta || ""} onChange={set("cta")} /></div>
            </div>
            <label className="check">
              <input type="checkbox" checked={!!offer.active} onChange={set("active")} /> Show this offer on the site
            </label>
            <div className="row" style={{ marginTop: "1.2rem" }}>
              <button className="btn btn-gold" style={{ flex: 1 }} onClick={save} disabled={busy}>
                {busy ? "Saving…" : "Save"}
              </button>
              <a className="btn btn-ghost" href="#" style={{ flex: 1 }}>View Site</a>
            </div>
          </>
        )}

        {msg && <p className="note ok">{msg}</p>}

        <details className="setup">
          <summary>Supabase setup (one-time)</summary>
          <ol>
            <li>Supabase → SQL editor → run the script below.</li>
            <li>Authentication → Users → add yourself as an admin user.</li>
            <li>Paste your Project URL + anon key at the top of App.jsx.</li>
          </ol>
          <pre>{SETUP_SQL}</pre>
        </details>
      </div>
    </div>
  );
}

/* ========================= APP ========================= */
/* ---------- routing ---------- */
function parseRoute(hash) {
  const p = (hash || "").replace(/^#\/?/, "").toLowerCase().split(/[?#]/)[0];
  if (p.startsWith("admin")) return "admin";
  if (p.startsWith("collections")) return "collections";
  if (p.startsWith("about")) return "about";
  if (p.startsWith("visit")) return "visit";
  return "home";
}
function useRoute() {
  const [route, setRoute] = useState(() =>
    typeof window !== "undefined" ? parseRoute(window.location.hash) : "home"
  );
  useEffect(() => {
    const on = () => setRoute(parseRoute(window.location.hash));
    window.addEventListener("hashchange", on);
    return () => window.removeEventListener("hashchange", on);
  }, []);
  return route;
}

/* ---------- inner-page header ---------- */
function PageHeader({ eyebrow, title, sub }) {
  const ref = useReveal();
  return (
    <header className="page-head" ref={ref}>
      <div className="reveal">
        <span className="eyebrow">{eyebrow}</span>
        <Heading>{title}</Heading>
        {sub && <p className="lead ph-sub">{sub}</p>}
      </div>
    </header>
  );
}

function FabWhatsApp() {
  return (
    <a className="fab" href={WA()} target="_blank" rel="noreferrer" aria-label="WhatsApp">
      <svg width="26" height="26" viewBox="0 0 24 24" fill="#090909" aria-hidden="true">
        <path d="M12 2a10 10 0 00-8.6 15l-1.3 4.7 4.8-1.3A10 10 0 1012 2zm0 2a8 8 0 11-4.2 14.8l-.3-.2-2.8.8.8-2.7-.2-.3A8 8 0 0112 4zm4.3 9.6c-.2-.1-1.4-.7-1.6-.8-.2-.1-.4-.1-.5.1l-.7.9c-.1.1-.3.2-.5.1a6.5 6.5 0 01-1.9-1.2 7 7 0 01-1.3-1.6c-.1-.2 0-.4.1-.5l.4-.4.2-.4v-.4l-.7-1.7c-.2-.4-.4-.4-.5-.4h-.5a1 1 0 00-.7.3A2.8 2.8 0 006 8.7c0 1.6 1.2 3.2 1.4 3.4.1.2 2.3 3.6 5.6 4.9 2.7 1.1 3 .9 3.5.8.6-.1 1.4-.6 1.6-1.1.2-.6.2-1 .1-1.1l-.4-.2z" />
      </svg>
    </a>
  );
}

/* ---------- page composition ---------- */
function PageView({ route }) {
  switch (route) {
    case "collections":
      return (
        <>
          <Collections />
          <Editorial />
          <Gallery />
        </>
      );
    case "about":
      return (
        <>
          <About />
          <Why />
          <Reviews />
        </>
      );
    case "visit":
      return (
        <>
          <PageHeader
            eyebrow="Visit Us"
            title="Come See Them Shine"
            sub="Find us on Divanpara Main Road, near Vishwakarma Temple, Rajkot."
          />
          <Store />
          <CTA />
        </>
      );
    default:
      return (
        <>
          <Hero />
          <Trust />
          <Offer />
          <CTA />
        </>
      );
  }
}

export default function App() {
  const target = useRoute();
  const [shown, setShown] = useState(target);
  const [phase, setPhase] = useState("in");

  useEffect(() => {
    if (target === shown) return;
    setPhase("out");
    const t = setTimeout(() => {
      setShown(target);
      setPhase("in");
      window.scrollTo(0, 0);
    }, 340);
    return () => clearTimeout(t);
  }, [target, shown]);

  const isAdmin = shown === "admin";

  return (
    <div className="jp-root">
      <style>{CSS}</style>
      {!isAdmin && <Nav route={shown} />}
      <main className={`page ${phase}`} key={shown}>
        {isAdmin ? <Admin /> : <PageView route={shown} />}
      </main>
      {!isAdmin && <Footer />}
      {!isAdmin && <FabWhatsApp />}
      <span className={`route-bar ${phase}`} aria-hidden="true" />
    </div>
  );
}

/* ============================ CSS ============================ */
const CSS = `
@import url('https://fonts.googleapis.com/css2?family=Bodoni+Moda:ital,opsz,wght@0,6..96,400;0,6..96,500;0,6..96,600;0,6..96,700;0,6..96,800;1,6..96,400&family=Inter:wght@300;400;500;600&display=swap');

*{box-sizing:border-box}
.jp-root{
  --gold:${GOLD}; --gold-lt:#f3dd96; --bg:#070406; --burg:#5c0018; --wine:#7a0c1f; --ruby:#9a1030;
  --white:#fff; --muted:#D8D8D8;
  background:
   radial-gradient(80% 50% at 50% -5%,rgba(122,12,31,.22),transparent 60%),
   radial-gradient(70% 60% at 100% 30%,rgba(92,0,24,.18),transparent 65%),
   radial-gradient(55% 45% at 0% 75%,rgba(15,92,58,.10),transparent 60%),
   radial-gradient(50% 40% at 90% 92%,rgba(20,52,128,.10),transparent 60%),
   var(--bg);
  background-attachment:fixed; color:var(--white);
  font-family:'Inter',system-ui,sans-serif; line-height:1.6;
  overflow-x:hidden;
}
.jp-root a{color:inherit;text-decoration:none}
img,iframe,video{display:block;max-width:100%}
h1,h2,h3,h4{font-family:'Bodoni Moda',Georgia,serif;font-weight:700;line-height:1.08;letter-spacing:.2px;margin:0}
.h2{font-size:clamp(1.9rem,4.2vw,3.2rem)}
.lead{color:var(--muted);font-size:clamp(1rem,1.5vw,1.15rem);font-weight:300;max-width:60ch}
.eyebrow{display:inline-block;font-size:.72rem;letter-spacing:.42em;text-transform:uppercase;color:var(--gold);margin-bottom:1rem}

/* reveal */
/* arrival: focus-pull (blur → sharp) + settle */
.reveal{opacity:0;filter:blur(13px);transform:translateY(26px) scale(.985);
  transition:opacity .9s cubic-bezier(.16,.84,.28,1),filter 1s cubic-bezier(.16,.84,.28,1),transform 1s cubic-bezier(.16,.84,.28,1)}
.reveal.in{opacity:1;filter:blur(0);transform:none}

/* headings: masked word-by-word rise */
.split,.glintwrap{position:relative;overflow:hidden;padding-bottom:.1em}
.wm{display:inline-block;overflow:hidden;vertical-align:bottom;line-height:1.05}
.w{display:inline-block;transform:translateY(118%);transition:transform .85s cubic-bezier(.16,.84,.28,1)}
.reveal.in .w{transform:translateY(0)}

/* gold light sweep across headline on arrival */
.glint{position:absolute;top:0;left:-35%;width:35%;height:100%;pointer-events:none;
  background:linear-gradient(105deg,transparent,rgba(246,231,173,.5),transparent);
  transform:skewX(-18deg);opacity:0}
.reveal.in .glint{animation:glint 1.25s ease .55s 1}
@keyframes glint{0%{left:-35%;opacity:0}28%{opacity:1}100%{left:135%;opacity:0}}

@media (prefers-reduced-motion:reduce){
  .reveal,.reveal.in,.w{opacity:1!important;filter:none!important;transform:none!important;transition:none}
  .glint{display:none}
  .spark{animation:none;opacity:.55}
  .hline.l2 .w{animation:none}
}

/* tilt */
.tilt{transition:transform .35s cubic-bezier(.2,.7,.2,1);transform-style:preserve-3d;will-change:transform}

/* buttons */
.btn{display:inline-flex;align-items:center;justify-content:center;gap:.55rem;
  padding:1rem 2rem;border-radius:999px;font-size:.76rem;letter-spacing:.16em;text-transform:uppercase;
  font-weight:600;cursor:pointer;border:1px solid transparent;transition:.3s}
.btn.sm{padding:.7rem 1.4rem;font-size:.85rem}
.btn.lg{padding:1.1rem 2.6rem;font-size:1rem}
.btn-gold{background:linear-gradient(135deg,#fbeec0,#f0d98a 35%,var(--gold) 75%,#b8902a);color:#1a0a0a;
  box-shadow:0 10px 34px rgba(212,175,55,.38)}
.btn-gold:hover{transform:translateY(-2px);box-shadow:0 16px 44px rgba(212,175,55,.42)}
.btn-ghost{border-color:rgba(212,175,55,.5);color:#f3e6c4;background:rgba(212,175,55,.05);backdrop-filter:blur(6px)}
.btn-ghost:hover{border-color:var(--gold);background:rgba(212,175,55,.12)}

/* glass */
.glass{background:linear-gradient(160deg,rgba(255,255,255,.06),rgba(255,255,255,.02));
  border:1px solid rgba(212,175,55,.22);border-radius:18px;backdrop-filter:blur(10px);
  position:relative;overflow:hidden}
.glass::after{content:"";position:absolute;inset:0;border-radius:inherit;pointer-events:none;
  background:radial-gradient(220px 160px at var(--gx,50%) var(--gy,0%),rgba(212,175,55,.14),transparent 60%);
  opacity:0;transition:opacity .4s}
.glass:hover::after{opacity:1}

/* ===== NAV ===== */
.nav{position:fixed;top:0;left:0;right:0;z-index:50;transition:.4s;
  padding:1.1rem clamp(1rem,4vw,3rem)}
.nav-solid{background:rgba(9,9,9,.82);backdrop-filter:blur(14px);
  border-bottom:1px solid rgba(212,175,55,.14);padding:.7rem clamp(1rem,4vw,3rem)}
.nav-inner{max-width:1280px;margin:0 auto;display:flex;align-items:center;justify-content:space-between}
.brand{display:flex;align-items:center;gap:.55rem;font-family:'Bodoni Moda',serif;font-size:1.12rem;font-weight:600}
.brand em{font-style:italic;color:var(--gold);font-size:.78rem;font-weight:400;letter-spacing:.06em}
.nav-links{display:flex;align-items:center;gap:2rem}
.nav-links a{font-size:.88rem;letter-spacing:.04em;color:var(--muted);transition:.25s}
.nav-links a:hover{color:var(--gold)}
.nav-cta{display:inline-flex;align-items:center;gap:.4rem;padding:.5rem 1.2rem;border:1px solid rgba(212,175,55,.5);border-radius:999px;color:#f3e6c4!important}
.nav-cta:hover{background:rgba(212,175,55,.12)}
.burger{display:none;background:none;border:0;flex-direction:column;gap:5px;cursor:pointer;padding:6px}
.burger span{width:24px;height:2px;background:var(--gold);display:block}

/* ===== HERO ===== */
.hero{position:relative;min-height:100vh;display:flex;align-items:center;justify-content:center;
  text-align:center;overflow:hidden;isolation:isolate}
.hero-video{position:absolute;inset:0;width:100%;height:100%;object-fit:cover;z-index:-3;
  transform:translateZ(0) scale(1.04)}
.hero-silk{position:absolute;inset:-10%;z-index:-4;
  background:
   radial-gradient(55% 75% at calc(50% + var(--mx,0)*40px) 28%,#9a1030 0%,transparent 52%),
   radial-gradient(65% 85% at 82% 72%,#3a0012 0%,transparent 58%),
   conic-gradient(from 205deg at 50% 50%,#0a0004,#5c0018,#120006,#7a0c1f,#070406);
  filter:saturate(1.25) contrast(1.08);animation:silk 22s ease-in-out infinite alternate}
@keyframes silk{0%{transform:scale(1.05) rotate(0deg)}100%{transform:scale(1.18) rotate(4deg)}}
.hero-grade{position:absolute;inset:0;z-index:-2;
  background:linear-gradient(180deg,rgba(7,4,6,.62),rgba(7,4,6,.28) 38%,rgba(7,4,6,.94))}
.hero-vignette{position:absolute;inset:0;z-index:-1;
  background:radial-gradient(110% 85% at 50% 38%,transparent 32%,rgba(7,4,6,.9))}

.particles{position:absolute;inset:0;z-index:0;pointer-events:none}
.mote{position:absolute;width:5px;height:5px;border-radius:50%;
  background:radial-gradient(circle,#f6e7ad,rgba(212,175,55,.1));
  box-shadow:0 0 10px 2px rgba(212,175,55,.35);animation:float linear infinite;opacity:.7}
@keyframes float{0%{transform:translateY(20px) scale(1);opacity:0}
  20%{opacity:.8}100%{transform:translateY(-90px) scale(1.1);opacity:0}}

.hero-content{position:relative;z-index:2;padding:0 1.4rem;max-width:880px;
  transform:translate3d(calc(var(--mx,0)*-22px),calc(var(--my,0)*-16px),0)}
.hero-kicker{display:flex;align-items:center;justify-content:center;gap:1rem;
  font-size:.72rem;letter-spacing:.4em;color:var(--gold);text-transform:uppercase}
.hero-kicker .rule{height:1px;width:46px;background:linear-gradient(90deg,transparent,var(--gold))}
.hero-kicker .rule:last-child{background:linear-gradient(90deg,var(--gold),transparent)}
.hero-title{margin:1.2rem 0 .4rem;font-size:clamp(3.1rem,10vw,7.4rem);font-weight:700;letter-spacing:.5px;overflow:hidden;padding-bottom:.06em}
.hline{display:block}
.hline.l2{color:var(--gold);font-style:italic}
.hero-title .w.hw{transform:translateY(118%);animation:wrise 1s cubic-bezier(.16,.84,.28,1) both}
@keyframes wrise{to{transform:translateY(0)}}
.hero-title{text-shadow:0 0 30px rgba(212,175,55,.13)}
.hline.l2 .w{
  background:linear-gradient(100deg,#bd942e 0%,#f7ebbf 26%,#d4af37 50%,#f7ebbf 74%,#bd942e 100%);
  background-size:220% 100%;-webkit-background-clip:text;background-clip:text;
  -webkit-text-fill-color:transparent;color:transparent;
  animation:wrise 1s cubic-bezier(.16,.84,.28,1) both, shine 3.8s ease-in-out 1.1s infinite}
@keyframes shine{0%{background-position:130% 0}50%{background-position:-30% 0}100%{background-position:130% 0}}
.title-wrap{position:relative;display:inline-block}
.sparkles{position:absolute;inset:-12% -8%;pointer-events:none;z-index:3}
.spark{position:absolute;color:#fdf2c8;opacity:0;line-height:0;
  filter:drop-shadow(0 0 6px rgba(212,175,55,.9));
  animation:twinkle 2.6s ease-in-out infinite}
.spark svg{width:100%;height:100%;display:block}
@keyframes twinkle{0%,100%{opacity:0;transform:scale(.2) rotate(0deg)}50%{opacity:1;transform:scale(1) rotate(90deg)}}
@keyframes rise{to{opacity:1;transform:none}}
.hero-sub{margin:1.4rem auto 0;max-width:42ch;color:var(--muted);font-weight:300;
  font-size:clamp(1rem,1.6vw,1.2rem);opacity:0;animation:rise 1.1s .42s forwards}
.hero-ctas{margin-top:2.7rem;display:flex;gap:1rem;justify-content:center;flex-wrap:wrap;
  opacity:0;animation:rise 1.1s .6s forwards}
.hero-badge{margin-top:2rem;display:inline-flex;align-items:center;gap:.7rem;
  font-size:.9rem;color:var(--muted);opacity:0;animation:rise 1.1s .78s forwards}
.hero-badge strong{font-family:'Bodoni Moda',serif;font-size:1.3rem;color:var(--gold)}
.stars-sm{color:var(--gold);letter-spacing:1px}
.scroll-cue{position:absolute;bottom:26px;left:50%;transform:translateX(-50%);
  width:24px;height:40px;border:1px solid rgba(212,175,55,.45);border-radius:14px;z-index:3}
.scroll-cue span{position:absolute;top:8px;left:50%;width:4px;height:8px;border-radius:2px;
  background:var(--gold);transform:translateX(-50%);animation:cue 1.6s ease-in-out infinite}
@keyframes cue{0%,100%{opacity:0;top:8px}50%{opacity:1;top:18px}}

/* ===== SECTIONS ===== */
.section{padding:clamp(5rem,11vw,9.5rem) clamp(1.4rem,5vw,4rem);max-width:1280px;margin:0 auto}
.section-head{text-align:center;margin-bottom:4.6rem;display:flex;flex-direction:column;align-items:center}
.section-head .lead{text-align:center;margin-top:1rem}
.of{color:var(--muted);font-size:.6em}

/* trust */
.trust{padding-top:clamp(2.5rem,6vw,4rem)}
.trust-grid{display:grid;grid-template-columns:repeat(6,1fr);gap:1rem}
.trust-card{padding:1.6rem 1.2rem;text-align:center;cursor:default}
.trust-ic{font-size:1.5rem;color:var(--gold)}
.trust-card h3{font-size:1.05rem;margin:.6rem 0 .2rem}
.trust-card p{font-size:.78rem;color:var(--muted);margin:0}

/* collections */
.col-grid{display:grid;grid-template-columns:repeat(4,1fr);gap:1.6rem}
.col-card{position:relative;border-radius:18px;overflow:hidden;min-height:300px;
  border:1px solid rgba(212,175,55,.14);display:flex;cursor:pointer}
.col-card.feature{grid-column:span 2;grid-row:span 2;min-height:620px}
.col-media{flex:1;display:flex;align-items:center;justify-content:center;position:relative;
  transition:transform .6s cubic-bezier(.2,.7,.2,1)}
.col-card:hover .col-media{transform:scale(1.07)}
.col-shine{position:absolute;inset:0;background:
  radial-gradient(60% 50% at 30% 20%,rgba(255,255,255,.16),transparent 60%)}
.col-overlay{position:absolute;inset:0;display:flex;flex-direction:column;justify-content:flex-end;
  padding:1.6rem;background:linear-gradient(180deg,transparent 40%,rgba(9,9,9,.86))}
.col-overlay h3{font-size:1.35rem}
.col-card.feature .col-overlay h3{font-size:2rem}
.col-overlay p{color:var(--muted);font-size:.85rem;margin:.3rem 0 .8rem}
.col-link{color:var(--gold);font-size:.85rem;letter-spacing:.05em;
  opacity:.85;transform:translateX(0);transition:.3s}
.col-card:hover .col-link{opacity:1;transform:translateX(4px)}

/* editorial */
.editorial{display:flex;flex-direction:column;gap:clamp(3rem,7vw,6rem)}
.ed-row{display:grid;grid-template-columns:1.05fr 1fr;gap:clamp(1.5rem,4vw,3.5rem);align-items:center}
.ed-row.rev{direction:rtl}.ed-row.rev>*{direction:ltr}
.ed-media{position:relative;border-radius:20px;min-height:380px;overflow:hidden;
  display:flex;align-items:center;justify-content:center;border:1px solid rgba(212,175,55,.16)}
.ed-shine{position:absolute;inset:0;background:radial-gradient(50% 50% at 40% 25%,rgba(255,255,255,.14),transparent 60%)}
.ed-body .btn{margin-top:1.6rem}

/* why */
.why-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:1.5rem}
.why-card{padding:2rem 1.6rem;cursor:default}
.why-num{font-family:'Bodoni Moda',serif;font-size:1.1rem;color:var(--gold);opacity:.7}
.why-card h3{font-size:1.2rem;margin:.6rem 0 .4rem}
.why-card p{color:var(--muted);font-size:.9rem;margin:0}

/* gallery */
.masonry{column-count:4;column-gap:1rem}
.m-item{break-inside:avoid;margin-bottom:1rem;border-radius:16px;overflow:hidden;position:relative;
  display:flex;align-items:flex-end;border:1px solid rgba(212,175,55,.12);cursor:pointer}
.m-shine{position:absolute;inset:0;background:radial-gradient(60% 50% at 40% 20%,rgba(255,255,255,.14),transparent 60%)}
.m-label{position:relative;z-index:2;padding:1rem;font-family:'Bodoni Moda',serif;font-size:1.05rem;
  width:100%;background:linear-gradient(180deg,transparent,rgba(9,9,9,.8))}

/* reviews */
.rev-grid{display:grid;grid-template-columns:repeat(3,1fr);gap:1.4rem}
.rev-card{padding:2rem 1.8rem;cursor:default}
.stars{color:var(--gold);letter-spacing:2px;font-size:1.1rem}
.rev-q{font-family:'Bodoni Moda',serif;font-size:1.25rem;font-style:italic;margin:1rem 0 1.2rem;line-height:1.4}
.rev-who{color:var(--muted);font-size:.78rem;letter-spacing:.05em}
.rev-cta{text-align:center;margin-top:2.4rem}

/* about */
.about{text-align:center}
.about-wrap{max-width:760px;margin:0 auto;display:flex;flex-direction:column;align-items:center}
.about-wrap .lead{text-align:center}
.about-pillars{display:flex;gap:2rem;margin-top:2rem;flex-wrap:wrap;justify-content:center}
.pillar{display:flex;align-items:center;gap:.6rem;font-family:'Bodoni Moda',serif;font-size:1.05rem}

/* store */
.store-grid{display:grid;grid-template-columns:1.1fr 1fr;gap:clamp(1.5rem,4vw,3rem);align-items:stretch}
.map-wrap{border-radius:20px;overflow:hidden;border:1px solid rgba(212,175,55,.18);min-height:380px}
.map-wrap iframe{width:100%;height:100%;min-height:380px;border:0;filter:grayscale(.3) contrast(1.05)}
.store-info{display:flex;flex-direction:column;justify-content:center}
.store-list{list-style:none;padding:0;margin:1.6rem 0}
.store-list li{padding:1rem 0;border-bottom:1px solid rgba(212,175,55,.12);color:var(--muted);font-size:.95rem}
.store-list span{display:block;color:var(--gold);font-size:.72rem;letter-spacing:.2em;text-transform:uppercase;margin-bottom:.3rem}
.store-actions{display:flex;gap:.8rem;flex-wrap:wrap}

/* whatsapp cta */
.wacta{position:relative;text-align:center;padding:clamp(4rem,9vw,7rem) 1.4rem;overflow:hidden;isolation:isolate}
.wacta-silk{position:absolute;inset:0;z-index:-1;
  background:radial-gradient(70% 120% at 50% 0%,#8a0f28,#2a0010 52%,#070406);
  animation:silk 24s ease-in-out infinite alternate}
.wacta-inner{max-width:680px;margin:0 auto;display:flex;flex-direction:column;align-items:center;gap:1.2rem}
.wacta-inner .lead{text-align:center}

/* footer */
.footer{border-top:1px solid rgba(212,175,55,.14);background:#060606;
  padding:clamp(3rem,6vw,5rem) clamp(1.2rem,5vw,4rem) 2rem}
.foot-grid{max-width:1280px;margin:0 auto;display:grid;grid-template-columns:1.4fr 1fr 1fr 1fr;gap:2rem}
.foot-col h4{font-size:.8rem;letter-spacing:.2em;text-transform:uppercase;color:var(--gold);margin-bottom:1rem;font-family:'Inter',sans-serif;font-weight:600}
.foot-col p,.foot-col a{display:block;color:var(--muted);font-size:.9rem;margin-bottom:.5rem}
.foot-col a:hover{color:var(--gold)}
.foot-note{margin-top:1rem}
.foot-bar{max-width:1280px;margin:2.5rem auto 0;padding-top:1.4rem;border-top:1px solid rgba(255,255,255,.06);
  display:flex;justify-content:space-between;flex-wrap:wrap;gap:.6rem;font-size:.78rem;color:#8a8a8a}
.foot-credit{color:var(--gold);opacity:.8}

/* fab */
.fab{position:fixed;right:18px;bottom:18px;z-index:60;width:54px;height:54px;border-radius:50%;
  display:flex;align-items:center;justify-content:center;
  background:linear-gradient(135deg,#f0d98a,var(--gold));
  box-shadow:0 10px 30px rgba(212,175,55,.4);transition:.3s}
.fab:hover{transform:scale(1.08)}

/* ===== OFFER ===== */
.offer{max-width:1100px;text-align:center;overflow:visible}
.zz-wrap{max-width:900px;margin:0 auto 1rem}
.zz{width:100%;height:68px;display:block;overflow:visible;cursor:crosshair}
.zz .zline{fill:none;stroke:rgba(212,175,55,.55);stroke-width:2;stroke-linejoin:round;stroke-linecap:round;
  stroke-dasharray:14 10;animation:zzflow 7s linear infinite}
@keyframes zzflow{to{stroke-dashoffset:-240}}
.zz .zdot{fill:#f6e7ad;filter:drop-shadow(0 0 9px rgba(212,175,55,.95));transition:cx .12s linear,cy .12s linear}
.offer-ticket{position:relative;max-width:620px;margin:1.6rem auto 0;text-align:center;
  background:linear-gradient(160deg,#2a0010,#120006);border:1px solid rgba(212,175,55,.32);
  border-radius:8px;padding:3.2rem 2.4rem 3rem;box-shadow:0 34px 90px rgba(0,0,0,.55);
  animation:floaty 6s ease-in-out infinite}
@keyframes floaty{0%,100%{transform:translateY(0)}50%{transform:translateY(-8px)}}
.offer-ticket>*{position:relative;z-index:2}
.offer-ticket::after{content:"";position:absolute;inset:0;border-radius:inherit;pointer-events:none;z-index:1;
  background:linear-gradient(105deg,transparent 42%,rgba(246,231,173,.16) 50%,transparent 58%);
  background-size:250% 100%;animation:tshine 5.5s ease-in-out infinite}
@keyframes tshine{0%{background-position:160% 0}100%{background-position:-60% 0}}
.zteeth{position:absolute;left:0;width:100%;height:10px;display:block;z-index:3}
.zteeth.top{top:-9px}.zteeth.bot{bottom:-9px}
.offer-kicker{font-size:.72rem;letter-spacing:.4em;text-transform:uppercase;color:var(--gold);margin:0}
.offer-highlight{font-family:'Bodoni Moda',serif;font-weight:700;font-size:clamp(2.6rem,7vw,4.6rem);
  color:var(--gold);line-height:1;margin:.4rem 0 .2rem}
.offer-title{font-size:clamp(1.3rem,3vw,1.9rem);margin:.4rem 0 .6rem}
.offer-sub{color:var(--muted);font-weight:300;max-width:42ch;margin:0 auto;font-size:.95rem}
.offer-code{display:inline-block;margin:1.4rem 0;padding:.55rem 1.2rem;border:1px dashed rgba(212,175,55,.6);
  border-radius:8px;letter-spacing:.22em;font-size:.82rem;color:#f3e6c4}
.offer-ticket .btn{margin-top:.2rem}
.seal{position:absolute;top:-26px;right:-16px;width:84px;height:84px;display:flex;align-items:center;justify-content:center;z-index:4}
.seal-ring{position:absolute;inset:0;width:100%;height:100%;animation:spin 14s linear infinite}
@keyframes spin{to{transform:rotate(360deg)}}
.seal-txt{font-family:'Bodoni Moda',serif;font-size:.78rem;letter-spacing:.08em;color:var(--gold);
  background:#120006;border-radius:50%;width:54px;height:54px;display:flex;align-items:center;justify-content:center;
  border:1px solid rgba(212,175,55,.4)}

/* ===== ADMIN ===== */
.admin{min-height:100vh;display:flex;align-items:center;justify-content:center;padding:2rem 1.2rem}
.admin-card{width:100%;max-width:560px;background:linear-gradient(160deg,rgba(255,255,255,.05),rgba(255,255,255,.02));
  border:1px solid rgba(212,175,55,.22);border-radius:18px;padding:clamp(1.6rem,4vw,2.4rem);backdrop-filter:blur(10px)}
.admin-h{font-size:1.4rem;margin:.4rem 0 1.4rem}
.admin .field{margin-bottom:1rem;text-align:left}
.admin .field label{display:block;font-size:.68rem;letter-spacing:.18em;text-transform:uppercase;color:var(--gold);margin-bottom:.4rem}
.inp{width:100%;background:rgba(0,0,0,.32);border:1px solid rgba(212,175,55,.25);border-radius:10px;
  color:#fff;padding:.8rem 1rem;font-family:'Inter',sans-serif;font-size:.95rem;outline:none}
.inp:focus{border-color:var(--gold)}
.admin .row{display:flex;gap:.8rem}
.check{display:flex;align-items:center;gap:.6rem;color:var(--muted);font-size:.9rem;margin-top:.4rem}
.note{color:#e8c98a;font-size:.85rem;margin:1rem 0 0;padding:.7rem 1rem;border:1px solid rgba(212,175,55,.2);border-radius:10px;background:rgba(212,175,55,.06)}
.note.ok{color:#cfe8c9;border-color:rgba(120,200,120,.25);background:rgba(120,200,120,.07)}
.setup{margin-top:1.6rem;border-top:1px solid rgba(212,175,55,.14);padding-top:1rem}
.setup summary{cursor:pointer;color:var(--gold);font-size:.85rem;letter-spacing:.04em}
.setup ol{color:var(--muted);font-size:.85rem;padding-left:1.1rem}
.setup pre{background:#060606;border:1px solid rgba(212,175,55,.14);border-radius:10px;padding:1rem;
  overflow:auto;font-size:.72rem;color:#cfc9b8;white-space:pre-wrap;line-height:1.5}

/* ===== PAGES & TRANSITIONS ===== */
.page{will-change:transform,opacity,filter}
.page.in{animation:pageIn .8s cubic-bezier(.16,.84,.28,1) both}
.page.out{animation:pageOut .34s cubic-bezier(.4,0,.6,1) both}
@keyframes pageIn{
  0%{opacity:0;filter:blur(16px);transform:translateY(30px) scale(.992);clip-path:inset(0 0 7% 0)}
  100%{opacity:1;filter:blur(0);transform:none;clip-path:inset(0 0 0 0)}}
@keyframes pageOut{to{opacity:0;filter:blur(11px);transform:translateY(-16px) scale(.992)}}

.page>.section:first-child,.page>.page-head:first-child{padding-top:clamp(7rem,12vw,10rem)}
.page-head{max-width:900px;margin:0 auto;text-align:center;
  padding:clamp(7rem,12vw,10rem) clamp(1.4rem,5vw,4rem) 0}
.page-head .ph-sub{margin:1.1rem auto 0;text-align:center}

.nav-links a.active{color:var(--gold)}

/* top gold progress bar during route change */
.route-bar{position:fixed;top:0;left:0;height:3px;width:0;z-index:70;opacity:0;
  background:linear-gradient(90deg,#f0d98a,var(--gold));box-shadow:0 0 12px rgba(212,175,55,.6)}
.route-bar.out{animation:rbout .34s linear forwards}
.route-bar.in{animation:rbin .8s ease forwards}
@keyframes rbout{0%{width:0;opacity:1}100%{width:72%;opacity:1}}
@keyframes rbin{0%{width:72%;opacity:1}55%{width:100%;opacity:1}100%{width:100%;opacity:0}}

@media (prefers-reduced-motion:reduce){
  .page.in,.page.out{animation:none}
  .route-bar{display:none}
}

/* ===== RESPONSIVE ===== */
@media (max-width:1024px){
  .trust-grid{grid-template-columns:repeat(3,1fr)}
  .col-grid{grid-template-columns:repeat(2,1fr)}
  .col-card.feature{grid-column:span 2;min-height:340px;grid-row:auto}
  .why-grid,.rev-grid{grid-template-columns:repeat(2,1fr)}
  .masonry{column-count:3}
}
@media (max-width:760px){
  .burger{display:flex}
  .nav-links{position:fixed;inset:0 0 auto 0;top:0;height:100vh;flex-direction:column;justify-content:center;
    gap:2rem;background:rgba(9,9,9,.97);backdrop-filter:blur(16px);transform:translateY(-100%);transition:.4s}
  .nav-links.open{transform:none}
  .nav-links a{font-size:1.3rem}
  .trust-grid{grid-template-columns:repeat(2,1fr)}
  .col-grid{grid-template-columns:1fr}
  .col-card.feature{grid-column:auto}
  .ed-row,.ed-row.rev{grid-template-columns:1fr;direction:ltr}
  .why-grid,.rev-grid{grid-template-columns:1fr}
  .masonry{column-count:2}
  .store-grid{grid-template-columns:1fr}
  .foot-grid{grid-template-columns:1fr 1fr}
  .foot-bar{flex-direction:column;text-align:center}
}
@media (max-width:760px){
  .seal{top:-20px;right:-6px;width:66px;height:66px}
  .seal-txt{width:44px;height:44px;font-size:.66rem}
  .admin .row{flex-direction:column;gap:0}
  .offer-ticket{padding:2.6rem 1.6rem}
}
@media (max-width:420px){
  .masonry{column-count:1}
  .foot-grid{grid-template-columns:1fr}
  .hero-ctas{flex-direction:column;width:100%}
  .hero-ctas .btn{width:100%}
}
`;
