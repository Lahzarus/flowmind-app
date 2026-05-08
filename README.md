# flowmind-app
import { useState, useRef, useEffect } from "react";

// ─── Data ────────────────────────────────────────────────────────────────────

const AUTOMATIONS = [
  {
    id: "email", icon: "✉️", label: "Email Reply", category: "Communication",
    description: "Draft professional replies to any email instantly",
    placeholder: "Paste the email you received...",
    systemPrompt: "You are an expert business communicator. Given an email, write a professional, warm, and concise reply. Include a proper greeting, clear body, and sign-off. Keep it under 150 words and make it sound human, not robotic.",
    outputLabel: "Generated Reply", color: "#3B82F6",
  },
  {
    id: "meeting", icon: "📋", label: "Meeting Notes", category: "Productivity",
    description: "Transform messy notes into structured summaries",
    placeholder: "Paste your raw meeting notes here...",
    systemPrompt: "You are a professional executive assistant. Convert raw meeting notes into a structured summary with these sections: **Key Decisions**, **Action Items** (with owners if mentioned), and **Next Steps**. Use bullet points. Be concise and clear.",
    outputLabel: "Structured Summary", color: "#8B5CF6",
  },
  {
    id: "leads", icon: "🎯", label: "Lead Qualifier", category: "Sales",
    description: "Score and analyze any sales lead in seconds",
    placeholder: "Describe the lead — company, size, pain points, budget...",
    systemPrompt: "You are a senior sales strategist. Analyze this sales lead and return: 1) Score (X/10) with one-line rationale, 2) Top 3 qualifying signals, 3) Top 2 red flags, 4) Recommended next action. Be direct, confident, and concise.",
    outputLabel: "Lead Analysis", color: "#F59E0B",
  },
  {
    id: "sop", icon: "📝", label: "SOP Generator", category: "Operations",
    description: "Create standard operating procedures instantly",
    placeholder: "Describe the process (e.g. 'onboarding a new client')...",
    systemPrompt: "You are a business process expert. Create a clear, professional SOP for the given process. Include: Purpose, Scope, Numbered step-by-step instructions, and Notes/Tips. Keep it practical and actionable.",
    outputLabel: "SOP Document", color: "#10B981",
  },
  {
    id: "content", icon: "✍️", label: "Content Writer", category: "Marketing",
    description: "Write compelling marketing copy and content",
    placeholder: "Describe what you need written (product, audience, tone)...",
    systemPrompt: "You are a world-class copywriter. Write compelling, persuasive marketing content based on the brief. Match the requested tone, keep it engaging, and make every word earn its place. Include a strong hook and clear call to action.",
    outputLabel: "Marketing Copy", color: "#EC4899",
  },
  {
    id: "data", icon: "🔍", label: "Data Extractor", category: "Analytics",
    description: "Pull structured data from any unstructured text",
    placeholder: "Paste any text — emails, docs, messages, reports...",
    systemPrompt: "You are a data extraction specialist. Extract all key structured information from the given text. Present it first as clean JSON with descriptive keys, then provide a 2-3 sentence plain-English summary of what was found.",
    outputLabel: "Extracted Data", color: "#06B6D4",
  },
  {
    id: "report", icon: "📊", label: "Report Writer", category: "Business",
    description: "Turn bullet points into polished business reports",
    placeholder: "List your key points, findings, or data...",
    systemPrompt: "You are a professional business writer. Transform the given points into a polished, well-structured business report with an Executive Summary, detailed sections, and a Conclusion. Use professional corporate tone.",
    outputLabel: "Business Report", color: "#F97316",
  },
  {
    id: "translate", icon: "🌐", label: "Smart Translator", category: "Communication",
    description: "Translate and localize content naturally",
    placeholder: "Paste the text to translate and specify target language...",
    systemPrompt: "You are a professional translator and localization expert. Translate the given text naturally — not word-for-word, but capturing tone, context, and cultural nuance. State the detected source language and target language clearly.",
    outputLabel: "Translation", color: "#6366F1",
  },
];

const CATEGORIES = ["All", ...Array.from(new Set(AUTOMATIONS.map(a => a.category)))];

// ─── Persistent History via localStorage ─────────────────────────────────────

