import React, { useState, useEffect, useMemo, useCallback } from 'react';
import { 
  Layout, 
  Plus, 
  Trash2, 
  CheckCircle2, 
  Clock, 
  Users, 
  MessageSquare, 
  Search, 
  Settings, 
  LogOut, 
  AlertCircle,
  Sparkles,
  ChevronRight,
  Filter
} from 'lucide-react';
import { initializeApp } from 'firebase/app';
import { 
  getAuth, 
  signInAnonymously, 
  signInWithCustomToken, 
  onAuthStateChanged 
} from 'firebase/auth';
import { 
  getFirestore, 
  collection, 
  doc, 
  onSnapshot, 
  addDoc, 
  updateDoc, 
  deleteDoc, 
  query, 
  serverTimestamp 
} from 'firebase/firestore';

/**
 * FIREBASE CONFIGURATION & INITIALIZATION
 * Following Rule 1 & 3: Auth before queries, specific paths.
 */
const firebaseConfig = JSON.parse(__firebase_config);
const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);
const appId = typeof __app_id !== 'undefined' ? __app_id : 'project-dashboard-v1';

// Gemini API Configuration
const apiKey = ""; 

export default function App() {
  const [user, setUser] = useState(null);
  const [tasks, setTasks] = useState([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  const [activeTab, setActiveTab] = useState('board'); // 'board' | 'team' | 'ai'
  const [isAddingTask, setIsAddingTask] = useState(false);
  const [aiSuggestion, setAiSuggestion] = useState('');
  const [isGenerating, setIsGenerating] = useState(false);

  // Auth Initialization (Rule 3)
  useEffect(() => {
    let mounted = true;

    const initAuth = async () => {
      try {
        if (typeof __initial_auth_token !== 'undefined' && __initial_auth_token) {
          await signInWithCustomToken(auth, __initial_auth_token);
        } else {
          await signInAnonymously(auth);
        }
      } catch (err) {
        if (mounted) setError("Authentication failed: " + err.message);
      }
    };

    initAuth();
    const unsubscribe = onAuthStateChanged(auth, (currentUser) => {
      if (mounted) {
        // Only update state if user UID actually changes to prevent recursion
        setUser(prev => (prev?.uid === currentUser?.uid ? prev : currentUser));
        // We handle loading in the data sync effect for a better UX, 
        // but if there's no user, we stop loading here.
        if (!currentUser) setLoading(false);
      }
    });

    return () => {
      mounted = false;
      unsubscribe();
    };
  }, []);

  // Firestore Data Sync (Rule 1 & 2)
  // Fix: Use user.uid as dependency to prevent infinite loops if user object reference is unstable
  useEffect(() => {
    if (!user?.uid) return;

    const tasksCollection = collection(db, 'artifacts', appId, 'public', 'data', 'tasks');
    
    const unsubscribe = onSnapshot(tasksCollection, 
      (snapshot) => {
        const tasksData = snapshot.docs.map(doc => ({
          id: doc.id,
          ...doc.data()
        }));
        // Sort in memory (Rule 2)
        const sortedTasks = tasksData.sort((a, b) => {
          const timeA = a.createdAt?.seconds || 0;
          const timeB = b.createdAt?.seconds || 0;
          return timeB - timeA;
        });
        setTasks(sortedTasks);
        setLoading(false);
      },
      (err) => {
        console.error("Firestore error:", err);
        setError("Sync error: check connection/permissions.");
        setLoading(false);
      }
    );

    return () => unsubscribe();
  }, [user?.uid]);

  const addTask = useCallback(async (e) => {
    e.preventDefault();
    if (!user) return;

    const formData = new FormData(e.currentTarget);
    const title = formData.get('title');
    const description = formData.get('description');
    const priority = formData.get('priority');

    try {
      const tasksCollection = collection(db, 'artifacts', appId, 'public', 'data', 'tasks');
      await addDoc(tasksCollection, {
        title,
        description,
        priority,
        status: 'todo',
        creatorId: user.uid,
        creatorName: user.displayName || 'Team Member',
        createdAt: serverTimestamp(),
      });
      setIsAddingTask(false);
    } catch (err) {
      setError("Could not add task: " + err.message);
    }
  }, [user]);

  const toggleTaskStatus = useCallback(async (taskId, currentStatus) => {
    if (!user) return;
    const nextStatus = currentStatus === 'done' ? 'todo' : 'done';
    try {
      const taskDoc = doc(db, 'artifacts', appId, 'public', 'data', 'tasks', taskId);
      await updateDoc(taskDoc, { status: nextStatus });
    } catch (err) {
      setError("Update failed: " + err.message);
    }
  }, [user]);

  const deleteTask = useCallback(async (taskId) => {
    if (!user) return;
    try {
      const taskDoc = doc(db, 'artifacts', appId, 'public', 'data', 'tasks', taskId);
      await deleteDoc(taskDoc);
    } catch (err) {
      setError("Delete failed: " + err.message);
    }
  }, [user]);

  const generateAISuggestion = async () => {
    if (!tasks.length) return;
    setIsGenerating(true);
    setAiSuggestion('');
    
    const taskContext = tasks.map(t => `${t.title} (${t.status})`).join(', ');
    const prompt = `Based on these project tasks: ${taskContext}, suggest 3 immediate next steps to improve efficiency. Keep it concise and professional.`;

    try {
      let retries = 0;
      const fetchWithRetry = async () => {
        const response = await fetch(`https://generativelanguage.googleapis.com/v1beta/models/gemini-2.5-flash-preview-09-2025:generateContent?key=${apiKey}`, {
          method: 'POST',
          headers: { 'Content-Type': 'application/json' },
          body: JSON.stringify({
            contents: [{ parts: [{ text: prompt }] }],
            systemInstruction: { parts: [{ text: "You are a senior project manager AI assistant." }] }
          })
        });
        
        if (!response.ok && retries < 5) {
          const delay = Math.pow(2, retries) * 1000;
          retries++;
          await new Promise(resolve => setTimeout(resolve, delay));
          return fetchWithRetry();
        }
        return response.json();
      };

      const result = await fetchWithRetry();
      const text = result.candidates?.[0]?.content?.parts?.[0]?.text;
      setAiSuggestion(text || "No suggestion available.");
    } catch (err) {
      setError("Gemini API error: " + err.message);
    } finally {
      setIsGenerating(false);
    }
  };

  if (loading) {
    return (
      <div className="flex items-center justify-center h-screen bg-slate-50">
        <div className="flex flex-col items-center gap-4">
          <div className="w-12 h-12 border-4 border-indigo-500 border-t-transparent rounded-full animate-spin"></div>
          <p className="text-slate-600 font-medium animate-pulse text-sm uppercase tracking-widest">Syncing Workspace...</p>
        </div>
      </div>
    );
  }

  return (
    <div className="min-h-screen bg-slate-50 flex flex-col md:flex-row text-slate-900 font-sans">
      {/* Sidebar - Desktop */}
      <aside className="hidden md:flex w-64 flex-col bg-white border-r border-slate-200 p-6">
        <div className="flex items-center gap-3 mb-10">
          <div className="w-10 h-10 bg-indigo-600 rounded-xl flex items-center justify-center text-white shadow-lg shadow-indigo-200">
            <Layout size={24} />
          </div>
          <h1 className="font-bold text-xl tracking-tight text-slate-800">ProSync</h1>
        </div>

        <nav className="flex-1 space-y-2">
          <button 
            onClick={() => setActiveTab('board')}
            className={`w-full flex items-center gap-3 px-4 py-3 rounded-lg transition-all ${activeTab === 'board' ? 'bg-indigo-50 text-indigo-700 font-semibold shadow-sm border border-indigo-100' : 'text-slate-500 hover:bg-slate-50 border border-transparent'}`}
          >
            <Layout size={20} />
            Board
          </button>
          <button 
            onClick={() => setActiveTab('team')}
            className={`w-full flex items-center gap-3 px-4 py-3 rounded-lg transition-all ${activeTab === 'team' ? 'bg-indigo-50 text-indigo-700 font-semibold shadow-sm border border-indigo-100' : 'text-slate-500 hover:bg-slate-50 border border-transparent'}`}
          >
            <Users size={20} />
            Team
          </button>
          <button 
            onClick={() => setActiveTab('ai')}
            className={`w-full flex items-center gap-3 px-4 py-3 rounded-lg transition-all ${activeTab === 'ai' ? 'bg-indigo-50 text-indigo-700 font-semibold shadow-sm border border-indigo-100' : 'text-slate-500 hover:bg-slate-50 border border-transparent'}`}
          >
            <Sparkles size={20} />
            AI Insights
          </button>
        </nav>

        <div className="mt-auto pt-6 border-t border-slate-100">
          <div className="p-4 bg-slate-50 rounded-xl border border-slate-100">
            <p className="text-[10px] text-slate-400 uppercase font-black mb-2 tracking-tighter">My Identifier</p>
            <p className="text-[10px] font-mono break-all text-slate-600 select-all">{user?.uid}</p>
          </div>
        </div>
      </aside>

      {/* Main Content */}
      <main className="flex-1 flex flex-col min-w-0">
        <header className="bg-white/80 backdrop-blur-md border-b border-slate-200 px-6 py-4 flex items-center justify-between sticky top-0 z-10">
          <div className="flex items-center gap-4">
            <h2 className="text-lg font-bold text-slate-800 capitalize">{activeTab}</h2>
            <div className="h-6 w-px bg-slate-200 hidden sm:block"></div>
            <div className="relative hidden sm:block">
              <Search className="absolute left-3 top-1/2 -translate-y-1/2 text-slate-400" size={16} />
              <input 
                type="text" 
                placeholder="Find a task..." 
                className="pl-10 pr-4 py-1.5 bg-slate-100 border-none rounded-full text-sm focus:ring-2 focus:ring-indigo-500 w-64 transition-all"
              />
            </div>
          </div>
          
          <div className="flex items-center gap-3">
            <button 
              onClick={() => setIsAddingTask(true)}
              className="bg-indigo-600 hover:bg-indigo-700 text-white px-4 py-2 rounded-lg flex items-center gap-2 text-sm font-bold transition-all shadow-lg shadow-indigo-100 active:scale-95"
            >
              <Plus size={18} />
              <span className="hidden sm:inline">New Task</span>
            </button>
          </div>
        </header>

        <div className="p-6 overflow-y-auto">
          {error && (
            <div className="mb-6 p-4 bg-red-50 border border-red-100 rounded-xl flex items-start gap-3 text-red-700 animate-in fade-in slide-in-from-top-2">
              <AlertCircle size={20} className="shrink-0 mt-0.5" />
              <div className="flex-1">
                <p className="font-bold text-sm">System Alert</p>
                <p className="text-xs opacity-90">{error}</p>
              </div>
              <button onClick={() => setError(null)} className="text-red-400 hover:text-red-600 transition-colors">
                <Trash2 size={16} />
              </button>
            </div>
          )}

          {activeTab === 'board' && (
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
              {/* Kanban Columns */}
              <div className="flex flex-col gap-4">
                <div className="flex items-center justify-between px-2">
                  <h3 className="font-black text-slate-500 flex items-center gap-2 text-[11px] uppercase tracking-widest">
                    To Do
                    <span className="bg-slate-200 text-slate-600 text-[10px] px-2 py-0.5 rounded-full font-bold">
                      {tasks.filter(t => t.status === 'todo').length}
                    </span>
                  </h3>
                </div>
                <div className="flex flex-col gap-4">
                  {tasks.filter(t => t.status === 'todo').map(task => (
                    <TaskCard key={task.id} task={task} onToggle={toggleTaskStatus} onDelete={deleteTask} />
                  ))}
                  {tasks.filter(t => t.status === 'todo').length === 0 && (
                    <EmptyState text="Clean slate" />
                  )}
                </div>
              </div>

              <div className="flex flex-col gap-4">
                <div className="flex items-center justify-between px-2">
                  <h3 className="font-black text-slate-500 flex items-center gap-2 text-[11px] uppercase tracking-widest">
                    In Progress
                    <span className="bg-blue-100 text-blue-600 text-[10px] px-2 py-0.5 rounded-full font-bold">0</span>
                  </h3>
                </div>
                <div className="flex flex-col gap-4">
                  <EmptyState text="No active work" />
                </div>
              </div>

              <div className="flex flex-col gap-4">
                <div className="flex items-center justify-between px-2">
                  <h3 className="font-black text-emerald-600 flex items-center gap-2 text-[11px] uppercase tracking-widest">
                    Done
                    <span className="bg-emerald-100 text-emerald-600 text-[10px] px-2 py-0.5 rounded-full font-bold">
                      {tasks.filter(t => t.status === 'done').length}
                    </span>
                  </h3>
                </div>
                <div className="flex flex-col gap-4 opacity-70">
                  {tasks.filter(t => t.status === 'done').map(task => (
                    <TaskCard key={task.id} task={task} onToggle={toggleTaskStatus} onDelete={deleteTask} />
                  ))}
                </div>
              </div>
            </div>
          )}

          {activeTab === 'ai' && (
            <div className="max-w-3xl mx-auto space-y-6">
              <div className="bg-gradient-to-br from-slate-800 to-indigo-900 p-10 rounded-3xl text-white shadow-2xl relative overflow-hidden group">
                <div className="absolute top-0 right-0 w-64 h-64 bg-white/5 rounded-full -translate-y-1/2 translate-x-1/2 blur-3xl group-hover:bg-white/10 transition-all duration-700"></div>
                <div className="relative z-10">
                  <div className="flex items-center gap-3 mb-6">
                    <div className="p-3 bg-indigo-500/30 rounded-2xl backdrop-blur-md">
                      <Sparkles size={28} className="text-indigo-200" />
                    </div>
                    <h3 className="text-2xl font-black tracking-tight">AI Project Consultant</h3>
                  </div>
                  <p className="text-indigo-100 mb-8 text-lg leading-relaxed max-w-xl">
                    Get an instant summary of your current board and actionable advice on bottleneck management.
                  </p>
                  <button 
                    onClick={generateAISuggestion}
                    disabled={isGenerating || tasks.length === 0}
                    className="bg-white text-indigo-900 px-8 py-3.5 rounded-2xl font-black text-sm uppercase tracking-widest hover:bg-indigo-50 transition-all disabled:opacity-50 flex items-center gap-3 shadow-xl active:scale-95"
                  >
                    {isGenerating ? (
                      <><Clock className="animate-spin" size={18} /> Processing...</>
                    ) : (
                      <><Plus size={18} /> Analyze Board</>
                    )}
                  </button>
                </div>
              </div>

              {aiSuggestion && (
                <div className="bg-white border border-slate-200 p-8 rounded-3xl shadow-sm animate-in fade-in slide-in-from-bottom-4 duration-500">
                  <h4 className="font-black text-slate-800 mb-6 flex items-center gap-2 text-sm uppercase tracking-widest">
                    <MessageSquare size={18} className="text-indigo-500" />
                    Strategic Insight
                  </h4>
                  <div className="prose prose-indigo max-w-none text-slate-600 whitespace-pre-wrap leading-relaxed text-sm">
                    {aiSuggestion}
                  </div>
                </div>
              )}
            </div>
          )}

          {activeTab === 'team' && (
            <div className="max-w-4xl mx-auto">
              <div className="bg-white p-8 rounded-3xl border border-slate-200 shadow-sm">
                <h3 className="font-black text-slate-800 mb-8 flex items-center gap-3 text-sm uppercase tracking-widest">
                  <Users size={20} className="text-indigo-500" />
                  Collaborators
                </h3>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
                  <div className="flex items-center justify-between p-4 bg-indigo-50 rounded-2xl border border-indigo-100">
                    <div className="flex items-center gap-4">
                      <div className="w-12 h-12 bg-indigo-600 rounded-full flex items-center justify-center text-white font-black text-lg shadow-md">
                        {user?.displayName?.charAt(0) || 'U'}
                      </div>
                      <div>
                        <p className="font-bold text-slate-800">You (Owner)</p>
                        <p className="text-[10px] text-indigo-600 font-mono">{user?.uid}</p>
                      </div>
                    </div>
                    <span className="px-3 py-1 bg-white text-emerald-600 text-[10px] font-black rounded-full shadow-sm">ONLINE</span>
                  </div>
                  <div className="p-6 border-2 border-dashed border-slate-200 rounded-2xl flex flex-col items-center justify-center text-center group hover:border-indigo-300 transition-colors cursor-pointer">
                    <p className="text-xs font-bold text-slate-400 mb-2 group-hover:text-indigo-400">Share project access</p>
                    <button className="text-indigo-600 font-black text-[10px] uppercase tracking-widest">Copy Invite Link</button>
                  </div>
                </div>
              </div>
            </div>
          )}
        </div>
      </main>

      {/* Add Task Modal */}
      {isAddingTask && (
        <div className="fixed inset-0 bg-slate-900/60 backdrop-blur-sm z-50 flex items-center justify-center p-4">
          <div className="bg-white w-full max-w-md rounded-3xl shadow-2xl animate-in zoom-in-95 duration-200 overflow-hidden border border-white/20">
            <div className="px-8 py-6 border-b border-slate-100 flex items-center justify-between bg-slate-50">
              <h3 className="font-black text-slate-800 uppercase text-xs tracking-widest">Create New Task</h3>
              <button onClick={() => setIsAddingTask(false)} className="p-2 hover:bg-slate-200 rounded-full transition-colors">
                <LogOut size={18} className="rotate-90 text-slate-400" />
              </button>
            </div>
            <form onSubmit={addTask} className="p-8 space-y-6">
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase mb-2 tracking-widest">Title</label>
                <input 
                  name="title" 
                  required 
                  autoFocus
                  placeholder="e.g. Design Login UI"
                  className="w-full px-5 py-3.5 bg-slate-100 border-none rounded-2xl focus:ring-2 focus:ring-indigo-500 transition-all text-sm font-medium"
                />
              </div>
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase mb-2 tracking-widest">Description</label>
                <textarea 
                  name="description" 
                  rows="3"
                  placeholder="What are the core requirements?"
                  className="w-full px-5 py-3.5 bg-slate-100 border-none rounded-2xl focus:ring-2 focus:ring-indigo-500 transition-all text-sm font-medium resize-none"
                ></textarea>
              </div>
              <div>
                <label className="block text-[10px] font-black text-slate-400 uppercase mb-2 tracking-widest">Priority</label>
                <div className="grid grid-cols-3 gap-3">
                  {['low', 'medium', 'high'].map(p => (
                    <label key={p} className="cursor-pointer">
                      <input type="radio" name="priority" value={p} defaultChecked={p==='medium'} className="peer hidden" />
                      <div className="text-center py-2.5 rounded-xl border-2 border-slate-100 bg-slate-50 text-[10px] font-bold uppercase tracking-widest text-slate-400 peer-checked:border-indigo-600 peer-checked:bg-indigo-50 peer-checked:text-indigo-600 transition-all">
                        {p}
                      </div>
                    </label>
                  ))}
                </div>
              </div>
              <div className="pt-4 flex gap-3">
                <button 
                  type="button"
                  onClick={() => setIsAddingTask(false)}
                  className="flex-1 px-6 py-4 rounded-2xl text-slate-400 font-bold text-xs uppercase tracking-widest hover:bg-slate-50 transition-colors"
                >
                  Cancel
                </button>
                <button 
                  type="submit"
                  className="flex-1 bg-indigo-600 text-white px-6 py-4 rounded-2xl font-black text-xs uppercase tracking-widest hover:bg-indigo-700 transition-all shadow-xl shadow-indigo-100 active:scale-95"
                >
                  Confirm
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
    </div>
  );
}

/**
 * Task Card Component
 */
const TaskCard = React.memo(({ task, onToggle, onDelete }) => {
  const priorityStyles = {
    low: 'bg-emerald-100 text-emerald-700 border-emerald-200',
    medium: 'bg-amber-100 text-amber-700 border-amber-200',
    high: 'bg-rose-100 text-rose-700 border-rose-200'
  };

  return (
    <div className={`group bg-white p-5 rounded-3xl border border-slate-200 shadow-sm transition-all hover:shadow-xl hover:translate-y-[-2px] hover:border-indigo-300 relative ${task.status === 'done' ? 'bg-slate-50/50 grayscale opacity-60' : ''}`}>
      <div className="flex items-start justify-between mb-4">
        <span className={`text-[9px] font-black uppercase px-2.5 py-1 rounded-full border ${priorityStyles[task.priority] || priorityStyles.medium}`}>
          {task.priority}
        </span>
        <button 
          onClick={() => onDelete(task.id)}
          className="p-2 text-slate-300 hover:text-rose-500 rounded-full hover:bg-rose-50 transition-all opacity-0 group-hover:opacity-100"
        >
          <Trash2 size={14} />
        </button>
      </div>
      
      <h4 className={`font-bold text-sm mb-2 leading-tight ${task.status === 'done' ? 'line-through text-slate-400' : 'text-slate-800'}`}>
        {task.title}
      </h4>
      <p className="text-xs text-slate-500 line-clamp-2 mb-6 font-medium leading-relaxed">
        {task.description || "Describe this task..."}
      </p>

      <div className="flex items-center justify-between pt-4 border-t border-slate-100">
        <div className="flex items-center gap-2">
          <div className="w-7 h-7 rounded-full bg-slate-200 border-2 border-white flex items-center justify-center text-[10px] font-black text-slate-500 shadow-sm">
            {task.creatorName?.charAt(0) || 'U'}
          </div>
          <span className="text-[10px] text-slate-400 font-bold uppercase tracking-tighter">
            {task.createdAt?.seconds ? new Date(task.createdAt.seconds * 1000).toLocaleDateString() : 'Syncing...'}
          </span>
        </div>
        <button 
          onClick={() => onToggle(task.id, task.status)}
          className={`flex items-center gap-2 px-3 py-1.5 rounded-full text-[10px] font-black uppercase tracking-widest transition-all ${task.status === 'done' ? 'bg-emerald-50 text-emerald-500' : 'bg-indigo-50 text-indigo-600 hover:bg-indigo-100'}`}
        >
          {task.status === 'done' ? (
            <><CheckCircle2 size={12} /> Done</>
          ) : (
            <><Clock size={12} /> Pending</>
          )}
        </button>
      </div>
    </div>
  );
});

function EmptyState({ text }) {
  return (
    <div className="py-12 px-6 border-2 border-dashed border-slate-100 rounded-3xl flex flex-col items-center justify-center text-center">
      <div className="w-12 h-12 bg-slate-50 rounded-full flex items-center justify-center mb-4 text-slate-200">
        <Filter size={20} />
      </div>
      <p className="text-[10px] font-black text-slate-300 uppercase tracking-widest">{text}</p>
    </div>
  );
}
