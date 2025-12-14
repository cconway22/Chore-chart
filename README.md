import React, { useState, useEffect, useMemo } from 'react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  onAuthStateChanged,
  signInWithCustomToken
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  addDoc, 
  updateDoc, 
  deleteDoc, 
  doc, 
  query, 
  onSnapshot,
  serverTimestamp,
  orderBy
} from 'firebase/firestore';
import { 
  Star, 
  CheckCircle2, 
  Circle, 
  DollarSign, 
  Plus, 
  Trash2, 
  clipboard,
  NotebookPen,
  Trophy,
  LayoutList,
  Library,
  Save,
  X
} from 'lucide-react';

// --- Firebase Configuration ---
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'default-app-id';

// --- Types ---
interface Chore {
  id: string;
  title: string;
  value: number;
  isDone: boolean;
  kid: string;
  createdAt: any;
}

interface ChoreTemplate {
  id: string;
  title: string;
  defaultValue: number;
}

interface Responsibility {
  id: string;
  note: string;
  kid: string;
  createdAt: any;
}

const KIDS = [
  { name: 'Finley', color: 'bg-blue-100 text-blue-800', border: 'border-blue-200', check: 'text-blue-500', btn: 'bg-blue-500 hover:bg-blue-600' },
  { name: 'James', color: 'bg-green-100 text-green-800', border: 'border-green-200', check: 'text-green-500', btn: 'bg-green-500 hover:bg-green-600' },
  { name: 'Liam', color: 'bg-orange-100 text-orange-800', border: 'border-orange-200', check: 'text-orange-500', btn: 'bg-orange-500 hover:bg-orange-600' }
];

// --- Components ---

