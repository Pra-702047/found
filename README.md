import React, { useEffect, useMemo, useRef, useState } from "react";

/**
 * Lost & Found Portal – Single‑File React App (Demo)
 * --------------------------------------------------
 * ✅ Features in this demo:
 * - Post Lost/Found items with images (stored locally in browser)
 * - Search & filter by type, category, location, date range
 * - Smart (demo) AI similarity matching on text fields
 * - Safe contact flow (mask contact, in‑app message)
 * - Simple moderation: banned words filter, rate limiting, flagging, admin review
 * - Image preview, drag & drop upload, pagination
 * - Admin mode (password: "admin") to approve/ban/delete
 *
 * ⚠️ Notes:
 * - This is a front‑end only demo. Data persists to localStorage.
 * - Replace local storage + mock AI with your backend/AI of choice later.
 */

// ----------------------------- Utility helpers -----------------------------

const LS_KEYS = {
  ITEMS: "lfp.items",
  MSGS: "lfp.messages",
  RATE: "lfp.rateLimiter",
  ADMIN: "lfp.adminMode",
};

const CATEGORIES = [
  "Electronics",
  "ID Card",
  "Keys",
  "Wallet",
  "Books",
  "Clothing",
  "Accessories",
  "Stationery",
  "Others",
];

const BANNED = ["scam", "bitcoin", "crypto double", "xxx", "adult", "loan shark"];

const nowISO = () => new Date().toISOString().slice(0, 10);
const uid = () => Math.random().toString(36).slice(2) + Date.now().toString(36);

function saveLS(key, value) {
  localStorage.setItem(key, JSON.stringify(value));
}
function loadLS(key, fallback) {
  try {
    const v = JSON.parse(localStorage.getItem(key));
    return v ?? fallback;
  } catch {
    return fallback;
  }
}

function clamp(n, a, b) { return Math.max(a, Math.min(b, n)); }

// Cosine similarity on term frequency vectors (very small demo)
function textSim(a, b) {
  const tokenize = (s) => (s || "").toLowerCase().replace(/[^a-z0-9\s]/g, " ").split(/\s+/).filter(Boolean);
  const wa = new Map();
  const wb = new Map();
  for (const t of tokenize(a)) wa.set(t, (wa.get(t) || 0) + 1);
  for (const t of tokenize(b)) wb.set(t, (wb.get(t) || 0) + 1);
  // dot
  let dot = 0;
  for (const [t, v] of wa) if (wb.has(t)) dot += v * wb.get(t);
  const mag = (w) => Math.sqrt(Array.from(w.values()).reduce((s, x) => s + x * x, 0));
  const ma = mag(wa), mb = mag(wb);
  return ma && mb ? dot / (ma * mb) : 0;
}

function maskContact(value) {
  if (!value) return "";
  if (value.includes("@")) {
    const [u, d] = value.split("@");
    return `${u.slice(0, 2)}***@${d}`;
  }
  return value.replace(/.(?=.{4}$)/g, "*");
}

// ------------------------------- Components --------------------------------

