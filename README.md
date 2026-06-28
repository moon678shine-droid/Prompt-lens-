# Prompt-lens-
import { useState, useRef, useCallback, useEffect } from "react";

const REMIX_STYLES = [
  { emoji: "✨", label: "Luxury", modifier: "ultra-luxury editorial, high-end fashion, gold accents, opulent atmosphere, Vogue cover quality, aspirational" },
  { emoji: "🎬", label: "Cinematic", modifier: "cinematic movie still, anamorphic lens flare, film grain, dramatic color grade, Hollywood blockbuster production value" },
  { emoji: "🖤", label: "Dark", modifier: "dark moody aesthetic, deep shadows, gothic atmosphere, noir style, dramatic contrast, brooding" },
  { emoji: "🌸", label: "Soft", modifier: "soft dreamy aesthetic, pastel tones, ethereal glow, romantic atmosphere, delicate diffused lighting" },
  { emoji: "👗", label: "Editorial", modifier: "high fashion editorial, magazine spread, avant-garde styling, artistic composition, fashion week" },
  { emoji: "🎨", label: "Anime", modifier: "anime art style, Studio Ghibli inspired, detailed illustration, vibrant colors, cel shading, expressive" },
  { emoji: "🌆", label: "Cyberpunk", modifier: "cyberpunk aesthetic, neon lights, futuristic city, rain-soaked streets, dystopian atmosphere, holographic" },
  { emoji: "🎭", label: "Vintage", modifier: "vintage film photography, 1970s aesthetic, grain texture, faded colors, retro warm mood, analog" },
];

const PLATFORMS = [
  { label: "Midjourney", key: "midjourney", color: "#7c3aed" },
  { label: "DALL-E 3", key: "dalle", color: "#0891b2" },
  { label: "Stable Diffusion", key: "sd", color: "#d97706" },
  { label: "Flux", key: "flux", color: "#059669" },
  { label: "Ideogram", key: "ideogram", color: "#db2777" },
];

const FOLDERS = ["All", "Fashion", "Characters", "Logos", "Wallpapers", "Architecture", "Portraits"];

const FIDELITY_LABELS = ["Free Interpretation", "Inspired By", "Similar Feel", "Close Match", "Near Exact"];

function CopyBtn({ text, label }) {
  const [copied, setCopied] = useState(false);
  return (
    <button onClick={() => { navigator.clipboard.writeText(text); setCopied(true); setTimeout(() => setCopied(false), 1800); }}
      style={{ padding: "5px 11px", borderRadius: 6, border: `1px solid ${copied ? "#7c3aed" : "#1e1b2e"}`, background: copied ? "rgba(124,58,237,0.15)" : "transparent", color: copied ? "#a78bfa" : "#555270", fontSize: 11, fontWeight: 600, cursor: "pointer", whiteSpace: "nowrap" }}>
      {copied ? "✓" : "Copy"} {label || ""}
    </button>
  );
}

function Tag({ label, value, color = "#7c3aed" }) {
  return (
    <div style={{ display: "inline-flex", alignItems: "center", gap: 5, background: "#0f0d1e", borderRadius: 7, padding: "4px 10px", border: "1px solid #1a1730" }}>
      <span style={{ fontSize: 9, fontWeight: 800, color, textTransform: "uppercase", letterSpacing: "0.1em" }}>{label}</span>
      <span style={{ fontSize: 12, color: "#c4b5fd" }}>{value}</span>
    </div>
  );
}

function Slider({ value, onChange }) {
  return (
    <div>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 8 }}>
        <span style={{ fontSize: 10, color: "#555270", fontWeight: 700, textTransform: "uppercase", letterSpacing: "0.1em" }}>Image Fidelity</span>
        <span style={{ fontSize: 12, color: "#a78bfa", fontWeight: 600 }}>{FIDELITY_LABELS[value - 1]}</span>
      </div>
      <input type="range" min={1} max={5} value={value} onChange={e => onChange(Number(e.target.value))}
        style={{ width: "100%", accentColor: "#7c3aed", cursor: "pointer" }} />
      <div style={{ display: "flex", justifyContent: "space-between", fontSize: 10, color: "#333050", marginTop: 4 }}>
        <span>Creative freedom</span><span>Exact recreation</span>
      </div>
    </div>
  );
}