export default function App() {
  const [user, setUser] = useState<any>(null);
  const [activeKid, setActiveKid] = useState<string>(KIDS[0].name);
  const [activeTab, setActiveTab] = useState<'chores' | 'responsibility' | 'money'>('chores');
  
  // Data State
  const [chores, setChores] = useState<Chore[]>([]);
  const [responsibilities, setResponsibilities] = useState<Responsibility[]>([]);
  const [choreTemplates, setChoreTemplates] = useState<ChoreTemplate[]>([]);
  
  // UI State
  const [isAddChoreOpen, setIsAddChoreOpen] = useState(false);
  const [newChoreTitle, setNewChoreTitle] = useState('');
  const [newChoreValue, setNewChoreValue] = useState('');
  const [saveAsTemplate, setSaveAsTemplate] = useState(false);

  const [isAddRespOpen, setIsAddRespOpen] = useState(false);
  const [newRespNote, setNewRespNote] = useState('');

  // --- Auth & Data Fetching ---
  useEffect(() => {
    const initAuth = async () => {
      if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
        await signInWithCustomToken(auth, __initial_auth_token);
      } else {
        await signInAnonymously(auth);
      }
    };
    initAuth();
    const unsubscribe = onAuthStateChanged(auth, setUser);
    return () => unsubscribe();
  }, []);

  useEffect(() => {
    if (!user) return;

    // Fetch Chores
    const choresQuery = query(collection(db, 'artifacts', appId, 'users', user.uid, 'chores'));
    const unsubChores = onSnapshot(choresQuery, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() } as Chore));
      data.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0));
      setChores(data);
    }, (error) => console.error("Error fetching chores:", error));

    // Fetch Responsibilities
    const respQuery = query(collection(db, 'artifacts', appId, 'users', user.uid, 'responsibilities'));
    const unsubResp = onSnapshot(respQuery, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() } as Responsibility));
      data.sort((a, b) => (b.createdAt?.seconds || 0) - (a.createdAt?.seconds || 0));
      setResponsibilities(data);
    }, (error) => console.error("Error fetching responsibilities:", error));

    // Fetch Chore Templates (Library)
    const tplQuery = query(collection(db, 'artifacts', appId, 'users', user.uid, 'chore_templates'));
    const unsubTpl = onSnapshot(tplQuery, (snapshot) => {
      const data = snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() } as ChoreTemplate));
      data.sort((a, b) => a.title.localeCompare(b.title));
      setChoreTemplates(data);
    }, (error) => console.error("Error fetching templates:", error));

    return () => {
      unsubChores();
      unsubResp();
      unsubTpl();
    };
  }, [user]);

  // --- Actions ---

  const handleAddChore = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!newChoreTitle.trim() || !user) return;
    
    const val = parseFloat(newChoreValue) || 0;

    // Add the actual chore
    await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'chores'), {
      title: newChoreTitle,
      value: val,
      isDone: false,
      kid: activeKid,
      createdAt: serverTimestamp()
    });

    // Save as template if checked
    if (saveAsTemplate) {
      // Check if it already exists to avoid duplicates (simple check)
      const exists = choreTemplates.some(t => t.title.toLowerCase() === newChoreTitle.toLowerCase());
      if (!exists) {
        await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'chore_templates'), {
          title: newChoreTitle,
          defaultValue: val
        });
      }
    }

    setNewChoreTitle('');
    setNewChoreValue('');
    setSaveAsTemplate(false);
    setIsAddChoreOpen(false);
  };

  const deleteTemplate = async (id: string, e: React.MouseEvent) => {
    e.stopPropagation(); // Prevent clicking the template when deleting
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'chore_templates', id));
  };

  const selectTemplate = (tpl: ChoreTemplate) => {
    setNewChoreTitle(tpl.title);
    setNewChoreValue(tpl.defaultValue.toString());
    setSaveAsTemplate(false);
  };

  const toggleChore = async (chore: Chore) => {
    if (!user) return;
    const choreRef = doc(db, 'artifacts', appId, 'users', user.uid, 'chores', chore.id);
    await updateDoc(choreRef, { isDone: !chore.isDone });
  };

  const deleteChore = async (id: string) => {
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'chores', id));
  };

  const handleAddResponsibility = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!newRespNote.trim() || !user) return;

    await addDoc(collection(db, 'artifacts', appId, 'users', user.uid, 'responsibilities'), {
      note: newRespNote,
      kid: activeKid,
      createdAt: serverTimestamp()
    });
    setNewRespNote('');
    setIsAddRespOpen(false);
  };

  const deleteResponsibility = async (id: string) => {
    if (!user) return;
    await deleteDoc(doc(db, 'artifacts', appId, 'users', user.uid, 'responsibilities', id));
  };

  // --- Filtering ---
  const currentKidData = KIDS.find(k => k.name === activeKid) || KIDS[0];
  
  const filteredChores = useMemo(() => 
    chores.filter(c => c.kid === activeKid), 
  [chores, activeKid]);

  const filteredResp = useMemo(() => 
    responsibilities.filter(r => r.kid === activeKid), 
  [responsibilities, activeKid]);

  const earnings = useMemo(() => {
    const totals: Record<string, number> = { Finley: 0, James: 0, Liam: 0 };
    chores.forEach(c => {
      if (c.isDone && totals[c.kid] !== undefined) {
        totals[c.kid] += c.value;
      }
    });
    return totals;
  }, [chores]);

  // --- Render Helpers ---

  if (!user) return <div className="min-h-screen flex items-center justify-center bg-slate-50 text-slate-400">Loading Family Chart...</div>;

  return (
    <div className="min-h-screen bg-slate-50 pb-24 font-sans text-slate-800">
      
      {/* Header / Kid Switcher */}
      <div className="bg-white shadow-sm sticky top-0 z-10">
        <div className="max-w-md mx-auto px-4 py-4">
          <h1 className="text-center font-bold text-xl text-slate-700 mb-4">Family Chore Chart</h1>
          <div className="flex justify-between gap-2 p-1 bg-slate-100 rounded-xl">
            {KIDS.map(kid => (
              <button
                key={kid.name}
                onClick={() => setActiveKid(kid.name)}
                className={`flex-1 py-2 rounded-lg text-sm font-semibold transition-all ${
                  activeKid === kid.name 
                    ? 'bg-white text-slate-800 shadow-md ring-1 ring-black/5' 
                    : 'text-slate-400 hover:text-slate-600'
                }`}
              >
                {kid.name}
              </button>
            ))}
          </div>
        </div>
      </div>

      {/* Main Content Area */}
      <div className="max-w-md mx-auto px-4 pt-6">
        
        {/* --- CHORES TAB --- */}
        {activeTab === 'chores' && (
          <div className="space-y-4">
            <div className="flex items-center justify-between mb-2">
              <h2 className="text-xl font-bold text-slate-700">{activeKid}'s Chores</h2>
              <span className={`px-3 py-1 rounded-full text-sm font-bold ${currentKidData.color}`}>
                Earned: ${earnings[activeKid].toFixed(2)}
              </span>
            </div>

            {filteredChores.length === 0 && (
              <div className="text-center py-10 bg-white rounded-2xl border-2 border-dashed border-slate-200">
                <p className="text-slate-400">No chores yet!</p>
                <button 
                  onClick={() => setIsAddChoreOpen(true)}
                  className="mt-2 text-sm font-bold text-blue-500 hover:underline"
                >
                  Add the first chore
                </button>
              </div>
            )}

            <div className="space-y-3">
              {filteredChores.map(chore => (
                <div 
                  key={chore.id} 
                  className={`flex items-center p-4 bg-white rounded-xl shadow-sm border transition-all ${
                    chore.isDone ? 'border-green-100 bg-green-50/50' : 'border-slate-100'
                  }`}
                >
                  <button 
                    onClick={() => toggleChore(chore)}
                    className="flex-shrink-0 mr-4 transition-transform active:scale-95"
                  >
                    {chore.isDone ? (
                      <CheckCircle2 className={`w-8 h-8 ${currentKidData.check} fill-current bg-white rounded-full`} />
                    ) : (
                      <Circle className="w-8 h-8 text-slate-300" />
                    )}
                  </button>
                  
                  <div className="flex-1">
                    <h3 className={`font-semibold text-lg ${chore.isDone ? 'text-slate-400 line-through' : 'text-slate-700'}`}>
                      {chore.title}
                    </h3>
                    {chore.value > 0 && (
                      <span className={`text-sm font-medium ${chore.isDone ? 'text-slate-400' : 'text-slate-500'}`}>
                        ${chore.value.toFixed(2)}
                      </span>
                    )}
                  </div>

                  <button 
                    onClick={() => deleteChore(chore.id)}
                    className="p-2 text-slate-300 hover:text-red-400 transition-colors"
                  >
                    <Trash2 className="w-5 h-5" />
                  </button>
                </div>
              ))}
            </div>
            
            {/* Add Chore Button (Floating) */}
            <button
              onClick={() => setIsAddChoreOpen(true)}
              className={`fixed bottom-24 right-6 w-14 h-14 rounded-full shadow-lg shadow-blue-500/30 flex items-center justify-center text-white transition-transform hover:scale-105 active:scale-95 ${currentKidData.btn}`}
            >
              <Plus className="w-8 h-8" />
            </button>
          </div>
        )}

        {/* --- RESPONSIBILITY TAB --- */}
        {activeTab === 'responsibility' && (
          <div className="space-y-6">
            <div className="flex flex-col items-center justify-center py-8 bg-white rounded-2xl shadow-sm border border-slate-100">
              <h2 className="text-slate-500 font-medium mb-2">Checkmarks for {activeKid}</h2>
              <div className="flex items-end gap-2 text-6xl font-black text-slate-800 tracking-tighter">
                {filteredResp.length}
                <Star className={`w-12 h-12 mb-2 fill-yellow-400 text-yellow-500`} />
              </div>
              <button 
                onClick={() => setIsAddRespOpen(true)}
                className={`mt-6 px-6 py-2 rounded-full font-bold text-white shadow-md transition-transform active:scale-95 ${currentKidData.btn}`}
              >
                + Add Checkmark
              </button>
            </div>

            <div className="space-y-3">
              <h3 className="font-bold text-slate-700 ml-1">History</h3>
              {filteredResp.length === 0 ? (
                <p className="text-center text-slate-400 py-4">No responsibility notes yet.</p>
              ) : (
                filteredResp.map(resp => (
                  <div key={resp.id} className="bg-white p-4 rounded-xl shadow-sm border border-slate-100 flex gap-3">
                    <div className="mt-1 bg-yellow-100 p-1.5 rounded-full h-fit">
                      <Star className="w-4 h-4 text-yellow-600 fill-yellow-600" />
                    </div>
                    <div className="flex-1">
                      <p className="text-slate-700 font-medium">{resp.note}</p>
                      <p className="text-xs text-slate-400 mt-1">
                         {resp.createdAt ? new Date(resp.createdAt.seconds * 1000).toLocaleDateString() : 'Just now'}
                      </p>
                    </div>
                    <button 
                      onClick={() => deleteResponsibility(resp.id)}
                      className="text-slate-300 hover:text-red-400"
                    >
                      <Trash2 className="w-4 h-4" />
                    </button>
                  </div>
                ))
              )}
            </div>
          </div>
        )}

        {/* --- MONEY/EARNINGS TAB --- */}
        {activeTab === 'money' && (
          <div className="space-y-6">
            <h2 className="text-2xl font-bold text-slate-800 text-center mb-6">Weekly Earnings</h2>
            
            <div className="grid gap-4">
              {KIDS.map((kid, idx) => {
                const amount = earnings[kid.name];
                const isLeader = Math.max(...Object.values(earnings)) === amount && amount > 0;
                
                return (
                  <div key={kid.name} className={`relative overflow-hidden bg-white p-6 rounded-2xl shadow-sm border-2 transition-all ${isLeader ? 'border-yellow-400 ring-4 ring-yellow-400/10' : 'border-slate-100'}`}>
                    {isLeader && (
                      <div className="absolute top-0 right-0 bg-yellow-400 text-yellow-900 px-3 py-1 rounded-bl-xl text-xs font-bold flex items-center gap-1">
                        <Trophy className="w-3 h-3" /> LEADER
                      </div>
                    )}
                    <div className="flex justify-between items-center">
                      <div className="flex items-center gap-3">
                        <div className={`w-12 h-12 rounded-full flex items-center justify-center text-lg font-bold ${kid.color}`}>
                          {kid.name[0]}
                        </div>
                        <div>
                          <h3 className="font-bold text-slate-700 text-lg">{kid.name}</h3>
                          <p className="text-sm text-slate-400">Total Earned</p>
                        </div>
                      </div>
                      <div className="text-3xl font-black text-slate-800 tracking-tight">
                        ${amount.toFixed(2)}
                      </div>
                    </div>
                  </div>
                );
              })}
            </div>
            
            <div className="bg-blue-50 p-4 rounded-xl text-blue-800 text-sm border border-blue-100 mt-8">
              <p className="flex gap-2">
                <DollarSign className="w-5 h-5 flex-shrink-0" />
                This shows the total value of all completed chores. Unchecking a chore will remove it from this total.
              </p>
            </div>
          </div>
        )}
      </div>

      {/* --- MODALS --- */}
      
      {/* Add Chore Modal */}
      {isAddChoreOpen && (
        <div className="fixed inset-0 bg-black/50 z-50 flex items-end sm:items-center justify-center p-4">
          <form 
            onSubmit={handleAddChore}
            className="bg-white w-full max-w-sm rounded-2xl p-6 shadow-2xl animate-in slide-in-from-bottom-10 max-h-[80vh] flex flex-col"
          >
            <div className="flex justify-between items-center mb-4">
              <h3 className="text-lg font-bold">New Chore for {activeKid}</h3>
              <button 
                type="button"
                onClick={() => setIsAddChoreOpen(false)}
                className="text-slate-400 hover:text-slate-600"
              >
                <X className="w-5 h-5" />
              </button>
            </div>

            <div className="flex-1 overflow-y-auto min-h-0">
              {/* Chore Library Section */}
              {choreTemplates.length > 0 && (
                <div className="mb-6">
                  <div className="flex items-center gap-2 mb-2 text-xs font-bold text-slate-400 uppercase tracking-wider">
                    <Library className="w-3 h-3" />
                    Saved Chores
                  </div>
                  <div className="flex flex-wrap gap-2">
                    {choreTemplates.map(tpl => (
                      <button
                        key={tpl.id}
                        type="button"
                        onClick={() => selectTemplate(tpl)}
                        className={`group relative px-3 py-2 rounded-lg text-sm font-medium border transition-all text-left ${
                          newChoreTitle === tpl.title 
                            ? 'bg-blue-50 border-blue-200 text-blue-700' 
                            : 'bg-slate-50 border-slate-200 text-slate-600 hover:border-blue-300'
                        }`}
                      >
                        {tpl.title} <span className="opacity-50 ml-1">${tpl.defaultValue}</span>
                        <div 
                          onClick={(e) => deleteTemplate(tpl.id, e)}
                          className="absolute -top-1 -right-1 bg-white rounded-full p-0.5 shadow-sm border border-slate-200 opacity-0 group-hover:opacity-100 transition-opacity hover:text-red-500"
                        >
                          <X className="w-3 h-3" />
                        </div>
                      </button>
                    ))}
                  </div>
                </div>
              )}

              <div className="space-y-4">
                <div>
                  <label className="block text-sm font-medium text-slate-600 mb-1">Task Name</label>
                  <input
                    autoFocus
                    type="text"
                    placeholder="e.g. Clean Room"
                    value={newChoreTitle}
                    onChange={e => setNewChoreTitle(e.target.value)}
                    className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                </div>
                <div>
                  <label className="block text-sm font-medium text-slate-600 mb-1">Value ($)</label>
                  <input
                    type="number"
                    step="0.25"
                    placeholder="0.00"
                    value={newChoreValue}
                    onChange={e => setNewChoreValue(e.target.value)}
                    className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl focus:outline-none focus:ring-2 focus:ring-blue-500"
                  />
                  <p className="text-xs text-slate-400 mt-1">Adjust price for {activeKid} if needed.</p>
                </div>
                
                <div className="flex items-center gap-2 p-3 bg-slate-50 rounded-xl border border-slate-100">
                  <input 
                    type="checkbox"
                    id="saveTemplate"
                    checked={saveAsTemplate}
                    onChange={e => setSaveAsTemplate(e.target.checked)}
                    className="w-5 h-5 rounded text-blue-500 focus:ring-blue-500 border-gray-300"
                  />
                  <label htmlFor="saveTemplate" className="text-sm font-medium text-slate-600 flex items-center gap-2 cursor-pointer select-none">
                    <Save className="w-4 h-4" />
                    Save to Chore Library
                  </label>
                </div>
              </div>
            </div>

            <div className="flex gap-3 mt-6 pt-4 border-t border-slate-100">
              <button 
                type="button" 
                onClick={() => setIsAddChoreOpen(false)}
                className="flex-1 py-3 font-semibold text-slate-600 bg-slate-100 rounded-xl hover:bg-slate-200"
              >
                Cancel
              </button>
              <button 
                type="submit"
                className={`flex-1 py-3 font-bold text-white rounded-xl ${currentKidData.btn}`}
              >
                Add Chore
              </button>
            </div>
          </form>
        </div>
      )}

      {/* Add Responsibility Modal */}
      {isAddRespOpen && (
        <div className="fixed inset-0 bg-black/50 z-50 flex items-end sm:items-center justify-center p-4">
          <form 
            onSubmit={handleAddResponsibility}
            className="bg-white w-full max-w-sm rounded-2xl p-6 shadow-2xl animate-in slide-in-from-bottom-10"
          >
            <h3 className="text-lg font-bold mb-4 flex items-center gap-2">
              <Star className="w-5 h-5 text-yellow-500 fill-yellow-500" />
              Add Checkmark
            </h3>
            <div>
              <label className="block text-sm font-medium text-slate-600 mb-1">What did {activeKid} do?</label>
              <textarea
                autoFocus
                rows={3}
                placeholder="e.g. Helped carry in groceries..."
                value={newRespNote}
                onChange={e => setNewRespNote(e.target.value)}
                className="w-full p-3 bg-slate-50 border border-slate-200 rounded-xl focus:outline-none focus:ring-2 focus:ring-yellow-500 resize-none"
              />
            </div>
            <div className="flex gap-3 mt-6">
              <button 
                type="button" 
                onClick={() => setIsAddRespOpen(false)}
                className="flex-1 py-3 font-semibold text-slate-600 bg-slate-100 rounded-xl hover:bg-slate-200"
              >
                Cancel
              </button>
              <button 
                type="submit"
                className="flex-1 py-3 font-bold text-white bg-yellow-500 hover:bg-yellow-600 rounded-xl"
              >
                Add Star
              </button>
            </div>
          </form>
        </div>
      )}

      {/* Bottom Navigation */}
      <div className="fixed bottom-0 left-0 right-0 bg-white border-t border-slate-100 px-6 py-3 pb-safe z-40">
        <div className="max-w-md mx-auto flex justify-around items-center">
          <button 
            onClick={() => setActiveTab('chores')}
            className={`flex flex-col items-center gap-1 transition-colors ${activeTab === 'chores' ? 'text-blue-600' : 'text-slate-400'}`}
          >
            <LayoutList className="w-6 h-6" />
            <span className="text-xs font-medium">Chores</span>
          </button>
          
          <button 
            onClick={() => setActiveTab('responsibility')}
            className={`flex flex-col items-center gap-1 transition-colors ${activeTab === 'responsibility' ? 'text-yellow-600' : 'text-slate-400'}`}
          >
            <NotebookPen className="w-6 h-6" />
            <span className="text-xs font-medium">Responsibility</span>
          </button>

          <button 
            onClick={() => setActiveTab('money')}
            className={`flex flex-col items-center gap-1 transition-colors ${activeTab === 'money' ? 'text-green-600' : 'text-slate-400'}`}
          >
            <DollarSign className="w-6 h-6" />
            <span className="text-xs font-medium">Money</span>
          </button>
        </div>
      </div>
    </div>
  );
}