const getHistory = () => {
  try { return JSON.parse(localStorage.getItem("fm_history") || "[]"); } catch { return []; }
};
const saveHistory = (h) => {
  try { localStorage.setItem("fm_history", JSON.stringify(h.slice(0, 50))); } catch {}
};

// ─── Main App ─────────────────────────────────────────────────────────────────

export default function FlowMindPro() {
  const [view, setView] = useState("app"); // "app" | "history" | "landing"
  const [selected, setSelected] = useState(AUTOMATIONS[0]);
  const [input, setInput] = useState("");
  const [output, setOutput] = useState("");
  const [loading, setLoading] = useState(false);
  const [ran, setRan] = useState(false);
  const [copied, setCopied] = useState(false);
  const [elapsed, setElapsed] = useState(null);
  const [tokens, setTokens] = useState(null);
  const [history, setHistory] = useState(getHistory());
  const [filterCat, setFilterCat] = useState("All");
  const [toast, setToast] = useState(null);
  const [apiKey, setApiKey] = useState(() => localStorage.getItem("fm_apikey") || "");
  const [showKeyModal, setShowKeyModal] = useState(false);
  const [keyInput, setKeyInput] = useState("");
  const outputRef = useRef(null);

  const showToast = (msg, type = "success") => {
    setToast({ msg, type });
    setTimeout(() => setToast(null), 3000);
  };

  const filtered = filterCat === "All" ? AUTOMATIONS : AUTOMATIONS.filter(a => a.category === filterCat);

  const runAutomation = async () => {
    if (!input.trim() || loading) return;
    setLoading(true); setOutput(""); setRan(false);
    const t0 = Date.now();
    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 1000,
          system: selected.systemPrompt,
          messages: [{ role: "user", content: input }],
        }),
      });
      const data = await res.json();
      if (data.error) throw new Error(data.error.message);
      const text = data.content?.map(b => b.text || "").join("") || "";
      const secs = ((Date.now() - t0) / 1000).toFixed(1);
      setOutput(text); setElapsed(secs); setTokens(data.usage?.output_tokens || null); setRan(true);
      const entry = {
        id: Date.now(), automation: selected.label, icon: selected.icon,
        input: input.slice(0, 120) + (input.length > 120 ? "…" : ""),
        output: text, elapsed: secs, tokens: data.usage?.output_tokens,
        date: new Date().toLocaleString(),
      };
      const newH = [entry, ...history];
      setHistory(newH); saveHistory(newH);
      setTimeout(() => outputRef.current?.scrollIntoView({ behavior: "smooth", block: "nearest" }), 100);
    } catch (e) {
      setOutput("⚠️ " + (e.message || "Something went wrong. Please try again."));
      setRan(true);
    } finally { setLoading(false); }
  };

  const reset = () => { setInput(""); setOutput(""); setRan(false); setElapsed(null); setTokens(null); };
  const handleCopy = () => { navigator.clipboard.writeText(output); setCopied(true); showToast("Copied to clipboard!"); setTimeout(() => setCopied(false), 2000); };
  const clearHistory = () => { setHistory([]); saveHistory([]); showToast("History cleared"); };

  const saveKey = () => {
    localStorage.setItem("fm_apikey", keyInput);
    setApiKey(keyInput);
    setShowKeyModal(false);
    showToast("API key saved!");
  };

  // ── Landing page ──
  if (view === "landing") return <Landing onEnter={() => setView("app")} />;

  return (
    <div style={{ minHeight: "100vh", background: "#080C14", color: "#E2E8F5", fontFamily: "'Plus Jakarta Sans', 'Segoe UI', sans-serif", display: "flex", flexDirection: "column" }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@300;400;500;600;700;800&family=JetBrains+Mono:wght@400;500&display=swap');
        *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }
        ::-webkit-scrollbar { width: 5px; } ::-webkit-scrollbar-track { background: #0D1220; } ::-webkit-scrollbar-thumb { background: #1E3A6E; border-radius: 4px; }
        textarea, input { resize: none; font-family: inherit; }
        @keyframes spin { to { transform: rotate(360deg); } }
        @keyframes fadeUp { from { opacity:0; transform:translateY(12px); } to { opacity:1; transform:translateY(0); } }
        @keyframes slideIn { from { opacity:0; transform:translateX(20px); } to { opacity:1; transform:translateX(0); } }
        @keyframes shimmer { 0%,100%{opacity:0.3} 50%{opacity:0.6} }
        @keyframes toastIn { from{opacity:0;transform:translateY(8px)} to{opacity:1;transform:translateY(0)} }
        .card-hover:hover { border-color: rgba(99,102,241,0.4) !important; background: rgba(255,255,255,0.04) !important; transform: translateY(-1px); }
        .btn-run:hover:not(:disabled) { filter: brightness(1.15); transform: translateY(-1px); }
        .nav-btn:hover { background: rgba(255,255,255,0.06) !important; }
        .history-item:hover { background: rgba(255,255,255,0.04) !important; }
      `}</style>

      {/* ── TOAST ── */}
      {toast && (
        <div style={{
          position: "fixed", bottom: 24, right: 24, zIndex: 1000,
          background: toast.type === "success" ? "#0F2A1A" : "#2A0F0F",
          border: `1px solid ${toast.type === "success" ? "rgba(34,197,94,0.3)" : "rgba(239,68,68,0.3)"}`,
          color: toast.type === "success" ? "#4ADE80" : "#F87171",
          padding: "12px 20px", borderRadius: 10, fontSize: 13, fontWeight: 500,
          animation: "toastIn 0.3s ease",
          display: "flex", alignItems: "center", gap: 8,
        }}>
          {toast.type === "success" ? "✓" : "⚠"} {toast.msg}
        </div>
      )}

      {/* ── API KEY MODAL ── */}
      {showKeyModal && (
        <div style={{
          position: "fixed", inset: 0, zIndex: 500,
          background: "rgba(0,0,0,0.7)", backdropFilter: "blur(8px)",
          display: "flex", alignItems: "center", justifyContent: "center",
        }} onClick={() => setShowKeyModal(false)}>
          <div style={{
            background: "#0D1220", border: "1px solid rgba(255,255,255,0.1)",
            borderRadius: 16, padding: 32, maxWidth: 440, width: "90%",
            animation: "fadeUp 0.3s ease",
          }} onClick={e => e.stopPropagation()}>
            <h3 style={{ fontSize: 17, fontWeight: 700, marginBottom: 8 }}>Anthropic API Key</h3>
            <p style={{ fontSize: 13, color: "#5A6A90", marginBottom: 20, lineHeight: 1.6 }}>
              Your key is stored locally in your browser and never sent to any server other than Anthropic's API.
            </p>
            <input
              type="password"
              value={keyInput}
              onChange={e => setKeyInput(e.target.value)}
              placeholder="sk-ant-..."
              style={{
                width: "100%", padding: "12px 16px",
                background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.1)",
                borderRadius: 10, color: "#E2E8F5", fontSize: 14, marginBottom: 16,
                outline: "none",
              }}
            />
            <div style={{ display: "flex", gap: 8 }}>
              <button onClick={() => setShowKeyModal(false)} style={{
                flex: 1, padding: "11px", background: "none",
                border: "1px solid rgba(255,255,255,0.1)", borderRadius: 10,
                color: "#5A6A90", fontSize: 13, cursor: "pointer",
              }}>Cancel</button>
              <button onClick={saveKey} style={{
                flex: 1, padding: "11px", background: "#4F46E5",
                border: "none", borderRadius: 10, color: "#fff", fontSize: 13,
                fontWeight: 600, cursor: "pointer",
              }}>Save Key</button>
            </div>
          </div>
        </div>
      )}

      {/* ── NAVBAR ── */}
      <nav style={{
        height: 60, borderBottom: "1px solid rgba(255,255,255,0.06)",
        background: "rgba(8,12,20,0.96)", backdropFilter: "blur(20px)",
        display: "flex", alignItems: "center", justifyContent: "space-between",
        padding: "0 28px", position: "sticky", top: 0, zIndex: 100,
        flexShrink: 0,
      }}>
        <div style={{ display: "flex", alignItems: "center", gap: 28 }}>
          <div style={{ display: "flex", alignItems: "center", gap: 9, cursor: "pointer" }} onClick={() => setView("landing")}>
            <div style={{
              width: 30, height: 30, borderRadius: 8,
              background: "linear-gradient(135deg, #4F46E5, #06B6D4)",
              display: "flex", alignItems: "center", justifyContent: "center",
              fontSize: 11, fontWeight: 800, color: "#fff", fontFamily: "JetBrains Mono, monospace",
            }}>FM</div>
            <span style={{ fontWeight: 700, fontSize: 15, letterSpacing: "-0.02em" }}>FlowMind</span>
          </div>
          <div style={{ display: "flex", gap: 4 }}>
            {[["app", "Automations"], ["history", `History${history.length ? ` (${history.length})` : ""}`]].map(([v, label]) => (
              <button key={v} className="nav-btn" onClick={() => setView(v)} style={{
                background: view === v ? "rgba(79,70,229,0.15)" : "none",
                border: view === v ? "1px solid rgba(79,70,229,0.3)" : "1px solid transparent",
                color: view === v ? "#818CF8" : "#5A6A90",
                padding: "6px 14px", borderRadius: 8, fontSize: 13,
                fontWeight: 500, cursor: "pointer", transition: "all 0.2s",
              }}>{label}</button>
            ))}
          </div>
        </div>
        <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
          <div style={{ display: "flex", alignItems: "center", gap: 6 }}>
            <div style={{ width: 6, height: 6, borderRadius: "50%", background: "#22C55E", boxShadow: "0 0 6px #22C55E" }} />
            <span style={{ fontSize: 11, color: "#22C55E", fontFamily: "JetBrains Mono, monospace" }}>LIVE</span>
          </div>
          <button onClick={() => { setKeyInput(apiKey); setShowKeyModal(true); }} style={{
            background: "rgba(255,255,255,0.04)", border: "1px solid rgba(255,255,255,0.08)",
            color: "#5A6A90", padding: "6px 14px", borderRadius: 8, fontSize: 12,
            cursor: "pointer", fontFamily: "inherit",
          }}>⚙ Settings</button>
        </div>
      </nav>

      {/* ── HISTORY VIEW ── */}
      {view === "history" && (
        <div style={{ flex: 1, maxWidth: 860, margin: "0 auto", width: "100%", padding: "36px 24px" }}>
          <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 28 }}>
            <div>
              <h2 style={{ fontSize: 22, fontWeight: 700, letterSpacing: "-0.02em" }}>Run History</h2>
              <p style={{ fontSize: 13, color: "#5A6A90", marginTop: 4 }}>{history.length} automation runs saved locally</p>
            </div>
            {history.length > 0 && (
              <button onClick={clearHistory} style={{
                background: "rgba(239,68,68,0.08)", border: "1px solid rgba(239,68,68,0.2)",
                color: "#F87171", padding: "8px 16px", borderRadius: 8, fontSize: 12,
                cursor: "pointer", fontFamily: "inherit",
              }}>Clear All</button>
            )}
          </div>
          {history.length === 0 ? (
            <div style={{ textAlign: "center", padding: "80px 0", color: "#2A3550" }}>
              <div style={{ fontSize: 40, marginBottom: 12 }}>📭</div>
              <p style={{ fontFamily: "JetBrains Mono, monospace", fontSize: 13 }}>No runs yet. Go run an automation!</p>
            </div>
          ) : (
            <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
              {history.map(h => (
                <div key={h.id} className="history-item" style={{
                  background: "rgba(255,255,255,0.02)", border: "1px solid rgba(255,255,255,0.06)",
                  borderRadius: 12, padding: 20, transition: "all 0.2s", cursor: "default",
                }}>
                  <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 12 }}>
                    <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
                      <span style={{ fontSize: 18 }}>{h.icon}</span>
                      <span style={{ fontWeight: 600, fontSize: 14 }}>{h.automation}</span>
                    </div>
                    <div style={{ display: "flex", gap: 12, alignItems: "center" }}>
                      {h.tokens && <span style={{ fontSize: 11, color: "#2A3A5A", fontFamily: "JetBrains Mono, monospace" }}>{h.tokens} tokens</span>}
                      {h.elapsed && <span style={{ fontSize: 11, color: "#2A3A5A", fontFamily: "JetBrains Mono, monospace" }}>{h.elapsed}s</span>}
                      <span style={{ fontSize: 11, color: "#2A3A5A" }}>{h.date}</span>
                    </div>
                  </div>
                  <div style={{ fontSize: 12, color: "#3A4A6A", marginBottom: 8, fontFamily: "JetBrains Mono, monospace" }}>INPUT</div>
                  <div style={{ fontSize: 13, color: "#7B8DB0", marginBottom: 14, lineHeight: 1.5 }}>{h.input}</div>
                  <div style={{ fontSize: 12, color: "#3A4A6A", marginBottom: 8, fontFamily: "JetBrains Mono, monospace" }}>OUTPUT</div>
                  <div style={{ fontSize: 13, color: "#C8D4F0", lineHeight: 1.65, whiteSpace: "pre-wrap", maxHeight: 120, overflow: "hidden", position: "relative" }}>
                    {h.output.slice(0, 300)}{h.output.length > 300 ? "…" : ""}
                  </div>
                  <button onClick={() => { navigator.clipboard.writeText(h.output); showToast("Copied!"); }} style={{
                    marginTop: 12, background: "none", border: "1px solid rgba(255,255,255,0.07)",
                    color: "#3A4A6A", padding: "6px 12px", borderRadius: 6, fontSize: 11,
                    cursor: "pointer", fontFamily: "inherit",
                  }}>Copy Output</button>
                </div>
              ))}
            </div>
          )}
        </div>
      )}

      {/* ── MAIN APP VIEW ── */}
      {view === "app" && (
        <div style={{ flex: 1, maxWidth: 1160, margin: "0 auto", width: "100%", padding: "36px 24px 80px" }}>

          {/* Header */}
          <div style={{ marginBottom: 32 }}>
            <h1 style={{ fontSize: "clamp(1.6rem,3.5vw,2.2rem)", fontWeight: 800, letterSpacing: "-0.03em", lineHeight: 1.15, marginBottom: 8 }}>
              AI Automation Suite
            </h1>
            <p style={{ color: "#4A5880", fontSize: 14 }}>Select a tool, paste your content, and get results in seconds.</p>
          </div>

          {/* Category Filter */}
          <div style={{ display: "flex", gap: 6, marginBottom: 20, flexWrap: "wrap" }}>
            {CATEGORIES.map(c => (
              <button key={c} onClick={() => setFilterCat(c)} style={{
                background: filterCat === c ? "rgba(79,70,229,0.18)" : "rgba(255,255,255,0.03)",
                border: filterCat === c ? "1px solid rgba(79,70,229,0.4)" : "1px solid rgba(255,255,255,0.07)",
                color: filterCat === c ? "#818CF8" : "#5A6A90",
                padding: "6px 14px", borderRadius: 100, fontSize: 12, fontWeight: 500,
                cursor: "pointer", transition: "all 0.2s",
              }}>{c}</button>
            ))}
          </div>

          {/* Automation Grid */}
          <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(200px, 1fr))", gap: 10, marginBottom: 32 }}>
            {filtered.map(a => (
              <div key={a.id} className="card-hover" onClick={() => { setSelected(a); reset(); }} style={{
                background: selected.id === a.id ? `${a.color}14` : "rgba(255,255,255,0.025)",
                border: selected.id === a.id ? `1px solid ${a.color}55` : "1px solid rgba(255,255,255,0.06)",
                borderRadius: 14, padding: "16px", cursor: "pointer",
                transition: "all 0.2s",
              }}>
                <div style={{ fontSize: 22, marginBottom: 10 }}>{a.icon}</div>
                <div style={{ fontSize: 12, color: a.color, fontWeight: 600, marginBottom: 3 }}>{a.label}</div>
                <div style={{ fontSize: 10, color: "#3A4A6A", fontFamily: "JetBrains Mono, monospace", marginBottom: 6, letterSpacing: "0.04em" }}>{a.category.toUpperCase()}</div>
                <div style={{ fontSize: 11.5, color: "#4A5880", lineHeight: 1.5 }}>{a.description}</div>
              </div>
            ))}
          </div>

          {/* Workspace */}
          <div style={{
            background: "rgba(255,255,255,0.02)", border: "1px solid rgba(255,255,255,0.07)",
            borderRadius: 18, overflow: "hidden",
          }}>
            {/* Workspace Header */}
            <div style={{
              padding: "16px 24px", borderBottom: "1px solid rgba(255,255,255,0.06)",
              display: "flex", alignItems: "center", justifyContent: "space-between",
              background: "rgba(255,255,255,0.02)",
            }}>
              <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
                <div style={{
                  width: 36, height: 36, borderRadius: 10,
                  background: `${selected.color}18`, border: `1px solid ${selected.color}30`,
                  display: "flex", alignItems: "center", justifyContent: "center", fontSize: 18,
                }}>{selected.icon}</div>
                <div>
                  <div style={{ fontWeight: 700, fontSize: 14 }}>{selected.label}</div>
                  <div style={{ fontSize: 11, color: "#4A5880" }}>{selected.description}</div>
                </div>
              </div>
              <span style={{ fontSize: 10, color: "#2A3550", fontFamily: "JetBrains Mono, monospace", letterSpacing: "0.08em" }}>WORKSPACE</span>
            </div>

            <div style={{ display: "grid", gridTemplateColumns: ran ? "1fr 1fr" : "1fr", gap: 0 }}>
              {/* Input */}
              <div style={{ borderRight: ran ? "1px solid rgba(255,255,255,0.06)" : "none" }}>
                <div style={{ padding: "8px 24px 0", display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                  <span style={{ fontSize: 10, color: "#2A3550", fontFamily: "JetBrains Mono, monospace", letterSpacing: "0.08em" }}>INPUT</span>
                  <span style={{ fontSize: 11, color: "#2A3550", fontFamily: "JetBrains Mono, monospace" }}>{input.length} chars</span>
                </div>
                <textarea
                  value={input} onChange={e => setInput(e.target.value)}
                  placeholder={selected.placeholder}
                  style={{
                    width: "100%", minHeight: 220, background: "transparent",
                    border: "none", outline: "none", color: "#C8D4F0",
                    fontSize: 13.5, lineHeight: 1.75, padding: "12px 24px 20px",
                  }}
                />
              </div>

              {/* Output */}
              {ran && (
                <div ref={outputRef} style={{ animation: "slideIn 0.35s ease" }}>
                  <div style={{ padding: "8px 24px 0", display: "flex", justifyContent: "space-between", alignItems: "center" }}>
                    <div style={{ display: "flex", alignItems: "center", gap: 6 }}>
                      <div style={{ width: 6, height: 6, borderRadius: "50%", background: "#22C55E", boxShadow: "0 0 5px #22C55E" }} />
                      <span style={{ fontSize: 10, color: "#22C55E", fontFamily: "JetBrains Mono, monospace", letterSpacing: "0.08em" }}>OUTPUT</span>
                    </div>
                    <div style={{ display: "flex", gap: 10 }}>
                      {elapsed && <span style={{ fontSize: 10, color: "#2A3550", fontFamily: "JetBrains Mono, monospace" }}>{elapsed}s</span>}
                      {tokens && <span style={{ fontSize: 10, color: "#2A3550", fontFamily: "JetBrains Mono, monospace" }}>{tokens}t</span>}
                    </div>
                  </div>
                  <div style={{
                    minHeight: 220, maxHeight: 400, overflowY: "auto",
                    padding: "12px 24px 20px", fontSize: 13.5, lineHeight: 1.75,
                    color: "#C8D4F0", whiteSpace: "pre-wrap",
                  }}>{output}</div>
                </div>
              )}

              {/* Loading shimmer */}
              {loading && (
                <div style={{ padding: "20px 24px" }}>
                  {[90, 75, 85, 60, 70].map((w, i) => (
                    <div key={i} style={{
                      height: 12, width: `${w}%`, borderRadius: 6, marginBottom: 12,
                      background: "#1A2540", animation: `shimmer 1.5s ease ${i * 0.12}s infinite`,
                    }} />
                  ))}
                </div>
              )}
            </div>

            {/* Action Bar */}
            <div style={{
              padding: "14px 24px", borderTop: "1px solid rgba(255,255,255,0.06)",
              display: "flex", alignItems: "center", justifyContent: "space-between",
              background: "rgba(255,255,255,0.015)",
            }}>
              <div style={{ display: "flex", gap: 8 }}>
                {ran && (
                  <>
                    <button onClick={handleCopy} style={{
                      background: copied ? "rgba(34,197,94,0.1)" : "rgba(255,255,255,0.04)",
                      border: `1px solid ${copied ? "rgba(34,197,94,0.3)" : "rgba(255,255,255,0.08)"}`,
                      color: copied ? "#4ADE80" : "#5A6A90",
                      padding: "8px 16px", borderRadius: 8, fontSize: 12,
                      cursor: "pointer", fontFamily: "inherit", transition: "all 0.2s",
                    }}>{copied ? "✓ Copied" : "Copy"}</button>
                    <button onClick={() => { setRan(false); runAutomation(); }} style={{
                      background: "rgba(255,255,255,0.03)", border: "1px solid rgba(255,255,255,0.08)",
                      color: "#5A6A90", padding: "8px 16px", borderRadius: 8, fontSize: 12,
                      cursor: "pointer", fontFamily: "inherit",
                    }}>↺ Re-run</button>
                    <button onClick={reset} style={{
                      background: "none", border: "1px solid rgba(255,255,255,0.06)",
                      color: "#3A4A6A", padding: "8px 16px", borderRadius: 8, fontSize: 12,
                      cursor: "pointer", fontFamily: "inherit",
                    }}>Reset</button>
                  </>
                )}
              </div>
              <button
                className="btn-run"
                onClick={runAutomation}
                disabled={!input.trim() || loading}
                style={{
                  background: !input.trim() || loading ? "rgba(79,70,229,0.2)" : "linear-gradient(135deg, #4F46E5, #6366F1)",
                  border: "none", color: !input.trim() || loading ? "#3A3A7A" : "#fff",
                  padding: "10px 28px", borderRadius: 10, fontSize: 13, fontWeight: 700,
                  cursor: !input.trim() || loading ? "not-allowed" : "pointer",
                  fontFamily: "inherit", transition: "all 0.2s",
                  display: "flex", alignItems: "center", gap: 8,
                  boxShadow: input.trim() && !loading ? "0 4px 24px rgba(79,70,229,0.35)" : "none",
                }}
              >
                {loading ? (
                  <>
                    <div style={{ width: 13, height: 13, border: "2px solid rgba(99,102,241,0.3)", borderTop: "2px solid #818CF8", borderRadius: "50%", animation: "spin 0.8s linear infinite" }} />
                    Running…
                  </>
                ) : "▶  Run Automation"}
              </button>
            </div>
          </div>

          {/* Stats Bar */}
          {history.length > 0 && (
            <div style={{ display: "flex", gap: 20, marginTop: 20, padding: "16px 20px", background: "rgba(255,255,255,0.015)", border: "1px solid rgba(255,255,255,0.05)", borderRadius: 12 }}>
              {[
                ["Total Runs", history.length],
                ["Automations Used", new Set(history.map(h => h.automation)).size],
                ["Tokens Used", history.reduce((s, h) => s + (h.tokens || 0), 0).toLocaleString()],
                ["Avg. Response", (history.reduce((s, h) => s + parseFloat(h.elapsed || 0), 0) / history.length).toFixed(1) + "s"],
              ].map(([label, val]) => (
                <div key={label} style={{ flex: 1, textAlign: "center" }}>
                  <div style={{ fontSize: 18, fontWeight: 700, letterSpacing: "-0.03em", color: "#818CF8" }}>{val}</div>
                  <div style={{ fontSize: 10, color: "#2A3550", marginTop: 2, fontFamily: "JetBrains Mono, monospace", letterSpacing: "0.05em" }}>{label.toUpperCase()}</div>
                </div>
              ))}
            </div>
          )}
        </div>
      )}

      {/* Footer */}
      <footer style={{ borderTop: "1px solid rgba(255,255,255,0.05)", padding: "20px 28px", display: "flex", alignItems: "center", justifyContent: "space-between", flexShrink: 0 }}>
        <div style={{ display: "flex", alignItems: "center", gap: 8 }}>
          <div style={{ width: 20, height: 20, borderRadius: 5, background: "linear-gradient(135deg, #4F46E5, #06B6D4)", display: "flex", alignItems: "center", justifyContent: "center", fontSize: 8, fontWeight: 800, color: "#fff" }}>FM</div>
          <span style={{ fontSize: 12, color: "#2A3550", fontWeight: 600 }}>FlowMind</span>
        </div>
        <span style={{ fontSize: 11, color: "#1A2540", fontFamily: "JetBrains Mono, monospace" }}>Powered by Claude AI · © 2026</span>
        <div style={{ display: "flex", gap: 16 }}>
          {["Privacy", "Terms", "Docs"].map(l => (
            <span key={l} style={{ fontSize: 11, color: "#2A3550", cursor: "pointer" }}>{l}</span>
          ))}
        </div>
      </footer>
    </div>
  );
}

// ─── Landing Page ─────────────────────────────────────────────────────────────

function Landing({ onEnter }) {
  return (
    <div style={{ minHeight: "100vh", background: "#080C14", color: "#E2E8F5", fontFamily: "'Plus Jakarta Sans', sans-serif", display: "flex", flexDirection: "column", alignItems: "center", justifyContent: "center", padding: "40px 24px", textAlign: "center", position: "relative", overflow: "hidden" }}>
      <style>{`@import url('https://fonts.googleapis.com/css2?family=Plus+Jakarta+Sans:wght@400;600;700;800&family=JetBrains+Mono:wght@400&display=swap');`}</style>

      {/* Glow */}
      <div style={{ position: "absolute", top: "20%", left: "50%", transform: "translateX(-50%)", width: 700, height: 500, background: "radial-gradient(ellipse, rgba(79,70,229,0.12) 0%, transparent 70%)", pointerEvents: "none" }} />
      <div style={{ position: "absolute", bottom: "10%", right: "10%", width: 400, height: 400, background: "radial-gradient(ellipse, rgba(6,182,212,0.07) 0%, transparent 70%)", pointerEvents: "none" }} />

      <div style={{ position: "relative", zIndex: 1, maxWidth: 680, animation: "fadeUp 0.7s ease" }}>
        <div style={{
          display: "inline-flex", alignItems: "center", gap: 6,
          background: "rgba(79,70,229,0.12)", border: "1px solid rgba(79,70,229,0.3)",
          color: "#818CF8", padding: "6px 16px", borderRadius: 100, fontSize: 11,
          fontFamily: "JetBrains Mono, monospace", letterSpacing: "0.06em", marginBottom: 28,
        }}>
          <span style={{ width: 6, height: 6, borderRadius: "50%", background: "#22C55E", display: "inline-block" }} />
          POWERED BY CLAUDE AI
        </div>

        <h1 style={{ fontSize: "clamp(2.4rem, 6vw, 4rem)", fontWeight: 800, letterSpacing: "-0.04em", lineHeight: 1.08, marginBottom: 20 }}>
          Automate your work<br />
          <span style={{ background: "linear-gradient(135deg, #818CF8, #06B6D4)", WebkitBackgroundClip: "text", WebkitTextFillColor: "transparent" }}>with AI in seconds.</span>
        </h1>

        <p style={{ fontSize: 16, color: "#4A5880", lineHeight: 1.7, maxWidth: 500, margin: "0 auto 40px" }}>
          8 professional AI automations — email replies, meeting notes, lead scoring, report writing, and more. No setup. No code. Just results.
        </p>

        <div style={{ display: "flex", gap: 10, justifyContent: "center", flexWrap: "wrap", marginBottom: 56 }}>
          <button onClick={onEnter} style={{
            background: "linear-gradient(135deg, #4F46E5, #6366F1)", border: "none",
            color: "#fff", padding: "14px 36px", borderRadius: 12, fontSize: 15,
            fontWeight: 700, cursor: "pointer", fontFamily: "inherit",
            boxShadow: "0 8px 32px rgba(79,70,229,0.4)", transition: "all 0.2s",
          }}>
            Launch App — Free →
          </button>
        </div>

        <div style={{ display: "flex", gap: 32, justifyContent: "center", flexWrap: "wrap" }}>
          {[["8", "AI Automations"], ["< 3s", "Avg Response"], ["100%", "Browser-based"]].map(([num, label]) => (
            <div key={label} style={{ textAlign: "center" }}>
              <div style={{ fontSize: 24, fontWeight: 800, letterSpacing: "-0.04em", color: "#818CF8" }}>{num}</div>
              <div style={{ fontSize: 11, color: "#2A3550", marginTop: 2, fontFamily: "JetBrains Mono, monospace", letterSpacing: "0.05em" }}>{label.toUpperCase()}</div>
            </div>
          ))}
        </div>
      </div>
    </div>
  );
}
