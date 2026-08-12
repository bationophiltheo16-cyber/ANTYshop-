import React, { useState, useEffect } from "react";

/* =============================================================================
 * ANTYshop — boutique de pagnes en ligne
 * =============================================================================
 * Site simple, un seul vendeur : catalogue de pagnes + commande directe sur
 * WhatsApp par article (pas de panier), espace admin protégé par mot de passe
 * pour gérer les produits. Même architecture que BK Traffic (React + Vite +
 * Supabase + Vercel), volontairement allégée puisqu'il n'y a qu'une seule
 * vendeuse — pas de marketplace, pas de livraison par zone, pas de paiement
 * en ligne intégré.
 * ============================================================================= */

/* ---------- CONFIG ---------- */
const WHATSAPP_NUMBER = "22656675099"; // +226 Burkina Faso — à corriger si le pays diffère
const ADMIN_PASSWORD = "antyshop2026"; // à changer une fois le site en ligne

const CATEGORIES = ["Tous", "Wax hollandais", "Bazin riche", "Kente", "Bogolan", "Autres"];

/* ---------- SUPABASE ---------- */
// Remplacer par les vraies valeurs de VOTRE projet Supabase (Project Settings > API)
const SUPABASE_URL = "https://VOTRE-PROJET.supabase.co";
const SUPABASE_ANON_KEY = "VOTRE_CLE_ANON_PUBLIC";

const sbHeaders = (extra = {}) => ({
  apikey: SUPABASE_ANON_KEY,
  Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
  "Content-Type": "application/json",
  ...extra,
});

async function fetchProducts() {
  try {
    const res = await fetch(`${SUPABASE_URL}/rest/v1/products?select=*&order=created_at.desc`, { headers: sbHeaders() });
    if (!res.ok) throw new Error("fetch failed");
    return await res.json();
  } catch (e) {
    console.error("Erreur chargement produits", e);
    return null;
  }
}
async function insertProductRow(product) {
  try {
    const res = await fetch(`${SUPABASE_URL}/rest/v1/products`, {
      method: "POST",
      headers: sbHeaders({ Prefer: "return=representation" }),
      body: JSON.stringify([product]),
    });
    if (!res.ok) return null;
    const data = await res.json();
    return data[0];
  } catch (e) {
    console.error("Erreur ajout produit", e);
    return null;
  }
}
async function updateProductRow(id, patch) {
  try {
    await fetch(`${SUPABASE_URL}/rest/v1/products?id=eq.${encodeURIComponent(id)}`, {
      method: "PATCH",
      headers: sbHeaders({ Prefer: "return=minimal" }),
      body: JSON.stringify(patch),
    });
  } catch (e) {
    console.error("Erreur modification produit", e);
  }
}
async function deleteProductRow(id) {
  try {
    await fetch(`${SUPABASE_URL}/rest/v1/products?id=eq.${encodeURIComponent(id)}`, {
      method: "DELETE",
      headers: sbHeaders(),
    });
  } catch (e) {
    console.error("Erreur suppression produit", e);
  }
}
async function uploadImage(file) {
  try {
    const ext = (file.name.split(".").pop() || "jpg").toLowerCase();
    const path = `products/${Date.now()}-${Math.random().toString(36).slice(2, 8)}.${ext}`;
    const res = await fetch(`${SUPABASE_URL}/storage/v1/object/antyshop-images/${path}`, {
      method: "POST",
      headers: {
        apikey: SUPABASE_ANON_KEY,
        Authorization: `Bearer ${SUPABASE_ANON_KEY}`,
        "Content-Type": file.type || "application/octet-stream",
      },
      body: file,
    });
    if (!res.ok) throw new Error("upload failed");
    return `${SUPABASE_URL}/storage/v1/object/public/antyshop-images/${path}`;
  } catch (e) {
    console.error("Erreur upload image", e);
    return null;
  }
}

const fmt = (n) => new Intl.NumberFormat("fr-FR").format(n) + " FCFA";

