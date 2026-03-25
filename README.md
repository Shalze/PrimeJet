import React, { useState, useEffect, useRef } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged, 
  createUserWithEmailAndPassword, 
  signInWithEmailAndPassword, 
  signOut,
  signInWithCustomToken
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  setDoc, 
  getDoc, 
  onSnapshot, 
  addDoc,
  deleteDoc
} from 'firebase/firestore';
import { 
  Anchor, Check, ChevronRight, Droplets, Flame, History, 
  Home, MapPin, MessageCircle, Moon, Settings, Shield, 
  Sun, User, Wrench, LogOut, Clock, Calendar, AlertTriangle, 
  Send, Wind, Navigation, Activity
} from 'lucide-react';

// --- Firebase Initialization ---
const firebaseConfig = typeof __firebase_config !== 'undefined' ? JSON.parse(__firebase_config) : {};
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'jetprime-app';

// --- AI Setup ---
const AI_API_KEY = ""; // Injected by environment

export default function App() {
  const [user, setUser] = useState(null);
  const [authLoading, setAuthLoading] = useState(true);
  
  // App State
  const [profile, setProfile] = useState(null);
  const [jetSki, setJetSki] = useState(null);
  const [activeTab, setActiveTab] = useState('dashboard'); // dashboard, rotinas, historico_nav, historico_man, chat
  
  // Data State
  const [navHistory, setNavHistory] = useState([]);
  const [maintHistory, setMaintHistory] = useState([]);
  
  // Weather State
  const [weather, setWeather] = useState(null);

  // --- 1. Authentication Flow ---
  useEffect(() => {
    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        }
      } catch (error) {
        console.error("Auth init error:", error);
      }
    };
    initAuth();

    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      setUser(currentUser);
      setAuthLoading(false);
    });

    return () => unsubscribe();
  }, []);

  // --- 2. Data Fetching ---
  useEffect(() => {
    if (!user) {
      setProfile(null);
      setJetSki(null);
      setNavHistory([]);
      setMaintHistory([]);
      return;
    }

    const userId = user.uid;
    
    // Fetch Profile & JetSki
    const profileRef = doc(db, 'artifacts', appId, 'users', userId, 'profile', 'info');
    const unsubProfile = onSnapshot(profileRef, (docSnap) => {
      if (docSnap.exists()) {
        const data = docSnap.data();
        setProfile(data.settings || { theme: 'dark', color: 'blue', fontSize: 'text-base' });
        setJetSki(data.jetSki || null);
      } else {
        setProfile({ theme: 'dark', color: 'blue', fontSize: 'text-base' });
      }
    }, (err) => console.error("Profile fetch error:", err));

    // Fetch Navigation History
    const navRef = collection(db, 'artifacts', appId, 'users', userId, 'navigation');
    const unsubNav = onSnapshot(navRef, (snapshot) => {
      const navs = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      // Sort in memory (Rule 2)
      navs.sort((a, b) => b.timestamp - a.timestamp);
      setNavHistory(navs);
    }, (err) => console.error("Nav fetch error:", err));

    // Fetch Maintenance History
    const maintRef = collection(db, 'artifacts', appId, 'users', userId, 'maintenance');
    const unsubMaint = onSnapshot(maintRef, (snapshot) => {
      const maints = snapshot.docs.map(d => ({ id: d.id, ...d.data() }));
      maints.sort((a, b) => b.timestamp - a.timestamp);
      setMaintHistory(maints);
    }, (err) => console.error("Maint fetch error:", err));

    // Função auxiliar para buscar clima
    const fetchWeather = (lat, lon) => {
      fetch(`https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current_weather=true`)
        .then(res => res.json())
        .then(data => setWeather(data.current_weather))
        .catch(err => console.error("Weather fetch error:", err));
    };

    // Fetch Weather based on REAL User Location
    if ("geolocation" in navigator) {
      navigator.geolocation.getCurrentPosition(
        (position) => {
          fetchWeather(position.coords.latitude, position.coords.longitude);
        },
        (error) => {
          console.warn("Geolocalização negada ou falhou. Usando localização padrão.", error);
          // Fallback location (ex: Vitória/Vila Velha) se o usuário negar permissão
          fetchWeather(-20.3297, -40.2925);
        }
      );
    } else {
      // Fallback para navegadores que não suportam geolocalização
      fetchWeather(-20.3297, -40.2925);
    }

    return () => {
      unsubProfile();
      unsubNav();
      unsubMaint();
    };
  }, [user]);

  // --- Theme Management ---
  const applyTheme = () => {
    if (!profile) return 'bg-gray-900 text-white font-sans text-base';
    const bg = profile.theme === 'dark' ? 'bg-slate-900 text-white' : 'bg-gray-50 text-gray-900';
    return `${bg} font-sans ${profile.fontSize}`;
  };

  const getPrimaryColor = () => {
    if (!profile) return 'bg-blue-600 hover:bg-blue-700';
    switch(profile.color) {
      case 'red': return 'bg-red-600 hover:bg-red-700 text-white';
      case 'green': return 'bg-emerald-600 hover:bg-emerald-700 text-white';
      case 'purple': return 'bg-purple-600 hover:bg-purple-700 text-white';
      case 'blue': default: return 'bg-blue-600 hover:bg-blue-700 text-white';
    }
  };

  const getPrimaryTextColor = () => {
    if (!profile) return 'text-blue-500';
    switch(profile.color) {
      case 'red': return 'text-red-500';
      case 'green': return 'text-emerald-500';
      case 'purple': return 'text-purple-500';
      case 'blue': default: return 'text-blue-500';
    }
  };

  // --- Loading State ---
  if (authLoading) {
    return <div className="min-h-screen bg-slate-900 flex items-center justify-center text-white">Carregando JetPrime...</div>;
  }

  // --- Render Flow ---
  if (!user) {
    return <AuthScreen />;
  }

  if (!jetSki) {
    return <RegistrationScreen user={user} setJetSki={setJetSki} profile={profile} getPrimaryColor={getPrimaryColor} />;
  }

  return (
    <div className={`min-h-screen flex flex-col md:flex-row transition-colors duration-300 ${applyTheme()}`}>
      
      {/* Sidebar / Bottom Nav */}
      <NavigationMenu activeTab={activeTab} setActiveTab={setActiveTab} profile={profile} getPrimaryTextColor={getPrimaryTextColor} />

      {/* Main Content Area */}
      <main className="flex-1 p-4 md:p-8 overflow-y-auto pb-24 md:pb-8">
        
        {/* Header */}
        <header className="flex justify-between items-center mb-6">
          <div className="flex items-center gap-2">
            <Anchor className={`w-8 h-8 ${getPrimaryTextColor()}`} />
            <h1 className="text-2xl font-bold tracking-tight">JetPrime</h1>
          </div>
          <div className="hidden md:flex flex-col items-center">
            <span className="text-sm font-medium opacity-75">{new Date().toLocaleDateString('pt-BR', { weekday: 'long' }).toUpperCase()}</span>
            <span className="font-bold">{new Date().toLocaleDateString('pt-BR')}</span>
          </div>
          <button onClick={() => setActiveTab('settings')} className="p-2 rounded-full hover:bg-gray-200 dark:hover:bg-slate-800 transition-colors">
            <Settings className="w-6 h-6" />
          </button>
        </header>

        {/* Dynamic Content */}
        <div className="max-w-4xl mx-auto space-y-6">
          {activeTab === 'dashboard' && (
            <Dashboard 
              user={user} 
              jetSki={jetSki} 
              weather={weather} 
              getPrimaryColor={getPrimaryColor} 
              profile={profile}
              navHistory={navHistory}
            />
          )}
          {activeTab === 'rotinas' && <MaintenanceRoutines getPrimaryColor={getPrimaryColor} profile={profile} />}
          {activeTab === 'historico_nav' && <NavHistory navHistory={navHistory} profile={profile} getPrimaryTextColor={getPrimaryTextColor} />}
          {activeTab === 'historico_man' && <MaintHistory maintHistory={maintHistory} profile={profile} getPrimaryTextColor={getPrimaryTextColor} />}
          {activeTab === 'chat' && <CaptainsChat user={user} profile={profile} getPrimaryColor={getPrimaryColor} />}
          {activeTab === 'settings' && <SettingsModal user={user} jetSki={jetSki} profile={profile} getPrimaryColor={getPrimaryColor} onClose={() => setActiveTab('dashboard')} />}
        </div>
      </main>
    </div>
  );
}

