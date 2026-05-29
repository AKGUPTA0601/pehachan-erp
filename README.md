import React, { useState, useEffect, useMemo, createContext, useContext, useCallback } from "react";
import {
  LayoutDashboard, Users, Store, Tags, ScrollText, Receipt, BarChart3, Plus, Search,
  Pencil, Trash2, X, Check, IndianRupee, TrendingUp, AlertCircle, Download, ArrowRight,
  Eye, CircleDot, Menu, Briefcase, Wallet, Clock, ChevronDown, Printer, Calendar, Megaphone
} from "lucide-react";
import {
  BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, PieChart, Pie, Cell, CartesianGrid
} from "recharts";

/* ============================================================================
   PEHACHAN ERP — Print Advertising Agency
   Workflow: Release Orders → Vendors → Clients → Rate Cards → Invoices → Reports
   Pricing: Height × Width (cm) × rate per sq.cm × number of release dates (insertions)
============================================================================ */

const PFX = "perp:"; // storage namespace (new model)

/* ---------------------------------- THEME --------------------------------- */
const T = {
  paper: "#F7F4EE", card: "#FFFFFF", ink: "#211D1A", inkSoft: "#6B635B",
  line: "#E6DFD4", accent: "#C8472B", accentSoft: "#F4E2DC",
  green: "#2E7D5B", greenSoft: "#E1F0E8", amber: "#B07A14", amberSoft: "#F6ECD6",
  blue: "#2B6CB0", blueSoft: "#E2ECF6", red: "#C0392B", redSoft: "#F7E2DF",
  gray: "#8A817A", graySoft: "#ECE7DF", sidebar: "#211D1A", sidebarSoft: "#3A332E",
};
const STATUS = {
  Draft: { c: T.gray, b: T.graySoft }, Issued: { c: T.blue, b: T.blueSoft },
  "In Progress": { c: T.amber, b: T.amberSoft }, Completed: { c: T.green, b: T.greenSoft },
  Sent: { c: T.blue, b: T.blueSoft }, "Partially Paid": { c: T.amber, b: T.amberSoft },
  Paid: { c: T.green, b: T.greenSoft }, Overdue: { c: T.red, b: T.redSoft },
};

/* ------------------------------- STORAGE LAYER ----------------------------- */
const mem = {};
async function sget(key) {
  try { if (typeof window !== "undefined" && window.storage) { const r = await window.storage.get(key); return r ? JSON.parse(r.value) : undefined; } } catch (e) {}
  return mem[key];
}
async function sset(key, val) {
  mem[key] = val;
  try { if (typeof window !== "undefined" && window.storage) await window.storage.set(key, JSON.stringify(val)); } catch (e) {}
}

/* --------------------------------- HELPERS --------------------------------- */
const inr = (n) => "₹" + new Intl.NumberFormat("en-IN", { maximumFractionDigits: 0 }).format(Math.round(n || 0));
const inr2 = (n) => "₹" + new Intl.NumberFormat("en-IN", { minimumFractionDigits: 2, maximumFractionDigits: 2 }).format(n || 0);
const today = () => new Date().toISOString().slice(0, 10);
const fmtDate = (d) => d ? new Date(d).toLocaleDateString("en-IN", { day: "2-digit", month: "short", year: "numeric" }) : "—";
const fmtShort = (d) => d ? new Date(d).toLocaleDateString("en-IN", { day: "2-digit", month: "short" }) : "—";
const uid = () => Date.now().toString(36) + Math.random().toString(36).slice(2, 7);

function nextId(prefix, arr) {
  let max = 0;
  (arr || []).forEach((x) => { const m = String(x.id || "").match(new RegExp("^" + prefix + "-(\\d+)$")); if (m) max = Math.max(max, parseInt(m[1], 10)); });
  return prefix + "-" + String(max + 1).padStart(4, "0");
}

const insertions = (l) => (l.releaseDates && l.releaseDates.length) || 1;
const lineArea = (l) => (Number(l.height) || 0) * (Number(l.width) || 0);
const lineAmount = (l) => lineArea(l) * (Number(l.rate) || 0) * insertions(l);

function computeTotals(lines, discountPct, gstPct) {
  const subtotal = (lines || []).reduce((s, l) => s + lineAmount(l), 0);
  const discount = subtotal * (Number(discountPct) || 0) / 100;
  const taxable = subtotal - discount;
  const gst = taxable * (Number(gstPct) || 0) / 100;
  return { subtotal, discount, taxable, gst, total: taxable + gst };
}

function downloadCSV(filename, rows) {
  if (!rows.length) return;
  const headers = Object.keys(rows[0]);
  const csv = [headers.join(",")].concat(rows.map((r) => headers.map((h) => `"${String(r[h] ?? "").replace(/"/g, '""')}"`).join(","))).join("\n");
  const a = document.createElement("a"); a.href = URL.createObjectURL(new Blob([csv], { type: "text/csv" })); a.download = filename; a.click();
}

/* ---------------------------------- SEED ----------------------------------- */
function seed() {
  const clients = [
    { id: "CL-0001", _uid: uid(), name: "Western Railway — PR Cell", contact: "Sr. PRO", email: "pro@wr.railnet.gov.in", phone: "022-22034567", gstin: "27AAAGM0289C1Z5", category: "Government", city: "Mumbai", credit: 5000000, terms: "45 days", address: "Churchgate, Mumbai 400020" },
    { id: "CL-0002", _uid: uid(), name: "Tulip Infratech Pvt. Ltd.", contact: "Marketing Head", email: "marketing@tulipinfra.com", phone: "0124-4567890", gstin: "06AABCT1234D1Z2", category: "Real Estate", city: "Gurugram", credit: 2500000, terms: "30 days", address: "Sector 70, Gurugram 122101" },
    { id: "CL-0003", _uid: uid(), name: "Credon Realtors LLP", contact: "Brand Manager", email: "brand@credon.in", phone: "0124-4001122", gstin: "06AALFC5678E1Z9", category: "Real Estate", city: "Gurugram", credit: 1500000, terms: "30 days", address: "Golf Course Road, Gurugram 122002" },
  ];
  const vendors = [
    { id: "VN-0001", _uid: uid(), name: "Dainik Jagran", type: "Newspaper", contact: "Ad Booking Desk", phone: "0120-3988000", locations: "Delhi, NCR, UP, Bihar", terms: "Net 30", commission: 15 },
    { id: "VN-0002", _uid: uid(), name: "Hindustan Times", type: "Newspaper", contact: "HT Media Ads", phone: "011-66561234", locations: "Delhi, Mumbai, Lucknow", terms: "Net 30", commission: 15 },
    { id: "VN-0003", _uid: uid(), name: "Navbharat Times", type: "Newspaper", contact: "TOI Group Ads", phone: "011-23492345", locations: "Delhi, NCR", terms: "Net 30", commission: 12 },
    { id: "VN-0004", _uid: uid(), name: "Amar Ujala", type: "Newspaper", contact: "Ad Desk", phone: "0120-3911111", locations: "UP, Uttarakhand, NCR", terms: "Net 30", commission: 12 },
  ];
  const rates = [
    { id: "RT-0001", _uid: uid(), vendor: "Dainik Jagran", publication: "Delhi Edition", color: "B&W", rate: 850 },
    { id: "RT-0002", _uid: uid(), vendor: "Dainik Jagran", publication: "Delhi Edition", color: "Color", rate: 1450 },
    { id: "RT-0003", _uid: uid(), vendor: "Hindustan Times", publication: "Delhi — Front Page", color: "Color", rate: 4200 },
    { id: "RT-0004", _uid: uid(), vendor: "Hindustan Times", publication: "Delhi — Inside Page", color: "B&W", rate: 1100 },
    { id: "RT-0005", _uid: uid(), vendor: "Navbharat Times", publication: "Delhi NCR", color: "Color", rate: 1250 },
    { id: "RT-0006", _uid: uid(), vendor: "Navbharat Times", publication: "Delhi NCR", color: "B&W", rate: 780 },
    { id: "RT-0007", _uid: uid(), vendor: "Amar Ujala", publication: "NCR Edition", color: "Color", rate: 980 },
  ];
  const roLines = [
    { _uid: uid(), rateId: "RT-0002", desc: "Dainik Jagran — Delhi Edition", color: "Color", height: 25, width: 16, rate: 1450, releaseDates: ["2026-06-05", "2026-06-12", "2026-06-19"] },
    { _uid: uid(), rateId: "RT-0004", desc: "Hindustan Times — Delhi Inside Page", color: "B&W", height: 20, width: 13, rate: 1100, releaseDates: ["2026-06-06", "2026-06-13"] },
  ];
  const releaseOrders = [
    { id: "RO-0001", _uid: uid(), clientId: "CL-0002", vendor: "Dainik Jagran", project: "Tulip GCR Launch Campaign", lines: roLines, discountPct: 5, gstPct: 18, status: "In Progress" },
    { id: "RO-0002", _uid: uid(), clientId: "CL-0001", vendor: "Hindustan Times", project: "WR Safety Awareness Drive", discountPct: 0, gstPct: 18, status: "Issued",
      lines: [{ _uid: uid(), rateId: "RT-0003", desc: "Hindustan Times — Front Page", color: "Color", height: 12, width: 25, rate: 4200, releaseDates: ["2026-05-20"] }] },
    { id: "RO-0003", _uid: uid(), clientId: "CL-0003", vendor: "Navbharat Times", project: "Westin Residences Festive Offer", discountPct: 8, gstPct: 18, status: "Draft",
      lines: [{ _uid: uid(), rateId: "RT-0005", desc: "Navbharat Times — Delhi NCR", color: "Color", height: 18, width: 16, rate: 1250, releaseDates: ["2026-06-01", "2026-06-08"] }] },
  ];
  const invoices = [
    { id: "INV-0001", _uid: uid(), roId: "RO-0001", clientId: "CL-0002", date: "2026-06-20", dueDate: "2026-07-20", lines: roLines, discountPct: 5, gstPct: 18, status: "Partially Paid", paid: 1500000 },
  ];
  return { clients, vendors, rates, releaseOrders, invoices, settings: { gstPct: 18, company: "Pehachan Advt. & Marketing Pvt. Ltd.", version: 2 } };
}
const COLLECTIONS = ["clients", "vendors", "rates", "releaseOrders", "invoices", "settings"];

