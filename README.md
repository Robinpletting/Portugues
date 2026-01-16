import React, { useState, useEffect } from 'react';
import { initializeApp } from 'firebase/app';
import { getAuth, signInAnonymously, signInWithCustomToken, onAuthStateChanged } from 'firebase/auth';
import { getFirestore, doc, setDoc, onSnapshot, collection, updateDoc, getDoc, deleteField } from 'firebase/firestore';

// --- CONFIGURATIE ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'port-pro-level-resume-v1';

const DEFAULT_PINS = {
  "Robin": "2024", "Isis": "1109", "Yuna": "8822", "Edric": "4590",
  "Judith": "3312", "Gea": "7765", "Michiel": "9012"
};

const CATS = ['A1', 'A2', 'B1', 'B2', 'C1', 'C2'];
const ADMIN_SECRET = "115639";
const UNLOCK_THRESHOLD = 80;
const QUESTIONS_PER_LEVEL = 40;

const RAW_VOCAB = {
  A1: [["Olá", "Hallo"], ["Bom dia", "Goedemorgen"], ["Obrigado", "Bedankt"], ["Sim", "Ja"], ["Não", "Nee"], ["Pão", "Brood"], ["Maçã", "Appel"], ["Água", "Water"], ["Leite", "Melk"], ["Café", "Koffie"], ["Mãe", "Moeder"], ["Pai", "Vader"], ["Filho", "Zoon"], ["Filha", "Dochter"], ["Criança", "Kind"], ["Casa", "Huis"], ["Escola", "School"], ["Carro", "Auto"], ["Livro", "Boek"], ["Caneta", "Pen"], ["Cão", "Hond"], ["Gato", "Kat"], ["Amigo", "Vriend"], ["Hoje", "Vandaag"], ["Amanhã", "Morgen"], ["Noite", "Nacht"], ["Dia", "Dag"], ["Hora", "Uur"], ["Minuto", "Minuut"], ["Segundo", "Seconde"], ["Vermelho", "Rood"], ["Azul", "Blauw"], ["Verde", "Groen"], ["Amarelo", "Geel"], ["Preto", "Zwart"], ["Branco", "Wit"], ["Um", "Een"], ["Dois", "Twee"], ["Três", "Drie"], ["Quatro", "Vier"]],
  A2: [["Caminhar", "Wandelen"], ["Correr", "Rennen"], ["Dormir", "Slapen"], ["Comer", "Eten"], ["Beber", "Drinken"], ["Cidade", "Stad"], ["Aldeia", "Dorp"], ["Rua", "Straat"], ["Praia", "Strand"], ["Campo", "Platteland"], ["Quente", "Warm"], ["Frio", "Koud"], ["Grande", "Groot"], ["Pequeno", "Klein"], ["Novo", "Nieuw"], ["Velho", "Oud"], ["Rápido", "Snel"], ["Lento", "Langzaam"], ["Fácil", "Makkelijk"], ["Difícil", "Moeilijk"], ["Feliz", "Gelukkig"], ["Triste", "Droevig"], ["Forte", "Sterk"], ["Fraco", "Zwak"], ["Rico", "Rijk"], ["Pobre", "Arm"], ["Barato", "Goedkoop"], ["Caro", "Duur"], ["Limpo", "Schoon"], ["Sujo", "Vuil"], ["Lindo", "Mooi"], ["Feio", "Lelijk"], ["Aberto", "Open"], ["Fechado", "Gesloten"], ["Perto", "Dichtbij"], ["Longe", "Ver"], ["Antes", "Voorheen"], ["Depois", "Daarna"], ["Sempre", "Altijd"], ["Nunca", "Nooit"]]
};