// ==========================================
// SCREENS & COMPONENTS
// ==========================================

// --- Auth Screen ---
function AuthScreen() {
  const [isLogin, setIsLogin] = useState(true);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');

  const handleAuth = async (e) => {
    e.preventDefault();
    setError('');
    try {
      if (isLogin) {
        await signInWithEmailAndPassword(auth, email, password);
      } else {
        await createUserWithEmailAndPassword(auth, email, password);
      }
    } catch (err) {
      setError("Erro de autenticação. Verifique suas credenciais.");
    }
  };

  const handleGuest = async () => {
    try {
      await signInAnonymously(auth);
    } catch (err) {
      setError("Erro ao entrar como visitante.");
    }
  };

  return (
    <div className="min-h-screen bg-slate-900 flex items-center justify-center p-4">
      <div className="bg-slate-800 p-8 rounded-2xl shadow-xl w-full max-w-md text-white border border-slate-700">
        <div className="flex justify-center mb-6">
          <Anchor className="w-16 h-16 text-blue-500" />
        </div>
        <h2 className="text-3xl font-bold text-center mb-2">JetPrime</h2>
        <p className="text-center text-slate-400 mb-8">Gerenciamento inteligente do seu Jet Ski</p>

        {error && <div className="bg-red-500/20 border border-red-500 text-red-300 p-3 rounded-lg mb-4 text-sm">{error}</div>}

        <form onSubmit={handleAuth} className="space-y-4">
          <div>
            <label className="block text-sm font-medium mb-1">E-mail</label>
            <input 
              type="email" 
              required
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="w-full bg-slate-900 border border-slate-700 rounded-lg p-3 text-white focus:ring-2 focus:ring-blue-500 outline-none" 
              placeholder="seu@email.com" 
            />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">Senha</label>
            <input 
              type="password" 
              required
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full bg-slate-900 border border-slate-700 rounded-lg p-3 text-white focus:ring-2 focus:ring-blue-500 outline-none" 
              placeholder="••••••••" 
            />
          </div>
          <button type="submit" className="w-full bg-blue-600 hover:bg-blue-700 text-white font-bold py-3 rounded-lg transition-colors">
            {isLogin ? 'Entrar' : 'Criar Conta'}
          </button>
        </form>

        <div className="mt-6 text-center">
          <button onClick={() => setIsLogin(!isLogin)} className="text-blue-400 hover:underline text-sm">
            {isLogin ? 'Não tem uma conta? Cadastre-se' : 'Já tem conta? Faça Login'}
          </button>
        </div>

        <div className="mt-6 pt-6 border-t border-slate-700 text-center">
          <button onClick={handleGuest} className="text-slate-400 hover:text-white transition-colors text-sm flex items-center justify-center w-full gap-2">
            <User className="w-4 h-4" />
            Entrar como Visitante
          </button>
        </div>
      </div>
    </div>
  );
}