export default function LostFoundApp() {
  const [items, setItems] = useState(() => loadLS(LS_KEYS.ITEMS, sampleSeed()));
  const [messages, setMessages] = useState(() => loadLS(LS_KEYS.MSGS, {})); // {itemId: [{id,from,text,date}]}
  const [query, setQuery] = useState("");
  const [typeFilter, setTypeFilter] = useState("all"); // all | lost | found
  const [categoryFilter, setCategoryFilter] = useState("all");
  const [locationFilter, setLocationFilter] = useState("");
  const [fromDate, setFromDate] = useState("");
  const [toDate, setToDate] = useState("");
  const [showForm, setShowForm] = useState(false);
  const [adminMode, setAdminMode] = useState(() => loadLS(LS_KEYS.ADMIN, false));
  const [activeTab, setActiveTab] = useState("browse"); // browse | inbox | admin
  const [toast, setToast] = useState(null);

  useEffect(() => saveLS(LS_KEYS.ITEMS, items), [items]);
  useEffect(() => saveLS(LS_KEYS.MSGS, messages), [messages]);
  useEffect(() => saveLS(LS_KEYS.ADMIN, adminMode), [adminMode]);

  const filtered = useMemo(() => {
    const q = query.trim().toLowerCase();
    const fd = fromDate ? new Date(fromDate).getTime() : -Infinity;
    const td = toDate ? new Date(toDate).getTime() : Infinity;
    return items
      .filter((it) => (typeFilter === "all" ? true : it.type === typeFilter))
      .filter((it) => (categoryFilter === "all" ? true : it.category === categoryFilter))
      .filter((it) => (locationFilter ? it.location.toLowerCase().includes(locationFilter.toLowerCase()) : true))
      .filter((it) => (!fromDate && !toDate ? true : (new Date(it.date).getTime() >= fd && new Date(it.date).getTime() <= td)))
      .filter((it) =>
        q
          ? [it.title, it.description, it.location, it.category].join(" ").toLowerCase().includes(q)
          : true
      )
      .sort((a, b) => new Date(b.createdAt) - new Date(a.createdAt));
  }, [items, query, typeFilter, categoryFilter, locationFilter, fromDate, toDate]);

  const approveItem = (id, approved) => setItems((prev) => prev.map((x) => (x.id === id ? { ...x, approved } : x)));
  const deleteItem = (id) => setItems((prev) => prev.filter((x) => x.id !== id));
  const toggleFlag = (id) => setItems((p) => p.map((x) => (x.id === id ? { ...x, flagged: !x.flagged } : x)));

  const sendMessage = (itemId, from, text) => {
    const msg = { id: uid(), from, text, date: new Date().toISOString() };
    setMessages((m) => ({ ...m, [itemId]: [...(m[itemId] || []), msg] }));
    setToast({ type: "success", text: "Message sent (demo). Owner can approve reveal." });
  };

  const pendingApproval = items.filter((i) => !i.approved);

  return (
    <div className="min-h-screen bg-gradient-to-b from-slate-50 to-slate-100 text-slate-900">
      <header className="sticky top-0 z-30 backdrop-blur bg-white/70 border-b border-slate-200">
        <div className="max-w-7xl mx-auto px-4 py-4 flex items-center justify-between">
          <div className="flex items-center gap-3">
            <Logo />
            <div>
              <h1 className="text-xl font-semibold">Lost & Found Portal</h1>
              <p className="text-xs text-slate-500 -mt-0.5">Campus / Community demo</p>
            </div>
          </div>
          <div className="flex items-center gap-2">
            <button
              className="px-3 py-2 rounded-xl bg-slate-900 text-white text-sm shadow hover:opacity-90"
              onClick={() => setShowForm(true)}
            >
              + Post Item
            </button>
            <AdminToggle adminMode={adminMode} setAdminMode={setAdminMode} />
          </div>
        </div>
      </header>

      <main className="max-w-7xl mx-auto px-4 py-6">
        <Tabs activeTab={activeTab} onChange={setActiveTab} counts={{
          browse: filtered.length,
          inbox: Object.values(messages).reduce((a, arr) => a + arr.length, 0),
          admin: pendingApproval.length,
        }} />

        {activeTab === "browse" && (
          <>
            <SearchBar
              query={query}
              setQuery={setQuery}
              typeFilter={typeFilter}
              setTypeFilter={setTypeFilter}
              categoryFilter={categoryFilter}
              setCategoryFilter={setCategoryFilter}
              locationFilter={locationFilter}
              setLocationFilter={setLocationFilter}
              fromDate={fromDate}
              toDate={toDate}
              setFromDate={setFromDate}
              setToDate={setToDate}
            />

            <ItemGrid
              items={filtered}
              onContact={(item) => {
                const text = prompt("Write a short message to the owner/finder:") || "Hi! I think this matches my item.";
                if (!text) return;
                sendMessage(item.id, "You", text);
              }}
              onFindMatches={(seed) => {
                const matches = rankMatches(seed, items).slice(0, 5);
                setToast({ type: "info", text: `Found ${matches.length} suggested match(es). Scroll to view.` });
                // Optional: highlight – we simply move likely matches to top visually by setting a transient field
                setItems((prev) => prev.map((it) => ({ ...it, boost: matches.find((m) => m.id === it.id)?.score || 0 })));
              }}
              adminMode={adminMode}
              approveItem={approveItem}
              deleteItem={deleteItem}
              toggleFlag={toggleFlag}
            />
          </>
        )}

        {activeTab === "inbox" && (
          <Inbox items={items} messages={messages} onAllowReveal={(itemId) => {
            setItems((prev) => prev.map((i) => i.id === itemId ? { ...i, revealApproved: true } : i));
            setToast({ type: "success", text: "Contact reveal approved for this item (demo)." });
          }} />
        )}

        {activeTab === "admin" && (
          <AdminPanel
            items={items}
            approveItem={approveItem}
            deleteItem={deleteItem}
            toggleFlag={toggleFlag}
          />
        )}
      </main>

      {showForm && (
        <PostItemModal
          onClose={() => setShowForm(false)}
          onSubmit={(payload) => {
            // Moderation: banned words
            const textBlob = `${payload.title} ${payload.description}`.toLowerCase();
            if (BANNED.some((w) => textBlob.includes(w))) {
              setToast({ type: "error", text: "Post blocked by content filter." });
              return;
            }
            // Rate limit: 5 posts/hour per device
            const rate = loadLS(LS_KEYS.RATE, []);
            const cutoff = Date.now() - 60 * 60 * 1000;
            const recent = rate.filter((t) => t > cutoff);
            if (recent.length >= 5) {
              setToast({ type: "error", text: "Rate limit reached. Try later." });
              return;
            }
            const item = {
              id: uid(),
              type: payload.type,
              title: payload.title.trim(),
              category: payload.category,
              description: payload.description.trim(),
              date: payload.date || nowISO(),
              location: payload.location.trim(),
              images: payload.images,
              contact: { name: payload.name.trim(), method: payload.contactMethod, value: payload.contactValue.trim() },
              approved: false,
              flagged: false,
              createdAt: new Date().toISOString(),
            };

            // Smart matches (demo)
            const suggestions = rankMatches(item, items.filter((i) => i.type !== item.type));
            if (suggestions.length) {
              setToast({ type: "info", text: `We found ${suggestions.length} potential match(es). Admin will review.` });
            }

            setItems((prev) => [item, ...prev]);
            saveLS(LS_KEYS.RATE, [...recent, Date.now()]);
            setShowForm(false);
          }}
        />
      )}

      {toast && (
        <Toast {...toast} onClose={() => setToast(null)} />
      )}

      <Footer />
    </div>
  );
}