export default function App() {
  const [user, setUser] = useState(null);
  const [activeUser, setActiveUser] = useState(localStorage.getItem('port_pro_user') || null);
  const [profile, setProfile] = useState(null);
  const [cat, setCat] = useState('A1');
  const [quiz, setQuiz] = useState(null);
  const [feedback, setFeedback] = useState(null); 
  const [result, setResult] = useState(null);
  const [showKeypad, setShowKeypad] = useState(false);
  const [adminInput, setAdminInput] = useState("");
  const [loginPin, setLoginPin] = useState("");
  const [selectedUserLogin, setSelectedUserLogin] = useState("");
  const [isAdminOpen, setIsAdminOpen] = useState(false);
  const [dbUsers, setDbUsers] = useState([]);
  const [loading, setLoading] = useState(true);
  
  // State voor het level dat aangeklikt is
  const [pendingLevel, setPendingLevel] = useState(null);

  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    return onAuthStateChanged(auth, setUser);
  }, []);

  useEffect(() => {
    if (!user || !activeUser) { 
      if (!activeUser) setLoading(false);
      return; 
    }
    const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', activeUser);
    return onSnapshot(docRef, (snap) => {
      if (snap.exists()) {
        setProfile(snap.data());
      } else {
        setDoc(docRef, { name: activeUser, xp: 0, scores: {}, pin: DEFAULT_PINS[activeUser] || "0000" });
      }
      setLoading(false);
    });
  }, [user, activeUser]);

  useEffect(() => {
    if (isAdminOpen && user) {
      return onSnapshot(collection(db, 'artifacts', appId, 'public', 'data', 'users'), (snap) => {
        setDbUsers(snap.docs.map(d => ({ id: d.id, ...d.data() })));
      });
    }
  }, [isAdminOpen, user]);

  const isUnlocked = (category, lvlIdx) => {
    if (isAdminOpen) return true;
    const catIdx = CATS.indexOf(category);
    if (catIdx === 0 && lvlIdx === 0) return true;
    if (lvlIdx > 0) return (profile?.scores?.[`${category}-${lvlIdx - 1}`] || 0) >= UNLOCK_THRESHOLD;
    if (catIdx > 0) return (profile?.scores?.[`${CATS[catIdx - 1]}-19`] || 0) >= UNLOCK_THRESHOLD;
    return false;
  };

  // Functie die wordt aangeroepen bij het klikken op een level
  const handleLevelClick = (lvlIdx) => {
    // Check of er een opgeslagen sessie is in het profiel
    if (profile?.activeSession) {
      setPendingLevel(lvlIdx); // Open de "Hervatten?" modal
    } else {
      startLevel(lvlIdx); // Start direct nieuw level
    }
  };

  const startLevel = async (lvlIdx, resumeData = null) => {
    setPendingLevel(null);
    if (resumeData) {
      setQuiz(resumeData);
      return;
    }

    const pool = RAW_VOCAB[cat] || RAW_VOCAB['A1'];
    let baseItems = [...pool].sort(() => Math.random() - 0.5);
    while(baseItems.length < QUESTIONS_PER_LEVEL) baseItems = [...baseItems, ...pool].sort(() => Math.random() - 0.5);
    
    const finalItems = baseItems.slice(0, QUESTIONS_PER_LEVEL);
    const questions = finalItems.map((item, idx) => {
      const ptToNl = idx % 2 === 0;
      const correct = ptToNl ? item[1] : item[0];
      const distractors = pool.map(p => ptToNl ? p[1] : p[0]).filter(v => v !== correct).sort(() => Math.random() - 0.5).slice(0, 3);
      return { q: ptToNl ? item[0] : item[1], a: correct, options: [correct, ...distractors].sort(() => Math.random() - 0.5) };
    });

    const newQuiz = { cat, lvlIdx, questions, current: 0, correct: 0, wrong: 0, total: QUESTIONS_PER_LEVEL };
    setQuiz(newQuiz);
    
    if (activeUser) {
      await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', activeUser), {
        activeSession: newQuiz
      });
    }
  };

  const handleAnswer = (selected) => {
    if (feedback) return;
    const isCorrect = selected === quiz.questions[quiz.current].a;
    setFeedback({ selected, isCorrect });
    
    setTimeout(async () => {
      const nextCorrect = isCorrect ? quiz.correct + 1 : quiz.correct;
      const nextWrong = isCorrect ? quiz.wrong : quiz.wrong + 1;
      const nextCurrent = quiz.current + 1;
      
      if (nextCurrent < quiz.total) {
        const updatedQuiz = { ...quiz, current: nextCurrent, correct: nextCorrect, wrong: nextWrong };
        setQuiz(updatedQuiz);
        setFeedback(null);
        if (activeUser) {
          updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', activeUser), {
            activeSession: updatedQuiz
          });
        }
      } else {
        const score = Math.round((nextCorrect / quiz.total) * 100);
        if (activeUser) {
          const docRef = doc(db, 'artifacts', appId, 'public', 'data', 'users', activeUser);
          await updateDoc(docRef, {
            xp: (profile?.xp || 0) + (nextCorrect * 5),
            scores: { ...profile?.scores, [`${cat}-${quiz.lvlIdx}`]: Math.max(score, profile?.scores?.[`${cat}-${quiz.lvlIdx}`] || 0) },
            activeSession: deleteField()
          });
        }
        setResult({ score, xp: nextCorrect * 5, correct: nextCorrect, wrong: nextWrong });
        setQuiz(null); setFeedback(null);
      }
    }, 800);
  };

  const handleLogin = async (val) => {
    if (val === 'C') { setLoginPin(""); return; }
    const newPin = loginPin + val;
    if (newPin.length <= 4) setLoginPin(newPin);
    if (newPin.length === 4) {
      const snap = await getDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', selectedUserLogin));
      const targetPin = snap.exists() ? snap.data().pin : DEFAULT_PINS[selectedUserLogin];
      if (targetPin === newPin) {
        localStorage.setItem('port_pro_user', selectedUserLogin);
        setActiveUser(selectedUserLogin);
        setLoginPin("");
      } else { setLoginPin(""); }
    }
  };

  if (loading) return <div className="min-h-screen flex flex-col items-center justify-center bg-[#fdfaf6] font-black italic text-2xl animate-pulse">CARREGANDO...</div>;

  return (
    <div className={`min-h-screen transition-all duration-300 ${feedback ? (feedback.isCorrect ? 'bg-emerald-400' : 'bg-rose-500') : 'bg-[#fdfaf6]'} text-[#2d2d2d]`} style={{ fontFamily: "'Special Elite', serif" }}>
      <link href="https://fonts.googleapis.com/css2?family=Special+Elite&display=swap" rel="stylesheet" />

      {!activeUser ? (
        /* LOGIN SCHERM */
        <div className="flex min-h-screen items-center justify-center p-6">
          <div className="w-full max-w-sm border-4 border-[#2d2d2d] p-10 bg-white shadow-[12px_12px_0px_#2d2d2d] rounded-[3rem]">
            <h1 className="text-3xl font-black mb-10 text-center uppercase italic tracking-tighter">Portu Pro</h1>
            {!selectedUserLogin ? (
              <div className="space-y-4">
                {Object.keys(DEFAULT_PINS).map(u => (
                  <button key={u} onClick={() => setSelectedUserLogin(u)} className="w-full py-5 border-4 border-[#2d2d2d] rounded-full font-black uppercase text-lg shadow-[4px_4px_0px_#2d2d2d] active:translate-y-1 active:shadow-none transition-all">{u}</button>
                ))}
              </div>
            ) : (
              <div>
                <button onClick={() => {setSelectedUserLogin(""); setLoginPin("");}} className="text-[10px] font-black uppercase mb-8 opacity-40">← Terug</button>
                <p className="text-center font-black mb-6 uppercase text-2xl italic underline">{selectedUserLogin}</p>
                <div className="flex justify-center gap-4 mb-10">
                  {[...Array(4)].map((_, i) => (
                    <div key={i} className={`w-6 h-6 rounded-full border-4 border-[#2d2d2d] transition-all ${loginPin.length > i ? 'bg-[#2d2d2d] scale-125' : 'bg-transparent'}`}></div>
                  ))}
                </div>
                <div className="grid grid-cols-3 gap-4">
                  {[1, 2, 3, 4, 5, 6, 7, 8, 9, 'C', 0].map(n => (
                    <button key={n} onClick={() => handleLogin(n)} className="h-16 border-4 border-[#2d2d2d] rounded-full font-black text-2xl shadow-[4px_4px_0px_#2d2d2d] active:shadow-none active:translate-y-1">{n}</button>
                  ))}
                </div>
              </div>
            )}
          </div>
        </div>
      ) : quiz ? (
        /* QUIZ ENGINE */
        <div className="max-w-xl mx-auto w-full flex flex-col h-screen p-4">
          <div className="flex flex-col gap-3 mt-4">
            <div className="flex justify-between items-center border-4 border-[#2d2d2d] p-4 bg-white shadow-[6px_6px_0px_#2d2d2d] rounded-[1.5rem]">
               <button onClick={() => setQuiz(null)} className="bg-zinc-800 text-white px-4 py-2 rounded-full text-[10px] font-black border-2 border-black">PAUZE</button>
               <div className="flex gap-4">
                  <div className="flex items-center gap-2">
                    <span className="w-3 h-3 bg-emerald-500 rounded-full border-2 border-black"></span>
                    <span className="font-black text-sm">{quiz.correct}</span>
                  </div>
                  <div className="flex items-center gap-2">
                    <span className="w-3 h-3 bg-rose-500 rounded-full border-2 border-black"></span>
                    <span className="font-black text-sm">{quiz.wrong}</span>
                  </div>
               </div>
               <span className="font-black text-xs opacity-50">{quiz.current + 1}/40</span>
            </div>
            <div className="w-full h-3 bg-zinc-200 border-4 border-[#2d2d2d] rounded-full overflow-hidden">
                <div className="h-full bg-[#2d2d2d] transition-all duration-500" style={{ width: `${((quiz.current + 1) / quiz.total) * 100}%` }}></div>
            </div>
          </div>
          <div className="flex-1 flex flex-col items-center justify-center">
            <div className={`bg-white border-4 border-[#2d2d2d] p-8 w-full shadow-[12px_12px_0px_#2d2d2d] rounded-[3rem] min-h-[480px] flex flex-col transition-all duration-300 ${feedback ? 'scale-95 opacity-90' : 'scale-100'}`}>
              <div className="flex-1 flex items-center justify-center text-center p-4">
                <h2 className="text-4xl sm:text-5xl font-black uppercase leading-tight tracking-tighter italic">{quiz.questions[quiz.current].q}</h2>
              </div>
              <div className="grid gap-3 mt-8">
                {quiz.questions[quiz.current].options.map((opt, i) => (
                  <button key={i} onClick={() => handleAnswer(opt)} 
                    className={`w-full py-5 border-4 rounded-full font-black uppercase text-sm transition-all
                    ${feedback ? (opt === quiz.questions[quiz.current].a ? 'bg-emerald-400 border-black' : (opt === feedback.selected ? 'bg-rose-500 border-black text-white' : 'opacity-20 grayscale')) 
                    : 'bg-white border-[#2d2d2d] shadow-[5px_5px_0px_#2d2d2d] active:translate-y-1 active:shadow-none'}`}>
                    {opt}
                  </button>
                ))}
              </div>
            </div>
          </div>
        </div>
      ) : (
        /* DASHBOARD */
        <div className="max-w-6xl mx-auto w-full p-6 flex flex-col flex-1">
          <header className="flex justify-between items-end border-b-8 border-[#2d2d2d] pb-10 mb-12">
            <div>
              <p className="text-[11px] font-black uppercase opacity-50 mb-2 tracking-[0.4em]">Estudante:</p>
              <h1 className="text-5xl font-black uppercase italic tracking-tighter">{activeUser}</h1>
            </div>
            <div className="flex items-center gap-6">
              <div className="bg-yellow-400 border-4 border-[#2d2d2d] px-10 py-4 font-black rounded-full shadow-[6px_6px_0px_#2d2d2d] text-2xl tracking-tighter">{profile?.xp || 0} XP</div>
              <button onClick={() => { localStorage.removeItem('port_pro_user'); setActiveUser(null); }} className="w-16 h-16 flex items-center justify-center border-4 border-[#2d2d2d] rounded-full bg-white shadow-[6px_6px_0px_#2d2d2d] active:translate-y-1 active:shadow-none">
                <svg className="w-8 h-8" fill="none" stroke="currentColor" viewBox="0 0 24 24"><path strokeLinecap="round" strokeLinejoin="round" strokeWidth="4" d="M17 16l4-4m0 0l-4-4m4 4H7"/></svg>
              </button>
            </div>
          </header>

          <nav className="flex gap-4 overflow-x-auto pb-10 no-scrollbar mb-10">
            {CATS.map(c => (
              <button key={c} onClick={() => setCat(c)} className={`px-14 py-6 border-4 border-[#2d2d2d] font-black uppercase rounded-full transition-all flex-shrink-0 text-xl ${cat === c ? 'bg-[#2d2d2d] text-white shadow-[8px_8px_0px_#fbbf24] scale-110' : 'bg-white shadow-[6px_6px_0px_#2d2d2d]'}`}>
                {c}
              </button>
            ))}
          </nav>

          <div className="grid grid-cols-2 sm:grid-cols-3 md:grid-cols-4 lg:grid-cols-5 gap-8 mb-32">
            {[...Array(20)].map((_, i) => {
              const score = profile?.scores?.[`${cat}-${i}`] || 0;
              const unlocked = isUnlocked(cat, i);
              return (
                <div key={i} onClick={() => unlocked && handleLevelClick(i)} 
                  className={`border-4 border-[#2d2d2d] p-8 rounded-[2.5rem] transition-all relative flex flex-col items-center justify-center text-center
                  ${unlocked ? 'cursor-pointer bg-white shadow-[10px_10px_0px_#2d2d2d] active:translate-y-1 active:shadow-none hover:bg-zinc-50' : 'opacity-20 grayscale bg-zinc-200 pointer-events-none'}`}>
                  <div className={`w-14 h-14 rounded-full border-4 border-[#2d2d2d] flex items-center justify-center mb-4 font-black text-2xl ${score >= UNLOCK_THRESHOLD ? 'bg-emerald-400' : 'bg-zinc-100'}`}>
                    {i+1}
                  </div>
                  <div className="text-[11px] font-black uppercase opacity-60 tracking-widest">Level {i+1}</div>
                  <div className="font-black text-lg mt-2 underline">{score > 0 ? `${score}%` : '--'}</div>
                </div>
              );
            })}
          </div>

          <footer className="mt-auto py-16 text-center">
            <button onClick={() => setShowKeypad(true)} className="text-[11px] font-black uppercase tracking-[0.8em] opacity-10 hover:opacity-100 transition-opacity">ADMIN CONSOLE</button>
          </footer>
        </div>
      )}

      {/* HERVATTEN MODAL - PAS ZICHTBAAR BIJ KLIK OP LEVEL */}
      {pendingLevel !== null && profile?.activeSession && (
        <div className="fixed inset-0 bg-black/80 z-[950] flex items-center justify-center p-6 backdrop-blur-md">
          <div className="bg-white border-8 border-[#2d2d2d] p-12 w-full max-w-md rounded-[4rem] shadow-[25px_25px_0px_#000] text-center">
            <h2 className="text-4xl font-black uppercase italic mb-4 leading-none text-emerald-600 tracking-tighter">VOORTGANG GEVONDEN</h2>
            <div className="bg-zinc-100 p-6 rounded-3xl mb-8 border-4 border-[#2d2d2d]">
               <p className="font-black text-lg uppercase italic underline">
                 {profile.activeSession.cat} - LEVEL {profile.activeSession.lvlIdx + 1}
               </p>
               <p className="mt-2 text-xs font-bold opacity-70">VRAAG {profile.activeSession.current + 1} VAN 40</p>
               <div className="flex justify-center gap-6 mt-4">
                  <div className="flex items-center gap-2">
                    <span className="w-3 h-3 bg-emerald-500 rounded-full border-2 border-black"></span>
                    <span className="font-black text-sm">{profile.activeSession.correct} GOED</span>
                  </div>
                  <div className="flex items-center gap-2">
                    <span className="w-3 h-3 bg-rose-500 rounded-full border-2 border-black"></span>
                    <span className="font-black text-sm">{profile.activeSession.wrong} FOUT</span>
                  </div>
               </div>
            </div>
            
            <p className="mb-8 font-black text-xs uppercase opacity-60">Wil je deze sessie afmaken of opnieuw beginnen met Level {pendingLevel + 1}?</p>

            <div className="grid gap-4">
              <button onClick={() => startLevel(profile.activeSession.lvlIdx, profile.activeSession)} className="w-full bg-emerald-400 border-4 border-black py-6 rounded-full font-black uppercase shadow-[6px_6px_0px_#000] active:translate-y-1">HERVATTEN</button>
              <button onClick={() => startLevel(pendingLevel)} className="w-full bg-white border-4 border-black py-4 rounded-full font-black uppercase text-xs shadow-[4px_4px_0px_#000] active:translate-y-1">NIEUWE START L{pendingLevel + 1}</button>
              <button onClick={() => setPendingLevel(null)} className="mt-2 text-[10px] font-black uppercase opacity-40">ANNULEREN</button>
            </div>
          </div>
        </div>
      )}

      {/* RESULT MODAL */}
      {result && (
        <div className="fixed inset-0 bg-black/95 flex items-center justify-center p-6 z-[900] backdrop-blur-3xl">
          <div className="bg-white border-[10px] border-[#2d2d2d] p-12 w-full max-w-md text-center rounded-[5rem] shadow-[30px_30px_0px_#000]">
            <h2 className="text-7xl font-black uppercase mb-6 italic tracking-tighter leading-none italic">
              {result.score >= UNLOCK_THRESHOLD ? "BOA!" : "TENTA NOVAMENTE"}
            </h2>
            <div className="bg-[#2d2d2d] text-white p-12 mb-10 rounded-[3rem] shadow-[15px_15px_0px_#fbbf24]">
              <p className={`text-8xl font-black mb-4 ${result.score >= UNLOCK_THRESHOLD ? 'text-emerald-400' : 'text-rose-400'}`}>{result.score}%</p>
              <div className="flex justify-center gap-6 font-black text-xs uppercase opacity-80 tracking-widest">
                <span className="text-emerald-400">{result.correct} GOED</span>
                <span className="text-rose-400">{result.wrong} FOUT</span>
              </div>
            </div>
            <button onClick={() => setResult(null)} className="w-full bg-sky-500 border-4 border-black text-white py-8 rounded-full font-black uppercase text-3xl shadow-[10px_10px_0px_#000] active:translate-y-1">CONTINUAR</button>
          </div>
        </div>
      )}

      {/* ADMIN ENTRY KEYPAD */}
      {showKeypad && (
        <div className="fixed inset-0 bg-black/90 z-[800] flex items-center justify-center p-6 backdrop-blur-xl">
          <div className="bg-white border-8 border-[#2d2d2d] p-12 w-full max-w-sm rounded-[4rem] shadow-[25px_25px_0px_#000]">
            <h2 className="text-center font-black uppercase mb-10 text-xs tracking-widest">Master Password</h2>
            <div className="flex justify-center gap-4 mb-12">
              {[...Array(6)].map((_, i) => (
                <div key={i} className={`w-5 h-5 rounded-full border-4 border-[#2d2d2d] ${adminInput.length > i ? 'bg-[#2d2d2d]' : 'bg-transparent'}`}></div>
              ))}
            </div>
            <div className="grid grid-cols-3 gap-5">
              {[1, 2, 3, 4, 5, 6, 7, 8, 9, 'C', 0, 'X'].map(k => (
                <button key={k} onClick={() => {
                  if (k === 'X') { setShowKeypad(false); setAdminInput(""); }
                  else if (k === 'C') setAdminInput("");
                  else if (adminInput.length < 6) {
                    const next = adminInput + k;
                    setAdminInput(next);
                    if (next === ADMIN_SECRET) { setIsAdminOpen(true); setShowKeypad(false); setAdminInput(""); }
                  }
                }} className="h-16 border-4 border-[#2d2d2d] rounded-full font-black text-2xl shadow-[6px_6px_0px_#2d2d2d] active:shadow-none active:translate-y-1">{k}</button>
              ))}
            </div>
          </div>
        </div>
      )}

      {/* ADMIN PANEL */}
      {isAdminOpen && (
        <div className="fixed inset-0 bg-[#fdfaf6] z-[700] p-8 border-[24px] border-[#2d2d2d] overflow-y-auto">
          <div className="max-w-5xl mx-auto w-full py-12">
            <div className="flex justify-between items-center mb-16 border-b-8 border-[#2d2d2d] pb-10">
              <h2 className="text-6xl font-black uppercase italic tracking-tighter">Admin Panel</h2>
              <button onClick={() => setIsAdminOpen(false)} className="bg-rose-500 border-4 border-black text-white px-12 py-5 rounded-full font-black uppercase shadow-[8px_8px_0px_#000]">SLUITEN</button>
            </div>
            <div className="grid gap-8">
              {Object.keys(DEFAULT_PINS).map(name => {
                const u = dbUsers.find(d => d.id === name);
                return (
                  <div key={name} className="border-8 border-[#2d2d2d] p-10 bg-white rounded-[3.5rem] shadow-[15px_15px_0px_#2d2d2d] flex flex-col md:flex-row justify-between items-center gap-10">
                    <div className="text-left flex-1">
                      <h3 className="text-4xl font-black uppercase italic">{name}</h3>
                      <div className="flex gap-8 mt-4 font-black text-sm uppercase opacity-70">
                        <span>XP: {u?.xp || 0}</span>
                        <span className="text-emerald-600">PIN: {u?.pin || DEFAULT_PINS[name]}</span>
                      </div>
                    </div>
                    <div className="flex flex-wrap gap-4">
                      <button onClick={async () => {
                        const next = prompt(`Nieuwe PIN voor ${name}:`, u?.pin || DEFAULT_PINS[name]);
                        if (next && next.length === 4) await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', name), { pin: next });
                      }} className="bg-emerald-100 border-4 border-black px-6 py-3 rounded-full font-black uppercase text-xs shadow-[4px_4px_0px_#000]">Edit PIN</button>
                      <button onClick={async () => {
                        const s = {}; CATS.forEach(c => { for(let j=0; j<20; j++) s[`${c}-${j}`] = 100; });
                        await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', name), { scores: s, xp: 10000 });
                      }} className="bg-yellow-400 border-4 border-black px-6 py-3 rounded-full font-black uppercase text-xs shadow-[4px_4px_0px_#000]">Unlock All</button>
                      <button onClick={async () => {
                        if(confirm(`Reset ${name}?`)) await updateDoc(doc(db, 'artifacts', appId, 'public', 'data', 'users', name), { scores: {}, xp: 0, activeSession: deleteField() });
                      }} className="bg-rose-500 text-white border-4 border-black px-6 py-3 rounded-full font-black uppercase text-xs shadow-[4px_4px_0px_#000]">Reset</button>
                    </div>
                  </div>
                );
              })}
            </div>
          </div>
        </div>
      )}
    </div>
  );
}