/* -------------------------------- DB CONTEXT ------------------------------- */
const ERPCtx = createContext(null);
const useDB = () => useContext(ERPCtx);

/* ============================== UI PRIMITIVES ============================== */
function Badge({ status }) {
  const s = STATUS[status] || { c: T.gray, b: T.graySoft };
  return <span className="inline-flex items-center gap-1" style={{ background: s.b, color: s.c, fontSize: 11, fontWeight: 600, padding: "3px 9px", borderRadius: 20, whiteSpace: "nowrap" }}><CircleDot size={9} /> {status}</span>;
}
function ColorPill({ c }) {
  const on = c === "Color";
  return <span style={{ background: on ? T.accentSoft : T.graySoft, color: on ? T.accent : T.ink, fontSize: 10.5, fontWeight: 700, padding: "2px 7px", borderRadius: 5, letterSpacing: 0.3 }}>{on ? "COLOUR" : "B&W"}</span>;
}
function Btn({ children, onClick, variant = "primary", size = "md", icon: Icon, type, style, title }) {
  const base = { display: "inline-flex", alignItems: "center", gap: 7, fontWeight: 600, borderRadius: 9, cursor: "pointer", border: "1px solid transparent", transition: "all .15s", fontSize: size === "sm" ? 12.5 : 13.5, padding: size === "sm" ? "6px 11px" : "9px 15px", whiteSpace: "nowrap", fontFamily: "inherit" };
  const v = { primary: { background: T.accent, color: "#fff" }, dark: { background: T.ink, color: T.paper }, ghost: { background: "transparent", color: T.ink, border: `1px solid ${T.line}` }, soft: { background: T.accentSoft, color: T.accent }, danger: { background: T.redSoft, color: T.red } };
  return <button title={title} onClick={onClick} type={type || "button"} style={{ ...base, ...v[variant], ...style }} onMouseEnter={(e) => (e.currentTarget.style.opacity = "0.88")} onMouseLeave={(e) => (e.currentTarget.style.opacity = "1")}>{Icon && <Icon size={size === "sm" ? 14 : 16} />} {children}</button>;
}
function Field({ label, children, span }) {
  return <label style={{ display: "flex", flexDirection: "column", gap: 5, gridColumn: span ? `span ${span}` : "auto" }}><span style={{ fontSize: 12, fontWeight: 600, color: T.inkSoft }}>{label}</span>{children}</label>;
}
const inputStyle = { padding: "9px 11px", borderRadius: 8, border: `1px solid ${T.line}`, fontSize: 13.5, background: "#fff", color: T.ink, outline: "none", width: "100%", fontFamily: "inherit" };
const TextInput = (p) => <input {...p} style={{ ...inputStyle, ...(p.style || {}) }} />;
const NumInput = (p) => <input type="number" {...p} style={{ ...inputStyle, ...(p.style || {}) }} />;
const TextArea = (p) => <textarea {...p} style={{ ...inputStyle, minHeight: 70, resize: "vertical", ...(p.style || {}) }} />;
function SelectInput({ value, onChange, options, placeholder }) {
  return <div style={{ position: "relative" }}>
    <select value={value} onChange={onChange} style={{ ...inputStyle, appearance: "none", paddingRight: 30 }}>
      <option value="">{placeholder || "Select…"}</option>
      {options.map((o) => <option key={o.value} value={o.value}>{o.label}</option>)}
    </select>
    <ChevronDown size={15} style={{ position: "absolute", right: 10, top: 11, color: T.inkSoft, pointerEvents: "none" }} />
  </div>;
}
function Modal({ open, onClose, title, children, width = 720 }) {
  if (!open) return null;
  return <div onClick={onClose} style={{ position: "fixed", inset: 0, background: "rgba(33,29,26,0.45)", zIndex: 50, display: "flex", alignItems: "flex-start", justifyContent: "center", padding: "40px 16px", overflowY: "auto" }}>
    <div onClick={(e) => e.stopPropagation()} style={{ background: T.card, borderRadius: 16, width: "100%", maxWidth: width, boxShadow: "0 24px 60px rgba(33,29,26,0.25)", overflow: "hidden" }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "18px 22px", borderBottom: `1px solid ${T.line}` }}>
        <h3 style={{ fontFamily: "Fraunces, serif", fontSize: 20, fontWeight: 600, color: T.ink, margin: 0 }}>{title}</h3>
        <button onClick={onClose} style={{ background: T.paper, border: "none", borderRadius: 8, padding: 7, cursor: "pointer", color: T.inkSoft }}><X size={18} /></button>
      </div>
      <div style={{ padding: 22, maxHeight: "72vh", overflowY: "auto" }}>{children}</div>
    </div>
  </div>;
}
const Card = ({ children, style }) => <div style={{ background: T.card, border: `1px solid ${T.line}`, borderRadius: 14, ...style }}>{children}</div>;
function Table({ columns, rows, empty }) {
  return <div style={{ overflowX: "auto" }}><table style={{ width: "100%", borderCollapse: "collapse", fontSize: 13.5 }}>
    <thead><tr>{columns.map((c) => <th key={c.key} style={{ textAlign: c.align || "left", padding: "12px 14px", fontSize: 11.5, fontWeight: 700, color: T.inkSoft, letterSpacing: 0.5, textTransform: "uppercase", borderBottom: `1.5px solid ${T.line}`, whiteSpace: "nowrap" }}>{c.label}</th>)}</tr></thead>
    <tbody>{rows.length === 0 ? <tr><td colSpan={columns.length} style={{ padding: 40, textAlign: "center", color: T.inkSoft }}>{empty || "No records yet."}</td></tr> : rows.map((r, i) => <tr key={r._uid || i} style={{ borderBottom: `1px solid ${T.line}` }} onMouseEnter={(e) => (e.currentTarget.style.background = T.paper)} onMouseLeave={(e) => (e.currentTarget.style.background = "transparent")}>{columns.map((c) => <td key={c.key} style={{ padding: "12px 14px", textAlign: c.align || "left", color: T.ink, verticalAlign: "middle" }}>{c.render ? c.render(r) : r[c.key]}</td>)}</tr>)}</tbody>
  </table></div>;
}
function PageHead({ title, subtitle, action }) {
  return <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-end", marginBottom: 22, flexWrap: "wrap", gap: 12 }}>
    <div><h1 style={{ fontFamily: "Fraunces, serif", fontSize: 30, fontWeight: 600, color: T.ink, margin: 0, lineHeight: 1.1 }}>{title}</h1>{subtitle && <p style={{ color: T.inkSoft, margin: "6px 0 0", fontSize: 14 }}>{subtitle}</p>}</div>
    {action}
  </div>;
}
function SearchBar({ value, onChange }) {
  return <div style={{ position: "relative", maxWidth: 320, flex: 1 }}><Search size={16} style={{ position: "absolute", left: 11, top: 10, color: T.inkSoft }} /><input value={value} onChange={(e) => onChange(e.target.value)} placeholder="Search…" style={{ ...inputStyle, paddingLeft: 34 }} /></div>;
}
function RowActions({ onView, onEdit, onDelete }) {
  const ab = { background: T.paper, border: `1px solid ${T.line}`, borderRadius: 7, padding: 6, cursor: "pointer", color: T.inkSoft, display: "inline-flex" };
  return <div style={{ display: "flex", gap: 6, justifyContent: "flex-end" }}>{onView && <button title="View" style={ab} onClick={onView}><Eye size={15} /></button>}{onEdit && <button title="Edit" style={ab} onClick={onEdit}><Pencil size={15} /></button>}{onDelete && <button title="Delete" style={{ ...ab, color: T.red }} onClick={onDelete}><Trash2 size={15} /></button>}</div>;
}
const grid2 = { display: "grid", gridTemplateColumns: "1fr 1fr", gap: 14 };
const mono = { fontFamily: "ui-monospace, monospace", fontSize: 12.5, fontWeight: 600, color: T.accent };
const chartTitle = { fontFamily: "Fraunces, serif", fontSize: 17, fontWeight: 600, color: T.ink, margin: "0 0 12px" };
const tooltipStyle = { borderRadius: 10, border: `1px solid ${T.line}`, fontSize: 13, boxShadow: "0 8px 24px rgba(0,0,0,.1)" };
function ModalFooter({ onCancel, onSave, saveLabel }) {
  return <div style={{ display: "flex", justifyContent: "flex-end", gap: 10, marginTop: 22, paddingTop: 16, borderTop: `1px solid ${T.line}` }}><Btn variant="ghost" onClick={onCancel}>Cancel</Btn><Btn icon={Check} onClick={onSave}>{saveLabel || "Save"}</Btn></div>;
}
function TotalsBlock({ totals }) {
  const Row = ({ l, v, bold }) => <div style={{ display: "flex", justifyContent: "space-between", padding: "5px 0", fontSize: bold ? 15 : 13.5, fontWeight: bold ? 700 : 500, color: bold ? T.ink : T.inkSoft }}><span>{l}</span><span style={{ fontVariantNumeric: "tabular-nums" }}>{v}</span></div>;
  return <div style={{ marginLeft: "auto", width: 280, marginTop: 14 }}>
    <Row l="Subtotal" v={inr2(totals.subtotal)} /><Row l="Discount" v={"− " + inr2(totals.discount)} /><Row l="Taxable" v={inr2(totals.taxable)} /><Row l="GST" v={inr2(totals.gst)} />
    <div style={{ borderTop: `1.5px solid ${T.ink}`, marginTop: 4 }}><Row l="Grand Total" v={inr2(totals.total)} bold /></div>
  </div>;
}

