# avto-tap
Salam xoş gəlmisiniz, Bura maşınınızın elanını yerləşdirərək qısa müddətdə sata bilərsiz, və qısa müddətdə özünüzə maşın tapa bilərsiz
import React, { useState, useEffect } from "react";

// AvtoTap - single-file React component (TailwindCSS assumed)
// Features:
// - Add / Edit car listings with fields (brand, model, year, fuel, price, type)
// - Free up to 10 listings per user (tracked in localStorage). From 11th onward each listing costs 1 AZN.
// - Promote (boost) a car to appear at the top. Promotion simulates a payment of 1 AZN.
// - Persistent storage via localStorage (demo-ready).
// - Simple filters and sorting (promoted & newest first).

export default function AvtoTapApp() {
  const STORAGE_KEY = "avtotap_listings_v1";
  const COUNT_KEY = "avtotap_user_post_count_v1";

  const [listings, setListings] = useState([]);
  const [form, setForm] = useState({
    brand: "",
    model: "",
    year: "",
    fuel: "",
    price: "",
    type: "salgil" // sale or rent or other
  });
  const [query, setQuery] = useState("");
  const [filterYear, setFilterYear] = useState("");
  const [userPostCount, setUserPostCount] = useState(0);
  const [message, setMessage] = useState(null);
  const [editingId, setEditingId] = useState(null);
  const [showPayment, setShowPayment] = useState(false);
  const [pendingAction, setPendingAction] = useState(null); // {type: 'post'|'promote', payload}

  useEffect(() => {
    const stored = JSON.parse(localStorage.getItem(STORAGE_KEY) || "[]");
    setListings(stored);
    const ct = parseInt(localStorage.getItem(COUNT_KEY) || "0", 10);
    setUserPostCount(ct);
  }, []);

  useEffect(() => {
    localStorage.setItem(STORAGE_KEY, JSON.stringify(listings));
  }, [listings]);

  useEffect(() => {
    localStorage.setItem(COUNT_KEY, String(userPostCount));
  }, [userPostCount]);

  function resetForm() {
    setForm({ brand: "", model: "", year: "", fuel: "", price: "", type: "salgil" });
    setEditingId(null);
  }

  function handleChange(e) {
    const { name, value } = e.target;
    setForm(prev => ({ ...prev, [name]: value }));
  }

  function validateForm() {
    if (!form.brand || !form.model || !form.year || !form.price) return false;
    if (isNaN(Number(form.year)) || isNaN(Number(form.price))) return false;
    return true;
  }

  function tryCreateListing() {
    // Pricing rule: first 10 listings by this user are free. From 11th onward each new listing costs 1 AZN.
    const FREE_LIMIT = 10;
    if (!validateForm()) {
      setMessage({ type: "error", text: "Zəhmət olmasa bütün sahələri düzgün doldurun." });
      return;
    }

    if (userPostCount >= FREE_LIMIT) {
      // require payment for posting
      setPendingAction({ type: "post", payload: { ...form } });
      setShowPayment(true);
      return;
    }

    // free posting
    const newListing = {
      id: Date.now(),
      ...form,
      promoted: false,
      createdAt: new Date().toISOString()
    };
    setListings(prev => [newListing, ...prev]);
    setUserPostCount(c => c + 1);
    resetForm();
    setMessage({ type: "success", text: "Elan uğurla əlavə edildi (ödənişsiz)." });
  }

  function saveEdit() {
    if (!validateForm()) {
      setMessage({ type: "error", text: "Zəhmət olmasa bütün sahələri düzgün doldurun." });
      return;
    }
    setListings(prev => prev.map(l => (l.id === editingId ? { ...l, ...form } : l)));
    setMessage({ type: "success", text: "Elan yeniləndi." });
    resetForm();
  }

  function editListing(id) {
    const l = listings.find(x => x.id === id);
    if (!l) return;
    setForm({ brand: l.brand, model: l.model, year: l.year, fuel: l.fuel, price: l.price, type: l.type });
    setEditingId(id);
    window.scrollTo({ top: 0, behavior: "smooth" });
  }

  function deleteListing(id) {
    if (!confirm("Elanı silmək istədiyinizə əminsiniz?")) return;
    setListings(prev => prev.filter(l => l.id !== id));
    setMessage({ type: "success", text: "Elan silindi." });
  }

  function promptPromote(listing) {
    // promoting costs 1 AZN (simulated)
    setPendingAction({ type: "promote", payload: { id: listing.id } });
    setShowPayment(true);
  }

  function processPayment(amountAzn) {
    // Simulate a payment processing delay
    setShowPayment(false);
    const act = pendingAction;
    setPendingAction(null);
    if (!act) return;

    if (act.type === "post") {
      // create listing after payment
      const newListing = {
        id: Date.now(),
        ...act.payload,
        promoted: false,
        createdAt: new Date().toISOString()
      };
      setListings(prev => [newListing, ...prev]);
      setUserPostCount(c => c + 1);
      resetForm();
      setMessage({ type: "success", text: `Ödəniş uğurlu oldu: ${amountAzn} AZN. Elan yerləşdirildi.` });
    } else if (act.type === "promote") {
      const id = act.payload.id;
      setListings(prev => {
        return prev
          .map(l => (l.id === id ? { ...l, promoted: true, promotedAt: new Date().toISOString() } : l))
          .sort((a, b) => {
            // promoted first by promotedAt, then newest
            if (a.promoted && b.promoted) return new Date(b.promotedAt) - new Date(a.promotedAt);
            if (a.promoted) return -1;
            if (b.promoted) return 1;
            return new Date(b.createdAt) - new Date(a.createdAt);
          });
      });
      setMessage({ type: "success", text: `Ödəniş uğurlu oldu: ${amountAzn} AZN. Elan irəli çəkildi.` });
    }
  }

  const visible = listings
    .filter(l => {
      if (query && !(`${l.brand} ${l.model}`.toLowerCase().includes(query.toLowerCase()))) return false;
      if (filterYear && String(l.year) !== String(filterYear)) return false;
      return true;
    })
    .sort((a, b) => {
      // promoted first (by promotedAt), then newest
      if (a.promoted && b.promoted) return new Date(b.promotedAt) - new Date(a.promotedAt);
      if (a.promoted) return -1;
      if (b.promoted) return 1;
      return new Date(b.createdAt) - new Date(a.createdAt);
    });

  return (
    <div className="min-h-screen bg-gray-50 p-4">
      <div className="max-w-5xl mx-auto">
        <header className="flex items-center justify-between mb-6">
          <h1 className="text-2xl font-bold">AvtoTap — Avtomobil elan platforması</h1>
          <div className="text-sm text-gray-600">Sizin pulsuz elan limitiniz: <strong>{Math.max(0, 10 - userPostCount)}</strong> (ilk 10 pulsuz), sonrakılar: <strong>1 AZN</strong> hər biri</div>
        </header>

        {/* Form */}
        <section className="bg-white p-4 rounded shadow mb-6">
          <h2 className="text-lg font-semibold mb-3">Yeni elan əlavə et</h2>
          <div className="grid grid-cols-1 md:grid-cols-3 gap-3">
            <input name="brand" value={form.brand} onChange={handleChange} placeholder="Marka (məs: Toyota)" className="p-2 border rounded" />
            <input name="model" value={form.model} onChange={handleChange} placeholder="Model (məs: Corolla)" className="p-2 border rounded" />
            <input name="year" value={form.year} onChange={handleChange} placeholder="İl (məs: 2018)" className="p-2 border rounded" />
            <input name="fuel" value={form.fuel} onChange={handleChange} placeholder="Yanacaq (məs: Benzin/Dizel)" className="p-2 border rounded" />
            <input name="price" value={form.price} onChange={handleChange} placeholder="Qiymət (AZN)" className="p-2 border rounded" />
            <select name="type" value={form.type} onChange={handleChange} className="p-2 border rounded">
              <option value="salgil">Satılır</option>
              <option value="kiraye">Kirayə</option>
              <option value="deyish">Dəyişilir</option>
            </select>
          </div>

          <div className="mt-3 flex gap-2">
            {editingId ? (
              <>
                <button onClick={saveEdit} className="px-4 py-2 bg-blue-600 text-white rounded">Yadda saxla</button>
                <button onClick={resetForm} className="px-4 py-2 border rounded">Ləğv et</button>
              </>
            ) : (
              <button onClick={tryCreateListing} className="px-4 py-2 bg-green-600 text-white rounded">Elan əlavə et</button>
            )}
            <button onClick={() => { setForm({ brand: "Toyota", model: "Corolla", year: "2018", fuel: "Benzin", price: "18000", type: "salgil" }) }} className="px-4 py-2 border rounded">Nümunə doldur</button>
          </div>
          <p className="text-sm text-gray-500 mt-2">Qeyd: 10-dan çox elan yerləşdirmək istəyirsinizsə, hər yeni elan üçün 1 AZN ödənişi tələb olunur. Elanı irəli çəkmək (promote) üçün də 1 AZN ödəyərək onu siyahının önünə çıxara bilərsiniz.</p>
        </section>

        {/* Filters + message */}
        <section className="mb-4 flex items-center gap-3">
          <input placeholder="Marka və ya model axtar" value={query} onChange={e => setQuery(e.target.value)} className="p-2 border rounded w-full" />
          <input placeholder="İl filtri" value={filterYear} onChange={e => setFilterYear(e.target.value)} className="p-2 border rounded w-32" />
          <button onClick={() => { setQuery(""); setFilterYear(""); }} className="px-3 py-2 border rounded">Təmizlə</button>
        </section>

        {message && (
          <div className={`p-3 rounded mb-4 ${message.type === "success" ? "bg-green-50 border border-green-200" : "bg-red-50 border border-red-200"}`}>
            {message.text}
          </div>
        )}

        {/* Listings */}
        <section className="grid grid-cols-1 md:grid-cols-2 gap-4">
          {visible.length === 0 && <div className="text-gray-600">Heç bir elan yoxdur — ilk elanınızı yerləşdirin!</div>}

          {visible.map(l => (
            <article key={l.id} className={`bg-white p-4 rounded shadow ${l.promoted ? "ring-4 ring-yellow-200" : ""}`}>
              <div className="flex justify-between items-start">
                <div>
                  <h3 className="text-lg font-semibold">{l.brand} {l.model} <span className="text-sm text-gray-500">({l.year})</span></h3>
                  <div className="text-sm text-gray-600">{l.fuel} • {l.type}</div>
                </div>
                <div className="text-right">
                  <div className="text-xl font-bold">{l.price} AZN</div>
                  {l.promoted && <div className="text-xs text-yellow-700">Promoted</div>}
                </div>
              </div>

              <div className="mt-3 flex gap-2">
                <button onClick={() => editListing(l.id)} className="px-3 py-1 border rounded">Redaktə et</button>
                <button onClick={() => deleteListing(l.id)} className="px-3 py-1 border rounded">Sil</button>
                <button onClick={() => promptPromote(l)} className="px-3 py-1 bg-yellow-400 rounded">İrəli çək (1 AZN)</button>
              </div>

              <div className="mt-2 text-xs text-gray-500">Elan yerləşdirilib: {new Date(l.createdAt).toLocaleString()}</div>
            </article>
          ))}
        </section>

        {/* Payment modal (simulated) */}
        {showPayment && (
          <div className="fixed inset-0 bg-black bg-opacity-40 flex items-center justify-center p-4">
            <div className="bg-white rounded p-6 w-full max-w-md">
              <h3 className="text-lg font-semibold mb-3">Ödəniş</h3>
              <p className="mb-3">Seçilmiş əməliyyat üçün 1 AZN ödənişi tələb olunur. Bu demo versiyada ödəniş sadəcə təsdiqlənir (simulated).</p>
              <div className="flex gap-2 justify-end">
                <button onClick={() => { setShowPayment(false); setPendingAction(null); }} className="px-4 py-2 border rounded">Ləğv et</button>
                <button onClick={() => processPayment(1)} className="px-4 py-2 bg-blue-600 text-white rounded">Ödənişi təsdiqlə (1 AZN)</button>
              </div>
            </div>
          </div>
        )}

        <footer className="mt-8 text-center text-sm text-gray-500">AvtoTap demo — canlı istifadə üçün ödəniş ağacları və ödəniş qapısı inteqrasiyası tələb olunur.</footer>
      </div>
    </div>
  );
}