export default function PromptLensV3() {
  const [tab, setTab] = useState("upload");
  const [image, setImage] = useState(null);
  const [imageBase64, setImageBase64] = useState(null);
  const [dragging, setDragging] = useState(false);
  const [loading, setLoading] = useState(false);
  const [remixLoading, setRemixLoading] = useState(null);
  const [result, setResult] = useState(null);
  const [error, setError] = useState(null);
  const [fidelity, setFidelity] = useState(3);
  const [activeFolder, setActiveFolder] = useState("All");
  const [showHistory, setShowHistory] = useState(false);
  const [history, setHistory] = useState(() => {
    try { return JSON.parse(localStorage.getItem("pl_history") || "[]"); } catch { return []; }
  });
  const [builder, setBuilder] = useState({ subject: "", style: "", lighting: "", camera: "", mood: "", extra: "" });
  const [builderResult, setBuilderResult] = useState(null);
  const [builderLoading, setBuilderLoading] = useState(false);
  const [expandedPlatform, setExpandedPlatform] = useState(null);
  const fileRef = useRef();

  useEffect(() => {
    try { localStorage.setItem("pl_history", JSON.stringify(history.slice(0, 20))); } catch {}
  }, [history]);

  const processFile = (file) => {
    if (!file?.type.startsWith("image/")) { setError("Upload a valid image file."); return; }
    setError(null); setResult(null); setBuilderResult(null);
    const img = new Image();
    const url = URL.createObjectURL(file);
    img.onload = () => {
      const canvas = document.createElement("canvas");
      const MAX = 1024;
      let w = img.width, h = img.height;
      if (w > MAX || h > MAX) { if (w > h) { h = Math.round(h * MAX / w); w = MAX; } else { w = Math.round(w * MAX / h); h = MAX; } }
      canvas.width = w; canvas.height = h;
      canvas.getContext("2d").drawImage(img, 0, 0, w, h);
      setImageBase64(canvas.toDataURL("image/jpeg", 0.85).split(",")[1]);
      setImage(url);
    };
    img.src = url;
  };

  const onDrop = useCallback((e) => { e.preventDefault(); setDragging(false); processFile(e.dataTransfer.files[0]); }, []);

  const callClaude = async (messages, system) => {
    const res = await fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({ model: "claude-sonnet-4-6", max_tokens: 1500, system, messages }),
    });
    const data = await res.json();
    return (data.content?.map(c => c.text || "").join("") || "").replace(/```json[\s\S]*?```|```/g, "").trim();
  };

  const fidelityInstruction = {
    1: "Treat the image only as loose inspiration. Reimagine completely — change style, subject feel, and composition freely.",
    2: "Use the image as a starting point. Keep the general vibe but allow creative liberties with style and details.",
    3: "Balance fidelity and creativity. Preserve the mood and composition but allow stylistic interpretation.",
    4: "Stay close to the original. Preserve subject, lighting, composition, and style with minor creative additions.",
    5: "Recreate as precisely as possible. Describe every detail — subject, pose, lighting, camera, colors, background — for near-exact reproduction.",
  }[fidelity];

  const analyze = async () => {
    if (!imageBase64) return;
    setLoading(true); setResult(null); setError(null);
    try {
      const raw = await callClaude([{
        role: "user",
        content: [
          { type: "image", source: { type: "base64", media_type: "image/jpeg", data: imageBase64 } },
          { type: "text", text: `Analyze this image and generate prompts. Fidelity instruction: ${fidelityInstruction}` }
        ]
      }], `You are an elite AI prompt engineer and visual analyst. Generate prompts to recreate images based on the fidelity level given.

Respond ONLY in JSON (no extra text, no markdown):
{
  "analysis": {
    "style": "visual style in 3-5 words",
    "mood": "mood in 3-5 words",
    "lighting": "lighting type",
    "camera": "camera/lens estimate",
    "composition": "composition description",
    "colors": "dominant palette",
    "key_elements": ["el1","el2","el3","el4","el5"]
  },
  "prompts": {
    "midjourney": "MJ prompt with --ar --style --v 6.1",
    "dalle": "Natural language DALL-E 3 prompt",
    "sd": "SD tags with quality boosters, masterpiece, best quality",
    "flux": "Detailed photographic Flux prompt",
    "ideogram": "Clear Ideogram prompt"
  },
  "negative_prompt": "bad anatomy, blurry, watermark, low quality, deformed...",
  "why_it_works": {
    "key_technique": "main technique that makes this image work",
    "lighting_reason": "why this lighting choice works",
    "composition_reason": "why this composition works",
    "pro_insight": "one thing beginners miss about recreating this type of image"
  },
  "tip": "one specific pro tip for best results"
}`);
      const parsed = JSON.parse(raw);
      setResult(parsed);
      const entry = { id: Date.now(), thumbnail: image, result: parsed, folder: "All", label: parsed.analysis?.style || "analyzed" };
      setHistory(h => [entry, ...h.slice(0, 19)]);
    } catch (e) {
      setError("Analysis failed — try a different image or check your connection.");
      console.error(e);
    }
    setLoading(false);
  };

  const remix = async (style) => {
    if (!result) return;
    setRemixLoading(style.label);
    try {
      const raw = await callClaude([{
        role: "user",
        content: `Base prompts: ${JSON.stringify(result.prompts)}\n\nRemix modifier: ${style.modifier}\n\nKeep the core subject/composition but completely transform the aesthetic.`
      }], `Remix AI image prompts into a new style. Return ONLY JSON:
{"prompts":{"midjourney":"...","dalle":"...","sd":"...","flux":"...","ideogram":"..."},"negative_prompt":"...","tip":"tip for this specific style"}`);
      const parsed = JSON.parse(raw);
      setResult(r => ({ ...r, ...parsed }));
    } catch { setError("Remix failed, try again."); }
    setRemixLoading(null);
  };

  const buildPrompt = async () => {
    if (!Object.values(builder).some(v => v.trim())) return;
    setBuilderLoading(true); setBuilderResult(null);
    try {
      const raw = await callClaude([{
        role: "user",
        content: `Generate AI image prompts from:\nSubject: ${builder.subject||"not specified"}\nStyle: ${builder.style||"not specified"}\nLighting: ${builder.lighting||"not specified"}\nCamera: ${builder.camera||"not specified"}\nMood: ${builder.mood||"not specified"}\nExtra: ${builder.extra||"none"}`
      }], `Build optimized AI image prompts from structured parameters. Return ONLY JSON:
{"prompts":{"midjourney":"...","dalle":"...","sd":"...","flux":"...","ideogram":"..."},"negative_prompt":"...","why_it_works":{"key_technique":"...","lighting_reason":"...","composition_reason":"...","pro_insight":"..."},"tip":"..."}`);
      setBuilderResult(JSON.parse(raw));
    } catch { setError("Builder failed, try again."); }
    setBuilderLoading(false);
  };

  const activeResult = result || builderResult;
  const displayedHistory = activeFolder === "All" ? history : history.filter(h => h.folder === activeFolder);

  const S = {
    page: { minHeight: "100vh", background: "#07060f", color: "#e2e0f0", fontFamily: "'Inter', system-ui, sans-serif" },
    header: { borderBottom: "1px solid #12102a", padding: "14px 24px", display: "flex", alignItems: "center", justifyContent: "space-between", background: "#09081a", position: "sticky", top: 0, zIndex: 20 },
    main: { maxWidth: 920, margin: "0 auto", padding: "24px 18px", display: "flex", gap: 20 },
    card: (active) => ({ background: "#0c0b1a", border: `1px solid ${active ? "#6d28d9" : "#14122a"}`, borderRadius: 13, overflow: "hidden", transition: "border-color 0.15s" }),
    section: { background: "#0c0b1a", border: "1px solid #14122a", borderRadius: 13, padding: "16px 18px" },
    label: { fontSize: 10, color: "#444060", fontWeight: 800, textTransform: "uppercase", letterSpacing: "0.12em", marginBottom: 8, display: "block" },
    input: { width: "100%", background: "#0f0d1e", border: "1px solid #1a1730", borderRadius: 8, padding: "9px 12px", color: "#e2e0f0", fontSize: 13, outline: "none", boxSizing: "border-box" },
    tabBtn: (active) => ({ padding: "7px 16px", borderRadius: 7, border: "none", background: active ? "rgba(109,40,217,0.2)" : "transparent", color: active ? "#a78bfa" : "#444060", fontSize: 12, fontWeight: 700, cursor: "pointer" }),
    primaryBtn: (loading) => ({ padding: "11px 22px", borderRadius: 10, border: "none", background: loading ? "#2d1f5e" : "linear-gradient(135deg,#6d28d9,#9333ea)", color: "#fff", fontSize: 14, fontWeight: 700, cursor: loading ? "not-allowed" : "pointer", transition: "all 0.2s" }),
  };

  return (
    <div style={S.page}>
      <div style={S.header}>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ width: 30, height: 30, background: "linear-gradient(135deg,#6d28d9,#9333ea)", borderRadius: 8, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 15 }}>🔮</div>
          <span style={{ fontWeight: 900, fontSize: 16, letterSpacing: "-0.5px" }}>PromptLens</span>
          <span style={{ fontSize: 10, background: "rgba(109,40,217,0.2)", color: "#a78bfa", padding: "2px 7px", borderRadius: 20, fontWeight: 700 }}>v3</span>
        </div>
        <div style={{ display: "flex", gap: 4 }}>
          <button style={S.tabBtn(tab === "upload")} onClick={() => setTab("upload")}>📸 Analyzer</button>
          <button style={S.tabBtn(tab === "builder")} onClick={() => setTab("builder")}>🛠 Builder</button>
          <button style={{ ...S.tabBtn(showHistory), position: "relative" }} onClick={() => setShowHistory(h => !h)}>
            🕓 History {history.length > 0 && <span style={{ position: "absolute", top: 3, right: 3, width: 6, height: 6, background: "#7c3aed", borderRadius: "50%" }} />}
          </button>
        </div>
      </div>

      <div style={S.main}>
        {/* Sidebar history */}
        {showHistory && (
          <div style={{ width: 200, flexShrink: 0 }}>
            <div style={{ ...S.section, padding: "12px 14px" }}>
              <span style={S.label}>Projects</span>
              <div style={{ display: "flex", flexDirection: "column", gap: 2, marginBottom: 12 }}>
                {FOLDERS.map(f => (
                  <button key={f} onClick={() => setActiveFolder(f)} style={{ padding: "6px 10px", borderRadius: 7, border: "none", background: activeFolder === f ? "rgba(109,40,217,0.2)" : "transparent", color: activeFolder === f ? "#a78bfa" : "#555270", fontSize: 12, textAlign: "left", cursor: "pointer" }}>
                    {f} {f === "All" ? `(${history.length})` : `(${history.filter(h => h.folder === f).length})`}
                  </button>
                ))}
              </div>
              <span style={S.label}>Recent</span>
              <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
                {displayedHistory.length === 0 && <span style={{ fontSize: 11, color: "#333050" }}>No history yet</span>}
                {displayedHistory.map(item => (
                  <div key={item.id} onClick={() => { setResult(item.result); setTab("upload"); }}
                    style={{ borderRadius: 8, overflow: "hidden", cursor: "pointer", border: "1px solid #14122a", transition: "border-color 0.15s" }}
                    onMouseEnter={e => e.currentTarget.style.borderColor = "#6d28d9"}
                    onMouseLeave={e => e.currentTarget.style.borderColor = "#14122a"}
                  >
                    <img src={item.thumbnail} alt="" style={{ width: "100%", height: 70, objectFit: "cover", display: "block" }} />
                    <div style={{ padding: "5px 7px", background: "#0c0b1a", fontSize: 10, color: "#444060", whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>{item.label}</div>
                  </div>
                ))}
              </div>
            </div>
          </div>
        )}

        {/* Main content */}
        <div style={{ flex: 1, display: "flex", flexDirection: "column", gap: 14 }}>

          {/* ANALYZER TAB */}
          {tab === "upload" && (
            <>
              {!image ? (
                <div onDragOver={e => { e.preventDefault(); setDragging(true); }} onDragLeave={() => setDragging(false)} onDrop={onDrop} onClick={() => fileRef.current.click()}
                  style={{ border: `2px dashed ${dragging ? "#6d28d9" : "#14122a"}`, borderRadius: 13, padding: "48px 24px", textAlign: "center", cursor: "pointer", background: dragging ? "rgba(109,40,217,0.05)" : "#0c0b1a", transition: "all 0.2s" }}>
                  <div style={{ fontSize: 38, marginBottom: 10 }}>🖼️</div>
                  <div style={{ fontSize: 15, fontWeight: 700, marginBottom: 6 }}>Drop your reference image</div>
                  <div style={{ fontSize: 12, color: "#444060" }}>JPG, PNG, WebP · auto-compressed · any source</div>
                  <input ref={fileRef} type="file" accept="image/*" onChange={e => processFile(e.target.files[0])} style={{ display: "none" }} />
                </div>
              ) : (
                <div style={S.card(false)}>
                  <img src={image} alt="uploaded" style={{ width: "100%", maxHeight: 300, objectFit: "contain", display: "block", background: "#06050e" }} />
                  <div style={{ padding: "14px 16px", display: "flex", flexDirection: "column", gap: 12 }}>
                    <Slider value={fidelity} onChange={setFidelity} />
                    <div style={{ display: "flex", gap: 10 }}>
                      <button onClick={analyze} disabled={loading} style={{ ...S.primaryBtn(loading), flex: 1 }}>
                        {loading ? "Analyzing…" : "🔍 Generate Complete Recipe"}
                      </button>
                      <button onClick={() => { setImage(null); setImageBase64(null); setResult(null); }}
                        style={{ padding: "11px 14px", borderRadius: 10, border: "1px solid #14122a", background: "transparent", color: "#444060", fontSize: 13, cursor: "pointer" }}>
                        Clear
                      </button>
                    </div>
                  </div>
                </div>
              )}

              {/* Analysis */}
              {result?.analysis && (
                <div style={S.section}>
                  <span style={S.label}>Visual Analysis</span>
                  <div style={{ display: "flex", gap: 6, flexWrap: "wrap", marginBottom: 10 }}>
                    <Tag label="Style" value={result.analysis.style} color="#7c3aed" />
                    <Tag label="Mood" value={result.analysis.mood} color="#0891b2" />
                    <Tag label="Light" value={result.analysis.lighting} color="#d97706" />
                    <Tag label="Camera" value={result.analysis.camera} color="#059669" />
                  </div>
                  <div style={{ fontSize: 12, color: "#444060", marginBottom: 10 }}>{result.analysis.composition} · {result.analysis.colors}</div>
                  <div style={{ display: "flex", gap: 5, flexWrap: "wrap" }}>
                    {result.analysis.key_elements?.map((el, i) => (
                      <span key={i} style={{ padding: "3px 8px", background: "#0f0d1e", borderRadius: 20, fontSize: 11, color: "#a78bfa", border: "1px solid #1a1730" }}>{el}</span>
                    ))}
                  </div>
                </div>
              )}

              {/* Remix */}
              {result && (
                <div style={S.section}>
                  <span style={S.label}>Prompt Remix</span>
                  <div style={{ display: "flex", gap: 7, flexWrap: "wrap" }}>
                    {REMIX_STYLES.map(s => (
                      <button key={s.label} onClick={() => remix(s)} disabled={!!remixLoading}
                        style={{ padding: "6px 13px", borderRadius: 20, border: "1px solid #1a1730", background: remixLoading === s.label ? "rgba(109,40,217,0.2)" : "#0f0d1e", color: remixLoading === s.label ? "#a78bfa" : "#c4b5fd", fontSize: 12, fontWeight: 500, cursor: "pointer", opacity: remixLoading && remixLoading !== s.label ? 0.35 : 1 }}>
                        {remixLoading === s.label ? "…" : s.emoji} {s.label}
                      </button>
                    ))}
                  </div>
                </div>
              )}
            </>
          )}

          {/* BUILDER TAB */}
          {tab === "builder" && (
            <div style={S.section}>
              <span style={S.label}>Visual Prompt Builder</span>
              <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginBottom: 14 }}>
                {[["Subject", "subject", "a woman in a white dress"],["Style", "style", "vogue editorial, fashion"],["Lighting", "lighting", "golden hour, soft diffused"],["Camera / Lens", "camera", "85mm portrait, eye level"],["Mood", "mood", "mysterious, powerful"],["Extra Details", "extra", "rain, bokeh, fog"]].map(([lbl, key, ph]) => (
                  <div key={key}>
                    <span style={S.label}>{lbl}</span>
                    <input style={S.input} placeholder={ph} value={builder[key]} onChange={e => setBuilder(b => ({ ...b, [key]: e.target.value }))} />
                  </div>
                ))}
              </div>
              <button onClick={buildPrompt} disabled={builderLoading} style={S.primaryBtn(builderLoading)}>
                {builderLoading ? "Building…" : "🛠 Build All Platform Prompts"}
              </button>
            </div>
          )}

          {/* SHARED OUTPUT */}
          {activeResult?.prompts && (
            <>
              {/* Platform prompts */}
              <div style={{ ...S.card(false), overflow: "visible" }}>
                <div style={{ padding: "12px 16px", borderBottom: "1px solid #14122a", display: "flex", justifyContent: "space-between", alignItems: "center", flexWrap: "wrap", gap: 8 }}>
                  <span style={S.label}>AI Generation Recipe</span>
                  <div style={{ display: "flex", gap: 6, flexWrap: "wrap" }}>
                    {PLATFORMS.map(p => activeResult.prompts[p.key] ? <CopyBtn key={p.key} text={activeResult.prompts[p.key]} label={p.label} /> : null)}
                  </div>
                </div>
                {PLATFORMS.map(p => !activeResult.prompts[p.key] ? null : (
                  <div key={p.key} style={{ borderBottom: "1px solid #0e0c1e" }}>
                    <div
                      onClick={() => setExpandedPlatform(expandedPlatform === p.key ? null : p.key)}
                      style={{ padding: "12px 16px", display: "flex", justifyContent: "space-between", alignItems: "center", cursor: "pointer" }}
                    >
                      <span style={{ fontSize: 11, fontWeight: 800, color: p.color, textTransform: "uppercase", letterSpacing: "0.08em" }}>{p.label}</span>
                      <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
                        <span style={{ fontSize: 11, color: "#333050" }}>{expandedPlatform === p.key ? "▲" : "▼"}</span>
                        <CopyBtn text={activeResult.prompts[p.key]} />
                      </div>
                    </div>
                    {expandedPlatform === p.key && (
                      <div style={{ padding: "0 16px 14px", fontSize: 12.5, color: "#b8b5d4", lineHeight: 1.7, fontFamily: "monospace", whiteSpace: "pre-wrap", wordBreak: "break-word" }}>
                        {activeResult.prompts[p.key]}
                      </div>
                    )}
                    {expandedPlatform !== p.key && (
                      <div style={{ padding: "0 16px 12px", fontSize: 12, color: "#555270", overflow: "hidden", textOverflow: "ellipsis", whiteSpace: "nowrap" }}>
                        {activeResult.prompts[p.key]}
                      </div>
                    )}
                  </div>
                ))}
              </div>

              {/* Negative prompt */}
              {activeResult.negative_prompt && (
                <div style={{ background: "rgba(220,38,38,0.04)", border: "1px solid rgba(220,38,38,0.12)", borderRadius: 13, padding: "14px 16px" }}>
                  <div style={{ display: "flex", justifyContent: "space-between", marginBottom: 8 }}>
                    <span style={{ ...S.label, color: "#f87171" }}>Negative Prompt</span>
                    <CopyBtn text={activeResult.negative_prompt} />
                  </div>
                  <div style={{ fontSize: 12.5, color: "#fca5a5", fontFamily: "monospace", lineHeight: 1.6 }}>{activeResult.negative_prompt}</div>
                </div>
              )}

              {/* Why it works */}
              {activeResult.why_it_works && (
                <div style={{ background: "rgba(8,145,178,0.05)", border: "1px solid rgba(8,145,178,0.15)", borderRadius: 13, padding: "16px 18px" }}>
                  <span style={{ ...S.label, color: "#22d3ee" }}>Why This Works</span>
                  <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
                    {[
                      ["🎯 Key Technique", activeResult.why_it_works.key_technique],
                      ["💡 Lighting Logic", activeResult.why_it_works.lighting_reason],
                      ["📐 Composition", activeResult.why_it_works.composition_reason],
                      ["🔑 Pro Insight", activeResult.why_it_works.pro_insight],
                    ].map(([lbl, val]) => val ? (
                      <div key={lbl} style={{ display: "flex", gap: 10, alignItems: "flex-start" }}>
                        <span style={{ fontSize: 11, fontWeight: 700, color: "#22d3ee", whiteSpace: "nowrap", minWidth: 110 }}>{lbl}</span>
                        <span style={{ fontSize: 12.5, color: "#94e3f5", lineHeight: 1.6 }}>{val}</span>
                      </div>
                    ) : null)}
                  </div>
                </div>
              )}

              {activeResult.tip && (
                <div style={{ padding: "11px 15px", background: "rgba(109,40,217,0.08)", borderRadius: 10, border: "1px solid rgba(109,40,217,0.2)", fontSize: 12.5, color: "#c4b5fd" }}>
                  <span style={{ fontWeight: 700, marginRight: 8 }}>✨ TIP</span>{activeResult.tip}
                </div>
              )}
            </>
          )}

          {error && (
            <div style={{ padding: "12px 15px", borderRadius: 10, background: "rgba(239,68,68,0.07)", border: "1px solid rgba(239,68,68,0.18)", color: "#f87171", fontSize: 13 }}>{error}</div>
          )}
        </div>
      </div>
    </div>
  );
}