/* ====================== LINE ITEM EDITOR (area-based) ====================== */
function LineCard({ line, index, update, remove, rates, vendor, onSaveRate }) {
  const [newDate, setNewDate] = useState(today());
  const [saved, setSaved] = useState(false);
  const area = lineArea(line);
  const ins = insertions(line);
  const amount = lineAmount(line);
  const pick = (rateId) => { const r = rates.find((x) => x.id === rateId); if (r) update({ rateId, desc: `${r.vendor} — ${r.publication}`, color: r.color, rate: r.rate }); else update({ rateId }); };
  const addDate = () => { if (newDate && !(line.releaseDates || []).includes(newDate)) update({ releaseDates: [...(line.releaseDates || []), newDate].sort() }); };
  const rmDate = (d) => update({ releaseDates: (line.releaseDates || []).filter((x) => x !== d) });
  const saveRate = () => { onSaveRate(line); setSaved(true); setTimeout(() => setSaved(false), 1600); };

  return (
    <div style={{ border: `1px solid ${T.line}`, borderRadius: 11, padding: 14, marginBottom: 11, background: T.paper }}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: 10 }}>
        <span style={{ fontSize: 12, fontWeight: 700, color: T.inkSoft, letterSpacing: 0.4 }}>LINE {index + 1}</span>
        <button onClick={remove} style={{ background: T.redSoft, color: T.red, border: "none", borderRadius: 7, padding: 6, cursor: "pointer", display: "inline-flex" }}><Trash2 size={14} /></button>
      </div>
      <div style={grid2}>
        <Field label="Pick from Rate Card (optional)"><SelectInput value={line.rateId || ""} placeholder="— Custom / type below —" onChange={(e) => pick(e.target.value)} options={rates.map((r) => ({ value: r.id, label: `${r.vendor} · ${r.publication} · ${r.color} · ${inr(r.rate)}/cm²` }))} /></Field>
        <Field label="Colour"><SelectInput value={line.color || ""} placeholder="Choose…" onChange={(e) => update({ color: e.target.value })} options={[{ value: "Color", label: "Colour" }, { value: "B&W", label: "Black & White" }]} /></Field>
      </div>
      <div style={{ marginTop: 12 }}><Field label="Publication / Description"><TextInput value={line.desc || ""} onChange={(e) => update({ desc: e.target.value })} placeholder="e.g. Dainik Jagran — Delhi Edition, Page 3" /></Field></div>
      <div style={{ display: "grid", gridTemplateColumns: "1fr 1fr 1fr 1.4fr", gap: 10, marginTop: 12, alignItems: "end" }}>
        <Field label="Height (cm)"><NumInput value={line.height ?? ""} onChange={(e) => update({ height: Number(e.target.value) })} /></Field>
        <Field label="Width (cm)"><NumInput value={line.width ?? ""} onChange={(e) => update({ width: Number(e.target.value) })} /></Field>
        <Field label="Area (cm²)"><div style={{ ...inputStyle, background: T.graySoft, fontWeight: 600, fontVariantNumeric: "tabular-nums" }}>{area.toLocaleString("en-IN")}</div></Field>
        <Field label="Rate (₹ / cm²) — editable">
          <div style={{ display: "flex", gap: 6 }}>
            <NumInput value={line.rate ?? ""} onChange={(e) => update({ rate: Number(e.target.value) })} style={{ textAlign: "right" }} />
            <button onClick={saveRate} title="Add this rate to the Rate Card" style={{ background: saved ? T.greenSoft : T.accentSoft, color: saved ? T.green : T.accent, border: "none", borderRadius: 8, padding: "0 11px", cursor: "pointer", fontWeight: 700, whiteSpace: "nowrap", display: "inline-flex", alignItems: "center", gap: 4 }}>{saved ? <Check size={15} /> : <Plus size={15} />}</button>
          </div>
        </Field>
      </div>
      <div style={{ marginTop: 14, borderTop: `1px dashed ${T.line}`, paddingTop: 12 }}>
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", flexWrap: "wrap", gap: 10 }}>
          <span style={{ fontSize: 12, fontWeight: 600, color: T.inkSoft, display: "inline-flex", alignItems: "center", gap: 6 }}><Calendar size={14} /> Release Dates (insertions)</span>
          <div style={{ display: "flex", gap: 6, alignItems: "center" }}>
            <input type="date" value={newDate} onChange={(e) => setNewDate(e.target.value)} style={{ ...inputStyle, width: "auto", padding: "6px 9px" }} />
            <Btn size="sm" icon={Plus} onClick={addDate}>Add</Btn>
          </div>
        </div>
        <div style={{ display: "flex", flexWrap: "wrap", gap: 7, marginTop: 10 }}>
          {(line.releaseDates || []).length === 0 && <span style={{ fontSize: 12, color: T.inkSoft }}>No dates yet — counted as 1 insertion until you add release dates.</span>}
          {(line.releaseDates || []).map((d) => (
            <span key={d} style={{ display: "inline-flex", alignItems: "center", gap: 6, background: "#fff", border: `1px solid ${T.line}`, borderRadius: 20, padding: "4px 6px 4px 11px", fontSize: 12.5, fontWeight: 600 }}>{fmtShort(d)}<button onClick={() => rmDate(d)} style={{ background: T.graySoft, border: "none", borderRadius: "50%", width: 18, height: 18, cursor: "pointer", color: T.inkSoft, display: "inline-flex", alignItems: "center", justifyContent: "center" }}><X size={11} /></button></span>
          ))}
        </div>
        <div style={{ display: "flex", justifyContent: "flex-end", gap: 18, marginTop: 12, fontSize: 13 }}>
          <span style={{ color: T.inkSoft }}>{ins} insertion{ins > 1 ? "s" : ""} × {area.toLocaleString("en-IN")} cm² × {inr(line.rate || 0)}</span>
          <span style={{ fontWeight: 700, fontVariantNumeric: "tabular-nums" }}>{inr(amount)}</span>
        </div>
      </div>
    </div>
  );
}

function LineEditor({ lines, setLines, rates, vendor, onSaveRate }) {
  const add = () => setLines([...lines, { _uid: uid(), rateId: "", desc: "", color: "Color", height: "", width: "", rate: 0, releaseDates: [] }]);
  const update = (u, patch) => setLines(lines.map((l) => (l._uid === u ? { ...l, ...patch } : l)));
  const remove = (u) => setLines(lines.filter((l) => l._uid !== u));
  return (
    <div>
      {lines.map((l, i) => <LineCard key={l._uid} line={l} index={i} rates={rates} vendor={vendor} onSaveRate={onSaveRate} update={(p) => update(l._uid, p)} remove={() => remove(l._uid)} />)}
      <Btn variant="ghost" icon={Plus} onClick={add}>Add line item</Btn>
    </div>
  );
}