// ------------------------------ Subcomponents -------------------------------

function Logo() {
  return (
    <div className="w-10 h-10 rounded-2xl bg-slate-900 text-white grid place-content-center shadow">
      <span className="font-bold">L&F</span>
    </div>
  );
}

function Tabs({ activeTab, onChange, counts }) {
  const tabs = [
    { id: "browse", label: `Browse (${counts.browse})` },
    { id: "inbox", label: `Inbox (${counts.inbox})` },
    { id: "admin", label: `Admin (${counts.admin})` },
  ];
  return (
    <div className="flex gap-2 mb-4">
      {tabs.map((t) => (
        <button
          key={t.id}
          onClick={() => onChange(t.id)}
          className={`px-3 py-2 rounded-xl text-sm border shadow-sm ${activeTab === t.id ? "bg-slate-900 text-white" : "bg-white"}`}
        >
          {t.label}
        </button>
      ))}
    </div>
  );
}

function SearchBar({ query, setQuery, typeFilter, setTypeFilter, categoryFilter, setCategoryFilter, locationFilter, setLocationFilter, fromDate, toDate, setFromDate, setToDate }) {
  return (
    <div className="grid md:grid-cols-6 gap-3 bg-white p-4 rounded-2xl shadow mb-6 border">
      <input value={query} onChange={(e) => setQuery(e.target.value)} placeholder="Search keywords…" className="md:col-span-2 px-3 py-2 rounded-xl border" />
      <select value={typeFilter} onChange={(e) => setTypeFilter(e.target.value)} className="px-3 py-2 rounded-xl border">
        <option value="all">All</option>
        <option value="lost">Lost</option>
        <option value="found">Found</option>
      </select>
      <select value={categoryFilter} onChange={(e) => setCategoryFilter(e.target.value)} className="px-3 py-2 rounded-xl border">
        <option value="all">All Categories</option>
        {CATEGORIES.map((c) => <option key={c} value={c}>{c}</option>)}
      </select>
      <input value={locationFilter} onChange={(e) => setLocationFilter(e.target.value)} placeholder="Location contains…" className="px-3 py-2 rounded-xl border" />
      <div className="grid grid-cols-2 gap-2">
        <input type="date" value={fromDate} onChange={(e) => setFromDate(e.target.value)} className="px-3 py-2 rounded-xl border" />
        <input type="date" value={toDate} onChange={(e) => setToDate(e.target.value)} className="px-3 py-2 rounded-xl border" />
      </div>
    </div>
  );
}