// --- Registration Screen ---
function RegistrationScreen({ user, setJetSki, profile, getPrimaryColor }) {
  const [marca, setMarca] = useState('');
  const [modelo, setModelo] = useState('');
  const [ano, setAno] = useState('');

  const handleRegister = async (e) => {
    e.preventDefault();
    const jetData = { marca, modelo, ano };
    const profileRef = doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'info');
    
    // Save to Firestore
    await setDoc(profileRef, {
      jetSki: jetData,
      settings: profile || { theme: 'dark', color: 'blue', fontSize: 'text-base' }
    }, { merge: true });
    
    setJetSki(jetData);
  };

  return (
    <div className="min-h-screen bg-slate-900 flex items-center justify-center p-4 text-white">
      <div className="bg-slate-800 p-8 rounded-2xl shadow-xl w-full max-w-md border border-slate-700">
        <h2 className="text-2xl font-bold mb-6 text-center">Cadastre seu Jet Ski</h2>
        <form onSubmit={handleRegister} className="space-y-4">
          <div>
            <label className="block text-sm font-medium mb-1">Marca</label>
            <input type="text" required value={marca} onChange={(e) => setMarca(e.target.value)} placeholder="Ex: Sea-Doo, Yamaha" className="w-full bg-slate-900 border border-slate-700 rounded-lg p-3 outline-none focus:border-blue-500" />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">Modelo</label>
            <input type="text" required value={modelo} onChange={(e) => setModelo(e.target.value)} placeholder="Ex: RXT-X 300" className="w-full bg-slate-900 border border-slate-700 rounded-lg p-3 outline-none focus:border-blue-500" />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1">Ano</label>
            <input type="number" required value={ano} onChange={(e) => setAno(e.target.value)} placeholder="Ex: 2023" className="w-full bg-slate-900 border border-slate-700 rounded-lg p-3 outline-none focus:border-blue-500" />
          </div>
          <button type="submit" className={`w-full py-3 rounded-lg font-bold transition-colors mt-6 ${getPrimaryColor()}`}>
            Salvar e Começar
          </button>
        </form>
      </div>
    </div>
  );
}

// --- Navigation Menu ---
function NavigationMenu({ activeTab, setActiveTab, profile, getPrimaryTextColor }) {
  const menuItems = [
    { id: 'dashboard', icon: Home, label: 'Início' },
    { id: 'rotinas', icon: Wrench, label: 'Manutenção' },
    { id: 'historico_nav', icon: MapPin, label: 'Navegação' },
    { id: 'historico_man', icon: History, label: 'Histórico' },
    { id: 'chat', icon: MessageCircle, label: 'Dicas do Capitão' },
  ];

  const isDark = profile?.theme === 'dark';

  return (
    <>
      {/* Desktop Sidebar */}
      <aside className={`hidden md:flex flex-col w-64 border-r ${isDark ? 'border-slate-800 bg-slate-900' : 'border-gray-200 bg-white'} p-4`}>
        <nav className="flex-1 space-y-2 mt-20">
          {menuItems.map(item => (
            <button
              key={item.id}
              onClick={() => setActiveTab(item.id)}
              className={`w-full flex items-center gap-3 px-4 py-3 rounded-xl font-medium transition-all
                ${activeTab === item.id 
                  ? (isDark ? 'bg-slate-800 text-white' : 'bg-gray-100 text-black') 
                  : 'opacity-60 hover:opacity-100'}`}
            >
              <item.icon className={`w-5 h-5 ${activeTab === item.id ? getPrimaryTextColor() : ''}`} />
              {item.label}
            </button>
          ))}
        </nav>
      </aside>

      {/* Mobile Bottom Bar */}
      <nav className={`md:hidden fixed bottom-0 left-0 right-0 z-50 border-t flex justify-around p-3 pb-safe
        ${isDark ? 'bg-slate-900 border-slate-800' : 'bg-white border-gray-200'}`}>
        {menuItems.map(item => (
          <button
            key={item.id}
            onClick={() => setActiveTab(item.id)}
            className={`flex flex-col items-center gap-1 p-2 rounded-lg transition-colors
              ${activeTab === item.id ? getPrimaryTextColor() : 'opacity-50'}`}
          >
            <item.icon className="w-6 h-6" />
            <span className="text-[10px] font-medium">{item.label}</span>
          </button>
        ))}
      </nav>
    </>
  );
}