/* ============================ DOCUMENT VIEWER ============================= */
function DocViewer({ doc, onClose, kind, db }) {
  if (!doc) return null;
  const client = db.clients.find((c) => c.id === doc.clientId);
  const totals = computeTotals(doc.lines, doc.discountPct, doc.gstPct);
  return (
    <Modal open={!!doc} onClose={onClose} title={`${kind} — ${doc.id}`} width={760}>
      <div style={{ border: `1px solid ${T.line}`, borderRadius: 12, padding: 24, background: "#fff" }}>
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "flex-start", borderBottom: `2px solid ${T.ink}`, paddingBottom: 14 }}>
          <div><div style={{ fontFamily: "Fraunces, serif", fontSize: 22, fontWeight: 700, color: T.ink }}>{db.settings.company}</div><div style={{ fontSize: 12, color: T.inkSoft, marginTop: 2 }}>Advertising · Marketing · Media Buying</div></div>
          <div style={{ textAlign: "right" }}><div style={{ fontFamily: "Fraunces, serif", fontSize: 18, fontWeight: 600, color: T.accent, letterSpacing: 1 }}>{kind}</div><div style={{ ...mono, marginTop: 4 }}>{doc.id}</div><div style={{ fontSize: 12, color: T.inkSoft }}>{fmtDate(doc.date || today())}</div></div>
        </div>
        <div style={{ display: "flex", justifyContent: "space-between", margin: "16px 0", fontSize: 13 }}>
          <div><div style={{ fontSize: 11, fontWeight: 700, color: T.inkSoft, textTransform: "uppercase" }}>{kind === "RELEASE ORDER" ? "Vendor / Publisher" : "Billed To"}</div>
            <div style={{ fontWeight: 600, marginTop: 3 }}>{kind === "RELEASE ORDER" ? doc.vendor : client?.name}</div>
            {kind !== "RELEASE ORDER" && <div style={{ color: T.inkSoft, fontSize: 12.5, maxWidth: 280 }}>{client?.address}</div>}
            {kind !== "RELEASE ORDER" && client?.gstin && <div style={{ ...mono, marginTop: 2 }}>GSTIN: {client.gstin}</div>}</div>
          <div style={{ textAlign: "right", fontSize: 12.5, color: T.inkSoft }}>{doc.project && <div><b style={{ color: T.ink }}>Campaign:</b> {doc.project}</div>}<div><b style={{ color: T.ink }}>Client:</b> {client?.name}</div>{doc.dueDate && <div><b style={{ color: T.ink }}>Due:</b> {fmtDate(doc.dueDate)}</div>}</div>
        </div>
        <table style={{ width: "100%", borderCollapse: "collapse", fontSize: 12.5 }}>
          <thead><tr style={{ background: T.paper }}><th style={docTh}>Description</th><th style={{ ...docTh, textAlign: "right" }}>Size (cm)</th><th style={{ ...docTh, textAlign: "right" }}>Rate/cm²</th><th style={{ ...docTh, textAlign: "center" }}>Insertions</th><th style={{ ...docTh, textAlign: "right" }}>Amount</th></tr></thead>
          <tbody>{doc.lines.map((l) => (
            <tr key={l._uid} style={{ borderBottom: `1px solid ${T.line}`, verticalAlign: "top" }}>
              <td style={docTd}><div style={{ fontWeight: 600, display: "flex", alignItems: "center", gap: 7 }}>{l.desc || "—"} <ColorPill c={l.color} /></div>{(l.releaseDates || []).length > 0 && <div style={{ fontSize: 11, color: T.inkSoft, marginTop: 3 }}>Dates: {l.releaseDates.map(fmtShort).join(", ")}</div>}</td>
              <td style={{ ...docTd, textAlign: "right" }}>{l.height}×{l.width}<div style={{ fontSize: 11, color: T.inkSoft }}>{lineArea(l).toLocaleString("en-IN")} cm²</div></td>
              <td style={{ ...docTd, textAlign: "right" }}>{inr2(l.rate)}</td>
              <td style={{ ...docTd, textAlign: "center" }}>{insertions(l)}</td>
              <td style={{ ...docTd, textAlign: "right", fontWeight: 600 }}>{inr2(lineAmount(l))}</td>
            </tr>))}</tbody>
        </table>
        <TotalsBlock totals={totals} />
        <div style={{ marginTop: 18, fontSize: 11, color: T.inkSoft, textAlign: "center" }}>System-generated document · Subject to standard terms & conditions.</div>
      </div>
      <div style={{ display: "flex", justifyContent: "flex-end", gap: 10, marginTop: 16 }}><Btn variant="ghost" icon={Printer} onClick={() => window.print()}>Print</Btn><Btn onClick={onClose}>Close</Btn></div>
    </Modal>
  );
}
const docTh = { padding: "9px 10px", fontSize: 11, fontWeight: 700, color: T.inkSoft, textAlign: "left", textTransform: "uppercase" };
const docTd = { padding: "9px 10px", color: T.ink };

/* ============================= RELEASE ORDERS ============================= */
function ReleaseOrders() {
  const { db, save, nav } = useDB();
  const [q, setQ] = useState(""); const [open, setOpen] = useState(false); const [editing, setEditing] = useState(null); const [viewing, setViewing] = useState(null);
  const blank = { clientId: "", vendor: "", project: "", lines: [], discountPct: 0, gstPct: db.settings.gstPct, status: "Draft" };
  const [form, setForm] = useState(blank);
  const f = (k, v) => setForm((p) => ({ ...p, [k]: v }));
  const clientName = (id) => db.clients.find((c) => c.id === id)?.name || "—";
  const rows = db.releaseOrders.filter((x) => (x.id + x.project + x.vendor + clientName(x.clientId)).toLowerCase().includes(q.toLowerCase()));
  const saveRateToCard = (line) => {
    const r = { _uid: uid(), id: nextId("RT", db.rates), vendor: form.vendor || line.vendor || "—", publication: line.desc || "Custom", color: line.color || "Color", rate: Number(line.rate) || 0 };
    save("rates", [...db.rates, r]);
  };
  const submit = () => {
    if (!form.clientId || form.lines.length === 0) { alert("Select a client and add at least one line item."); return; }
    if (editing) save("releaseOrders", db.releaseOrders.map((x) => (x._uid === editing ? { ...form } : x)));
    else save("releaseOrders", [...db.releaseOrders, { ...form, _uid: uid(), id: nextId("RO", db.releaseOrders) }]);
    setOpen(false);
  };
  const del = (x) => { if (confirm(`Delete ${x.id}?`)) save("releaseOrders", db.releaseOrders.filter((y) => y._uid !== x._uid)); };
  const setStatus = (x, s) => save("releaseOrders", db.releaseOrders.map((y) => (y._uid === x._uid ? { ...y, status: s } : y)));
  const genInvoice = (x) => { save("invoices", [...db.invoices, { _uid: uid(), id: nextId("INV", db.invoices), roId: x.id, clientId: x.clientId, date: today(), dueDate: today(), lines: x.lines, discountPct: x.discountPct, gstPct: x.gstPct, status: "Draft", paid: 0 }]); nav("invoices"); };
  const roDates = (x) => { const all = (x.lines || []).flatMap((l) => l.releaseDates || []); return { first: all.sort()[0], count: all.length }; };

  return (
    <div>
      <PageHead title="Release Orders" subtitle="The heart of the workflow — book ad space with vendors, by release date & size."
        action={<Btn icon={Plus} onClick={() => { setForm({ ...blank, gstPct: db.settings.gstPct }); setEditing(null); setOpen(true); }}>New Release Order</Btn>} />
      <Card>
        <div style={{ padding: 14, borderBottom: `1px solid ${T.line}` }}><SearchBar value={q} onChange={setQ} /></div>
        <Table empty="No release orders yet." rows={rows} columns={[
          { key: "id", label: "RO #", render: (r) => <span style={mono}>{r.id}</span> },
          { key: "project", label: "Campaign / Client", render: (r) => <div><div style={{ fontWeight: 600 }}>{r.project}</div><div style={{ fontSize: 12, color: T.inkSoft }}>{clientName(r.clientId)} · {r.vendor}</div></div> },
          { key: "rel", label: "Release", render: (r) => { const d = roDates(r); return <span style={{ fontSize: 12.5 }}>{d.count ? `${fmtShort(d.first)} +${d.count - 1 >= 0 ? d.count - 1 : 0} more` : "—"}<div style={{ color: T.inkSoft }}>{d.count} insertion{d.count !== 1 ? "s" : ""}</div></span>; } },
          { key: "total", label: "Value", align: "right", render: (r) => inr(computeTotals(r.lines, r.discountPct, r.gstPct).total) },
          { key: "status", label: "Status", render: (r) => <Badge status={r.status} /> },
          { key: "act", label: "Actions", align: "right", render: (r) => <div style={{ display: "flex", gap: 6, justifyContent: "flex-end", alignItems: "center" }}>
            {r.status === "Draft" && <Btn size="sm" variant="soft" onClick={() => setStatus(r, "Issued")}>Issue</Btn>}
            {r.status === "Issued" && <Btn size="sm" variant="soft" onClick={() => setStatus(r, "In Progress")}>Start</Btn>}
            {r.status === "In Progress" && <Btn size="sm" variant="soft" onClick={() => setStatus(r, "Completed")}>Complete</Btn>}
            <Btn size="sm" variant="ghost" icon={Receipt} onClick={() => genInvoice(r)}>Invoice</Btn>
            <RowActions onView={() => setViewing(r)} onEdit={() => { setForm(r); setEditing(r._uid); setOpen(true); }} onDelete={() => del(r)} />
          </div> },
        ]} />
      </Card>

      <Modal open={open} onClose={() => setOpen(false)} title={editing ? "Edit Release Order" : "New Release Order"} width={860}>
        <div style={grid2}>
          <Field label="Client"><SelectInput value={form.clientId} onChange={(e) => f("clientId", e.target.value)} options={db.clients.map((c) => ({ value: c.id, label: c.name }))} /></Field>
          <Field label="Vendor / Publisher"><SelectInput value={form.vendor} onChange={(e) => f("vendor", e.target.value)} options={db.vendors.map((v) => ({ value: v.name, label: v.name }))} /></Field>
          <Field label="Campaign / Project" span={2}><TextInput value={form.project} onChange={(e) => f("project", e.target.value)} /></Field>
        </div>
        <div style={{ marginTop: 16, marginBottom: 8, fontSize: 12, fontWeight: 700, color: T.inkSoft, textTransform: "uppercase", letterSpacing: 0.4 }}>Insertions (rates fully editable · use + to push a new rate into the Rate Card)</div>
        <LineEditor lines={form.lines} setLines={(l) => f("lines", l)} rates={db.rates} vendor={form.vendor} onSaveRate={saveRateToCard} />
        <div style={{ ...grid2, marginTop: 14 }}>
          <Field label="Discount %"><NumInput value={form.discountPct} onChange={(e) => f("discountPct", Number(e.target.value))} /></Field>
          <Field label="GST %"><NumInput value={form.gstPct} onChange={(e) => f("gstPct", Number(e.target.value))} /></Field>
          <Field label="Status"><SelectInput value={form.status} onChange={(e) => f("status", e.target.value)} options={["Draft", "Issued", "In Progress", "Completed"].map((x) => ({ value: x, label: x }))} /></Field>
        </div>
        <TotalsBlock totals={computeTotals(form.lines, form.discountPct, form.gstPct)} />
        <ModalFooter onCancel={() => setOpen(false)} onSave={submit} />
      </Modal>
      <DocViewer doc={viewing} onClose={() => setViewing(null)} kind="RELEASE ORDER" db={db} />
    </div>
  );
}