function ItemGrid({ items, onContact, onFindMatches, adminMode, approveItem, deleteItem, toggleFlag }) {
  const [page, setPage] = useState(1);
  const perPage = 9;
  const total = Math.max(1, Math.ceil(items.length / perPage));
  const view = useMemo(() => items
    .slice()
    .sort((a, b) => (b.boost || 0) - (a.boost || 0) || new Date(b.createdAt) - new Date(a.createdAt))
    .slice((page - 1) * perPage, page * perPage), [items, page]);

  useEffect(() => { if (page > total) setPage(total); }, [total]);

  return (
    <div>
      {items.length === 0 && (
        <div className="p-8 text-center text-slate-500">No items match your filters yet.</div>
      )}
      <div className="grid sm:grid-cols-2 lg:grid-cols-3 gap-4">
        {view.map((it) => (
          <ItemCard key={it.id} item={it}
            onContact={() => onContact(it)}
            onFindMatches={() => onFindMatches(it)}
            adminMode={adminMode}
            approveItem={approveItem}
            deleteItem={deleteItem}
            toggleFlag={toggleFlag}
          />
        ))}
      </div>
      {items.length > perPage && (
        <div className="mt-4 flex items-center justify-center gap-2">
          <button className="px-3 py-1 rounded-lg border bg-white" onClick={() => setPage((p) => clamp(p - 1, 1, total))}>Prev</button>
          <span className="text-sm text-slate-600">Page {page} / {total}</span>
          <button className="px-3 py-1 rounded-lg border bg-white" onClick={() => setPage((p) => clamp(p + 1, 1, total))}>Next</button>
        </div>
      )}
    </div>
  );
}

function ItemCard({ item, onContact, onFindMatches, adminMode, approveItem, deleteItem, toggleFlag }) {
  const cover = item.images?.[0];
  return (
    <div className={`rounded-2xl border shadow-sm bg-white overflow-hidden ${item.flagged ? 'ring-2 ring-rose-400' : ''}`}>
      {cover ? (
        <img src={cover} alt="item" className="w-full h-40 object-cover" />
      ) : (
        <div className={`w-full h-40 grid place-content-center ${item.type === 'lost' ? 'bg-amber-50' : 'bg-emerald-50'}`}>
          <span className="text-slate-400">No image</span>
        </div>
      )}
      <div className="p-4">
        <div className="flex items-center justify-between mb-1">
          <span className={`px-2 py-0.5 rounded-lg text-xs ${item.type === 'lost' ? 'bg-amber-100 text-amber-800' : 'bg-emerald-100 text-emerald-800'}`}>{item.type.toUpperCase()}</span>
          {!item.approved && <span className="text-[10px] px-2 py-0.5 rounded bg-slate-100">Pending approval</span>}
        </div>
        <h3 className="font-semibold">{item.title}</h3>
        <p className="text-xs text-slate-500 mt-0.5">{item.category} • {item.location} • {item.date}</p>
        <p className="text-sm text-slate-700 mt-2 line-clamp-3">{item.description}</p>
        <div className="mt-3 flex flex-wrap gap-2">
          <button onClick={onFindMatches} className="px-3 py-1.5 rounded-xl border bg-white text-sm">Find Matches</button>
          <button onClick={onContact} className="px-3 py-1.5 rounded-xl bg-slate-900 text-white text-sm">Safe Contact</button>
          {item.revealApproved && (
            <span title="Owner approved reveal" className="px-2 py-1 rounded-lg bg-green-50 text-green-700 text-xs">Contact: {item.contact.method} {maskContact(item.contact.value)}</span>
          )}
        </div>

        {adminMode && (
          <div className="mt-3 flex flex-wrap gap-2 border-t pt-3">
            <button onClick={() => approveItem(item.id, !item.approved)} className="px-2 py-1 rounded-lg text-xs border bg-white">{item.approved ? 'Unapprove' : 'Approve'}</button>
            <button onClick={() => toggleFlag(item.id)} className="px-2 py-1 rounded-lg text-xs border bg-rose-50 text-rose-700">{item.flagged ? 'Unflag' : 'Flag'}</button>
            <button onClick={() => deleteItem(item.id)} className="px-2 py-1 rounded-lg text-xs border bg-slate-50 text-slate-700">Delete</button>
          </div>
        )}
      </div>
    </div>
  );
}

