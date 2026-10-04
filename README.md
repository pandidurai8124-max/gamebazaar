# gamebazaar
Gaming marketplace websiteb.```react
import React, { useState, useEffect, useMemo } from 'react';
import { 
  Shield, Search, Plus, Home as HomeIcon, User, 
  Settings, Star, TrendingUp, AlertTriangle, CheckCircle2, 
  ChevronRight, Gamepad2, ShoppingCart, Heart, Activity,
  Menu, X, LogOut, Info, Trash2
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged, signOut } from 'firebase/auth';
import { getFirestore, collection, addDoc, onSnapshot, deleteDoc, doc } from 'firebase/firestore';

// Firebase Setup
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {
  apiKey: "demo-key",
  projectId: "demo-project"
};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'gamebazaar-dev';

const CATEGORIES = ['All', 'Strategy', 'Battle Royale', 'RPG', 'Sports', 'Other'];

// Custom Toast Component to replace alert()
const Toast = ({ message, type, onClose }) => {
  useEffect(() => {
    const timer = setTimeout(onClose, 3000);
    return () => clearTimeout(timer);
  }, [onClose]);

  return (
    <div className={`fixed bottom-4 right-4 z-50 flex items-center gap-2 px-4 py-3 rounded-lg shadow-lg border ${
      type === 'error' ? 'bg-red-900/90 border-red-500 text-red-100' : 'bg-green-900/90 border-green-500 text-green-100'
    } animate-bounce`}>
      {type === 'error' ? <AlertTriangle size={18} /> : <CheckCircle2 size={18} />}
      <span className="font-medium">{message}</span>
      <button onClick={onClose} className="ml-2 hover:opacity-75"><X size={16} /></button>
    </div>
  );
};

export default function GameBazaarApp() {
  const [user, setUser] = useState(null);
  const [authLoading, setAuthLoading] = useState(true);
  const [activeView, setActiveView] = useState('home'); // home, seller, buyer, admin
  const [listings, setListings] = useState([]);
  const [searchQuery, setSearchQuery] = useState('');
  const [selectedCategory, setSelectedCategory] = useState('All');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [toast, setToast] = useState(null);
  const [mobileMenuOpen, setMobileMenuOpen] = useState(false);

  const [newListing, setNewListing] = useState({
    title: '', category: 'Strategy', level: '', price: '', description: ''
  });

  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (error) {
        console.error("Auth error:", error);
        setToast({ message: 'Authentication failed', type: 'error' });
      }
    };
    initAuth();
    
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      setAuthLoading(false);
    });
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;
    const listingsRef = collection(db, 'artifacts', appId, 'public', 'data', 'listings');
    
    const unsubscribe = onSnapshot(listingsRef, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
      data.sort((a, b) => b.createdAt - a.createdAt);
      setListings(data);
    }, (error) => {
      console.error("Fetch error:", error);
      setToast({ message: 'Failed to load listings', type: 'error' });
    });

    return () => unsubscribe();
  }, [user]);

  const handleCreateListing = async (e) => {
    e.preventDefault();
    if (!user || isSubmitting) return;
    setIsSubmitting(true);

    try {
      const listingsRef = collection(db, 'artifacts', appId, 'public', 'data', 'listings');
      await addDoc(listingsRef, {
        ...newListing,
        price: Number(newListing.price),
        sellerId: user.uid,
        sellerRating: (Math.random() * (5.0 - 4.2) + 4.2).toFixed(1),
        createdAt: Date.now(),
        verified: Math.random() > 0.3
      });
      
      setNewListing({ title: '', category: 'Strategy', level: '', price: '', description: '' });
      setToast({ message: 'Listing published successfully!', type: 'success' });
      setActiveView('home');
    } catch (error) {
      setToast({ message: 'Error creating listing', type: 'error' });
    } finally {
      setIsSubmitting(false);
    }
  };

  const handleDeleteListing = async (id) => {
    if (!user) return;
    try {
      await deleteDoc(doc(db, 'artifacts', appId, 'public', 'data', 'listings', id));
      setToast({ message: 'Listing removed', type: 'success' });
    } catch (error) {
      setToast({ message: 'Failed to remove listing', type: 'error' });
    }
  };

  const filteredListings = useMemo(() => {
    return listings.filter(listing => {
      const matchesSearch = listing.title.toLowerCase().includes(searchQuery.toLowerCase()) || 
                            listing.description.toLowerCase().includes(searchQuery.toLowerCase());
      const matchesCategory = selectedCategory === 'All' || listing.category === selectedCategory;
      return matchesSearch && matchesCategory;
    });
  }, [listings, searchQuery, selectedCategory]);

  const userListings = useMemo(() => {
    if (!user) return [];
    return listings.filter(l => l.sellerId === user.uid);
  }, [listings, user]);

  const ListingCard = ({ listing, isOwner = false }) => (
    <div className="bg-[#1a1a24] rounded-2xl overflow-hidden border border-gray-800 hover:border-purple-500/50 transition-all duration-300 group shadow-lg hover:shadow-purple-500/10 flex flex-col">
      <div className="h-36 bg-gradient-to-br from-gray-800 to-gray-900 relative p-4 flex items-start justify-between">
        <span className="bg-purple-600/90 text-white text-xs px-2.5 py-1 rounded-full font-semibold backdrop-blur-sm shadow-sm">
          {listing.category}
        </span>
        <div className="flex items-center gap-1 bg-black/60 text-yellow-400 text-xs px-2 py-1 rounded-full backdrop-blur-sm">
          <Star size={12} fill="currentColor" /> {listing.sellerRating}
        </div>
      </div>
      
      <div className="p-5 flex-1 flex flex-col">
        <h3 className="text-lg font-bold text-gray-100 group-hover:text-purple-400 transition-colors line-clamp-1 mb-2">{listing.title}</h3>
        <p className="text-gray-400 text-sm mb-4 line-clamp-2 flex-1">{listing.description}</p>
        
        <div className="flex justify-between items-center mb-4 text-xs font-medium text-gray-300">
          <span className="bg-gray-800/80 px-2 py-1 rounded-md border border-gray-700">Lvl {listing.level || 'N/A'}</span>
          {listing.verified && (
            <span className="flex items-center text-blue-400 gap-1">
              <CheckCircle2 size={14} /> Verified
            </span>
          )}
        </div>
        
        <div className="flex items-center justify-between pt-4 border-t border-gray-800/80 mt-auto">
          <span className="text-xl font-black text-white">
            ₹{listing.price.toLocaleString('en-IN')}
          </span>
          {isOwner ? (
            <button onClick={() => handleDeleteListing(listing.id)} className="text-red-400 hover:text-red-300 hover:bg-red-400/10 p-2 rounded-lg transition-colors" title="Delete">
              <Trash2 size={18} />
            </button>
          ) : (
            <button className="bg-purple-600 hover:bg-purple-500 text-white px-4 py-2 rounded-xl text-sm font-bold transition-all shadow-md shadow-purple-900/20">
              Details
            </button>
          )}
        </div>
      </div>
    </div>
  );

  if (authLoading) {
    return (
      <div className="min-h-screen bg-[#0a0a0f] flex flex-col items-center justify-center gap-4">
        <Gamepad2 className="text-purple-500 animate-pulse" size={48} />
        <div className="text-gray-400 font-medium">Loading GameBazaar...</div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-[#0a0a0f] text-gray-200 font-sans selection:bg-purple-500/30 overflow-x-hidden">
      {toast && <Toast message={toast.message} type={toast.type} onClose={() => setToast(null)} />}
      
      {/* Navbar */}
      <nav className="sticky top-0 z-50 bg-[#0a0a0f]/90 backdrop-blur-md border-b border-gray-800">
        <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
          <div className="flex items-center justify-between h-20">
            <div className="flex items-center cursor-pointer group" onClick={() => setActiveView('home')}>
              <div className="bg-purple-600/20 p-2 rounded-xl mr-3 group-hover:bg-purple-600/30 transition-colors">
                <Gamepad2 className="text-purple-500" size={28} />
              </div>
              <span className="text-2xl font-black tracking-tight text-white">Game<span className="text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-blue-500">Bazaar</span></span>
            </div>
            
            {/* Desktop Nav */}
            <div className="hidden md:flex items-center gap-6">
              <button onClick={() => setActiveView('home')} className={`font-semibold text-sm transition-colors ${activeView === 'home' ? 'text-white' : 'text-gray-400 hover:text-gray-200'}`}>Home</button>
              <button onClick={() => setActiveView('buyer')} className={`font-semibold text-sm transition-colors ${activeView === 'buyer' ? 'text-white' : 'text-gray-400 hover:text-gray-200'}`}>Buyer Dashboard</button>
              <button onClick={() => setActiveView('seller')} className={`font-semibold text-sm transition-colors ${activeView === 'seller' ? 'text-white' : 'text-gray-400 hover:text-gray-200'}`}>Seller Hub</button>
              <button onClick={() => setActiveView('admin')} className={`font-semibold text-sm transition-colors ${activeView === 'admin' ? 'text-white' : 'text-gray-400 hover:text-gray-200'}`}>Admin</button>
              
              <div className="h-6 w-px bg-gray-800 mx-2"></div>
              <div className="flex items-center gap-3 bg-[#1a1a24] pl-2 pr-4 py-1.5 rounded-full border border-gray-800">
                <div className="w-8 h-8 rounded-full bg-gradient-to-tr from-purple-600 to-blue-600 flex items-center justify-center text-xs font-bold text-white shadow-lg">
                  {user?.uid.substring(0, 2).toUpperCase() || 'U'}
                </div>
                <span className="text-xs font-medium text-gray-400 max-w-[80px] truncate">{user?.uid}</span>
              </div>
            </div>

            {/* Mobile Menu Toggle */}
            <button className="md:hidden p-2 text-gray-400 hover:text-white" onClick={() => setMobileMenuOpen(!mobileMenuOpen)}>
              {mobileMenuOpen ? <X size={24} /> : <Menu size={24} />}
            </button>
          </div>
        </div>

        {/* Mobile Nav Dropdown */}
        {mobileMenuOpen && (
          <div className="md:hidden absolute top-20 left-0 w-full bg-[#0a0a0f] border-b border-gray-800 py-4 px-4 flex flex-col gap-4 shadow-2xl">
            {['home', 'buyer', 'seller', 'admin'].map(view => (
              <button 
                key={view}
                onClick={() => { setActiveView(view); setMobileMenuOpen(false); }}
                className={`text-left px-4 py-3 rounded-lg font-medium capitalize ${activeView === view ? 'bg-purple-600/10 text-purple-400' : 'text-gray-400'}`}
              >
                {view} Dashboard
              </button>
            ))}
          </div>
        )}
      </nav>

      {/* Main Content Area */}
      <main className="pb-20">
        
        {}
        {activeView === 'home' && (
          <div className="animate-in fade-in duration-500">
            {/* Hero Section */}
            <div className="relative overflow-hidden border-b border-gray-800/50 bg-[#0a0a0f]">
              <div className="absolute inset-0 bg-[radial-gradient(ellipse_at_top,_var(--tw-gradient-stops))] from-purple-900/20 via-[#0a0a0f] to-[#0a0a0f]"></div>
              
              <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-20 lg:py-32 relative z-10 text-center flex flex-col items-center">
                <div className="inline-flex items-center gap-2 px-3 py-1.5 rounded-full bg-purple-500/10 border border-purple-500/20 text-purple-300 text-xs font-semibold mb-8">
                  <Star size={14} /> The #1 Marketplace for Gamers
                </div>
                <h1 className="text-4xl md:text-6xl lg:text-7xl font-black tracking-tight mb-6 max-w-4xl text-white">
                  Buy & Sell Gaming Accounts & <span className="text-transparent bg-clip-text bg-gradient-to-r from-purple-400 to-blue-400">Digital Items</span>
                </h1>
                <p className="text-lg md:text-xl text-gray-400 max-w-2xl mx-auto mb-10 leading-relaxed">
                  Join thousands of verified gamers trading safely. Secure transactions, anti-scam protection, and instant delivery.
                </p>
                
                {/* Search Bar in Hero */}
                <div className="w-full max-w-2xl relative mb-12 group">
                  <Search className="absolute left-4 top-1/2 transform -translate-y-1/2 text-gray-400 group-focus-within:text-purple-400 transition-colors" size={22} />
                  <input
                    type="text"
                    placeholder="Search accounts, skins, currency..."
                    value={searchQuery}
                    onChange={(e) => setSearchQuery(e.target.value)}
                    className="w-full bg-[#1a1a24] border border-gray-700/80 rounded-2xl py-4 pl-12 pr-6 focus:outline-none focus:border-purple-500 focus:ring-1 focus:ring-purple-500 transition-all text-lg shadow-xl shadow-black/50"
                  />
                </div>
              </div>
            </div>

            <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-16">
              {/* Category Pills */}
              <div className="flex overflow-x-auto pb-6 mb-8 gap-3 hide-scrollbar snap-x">
                {CATEGORIES.map(cat => (
                  <button
                    key={cat}
                    onClick={() => setSelectedCategory(cat)}
                    className={`snap-start whitespace-nowrap px-6 py-2.5 rounded-xl text-sm font-bold transition-all ${
                      selectedCategory === cat 
                        ? 'bg-purple-600 text-white shadow-lg shadow-purple-600/25 border border-purple-500' 
                        : 'bg-[#1a1a24] text-gray-400 hover:text-white border border-gray-800 hover:border-gray-700'
                    }`}
                  >
                    {cat}
                  </button>
                ))}
              </div>

              {/* Listings Grid */}
              <div className="mb-20">
                <h2 className="text-2xl font-bold mb-8 text-white flex items-center gap-2">
                  <TrendingUp className="text-purple-500" /> Featured Listings
                </h2>
                
                {filteredListings.length > 0 ? (
                  <div className="grid grid-cols-1 sm:grid-cols-2 lg:grid-cols-3 xl:grid-cols-4 gap-6">
                    {filteredListings.map(listing => (
                      <ListingCard key={listing.id} listing={listing} />
                    ))}
                  </div>
                ) : (
                  <div className="text-center py-24 bg-[#1a1a24] rounded-3xl border border-gray-800 border-dashed">
                    <Gamepad2 className="mx-auto text-gray-600 mb-4 opacity-50" size={56} />
                    <h3 className="text-xl font-bold text-gray-300 mb-2">No listings found</h3>
                    <p className="text-gray-500 max-w-md mx-auto">We couldn't find any items matching your criteria. Try adjusting your filters or search term.</p>
                  </div>
                )}
              </div>

              {/* Info Sections */}
              <div className="grid grid-cols-1 lg:grid-cols-2 gap-6 mt-12">
                {/* How It Works */}
                <div className="bg-gradient-to-br from-[#1a1a24] to-[#0f0f14] p-8 rounded-3xl border border-gray-800/80">
                  <h3 className="text-2xl font-bold mb-8 flex items-center gap-3 text-white">
                    <div className="p-2.5 bg-blue-500/10 rounded-xl text-blue-400"><Settings size={24} /></div>
                    How It Works
                  </h3>
                  <div className="space-y-8">
                    {[
                      { num: '1', title: 'Create Account', desc: 'Sign up securely using our Firebase-powered authentication.' },
                      { num: '2', title: 'Browse or List', desc: 'Find your dream account or post your assets for sale in seconds.' },
                      { num: '3', title: 'Secure Transfer', desc: 'Communicate safely and complete transactions with peace of mind.' }
                    ].map((step, i) => (
                      <div key={i} className="flex gap-5">
                        <div className="flex-shrink-0 w-12 h-12 rounded-2xl bg-[#0a0a0f] flex items-center justify-center font-black text-xl text-gray-300 border border-gray-800 shadow-inner">{step.num}</div>
                        <div>
                          <h4 className="font-bold text-lg text-gray-100">{step.title}</h4>
                          <p className="text-gray-400 text-sm mt-1 leading-relaxed">{step.desc}</p>
                        </div>
                      </div>
                    ))}
                  </div>
                </div>

                {/* Trust & Safety */}
                <div className="bg-gradient-to-br from-[#1a1a24] to-[#0f0f14] p-8 rounded-3xl border border-gray-800/80">
                  <h3 className="text-2xl font-bold mb-8 flex items-center gap-3 text-white">
                    <div className="p-2.5 bg-green-500/10 rounded-xl text-green-400"><Shield size={24} /></div>
                    Trust & Safety
                  </h3>
                  <div className="space-y-4">
                    <div className="bg-[#0a0a0f]/50 p-5 rounded-2xl border border-gray-800/50">
                      <h4 className="font-bold text-white flex items-center gap-2 mb-1"><CheckCircle2 size={18} className="text-green-500" /> Verified Sellers</h4>
                      <p className="text-gray-400 text-sm">We monitor seller history to maintain a safe marketplace environment.</p>
                    </div>
                    <div className="bg-[#0a0a0f]/50 p-5 rounded-2xl border border-gray-800/50">
                      <h4 className="font-bold text-white flex items-center gap-2 mb-1"><Shield size={18} className="text-blue-500" /> Buyer Protection</h4>
                      <p className="text-gray-400 text-sm">Guidelines and support to ensure you get exactly what you pay for.</p>
                    </div>
                    <div className="bg-red-950/20 p-5 rounded-2xl border border-red-900/30">
                      <h4 className="font-bold text-red-400 flex items-center gap-2 mb-1"><AlertTriangle size={18} /> Anti-Scam Policy</h4>
                      <p className="text-red-300/70 text-sm">Never share sensitive passwords off-platform. Report suspicious behavior instantly.</p>
                    </div>
                  </div>
                </div>
              </div>
            </div>
          </div>
        )}

        {}
        {activeView === 'seller' && (
          <div className="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8 py-10 animate-in fade-in duration-300">
            <div className="mb-8">
              <h1 className="text-3xl font-black text-white mb-2">Seller Hub</h1>
              <p className="text-gray-400">Manage your active listings and track your earnings.</p>
            </div>

            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
              {/* Add Listing Form */}
            