/* ================================= VENDORS ================================ */
function Vendors() {
  const { db, save } = useDB();
  const [q, setQ] = useState(""); const [open, setOpen] = useState(false); const [editing, setEditing] = useState(null);
  const blank = { name: "", type: "Newspaper", contact: "", phone: "", locations: "", terms: "Net 30", commission: 0 };
  const [form, setForm] = useState(blank);
  const f = (k, v) => setForm((p) => ({ ...p, [k]: v }));
  const rows = db.vendors.filter((v) => (v.name + v.type + v.locations).toLowerCase().includes(q.toLowerCase()));
  const submit = () => { if (!form.name) return; if (editing) save("vendors", db.vendors.map((v) => (v._uid === editing ? { ...form } : v))); else save("vendors", [...db.vendors, { ...form, _uid: uid(), id: nextId("VN", db.vendors) }]); setOpen(false); };
  const del = (v) => { if (confirm(`Delete ${v.name}?`)) save("vendors", db.vendors.filter((x) => x._uid !== v._uid)); };
  return (
    <div>
      <PageHead title="Vendors & Publishers" subtitle="Newspapers and media partners you book and pay against."
        action={<Btn icon={Plus} onClick={() => { setForm(blank); setEditing(null); setOpen(true); }}>New Vendor</Btn>} />
      <Card>
        <div style={{ padding: 14, borderBottom: `1px solid ${T.line}` }}><SearchBar value={q} onChange={setQ} /></div>
        <Table empty="No vendors yet." rows={rows} columns={[
          { key: "id", label: "ID", render: (r) => <span style={mono}>{r.id}</span> },
          { key: "name", label: "Vendor", render: (r) => <div><div style={{ fontWeight: 600 }}>{r.name}</div><div style={{ fontSize: 12, color: T.inkSoft }}>{r.contact} · {r.phone}</div></div> },
          { key: "type", label: "Type" },
          { key: "locations", label: "Coverage" },
          { key: "commission", label: "Comm. %", align: "center", render: (r) => r.commission + "%" },
          { key: "terms", label: "Terms", align: "center" },
          { key: "act", label: "", align: "right", render: (r) => <RowActions onEdit={() => { setForm(r); setEditing(r._uid); setOpen(true); }} onDelete={() => del(r)} /> },
        ]} />
      </Card>
      <Modal open={open} onClose={() => setOpen(false)} title={editing ? "Edit Vendor" : "New Vendor"}>
        <div style={grid2}>
          <Field label="Vendor / Publisher Name" span={2}><TextInput value={form.name} onChange={(e) => f("name", e.target.value)} /></Field>
          <Field label="Type"><SelectInput value={form.type} onChange={(e) => f("type", e.target.value)} options={["Newspaper", "Magazine", "Hoarding", "Radio", "Transit", "Digital"].map((x) => ({ value: x, label: x }))} /></Field>
          <Field label="Contact Desk"><TextInput value={form.contact} onChange={(e) => f("contact", e.target.value)} /></Field>
          <Field label="Phone"><TextInput value={form.phone} onChange={(e) => f("phone", e.target.value)} /></Field>
          <Field label="Commission %"><NumInput value={form.commission} onChange={(e) => f("commission", Number(e.target.value))} /></Field>
          <Field label="Locations Covered" span={2}><TextInput value={form.locations} onChange={(e) => f("locations", e.target.value)} /></Field>
          <Field label="Payment Terms"><SelectInput value={form.terms} onChange={(e) => f("terms", e.target.value)} options={["Net 30", "Net 45", "50% advance", "Advance", "On publication"].map((x) => ({ value: x, label: x }))} /></Field>
        </div>
        <ModalFooter onCancel={() => setOpen(false)} onSave={submit} />
      </Modal>
    </div>
  );
}

/* ================================= CLIENTS ================================ */
function Clients() {
  const { db, save } = useDB();
  const [q, setQ] = useState(""); const [open, setOpen] = useState(false); const [editing, setEditing] = useState(null);
  const blank = { name: "", contact: "", email: "", phone: "", gstin: "", category: "Real Estate", city: "", credit: 0, terms: "30 days", address: "" };
  const [form, setForm] = useState(blank);
  const f = (k, v) => setForm((p) => ({ ...p, [k]: v }));
  const rows = db.clients.filter((c) => (c.name + c.city + c.category).toLowerCase().includes(q.toLowerCase()));
  const submit = () => { if (!form.name) return; if (editing) save("clients", db.clients.map((c) => (c._uid === editing ? { ...form } : c))); else save("clients", [...db.clients, { ...form, _uid: uid(), id: nextId("CL", db.clients) }]); setOpen(false); };
  const del = (c) => { if (confirm(`Delete ${c.name}?`)) save("clients", db.clients.filter((x) => x._uid !== c._uid)); };
  return (
    <div>
      <PageHead title="Clients" subtitle="Advertisers, GST details, credit limits and payment terms."
        action={<Btn icon={Plus} onClick={() => { setForm(blank); setEditing(null); setOpen(true); }}>New Client</Btn>} />
      <Card>
        <div style={{ padding: 14, borderBottom: `1px solid ${T.line}` }}><SearchBar value={q} onChange={setQ} /></div>
        <Table empty="No clients yet." rows={rows} columns={[
          { key: "id", label: "ID", render: (r) => <span style={mono}>{r.id}</span> },
          { key: "name", label: "Client", render: (r) => <div><div style={{ fontWeight: 600 }}>{r.name}</div><div style={{ fontSize: 12, color: T.inkSoft }}>{r.city} · {r.contact}</div></div> },
          { key: "category", label: "Category" },
          { key: "gstin", label: "GSTIN", render: (r) => <span style={mono}>{r.gstin || "—"}</span> },
          { key: "credit", label: "Credit Limit", align: "right", render: (r) => inr(r.credit) },
          { key: "terms", label: "Terms", align: "center" },
          { key: "act", label: "", align: "right", render: (r) => <RowActions onEdit={() => { setForm(r); setEditing(r._uid); setOpen(true); }} onDelete={() => del(r)} /> },
        ]} />
      </Card>
      <Modal open={open} onClose={() => setOpen(false)} title={editing ? "Edit Client" : "New Client"}>
        <div style={grid2}>
          <Field label="Client Name" span={2}><TextInput value={form.name} onChange={(e) => f("name", e.target.value)} /></Field>
          <Field label="Contact Person"><TextInput value={form.contact} onChange={(e) => f("contact", e.target.value)} /></Field>
          <Field label="Category"><SelectInput value={form.category} onChange={(e) => f("category", e.target.value)} options={["Real Estate", "Government", "Retail", "FMCG", "Education", "Other"].map((x) => ({ value: x, label: x }))} /></Field>
          <Field label="Email"><TextInput value={form.email} onChange={(e) => f("email", e.target.value)} /></Field>
          <Field label="Phone"><TextInput value={form.phone} onChange={(e) => f("phone", e.target.value)} /></Field>
          <Field label="GSTIN"><TextInput value={form.gstin} onChange={(e) => f("gstin", e.target.value)} /></Field>
          <Field label="City"><TextInput value={form.city} onChange={(e) => f("city", e.target.value)} /></Field>
          <Field label="Credit Limit (₹)"><NumInput value={form.credit} onChange={(e) => f("credit", Number(e.target.value))} /></Field>
          <Field label="Payment Terms"><SelectInput value={form.terms} onChange={(e) => f("terms", e.target.value)} options={["15 days", "30 days", "45 days", "60 days", "Advance"].map((x) => ({ value: x, label: x }))} /></Field>
          <Field label="Billing Address" span={2}><TextArea value={form.address} onChange={(e) => f("address", e.target.value)} /></Field>
        </div>
        <ModalFooter onCancel={() => setOpen(false)} onSave={submit} />
      </Modal>
    </div>
  );
}