// --- Dashboard ---
function Dashboard({ user, jetSki, weather, getPrimaryColor, profile, navHistory }) {
  const isDark = profile?.theme === 'dark';
  const cardBg = isDark ? 'bg-slate-800 border-slate-700' : 'bg-white border-gray-200 shadow-sm';
  
  // Calculate total hours for maintenance alert
  const totalMinutes = navHistory.reduce((acc, curr) => acc + Number(curr.duration || 0), 0);
  const totalHours = (totalMinutes / 60).toFixed(1);
  const needsMaintenance = totalHours > 50; // Arbitrary alert logic

  return (
    <div className="space-y-6 animate-fade-in">
      
      {/* Jet Ski Info & Weather */}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        <div className={`p-6 rounded-2xl border ${cardBg} flex items-center justify-between`}>
          <div>
            <h2 className="text-xl font-bold">{jetSki.marca} {jetSki.modelo}</h2>
            <p className="opacity-70">Ano: {jetSki.ano}</p>
          </div>
          <Anchor className="w-10 h-10 opacity-20" />
        </div>

        {weather && (
          <div className={`p-6 rounded-2xl border ${cardBg} flex items-center justify-between`}>
            <div>
              <p className="text-sm opacity-70 mb-1">Previsão do Tempo (Local)</p>
              <div className="flex items-center gap-2">
                <Wind className="w-6 h-6" />
                <span className="text-2xl font-bold">{weather.temperature}°C</span>
              </div>
            </div>
            <div className="text-right">
              <p className="text-sm opacity-70">Vento</p>
              <p className="font-semibold">{weather.windspeed} km/h</p>
            </div>
          </div>
        )}
      </div>

      {/* Navigation Form Block */}
      <NavigationForm user={user} getPrimaryColor={getPrimaryColor} isDark={isDark} cardBg={cardBg} />

      {/* Maintenance Alert */}
      <div className={`p-4 rounded-xl border flex items-center gap-4 ${needsMaintenance ? 'bg-red-500/10 border-red-500/50 text-red-500' : 'bg-emerald-500/10 border-emerald-500/50 text-emerald-500'}`}>
        <AlertTriangle className="w-8 h-8 flex-shrink-0" />
        <div>
          <h3 className="font-bold">{needsMaintenance ? 'Alerta de Manutenção' : 'Tudo em Ordem'}</h3>
          <p className="text-sm opacity-90">
            {needsMaintenance 
              ? 'Seu Jet Ski passou de 50h de uso. Recomendamos agendar uma revisão.' 
              : `Total de uso: ${totalHours}h. Nenhuma manutenção crítica pendente.`}
          </p>
        </div>
      </div>

      {/* Pre-Use Checklist */}
      <PreUseChecklist user={user} getPrimaryColor={getPrimaryColor} cardBg={cardBg} />

    </div>
  );
}

// --- Navigation Form Component ---
function NavigationForm({ user, getPrimaryColor, isDark, cardBg }) {
  const [local, setLocal] = useState('');
  const [duracao, setDuracao] = useState('');
  const [data, setData] = useState(new Date().toISOString().split('T')[0]);
  const [tipoAgua, setTipoAgua] = useState('Salgada');
  const [combAntes, setCombAntes] = useState('');
  const [combDepois, setCombDepois] = useState('');
  const [saved, setSaved] = useState(false);

  const handleSave = async (e) => {
    e.preventDefault();
    if (!user) return;

    // Calculate Consumption
    const diff = Number(combAntes) - Number(combDepois);
    const hours = Number(duracao) / 60;
    let consumo = "Indeterminado";
    
    if (diff > 0 && hours > 0) {
      const rate = diff / hours;
      if (rate < 15) consumo = "Baixo";
      else if (rate <= 25) consumo = "Médio";
      else consumo = "Alto";
    }

    const navData = {
      local,
      duration: Number(duracao),
      date: data,
      waterType: tipoAgua,
      fuelBefore: Number(combAntes),
      fuelAfter: Number(combDepois),
      consumption: consumo,
      timestamp: Date.now()
    };

    try {
      await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'navigation'), navData);
      setSaved(true);
      setTimeout(() => {
        setSaved(false);
        setLocal(''); setDuracao(''); setCombAntes(''); setCombDepois('');
      }, 3000);
    } catch (error) {
      console.error("Erro ao salvar navegação:", error);
    }
  };

  const inputClass = `w-full p-3 rounded-lg border outline-none transition-colors ${isDark ? 'bg-slate-900 border-slate-700 focus:border-blue-500' : 'bg-gray-50 border-gray-300 focus:border-blue-500'}`;

  return (
    <div className={`p-6 rounded-2xl border ${cardBg}`}>
      <div className="flex items-center gap-2 mb-6">
        <Navigation className="w-6 h-6" />
        <h2 className="text-xl font-bold">Registro de Navegação</h2>
      </div>

      <form onSubmit={handleSave} className="space-y-4">
        <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
          <div>
            <label className="block text-sm font-medium mb-1 opacity-80">Local (Praia/Rio)</label>
            <input type="text" required value={local} onChange={e => setLocal(e.target.value)} className={inputClass} placeholder="Ex: Praia da Costa" />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1 opacity-80">Data</label>
            <input type="date" required value={data} onChange={e => setData(e.target.value)} className={inputClass} />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1 opacity-80">Tempo (Minutos)</label>
            <input type="number" required value={duracao} onChange={e => setDuracao(e.target.value)} className={inputClass} placeholder="Ex: 120" />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1 opacity-80">Tipo de Água</label>
            <select value={tipoAgua} onChange={e => setTipoAgua(e.target.value)} className={inputClass}>
              <option value="Salgada">Água Salgada (Mar)</option>
              <option value="Doce">Água Doce (Rio/Lagoa)</option>
            </select>
          </div>
          <div>
            <label className="block text-sm font-medium mb-1 opacity-80">Combustível Antes (L/%)</label>
            <input type="number" required value={combAntes} onChange={e => setCombAntes(e.target.value)} className={inputClass} placeholder="Ex: 60" />
          </div>
          <div>
            <label className="block text-sm font-medium mb-1 opacity-80">Combustível Depois (L/%)</label>
            <input type="number" required value={combDepois} onChange={e => setCombDepois(e.target.value)} className={inputClass} placeholder="Ex: 20" />
          </div>
        </div>

        <button type="submit" className={`w-full py-4 rounded-xl font-bold transition-all flex justify-center items-center gap-2 mt-4 ${saved ? 'bg-emerald-600 text-white' : getPrimaryColor()}`}>
          {saved ? <><Check className="w-5 h-5"/> Salvo com Sucesso</> : 'Salvar Navegação'}
        </button>
      </form>
    </div>
  );
}

