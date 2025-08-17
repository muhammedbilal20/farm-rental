import { useEffect, useMemo, useState } from "react";

// Simple, single-file React app that lets owners add farm equipment (tractor, trailer, harvester, etc.)
// with their own rates and location. Renters can search/filter and use "near me" to find equipment.
// Data is stored in localStorage for demo. Replace with your backend later.

function haversineKm(lat1, lon1, lat2, lon2) {
  const R = 6371; // km
  const toRad = (d) => (d * Math.PI) / 180;
  const dLat = toRad(lat2 - lat1);
  const dLon = toRad(lon2 - lon1);
  const a =
    Math.sin(dLat / 2) * Math.sin(dLat / 2) +
    Math.cos(toRad(lat1)) * Math.cos(toRad(lat2)) *
      Math.sin(dLon / 2) * Math.sin(dLon / 2);
  const c = 2 * Math.atan2(Math.sqrt(a), Math.sqrt(1 - a));
  return R * c;
}

const CATEGORIES = [
  "Tractor",
  "Trailer",
  "Harvester",
  "Tiller",
  "Seeder",
  "Sprayer",
  "Baler",
  "Other",
];

const seedItems = [
  {
    id: "1",
    title: "Mahindra 575 Tractor",
    category: "Tractor",
    ratePerDay: 3500,
    locationName: "Near Village Market",
    lat: 12.9716,
    lon: 77.5946,
    image:
      "https://images.unsplash.com/photo-1609257349454-8d0e90a962f8?auto=format&fit=crop&w=1200&q=60",
    ownerName: "Ravi",
    ownerPhone: "+91-90000-11111",
    description: "Well-maintained tractor, ideal for ploughing and transport.",
  },
  {
    id: "2",
    title: "John Deere Harvester",
    category: "Harvester",
    ratePerDay: 9000,
    locationName: "Kallur Farm Gate",
    lat: 16.5062,
    lon: 80.6480,
    image:
      "https://images.unsplash.com/photo-1590086782792-42dd2350140b?auto=format&fit=crop&w=1200&q=60",
    ownerName: "Sita",
    ownerPhone: "+91-90000-22222",
    description: "High-capacity combine harvester. Operator available on request.",
  },
  {
    id: "3",
    title: "Twin Axle Trailer",
    category: "Trailer",
    ratePerDay: 1800,
    locationName: "Canal Road Yard",
    lat: 19.0760,
    lon: 72.8777,
    image:
      "https://images.unsplash.com/photo-1516116216624-53e697fedbea?auto=format&fit=crop&w=1200&q=60",
    ownerName: "Rahul",
    ownerPhone: "+91-90000-33333",
    description: "Sturdy trailer for crop transport, 3-ton capacity.",
  },
];

function getStored() {
  try {
    const raw = localStorage.getItem("farm_rent_items");
    if (!raw) {
      localStorage.setItem("farm_rent_items", JSON.stringify(seedItems));
      return seedItems;
    }
    return JSON.parse(raw);
  } catch (e) {
    return seedItems;
  }
}

function setStored(items) {
  localStorage.setItem("farm_rent_items", JSON.stringify(items));
}

function useLocalItems() {
  const [items, setItems] = useState(getStored());
  useEffect(() => setStored(items), [items]);
  return [items, setItems];
}