function PostItemModal({ onClose, onSubmit }) {
  const [type, setType] = useState("lost");
  const [title, setTitle] = useState("");
  const [category, setCategory] = useState(CATEGORIES[0]);
  const [description, setDescription] = useState("");
  const [date, setDate] = useState(nowISO());
  const [location, setLocation] = useState("");
  const [images, setImages] = useState([]);
  const [name, setName] = useState("");
  const [contactMethod, setContactMethod] = useState("email");
  const [contactValue, setContactValue] = useState("");
  const [agree, setAgree] = useState(false);

  const fileRef = useRef();

  const onFiles = async (files) => {
    const list = Array.from(files).slice(0, 4);
    const datas = [];
    for (const f of list) {
      const data = await fileToDataURL(f);
      datas.push(data);
    }
    setImages((prev) => [...prev, ...datas].slice(0, 4));
  };

  const canSubmit = title && description && location && name && contactValue && agree;

  return (
    <div className="fixed inset-0 z-40 grid place-items-center bg-black/40 p-4" onClick={onClose}>
      <div className="w-full max-w-2xl bg-white rounded-2xl shadow-xl border overflow-hidden" onClick={(e) => e.stopPropagation()}>
        <div className="px-5 py-3 border-b flex items-center justify-between bg-slate-50">
          <h2 className="font-semibold">Post Lost/Found Item</h2>
          <button className="text-slate-500" onClick={onClose}>✕</button>
        </div>
        <div className="p-5 grid gap-4">
          <div className="grid grid-cols-2 gap-3">
            <div className="grid gap-1">
              <label className="text-sm">Type</label>
              <select value={type} onChange={(e) => setType(e.target.value)} className="px-3 py-2 rounded-xl border">
                <option value="lost">Lost</option>
                <option value="found">Found</option>
              </select>
            </div>
            <div className="grid gap-1">
              <label className="text-sm">Category</label>
              <select value={category} onChange={(e) => setCategory(e.target.value)} className="px-3 py-2 rounded-xl border">
                {CATEGORIES.map((c) => <option key={c} value={c}>{c}</option>)}
              </select>
            </div>
          </div>

          <div className="grid md:grid-cols-2 gap-3">
            <div className="grid gap-1">
              <label className="text-sm">Title</label>
              <input value={title} onChange={(e) => setTitle(e.target.value)} placeholder="e.g., Black wallet with zipper" className="px-3 py-2 rounded-xl border" />
            </div>
            <div className="grid gap-1">
              <label className="text-sm">Location</label>
              <input value={location} onChange={(e) => setLocation(e.target.value)} placeholder="e.g., Library ground floor" className="px-3 py-2 rounded-xl border" />
            </div>
          </div>

          <div className="grid gap-1">
            <label className="text-sm">Description</label>
            <textarea value={description} onChange={(e) => setDescription(e.target.value)} rows={3} placeholder="Brand, color, distinct marks, contents…" className="px-3 py-2 rounded-xl border" />
          </div>

          <div className="grid md:grid-cols-3 gap-3">
            <div className="grid gap-1">
              <label className="text-sm">Date</label>
              <input type="date" value={date} onChange={(e) => setDate(e.target.value)} className="px-3 py-2 rounded-xl border" />
            </div>
            <div className="md:col-span-2 grid gap-1">
              <label className="text-sm">Upload images (max 4)</label>
              <div
                onDragOver={(e) => e.preventDefault()}
                onDrop={(e) => { e.preventDefault(); onFiles(e.dataTransfer.files); }}
                className="rounded-xl border border-dashed p-3 text-sm text-slate-500 bg-slate-50"
              >
                Drag & drop or <button className="underline" onClick={() => fileRef.current?.click()} type="button">browse</button>
                <input ref={fileRef} type="file" accept="image/*" multiple hidden onChange={(e) => onFiles(e.target.files)} />
                <div className="mt-3 flex gap-2 flex-wrap">
                  {images.map((src, i) => (
                    <div key={i} className="relative">
                      <img src={src} alt="preview" className="w-20 h-20 object-cover rounded-lg border" />
                      <button type="button" onClick={() => setImages((prev) => prev.filter((_, j) => j !== i))} className="absolute -top-2 -right-2 bg-white border rounded-full w-6 h-6 grid place-content-center shadow">×</button>
                    </div>
                  ))}
                </div>
              </div>
            </div>
          </div>

          <div className="grid md:grid-cols-3 gap-3">
            <div className="grid gap-1">
              <label className="text-sm">Your name</label>
              <input value={name} onChange={(e) => setName(e.target.value)} className="px-3 py-2 rounded-xl border" />
            </div>
            <div className="grid gap-1">
              <label className="text-sm">Preferred contact</label>
              <select value={contactMethod} onChange={(e) => setContactMethod(e.target.value)} className="px-3 py-2 rounded-xl border">
                <option value="email">Email</option>
                <option value="phone">Phone</option>
              </select>
            </div>
            <div className="grid gap-1">
              <label className="text-sm">Contact detail</label>
              <input value={contactValue} onChange={(e) => setContactValue(e.target.value)} placeholder="Will be masked by default" className="px-3 py-2 rounded-xl border" />
            </div>
          </div>

          <label className="flex items-center gap-2 text-sm">
            <input type="checkbox" checked={agree} onChange={(e) => setAgree(e.target.checked)} />
            I agree to community guidelines and allow admins to moderate my post.
          </label>

          <div className="flex items-center justify-end gap-2 pt-2">
            <button onClick={onClose} className="px-3 py-2 rounded-xl border bg-white">Cancel</button>
            <button
              onClick={() => canSubmit && onSubmit({ type, title, category, description, date, location, images, name, contactMethod, contactValue })}
              className={`px-4 py-2 rounded-xl text-white ${canSubmit ? 'bg-slate-900' : 'bg-slate-400 cursor-not-allowed'}`}
              disabled={!canSubmit}
            >
              Submit
            </button>
          </div>
        </div>
      </div>
    </div>
  );
}