/* ================================ RATE CARDS ============================== */
function RateCard() {
  const { db, save } = useDB();
  const [q, setQ] = useState(""); const [open, setOpen] = useState(false); const [editing, setEditing] = useState(null);
  const blank = { vendor: "", publication: "", color: "Color", rate: 0 };
  const [form, setForm] = useState(blank);
  const f = (k, v) => setForm((p) => ({ ...p, [k]: v }));
  const rows = db.rates.filter((r) => (r.vendor + r.publication + r.color).toLowerCase().includes(q.toLowerCase()));
  const submit = () => { if (!form.vendor) return; if (editing) save("rates", db.rates.map((r) => (r._uid === editing ? { ...form } : r))); else save("rates", [...db.rates, { ...form, _uid: uid(), id: nextId("RT", db.rates) }]); setOpen(false); };
  const del = (r) => { if (confirm("Delete this rate?")) save("rates", db.rates.filter((x) => x._uid !== r._uid)); };
  return (
    <div>
      <PageHead title="Rate Cards" subtitle="Reference rates per cm² by vendor, publication & colour. Editable anywhere they're used."
        action={<Btn icon={Plus} onClick={() => { setForm(blank); setEditing(null); setOpen(true); }}>New Rate</Btn>} />
      <Card>
        <div style={{ padding: 14, borderBottom: `1px solid ${T.line}` }}><SearchBar value={q} onChange={setQ} /></div>
        <Table empty="No rates configured." rows={rows} columns={[
          { key: "id", label: "ID", render: (r) => <span style={mono}>{r.id}</span> },
          { key: "vendor", label: "Vendor" },
          { key: "publication", label: "Publication / Edition", render: (r) => <span style={{ fontWeight: 600 }}>{r.publication}</span> },
          { key: "color", label: "Colour", render: (r) => <ColorPill c={r.color} /> },
          { key: "rate", label: "Rate", align: "right", render: (r) => <span style={{ fontWeight: 600 }}>{inr(r.rate)}<span style={{ color: T.inkSoft, fontWeight: 400 }}> /cm²</span></span> },
          { key: "act", label: "", align: "right", render: (r) => <RowActions onEdit={() => { setForm(r); setEditing(r._uid); setOpen(true); }} onDelete={() => del(r)} /> },
        ]} />
      </Card>
      <Modal open={open} onClose={() => setOpen(false)} title={editing ? "Edit Rate" : "New Rate"}>
        <div style={grid2}>
          <Field label="Vendor / Publisher"><SelectInput value={form.vendor} onChange={(e) => f("vendor", e.target.value)} options={db.vendors.map((v) => ({ value: v.name, label: v.name }))} /></Field>
          <Field label="Colour"><SelectInput value={form.color} onChange={(e) => f("color", e.target.value)} options={[{ value: "Color", label: "Colour" }, { value: "B&W", label: "Black & White" }]} /></Field>
          <Field label="Publication / Edition" span={2}><TextInput value={form.publication} onChange={(e) => f("publication", e.target.value)} placeholder="e.g. Delhi — Front Page" /></Field>
          <Field label="Rate (₹ per cm²)"><NumInput value={form.rate} onChange={(e) => f("rate", Number(e.target.value))} /></Field>
        </div>
        <ModalFooter onCancel={() => setOpen(false)} onSave={submit} />
      </Modal>
    </div>
  );
}

/* =============================== INVOICES ================================= */
function Invoices() {
  const { db, save } = useDB();
  const [q, setQ] = useState(""); const [open, setOpen] = useState(false); const [editing, setEditing] = useState(null);
  const [viewing, setViewing] = useState(null); const [payFor, setPayFor] = useState(null); const [payAmt, setPayAmt] = useState(0);
  const blank = { roId: "", clientId: "", date: today(), dueDate: today(), lines: [], discountPct: 0, gstPct: db.settings.gstPct, status: "Draft", paid: 0 };
  const [form, setForm] = useState(blank);
  const f = (k, v) => setForm((p) => ({ ...p, [k]: v }));
  const clientName = (id) => db.clients.find((c) => c.id === id)?.name || "—";
  const isOverdue = (x) => x.status !== "Paid" && x.dueDate && x.dueDate < today();
  const rows = db.invoices.filter((x) => (x.id + clientName(x.clientId) + (x.roId || "")).toLowerCase().includes(q.toLowerCase()));
  const saveRateToCard = (line) => save("rates", [...db.rates, { _uid: uid(), id: nextId("RT", db.rates), vendor: line.vendor || "—", publication: line.desc || "Custom", color: line.color || "Color", rate: Number(line.rate) || 0 }]);
  const submit = () => { if (!form.clientId || form.lines.length === 0) { alert("Client and line items required."); return; } if (editing) save("invoices", db.invoices.map((x) => (x._uid === editing ? { ...form } : x))); else save("invoices", [...db.invoices, { ...form, _uid: uid(), id: nextId("INV", db.invoices) }]); setOpen(false); };
  const del = (x) => { if (confirm(`Delete ${x.id}?`)) save("invoices", db.invoices.filter((y) => y._uid !== x._uid)); };
  const setStatus = (x, s) => save("invoices", db.invoices.map((y) => (y._uid === x._uid ? { ...y, status: s } : y)));
  const recordPayment = () => { const total = computeTotals(payFor.lines, payFor.discountPct, payFor.gstPct).total; const paid = (payFor.paid || 0) + Number(payAmt); const status = paid >= total ? "Paid" : paid > 0 ? "Partially Paid" : payFor.status; save("invoices", db.invoices.map((y) => (y._uid === payFor._uid ? { ...y, paid, status } : y))); setPayFor(null); setPayAmt(0); };
  const prefillFromRO = (roId) => { const ro = db.releaseOrders.find((r) => r.id === roId); if (ro) setForm((p) => ({ ...p, roId, clientId: ro.clientId, lines: ro.lines, discountPct: ro.discountPct, gstPct: ro.gstPct })); else f("roId", roId); };

  return (
    <div>
      <PageHead title="Invoices & Billing" subtitle="GST-compliant client billing with B&W / colour line detail and payment tracking."
        action={<Btn icon={Plus} onClick={() => { setForm({ ...blank, gstPct: db.settings.gstPct }); setEditing(null); setOpen(true); }}>New Invoice</Btn>} />
      <Card>
        <div style={{ padding: 14, borderBottom: `1px solid ${T.line}` }}><SearchBar value={q} onChange={setQ} /></div>
        <Table empty="No invoices yet. Generate one from a release order." rows={rows} columns={[
          { key: "id", label: "Invoice #", render: (r) => <span style={mono}>{r.id}</span> },
          { key: "client", label: "Client / RO", render: (r) => <div><div style={{ fontWeight: 600 }}>{clientName(r.clientId)}</div><div style={{ fontSize: 12, color: T.inkSoft }}>{r.roId || "Direct"} · due {fmtDate(r.dueDate)}</div></div> },
          { key: "total", label: "Total", align: "right", render: (r) => inr(computeTotals(r.lines, r.discountPct, r.gstPct).total) },
          { key: "paid", label: "Paid", align: "right", render: (r) => <span style={{ color: T.green }}>{inr(r.paid || 0)}</span> },
          { key: "bal", label: "Balance", align: "right", render: (r) => { const b = computeTotals(r.lines, r.discountPct, r.gstPct).total - (r.paid || 0); return <span style={{ color: b > 0 ? T.red : T.green, fontWeight: 600 }}>{inr(b)}</span>; } },
          { key: "status", label: "Status", render: (r) => <Badge status={isOverdue(r) ? "Overdue" : r.status} /> },
          { key: "act", label: "Actions", align: "right", render: (r) => <div style={{ display: "flex", gap: 6, justifyContent: "flex-end", alignItems: "center" }}>
            {r.status === "Draft" && <Btn size="sm" variant="soft" onClick={() => setStatus(r, "Sent")}>Send</Btn>}
            {r.status !== "Paid" && <Btn size="sm" icon={Wallet} onClick={() => { setPayFor(r); setPayAmt(0); }}>Pay</Btn>}
            <RowActions onView={() => setViewing(r)} onEdit={() => { setForm(r); setEditing(r._uid); setOpen(true); }} onDelete={() => del(r)} />
          </div> },
        ]} />
      </Card>

      <Modal open={open} onClose={() => setOpen(false)} title={editing ? "Edit Invoice" : "New Invoice"} width={860}>
        <div style={grid2}>
          <Field label="Client"><SelectInput value={form.clientId} onChange={(e) => f("clientId", e.target.value)} options={db.clients.map((c) => ({ value: c.id, label: c.name }))} /></Field>
          <Field label="Pull from Release Order"><SelectInput value={form.roId} onChange={(e) => prefillFromRO(e.target.value)} options={db.releaseOrders.map((r) => ({ value: r.id, label: r.id + " · " + r.project }))} /></Field>
          <Field label="Invoice Date"><TextInput type="date" value={form.date} onChange={(e) => f("date", e.target.value)} /></Field>
          <Field label="Due Date"><TextInput type="date" value={form.dueDate} onChange={(e) => f("dueDate", e.target.value)} /></Field>
        </div>
        <div style={{ marginTop: 16, marginBottom: 8, fontSize: 12, fontWeight: 700, color: T.inkSoft, textTransform: "uppercase", letterSpacing: 0.4 }}>Insertions (editable · + adds rate to card)</div>
        <LineEditor lines={form.lines} setLines={(l) => f("lines", l)} rates={db.rates} vendor="" onSaveRate={saveRateToCard} />
        <div style={{ ...grid2, marginTop: 14 }}>
          <Field label="Discount %"><NumInput value={form.discountPct} onChange={(e) => f("discountPct", Number(e.target.value))} /></Field>
          <Field label="GST %"><NumInput value={form.gstPct} onChange={(e) => f("gstPct", Number(e.target.value))} /></Field>
        </div>
        <TotalsBlock totals={computeTotals(form.lines, form.discountPct, form.gstPct)} />
        <ModalFooter onCancel={() => setOpen(false)} onSave={submit} />
      </Modal>

      <Modal open={!!payFor} onClose={() => setPayFor(null)} title="Record Payment" width={460}>
        {payFor && (() => { const total = computeTotals(payFor.lines, payFor.discountPct, payFor.gstPct).total; const bal = total - (payFor.paid || 0); return (
          <div>
            <div style={{ background: T.paper, borderRadius: 10, padding: 14, marginBottom: 14, fontSize: 13.5 }}>
              <div style={{ display: "flex", justifyContent: "space-between", padding: "3px 0" }}><span style={{ color: T.inkSoft }}>Invoice</span><b>{payFor.id}</b></div>
              <div style={{ display: "flex", justifyContent: "space-between", padding: "3px 0" }}><span style={{ color: T.inkSoft }}>Total</span><b>{inr2(total)}</b></div>
              <div style={{ display: "flex", justifyContent: "space-between", padding: "3px 0" }}><span style={{ color: T.inkSoft }}>Already paid</span><b style={{ color: T.green }}>{inr2(payFor.paid || 0)}</b></div>
              <div style={{ display: "flex", justifyContent: "space-between", padding: "3px 0", borderTop: `1px solid ${T.line}`, marginTop: 4 }}><span style={{ color: T.inkSoft }}>Balance due</span><b style={{ color: T.red }}>{inr2(bal)}</b></div>
            </div>
            <Field label="Payment Amount (₹)"><NumInput value={payAmt} onChange={(e) => setPayAmt(e.target.value)} /></Field>
            <div style={{ marginTop: 8 }}><Btn size="sm" variant="ghost" onClick={() => setPayAmt(bal)}>Full balance</Btn></div>
            <ModalFooter onCancel={() => setPayFor(null)} onSave={recordPayment} saveLabel="Record Payment" />
          </div>); })()}
      </Modal>
      <DocViewer doc={viewing} onClose={() => setViewing(null)} kind="TAX INVOICE" db={db} />
    </div>
  );
}