// --- Pre-Use Checklist ---
function PreUseChecklist({ user, getPrimaryColor, cardBg }) {
  const categories = {
    Motor: ['Cabos Conectados', 'Sem Cheiro Forte', 'Mangueiras Firmes', 'Sem Oxidação'],
    Bateria: ['Terminais Firmes', 'Partida Forte', 'Sem pó branco'],
    Casco: ['Sem Trincas', 'Sem água acumulada', 'Tampa de Drenagem fechada'],
    Sistema_Agua: ['Entrada de água limpa', 'Grade sem obstrução'],
    Seguranca: ['Colete', 'Corta Corrente Funcionando', 'Combustível', 'Documento', 'Jet bem preso']
  };

  const [checkedItems, setCheckedItems] = useState({});
  const [completed, setCompleted] = useState(false);

  const toggleCheck = (item) => {
    setCheckedItems(prev => ({ ...prev, [item]: !prev[item] }));
  };

  const totalItems = Object.values(categories).flat().length;
  const checkedCount = Object.values(checkedItems).filter(Boolean).length;
  const isAllChecked = checkedCount === totalItems;

  const handleConfirm = async () => {
    if (!isAllChecked || !user) return;
    
    try {
      await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'maintenance'), {
        type: 'Checklist Pré-Uso',
        timestamp: Date.now(),
        items: Object.keys(checkedItems).filter(k => checkedItems[k])
      });
      setCompleted(true);
      setTimeout(() => {
        setCompleted(false);
        setCheckedItems({});
      }, 5000);
    } catch (err) {
      console.error("Erro ao salvar checklist:", err);
    }
  };

  return (
    <div className={`p-6 rounded-2xl border ${cardBg}`}>
      <h2 className="text-xl font-bold mb-4 flex items-center gap-2">
        <Check className="w-6 h-6" />
        Checklist de Pré-Uso
      </h2>
      
      {completed ? (
        <div className="py-12 text-center animate-fade-in flex flex-col items-center">
          <div className="w-20 h-20 bg-emerald-500 rounded-full flex items-center justify-center mb-4">
            <Anchor className="w-10 h-10 text-white" />
          </div>
          <h3 className="text-3xl font-bold text-emerald-500 mb-2">Pronto para Navegar!</h3>
          <p className="opacity-70">Checklist salvo no seu histórico de manutenção.</p>
        </div>
      ) : (
        <div className="space-y-6">
          {Object.entries(categories).map(([cat, items]) => (
            <div key={cat}>
              <h3 className="font-semibold text-lg mb-2 opacity-90 border-b border-opacity-20 pb-1">{cat.replace('_', ' ')}</h3>
              <div className="grid grid-cols-1 sm:grid-cols-2 gap-2">
                {items.map(item => (
                  <label key={item} className="flex items-center gap-3 p-2 hover:bg-black/5 dark:hover:bg-white/5 rounded-lg cursor-pointer transition-colors">
                    <div className={`w-6 h-6 rounded border flex items-center justify-center transition-colors ${checkedItems[item] ? 'bg-blue-500 border-blue-500' : 'border-gray-400'}`}>
                      {checkedItems[item] && <Check className="w-4 h-4 text-white" />}
                    </div>
                    <span className="opacity-90">{item}</span>
                  </label>
                ))}
              </div>
            </div>
          ))}

          <div className="pt-4 mt-4 border-t border-opacity-20">
            <div className="flex justify-between items-center mb-4 text-sm opacity-70">
              <span>Progresso</span>
              <span>{checkedCount} / {totalItems}</span>
            </div>
            <div className="w-full bg-gray-200 dark:bg-slate-700 rounded-full h-2.5 mb-6">
              <div className="bg-blue-600 h-2.5 rounded-full transition-all duration-500" style={{ width: `${(checkedCount / totalItems) * 100}%` }}></div>
            </div>

            <button 
              onClick={handleConfirm}
              disabled={!isAllChecked}
              className={`w-full py-4 rounded-xl font-bold transition-all text-white
                ${isAllChecked ? 'bg-emerald-600 hover:bg-emerald-700 shadow-lg' : 'bg-gray-400 dark:bg-slate-700 cursor-not-allowed opacity-50'}`}
            >
              Confirmar Checklist
            </button>
          </div>
        </div>
      )}
    </div>
  );
}