function Inbox({ items, messages, onAllowReveal }) {
  const owned = items.filter((i) => i.approved); // assume approved posts belong to you in demo
  return (
    <div className="grid gap-4">
      {owned.map((it) => (
        <div key={it.id} className="bg-white rounded-2xl border shadow-sm p-4">
          <div className="flex items-center justify-between">
            <div>
              <h3 className="font-semibold">{it.title}</h3>
              <p className="text-xs text-slate-500">{it.type.toUpperCase()} • {it.location} • {it.date}</p>
            </div>
            <div className="flex items-center gap-2">
              <button onClick={() => onAllowReveal(it.id)} className="px-3 py-1.5 rounded-xl border bg-white text-sm">Allow contact reveal</button>
            </div>
          </div>
          <div className="mt-3 grid gap-2">
            {(messages[it.id] || []).length === 0 && (
              <p className="text-sm text-slate-500">No messages yet.</p>
            )}
            {(messages[it.id] || []).map((m) => (
              <div key={m.id} className="text-sm">
                <span className="text-slate-500">{new Date(m.date).toLocaleString()}</span>
                <div className="mt-1 p-2 rounded-lg bg-slate-50 border">{m.text}</div>
              </div>
            ))}
          </div>
        </div>
      ))}
    </div>
  );
}