/* ---------- ICÔNES ---------- */
const Icon = {
  search: (p) => (
    <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}>
      <circle cx="11" cy="11" r="6.5" />
      <path d="m20 20-4.3-4.3" strokeLinecap="round" />
    </svg>
  ),
  chat: (p) => (
    <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}>
      <path d="M4 12a8 8 0 1 1 3.2 6.4L4 20l1.3-3.6A7.96 7.96 0 0 1 4 12Z" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  ),
  lock: (p) => (
    <svg viewBox="0 0 24 24" width="15" height="15" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}>
      <rect x="5" y="10.5" width="14" height="9" rx="1.5" />
      <path d="M8 10.5V8a4 4 0 0 1 8 0v2.5" strokeLinecap="round" />
    </svg>
  ),
  plus: (p) => (
    <svg viewBox="0 0 24 24" width="14" height="14" fill="none" stroke="currentColor" strokeWidth="2.2" {...p}>
      <path d="M12 5v14M5 12h14" strokeLinecap="round" />
    </svg>
  ),
  trash: (p) => (
    <svg viewBox="0 0 24 24" width="16" height="16" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}>
      <path d="M4 7h16M9 7V5a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2m-8 0 1 13a1 1 0 0 0 1 1h6a1 1 0 0 0 1-1l1-13" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  ),
  back: (p) => (
    <svg viewBox="0 0 24 24" width="18" height="18" fill="none" stroke="currentColor" strokeWidth="1.8" {...p}>
      <path d="M15 5 8 12l7 7" strokeLinecap="round" strokeLinejoin="round" />
    </svg>
  ),
};

/* ---------- SIGNATURE VISUELLE : liseré façon lisière de pagne ---------- */
function FabricEdge({ color = "#1B1F3B" }) {
  return (
    <svg viewBox="0 0 200 10" preserveAspectRatio="none" style={{ width: "100%", height: 10, display: "block" }}>
      <polyline
        points="0,5 10,0 20,10 30,0 40,10 50,0 60,10 70,0 80,10 90,0 100,10 110,0 120,10 130,0 140,10 150,0 160,10 170,0 180,10 190,0 200,5"
        fill="none"
        stroke={color}
        strokeWidth="1.6"
      />
    </svg>
  );
}