// --- Maintenance Routines ---
function MaintenanceRoutines({ getPrimaryColor, profile }) {
  const isDark = profile?.theme === 'dark';
  const cardBg = isDark ? 'bg-slate-800 border-slate-700' : 'bg-white border-gray-200';
  const [activeSubTab, setActiveSubTab] = useState('Pós-Uso');

  const routines = {
    'Pós-Uso': ['Adoçamento do Jet', 'Lavagem Externa', 'Limpeza do Motor', 'Secar Porão'],
    'Semanal': ['Bateria', 'Nível do Óleo', 'Ligar o Jet 5s', 'Bujões'],
    'Quinzenal': ['Turbina', 'Direção', 'Braçadeiras', 'Sensores'],
    'Mensal': ['Lubrificar', 'Fusíveis', 'Velas', 'Casco']
  };

  return (
    <div className={`p-6 rounded-2xl border ${cardBg} min-h-[60vh] animate-fade-in`}>
      <h2 className="text-2xl font-bold mb-6 flex items-center gap-2">
        <Wrench className="w-6 h-6" />
        Rotinas de Manutenção
      </h2>

      <div className="flex overflow-x-auto gap-2 mb-6 pb-2 scrollbar-hide">
        {Object.keys(routines).map(tab => (
          <button
            key={tab}
            onClick={() => setActiveSubTab(tab)}
            className={`px-4 py-2 rounded-lg font-medium whitespace-nowrap transition-colors
              ${activeSubTab === tab ? getPrimaryColor() : isDark ? 'bg-slate-700' : 'bg-gray-100'}`}
          >
            {tab}
          </button>
        ))}
      </div>

      <div className="space-y-4">
        {routines[activeSubTab].map((task, idx) => (
          <div key={idx} className={`p-4 rounded-xl border flex items-center gap-4 ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-200'}`}>
            <div className="w-10 h-10 rounded-full bg-blue-500/10 text-blue-500 flex items-center justify-center">
              <Check className="w-5 h-5" />
            </div>
            <span className="font-medium text-lg">{task}</span>
          </div>
        ))}
      </div>
    </div>
  );
}

// --- Navigation History ---
function NavHistory({ navHistory, profile, getPrimaryTextColor }) {
  const isDark = profile?.theme === 'dark';
  const cardBg = isDark ? 'bg-slate-800 border-slate-700' : 'bg-white border-gray-200';

  // Simple aggregation for chart (last 7 entries)
  const chartData = [...navHistory].reverse().slice(-7);
  const maxDuration = Math.max(...chartData.map(d => d.duration), 10);

  return (
    <div className="space-y-6 animate-fade-in">
      
      {/* Chart Section */}
      <div className={`p-6 rounded-2xl border ${cardBg}`}>
        <h3 className="text-lg font-bold mb-6 flex items-center gap-2">
          <Activity className="w-5 h-5" />
          Gráfico de Uso (Minutos)
        </h3>
        {chartData.length > 0 ? (
          <div className="flex items-end gap-2 h-48 mt-4">
            {chartData.map((data, i) => {
              const height = (data.duration / maxDuration) * 100;
              return (
                <div key={i} className="flex-1 flex flex-col items-center gap-2 group">
                  <div className="w-full relative bg-blue-500 rounded-t-sm transition-all duration-500 hover:bg-blue-400" style={{ height: `${height}%` }}>
                    <span className="absolute -top-6 left-1/2 -translate-x-1/2 text-xs font-bold opacity-0 group-hover:opacity-100 transition-opacity">
                      {data.duration}m
                    </span>
                  </div>
                  <span className="text-[10px] opacity-60 rotate-45 transform origin-left truncate w-full mt-2">
                    {new Date(data.date).toLocaleDateString('pt-BR', {day: '2-digit', month: '2-digit'})}
                  </span>
                </div>
              );
            })}
          </div>
        ) : (
          <div className="h-48 flex items-center justify-center opacity-50">Nenhum dado para exibir no gráfico.</div>
        )}
      </div>

      {/* List Section */}
      <div className={`p-6 rounded-2xl border ${cardBg}`}>
        <h3 className="text-lg font-bold mb-4">Histórico Detalhado</h3>
        {navHistory.length === 0 ? (
          <p className="opacity-60 text-center py-8">Nenhuma navegação registrada ainda.</p>
        ) : (
          <div className="space-y-3">
            {navHistory.map(nav => (
              <div key={nav.id} className={`p-4 rounded-xl border flex flex-col md:flex-row justify-between gap-4 ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-200'}`}>
                <div className="flex flex-col">
                  <span className="font-bold text-lg">{nav.local}</span>
                  <span className="text-sm opacity-70">{new Date(nav.date).toLocaleDateString('pt-BR')}</span>
                </div>
                <div className="grid grid-cols-2 gap-4 text-sm">
                  <div>
                    <p className="opacity-70">Tempo</p>
                    <p className="font-semibold">{nav.duration} min</p>
                  </div>
                  <div>
                    <p className="opacity-70">Água</p>
                    <p className="font-semibold">{nav.waterType}</p>
                  </div>
                  <div>
                    <p className="opacity-70">Consumo</p>
                    <p className={`font-semibold ${nav.consumption === 'Alto' ? 'text-red-500' : nav.consumption === 'Baixo' ? 'text-emerald-500' : 'text-yellow-500'}`}>
                      {nav.consumption}
                    </p>
                  </div>
                </div>
              </div>
            ))}
          </div>
        )}
      </div>
    </div>
  );
}

// --- Maintenance History ---
function MaintHistory({ maintHistory, profile }) {
  const isDark = profile?.theme === 'dark';
  const cardBg = isDark ? 'bg-slate-800 border-slate-700' : 'bg-white border-gray-200';

  return (
    <div className={`p-6 rounded-2xl border ${cardBg} min-h-[60vh] animate-fade-in`}>
      <h2 className="text-2xl font-bold mb-6 flex items-center gap-2">
        <History className="w-6 h-6" />
        Registros de Checklists
      </h2>

      {maintHistory.length === 0 ? (
        <p className="opacity-60 text-center py-12">Nenhum checklist registrado ainda.</p>
      ) : (
        <div className="space-y-4">
          {maintHistory.map(maint => (
            <div key={maint.id} className={`p-5 rounded-xl border ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-200'}`}>
              <div className="flex justify-between items-center mb-3">
                <h3 className="font-bold text-lg text-emerald-500 flex items-center gap-2">
                  <Shield className="w-5 h-5" />
                  {maint.type}
                </h3>
                <span className="text-sm opacity-60">
                  {new Date(maint.timestamp).toLocaleString('pt-BR')}
                </span>
              </div>
              <div className="flex flex-wrap gap-2">
                {maint.items && maint.items.map((item, idx) => (
                  <span key={idx} className={`text-xs px-2 py-1 rounded-full ${isDark ? 'bg-slate-800' : 'bg-white border border-gray-300'}`}>
                    {item}
                  </span>
                ))}
              </div>
            </div>
          ))}
        </div>
      )}
    </div>
  );
}