function AdminPanel({ items, approveItem, deleteItem, toggleFlag }) {
  const pending = items.filter((i) => !i.approved);
  const flagged = items.filter((i) => i.flagged);
  return (
    <div className="grid lg:grid-cols-2 gap-4">
      <section className="bg-white rounded-2xl border shadow-sm p-4">
        <h3 className="font-semibold mb-2">Pending approval ({pending.length})</h3>
        <div className="grid gap-3">
          {pending.map((i) => (
            <div key={i.id} className="p-3 rounded-xl border">
              <div className="flex items-center justify-between">
                <div>
                  <p className="font-medium">{i.title} <span className="text-xs text-slate-500">({i.type}/{i.category})</span></p>
                  <p className="text-xs text-slate-500">{i.location} • {i.date}</p>
                </div>
                <div className="flex gap-2">
                  <button onClick={() => approveItem(i.id, true)} className="px-2 py-1 rounded-lg text-xs border bg-emerald-50 text-emerald-700">Approve</button>
                  <button onClick={() => toggleFlag(i.id)} className="px-2 py-1 rounded-lg text-xs border bg-rose-50 text-rose-700">{i.flagged ? 'Unflag' : 'Flag'}</button>
                  <button onClick={() => deleteItem(i.id)} className="px-2 py-1 rounded-lg text-xs border bg-slate-50">Delete</button>
                </div>
              </div>
              <p className="text-sm text-slate-700 mt-2 line-clamp-2">{i.description}</p>
            </div>
          ))}
          {pending.length === 0 && <p className="text-sm text-slate-500">Nothing pending.</p>}
        </div>
      </section>
      <section className="bg-white rounded-2xl border shadow-sm p-4">
        <h3 className="font-semibold mb-2">Flagged posts ({flagged.length})</h3>
        <div className="grid gap-3">
          {flagged.map((i) => (
            <div key={i.id} className="p-3 rounded-xl border">
              <div className="flex items-center justify-between">
                <div>
                  <p className="font-medium">{i.title}</p>
                  <p className="text-xs text-slate-500">{i.location} • {i.date}</p>
                </div>
                <div className="flex gap-2">
                  <button onClick={() => toggleFlag(i.id)} className="px-2 py-1 rounded-lg text-xs border bg-white">Unflag</button>
                  <button onClick={() => deleteItem(i.id)} className="px-2 py-1 rounded-lg text-xs border bg-slate-50">Delete</button>
                </div>
              </div>
              <p className="text-sm text-slate-700 mt-2 line-clamp-2">{i.description}</p>
            </div>
          ))}
          {flagged.length === 0 && <p className="text-sm text-slate-500">No flagged posts.</p>}
        </div>
      </section>
    </div>
  );
}

function AdminToggle({ adminMode, setAdminMode }) {
  const [open, setOpen] = useState(false);
  const [pass, setPass] = useState("");
  return (
    <>
      <button
        className={`px-3 py-2 rounded-xl border text-sm ${adminMode ? 'bg-green-50 text-green-800' : 'bg-white'}`}
        onClick={() => setOpen(true)}
      >
        {adminMode ? 'Admin: ON' : 'Admin Login'}
      </button>
      {open && (
        <div className="fixed inset-0 z-40 grid place-items-center bg-black/40 p-4" onClick={() => setOpen(false)}>
          <div className="w-full max-w-sm bg-white rounded-2xl shadow-xl border overflow-hidden" onClick={(e) => e.stopPropagation()}>
            <div className="px-5 py-3 border-b bg-slate-50">
              <h2 className="font-semibold">Admin Authentication</h2>
            </div>
            <div className="p-5 grid gap-3">
              <input type="password" value={pass} onChange={(e) => setPass(e.target.value)} placeholder="Password is 'admin' in demo" className="px-3 py-2 rounded-xl border" />
              <div className="flex items-center justify-end gap-2">
                <button onClick={() => setOpen(false)} className="px-3 py-2 rounded-xl border bg-white">Cancel</button>
                <button onClick={() => { if (pass === 'admin') { setAdminMode(true); setOpen(false); } }} className="px-4 py-2 rounded-xl bg-slate-900 text-white">Login</button>
                {adminMode && <button onClick={() => { setAdminMode(false); setOpen(false); }} className="px-3 py-2 rounded-xl border bg-white">Logout</button>}
              </div>
            </div>
          </div>
        </div>
      )}
    </>
  );
}