/* ---------- APP ---------- */
export default function App() {
  const [view, setView] = useState("shop"); // shop | admin-login | admin
  const [products, setProducts] = useState([]);
  const [loading, setLoading] = useState(true);
  const [category, setCategory] = useState("Tous");
  const [search, setSearch] = useState("");
  const [pwd, setPwd] = useState("");
  const [pwdError, setPwdError] = useState(false);
  const [editingProduct, setEditingProduct] = useState(null);

  useEffect(() => {
    (async () => {
      const stored = await fetchProducts();
      if (stored) setProducts(stored);
      setLoading(false);
    })();
  }, []);

  const filtered = products
    .filter((p) => category === "Tous" || p.category === category)
    .filter((p) => {
      if (!search.trim()) return true;
      const q = search.trim().toLowerCase();
      return (p.name || "").toLowerCase().includes(q) || (p.description || "").toLowerCase().includes(q);
    });

  function orderOnWhatsapp(p) {
    const msg = `Bonjour ANTYshop, je suis intéressé(e) par :\n\n${p.name}\n${fmt(p.price)}\n\nEst-il toujours disponible ?`;
    window.open(`https://wa.me/${WHATSAPP_NUMBER}?text=${encodeURIComponent(msg)}`, "_blank");
  }

  function tryAdminLogin() {
    if (pwd === ADMIN_PASSWORD) {
      setPwdError(false);
      setPwd("");
      setView("admin");
    } else {
      setPwdError(true);
    }
  }

  async function handleSaveProduct(p) {
    if (p.id) {
      const { id, ...rest } = p;
      await updateProductRow(id, rest);
      setProducts((prev) => prev.map((x) => (x.id === p.id ? p : x)));
    } else {
      const newProduct = { ...p, id: undefined };
      delete newProduct.id;
      const saved = await insertProductRow(newProduct);
      if (saved) setProducts((prev) => [saved, ...prev]);
    }
    setEditingProduct(null);
  }

  async function handleDeleteProduct(id) {
    setProducts((prev) => prev.filter((p) => p.id !== id));
    await deleteProductRow(id);
  }

  const font = {
    display: "'Unbounded', 'Arial Black', sans-serif",
    body: "'Work Sans', -apple-system, sans-serif",
  };

  const INK = "#1B1F3B";
  const IVORY = "#FBF6EE";
  const MAGENTA = "#E1257A";
  const YELLOW = "#F2B705";

  if (loading) {
    return (
      <div style={{ minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", background: IVORY, fontFamily: font.body, color: INK }}>
        Chargement…
      </div>
    );
  }

  return (
    <div style={{ fontFamily: font.body, background: IVORY, color: INK, minHeight: "100vh", paddingBottom: 40 }}>
      <style>{`
        @import url('https://fonts.googleapis.com/css2?family=Unbounded:wght@700;800;900&family=Work+Sans:wght@400;500;600;700&display=swap');
        * { box-sizing: border-box; }
        button { font-family: inherit; cursor: pointer; }
        input, textarea, select { font-family: inherit; }
        ::selection { background: ${MAGENTA}; color: #fff; }
      `}</style>

      {view === "shop" && (
        <ShopView
          products={filtered}
          category={category}
          setCategory={setCategory}
          search={search}
          setSearch={setSearch}
          onOrder={orderOnWhatsapp}
          onOpenAdmin={() => setView("admin-login")}
          font={font}
          colors={{ INK, IVORY, MAGENTA, YELLOW }}
        />
      )}

      {view === "admin-login" && (
        <AdminLogin
          pwd={pwd}
          setPwd={setPwd}
          error={pwdError}
          onSubmit={tryAdminLogin}
          onBack={() => setView("shop")}
          font={font}
          colors={{ INK, IVORY, MAGENTA, YELLOW }}
        />
      )}

      {view === "admin" && (
        <AdminView
          products={products}
          onEdit={setEditingProduct}
          onDelete={handleDeleteProduct}
          onNew={() => setEditingProduct({ name: "", category: CATEGORIES[1], price: "", image: "", description: "", available: true })}
          onLogout={() => setView("shop")}
          font={font}
          colors={{ INK, IVORY, MAGENTA, YELLOW }}
        />
      )}

      {editingProduct && (
        <ProductEditor
          product={editingProduct}
          onSave={handleSaveProduct}
          onCancel={() => setEditingProduct(null)}
          font={font}
          colors={{ INK, IVORY, MAGENTA, YELLOW }}
        />
      )}
    </div>
  );
}

/* ---------- SHOP VIEW ---------- */
function ShopView({ products, category, setCategory, search, setSearch, onOrder, onOpenAdmin, font, colors }) {
  const { INK, IVORY, MAGENTA, YELLOW } = colors;
  return (
    <div>
      {/* Header */}
      <div style={{ padding: "32px 20px 0", textAlign: "center" }}>
        <div style={{ fontFamily: font.display, fontWeight: 900, fontSize: 34, letterSpacing: -0.5, color: INK }}>
          ANTY<span style={{ color: MAGENTA }}>shop</span>
        </div>
        <div style={{ fontSize: 12, color: "#6b6558", letterSpacing: 1.5, textTransform: "uppercase", marginTop: 4 }}>
          Pagnes &amp; tissus — Burkina Faso
        </div>
      </div>

      <div style={{ padding: "18px 0" }}>
        <FabricEdge color={INK} />
      </div>

      {/* Search */}
      <div style={{ padding: "0 20px 16px" }}>
        <div style={{ position: "relative" }}>
          <span style={{ position: "absolute", left: 14, top: "50%", transform: "translateY(-50%)", color: "#9a9484" }}>
            <Icon.search />
          </span>
          <input
            value={search}
            onChange={(e) => setSearch(e.target.value)}
            placeholder="Rechercher un pagne…"
            style={{ width: "100%", border: `1.5px solid ${INK}22`, borderRadius: 14, padding: "13px 16px 13px 42px", fontSize: 14, background: "#fff" }}
          />
        </div>
      </div>

      {/* Categories */}
      <div style={{ display: "flex", gap: 8, overflowX: "auto", padding: "0 20px 20px" }}>
        {CATEGORIES.map((c) => (
          <button
            key={c}
            onClick={() => setCategory(c)}
            style={{
              flex: "0 0 auto",
              padding: "8px 16px",
              borderRadius: 20,
              border: `1.5px solid ${INK}`,
              background: category === c ? INK : "#fff",
              color: category === c ? "#fff" : INK,
              fontSize: 12.5,
              fontWeight: 600,
              whiteSpace: "nowrap",
            }}
          >
            {c}
          </button>
        ))}
      </div>

      {/* Product grid */}
      <div style={{ padding: "0 16px", display: "grid", gridTemplateColumns: "1fr 1fr", gap: 14 }}>
        {products.length === 0 && (
          <div style={{ gridColumn: "1 / -1", textAlign: "center", color: "#9a9484", fontSize: 13.5, padding: "50px 20px" }}>
            Aucun pagne ne correspond à votre recherche.
          </div>
        )}
        {products.map((p) => {
          const isAvailable = p.available !== false;
          return (
            <div key={p.id} style={{ background: "#fff", borderRadius: 16, overflow: "hidden", border: `1px solid ${INK}14`, opacity: isAvailable ? 1 : 0.6 }}>
              <div style={{ width: "100%", aspectRatio: "4 / 5", background: "#F0EBDF", overflow: "hidden", position: "relative" }}>
                {p.image && <img src={p.image} alt={p.name} style={{ width: "100%", height: "100%", objectFit: "cover" }} />}
                {!isAvailable && (
                  <span style={{ position: "absolute", top: 8, left: 8, background: INK, color: "#fff", fontSize: 9.5, fontWeight: 700, padding: "4px 8px", borderRadius: 8 }}>
                    VENDU
                  </span>
                )}
              </div>
              <div style={{ padding: "10px 12px 12px" }}>
                <div style={{ fontSize: 9.5, letterSpacing: 0.6, color: MAGENTA, textTransform: "uppercase", fontWeight: 700, marginBottom: 3 }}>{p.category}</div>
                <div style={{ fontSize: 13, fontWeight: 600, lineHeight: 1.3, marginBottom: 5, minHeight: 34 }}>{p.name}</div>
                <div style={{ fontSize: 14, fontWeight: 800, marginBottom: 8, color: INK }}>{fmt(p.price)}</div>
                <button
                  onClick={() => isAvailable && onOrder(p)}
                  disabled={!isAvailable}
                  style={{
                    width: "100%",
                    padding: "9px 0",
                    borderRadius: 10,
                    border: "none",
                    background: isAvailable ? MAGENTA : "#e5e0d3",
                    color: isAvailable ? "#fff" : "#9a9484",
                    fontSize: 12,
                    fontWeight: 700,
                    display: "flex",
                    alignItems: "center",
                    justifyContent: "center",
                    gap: 5,
                  }}
                >
                  <Icon.chat style={{ width: 14, height: 14 }} /> {isAvailable ? "Commander" : "Vendu"}
                </button>
              </div>
            </div>
          );
        })}
      </div>

      <div style={{ padding: "40px 20px 10px" }}>
        <FabricEdge color={INK} />
      </div>

      <div style={{ textAlign: "center", padding: "20px 20px 0" }}>
        <button
          onClick={onOpenAdmin}
          style={{ background: "none", border: "none", color: "#9a9484", fontSize: 11.5, display: "inline-flex", alignItems: "center", gap: 5 }}
        >
          <Icon.lock /> Espace admin
        </button>
      </div>
    </div>
  );
}

/* ---------- ADMIN LOGIN ---------- */
function AdminLogin({ pwd, setPwd, error, onSubmit, onBack, font, colors }) {
  const { INK, MAGENTA } = colors;
  return (
    <div style={{ minHeight: "100vh", display: "flex", flexDirection: "column", justifyContent: "center", padding: "20px 28px" }}>
      <button onClick={onBack} style={{ background: "none", border: "none", display: "flex", marginBottom: 30, color: INK }}><Icon.back /></button>
      <div style={{ display: "flex", justifyContent: "center", marginBottom: 18 }}>
        <div style={{ width: 52, height: 52, borderRadius: "50%", background: INK, color: "#fff", display: "flex", alignItems: "center", justifyContent: "center" }}>
          <Icon.lock />
        </div>
      </div>
      <h2 style={{ fontFamily: font.display, fontWeight: 800, fontSize: 19, textAlign: "center", margin: "0 0 6px", color: INK }}>Espace admin</h2>
      <p style={{ fontSize: 12.5, color: "#9a9484", textAlign: "center", margin: "0 0 24px" }}>Réservé à ANTYshop</p>
      <input
        type="password"
        value={pwd}
        onChange={(e) => setPwd(e.target.value)}
        onKeyDown={(e) => e.key === "Enter" && onSubmit()}
        placeholder="Mot de passe"
        style={{ border: `1.5px solid ${error ? "#c0392b" : INK + "33"}`, borderRadius: 12, padding: "14px 16px", fontSize: 14, marginBottom: error ? 8 : 18, textAlign: "center" }}
      />
      {error && <div style={{ fontSize: 12, color: "#c0392b", textAlign: "center", marginBottom: 14 }}>Mot de passe incorrect.</div>}
      <button onClick={onSubmit} style={{ padding: "14px 0", borderRadius: 12, border: "none", background: MAGENTA, color: "#fff", fontSize: 13.5, fontWeight: 700 }}>
        Entrer
      </button>
    </div>
  );
}

/* ---------- ADMIN VIEW ---------- */
function AdminView({ products, onEdit, onDelete, onNew, onLogout, font, colors }) {
  const { INK, MAGENTA } = colors;
  const [confirmDelete, setConfirmDelete] = useState(null);

  return (
    <div style={{ padding: "24px 20px 30px" }}>
      <div style={{ display: "flex", alignItems: "center", justifyContent: "space-between", marginBottom: 20 }}>
        <h2 style={{ fontFamily: font.display, fontWeight: 800, fontSize: 19, margin: 0, color: INK }}>Mes pagnes</h2>
        <button
          onClick={() => setConfirmDelete({ logout: true })}
          style={{ background: "none", border: `1.5px solid ${INK}33`, borderRadius: 20, padding: "6px 14px", fontSize: 11.5, color: INK }}
        >
          Quitter
        </button>
      </div>

      <button
        onClick={onNew}
        style={{ width: "100%", padding: "12px 0", borderRadius: 12, border: `1.5px dashed ${INK}44`, background: "#fff", color: INK, fontSize: 12.5, fontWeight: 600, marginBottom: 16, display: "flex", alignItems: "center", justifyContent: "center", gap: 6 }}
      >
        <Icon.plus /> Ajouter un pagne
      </button>

      {products.length === 0 && (
        <div style={{ textAlign: "center", color: "#9a9484", fontSize: 13, padding: "30px 20px" }}>Aucun pagne pour l'instant.</div>
      )}

      <div style={{ display: "flex", flexDirection: "column", gap: 10 }}>
        {products.map((p) => (
          <div key={p.id} style={{ display: "flex", gap: 10, background: "#fff", border: `1px solid ${INK}14`, borderRadius: 14, padding: 10, alignItems: "center" }}>
            <div style={{ width: 46, height: 46, borderRadius: 10, overflow: "hidden", background: "#F0EBDF", flex: "0 0 auto" }}>
              {p.image && <img src={p.image} alt={p.name} style={{ width: "100%", height: "100%", objectFit: "cover" }} />}
            </div>
            <div style={{ flex: 1, minWidth: 0 }}>
              <div style={{ fontSize: 13, fontWeight: 600, whiteSpace: "nowrap", overflow: "hidden", textOverflow: "ellipsis" }}>{p.name}</div>
              <div style={{ fontSize: 12, color: "#6b6558" }}>{fmt(p.price)} · {p.category} {p.available === false ? "· Vendu" : ""}</div>
            </div>
            <button onClick={() => onEdit(p)} style={{ background: "none", border: `1.5px solid ${INK}33`, borderRadius: 8, padding: "6px 10px", fontSize: 11 }}>Modifier</button>
            <button onClick={() => setConfirmDelete({ id: p.id, name: p.name })} style={{ background: "none", border: "none", color: "#c0392b" }}><Icon.trash /></button>
          </div>
        ))}
      </div>

      {confirmDelete && (
        <div style={{ position: "fixed", inset: 0, background: "rgba(27,31,59,0.5)", display: "flex", alignItems: "center", justifyContent: "center", zIndex: 60, padding: 24 }}>
          <div style={{ background: "#fff", borderRadius: 16, padding: "22px 20px", maxWidth: 300, width: "100%" }}>
            <p style={{ fontSize: 14, color: INK, lineHeight: 1.5, margin: "0 0 18px" }}>
              {confirmDelete.logout ? "Voulez-vous vraiment quitter l'espace admin ?" : `Supprimer "${confirmDelete.name}" ? Cette action est définitive.`}
            </p>
            <div style={{ display: "flex", gap: 10 }}>
              <button onClick={() => setConfirmDelete(null)} style={{ flex: 1, padding: "12px 0", borderRadius: 10, border: `1.5px solid ${INK}33`, background: "#fff", color: INK, fontSize: 13, fontWeight: 600 }}>
                Annuler
              </button>
              <button
                onClick={() => {
                  if (confirmDelete.logout) onLogout();
                  else onDelete(confirmDelete.id);
                  setConfirmDelete(null);
                }}
                style={{ flex: 1, padding: "12px 0", borderRadius: 10, border: "none", background: confirmDelete.logout ? INK : "#c0392b", color: "#fff", fontSize: 13, fontWeight: 700 }}
              >
                {confirmDelete.logout ? "Quitter" : "Supprimer"}
              </button>
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

/* ---------- PRODUCT EDITOR (modal) ---------- */
function ProductEditor({ product, onSave, onCancel, font, colors }) {
  const { INK, MAGENTA } = colors;
  const [form, setForm] = useState(product);
  const [uploading, setUploading] = useState(false);
  const set = (k) => (e) => setForm((f) => ({ ...f, [k]: e.target.value }));

  const inputStyle = { border: `1.5px solid ${INK}22`, borderRadius: 10, padding: "11px 12px", fontSize: 13.5, width: "100%" };

  async function handleFile(e) {
    const file = e.target.files?.[0];
    if (!file) return;
    setUploading(true);
    const url = await uploadImage(file);
    setUploading(false);
    if (url) setForm((f) => ({ ...f, image: url }));
    else alert("L'envoi de la photo a échoué, réessayez.");
  }

  return (
    <div style={{ position: "fixed", inset: 0, background: "rgba(27,31,59,0.5)", display: "flex", alignItems: "flex-end", zIndex: 50 }}>
      <div style={{ background: "#fff", borderRadius: "20px 20px 0 0", padding: "22px 20px 30px", width: "100%", maxHeight: "88vh", overflowY: "auto" }}>
        <h3 style={{ fontFamily: font.display, fontWeight: 800, fontSize: 17, margin: "0 0 18px", color: INK }}>
          {product.id ? "Modifier le pagne" : "Nouveau pagne"}
        </h3>
        <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
          <div>
            <label style={{ fontSize: 11, color: "#9a9484", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Nom</label>
            <input value={form.name} onChange={set("name")} style={inputStyle} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9484", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Catégorie</label>
            <select value={form.category} onChange={set("category")} style={inputStyle}>
              {CATEGORIES.filter((c) => c !== "Tous").map((c) => <option key={c} value={c}>{c}</option>)}
            </select>
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9484", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Prix (FCFA)</label>
            <input type="number" value={form.price} onChange={set("price")} style={inputStyle} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9484", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Photo</label>
            <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
              <div style={{ width: 56, height: 56, borderRadius: 10, overflow: "hidden", background: "#F0EBDF", flex: "0 0 auto" }}>
                {form.image && <img src={form.image} alt="" style={{ width: "100%", height: "100%", objectFit: "cover" }} />}
              </div>
              <label style={{ padding: "10px 14px", borderRadius: 10, border: `1.5px solid ${INK}`, background: "#fff", color: INK, fontSize: 12, fontWeight: 700, cursor: "pointer" }}>
                {uploading ? "Envoi…" : "Choisir une photo"}
                <input type="file" accept="image/*" onChange={handleFile} style={{ display: "none" }} />
              </label>
            </div>
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9484", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Description</label>
            <textarea rows={3} value={form.description} onChange={set("description")} style={{ ...inputStyle, resize: "vertical" }} />
          </div>
          <div>
            <label style={{ fontSize: 11, color: "#9a9484", letterSpacing: 0.4, textTransform: "uppercase", display: "block", marginBottom: 5 }}>Disponibilité</label>
            <div style={{ display: "flex", gap: 8 }}>
              <button
                type="button"
                onClick={() => setForm((f) => ({ ...f, available: true }))}
                style={{ flex: 1, padding: "10px 0", borderRadius: 10, border: `1.5px solid ${INK}`, background: form.available !== false ? INK : "#fff", color: form.available !== false ? "#fff" : INK, fontSize: 12.5, fontWeight: 700 }}
              >
                Disponible
              </button>
              <button
                type="button"
                onClick={() => setForm((f) => ({ ...f, available: false }))}
                style={{ flex: 1, padding: "10px 0", borderRadius: 10, border: `1.5px solid ${INK}`, background: form.available === false ? INK : "#fff", color: form.available === false ? "#fff" : INK, fontSize: 12.5, fontWeight: 700 }}
              >
                Vendu
              </button>
            </div>
          </div>
        </div>
        <div style={{ display: "flex", gap: 10, marginTop: 20 }}>
          <button onClick={onCancel} style={{ flex: 1, padding: "13px 0", borderRadius: 12, border: `1.5px solid ${INK}33`, background: "#fff", color: INK, fontSize: 13, fontWeight: 600 }}>Annuler</button>
          <button
            onClick={() => onSave({ ...form, price: Number(form.price) || 0 })}
            style={{ flex: 1, padding: "13px 0", borderRadius: 12, border: "none", background: MAGENTA, color: "#fff", fontSize: 13, fontWeight: 700 }}
          >
            Enregistrer
          </button>
        </div>
      </div>
    </div>
  );
}