/* =========================== REPORTS & MASTER ============================= */
function Reports() {
  const { db, save } = useDB();
  const [company, setCompany] = useState(db.settings.company);
  const [gst, setGst] = useState(db.settings.gstPct);

  const clientReport = db.clients.map((c) => {
    const invs = db.invoices.filter((i) => i.clientId === c.id);
    const billed = invs.reduce((s, i) => s + computeTotals(i.lines, i.discountPct, i.gstPct).total, 0);
    const paid = invs.reduce((s, i) => s + (i.paid || 0), 0);
    return { Client: c.name, Category: c.category, Invoices: invs.length, Billed: Math.round(billed), Collected: Math.round(paid), Outstanding: Math.round(billed - paid) };
  });
  const vendorPay = db.vendors.map((v) => {
    const ros = db.releaseOrders.filter((r) => r.vendor === v.name);
    let colour = 0, bw = 0;
    ros.forEach((r) => r.lines.forEach((l) => { const amt = lineAmount(l); if (l.color === "Color") colour += amt; else bw += amt; }));
    return { Vendor: v.name, ROs: ros.length, "Colour Value": Math.round(colour), "B&W Value": Math.round(bw), "Total Payable": Math.round(colour + bw) };
  }).filter((v) => v.ROs > 0 || true);

  return (
    <div>
      <PageHead title="Reports & Master" subtitle="Client revenue, vendor payables (B&W / colour split) and master settings." />
      <Card style={{ marginBottom: 16 }}>
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "14px 18px", borderBottom: `1px solid ${T.line}` }}><h3 style={{ ...chartTitle, margin: 0 }}>Client Revenue</h3><Btn size="sm" variant="ghost" icon={Download} onClick={() => downloadCSV("client-report.csv", clientReport)}>Export CSV</Btn></div>
        <Table empty="No data." rows={clientReport.map((r, i) => ({ ...r, _uid: i }))} columns={[
          { key: "Client", label: "Client", render: (r) => <b>{r.Client}</b> }, { key: "Category", label: "Category" }, { key: "Invoices", label: "Invoices", align: "center" },
          { key: "Billed", label: "Billed", align: "right", render: (r) => inr(r.Billed) }, { key: "Collected", label: "Collected", align: "right", render: (r) => <span style={{ color: T.green }}>{inr(r.Collected)}</span> },
          { key: "Outstanding", label: "Outstanding", align: "right", render: (r) => <span style={{ color: r.Outstanding > 0 ? T.red : T.green, fontWeight: 600 }}>{inr(r.Outstanding)}</span> },
        ]} />
      </Card>
      <Card style={{ marginBottom: 16 }}>
        <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "14px 18px", borderBottom: `1px solid ${T.line}` }}><h3 style={{ ...chartTitle, margin: 0 }}>Vendor Payables — Colour vs B&W</h3><Btn size="sm" variant="ghost" icon={Download} onClick={() => downloadCSV("vendor-payables.csv", vendorPay)}>Export CSV</Btn></div>
        <Table empty="No data." rows={vendorPay.map((r, i) => ({ ...r, _uid: i }))} columns={[
          { key: "Vendor", label: "Vendor", render: (r) => <b>{r.Vendor}</b> }, { key: "ROs", label: "ROs", align: "center" },
          { key: "Colour Value", label: "Colour", align: "right", render: (r) => <span style={{ color: T.accent }}>{inr(r["Colour Value"])}</span> },
          { key: "B&W Value", label: "B&W", align: "right", render: (r) => inr(r["B&W Value"]) },
          { key: "Total Payable", label: "Total Payable", align: "right", render: (r) => <b>{inr(r["Total Payable"])}</b> },
        ]} />
      </Card>
      <Card style={{ padding: 20 }}>
        <h3 style={chartTitle}>Master Settings</h3>
        <div style={{ ...grid2, maxWidth: 560 }}>
          <Field label="Company Name" span={2}><TextInput value={company} onChange={(e) => setCompany(e.target.value)} /></Field>
          <Field label="Default GST %"><NumInput value={gst} onChange={(e) => setGst(Number(e.target.value))} /></Field>
        </div>
        <div style={{ marginTop: 14 }}><Btn icon={Check} onClick={() => save("settings", { ...db.settings, company, gstPct: gst })}>Save Settings</Btn></div>
      </Card>
    </div>
  );
}

/* ================================ DASHBOARD =============================== */
function Dashboard() {
  const { db, nav } = useDB();
  const k = useMemo(() => {
    const revenue = db.invoices.reduce((s, i) => s + computeTotals(i.lines, i.discountPct, i.gstPct).total, 0);
    const collected = db.invoices.reduce((s, i) => s + (i.paid || 0), 0);
    const booked = db.releaseOrders.reduce((s, r) => s + computeTotals(r.lines, r.discountPct, r.gstPct).total, 0);
    const active = db.releaseOrders.filter((r) => ["Issued", "In Progress"].includes(r.status)).length;
    return { revenue, collected, outstanding: revenue - collected, booked, active };
  }, [db]);
  const monthly = useMemo(() => { const m = {}; db.invoices.forEach((i) => { const k2 = (i.date || today()).slice(0, 7); m[k2] = (m[k2] || 0) + computeTotals(i.lines, i.discountPct, i.gstPct).total; }); return Object.entries(m).sort().map(([mo, v]) => ({ month: new Date(mo + "-01").toLocaleDateString("en-IN", { month: "short" }), value: Math.round(v) })); }, [db]);
  const roStatus = useMemo(() => { const m = {}; db.releaseOrders.forEach((r) => (m[r.status] = (m[r.status] || 0) + 1)); return Object.entries(m).map(([name, value]) => ({ name, value })); }, [db]);
  const PIE = [T.accent, T.blue, T.green, T.amber, T.gray];
  const stats = [
    { label: "Total Revenue", value: inr(k.revenue), icon: IndianRupee, tint: T.accent, soft: T.accentSoft, sub: "Invoiced to date" },
    { label: "Collected", value: inr(k.collected), icon: Wallet, tint: T.green, soft: T.greenSoft, sub: "Payments received" },
    { label: "Outstanding", value: inr(k.outstanding), icon: AlertCircle, tint: T.red, soft: T.redSoft, sub: "Receivables pending" },
    { label: "Active Campaigns", value: k.active, icon: Megaphone, tint: T.blue, soft: T.blueSoft, sub: "Release orders running" },
    { label: "Booked Value", value: inr(k.booked), icon: TrendingUp, tint: T.amber, soft: T.amberSoft, sub: "All release orders" },
  ];
  return (
    <div>
      <PageHead title="Dashboard" subtitle="Business health across the print advertising lifecycle." />
      <div style={{ display: "grid", gridTemplateColumns: "repeat(auto-fit,minmax(190px,1fr))", gap: 14, marginBottom: 18 }}>
        {stats.map((s) => <Card key={s.label} style={{ padding: 18 }}>
          <div style={{ background: s.soft, color: s.tint, borderRadius: 10, padding: 9, display: "inline-flex" }}><s.icon size={20} /></div>
          <div style={{ fontSize: 26, fontWeight: 700, color: T.ink, marginTop: 14, fontVariantNumeric: "tabular-nums" }}>{s.value}</div>
          <div style={{ fontSize: 13, fontWeight: 600, color: T.ink, marginTop: 2 }}>{s.label}</div>
          <div style={{ fontSize: 11.5, color: T.inkSoft, marginTop: 1 }}>{s.sub}</div>
        </Card>)}
      </div>
      <div style={{ display: "grid", gridTemplateColumns: "1.6fr 1fr", gap: 16, marginBottom: 16 }}>
        <Card style={{ padding: 20 }}><h3 style={chartTitle}>Monthly Revenue</h3>
          <ResponsiveContainer width="100%" height={250}><BarChart data={monthly}><CartesianGrid strokeDasharray="3 3" stroke={T.line} vertical={false} /><XAxis dataKey="month" tick={{ fontSize: 12, fill: T.inkSoft }} axisLine={false} tickLine={false} /><YAxis tick={{ fontSize: 11, fill: T.inkSoft }} axisLine={false} tickLine={false} tickFormatter={(v) => "₹" + (v / 100000).toFixed(0) + "L"} /><Tooltip formatter={(v) => inr(v)} contentStyle={tooltipStyle} /><Bar dataKey="value" fill={T.accent} radius={[6, 6, 0, 0]} maxBarSize={48} /></BarChart></ResponsiveContainer>
        </Card>
        <Card style={{ padding: 20 }}><h3 style={chartTitle}>Release Order Status</h3>
          <ResponsiveContainer width="100%" height={250}><PieChart><Pie data={roStatus} dataKey="value" nameKey="name" cx="50%" cy="50%" innerRadius={48} outerRadius={88} paddingAngle={3}>{roStatus.map((e, i) => <Cell key={i} fill={STATUS[e.name]?.c || PIE[i % PIE.length]} />)}</Pie><Tooltip contentStyle={tooltipStyle} /></PieChart></ResponsiveContainer>
          <div style={{ display: "flex", flexWrap: "wrap", gap: 8, justifyContent: "center" }}>{roStatus.map((e, i) => <span key={i} style={{ fontSize: 11.5, color: T.inkSoft, display: "inline-flex", alignItems: "center", gap: 4 }}><span style={{ width: 9, height: 9, borderRadius: 3, background: STATUS[e.name]?.c || PIE[i % PIE.length] }} />{e.name} ({e.value})</span>)}</div>
        </Card>
      </div>
      <Card style={{ padding: 20 }}><h3 style={chartTitle}>Recent Activity</h3>
        <div>{[...db.releaseOrders.map((r) => ({ t: ((r.lines || []).flatMap((l) => l.releaseDates || []).sort()[0]) || today(), label: `${r.id} · ${r.project}`, s: r.status, go: "releaseOrders" })), ...db.invoices.map((i) => ({ t: i.date, label: `${i.id} · invoice raised`, s: i.status, go: "invoices" }))].sort((a, b) => (b.t || "").localeCompare(a.t || "")).slice(0, 6).map((a, i) => (
          <div key={i} onClick={() => nav(a.go)} style={{ display: "flex", justifyContent: "space-between", alignItems: "center", padding: "10px 0", borderBottom: `1px solid ${T.line}`, cursor: "pointer" }}>
            <div><div style={{ fontSize: 13, fontWeight: 600, color: T.ink }}>{a.label}</div><div style={{ fontSize: 11.5, color: T.inkSoft }}>{fmtDate(a.t)}</div></div><Badge status={a.s} />
          </div>))}</div>
      </Card>
    </div>
  );
}