function Toast({ type = "info", text, onClose }) {
  useEffect(() => {
    const t = setTimeout(onClose, 3500);
    return () => clearTimeout(t);
  }, [onClose]);
  return (
    <div className={`fixed bottom-4 right-4 px-4 py-2 rounded-xl shadow-lg text-sm ${
      type === 'error' ? 'bg-rose-600 text-white' : type === 'success' ? 'bg-emerald-600 text-white' : 'bg-slate-900 text-white'
    }`}>
      <div className="flex items-center gap-3">
        <span>{text}</span>
        <button className="opacity-80" onClick={onClose}>×</button>
      </div>
    </div>
  );
}

function Footer() {
  return (
    <footer className="mt-10 pb-10 text-center text-xs text-slate-500">
      Built as a front‑end demo. Replace localStorage with your backend (e.g., Node/Express + Postgres),
      and wire AI (e.g., vector DB + embedding) for production.
    </footer>
  );
}

// ------------------------------ Demo Data/AI -------------------------------

function sampleSeed() {
  const seed = [
    {
      id: uid(),
      type: "lost",
      title: "Black Leather Wallet",
      category: "Wallet",
      description: "Black wallet with zipper, contains college ID and a photo.",
      date: nowISO(),
      location: "Library Ground Floor",
      images: [],
      contact: { name: "Aarav", method: "email", value: "aarav@example.com" },
      approved: true,
      flagged: false,
      createdAt: new Date().toISOString(),
    },
    {
      id: uid(),
      type: "found",
      title: "Set of Keys with Blue Tag",
      category: "Keys",
      description: "3 keys on a ring with a blue silicone tag.",
      date: nowISO(),
      location: "Cafeteria",
      images: [],
      contact: { name: "Riya", method: "phone", value: "+91 98765 43210" },
      approved: true,
      flagged: false,
      createdAt: new Date().toISOString(),
    },
    {
      id: uid(),
      type: "found",
      title: "Brown Wallet with Stitching",
      category: "Wallet",
      description: "Brown leather wallet, no cash, has metro card.",
      date: nowISO(),
      location: "Library Entrance",
      images: [],
      contact: { name: "Kabir", method: "email", value: "kabir@example.com" },
      approved: true,
      flagged: false,
      createdAt: new Date().toISOString(),
    },
  ];
  return seed;
}

function fileToDataURL(file) {
  return new Promise((res, rej) => {
    const fr = new FileReader();
    fr.onload = () => res(fr.result);
    fr.onerror = rej;
    fr.readAsDataURL(file);
  });
}

// Rank potential matches (opposite type) by simple text + meta similarity
function rankMatches(seed, pool) {
  const others = pool || [];
  const seedBlob = [seed.title, seed.description, seed.category, seed.location].join(" ");
  const scored = others
    .filter((o) => o.type !== seed.type)
    .map((o) => {
      const blob = [o.title, o.description, o.category, o.location].join(" ");
      let score = textSim(seedBlob, blob);
      if (o.category === seed.category) score += 0.15;
      // Date proximity (within 7 days)
      const d1 = new Date(seed.date).getTime();
      const d2 = new Date(o.date).getTime();
      const diff = Math.abs(d1 - d2) / (1000 * 60 * 60 * 24);
      if (diff <= 7) score += 0.1;
      return { id: o.id, score };
    })
    .filter((s) => s.score > 0.15)
    .sort((a, b) => b.score - a.score);
  return scored;
}