export default function FarmEquipmentRental() {
  const [items, setItems] = useLocalItems();
  const [tab, setTab] = useState("browse"); // browse | add | my

  // renter filters
  const [query, setQuery] = useState("");
  const [category, setCategory] = useState("");
  const [minRate, setMinRate] = useState("");
  const [maxRate, setMaxRate] = useState("");
  const [sortBy, setSortBy] = useState("distance"); // distance | price | newest

  // renter geo
  const [myLoc, setMyLoc] = useState(null); // {lat, lon}
  const [maxKm, setMaxKm] = useState(50);

  // owner form
  const [form, setForm] = useState({
    title: "",
    category: "Tractor",
    ratePerDay: "",
    locationName: "",
    lat: "",
    lon: "",
    image: "",
    ownerName: "",
    ownerPhone: "",
    description: "",
  });

  // booking state (demo)
  const [bookingItem, setBookingItem] = useState(null);
  const [fromDate, setFromDate] = useState("");
  const [toDate, setToDate] = useState("");

  const itemsWithDistance = useMemo(() => {
    return items.map((it) => {
      let distanceKm = null;
      if (myLoc && typeof it.lat === "number" && typeof it.lon === "number") {
        distanceKm = haversineKm(myLoc.lat, myLoc.lon, it.lat, it.lon);
      }
      return { ...it, distanceKm };
    });
  }, [items, myLoc]);

  const filtered = useMemo(() => {
    return itemsWithDistance
      .filter((it) =>
        query
          ? it.title.toLowerCase().includes(query.toLowerCase()) ||
            it.description.toLowerCase().includes(query.toLowerCase())
          : true
      )
      .filter((it) => (category ? it.category === category : true))
      .filter((it) => (minRate ? it.ratePerDay >= Number(minRate) : true))
      .filter((it) => (maxRate ? it.ratePerDay <= Number(maxRate) : true))
      .filter((it) =>
        myLoc && it.distanceKm !== null ? it.distanceKm <= Number(maxKm) : true
      )
      .sort((a, b) => {
        if (sortBy === "price") return a.ratePerDay - b.ratePerDay;
        if (sortBy === "newest") return Number(b.id) - Number(a.id);
        // distance default
        const da = a.distanceKm ?? Infinity;
        const db = b.distanceKm ?? Infinity;
        return da - db;
      });
  }, [itemsWithDistance, query, category, minRate, maxRate, myLoc, maxKm, sortBy]);

  function useMyLocation() {
    if (!navigator.geolocation) {
      alert("Geolocation not supported in this browser.");
      return;
    }
    navigator.geolocation.getCurrentPosition(
      (pos) => setMyLoc({ lat: pos.coords.latitude, lon: pos.coords.longitude }),
      () => alert("Unable to fetch your location.")
    );
  }

  function fillOwnerCoords() {
    if (!navigator.geolocation) {
      alert("Geolocation not supported in this browser.");
      return;
    }
    navigator.geolocation.getCurrentPosition(
      (pos) =>
        setForm((f) => ({
          ...f,
          lat: pos.coords.latitude.toFixed(6),
          lon: pos.coords.longitude.toFixed(6),
        })),
      () => alert("Unable to fetch your location.")
    );
  }

  function addItem(e) {
    e.preventDefault();
    const required = [
      "title",
      "category",
      "ratePerDay",
      "locationName",
      "lat",
      "lon",
      "ownerName",
      "ownerPhone",
    ];
    for (const k of required) {
      if (!form[k]) {
        alert(`Please fill ${k}.`);
        return;
      }
    }
    const newItem = {
      id: String(Date.now()),
      title: form.title.trim(),
      category: form.category,
      ratePerDay: Number(form.ratePerDay),
      locationName: form.locationName.trim(),
      lat: Number(form.lat),
      lon: Number(form.lon),
      image:
        form.image ||
        "https://images.unsplash.com/photo-1542291026-7eec264c27ff?auto=format&fit=crop&w=1200&q=60",
      ownerName: form.ownerName.trim(),
      ownerPhone: form.ownerPhone.trim(),
      description: form.description.trim(),
    };
    setItems((prev) => [newItem, ...prev]);
    setForm({
      title: "",
      category: "Tractor",
      ratePerDay: "",
      locationName: "",
      lat: "",
      lon: "",
      image: "",
      ownerName: "",
      ownerPhone: "",
      description: "",
    });
    setTab("browse");
  }

  function openBooking(item) {
    setBookingItem(item);
    setFromDate("");
    setToDate("");
  }

  function confirmBooking() {
    if (!fromDate || !toDate) {
      alert("Please pick dates.");
      return;
    }
    const start = new Date(fromDate);
    const end = new Date(toDate);
    if (end < start) {
      alert("End date must be after start date.");
      return;
    }
    const days = Math.ceil((end - start) / (1000 * 60 * 60 * 24)) + 1; // inclusive
    const total = days * bookingItem.ratePerDay;
    alert(
      `Booking placed!\n\nItem: ${bookingItem.title}\nDates: ${fromDate} → ${toDate} (${days} days)\nTotal: ₹${total.toLocaleString()}\n\nContact owner to confirm: ${bookingItem.ownerName} (${bookingItem.ownerPhone})`
    );
    setBookingItem(null);
  }

  return (
    <div className="min-h-screen bg-gray-50">
      {/* Top bar */}
      <header className="sticky top-0 z-10 bg-white/90 backdrop-blur border-b border-gray-200">
        <div className="max-w-6xl mx-auto px-4 py-3 flex items-center gap-3">
          <span className="text-2xl font-bold text-blue-700">AgriRent</span>
          <nav className="ml-auto flex gap-2">
            <button
              className={`px-3 py-1.5 rounded-2xl text-sm ${
                tab === "browse"
                  ? "bg-blue-600 text-white"
                  : "bg-gray-100 text-gray-700"
              }`}
              onClick={() => setTab("browse")}
            >
              Browse / Rent
            </button>
            <button
              className={`px-3 py-1.5 rounded-2xl text-sm ${
                tab === "add" ? "bg-blue-600 text-white" : "bg-gray-100 text-gray-700"
              }`}
              onClick={() => setTab("add")}
            >
              Add Equipment (Owner)
            </button>
          </nav>
        </div>
      </header>

      <main className="max-w-6xl mx-auto p-4">
        {tab === "browse" && (
          <section>
            {/* Filters */}
            <div className="bg-white rounded-2xl shadow p-4 md:p-5 border border-gray-100">
              <div className="grid md:grid-cols-6 gap-3 items-end">
                <div className="md:col-span-2">
                  <label className="text-sm text-gray-600">Search</label>
                  <input
                    value={query}
                    onChange={(e) => setQuery(e.target.value)}
                    placeholder="tractor, trailer, harvester..."
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  />
                </div>
                <div>
                  <label className="text-sm text-gray-600">Category</label>
                  <select
                    value={category}
                    onChange={(e) => setCategory(e.target.value)}
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  >
                    <option value="">All</option>
                    {CATEGORIES.map((c) => (
                      <option key={c} value={c}>
                        {c}
                      </option>
                    ))}
                  </select>
                </div>
                <div>
                  <label className="text-sm text-gray-600">Min Rate (₹/day)</label>
                  <input
                    type="number"
                    value={minRate}
                    onChange={(e) => setMinRate(e.target.value)}
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  />
                </div>
                <div>
                  <label className="text-sm text-gray-600">Max Rate (₹/day)</label>
                  <input
                    type="number"
                    value={maxRate}
                    onChange={(e) => setMaxRate(e.target.value)}
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  />
                </div>
                <div>
                  <label className="text-sm text-gray-600">Sort By</label>
                  <select
                    value={sortBy}
                    onChange={(e) => setSortBy(e.target.value)}
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  >
                    <option value="distance">Nearest</option>
                    <option value="price">Price</option>
                    <option value="newest">Newest</option>
                  </select>
                </div>
                <div className="md:col-span-6 border-t pt-3 mt-1 grid md:grid-cols-4 gap-3">
                  <div className="md:col-span-2 flex items-end gap-2">
                    <div className="flex-1">
                      <label className="text-sm text-gray-600">Within (km)</label>
                      <input
                        type="number"
                        value={maxKm}
                        onChange={(e) => setMaxKm(e.target.value)}
                        className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                      />
                    </div>
                    <button
                      onClick={useMyLocation}
                      className="h-10 mt-6 px-3 rounded-xl bg-blue-600 text-white hover:bg-blue-700"
                    >
                      Use My Location
                    </button>
                    {myLoc && (
                      <span className="text-xs text-gray-600 mt-6">📍 {myLoc.lat.toFixed(3)},{" "}{myLoc.lon.toFixed(3)}</span>
                    )}
                  </div>
                </div>
              </div>
            </div>

            {/* Results */}
            <div className="grid sm:grid-cols-2 lg:grid-cols-3 gap-4 mt-5">
              {filtered.map((it) => (
                <article
                  key={it.id}
                  className="bg-white border border-gray-100 rounded-2xl shadow hover:shadow-md transition p-3 flex flex-col"
                >
                  <img
                    src={it.image}
                    alt={it.title}
                    className="w-full h-44 object-cover rounded-xl"
                    loading="lazy"
                  />
                  <div className="mt-3 flex-1">
                    <div className="flex items-center justify-between">
                      <h3 className="font-semibold text-gray-800">{it.title}</h3>
                      <span className="text-xs px-2 py-0.5 rounded-full bg-gray-100 text-gray-700">{it.category}</span>
                    </div>
                    <p className="text-sm text-gray-600 mt-1 line-clamp-2">{it.description}</p>
                    <div className="mt-2 text-sm text-gray-700">📍 {it.locationName}</div>
                    {it.distanceKm != null && (
                      <div className="text-sm text-gray-500">🧭 {it.distanceKm.toFixed(1)} km away</div>
                    )}
                  </div>
                  <div className="mt-3 flex items-center justify-between">
                    <div className="text-lg font-bold text-blue-700">₹{it.ratePerDay.toLocaleString()}<span className="text-sm text-gray-500">/day</span></div>
                    <button
                      onClick={() => openBooking(it)}
                      className="px-3 py-2 rounded-xl bg-blue-600 text-white hover:bg-blue-700"
                    >
                      Rent Now
                    </button>
                  </div>
                  <div className="mt-2 text-xs text-gray-600">Owner: {it.ownerName} · {it.ownerPhone}</div>
                </article>
              ))}
            </div>

            {filtered.length === 0 && (
              <p className="text-center text-gray-500 mt-8">No equipment matches your filters. Try clearing some filters or extend the distance.</p>
            )}
          </section>
        )}

        {tab === "add" && (
          <section className="bg-white rounded-2xl shadow p-4 md:p-6 border border-gray-100 max-w-3xl">
            <h2 className="text-xl font-semibold mb-4">Add Your Equipment</h2>
            <form className="grid md:grid-cols-2 gap-4" onSubmit={addItem}>
              <div className="md:col-span-2">
                <label className="text-sm text-gray-600">Title</label>
                <input
                  value={form.title}
                  onChange={(e) => setForm({ ...form, title: e.target.value })}
                  placeholder="e.g., Swaraj 744 FE Tractor"
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  required
                />
              </div>
              <div>
                <label className="text-sm text-gray-600">Category</label>
                <select
                  value={form.category}
                  onChange={(e) => setForm({ ...form, category: e.target.value })}
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                >
                  {CATEGORIES.map((c) => (
                    <option key={c} value={c}>{c}</option>
                  ))}
                </select>
              </div>
              <div>
                <label className="text-sm text-gray-600">Rate (₹/day)</label>
                <input
                  type="number"
                  value={form.ratePerDay}
                  onChange={(e) => setForm({ ...form, ratePerDay: e.target.value })}
                  placeholder="e.g., 3500"
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  required
                />
              </div>
              <div className="md:col-span-2">
                <label className="text-sm text-gray-600">Description</label>
                <textarea
                  value={form.description}
                  onChange={(e) => setForm({ ...form, description: e.target.value })}
                  placeholder="Short details, attachments, operator availability, etc."
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  rows={3}
                />
              </div>
              <div className="md:col-span-2">
                <label className="text-sm text-gray-600">Image URL</label>
                <input
                  value={form.image}
                  onChange={(e) => setForm({ ...form, image: e.target.value })}
                  placeholder="https://..."
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                />
              </div>
              <div>
                <label className="text-sm text-gray-600">Location Name (landmark)</label>
                <input
                  value={form.locationName}
                  onChange={(e) => setForm({ ...form, locationName: e.target.value })}
                  placeholder="e.g., Main Road Checkpost"
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  required
                />
              </div>
              <div className="grid grid-cols-2 gap-3">
                <div>
                  <label className="text-sm text-gray-600">Latitude</label>
                  <input
                    value={form.lat}
                    onChange={(e) => setForm({ ...form, lat: e.target.value })}
                    placeholder="12.9716"
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                    required
                  />
                </div>
                <div>
                  <label className="text-sm text-gray-600">Longitude</label>
                  <input
                    value={form.lon}
                    onChange={(e) => setForm({ ...form, lon: e.target.value })}
                    placeholder="77.5946"
                    className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                    required
                  />
                </div>
                <button
                  type="button"
                  onClick={fillOwnerCoords}
                  className="col-span-2 h-10 mt-1 rounded-xl bg-gray-900 text-white hover:bg-black"
                >
                  Use My Current Location
                </button>
              </div>
              <div>
                <label className="text-sm text-gray-600">Owner Name</label>
                <input
                  value={form.ownerName}
                  onChange={(e) => setForm({ ...form, ownerName: e.target.value })}
                  placeholder="Your full name"
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  required
                />
              </div>
              <div>
                <label className="text-sm text-gray-600">Owner Phone</label>
                <input
                  value={form.ownerPhone}
                  onChange={(e) => setForm({ ...form, ownerPhone: e.target.value })}
                  placeholder="e.g., +91-9xxxx-xxxxx"
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                  required
                />
              </div>

              <div className="md:col-span-2 flex gap-3 mt-2">
                <button
                  type="submit"
                  className="px-4 py-2 rounded-xl bg-blue-600 text-white hover:bg-blue-700"
                >
                  Add Listing
                </button>
                <button
                  type="button"
                  onClick={() => setTab("browse")}
                  className="px-4 py-2 rounded-xl bg-gray-100 text-gray-800 hover:bg-gray-200"
                >
                  Cancel
                </button>
              </div>
            </form>
          </section>
        )}
      </main>

      {/* Booking modal */}
      {bookingItem && (
        <div className="fixed inset-0 bg-black/40 backdrop-blur-sm flex items-center justify-center p-4">
          <div className="bg-white rounded-2xl shadow-xl w-full max-w-lg p-5">
            <div className="flex items-start justify-between">
              <h3 className="text-lg font-semibold">Book: {bookingItem.title}</h3>
              <button
                className="text-gray-500 hover:text-gray-800"
                onClick={() => setBookingItem(null)}
              >
                ✕
              </button>
            </div>
            <p className="text-sm text-gray-600 mt-1">Rate: ₹{bookingItem.ratePerDay.toLocaleString()}/day</p>
            <div className="grid grid-cols-2 gap-3 mt-4">
              <div>
                <label className="text-sm text-gray-600">From</label>
                <input
                  type="date"
                  value={fromDate}
                  onChange={(e) => setFromDate(e.target.value)}
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                />
              </div>
              <div>
                <label className="text-sm text-gray-600">To</label>
                <input
                  type="date"
                  value={toDate}
                  onChange={(e) => setToDate(e.target.value)}
                  className="w-full mt-1 rounded-xl border-gray-300 focus:border-blue-500 focus:ring-blue-500"
                />
              </div>
            </div>
            <div className="mt-5 flex items-center justify-end gap-3">
              <button
                className="px-4 py-2 rounded-xl bg-gray-100 text-gray-800 hover:bg-gray-200"
                onClick={() => setBookingItem(null)}
              >
                Cancel
              </button>
              <button
                className="px-4 py-2 rounded-xl bg-blue-600 text-white hover:bg-blue-700"
                onClick={confirmBooking}
              >
                Confirm Booking
              </button>
            </div>
            <p className="text-xs text-gray-500 mt-3">Note: Demo only. In production, store bookings and send SMS/WhatsApp to the owner.</p>
          </div>
        </div>
      )}

      <footer className="text-center text-xs text-gray-500 py-8">© {new Date().getFullYear()} AgriRent Demo</footer>
    </div>
  );
}