/* ================================== APP =================================== */
const NAV = [
  { key: "dashboard", label: "Dashboard", icon: LayoutDashboard, group: "Overview" },
  { key: "releaseOrders", label: "Release Orders", icon: ScrollText, group: "Workflow", n: 1 },
  { key: "vendors", label: "Vendors & Publishers", icon: Store, group: "Workflow", n: 2 },
  { key: "clients", label: "Clients", icon: Users, group: "Workflow", n: 3 },
  { key: "rates", label: "Rate Cards", icon: Tags, group: "Workflow", n: 4 },
  { key: "invoices", label: "Invoices", icon: Receipt, group: "Workflow", n: 5 },
  { key: "reports", label: "Reports & Master", icon: BarChart3, group: "Workflow", n: 6 },
];

export default function App() {
  const [db, setDb] = useState(null);
  const [view, setView] = useState("dashboard");
  const [sidebarOpen, setSidebarOpen] = useState(true);

  useEffect(() => { (async () => {
    const loaded = {};
    for (const c of COLLECTIONS) loaded[c] = await sget(PFX + c);
    if (!loaded.clients || !loaded.settings || loaded.settings.version !== 2) {
      const s = seed(); for (const c of COLLECTIONS) await sset(PFX + c, s[c]); setDb(s);
    } else setDb(loaded);
  })(); }, []);

  const save = useCallback((collection, val) => { setDb((prev) => { const next = { ...prev, [collection]: val }; sset(PFX + collection, val); return next; }); }, []);
  const nav = useCallback((v) => setView(v), []);

  if (!db) return <div style={{ height: "100vh", display: "flex", alignItems: "center", justifyContent: "center", background: T.paper, fontFamily: "'Hanken Grotesk', sans-serif", color: T.inkSoft }}><div style={{ textAlign: "center" }}><div style={{ fontFamily: "Fraunces, serif", fontSize: 26, color: T.ink }}>Pehachan ERP</div><div style={{ marginTop: 8 }}>Loading workspace…</div></div></div>;

  const grouped = NAV.reduce((a, n) => { (a[n.group] = a[n.group] || []).push(n); return a; }, {});
  const Views = { dashboard: Dashboard, releaseOrders: ReleaseOrders, vendors: Vendors, clients: Clients, rates: RateCard, invoices: Invoices, reports: Reports };
  const Current = Views[view];

  return (
    <ERPCtx.Provider value={{ db, save, nav }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Fraunces:opsz,wght@9..144,500;9..144,600;9..144,700&family=Hanken+Grotesk:wght@400;500;600;700&display=swap');
        * { box-sizing: border-box; }
        body { margin: 0; }
        ::-webkit-scrollbar { width: 9px; height: 9px; }
        ::-webkit-scrollbar-thumb { background: ${T.line}; border-radius: 8px; }
        select option { color: ${T.ink}; }
        @media print { .no-print { display: none !important; } }
      `}</style>
      <div style={{ display: "flex", height: "100vh", fontFamily: "'Hanken Grotesk', sans-serif", background: T.paper, color: T.ink, overflow: "hidden" }}>
        <aside className="no-print" style={{ width: sidebarOpen ? 252 : 0, background: T.sidebar, color: T.paper, flexShrink: 0, transition: "width .2s", overflow: "hidden", display: "flex", flexDirection: "column" }}>
          <div style={{ padding: "22px 22px 16px", borderBottom: `1px solid ${T.sidebarSoft}` }}>
            <div style={{ display: "flex", alignItems: "center", gap: 10 }}>
              <div style={{ background: T.accent, borderRadius: 9, padding: 7, display: "inline-flex" }}><Briefcase size={18} color="#fff" /></div>
              <div><div style={{ fontFamily: "Fraunces, serif", fontSize: 19, fontWeight: 700, lineHeight: 1 }}>Pehachan</div><div style={{ fontSize: 10.5, color: "#A89E94", letterSpacing: 1, marginTop: 2 }}>PRINT AD ERP</div></div>
            </div>
          </div>
          <nav style={{ padding: "12px 12px", flex: 1, overflowY: "auto" }}>
            {Object.entries(grouped).map(([g, items]) => (
              <div key={g} style={{ marginBottom: 14 }}>
                <div style={{ fontSize: 10, fontWeight: 700, color: "#7E756C", letterSpacing: 1.2, padding: "4px 12px 6px", textTransform: "uppercase" }}>{g}</div>
                {items.map((nn) => { const active = view === nn.key; return (
                  <button key={nn.key} onClick={() => setView(nn.key)} style={{ display: "flex", alignItems: "center", gap: 11, width: "100%", textAlign: "left", padding: "9px 12px", borderRadius: 9, border: "none", cursor: "pointer", marginBottom: 2, fontSize: 13.5, fontWeight: active ? 600 : 500, fontFamily: "inherit", background: active ? T.accent : "transparent", color: active ? "#fff" : "#D8CFC5", transition: "all .15s" }} onMouseEnter={(e) => { if (!active) e.currentTarget.style.background = T.sidebarSoft; }} onMouseLeave={(e) => { if (!active) e.currentTarget.style.background = "transparent"; }}>
                    {nn.n && <span style={{ width: 19, height: 19, borderRadius: 6, background: active ? "rgba(255,255,255,.2)" : T.sidebarSoft, color: active ? "#fff" : "#A89E94", fontSize: 11, fontWeight: 700, display: "inline-flex", alignItems: "center", justifyContent: "center", flexShrink: 0 }}>{nn.n}</span>}
                    <nn.icon size={17} /> {nn.label}
                  </button>); })}
              </div>
            ))}
          </nav>
          <div style={{ padding: 16, borderTop: `1px solid ${T.sidebarSoft}`, fontSize: 11, color: "#7E756C" }}>Pricing: H × W (cm) × ₹/cm² × insertions</div>
        </aside>
        <div style={{ flex: 1, display: "flex", flexDirection: "column", minWidth: 0 }}>
          <header className="no-print" style={{ height: 60, background: T.card, borderBottom: `1px solid ${T.line}`, display: "flex", alignItems: "center", justifyContent: "space-between", padding: "0 22px", flexShrink: 0 }}>
            <div style={{ display: "flex", alignItems: "center", gap: 12 }}><button onClick={() => setSidebarOpen(!sidebarOpen)} style={{ background: T.paper, border: `1px solid ${T.line}`, borderRadius: 8, padding: 8, cursor: "pointer", color: T.ink, display: "inline-flex" }}><Menu size={17} /></button><div style={{ fontSize: 13.5, color: T.inkSoft }}>{db.settings.company}</div></div>
            <div style={{ display: "flex", alignItems: "center", gap: 10 }}><span style={{ fontSize: 12.5, color: T.inkSoft, display: "flex", alignItems: "center", gap: 6 }}><Clock size={14} /> {fmtDate(today())}</span><div style={{ width: 34, height: 34, borderRadius: "50%", background: T.ink, color: T.paper, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 13, fontWeight: 700 }}>AG</div></div>
          </header>
          <main style={{ flex: 1, overflowY: "auto", padding: "26px 28px" }}><Current /></main>
        </div>
      </div>
    </ERPCtx.Provider>
  );
}