// --- Dicas do Capitão (AI Chat) ---
function CaptainsChat({ user, profile, getPrimaryColor }) {
  const isDark = profile?.theme === 'dark';
  const cardBg = isDark ? 'bg-slate-800 border-slate-700' : 'bg-white border-gray-200';
  
  const [messages, setMessages] = useState([
    { role: 'model', text: 'Olá, marujo! Eu sou o Capitão, inteligência artificial do JetPrime. Como posso ajudar com seu Jet Ski hoje? Dúvidas de manutenção, dicas de navegação ou alertas de segurança?' }
  ]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const messagesEndRef = useRef(null);

  const scrollToBottom = () => {
    messagesEndRef.current?.scrollIntoView({ behavior: "smooth" });
  };

  useEffect(scrollToBottom, [messages]);

  const handleSend = async (e) => {
    e.preventDefault();
    if (!input.trim() || loading) return;

    const userMsg = { role: 'user', text: input };
    setMessages(prev => [...prev, userMsg]);
    setInput('');
    setLoading(true);

    try {
      // System prompt injection via contents structure
      const systemInstruction = "Você é o Capitão, um especialista náutico altamente experiente em Jet Skis. Seu objetivo é ajudar usuários do app JetPrime com dicas de manutenção, segurança, navegação e solução de problemas mecânicos. Responda de forma direta, amigável, como um lobo do mar experiente, e sempre em português brasileiro.";
      
      const payload = {
        contents: [...messages, userMsg].map(m => ({
          role: m.role,
          parts: [{ text: m.text }]
        })),
        systemInstruction: { parts: [{ text: systemInstruction }] }
      };

      const res = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${AI_API_KEY}`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload)
      });

      if (!res.ok) throw new Error("Falha na comunicação com o Capitão");

      const data = await res.json();
      const aiText = data.candidates?.[0]?.content?.parts?.[0]?.text || "Desculpe, o rádio falhou. Tente novamente marujo.";
      
      setMessages(prev => [...prev, { role: 'model', text: aiText }]);
    } catch (err) {
      console.error(err);
      setMessages(prev => [...prev, { role: 'model', text: "Erro de conexão com o sistema. Tente novamente mais tarde." }]);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className={`flex flex-col h-[75vh] md:h-[80vh] rounded-2xl border ${cardBg} overflow-hidden animate-fade-in`}>
      {/* Header */}
      <div className={`p-4 border-b flex items-center gap-3 ${isDark ? 'border-slate-700 bg-slate-800' : 'border-gray-200 bg-gray-50'}`}>
        <div className="w-10 h-10 rounded-full bg-blue-600 flex items-center justify-center text-white">
          <Anchor className="w-6 h-6" />
        </div>
        <div>
          <h2 className="font-bold">Dicas do Capitão</h2>
          <span className="text-xs text-emerald-500 font-medium flex items-center gap-1">
            <span className="w-2 h-2 rounded-full bg-emerald-500 inline-block"></span> Online (IA)
          </span>
        </div>
      </div>

      {/* Messages */}
      <div className="flex-1 overflow-y-auto p-4 space-y-4">
        {messages.map((msg, idx) => (
          <div key={idx} className={`flex ${msg.role === 'user' ? 'justify-end' : 'justify-start'}`}>
            <div className={`max-w-[80%] p-3 rounded-2xl ${
              msg.role === 'user' 
                ? `${getPrimaryColor()} rounded-tr-none text-white` 
                : `${isDark ? 'bg-slate-700 text-white' : 'bg-gray-200 text-black'} rounded-tl-none`
            }`}>
              <p className="whitespace-pre-wrap text-sm leading-relaxed">{msg.text}</p>
            </div>
          </div>
        ))}
        {loading && (
          <div className="flex justify-start">
            <div className={`max-w-[80%] p-3 rounded-2xl rounded-tl-none ${isDark ? 'bg-slate-700' : 'bg-gray-200'}`}>
              <div className="flex gap-1">
                <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce"></span>
                <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-75"></span>
                <span className="w-2 h-2 bg-gray-400 rounded-full animate-bounce delay-150"></span>
              </div>
            </div>
          </div>
        )}
        <div ref={messagesEndRef} />
      </div>

      {/* Input */}
      <form onSubmit={handleSend} className={`p-4 border-t flex gap-2 ${isDark ? 'border-slate-700 bg-slate-900' : 'border-gray-200 bg-white'}`}>
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Pergunte ao Capitão..."
          className={`flex-1 p-3 rounded-xl border outline-none ${isDark ? 'bg-slate-800 border-slate-700 focus:border-blue-500' : 'bg-gray-100 border-gray-300 focus:border-blue-500'}`}
        />
        <button type="submit" disabled={loading} className={`p-3 rounded-xl text-white transition-colors ${getPrimaryColor()} disabled:opacity-50 flex items-center justify-center w-14`}>
          <Send className="w-5 h-5" />
        </button>
      </form>
    </div>
  );
}

// --- Settings Modal ---
function SettingsModal({ user, jetSki, profile, getPrimaryColor, onClose }) {
  const isDark = profile?.theme === 'dark';
  const cardBg = isDark ? 'bg-slate-800 border-slate-700' : 'bg-white border-gray-200';

  const [formTheme, setFormTheme] = useState(profile?.theme || 'dark');
  const [formColor, setFormColor] = useState(profile?.color || 'blue');
  const [formFontSize, setFormFontSize] = useState(profile?.fontSize || 'text-base');
  
  const [marca, setMarca] = useState(jetSki?.marca || '');
  const [modelo, setModelo] = useState(jetSki?.modelo || '');
  const [ano, setAno] = useState(jetSki?.ano || '');

  const handleSave = async () => {
    if (!user) return;
    const profileRef = doc(db, 'artifacts', appId, 'users', user.uid, 'profile', 'info');
    await setDoc(profileRef, {
      jetSki: { marca, modelo, ano },
      settings: { theme: formTheme, color: formColor, fontSize: formFontSize }
    }, { merge: true });
    onClose();
  };

  const handleLogout = async () => {
    await signOut(auth);
  };

  return (
    <div className={`p-6 rounded-2xl border ${cardBg} animate-fade-in`}>
      <div className="flex justify-between items-center mb-6 border-b border-opacity-20 pb-4">
        <h2 className="text-2xl font-bold flex items-center gap-2">
          <Settings className="w-6 h-6" /> Configurações
        </h2>
        <button onClick={onClose} className="p-2 bg-gray-500/10 rounded-full hover:bg-gray-500/20">
          <ChevronRight className="w-6 h-6" />
        </button>
      </div>

      <div className="space-y-8">
        {/* Jet Ski Data */}
        <section>
          <h3 className="text-lg font-semibold mb-4 opacity-90">Dados do Jet Ski</h3>
          <div className="grid grid-cols-1 md:grid-cols-3 gap-4">
            <div>
              <label className="block text-sm mb-1 opacity-70">Marca</label>
              <input type="text" value={marca} onChange={e => setMarca(e.target.value)} className={`w-full p-3 rounded-lg border ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-300'}`} />
            </div>
            <div>
              <label className="block text-sm mb-1 opacity-70">Modelo</label>
              <input type="text" value={modelo} onChange={e => setModelo(e.target.value)} className={`w-full p-3 rounded-lg border ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-300'}`} />
            </div>
            <div>
              <label className="block text-sm mb-1 opacity-70">Ano</label>
              <input type="number" value={ano} onChange={e => setAno(e.target.value)} className={`w-full p-3 rounded-lg border ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-300'}`} />
            </div>
          </div>
        </section>

        {/* Appearance */}
        <section>
          <h3 className="text-lg font-semibold mb-4 opacity-90">Aparência</h3>
          
          <div className="space-y-4">
            <div>
              <label className="block text-sm mb-2 opacity-70">Tema (Modo Noturno)</label>
              <div className="flex gap-2">
                <button onClick={() => setFormTheme('light')} className={`flex-1 py-2 px-4 rounded-lg flex items-center justify-center gap-2 border ${formTheme === 'light' ? 'border-blue-500 bg-blue-500/10' : 'border-gray-500/30'}`}>
                  <Sun className="w-5 h-5" /> Claro
                </button>
                <button onClick={() => setFormTheme('dark')} className={`flex-1 py-2 px-4 rounded-lg flex items-center justify-center gap-2 border ${formTheme === 'dark' ? 'border-blue-500 bg-blue-500/10' : 'border-gray-500/30'}`}>
                  <Moon className="w-5 h-5" /> Escuro
                </button>
              </div>
            </div>

            <div>
              <label className="block text-sm mb-2 opacity-70">Cor Principal</label>
              <div className="flex gap-3">
                {['blue', 'red', 'green', 'purple'].map(color => (
                  <button 
                    key={color} 
                    onClick={() => setFormColor(color)}
                    className={`w-10 h-10 rounded-full flex items-center justify-center 
                      ${color === 'blue' ? 'bg-blue-500' : color === 'red' ? 'bg-red-500' : color === 'green' ? 'bg-emerald-500' : 'bg-purple-500'}`}
                  >
                    {formColor === color && <Check className="w-5 h-5 text-white" />}
                  </button>
                ))}
              </div>
            </div>

            <div>
              <label className="block text-sm mb-2 opacity-70">Tamanho da Fonte</label>
              <select value={formFontSize} onChange={e => setFormFontSize(e.target.value)} className={`w-full p-3 rounded-lg border outline-none ${isDark ? 'bg-slate-900 border-slate-700' : 'bg-gray-50 border-gray-300'}`}>
                <option value="text-sm">Pequena</option>
                <option value="text-base">Normal</option>
                <option value="text-lg">Grande</option>
              </select>
            </div>
          </div>
        </section>

        <div className="pt-6 border-t border-opacity-20 flex flex-col gap-3">
          <button onClick={handleSave} className={`w-full py-4 rounded-xl font-bold text-white transition-colors ${getPrimaryColor()}`}>
            Salvar Configurações
          </button>
          
          <button onClick={handleLogout} className="w-full py-4 rounded-xl font-bold text-red-500 bg-red-500/10 hover:bg-red-500/20 transition-colors flex items-center justify-center gap-2">
            <LogOut className="w-5 h-5" /> Sair da Conta
          </button>
        </div>
      </div>
    </div>
  );
}
